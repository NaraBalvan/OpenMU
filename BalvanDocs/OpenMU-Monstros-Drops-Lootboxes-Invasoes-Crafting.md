# OpenMU Moderno: Monstros, Drops, Lootboxes, Invasões e Crafting

**Projeto:** MUnique/OpenMU  
**Branch:** `meu-servidor-mu`  
**Data:** 2026-06-13  
**Autor:** Documentação gerada por Kimi Code CLI para o servidor Balvan

---

## 1. Visão Geral

No OpenMU moderno, a configuração do jogo é centralizada em **entidades persistentes no banco de dados** (PostgreSQL, via Entity Framework Core). Os dados iniciais — monstros, drops, mapas, receitas de crafting, lootboxes e invasões — são criados por plugins de inicialização em C# localizados em:

```
src/Persistence/Initialization/
```

A hierarquia de versões funciona assim:

- `Version075` → base do jogo
- `Version095d` → herda e estende a base
- `VersionSeasonSix` → herda da 0.95d e adiciona conteúdo da Season 6

A entidade raiz é `GameConfiguration`, que agrega mapas, monstros, itens, grupos de drop e receitas de crafting.

> **Diferença principal para o MU antigo:** antigamente essas configurações ficavam em arquivos `.dat` e `.txt` editados manualmente. Hoje tudo é código C# de inicialização + banco de dados. Isso dá mais segurança, versionamento e possibilidade de testes, mas exige recompilar/reinicializar o banco para grandes mudanças iniciais.

---

## 2. Organização dos Monstros

### 2.1 Definição de Monstro

Todo monstro, NPC, guarda ou trap compartilha a mesma entidade: `MonsterDefinition`.

**Arquivo:** `src/DataModel/Configuration/MonsterDefinition.cs`

```csharp
public partial class MonsterDefinition
{
    public short Number { get; set; }
    public LocalizedString Designation { get; set; }
    public NpcWindow NpcWindow { get; set; }        // ChaosMachine, Merchant, PetTrainer...
    public NpcObjectKind ObjectKind { get; set; }   // Monster, PassiveNpc, Guard, Trap...
    public virtual ICollection<DropItemGroup> DropItemGroups { get; }
    public virtual ICollection<ItemCrafting.ItemCrafting> ItemCraftings { get; }
    public virtual ICollection<QuestDefinition> Quests { get; }
    public virtual ICollection<MonsterAttribute> Attributes { get; }
}
```

- **Monstros de ataque** têm `ObjectKind = Monster` e carregam `DropItemGroups`.
- **NPCs funcionais** (Chaos Goblin, Elphis, Pet Trainer, etc.) têm `NpcWindow` preenchido e `ItemCraftings` com receitas.
- **NPCs de quest** carregam `Quests`.

### 2.2 Classificação por Raridade

**O OpenMU não possui classificação nativa de Normal/Elite/Boss.** O enum `NpcObjectKind` apenas separa o **tipo de objeto**:

```csharp
public enum NpcObjectKind
{
    Monster,      // qualquer monstro hostil
    PassiveNpc,   // NPC passivo (mercador)
    Guard,        // guarda
    Trap,         // armadilha
    Gate,         // portão
    Statue,       // estátua
    SoccerBall,   // bola de futebol
    Destructible  // objeto destrutível
}
```

A única exceção que isola drops é `Destructible`: objetos destrutíveis usam **apenas** seus próprios `DropItemGroups`, ignorando os drops do mapa/personagem/quest.

### 2.3 Spawn de Monstros

Os spawns são definidos por `MonsterSpawnArea`, ligando um `MonsterDefinition` a um `GameMapDefinition`.

**Arquivo:** `src/DataModel/Configuration/MonsterSpawnArea.cs`

```csharp
public partial class MonsterSpawnArea
{
    public virtual MonsterDefinition? MonsterDefinition { get; set; }
    public virtual GameMapDefinition? GameMap { get; set; }
    public byte X1, Y1, X2, Y2;          // área retangular (ou ponto se iguais)
    public short Quantity { get; set; }
    public SpawnTrigger SpawnTrigger { get; set; }  // Automatic, AutomaticDuringEvent, OnceAtEventStart, Wandering...
    public byte WaveNumber { get; set; }
    public int? MaximumHealthOverride { get; set; }
}
```

### 2.4 Inicialização de Mapas

Cada mapa tem um initializer que herda de `BaseMapInitializer`.

