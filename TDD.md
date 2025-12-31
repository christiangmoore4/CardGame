# TECHNICAL DESIGN DOCUMENT
## Card-Based Roguelike: "The Guild Builder"
### Version 1.0 - Implementation Specification

**Reference Document:** GDD v2.0
**Last Updated:** 2025-12-30
**Status:** Ready for Development

---

## TABLE OF CONTENTS

1. [Technology Stack](#1-technology-stack)
2. [System Architecture](#2-system-architecture)
3. [Game State Machine](#3-game-state-machine)
4. [Data Structures](#4-data-structures)
5. [Core Systems](#5-core-systems)
6. [UI/UX Specifications](#6-uiux-specifications)
7. [Graphics Pipeline](#7-graphics-pipeline)
8. [Audio System](#8-audio-system)
9. [Save System](#9-save-system)
10. [Implementation Roadmap](#10-implementation-roadmap)
11. [File Structure](#11-file-structure)
12. [Testing Strategy](#12-testing-strategy)

---

## 1. TECHNOLOGY STACK

### 1.1 Recommended Stack (Unity/C#)

**Engine:** Unity 2022.3 LTS or later
- **Pros:**
  - Mature 2D support
  - Excellent UI system (Unity UI/UI Toolkit)
  - Strong asset pipeline
  - Cross-platform (PC, Mac, Mobile, Web)
  - Large community and assets
- **Cons:**
  - Licensing for commercial use
  - Heavier runtime

**Language:** C# 10+
- Modern features (records, pattern matching)
- Strong typing for card definitions
- LINQ for collection operations

**UI Framework:** Unity UI Toolkit (recommended) or Unity UI (fallback)
- UI Toolkit for modern, scalable UI
- UXML for UI layouts
- USS for styling

**Data Format:** JSON + ScriptableObjects
- ScriptableObjects for card definitions
- JSON for save data, meta-progression
- Easy to edit and version control

**Version Control:** Git + Git LFS (for art assets)

**Build Targets (Initial):** PC (Windows/Mac/Linux), WebGL

---

### 1.2 Alternative Stack (Godot/GDScript or C#)

**Engine:** Godot 4.2+
- **Pros:**
  - Free and open-source
  - Lightweight
  - Excellent 2D support
  - Built-in scene system perfect for card game
- **Cons:**
  - Smaller community
  - Fewer ready-made assets

**Language:** GDScript or C#

**UI:** Godot's Control nodes

**Data Format:** JSON or Godot Resources

---

### 1.3 Alternative Stack (Web - Phaser.js)

**Framework:** Phaser 3
- **Pros:**
  - Web-native
  - Easy distribution
  - TypeScript support
- **Cons:**
  - Performance concerns for complex games
  - Less mature tooling for roguelikes

**Language:** TypeScript

**UI:** Phaser UI components + HTML/CSS overlay

**Data Format:** JSON

---

### 1.4 Recommended Choice

**Unity with C#** for the following reasons:
1. Best balance of power and ease of use
2. Excellent card game frameworks available (e.g., Card Game Framework)
3. Strong visual tooling for non-programmers
4. Easy to prototype and iterate
5. Cross-platform with minimal effort
6. Can easily add VFX and juice later

---

## 2. SYSTEM ARCHITECTURE

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────┐
│          PRESENTATION LAYER                 │
│  (UI, Animations, Sound, Input Handling)    │
├─────────────────────────────────────────────┤
│          GAME LOGIC LAYER                   │
│  (State Machine, Combat, Card System)       │
├─────────────────────────────────────────────┤
│          DATA LAYER                         │
│  (Card Definitions, Save Data, Configs)     │
└─────────────────────────────────────────────┘
```

### 2.2 Design Patterns

**State Pattern** - Game state management
```csharp
public abstract class GameState {
    public abstract void Enter();
    public abstract void Update();
    public abstract void Exit();
}
```

**Observer Pattern** - Event system for UI updates
```csharp
public class GameEvents {
    public static event Action<int> OnHealthChanged;
    public static event Action<int> OnGoldChanged;
    public static event Action<Card> OnCardPlayed;
    // etc.
}
```

**Factory Pattern** - Card and enemy creation
```csharp
public class CardFactory {
    public static Card CreateCard(CardData data);
}
```

**Command Pattern** - Undo/redo for card actions (optional)
```csharp
public interface ICommand {
    void Execute();
    void Undo();
}
```

**Singleton Pattern** - Game managers (use sparingly)
```csharp
public class GameManager : MonoBehaviour {
    public static GameManager Instance { get; private set; }
}
```

### 2.3 Core Managers

**GameManager** - Top-level game coordination
- Initializes all systems
- Manages game state transitions
- Handles scene loading

**CombatManager** - Handles combat flow
- Turn structure (GDD 4.3)
- Targeting resolution
- Combat phase execution

**DeckManager** - Deck operations
- Shuffle, draw, discard
- Fatigue mechanic (GDD 2.1)

**BoardManager** - Board state
- 12-slot grid (GDD 4.1)
- Unit placement and removal
- Visual board representation

**EnemyManager** - Enemy AI and behavior
- Intent calculation
- Enemy rotation (GDD 5.3)
- Enemy actions

**UIManager** - UI coordination
- Screen transitions
- Popup management
- HUD updates

**AudioManager** - Sound playback
- SFX pooling
- Music management
- Volume control

**SaveManager** - Persistence
- Save/load runs
- Meta-progression data
- Settings

**EventManager** - Event bus
- Decouples systems
- Centralized event handling

---

## 3. GAME STATE MACHINE

### 3.1 State Diagram

```
[MainMenu]
    ↓
[ClassSelect]
    ↓
[RunStart] → [Map] ⇄ [Combat] → [Victory/Defeat]
                ↓       ↓
              [Shop]  [Reward]
                ↓       ↓
              [Rest]  [Draft]
                ↓       ↓
              [Event]   ↓
                ↓ ←────┘
              [Boss]
                ↓
            [Superboss]
                ↓
           [RunComplete]
                ↓
            [MetaHub]
                ↓
          [MainMenu]
```

### 3.2 State Definitions

**MainMenuState**
- Entry: Load main menu scene, play title music
- UI: Title, New Run, Continue, Settings, Quit
- Transitions:
  - New Run → ClassSelectState
  - Continue → Map/Combat (restore save)
  - Settings → SettingsState
  - Quit → Application.Quit()

**ClassSelectState**
- Entry: Display 12 face cards (GDD 6)
- UI: Class cards with lore, stats, passive/feat preview
- Transitions:
  - Select class → RunStartState
  - Back → MainMenuState

**RunStartState**
- Entry: Initialize run data, create starting deck (GDD 8)
- Logic:
  - Set Leader HP (GDD 2.1)
  - Apply meta-upgrades (GDD 9.5)
  - Generate Level 1 map
- Transitions:
  - Auto → MapState

**MapState**
- Entry: Display current level map
- UI: Node graph with paths (GDD 9.1)
- Node types: Skirmish, Elite, Event, Shop, Rest, Boss
- Transitions:
  - Select Skirmish/Elite → CombatState
  - Select Event → EventState
  - Select Shop → ShopState
  - Select Rest → RestState
  - Select Boss → CombatState (boss variant)

**CombatState**
- Entry: Load combat scene, spawn enemies (GDD 5)
- Sub-states:
  - **CombatStart**: Mulligan (GDD 2.4), initial draw
  - **PlayerTurnStart**: Roll dice (GDD 2.1), draw card
  - **PlayerDeployment**: Play cards, use abilities
  - **PlayerCombat**: Rearguard → Vanguard attacks (GDD 4.3)
  - **EnemyCombat**: Enemies attack (GDD 4.4)
  - **TurnEnd**: Cleanup, check win/loss
- Transitions:
  - Victory → RewardState
  - Defeat → DefeatState

**RewardState**
- Entry: Calculate rewards (gold, card drafts)
- UI: Gold gained, draft pool (GDD 9.4)
- Transitions:
  - Continue → MapState (next level or same level)

**DraftState**
- Entry: Present 5 cards (GDD 9.4)
- UI: Card previews, Skip for 8 Gold
- Transitions:
  - Select card or skip → MapState

**ShopState**
- Entry: Generate shop inventory (GDD 9.3)
- UI: Cards for sale, relics, services
- Transitions:
  - Leave → MapState

**RestState**
- Entry: Present rest options
- UI: Heal Leader, Upgrade Card, Transform Diamond
- Transitions:
  - Select option → MapState

**EventState**
- Entry: Roll dice, present event text (GDD 9.1)
- UI: Event description, choices with dice requirements
- Transitions:
  - Make choice → MapState

**DefeatState**
- Entry: Calculate Renown earned (GDD 9.5)
- UI: Run summary, stats, Renown gained
- Transitions:
  - Continue → MetaHubState

**VictoryState**
- Entry: Calculate Renown, unlock achievements
- UI: Victory screen, stats, Renown
- Transitions:
  - Continue → MetaHubState

**MetaHubState**
- Entry: Display meta-progression (The Forge)
- UI: Renown balance, class upgrades (GDD 9.5)
- Transitions:
  - New Run → ClassSelectState
  - Main Menu → MainMenuState

**SettingsState**
- Entry: Load settings
- UI: Volume, graphics, controls, keybinds
- Transitions:
  - Back → Previous state

---

## 4. DATA STRUCTURES

### 4.1 Card System

**CardData (ScriptableObject)**
```csharp
[CreateAssetMenu(fileName = "Card", menuName = "Guild/Card")]
public class CardData : ScriptableObject {
    public string cardName;
    public CardSuit suit;
    public int rank; // 2-11 (Ace=11)
    public int diceCost;
    public Sprite artwork;
    public string description;

    // Stats
    public int baseHP;
    public int baseAttack;

    // Effects
    public CardEffect vanguardEffect;
    public CardEffect rearguardEffect;
    public CardEffect diamondEffect; // null if not Diamond

    // Tags
    public List<CardTag> tags; // Consumable, Mechanical, Undead, etc.
}

public enum CardSuit {
    Spades,   // Strikers
    Clubs,    // Defenders
    Hearts,   // Menders
    Diamonds  // Spells
}

public enum CardTag {
    Consumable,
    Mechanical,
    Undead,
    CannotRemove,
    Unplayable
}
```

**Card (Runtime Instance)**
```csharp
public class Card {
    public CardData data;
    public int instanceID; // Unique ID for this instance

    // Runtime stats (can differ from base due to upgrades/buffs)
    public int currentHP;
    public int maxHP;
    public int attack;
    public int cost;

    // Status effects
    public List<StatusEffect> statusEffects;

    // Board position (null if in hand/deck/discard)
    public BoardSlot slot;

    // Upgrade level
    public int upgradeLevel; // 0 = base, 1 = upgraded (+2/+2 for units)

    public Card(CardData data) {
        this.data = data;
        this.instanceID = System.Guid.NewGuid().GetHashCode();
        ResetStats();
    }

    public void ResetStats() {
        // Initialize based on suit and rank (GDD 3)
        switch (data.suit) {
            case CardSuit.Spades:
                maxHP = data.rank;
                attack = data.rank; // Dynamic for Vanguard
                break;
            case CardSuit.Clubs:
                maxHP = Mathf.CeilToInt(data.rank * 1.5f);
                attack = data.rank / 2;
                break;
            case CardSuit.Hearts:
                maxHP = data.rank;
                attack = data.rank / 2;
                break;
            case CardSuit.Diamonds:
                // Spells don't have HP/ATK
                break;
        }
        currentHP = maxHP;
        cost = data.diceCost;
    }

    public void TakeDamage(int amount, DamageType type = DamageType.Normal) {
        // Handle armor, shields, true damage, etc.
    }

    public void Heal(int amount) {
        // Handle overheal for Hearts (GDD 3.3)
    }
}

public enum DamageType {
    Normal,
    True // Ignores shields/armor
}
```

### 4.2 Board System

**BoardSlot**
```csharp
public enum SlotType {
    Vanguard, // V1-V5
    Rearguard // R1-R7
}

public class BoardSlot {
    public SlotType type;
    public int index; // 0-4 for Vanguard, 0-6 for Rearguard
    public Card occupyingCard; // null if empty

    public bool IsEmpty => occupyingCard == null;

    // Get slot directly in front (for Rearguard support)
    public BoardSlot GetFrontSlot() {
        if (type == SlotType.Vanguard) return null;

        // Alignment mapping (GDD 4.1)
        switch (index) {
            case 0: return BoardManager.Instance.GetVanguardSlot(0); // R1→V1
            case 1: return BoardManager.Instance.GetVanguardSlot(0); // R2→V1
            case 2: return BoardManager.Instance.GetVanguardSlot(1); // R3→V2
            case 3: return BoardManager.Instance.GetVanguardSlot(2); // R4→V3
            case 4: return BoardManager.Instance.GetVanguardSlot(3); // R5→V4
            case 5: return BoardManager.Instance.GetVanguardSlot(4); // R6→V5
            case 6: return BoardManager.Instance.GetVanguardSlot(4); // R7→V5
            default: return null;
        }
    }
}
```

**BoardState**
```csharp
public class BoardState {
    public BoardSlot[] vanguardSlots = new BoardSlot[5];
    public BoardSlot[] rearguardSlots = new BoardSlot[7];

    public void Initialize() {
        for (int i = 0; i < 5; i++) {
            vanguardSlots[i] = new BoardSlot { type = SlotType.Vanguard, index = i };
        }
        for (int i = 0; i < 7; i++) {
            rearguardSlots[i] = new BoardSlot { type = SlotType.Rearguard, index = i };
        }
    }

    public bool PlayCard(Card card, BoardSlot slot) {
        if (slot.occupyingCard != null) {
            // Retire existing unit (GDD 4.2)
            RetireCard(slot.occupyingCard);
        }
        slot.occupyingCard = card;
        card.slot = slot;
        return true;
    }

    public void RetireCard(Card card) {
        // Move to discard, no "On Death" triggers
        card.slot.occupyingCard = null;
        card.slot = null;
        DeckManager.Instance.MoveToDiscard(card);
    }

    public List<Card> GetAllUnits() {
        List<Card> units = new List<Card>();
        foreach (var slot in vanguardSlots) {
            if (!slot.IsEmpty) units.Add(slot.occupyingCard);
        }
        foreach (var slot in rearguardSlots) {
            if (!slot.IsEmpty) units.Add(slot.occupyingCard);
        }
        return units;
    }
}
```

### 4.3 Enemy System

**EnemyData (ScriptableObject)**
```csharp
[CreateAssetMenu(fileName = "Enemy", menuName = "Guild/Enemy")]
public class EnemyData : ScriptableObject {
    public string enemyName;
    public EnemyType type;
    public Sprite sprite;

    public int baseHP;
    public int baseAttack;

    public List<EnemyIntent> possibleIntents;
    public EnemyAI aiScript;
}

public enum EnemyType {
    Brute,      // Frontline tank
    Striker,    // Frontline DPS
    Elite,      // Frontline special
    Archer,     // Backline ranged
    Healer,     // Backline support
    Buffer,     // Backline support
    Summoner,   // Backline special
    Boss        // Unique
}

public enum IntentType {
    Attack,     // Sword icon
    Defend,     // Shield icon
    Buff,       // Sparkles icon
    Heavy       // Skull icon
}
```

**Enemy (Runtime)**
```csharp
public class Enemy {
    public EnemyData data;
    public int currentHP;
    public int maxHP;
    public int attack;

    public List<StatusEffect> statusEffects;
    public EnemyIntent currentIntent;
    public int position; // Position in enemy formation
    public bool isFrontline;

    public Enemy(EnemyData data) {
        this.data = data;
        this.maxHP = data.baseHP;
        this.currentHP = maxHP;
        this.attack = data.baseAttack;
    }

    public void CalculateIntent() {
        // AI determines next action (GDD 5.1)
        currentIntent = data.aiScript.GetNextIntent(this);
    }

    public void ExecuteIntent(BoardState playerBoard) {
        // Perform the action based on intent
        switch (currentIntent.type) {
            case IntentType.Attack:
                AttackPlayer(playerBoard);
                break;
            // etc.
        }
    }
}
```

### 4.4 Status Effects

**StatusEffect**
```csharp
public abstract class StatusEffect {
    public string name;
    public int stacks;
    public Sprite icon;

    public abstract void OnTurnStart(Card card);
    public abstract void OnTurnEnd(Card card);
    public abstract void OnAttack(Card card);
}

// Examples (GDD 11)
public class PoisonEffect : StatusEffect {
    public override void OnTurnEnd(Card card) {
        card.TakeDamage(stacks, DamageType.True);
        // Poison doesn't decrease (GDD 11)
    }
}

public class BurnEffect : StatusEffect {
    public override void OnTurnStart(Card card) {
        card.TakeDamage(stacks, DamageType.Normal);
        stacks--; // Burn decreases (GDD 11)
        if (stacks <= 0) {
            card.statusEffects.Remove(this);
        }
    }
}

public class RegenerationEffect : StatusEffect {
    public int duration;

    public override void OnTurnStart(Card card) {
        card.Heal(stacks);
        duration--;
        if (duration <= 0) {
            card.statusEffects.Remove(this);
        }
    }
}
```

### 4.5 Run Data

**RunData**
```csharp
[System.Serializable]
public class RunData {
    // Leader
    public LeaderData leader; // Which face card
    public int currentHP;
    public int maxHP;

    // Deck
    public List<int> deckCardIDs; // References to CardData
    public List<int> upgradeStatus; // 0 = not upgraded, 1 = upgraded

    // Progression
    public int currentLevel; // 1-7
    public int gold;
    public List<int> relicIDs;

    // Meta
    public DiceConfig diceConfig; // 2d6, 3d6, etc. (GDD 12.1)

    // Save
    public string runID;
    public System.DateTime lastSaved;

    public RunData(LeaderData leader) {
        this.leader = leader;
        this.currentHP = leader.startingHP;
        this.maxHP = leader.startingHP;
        this.currentLevel = 1;
        this.gold = leader.startingGold; // 0 normally, 50 for Merchant Prince
        this.runID = System.Guid.NewGuid().ToString();
    }
}
```

### 4.6 Meta Progression

**MetaData**
```csharp
[System.Serializable]
public class MetaData {
    public int renown; // Currency (GDD 9.5)

    // Per-class upgrades
    public Dictionary<string, ClassUpgrades> classUpgrades;

    // Unlocks
    public List<string> unlockedClasses; // Start with all 12
    public List<string> unlockedCards;
    public List<string> unlockedRelics;

    // Stats
    public int totalRuns;
    public int victories;
    public Dictionary<string, int> classVictories;
}

[System.Serializable]
public class ClassUpgrades {
    public int hpRanks; // 0-5, +2 HP each (GDD 9.5)
    public int goldRanks;
    public DiceConfig diceConfig;
    public List<string> veterancyUnlocks;
    public Dictionary<string, int> featUpgrades;
}
```

---

## 5. CORE SYSTEMS

### 5.1 Combat System

**CombatManager Responsibilities:**
- Execute turn structure (GDD 4.3)
- Handle combat phases
- Check win/loss conditions
- Apply fatigue at round 15 (GDD 4.5)

**Turn Flow Implementation:**
```csharp
public class CombatManager : MonoBehaviour {
    public BoardState playerBoard;
    public List<Enemy> enemies;
    public int currentTurn;

    public void StartTurn() {
        currentTurn++;

        // 1. Start Phase (GDD 4.3)
        RollDice();
        DrawCard();
        TriggerStartOfTurnEffects();

        // 2. Deployment Phase (player input, handled by UI)
        StartCoroutine(WaitForPlayerDeployment());
    }

    IEnumerator WaitForPlayerDeployment() {
        // Wait for player to finish playing cards
        yield return new WaitUntil(() => playerHasEndedTurn);

        // 3. Player Combat Phase
        ExecutePlayerCombat();

        // 4. Enemy Combat Phase
        ExecuteEnemyCombat();

        // 5. End Phase
        ExecuteTurnEnd();

        // Check win/loss
        if (CheckVictory()) {
            EndCombat(true);
        } else if (CheckDefeat()) {
            EndCombat(false);
        } else {
            StartTurn(); // Next turn
        }
    }

    void ExecutePlayerCombat() {
        // Rearguard acts first (GDD 4.3)
        foreach (var slot in playerBoard.rearguardSlots) {
            if (!slot.IsEmpty) {
                ExecuteRearguardAction(slot.occupyingCard);
            }
        }

        // Then Vanguard attacks
        foreach (var slot in playerBoard.vanguardSlots) {
            if (!slot.IsEmpty) {
                ExecuteVanguardAttack(slot.occupyingCard);
            }
        }
    }

    void ExecuteRearguardAction(Card card) {
        switch (card.data.suit) {
            case CardSuit.Spades:
                // Ranged attack: Target lowest HP enemy (GDD 3.1)
                Enemy target = GetLowestHPEnemy();
                if (target != null) {
                    int damage = card.data.rank <= 5 ? 2 : (card.data.rank <= 10 ? 4 : 6);
                    target.TakeDamage(damage);
                }
                break;

            case CardSuit.Clubs:
                // Shield: Grant to unit in front (GDD 3.2)
                BoardSlot frontSlot = card.slot.GetFrontSlot();
                if (frontSlot != null && !frontSlot.IsEmpty) {
                    int shieldAmount = card.data.rank / 2;
                    frontSlot.occupyingCard.GainShield(shieldAmount);
                }
                break;

            case CardSuit.Hearts:
                // Heal: Heal unit in front (GDD 3.3)
                BoardSlot frontSlot2 = card.slot.GetFrontSlot();
                if (frontSlot2 != null && !frontSlot2.IsEmpty) {
                    int healAmount = card.data.rank / 2;
                    frontSlot2.occupyingCard.Heal(healAmount);
                }
                break;
        }
    }

    void ExecuteVanguardAttack(Card card) {
        // Attack enemy directly across (GDD 4.4)
        Enemy target = GetEnemyAcrossFrom(card.slot.index);
        if (target == null) {
            // Cross-lane targeting (GDD 4.4)
            target = GetNearestEnemy(card.slot.index);
        }

        if (target != null) {
            int damage = card.attack;
            if (card.data.suit == CardSuit.Spades) {
                damage = card.currentHP; // Spades deal damage = current HP
            }
            target.TakeDamage(damage, card);
        }
    }

    void ExecuteEnemyCombat() {
        // Enemies activate left to right (GDD 4.3)
        foreach (var enemy in enemies.OrderBy(e => e.position)) {
            if (enemy.currentHP > 0) {
                enemy.ExecuteIntent(playerBoard);
            }
        }
    }

    void ExecuteTurnEnd() {
        // Cleanup shields (GDD 4.3)
        foreach (var card in playerBoard.GetAllUnits()) {
            card.RemoveExpiredShields();
        }

        // Trigger end-of-turn effects
        TriggerEndOfTurnEffects();

        // Battle Fatigue (GDD 4.5)
        if (currentTurn >= 15) {
            int damage = currentTurn - 14; // 1, 2, 3, 4...
            ApplyFatigueDamage(damage);
        }
    }
}
```

### 5.2 Deck System

**DeckManager**
```csharp
public class DeckManager : MonoBehaviour {
    public List<Card> deck = new List<Card>();
    public List<Card> hand = new List<Card>();
    public List<Card> discard = new List<Card>();

    private int fatigueCount = 0;

    public void Shuffle() {
        // Fisher-Yates shuffle
        for (int i = deck.Count - 1; i > 0; i--) {
            int j = Random.Range(0, i + 1);
            Card temp = deck[i];
            deck[i] = deck[j];
            deck[j] = temp;
        }
    }

    public Card DrawCard() {
        if (deck.Count == 0) {
            // Fatigue: Shuffle discard, then remove 1 card (GDD 2.1)
            if (discard.Count == 0) {
                Debug.LogWarning("Deck and discard both empty!");
                return null;
            }

            deck.AddRange(discard);
            discard.Clear();
            Shuffle();

            // Remove random card permanently
            if (deck.Count > 0) {
                int removeIndex = Random.Range(0, deck.Count);
                Card removed = deck[removeIndex];
                deck.RemoveAt(removeIndex);
                fatigueCount++;
                GameEvents.OnFatigue?.Invoke(removed);
            }
        }

        if (deck.Count == 0) return null;

        Card card = deck[0];
        deck.RemoveAt(0);
        hand.Add(card);
        GameEvents.OnCardDrawn?.Invoke(card);
        return card;
    }

    public void MoveToDiscard(Card card) {
        hand.Remove(card);
        discard.Add(card);
        GameEvents.OnCardDiscarded?.Invoke(card);
    }

    public void Mulligan(List<Card> cardsToMulligan) {
        foreach (var card in cardsToMulligan) {
            hand.Remove(card);
            deck.Add(card);
        }
        Shuffle();

        // Redraw same number
        for (int i = 0; i < cardsToMulligan.Count; i++) {
            DrawCard();
        }
    }
}
```

### 5.3 Dice System

**DiceManager**
```csharp
public class DiceManager : MonoBehaviour {
    public DiceConfig config; // 2d6, 3d6, etc.
    public int currentDice;
    public int reservedDice; // Max 2 (GDD 2.4)

    public void RollDice() {
        int total = reservedDice; // Start with reserved
        reservedDice = 0;

        // Roll based on config (GDD 12.1)
        foreach (var die in config.dice) {
            total += Random.Range(1, die + 1);
        }

        currentDice = total;
        GameEvents.OnDiceRolled?.Invoke(currentDice);
    }

    public bool SpendDice(int amount) {
        if (currentDice >= amount) {
            currentDice -= amount;
            GameEvents.OnDiceSpent?.Invoke(amount);
            return true;
        }
        return false;
    }

    public void ReserveDice(int amount) {
        if (currentDice >= amount && reservedDice + amount <= 2) {
            currentDice -= amount;
            reservedDice += amount;
            GameEvents.OnDiceReserved?.Invoke(amount);
        }
    }

    public void Exertion(Card card) {
        // Discard card, gain half its cost (GDD 2.4)
        int diceGain = card.cost / 2;
        currentDice += diceGain;
        DeckManager.Instance.MoveToDiscard(card);
        GameEvents.OnExertion?.Invoke(card, diceGain);
    }
}

[System.Serializable]
public class DiceConfig {
    public List<int> dice; // e.g., [6, 6] for 2d6, [6, 6, 4] for 2d6+1d4

    public string GetDisplayName() {
        var grouped = dice.GroupBy(d => d);
        return string.Join(" + ", grouped.Select(g => $"{g.Count()}d{g.Key}"));
    }
}
```

### 5.4 Targeting System

**TargetSelector**
```csharp
public class TargetSelector : MonoBehaviour {
    public enum TargetType {
        EnemyUnit,
        FriendlyUnit,
        EnemyAll,
        FriendlyAll,
        BoardSlot,
        None
    }

    public static Enemy GetLowestHPEnemy() {
        var enemies = CombatManager.Instance.enemies
            .Where(e => e.currentHP > 0)
            .OrderBy(e => e.currentHP);
        return enemies.FirstOrDefault();
    }

    public static Enemy GetEnemyAtPosition(int position) {
        return CombatManager.Instance.enemies
            .FirstOrDefault(e => e.position == position && e.isFrontline);
    }

    public static Enemy GetNearestEnemy(int fromVanguardSlot) {
        // Cross-lane targeting (GDD 4.4)
        // Priority: Left → Right → Backline
        var frontline = CombatManager.Instance.enemies.Where(e => e.isFrontline).ToList();

        // Try left
        for (int i = fromVanguardSlot - 1; i >= 0; i--) {
            var enemy = frontline.FirstOrDefault(e => e.position == i);
            if (enemy != null) return enemy;
        }

        // Try right
        for (int i = fromVanguardSlot + 1; i < 5; i++) {
            var enemy = frontline.FirstOrDefault(e => e.position == i);
            if (enemy != null) return enemy;
        }

        // Try backline
        return CombatManager.Instance.enemies
            .FirstOrDefault(e => !e.isFrontline && e.currentHP > 0);
    }
}
```

---

## 6. UI/UX SPECIFICATIONS

### 6.1 Screen List

1. **Main Menu**
2. **Class Select**
3. **Map Screen**
4. **Combat Screen** (Primary gameplay)
5. **Reward Screen**
6. **Draft Screen**
7. **Shop Screen**
8. **Rest Screen**
9. **Event Screen**
10. **Defeat Screen**
11. **Victory Screen**
12. **Meta Hub (The Forge)**
13. **Settings Screen**
14. **Collection/Codex** (Optional)

### 6.2 Combat Screen (Detailed)

**Layout:**
```
┌─────────────────────────────────────────────────────────────┐
│ HP: 25/30      Gold: 45      Turn: 5      [Settings] [Quit] │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│              ENEMY FORMATION                                  │
│         [Enemy1] [Enemy2] [Enemy3]                           │
│         Intent   Intent   Intent                             │
│            ↓        ↓        ↓                               │
├─────────────────────────────────────────────────────────────┤
│              PLAYER BOARD                                     │
│                                                               │
│  VANGUARD: [Slot] [Slot] [Slot] [Slot] [Slot]               │
│             V1     V2     V3     V4     V5                   │
│                                                               │
│  REARGUARD: [Slot] [Slot] [Slot] [Slot] [Slot] [Slot] [Slot]│
│              R1     R2     R3     R4     R5     R6     R7    │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│              HAND (Cards)                                     │
│   [Card] [Card] [Card] [Card] [Card]                        │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│  Dice: ⚀⚁⚂⚃⚄⚅ = 21      Reserved: ⚁⚁ (2)                   │
│  [Punch (1)] [Guard (1)] [Cycle (2)] [End Turn]             │
│  Deck: 12   Discard: 8                                       │
└─────────────────────────────────────────────────────────────┘
```

**UI Elements:**

**Header Bar:**
- Leader HP bar (animated)
- Gold counter
- Turn counter
- Settings/Pause button
- Quit/Forfeit button

**Enemy Area:**
- Enemy sprites with HP bars
- Intent icons above each enemy
- Damage/heal numbers (floating text)
- Status effect icons

**Player Board:**
- 12 slot zones (clickable for placement)
- Card visuals in slots with HP/ATK
- Status effect icons on cards
- Highlight on hover
- Glow effect for valid targets

**Hand Area:**
- Card fan layout at bottom
- Drag-and-drop to play
- Hover to enlarge (preview)
- Cost indicator (dice icons)
- Grayed out if not enough dice

**Dice Pool:**
- Visual dice showing roll results
- Total dice number
- Reserved dice counter
- Leader action buttons (Punch, Guard, Cycle)
- End Turn button (prominent)

**Deck/Discard Indicators:**
- Small icons showing counts
- Click to view full list (popup)

**Animations:**
- Card draw (slide from deck)
- Card play (fly to slot)
- Attack (unit lunges, projectile for ranged)
- Damage (shake, red flash, numbers)
- Heal (green particles, numbers)
- Death (fade out, move to discard)
- Dice roll (3D dice tumble)

### 6.3 Map Screen

**Layout:**
```
Level 3/7 - The Cursed Wastes

     [Boss]
       ↑
    ┌──┴──┐
  [?]    [Shop]
    ↑      ↑
  ┌─┴─┐  ┌─┴─┐
 [E] [S] [R] [S]
    ↑  ╲ ╱  ↑
    └────●────┘
      Start

Legend:
S = Skirmish
E = Elite
? = Event
R = Rest
$ = Shop
```

**UI Elements:**
- Node graph with paths
- Current position marker
- Available next nodes (highlighted)
- Node type icons
- Level name/number
- Progress bar (nodes completed)

### 6.4 Reward Screen

**Layout:**
```
┌─────────────────────────────────────────────┐
│             VICTORY!                        │
│                                             │
│  Gold Earned: +12                           │
│  Total Gold: 57                             │
│                                             │
│  Choose 1 Card to Add:                      │
│  [Card1] [Card2] [Card3] [Card4] [Card5]   │
│                                             │
│  [Skip for 8 Gold]      [Continue]         │
└─────────────────────────────────────────────┘
```

### 6.5 Shop Screen

**Layout:**
```
┌─────────────────────────────────────────────┐
│  THE MERCHANT'S TENT           Gold: 57     │
├─────────────────────────────────────────────┤
│  CARDS                                      │
│  [4♠ - 15g] [7♣ - 25g] [3♦ - 20g]          │
│                                             │
│  RELICS                                     │
│  [Whetstone - 45g] [Pocket Watch - 60g]    │
│                                             │
│  SERVICES                                   │
│  [Upgrade Card - 30g]                       │
│  [Remove Card - 25g]                        │
│  [Transform Card - 50g]                     │
│                                             │
│  [Leave Shop]                               │
└─────────────────────────────────────────────┘
```

### 6.6 Meta Hub (The Forge)

**Layout:**
```
┌─────────────────────────────────────────────┐
│  THE FORGE          Renown: 450             │
├─────────────────────────────────────────────┤
│  SELECT CLASS TO UPGRADE:                   │
│  [J♠] [Q♠] [K♠] [J♣] [Q♣] [K♣]            │
│  [J♥] [Q♥] [K♥] [J♦] [Q♦] [K♦]            │
│                                             │
│  ADVENTURER (Jack of Spades)                │
│  ├─ HP Rank: ●●○○○ (+4 HP) [Upgrade: 50R]  │
│  ├─ Dice: 2d6 → 2d6+1d4 [Unlock: 100R]     │
│  ├─ Feat: "My Friends" (base) [Upgrade: 75R]│
│  └─ Veterancy: [Locked] [Unlock: 200R]     │
│                                             │
│  [New Run]  [Back to Menu]                 │
└─────────────────────────────────────────────┘
```

### 6.7 Card Visual Design

**Card Template:**
```
┌───────────────┐
│ ♠ 5          │  ← Suit/Rank
├───────────────┤
│               │
│   [Artwork]   │  ← Card art
│               │
├───────────────┤
│ Striker       │  ← Type
│ 5 HP / 5 ATK  │  ← Stats
├───────────────┤
│ Vanguard:     │
│ Melee ATK     │  ← Effect text
│               │
│ Rearguard:    │
│ Ranged (2 dmg)│
└───────────────┘
  Cost: ⚂⚂⚂⚂⚂ (5)
```

**Card States:**
- **Default:** Neutral color
- **Hover:** Slight lift, glow
- **Playable:** Green glow
- **Not enough dice:** Red tint, grayed out
- **Selected:** Blue glow
- **In slot:** Show HP/ATK, status icons

---

## 7. GRAPHICS PIPELINE

### 7.1 Art Style Recommendations

**Option 1: Pixel Art**
- Pros: Faster to produce, nostalgic, scales well
- Cons: May feel less premium
- Resolution: 32x32 or 64x64 for cards
- Examples: Slay the Spire (pixel alternative)

**Option 2: Hand-Drawn 2D**
- Pros: Unique, characterful, professional
- Cons: More expensive, requires artist
- Resolution: 512x512 for cards (downscale as needed)
- Examples: Griftlands, Monster Train

**Option 3: Low-Poly 3D**
- Pros: Can reuse models, animated
- Cons: Requires 3D skills
- Examples: Dicey Dungeons

**Recommended:** **Hand-Drawn 2D** for cards, **Pixel Art** for UI/VFX

### 7.2 Asset List

**Cards (12 suits × 10 ranks = 120+ unique):**
- 120 unit card artworks (Spades, Clubs, Hearts 2-Ace)
- 120 diamond spell icons
- 12 face card leader portraits
- 16+ Joker card artworks
- Card frame templates (by suit)
- Upgrade indicator (star, glow, etc.)

**Enemies (50+ types across 7 levels):**
- Frontline enemies: Brutes, Strikers, Elites
- Backline enemies: Archers, Healers, Buffers, Summoners
- 7 Level Bosses
- 4 Superbosses
- Enemy intent icons (Sword, Shield, Sparkles, Skull)

**UI Elements:**
- Background for each screen (Main Menu, Combat, Map, etc.)
- Button templates (normal, hover, pressed, disabled)
- Icon set (health, gold, dice, settings, etc.)
- Status effect icons (Poison, Burn, Armor, Shield, etc.)
- Relic icons (40+ relics)
- Node icons for map (Skirmish, Elite, Event, Shop, Rest, Boss)

**Board:**
- Slot frame graphics (Vanguard vs Rearguard distinction)
- Board background
- Slot highlights (hover, valid target, invalid)

**VFX Sprites:**
- Damage numbers (pop-up text)
- Heal numbers (green text)
- Hit effects (slash, impact, blood)
- Spell effects (fire, ice, holy, dark, etc.)
- Buff/debuff particles
- Death animations (fade, explosion, etc.)

**Misc:**
- Dice (6-sided, 4-sided, 12-sided)
- Card back design
- Loading screens
- Transition effects

### 7.3 Asset Production Pipeline

**Phase 1: Placeholders**
- Use solid color rectangles with text
- Basic icons from free icon packs (Noun Project, Font Awesome)
- Focus on functionality

**Phase 2: Prototype Art**
- Commission or create simple art for 12 classes
- Basic enemy sprites (reuse/recolor)
- Core UI elements

**Phase 3: Production Art**
- Full card artwork for all 120+ cards
- Unique enemy designs
- Polished UI
- VFX and particle effects

**Phase 4: Polish**
- Animations (card draw, attacks, etc.)
- Screen transitions
- Juice (screen shake, slow-mo on kills, etc.)

### 7.4 Animation Requirements

**Essential:**
- Card draw (0.3s slide)
- Card play (0.5s fly to slot)
- Attack (0.4s lunge/projectile)
- Damage number (1s fade up)
- Death (0.6s fade)

**Nice-to-Have:**
- Idle animations for units
- Enemy intent reveal
- Dice roll animation
- Victory/defeat sequences
- Relic acquisition

**Tools:**
- Unity Animator for sprite animations
- DOTween for UI tweens
- Particle System for VFX

---

## 8. AUDIO SYSTEM

### 8.1 Music Tracks

**Required Tracks:**
1. **Main Menu Theme** (1-2 min loop, orchestral, hopeful)
2. **Map/Exploration Theme** (2 min loop, ambient, tense)
3. **Combat Theme - Standard** (2 min loop, fast tempo, action)
4. **Combat Theme - Boss** (2 min loop, epic, intense)
5. **Combat Theme - Superboss** (2.5 min loop, desperate, climactic)
6. **Shop Theme** (1.5 min loop, quirky, light)
7. **Victory Fanfare** (15s, triumphant)
8. **Defeat Theme** (20s, somber)
9. **Meta Hub Theme** (2 min loop, contemplative)

**Optional:**
- Class-specific themes for leader selection
- Level-specific variations

**Style:** Orchestral with some electronic elements, fantasy RPG

**Tools:**
- License royalty-free music (Epidemic Sound, Artlist)
- Commission composer (Fiverr, SoundBetter)
- Use free music (CC0) for prototype

### 8.2 Sound Effects

**UI Sounds:**
- Button hover
- Button click
- Card hover
- Card select/deselect
- Page turn
- Coin drop (gold gain)
- Error/invalid action
- Purchase success
- Draft card appear

**Combat Sounds:**
- Dice roll (tumbling)
- Card draw (whoosh)
- Card play (thump)
- Attack - Melee (slash, thud)
- Attack - Ranged (whoosh, impact)
- Damage taken (grunt, hit)
- Heal (chime, sparkle)
- Shield gain (ding)
- Status effect apply (various)
- Unit death (fade, collapse)
- Enemy intent reveal (sting)
- Turn start (bell)
- Victory (fanfare)
- Defeat (low horn)

**Spell Sounds (Diamond Cards):**
- Generic cast (whoosh)
- Fire spell (fireball)
- Ice spell (freeze)
- Holy spell (chime)
- Dark spell (ominous)
- Buff (sparkle)
- Debuff (curse)

**Total SFX:** ~50-70 effects

**Sources:**
- Freesound.org (CC0)
- Unity Asset Store (SFX packs)
- Commission sound designer

### 8.3 AudioManager Implementation

```csharp
public class AudioManager : MonoBehaviour {
    public static AudioManager Instance;

    [Header("Music")]
    public AudioSource musicSource;
    public List<AudioClip> musicTracks;

    [Header("SFX")]
    public AudioSource sfxSource;
    public Dictionary<string, AudioClip> sfxClips;

    [Header("Settings")]
    public float musicVolume = 0.7f;
    public float sfxVolume = 1.0f;

    public void PlayMusic(string trackName, bool loop = true) {
        var clip = musicTracks.Find(c => c.name == trackName);
        if (clip != null) {
            musicSource.clip = clip;
            musicSource.loop = loop;
            musicSource.volume = musicVolume;
            musicSource.Play();
        }
    }

    public void PlaySFX(string sfxName) {
        if (sfxClips.ContainsKey(sfxName)) {
            sfxSource.PlayOneShot(sfxClips[sfxName], sfxVolume);
        }
    }

    public void FadeMusicTo(string trackName, float duration = 1f) {
        StartCoroutine(FadeMusicCoroutine(trackName, duration));
    }

    IEnumerator FadeMusicCoroutine(string trackName, float duration) {
        // Fade out current
        float startVol = musicSource.volume;
        for (float t = 0; t < duration / 2; t += Time.deltaTime) {
            musicSource.volume = Mathf.Lerp(startVol, 0, t / (duration / 2));
            yield return null;
        }

        // Switch track
        PlayMusic(trackName);

        // Fade in new
        for (float t = 0; t < duration / 2; t += Time.deltaTime) {
            musicSource.volume = Mathf.Lerp(0, musicVolume, t / (duration / 2));
            yield return null;
        }
    }
}
```

---

## 9. SAVE SYSTEM

### 9.1 Save Data Structure

**Files:**
- `meta.json` - Meta-progression (Renown, unlocks, class upgrades)
- `run_[runID].json` - Current run data (if any)
- `settings.json` - User settings

**Save Locations:**
- Windows: `%APPDATA%/GuildBuilder/`
- Mac: `~/Library/Application Support/GuildBuilder/`
- Linux: `~/.local/share/GuildBuilder/`

### 9.2 SaveManager Implementation

```csharp
public class SaveManager : MonoBehaviour {
    private string savePath;

    void Awake() {
        savePath = Application.persistentDataPath + "/Saves/";
        if (!Directory.Exists(savePath)) {
            Directory.CreateDirectory(savePath);
        }
    }

    public void SaveRun(RunData run) {
        string json = JsonUtility.ToJson(run, true);
        string path = savePath + $"run_{run.runID}.json";
        File.WriteAllText(path, json);
    }

    public RunData LoadRun(string runID) {
        string path = savePath + $"run_{runID}.json";
        if (File.Exists(path)) {
            string json = File.ReadAllText(path);
            return JsonUtility.FromJson<RunData>(json);
        }
        return null;
    }

    public void DeleteRun(string runID) {
        string path = savePath + $"run_{runID}.json";
        if (File.Exists(path)) {
            File.Delete(path);
        }
    }

    public void SaveMeta(MetaData meta) {
        string json = JsonUtility.ToJson(meta, true);
        File.WriteAllText(savePath + "meta.json", json);
    }

    public MetaData LoadMeta() {
        string path = savePath + "meta.json";
        if (File.Exists(path)) {
            string json = File.ReadAllText(path);
            return JsonUtility.FromJson<MetaData>(json);
        }
        return new MetaData(); // New meta data
    }
}
```

### 9.3 Auto-Save Strategy

**When to Save:**
- After each combat
- After draft/shop/event
- When entering map screen
- Every 5 minutes during gameplay (background)
- On application quit

**Don't Save During:**
- Active combat (mid-turn)
- Animations

---

## 10. IMPLEMENTATION ROADMAP

### 10.1 Milestone 1: Core Combat Prototype (4-6 weeks)

**Goal:** Playable combat with 1 class, basic enemies

**Tasks:**
- [ ] Project setup (Unity, Git, folder structure)
- [ ] Data structures (Card, Enemy, BoardSlot)
- [ ] Card database (20 cards: 10 Spades, 5 Clubs, 5 Hearts for Adventurer)
- [ ] Board system (12 slots, visual grid)
- [ ] Hand system (draw, play, discard)
- [ ] Dice system (2d6 roll, spend, reserve)
- [ ] Basic combat flow (turn structure)
- [ ] Player actions (Vanguard/Rearguard attacks, Leader actions)
- [ ] Enemy AI (simple: attack lowest HP unit)
- [ ] Damage/heal calculations
- [ ] Win/loss conditions
- [ ] Placeholder UI (all text/buttons)
- [ ] 1 combat encounter (3 basic enemies)

**Deliverable:** Can fight 1 combat from start to finish

---

### 10.2 Milestone 2: Full Combat System (4-6 weeks)

**Goal:** Complete combat with all mechanics

**Tasks:**
- [ ] All 4 suits functional (Spades, Clubs, Hearts, Diamonds)
- [ ] Diamond cards (10 spells for Adventurer)
- [ ] Status effects (Poison, Burn, Armor, Shield, Regen, Bleed)
- [ ] Advanced targeting (cross-lane, lowest HP, etc.)
- [ ] Enemy formations (frontline/backline)
- [ ] Enemy rotation mechanic
- [ ] Intent system (visual icons)
- [ ] Breach mechanic (empty Vanguard → Rearguard damage)
- [ ] Battle Fatigue (round 15+)
- [ ] Mulligan
- [ ] Improved UI (card visuals, HP bars, animations)
- [ ] 5 enemy types, 3 encounter variants

**Deliverable:** Fully functional combat with variety

---

### 10.3 Milestone 3: Run Structure (3-4 weeks)

**Goal:** Complete one run from start to finish

**Tasks:**
- [ ] Map generation (7 levels, node graph)
- [ ] Map UI (nodes, paths, current position)
- [ ] Reward system (gold, card drafts)
- [ ] Draft screen (5 cards, skip for gold)
- [ ] Shop system (buy cards, relics, services)
- [ ] Rest sites (heal, upgrade, transform)
- [ ] Events (3 event variants, dice-based choices)
- [ ] Boss encounters (1 boss with Twisted Rule)
- [ ] Superboss (1 superboss)
- [ ] Victory/defeat screens
- [ ] Save/load system (basic)

**Deliverable:** Can play full run with 1 class

---

### 10.4 Milestone 4: All Classes (3-4 weeks)

**Goal:** All 12 classes playable

**Tasks:**
- [ ] Card database expansion (all 120 Diamond cards)
- [ ] Starting decks for all 12 classes
- [ ] Class select screen
- [ ] Leader passives implementation
- [ ] Leader feats implementation
- [ ] Class-specific card interactions
- [ ] Balance tuning

**Deliverable:** 12 unique classes, full replayability

---

### 10.5 Milestone 5: Meta-Progression (2-3 weeks)

**Goal:** The Forge, Renown system

**Tasks:**
- [ ] Meta-progression data structures
- [ ] Renown calculation (based on run progress)
- [ ] Meta Hub UI
- [ ] Class upgrades (HP ranks, Dice, Feats, Veterancy)
- [ ] Unlock system
- [ ] Save/load meta data

**Deliverable:** Long-term progression loop

---

### 10.6 Milestone 6: Content & Balance (4-6 weeks)

**Goal:** Full enemy roster, relics, Jokers

**Tasks:**
- [ ] 50+ enemy types (distributed across 7 levels)
- [ ] 40+ relics (Common, Uncommon, Rare, Boss)
- [ ] 16+ Joker cards
- [ ] 7 Level Bosses
- [ ] 4 Superbosses
- [ ] 20+ events
- [ ] Balance pass (card costs, enemy HP/damage, economy)
- [ ] Playtesting

**Deliverable:** Full content, balanced game

---

### 10.7 Milestone 7: Polish & Juice (3-4 weeks)

**Goal:** Professional presentation

**Tasks:**
- [ ] Art pass (replace all placeholders)
- [ ] Animations (attacks, card play, deaths)
- [ ] VFX (particles, screen shake, slow-mo)
- [ ] Sound effects (full suite)
- [ ] Music (all tracks)
- [ ] UI polish (transitions, effects)
- [ ] Tutorial/onboarding
- [ ] Accessibility options (colorblind mode, text size)

**Deliverable:** Launch-ready game

---

### 10.8 Milestone 8: Launch Prep (2-3 weeks)

**Goal:** Release on PC/Web

**Tasks:**
- [ ] Final bug fixing
- [ ] Performance optimization
- [ ] Build for all platforms (Windows, Mac, Linux, WebGL)
- [ ] Store pages (Steam, Itch.io)
- [ ] Marketing materials (trailer, screenshots)
- [ ] Press kit
- [ ] Launch!

**Total Timeline: 25-35 weeks (6-9 months)**

---

## 11. FILE STRUCTURE

```
CardGame/
├── Assets/
│   ├── Art/
│   │   ├── Cards/
│   │   │   ├── Spades/
│   │   │   ├── Clubs/
│   │   │   ├── Hearts/
│   │   │   ├── Diamonds/
│   │   │   ├── Jokers/
│   │   │   └── Leaders/
│   │   ├── Enemies/
│   │   ├── UI/
│   │   ├── VFX/
│   │   └── Backgrounds/
│   ├── Audio/
│   │   ├── Music/
│   │   └── SFX/
│   ├── Data/
│   │   ├── Cards/
│   │   │   └── [CardData ScriptableObjects]
│   │   ├── Enemies/
│   │   │   └── [EnemyData ScriptableObjects]
│   │   ├── Relics/
│   │   ├── Events/
│   │   └── Leaders/
│   ├── Prefabs/
│   │   ├── Cards/
│   │   ├── UI/
│   │   └── Effects/
│   ├── Scenes/
│   │   ├── MainMenu.unity
│   │   ├── ClassSelect.unity
│   │   ├── Combat.unity
│   │   ├── Map.unity
│   │   ├── Shop.unity
│   │   └── MetaHub.unity
│   ├── Scripts/
│   │   ├── Core/
│   │   │   ├── GameManager.cs
│   │   │   ├── GameState.cs
│   │   │   └── GameEvents.cs
│   │   ├── Combat/
│   │   │   ├── CombatManager.cs
│   │   │   ├── BoardManager.cs
│   │   │   ├── DiceManager.cs
│   │   │   ├── TargetSelector.cs
│   │   │   └── TurnController.cs
│   │   ├── Cards/
│   │   │   ├── Card.cs
│   │   │   ├── CardData.cs
│   │   │   ├── CardFactory.cs
│   │   │   ├── DeckManager.cs
│   │   │   └── Effects/
│   │   ├── Enemies/
│   │   │   ├── Enemy.cs
│   │   │   ├── EnemyData.cs
│   │   │   ├── EnemyManager.cs
│   │   │   └── AI/
│   │   ├── UI/
│   │   │   ├── UIManager.cs
│   │   │   ├── CombatUI.cs
│   │   │   ├── MapUI.cs
│   │   │   ├── ShopUI.cs
│   │   │   └── Screens/
│   │   ├── Progression/
│   │   │   ├── RunData.cs
│   │   │   ├── MetaData.cs
│   │   │   └── MetaManager.cs
│   │   ├── Systems/
│   │   │   ├── AudioManager.cs
│   │   │   ├── SaveManager.cs
│   │   │   └── EventManager.cs
│   │   └── Utilities/
│   │       ├── Singleton.cs
│   │       ├── Extensions.cs
│   │       └── Constants.cs
│   └── Settings/
│       └── [UI Toolkit files, Input System]
├── Packages/
├── ProjectSettings/
├── GDD (Game Design Document)
├── TDD.md (This document)
└── README.md
```

---

## 12. TESTING STRATEGY

### 12.1 Unit Testing

**Framework:** Unity Test Framework (NUnit)

**What to Test:**
- Damage calculations
- Card effect resolution
- Dice rolling (statistical distribution)
- Deck shuffling (randomness)
- Targeting logic
- Win/loss conditions
- Save/load integrity

**Example:**
```csharp
[Test]
public void TestSpadesVanguardDamage() {
    Card card = new Card(SpadesCardData);
    card.currentHP = 5;
    Enemy enemy = new Enemy(BasicEnemyData);

    int damage = card.CalculateVanguardDamage();
    Assert.AreEqual(5, damage); // Spades deal damage = current HP
}
```

### 12.2 Integration Testing

**Test Scenarios:**
- Full combat from start to finish
- Complete run (Level 1 → Level 7)
- All 12 classes starting decks
- Save/load mid-combat
- Edge cases (0 HP, 0 cards, 0 dice)

### 12.3 Balance Testing

**Metrics to Track:**
- Average run length (turns, time)
- Win rate per class
- Card pick rates (which cards are chosen in drafts)
- Gold economy (average gold earned/spent)
- Most/least used cards

**Tools:**
- Unity Analytics (optional)
- Custom logging
- Playtesting spreadsheets

### 12.4 Playtesting

**Phases:**
1. **Internal** (developer only) - Milestone 2+
2. **Closed Alpha** (5-10 friends) - Milestone 5
3. **Closed Beta** (50-100 players) - Milestone 6
4. **Open Beta** (public) - Milestone 7

**Feedback Focus:**
- Is combat fun and strategic?
- Are classes balanced?
- Is the difficulty curve fair?
- Is the UI intuitive?
- Are there any bugs?

---

## 13. TECHNICAL RISKS & MITIGATIONS

### 13.1 Performance Risks

**Risk:** Lag during combat with many units/effects
**Mitigation:**
- Object pooling for cards, enemies, VFX
- Limit particle effects
- Optimize draw calls (sprite atlases)
- Profile regularly

**Risk:** Long load times
**Mitigation:**
- Asynchronous scene loading
- Lazy loading of card art
- Compress textures

### 13.2 Complexity Risks

**Risk:** 120 Diamond cards too many to balance
**Mitigation:**
- Start with fewer (30-40 core spells)
- Expand over time post-launch
- Use data-driven design (easy to tweak values)

**Risk:** AI too difficult to program
**Mitigation:**
- Start with simple AI (random actions)
- Gradually add smarter behaviors
- Use behavior trees if needed

### 13.3 Scope Risks

**Risk:** Feature creep
**Mitigation:**
- Stick to GDD
- Use "nice-to-have" vs "must-have" labels
- Cut features if behind schedule

**Risk:** Art production bottleneck
**Mitigation:**
- Use placeholders liberally
- Commission art in batches
- Consider procedural generation for backgrounds

---

## 14. TECHNOLOGY RECOMMENDATIONS

### 14.1 Unity Packages (Optional but Helpful)

**DOTween** - Tweening library for animations
- Free, easy to use
- Essential for UI/card animations

**Odin Inspector** - Better inspector for Unity
- Paid ($55), very helpful for data entry
- Makes ScriptableObjects much easier

**TextMesh Pro** - Better text rendering
- Free (built-in to Unity)
- Essential for crisp UI text

**Rewired** - Input system
- Paid ($45), best for gamepad support
- Alternative: Unity's new Input System (free)

**Easy Save** - Save/load utility
- Paid ($30), saves time
- Alternative: Write your own (not hard)

### 14.2 External Tools

**Trello/Notion** - Task management
**Google Sheets** - Balance spreadsheets
**Figma** - UI mockups
**Aseprite** - Pixel art ($20)
**GIMP/Photoshop** - 2D art
**Audacity** - Audio editing (free)
**FMOD/Wwise** - Advanced audio (overkill for this project)

---

## 15. CONCLUSION

This TDD provides a comprehensive roadmap for implementing "The Guild Builder" as specified in the GDD. The architecture is modular, data-driven, and scalable. The implementation roadmap breaks the project into manageable milestones.

**Next Steps:**
1. Choose technology stack (recommend Unity/C#)
2. Set up project repository
3. Begin Milestone 1 (Core Combat Prototype)
4. Iterate based on playtesting

**Key Success Factors:**
- Stay true to the GDD design
- Prioritize core combat fun over features
- Use placeholders to move fast
- Test early and often
- Don't over-engineer

**Estimated Dev Time:** 6-9 months (solo) or 3-5 months (small team)

---

**Document Version:** 1.0
**Last Updated:** 2025-12-30
**Maintained By:** Development Team
**Reference:** GDD v2.0