**Arquivo:** `src/Persistence/Initialization/BaseMapInitializer.cs`

O `BaseMapInitializer` oferece:

- Helpers para criar spawns (`CreateMonsterSpawn`).
- Registro de `DropItemGroup`s padrão em todos os mapas (`RegisterDefaultDropItemGroup`).
- `InitializeDropItemGroups` que adiciona os grupos padrão ao mapa.

Exemplos de mapas:

| Mapa | Arquivo Season 6 |
|---|---|
| Lorencia | `VersionSeasonSix/Maps/Lorencia.cs` |
| Devias | `VersionSeasonSix/Maps/Devias.cs` |
| Dungeon | `VersionSeasonSix/Maps/Dungeon.cs` |
| Kalima 7 | `VersionSeasonSix/Maps/Kalima7.cs` |
| Raklion Boss | `VersionSeasonSix/Maps/RaklionBoss.cs` |
| Land of Trials | `VersionSeasonSix/Maps/LandOfTrials.cs` |
| Barracks of Balgass | `VersionSeasonSix/Maps/BarracksOfBalgass.cs` |
| Refuge of Balgass | `VersionSeasonSix/Maps/BalgassRefuge.cs` |

### 2.5 Onde Ficam os Bosses

Bosses como Kundun, Erohim, Selupan e Balgass são `MonsterDefinition` comuns, com spawns fixos em seus mapas e `DropItemGroups` específicos.

| Boss | Mapa | Arquivo típico |
|---|---|---|
| Kundun | Kalima 7 | `VersionSeasonSix/Maps/Kalima7.cs` |
| Erohim | Land of Trials | `VersionSeasonSix/Maps/LandOfTrials.cs` |
| Selupan | Raklion Boss | `VersionSeasonSix/Maps/RaklionBoss.cs` |
| Balgass | Refuge of Balgass | `VersionSeasonSix/Maps/BalgassRefuge.cs` |

---

## 3. Organização dos Drops

### 3.1 Entidade de Grupo de Drop

**Arquivo:** `src/DataModel/Configuration/DropItemGroup.cs`

```csharp
public partial class DropItemGroup
{
    public LocalizedString Description { get; set; }
    public double Chance { get; set; }              // 0.0 .. 1.0
    public byte? MinimumMonsterLevel, MaximumMonsterLevel;
    public virtual MonsterDefinition? Monster { get; set; }
    public byte? ItemLevel { get; set; }
    public SpecialItemType ItemType { get; set; }   // Money, RandomItem, Excellent, Ancient, SocketItem, Jewel
    public virtual ICollection<ItemDefinition> PossibleItems { get; }
}
```

### 3.2 Fontes de Drop

O `DefaultDropGenerator` reúne drops de quatro fontes:

1. **Monstro** (`monster.DropItemGroups`)
2. **Personagem** (`character.DropItemGroups`)
3. **Mapa** (`map.DropItemGroups`)
4. **Quests** (`GetQuestDropItemGroups` de `Character` / `Party`)

**Arquivo:** `src/GameLogic/DefaultDropGenerator.cs`

```csharp
public async ValueTask<(IEnumerable<Item> Items, uint? Money)> GenerateItemDropsAsync(
    MonsterDefinition monster, int gainedExperience, Player player)
{
    this.PartitionDropGroups(monster.DropItemGroups ?? []);
    this.PartitionDropGroups(character.DropItemGroups ?? [], monster);
    this.PartitionDropGroups(map.DropItemGroups ?? [], monster);
    this.PartitionDropGroups(await GetQuestItemGroupsAsync(player) ?? [], monster);
    return this.GenerateDrops(monster, gainedExperience);
}
```

### 3.3 Lógica de Geração

- `PartitionDropGroups` separa grupos **garantidos** (`Chance >= 1.0`) de grupos **probabilísticos**.
- Há filtro por level do monstro (`DropLevelMaxGap`, `CanDropAtMonsterLevel`).
- Gera atributos especiais: excellent, ancient, socket, random options, luck, skill.

### 3.4 Drops Padrão Globais

**Arquivo:** `src/Persistence/Initialization/GameConfigurationInitializerBase.cs`

```csharp
private void AddItemDropGroups()
{
    var moneyDropItemGroup = this.Context.CreateNew<DropItemGroup>();
    moneyDropItemGroup.Chance = 0.5;
    moneyDropItemGroup.ItemType = SpecialItemType.Money;
    BaseMapInitializer.RegisterDefaultDropItemGroup(moneyDropItemGroup);
    // randomItem 30%, excellent 0.01%, jewels 0.1%
}
```

Esses grupos são aplicados a **todos os mapas** via `BaseMapInitializer.InitializeDropItemGroups`.

### 3.5 Drops de Quest

**Arquivos:**

- `src/DataModel/Configuration/QuestItemRequirement.cs`
- `src/DataModel/Configuration/QuestDefinition.cs`
- `src/GameLogic/CharacterExtensions.cs`
- `src/GameLogic/Party.cs`

A extensão `Character.GetQuestDropItemGroups` retorna os `DropItemGroup`s dos itens requeridos das quests ativas. O `DefaultDropGenerator` os inclui no pool de drops. Existe também suporte a party (`Party.GetQuestDropItemGroupsAsync`).

### 3.6 Como Vincular Drops a Monstros Específicos

Para adicionar um drop exclusivo a um monstro:

```csharp
var itemDrop = this.Context.CreateNew<DropItemGroup>();
itemDrop.Chance = 1.0;                            // 100% de chance
itemDrop.Monster = monsterDefinition;             // vincula ao monstro
itemDrop.PossibleItems.Add(itemDefinition);       // item que vai dropar
monsterDefinition.DropItemGroups.Add(itemDrop);
```

> **Importante:** os drops do monstro são **adicionados** aos drops do mapa, não os substituem. Se o monstro estiver em um mapa com drop de ZEN de 50%, ele também terá chance de dropar ZEN.

### 3.7 Como Vincular Drops a um Mapa

```csharp
var dropGroup = this.Context.CreateNew<DropItemGroup>();
dropGroup.Chance = 0.1;
dropGroup.ItemType = SpecialItemType.Jewel;
dropGroup.PossibleItems.Add(jewelDefinition);
mapDefinition.DropItemGroups.Add(dropGroup);
```

---

## 4. Lootboxes (Box of Luck, Box of Kundun, etc.)

### 4.1 Conceito

Lootboxes são itens consumíveis que, ao serem dropados no chão, geram um item aleatório (ou ZEN). No OpenMU, são modeladas como `ItemDefinition` com uma coleção de `ItemDropItemGroup`.

### 4.2 Entidade ItemDropItemGroup

**Arquivo:** `src/DataModel/Configuration/ItemDropItemGroup.cs`

```csharp
public partial class ItemDropItemGroup : DropItemGroup
{
    public byte SourceItemLevel { get; set; }       // nível do item consumido
    public int MoneyAmount { get; set; }            // ZEN fallback
    public byte MinimumLevel { get; set; }          // nível mínimo do resultado
    public byte MaximumLevel { get; set; }          // nível máximo do resultado
    public short RequiredCharacterLevel { get; set; }
    public ItemDropEffect DropEffect { get; set; }  // Fireworks, FanfareSound, Swirl...
}
```

### 4.3 Ativação da Lootbox

**Arquivo:** `src/GameLogic/PlayerActions/Items/ItemBoxDroppedPlugIn.cs`

Quando o jogador dropa uma caixa no chão:

1. Verifica se o `ItemDefinition` possui `DropItems`.
2. Filtra grupos cujo `SourceItemLevel` corresponde ao level do item.
3. Verifica `RequiredCharacterLevel`.
4. Chama `player.GameContext.DropGenerator.GenerateItemDrop(itemDropGroups)`.
5. Gera o item resultante ou `DroppedMoney` no chão.
6. Aplica efeito visual (`ShowDropEffectAsync`).

### 4.4 Exemplo: Box of Luck (Season 6)

**Arquivo:** `src/Persistence/Initialization/VersionSeasonSix/Items/BoxOfLuck.cs`

```csharp
private void CreateBoxOfLuck()
{
    var box = this.CreateBox("Box of Luck", 14, 11);
    var boxOfLuck = this.Context.CreateNew<ItemDropItemGroup>();
    boxOfLuck.ItemType = SpecialItemType.RandomItem;
    boxOfLuck.SourceItemLevel = 0;      // Box of Luck normal
    boxOfLuck.Chance = 0.5;
    boxOfLuck.MinimumLevel = 6;
    boxOfLuck.MaximumLevel = 6;
    box.DropItems.Add(boxOfLuck);

    this.AddDropItem(boxOfLuck, 0, 3);  // Katana
    // ... outros itens

    this.AddMoneyDropFallback(box, 10000, boxOfLuck);  // fallback de ZEN
}
```

### 4.5 Box of Kundun +1..+5

No mesmo arquivo, a Season 6 define vários níveis da mesma caixa (`SourceItemLevel`):

| Caixa | SourceItemLevel | Exemplos de drops |
|---|---|---|
| Box of Luck | 0 | Itens aleatórios +6, ZEN fallback |
| Star of Christmas | 1 | Event items |
| Firecrackers | 2 | Event items |
| Heart of Love | 3 | Event items |
| Silver/Gold Medal | 4/5 | Event items |
| Box of Heaven | 6 | Event items |
| Box of Kundun +1 | 8 | Itens raros/joias |
| Box of Kundun +2 | 9 | Itens raros/joias |
| Box of Kundun +3 | 10 | Itens raros/joias |
| Box of Kundun +4 | 11 | Itens raros/joias |
| Box of Kundun +5 | 12 | Itens raros/joias |

Os monstros de invasão Golden/Red Dragon dropam Box of Kundun com `ItemLevel = 7 + lvl`.

---

## 5. Invasões (Golden Invasion, Red Dragon)

### 5.1 Arquitetura Base

**Arquivo:** `src/GameLogic/PlugIns/InvasionEvents/BaseInvasionPlugIn.cs`

Invasões são plugins periódicos (`PeriodicTaskBasePlugIn`) que:

1. Preparam o evento (`OnPrepareEventAsync`)
2. Iniciam (`OnStartedAsync`)
3. Spawnam monstros dinamicamente
4. Finalizam e limpam (`OnFinishedAsync`)

```csharp
protected async ValueTask CreateMonstersAsync(IGameContext gameContext, ILogger logger,
    GameMap gameMap, MonsterDefinition monsterDefinition, ushort quantity, byte? x = null, byte? y = null)
{
    for (var i = 0; i < quantity; i++)
    {
        var area = new MonsterSpawnArea
        {
            GameMap = gameMap.Definition,
            MonsterDefinition = monsterDefinition,
            SpawnTrigger = SpawnTrigger.OnceAtEventStart,
            Quantity = 1,
            X1 = x ?? p.X, ...
        };
        var monster = new Monster(area, monsterDefinition, gameMap, ...);
        monster.Initialize();
        await gameMap.AddAsync(monster);
        state.AddMonster(monster);
    }
}
```

### 5.2 Configurações Padrão

**Arquivo:** `src/GameLogic/PlugIns/InvasionEvents/InvasionConfigurationDefaults.cs`

```csharp
internal static class InvasionConfigurationDefaults
{
    public static PeriodicInvasionConfiguration Golden => new()
    {
        TaskDuration = TimeSpan.FromMinutes(30),
        Timetable = PeriodicTaskConfiguration.GenerateTimeSequence(TimeSpan.FromHours(4)).ToList(),
        Mobs = [
            new(InvasionMonsters.GoldenBudgeDragon, 20, [InvasionMaps.Lorencia], SpawnMapStrategy.RandomMap),
            new(InvasionMonsters.GoldenGoblin, ...),
            ...
        ]
    };
}
```

### 5.3 Invasões Implementadas

| Invasão | Arquivo | Monstros |
|---|---|---|
| Golden Invasion | `GoldenInvasionPlugIn.cs` | Golden Budge Dragon, Goblin, Soldier, Titan, Vepar, Lizard King, Wheel, Tantallos, Dragon |
| Red Dragon Invasion | `RedDragonInvasionPlugIn.cs` | 5 Red Dragons em Lorencia/Noria/Devias |

### 5.4 Definição dos Monstros de Invasão

**Arquivos:**

- `src/Persistence/Initialization/Version095d/InvasionMobsInitialization.cs`
- `src/Persistence/Initialization/VersionSeasonSix/InvasionMobsInitialization.cs`

Exemplo: adicionar Box of Kundun aos monstros Golden:

```csharp
protected void AddBoxOfKundunToMonster(byte lvl, MonsterDefinition monster)
{
    var itemDrop = this.Context.CreateNew<DropItemGroup>();
    itemDrop.Chance = 1;
    itemDrop.ItemLevel = (byte)(7 + lvl);
    itemDrop.Description = $"Box of Kundun +{lvl}";
    itemDrop.Monster = monster;
    itemDrop.PossibleItems.Add(this.GameConfiguration.Items.First(item => item.Group == 14 && item.Number == 11));
    monster.DropItemGroups.Add(itemDrop);
}
```

O Red Dragon dropa joias diretamente:

```csharp
itemDrop.PossibleItems.Add(bless);
itemDrop.PossibleItems.Add(soul);
itemDrop.PossibleItems.Add(chaos);
```

### 5.5 Monstros de Invasão Recebem Drops do Mapa?

**Sim.** Os monstros de invasão são spawnados em mapas normais (Lorencia, Noria, Devias, Atlans, Tarkan) e usam o `DefaultDropGenerator` padrão do jogo. Como o mapa possui os grupos padrão (ZEN 50%, random item 30%, etc.), os monstros de invasão **também podem dropar ZEN e itens aleatórios do mapa**, além de seus drops específicos garantidos.

### 5.6 Tabela de Monstros de Invasão

| Monstro | Tipo | Drops notáveis |
|---|---|---|
| Golden Budge Dragon | Invasão Golden | Box of Kundun +1 |
| Golden Goblin | Invasão Golden | Box of Kundun +1/+2 |
| Golden Soldier | Invasão Golden | Box of Kundun +2/+3 |
| Golden Titan | Invasão Golden | Box of Kundun +3/+4 |
| Golden Vepar | Invasão Golden | Box of Kundun +4/+5 |
| Golden Lizard King | Invasão Golden | Box of Kundun +5 |
| Golden Wheel | Invasão Golden | Box of Kundun +5 |
| Golden Tantallos | Invasão Golden | Box of Kundun +5 |
| Golden Dragon | Invasão Golden | Box of Kundun +5 |
| Red Dragon | Invasão Red Dragon | Jewels (Bless, Soul, Chaos) |

---

## 6. Crafting (Chaos Machine / Goblin Craft)

### 6.1 Entidade de Receita

**Arquivo:** `src/DataModel/Configuration/ItemCrafting.cs`

Cada NPC funcional (`MonsterDefinition`) carrega uma coleção de `ItemCrafting`. A receita pode usar:

- `SimpleCraftingSettings` → handler genérico
- `ItemCraftingHandlerClassName` → handler customizado

### 6.2 Handler Genérico

**Arquivo:** `src/GameLogic/PlayerActions/Items/SimpleItemCraftingHandler.cs`

```csharp
public override CraftingResult? TryGetRequiredItems(Player player, out IList<CraftingRequiredItemLink> items, out byte successRate)
{
    int rate = this._settings.SuccessPercent;
    foreach (var requiredItem in this._settings.RequiredItems.OrderByDescending(i => i.MinimumAmount))
    {
        var foundItems = storage.Where(item => this.RequiredItemMatches(item, requiredItem)).ToList();
        if (itemCount < requiredItem.MinimumAmount) return CraftingResult.LackingMixItems;
        // adiciona % por luck, exc, ancient, guardian, socket, NpcPriceDivisor
    }
    successRate = (byte)Math.Min(100, rate);
    return default;
}
```

O handler:

- Valida itens requeridos
- Calcula taxa de sucesso com bônus (luck, excellent, ancient, guardian, socket)
- Cobra ZEN
- Cria/modifica itens resultantes (`CreateOrModifyResultItemsAsync`)
- Adiciona luck, skill, excellent options aleatórios

### 6.3 Handler Base

**Arquivo:** `src/GameLogic/PlayerActions/Items/BaseItemCraftingHandler.cs`

Define a estrutura comum: validação, consumo de itens, cobrança de ZEN, criação de resultado e notificação ao jogador.

### 6.4 Configuração das Receitas (Season 6)

**Arquivo:** `src/Persistence/Initialization/VersionSeasonSix/ChaosMixes.cs`

```csharp
public override void Initialize()
{
    var chaosGoblin = this.GameConfiguration.Monsters.First(m => m.NpcWindow == NpcWindow.ChaosMachine);
    chaosGoblin.ItemCraftings.Add(this.ChaosWeaponCrafting());
    chaosGoblin.ItemCraftings.Add(this.FirstWingsCrafting());
    chaosGoblin.ItemCraftings.Add(this.ItemLevelUpgradeCrafting(3, 10));
    // ...
}
```

Além do Chaos Goblin, existem NPCs especializados:

| NPC | `NpcWindow` | Exemplos de receitas |
|---|---|---|
| Chaos Goblin | `ChaosMachine` | Chaos Weapon, First Wings, Item Level Upgrades, Seed Spheres |
| Pet Trainer | `PetTrainer` | Dark Horse, Dark Spirit, Fenrir |
| Elphis | `ElphisRefinery` | Refinery |
| Osbourne | `OsbourneRefinery` | Refinery |
| Jerridon | `JerridonRefinery` | Refinery |
| Cherry Blossom Spirit | `CherryBlossomSpirit` | Event mixes |

---

## 7. Sistema de ZEN Atual

### 7.1 Representação do ZEN

O ZEN é tratado como um valor inteiro no inventário (`ItemStorage.Money`).

**Arquivos:**

- `src/GameLogic/Player.cs`
- `src/GameLogic/DroppedMoney.cs`
- `src/DataModel/Entities/ItemStorage.cs`

```csharp
public int Money
{
    get => this.SelectedCharacter?.Inventory?.Money ?? 0;
    set
    {
        if (this.SelectedCharacter?.Inventory.Money != value)
        {
            this.SelectedCharacter.Inventory.Money = value;
            _ = this.InvokeViewPlugInAsync<IUpdateMoneyPlugIn>(p => p.UpdateMoneyAsync());
        }
    }
}
```

### 7.2 Drop de ZEN

**Arquivo:** `src/GameLogic/DefaultDropGenerator.cs`

A fórmula atual de drop de ZEN é:

```csharp
droppedMoney = (uint)(gainedExperience + BaseMoneyDrop);  // BaseMoneyDrop = 7
```

Ou seja:

```text
ZenDrop = Experiência do Monstro + 7
```

Depois disso, pode ser aplicado:

- `Stats.MoneyAmountRate` — multiplicador global (padrão `1.0f`).
- Divisão em party, se aplicável.

### 7.3 Pickup de ZEN

**Arquivo:** `src/GameLogic/DroppedMoney.cs`

```csharp
public async ValueTask<bool> TryPickUpByAsync(Player player)
{
    if (player.Party is { } party)
    {
        var share = (int)(this.Amount / partyMembers.Count);
        foreach (var member in partyMembers)
            member.TryAddMoney((int)(share * member.Attributes![Stats.MoneyAmountRate]));
    }
    else
    {
        // TryAddMoney com ClampMoneyOnPickup
    }
}
```

### 7.4 Configuração do Drop de ZEN Padrão

**Arquivo:** `src/Persistence/Initialization/GameConfigurationInitializerBase.cs`

```csharp
var moneyDropItemGroup = this.Context.CreateNew<DropItemGroup>();
moneyDropItemGroup.SetGuid(1);
moneyDropItemGroup.Chance = 0.5;                 // 50% de chance
moneyDropItemGroup.ItemType = SpecialItemType.Money;
moneyDropItemGroup.Description = "The common money drop item group (50 % drop chance)";
this.GameConfiguration.DropItemGroups.Add(moneyDropItemGroup);
BaseMapInitializer.RegisterDefaultDropItemGroup(moneyDropItemGroup);
```

Esse grupo é aplicado a **todos os mapas** via `BaseMapInitializer.InitializeDropItemGroups`.

---

## 8. Proposta: Sistema Modular de Drop de ZEN

### 8.1 Fórmula Proposta para Monstros Normais

```text
ZenDropNormal = VALOR_MAPA × NIVEL_MONSTRO × MULTIPLICADOR_SERVER
```

Onde:

- `VALOR_MAPA`: peso numérico atribuído a cada mapa (ex: Lorencia = 10, Tarkan = 25, Raklion = 50).
- `NIVEL_MONSTRO`: nível nativo do monstro configurado na tabela.
- `MULTIPLICADOR_SERVER`: variável global do servidor (ex: 1 para Hard, 5 para Medium, 50 para Easy).

### 8.2 Fórmula Proposta para Monstros Especiais

```text
ZenDropEspecial = VALOR_FIXO_OU_TABELA × MULTIPLICADOR_SERVER
```

Monstros especiais/eventos (Kundun, Golden, Red Dragon, bosses de invasão) teriam valores fixos ou uma tabela própria, independente do mapa onde nascerem.

### 8.3 Necessidades Iniciais para Implementar

1. **Adicionar `MapZenValue` em `GameMapDefinition`**
   - Uma propriedade numérica que representa `VALOR_MAPA`.

2. **Adicionar `ZenDropType` ou flag em `MonsterDefinition`**
   - Para classificar se o monstro usa a fórmula normal ou a especial.
   - Ou adicionar uma propriedade `IsSpecialMonster`/`IgnoreMapDrops`.

3. **Adicionar `ServerZenMultiplier` em `GameConfiguration`**
   - Variável global `MULTIPLICADOR_SERVER`.

4. **Modificar `DefaultDropGenerator`**
   - Quando o grupo de ZEN for selecionado, usar a fórmula correspondente ao tipo do monstro.
   - Para monstros especiais, consultar uma tabela de valores fixos.

5. **Migrations do banco de dados**
   - Adicionar colunas nas tabelas correspondentes.

6. **Inicialização de dados**
   - Preencher `VALOR_MAPA` para todos os mapas.
   - Preencher classificação de monstros especiais.

### 8.4 Limitações

- **Não há classificação nativa de Normal/Elite/Boss** — precisaria criar uma flag ou enum.
- **Monstros especiais recebem drops do mapa** — para isolá-los completamente, seria necessária uma flag `IgnoreMapDrops`.
- **ZEN é hardcoded** como única moeda — qualquer sistema de múltiplas moedas exigiria mudanças estruturais maiores.
- **Inicialização em C#** — toda mudança nos dados iniciais exige recompilação/reinicialização do banco (diferente do antigo `.txt` que podia ser editado em produção).

### 8.5 Pergunta-chave

> Dá para fazer a classificação sem editar monstro por monstro?

**Não automaticamente.** O OpenMU não tem uma classificação pronta. As opções são:

1. **Adicionar flag no `MonsterDefinition`** e marcar cada monstro especial manualmente.
2. **Classificar por heurísticas** (ex: monstro com `DropItemGroups` garantidos próprios → especial), mas isso é frágil.
3. **Manter uma lista externa** de números de monstros especiais, mas dificulta manutenção.

Com agents de IA dá para automatizar a marcação, mas o ideal é ter uma abordagem data-driven para facilitar correções futuras.

---

## 9. Arquivos Mais Relevantes

### Modelo de Dados / Configuração

| Arquivo | Descrição |
|---|---|
| `src/DataModel/Configuration/MonsterDefinition.cs` | Definição de monstro/NPC |
| `src/DataModel/Configuration/MonsterSpawnArea.cs` | Spawn de monstros nos mapas |
| `src/DataModel/Configuration/MonsterAttribute.cs` | Atributos de monstros (HP, dano, etc.) |
| `src/DataModel/Configuration/DropItemGroup.cs` | Grupo de drop genérico |
| `src/DataModel/Configuration/ItemDropItemGroup.cs` | Grupo de drop de lootbox/caixa |
| `src/DataModel/Configuration/GameConfiguration.cs` | Configuração global do jogo |
| `src/DataModel/Configuration/GameMapDefinition.cs` | Definição de mapa |
| `src/DataModel/Configuration/ItemCrafting.cs` | Receita de crafting |
| `src/DataModel/Configuration/ItemCrafting/SimpleCraftingSettings.cs` | Configuração de crafting simples |
| `src/DataModel/Entities/ItemStorage.cs` | Armazena ZEN do inventário/vault |

### Lógica do Jogo

| Arquivo | Descrição |
|---|---|
| `src/GameLogic/DefaultDropGenerator.cs` | Gera drops de itens e ZEN |
| `src/GameLogic/IDropGenerator.cs` | Interface do gerador de drops |
| `src/GameLogic/DroppedMoney.cs` | Objeto de ZEN no chão |
| `src/GameLogic/Player.cs` | Propriedade `Money` e operações |
| `src/GameLogic/NPC/AttackableNpcBase.cs` | Morte do monstro e chamada de drop |
| `src/GameLogic/ItemPriceCalculator.cs` | Preços de compra/venda/reparo |
| `src/GameLogic/Attributes/Stats.cs` | `MoneyAmountRate` |

### Ações de Jogador

| Arquivo | Descrição |
|---|---|
| `src/GameLogic/PlayerActions/Items/PickupItemAction.cs` | Pegar item/ZEN do chão |
| `src/GameLogic/PlayerActions/Items/SellItemToNpcAction.cs` | Vender item a NPC |
| `src/GameLogic/PlayerActions/Items/BuyNpcItemAction.cs` | Comprar item de NPC |
| `src/GameLogic/PlayerActions/Items/ItemRepairAction.cs` | Reparar item |
| `src/GameLogic/PlayerActions/Items/SimpleItemCraftingHandler.cs` | Handler de crafting |
| `src/GameLogic/PlayerActions/Items/ItemBoxDroppedPlugIn.cs` | Ativação de lootbox |
| `src/GameLogic/PlayerActions/Trade/TradeMoneyAction.cs` | Trade de ZEN |
| `src/GameLogic/PlayerActions/WarpAction.cs` | Custo de warp |

### Invasões e Eventos

| Arquivo | Descrição |
|---|---|
| `src/GameLogic/PlugIns/InvasionEvents/BaseInvasionPlugIn.cs` | Base das invasões |
| `src/GameLogic/PlugIns/InvasionEvents/InvasionConfigurationDefaults.cs` | Configuração padrão de invasões |
| `src/GameLogic/PlugIns/InvasionEvents/GoldenInvasionPlugIn.cs` | Invasão Golden |
| `src/GameLogic/PlugIns/InvasionEvents/RedDragonInvasionPlugIn.cs` | Invasão Red Dragon |
| `src/GameLogic/MiniGames/MiniGameContext.cs` | Contexto de mini-games |
| `src/GameLogic/MiniGames/ChaosCastleContext.cs` | Chaos Castle com drop customizado |
| `src/GameLogic/MiniGames/ChaosCastleDropGenerator.cs` | Gerador de drops do Chaos Castle |

### Inicialização de Dados

| Arquivo | Descrição |
|---|---|
| `src/Persistence/Initialization/GameConfigurationInitializerBase.cs` | Grupos de drop padrão |
| `src/Persistence/Initialization/BaseMapInitializer.cs` | Aplica drops padrão aos mapas |
| `src/Persistence/Initialization/Version095d/InvasionMobsInitialization.cs` | Monstros de invasão 0.95d |
| `src/Persistence/Initialization/VersionSeasonSix/InvasionMobsInitialization.cs` | Monstros de invasão Season 6 |
| `src/Persistence/Initialization/VersionSeasonSix/Items/BoxOfLuck.cs` | Lootboxes Season 6 |
| `src/Persistence/Initialization/VersionSeasonSix/ChaosMixes.cs` | Receitas do Chaos Goblin Season 6 |
| `src/Persistence/Initialization/VersionSeasonSix/Maps/Kalima7.cs` | Kundun e Kalima 7 |
| `src/Persistence/Initialization/VersionSeasonSix/Maps/LandOfTrials.cs` | Erohim e Land of Trials |
| `src/Persistence/Initialization/VersionSeasonSix/Maps/RaklionBoss.cs` | Selupan |
| `src/Persistence/Initialization/VersionSeasonSix/Maps/BalgassRefuge.cs` | Balgass |

---

## 10. Conclusão

O OpenMU moderno organiza monstros, drops, lootboxes, invasões e crafting de forma altamente centralizada e data-driven:

- **Monstros/NPCs** são `MonsterDefinition`, reutilizados para spawns, crafting e quests.
- **Spawns** são `MonsterSpawnArea`, podendo ser automáticos, eventuais ou de invasão.
- **Drops** vêm de múltiplas fontes (monstro, mapa, personagem, quest) e são processados por `DefaultDropGenerator`.
- **Lootboxes** usam `ItemDropItemGroup` e são ativadas por `ItemBoxDroppedPlugIn`.
- **Invasões** são plugins periódicos que spawnam monstros dinamicamente via `BaseInvasionPlugIn`.
- **Crafting** associa receitas a NPCs através de `ItemCrafting`, com handlers genéricos ou customizados.
- **ZEN** é uma moeda única hardcoded no inventário; qualquer modularização exigiria mudanças estruturais no DataModel, GameLogic e persistência.

A proposta de criar um sistema modular de drop de ZEN com fórmulas separadas para monstros normais e especiais é viável, mas exige:

1. Adicionar campos no modelo de dados (`GameMapDefinition`, `MonsterDefinition`, `GameConfiguration`).
2. Modificar a lógica de geração de ZEN em `DefaultDropGenerator`.
3. Criar migrations para o banco de dados.
4. Configurar os valores iniciais para todos os mapas e monstros especiais.
5. Decidir se monstros especiais continuarão recebendo drops padrão do mapa ou serão isolados.

---

*Documentação gerada automaticamente para auxiliar no planejamento e desenvolvimento do servidor OpenMU Balvan.*
