# Tarnish RSPS - Internal Documentation

> **Comprehensive onboarding guide for understanding the Tarnish RuneScape Private Server architecture, subsystems, and implementation details.**

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Getting Started](#getting-started)
3. [Architecture](#architecture)
4. [Server-Client Communication](#server-client-communication)
5. [Core Subsystems](#core-subsystems)
6. [Game Systems](#game-systems)
7. [Development Guide](#development-guide)
8. [Configuration](#configuration)
9. [Database & Persistence](#database--persistence)
10. [Deployment](#deployment)

---

## Project Overview

### What is Tarnish?

Tarnish is a **RuneScape Private Server (RSPS)** - a custom implementation of the Old School RuneScape game server and client. It provides:

- **Full OSRS gameplay** with custom content and modifications
- **Modern architecture** using Netty, Kotlin, and Java 21
- **RuneLite client integration** with all plugins built-in
- **Dual persistence** supporting both file-based and database storage
- **Plugin system** for extensible content development
- **High performance** with parallel processing and optimized I/O

### Technology Stack

**Server (game-server):**
- Java 21 + Kotlin
- Netty 4.2.2 (async networking)
- Gradle build system
- PostgreSQL/MySQL support via HikariCP
- ZGC garbage collector
- Virtual threads for async operations

**Client (game-client):**
- Java 11 (RuneLite compatibility)
- LWJGL for OpenGL rendering
- RuneLite framework integration
- 20+ built-in plugins (117 HD, GPU, etc.)

---

## Getting Started

### Prerequisites

- **JDK 21** - For server (default/primary)
- **JDK 11** - For client
- **Gradle** - Build system (wrapper included)
- **Cache files** - OSRS cache data
- **SwiftFUP** - Cache file server (separate project)

### Directory Structure

```
tarnish/
├── game-server/              # Server application
│   ├── src/main/java/        # Java source code
│   │   └── com/osroyale/     # Main package
│   ├── src/main/kotlin/      # Kotlin source code
│   ├── plugins/              # Plugin system source
│   ├── data/                 # Game data
│   │   ├── cache/            # Cache files
│   │   ├── profile/save/     # Player save files (JSON)
│   │   └── def/              # Definition files
│   ├── settings.toml         # Server configuration
│   └── build.gradle.kts      # Build configuration
│
├── game-client/              # Client application
│   ├── src/main/java/        # Java source code
│   │   ├── com/osroyale/     # Client core
│   │   └── net/runelite/     # RuneLite integration
│   ├── src/main/resources/   # Client resources
│   └── build.gradle.kts      # Build configuration
│
├── gradle/                   # Gradle wrapper
├── .github/workflows/        # CI/CD configuration
├── settings.gradle.kts       # Multi-module setup
└── INTERNAL_DOCS.md          # This file
```

### Building the Project

```bash
# Build everything (server + client)
./gradlew build

# Build only server
./gradlew :game-server:build

# Build only client
./gradlew :game-client:build

# Clean build artifacts
./gradlew clean
```

### Running the Server

```bash
# Via Gradle (recommended for development)
./gradlew :game-server:run

# Via JAR (production)
java -jar game-server/build/libs/game-server-all.jar
```

**Server startup sequence:**
1. Load cache files from `data/cache/`
2. Parse item/NPC definitions
3. Load plugins from `plugins/` package
4. Initialize networking on port `43594` (default)
5. Start game engine with 600ms tick cycle

### Running the Client

```bash
# Via Gradle
./gradlew :game-client:run

# Via JAR
java -jar game-client/build/libs/game-client-all.jar
```

**Client startup:**
1. Initialize RuneLite framework
2. Load built-in plugins
3. Connect to SwiftFUP cache server
4. Display login screen
5. Connect to game server

---

## Architecture

### High-Level Overview

```
┌─────────────────┐         Network          ┌─────────────────┐
│  Game Client    │◄─────────────────────────►│  Game Server    │
│  (RuneLite)     │    Netty + ISAAC Cipher  │  (Netty)        │
└─────────────────┘                           └─────────────────┘
        │                                              │
        │                                              │
        ▼                                              ▼
┌─────────────────┐                           ┌─────────────────┐
│  SwiftFUP       │                           │  Database       │
│  Cache Server   │                           │  PostgreSQL     │
└─────────────────┘                           └─────────────────┘
```

### Module Structure

**Multi-module Gradle project:**

```kotlin
// settings.gradle.kts
include("game-server")
include("game-client")
```

Each module is independently buildable with its own dependencies and configuration.

### Server Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                         Main Thread                           │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              GameThread (600ms tick cycle)             │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │  │
│  │  │ Pre-     │→│ Process  │→│ Post-    │→ Sync       │  │
│  │  │ Update   │  │ World    │  │ Update   │  Clients   │  │
│  │  └──────────┘  └──────────┘  └──────────┘            │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
         ▲                                          │
         │                                          │
         │        Netty Event Loop Threads          ▼
┌────────┴──────────────────────────────────────────────┐
│  LoginExecutorService  │  I/O Workers  │  Encoders    │
└───────────────────────────────────────────────────────┘
```

### Client Architecture

```
┌──────────────────────────────────────────────────────┐
│              RuneLite Client Framework                │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐    │
│  │  Renderer  │  │  Plugin    │  │  Network   │    │
│  │  (GPU/CPU) │  │  Manager   │  │  Handler   │    │
│  └────────────┘  └────────────┘  └────────────┘    │
└──────────────────────────────────────────────────────┘
         │                │                │
         ▼                ▼                ▼
    LWJGL/OpenGL    Built-in Plugins   BufferedConnection
```

### Key Design Patterns

1. **Singleton:** `World.java` - single game world instance
2. **Observer:** Event bus system for plugin notifications
3. **Strategy:** Combat system with pluggable strategies
4. **Factory:** Packet creation, dialogue building
5. **Command:** Packet listener pattern
6. **Template Method:** Skill actions, activity phases

---

## Server-Client Communication

### Connection Flow

#### 1. Handshake Phase

```
Client                                Server
  │                                      │
  ├─────── Handshake (opcode 14) ──────►│
  │                                      │
  │◄───── Server Seed (8 bytes) ────────┤
  │       + Server metadata             │
```

#### 2. Login Phase

```
Client                                Server
  │                                      │
  ├─── Connection Type (16 or 18) ─────►│
  │    NEW_CONNECTION or RECONNECTION    │
  │                                      │
  ├────── Login Payload (RSA) ─────────►│
  │  - Version check                     │
  │  - Username/password                 │
  │  - Client seeds                      │
  │                                      │
  │◄───── LoginResponse Code ───────────┤
  │  2 = Success, 3 = Bad creds, etc.   │
```

#### 3. Game Session

```
Client                                Server
  │                                      │
  │◄═══ ISAAC Cipher Initialized ═══════│
  │                                      │
  ├───── Game Packets (encrypted) ─────►│
  │  - Movement, clicks, commands        │
  │                                      │
  │◄──── Update Packets (encrypted) ────┤
  │  - Player updates, NPC updates       │
  │  - Interface updates, messages       │
```

### Packet System

#### Packet Structure

```
┌──────────┬──────────┬─────────────────┐
│  Opcode  │   Size   │     Payload     │
│ (1 byte) │ (0-2 b)  │   (variable)    │
└──────────┴──────────┴─────────────────┘
```

**Size Types:**
- **FIXED:** Size predetermined (no header)
- **VAR_BYTE:** 1-byte size header (max 255 bytes)
- **VAR_SHORT:** 2-byte size header (max 65535 bytes)

#### Server → Client Packets

Located: `game-server/src/main/java/com/osroyale/net/packet/out/`

Key packets:
- **SendMessage** - Chat messages
- **SendUpdateItems** - Inventory/equipment updates
- **SendPlayerUpdate** - Player position/appearance sync
- **SendNpcUpdate** - NPC position/animation sync
- **SendSkill** - Skill level/XP updates
- **SendInterface** - UI rendering
- **SendPlayerOption** - Right-click player options

#### Client → Server Packets

Located: `game-server/src/main/java/com/osroyale/net/packet/in/`

Key packets:
- **CommandPacketListener** - `::` commands
- **ItemClickEvent** - Item interactions (1st/2nd/3rd click)
- **NpcClickEvent** - NPC interactions
- **ObjectClickEvent** - Object interactions
- **MovementPacket** - Player movement
- **DialoguePacket** - Dialogue option selection

### Network Security

#### ISAAC Cipher

- **Stream cipher** for packet encryption
- Initialized during login with client/server seeds
- Separate cipher instances for encoding vs decoding
- XOR-based operation on packet opcodes

```java
// Server-side initialization (LoginDecoder.java)
isaacEncoder = new IsaacCipher(serverSeed);
isaacDecoder = new IsaacCipher(clientSeed);
```

#### RSA Encryption

- **Login credentials** encrypted with RSA public key
- Server holds private key for decryption
- Modulus and exponent configured in `settings.toml`

```toml
[network]
rsa_modulus = "102353038900255891527..."
rsa_exponent = "53925997225795133719..."
```

#### Rate Limiting

```java
// Connection limits per IP
Config.CONNECTION_LIMIT = 3  // Max simultaneous connections

// Failed login protection
Config.FAILED_LOGIN_ATTEMPTS = 5
Config.FAILED_LOGIN_TIMEOUT = 1  // Minutes

// Idle disconnect
Config.IDLE_TIMEOUT = 30  // Seconds

// Packet throttling
Config.CLIENT_PACKET_THRESHOLD = 30  // Per tick
Config.SERVER_PACKET_THRESHOLD = 1000  // Per tick
```

---

## Core Subsystems

### 1. Game Engine

**Class:** `com.osroyale.game.engine.GameThread`

**Tick Cycle:** 600ms (100 ticks = 1 minute)

**Main Loop:**

```java
while (running) {
    long start = System.currentTimeMillis();

    // 1. Process login queue
    World.get().dequeLogin();

    // 2. Process logout queue
    World.get().dequeLogout();

    // 3. NPC pre-update (AI, movement)
    for (Npc npc : World.get().getNpcs()) {
        npc.preUpdate();
    }

    // 4. Player pre-update (skills, combat, movement)
    for (Player player : World.get().getPlayers()) {
        player.preUpdate();
    }

    // 5. Process world tasks
    World.get().process();

    // 6. Sequence entities (walk queues)
    World.get().sequence();

    // 7. Synchronize clients (send updates)
    synchronizer.synchronize(players);

    // 8. Player post-update (reset flags)
    for (Player player : World.get().getPlayers()) {
        player.postUpdate();
    }

    long elapsed = System.currentTimeMillis() - start;
    Thread.sleep(Math.max(0, TICK_MILLIS - elapsed));
}
```

**Synchronizer Modes:**

- **SequentialClientSynchronizer:** Updates players one-by-one
- **ParallelClientSynchronizer:** Uses parallel streams for multi-core efficiency

Configure in `settings.toml`:
```toml
[server]
parallel_game_engine = false  # true for parallel mode
```

### 2. World Management

**Class:** `com.osroyale.game.world.World`

**Singleton Pattern:**
```java
World world = World.get();
```

**Responsibilities:**
- Player registration/removal
- NPC spawning/management
- Region management for spatial queries
- Task scheduling
- Global messaging
- Save/load game state

**Player Management:**
```java
// Login queue (thread-safe)
ConcurrentLinkedQueue<Player> logins = new ConcurrentLinkedQueue<>();

// Active players (MobList with capacity Config.MAX_PLAYERS)
MobList<Player> players = new MobList<>(Config.MAX_PLAYERS);

// Registration
world.queueLogin(player);  // Add to queue
world.dequeLogin();         // Process in game thread
```

**NPC Management:**
```java
// Active NPCs (MobList with capacity Config.MAX_NPCS)
MobList<Npc> npcs = new MobList<>(Config.MAX_NPCS);

// Spawning
world.register(new Npc(npcId, position));
```

**Task Scheduling:**
```java
// One-time task after delay
world.schedule(new Task(delayTicks) {
    @Override
    protected void execute() {
        // Task logic
    }
});

// Repeating task
world.schedule(new Task(delayTicks, true) {
    @Override
    protected void execute() {
        // Repeating logic
    }
});
```

### 3. Region Management

**Class:** `com.osroyale.game.world.region.RegionManager`

**Purpose:** Spatial partitioning for efficient entity queries

**Regions:**
- Game world divided into 8x8 tile regions
- Each region tracks: Players, NPCs, Objects, Ground items

**Usage:**
```java
// Get region for position
Region region = RegionManager.getRegion(position);

// Get nearby players (within view distance)
Set<Player> localPlayers = region.getPlayers(position, viewDistance);

// Add/remove objects
region.addObject(GameObject);
region.removeObject(GameObject);
```

### 4. Plugin System

**Class:** `com.osroyale.game.plugin.PluginManager`

**Architecture:** Event-driven plugin discovery and loading

**Plugin Discovery:**
```java
// ClassGraph scans for PluginContext subclasses
ClassGraph classGraph = new ClassGraph()
    .enableAllInfo()
    .acceptPackages("plugin");

ScanResult result = classGraph.scan();
List<Class<?>> plugins = result.getSubclasses(PluginContext.class);
```

**Creating a Plugin:**

```java
package plugin.click.button;

import com.osroyale.game.plugin.PluginContext;
import com.osroyale.game.world.entity.mob.player.Player;

public class MyCustomPlugin extends PluginContext {

    @Override
    protected void onClick(Player player, int button) {
        if (button == 12345) {
            player.message("You clicked my custom button!");
        }
    }

    @Override
    protected void firstClickNpc(Player player, NpcClickEvent event) {
        if (event.getNpc().id == 1234) {
            player.message("You clicked my custom NPC!");
            event.cancel();
        }
    }
}
```

**Plugin Events:**
- `onClick(button)` - Interface button clicks
- `firstClickItem/Npc/Object` - First click interactions
- `secondClickItem/Npc/Object` - Second click interactions
- `thirdClickItem/Npc/Object` - Third click interactions
- `itemOnItem/Npc/Object/Player` - Item-on-X interactions
- `handleCommand(parser)` - `::` commands
- `onPickupItem/DropItem` - Ground item interactions
- `onMovement` - Player movement events

**Plugin Loading:**
```
Starter.processParallelStartupTasks()
  → PluginManager.load("plugin")
    → ClassGraph scan
      → Instantiate plugins
        → Subscribe to PlayerDataBus
          → Ready to handle events
```

### 5. Networking

**Bootstrap:** `org.jire.tarnishps.BootstrapFactory`

**Server Initialization:**
```java
ServerBootstrap bootstrap = new ServerBootstrap()
    .group(parentGroup, childGroup)
    .channel(bestChannelClass())  // io_uring, epoll, kqueue, or NIO
    .childHandler(new ServerPipelineInitializer())
    .option(ChannelOption.SO_BACKLOG, 128)
    .childOption(ChannelOption.TCP_NODELAY, true)
    .childOption(ChannelOption.SO_KEEPALIVE, true);
```

**Pipeline Stages:**

```
IdleStateHandler (30s timeout)
  ↓
HAProxyMessageDecoder (if enabled)
  ↓
LoginDecoder → GamePacketDecoder (after login)
  ↓
LoginResponseEncoder → GamePacketEncoder (after login)
  ↓
ChannelHandler (session management)
```

**Session Types:**
- **LoginSession:** Pre-authentication state
- **GameSession:** Post-authentication, linked to Player

**HAProxy Support:**
```toml
[network]
support_haproxy = true  # Enable for load balancing
```

When enabled, the server reads the HAProxy PROXY protocol header to extract the real client IP.

---

## Game Systems

### Combat System

**Location:** `game-server/src/main/java/com/osroyale/game/world/entity/combat/`

#### Combat Flow

```
Player.attack(target)
  ↓
Combat.attack(defender)
  ↓
CombatStrategy.canAttack() ← validation
  ↓
CombatStrategy.getAttackDelay() ← weapon speed
  ↓
CombatStrategy.getHits() ← damage calculation
  ↓
Combat.submit(CombatData) ← queue combat
  ↓
Combat.tick() ← process each tick
  ↓
Hit.apply() ← apply damage with delay
```

#### Combat Strategies

**Base Class:** `CombatStrategy<T extends Mob>`

**Player Strategies:**
- `PlayerMeleeStrategy` - Melee combat
- `PlayerRangedStrategy` - Ranged combat
- `PlayerMagicStrategy` - Magic combat

**Weapon-Specific:**
- `ToxicBlowpipeStrategy` - Blowpipe mechanics
- `ScytheOfViturStrategy` - Scythe multi-hit
- `TridentOfTheSeasStrategy` - Built-in spell

**NPC Strategies:**
- `MeleeStrategy` - Basic melee
- `RangedStrategy` - Basic ranged
- `MagicStrategy` - Basic magic
- `MultiStrategy` - Multiple attack styles

#### Combat Formula

**Accuracy Roll:**
```java
int attackRoll = random(effectiveAttackLevel * (attackBonus + 64));
int defenceRoll = random(effectiveDefenceLevel * (defenceBonus + 64));
boolean hit = attackRoll > defenceRoll;
```

**Max Hit (Melee):**
```java
int effectiveStrength = strengthLevel + strengthBonus + 8;
int maxHit = (effectiveStrength * strengthEquipmentBonus + 320) / 640;
// Apply prayers, void, etc.
```

**Max Hit (Ranged):**
```java
int effectiveRanged = rangedLevel + rangedBonus + 8;
int maxHit = (effectiveRanged * rangedStrengthBonus + 320) / 640;
// Apply prayers, void, etc.
```

**Max Hit (Magic):**
```java
int maxHit = spellBaseDamage;
// Apply magic damage bonus %
maxHit = (int) (maxHit * (1 + magicDamageBonus / 100.0));
```

#### Combat Listeners

**Purpose:** Modify combat calculations based on equipment/prayers

**Examples:**
- `SlayerHelmListener` - 15% bonus on slayer task
- `VoidKnightMeleeListener` - 10% damage + accuracy
- `ElysianListener` - 70% damage reduction chance
- `AttackPrayerListener` - Prayer attack bonuses
- `DefencePrayerListener` - Prayer defence bonuses
- `VengeanceListener` - Reflect damage

**Registration:**
```java
CombatListenerManager.load();  // In Starter.java
```

#### Special Attacks

**Enum:** `CombatSpecial`

**Examples:**
- **Dragon Claws:** 4-hit combo with specific damage distribution
- **Armadyl Godsword:** High damage + damage-based stat drain
- **Dark Bow:** Double arrow shot
- **Granite Maul:** Instant attack (no delay)

**Implementation:**
```java
public enum CombatSpecial {
    DRAGON_CLAWS(13652, 50, CombatType.MELEE, new DragonClawsSpecial()),
    AGS(11802, 50, CombatType.MELEE, new AgsSpecial()),
    // ...
}
```

**Usage:**
```java
player.getCombat().performSpecialAttack(target);
```

### Skill System

**Location:** `game-server/src/main/java/com/osroyale/content/skill/impl/`

#### Core Classes

**Skill.java:**
- 23 skills with IDs 0-22
- Fields: level, experience, max level
- Experience table for levels 1-99
- Methods: `addExperience()`, `setLevel()`, `getMaxLevel()`

**SkillManager.java:**
- Manages all skills for a Player
- Experience modification (Config multipliers)
- Level-up handling
- Skill refresh updates

#### Skill List

0. Attack
1. Defence
2. Strength
3. Hitpoints
4. Ranged
5. Prayer
6. Magic
7. Cooking
8. Woodcutting
9. Fletching
10. Fishing
11. Firemaking
12. Crafting
13. Smithing
14. Mining
15. Herblore
16. Agility
17. Thieving
18. Slayer
19. Farming
20. Runecrafting
21. Construction
22. Hunter

#### Skill Actions

**Pattern:** `SkillAction` base class

**Example: Woodcutting**

```java
public class WoodcuttingAction extends HarvestingSkillAction {

    @Override
    public boolean canExecute() {
        return player.getSkills().getLevel(Skill.WOODCUTTING) >= tree.level
            && player.inventory.hasCapacity(tree.log);
    }

    @Override
    public void execute() {
        player.animation(axe.animation);
        if (success()) {
            player.inventory.add(tree.log);
            player.skills.addExperience(Skill.WOODCUTTING, tree.experience);

            if (depleted()) {
                respawnTree();
            }
        }
    }
}
```

#### Experience Modifiers

**Configuration:** `settings.toml`

```toml
[game]
combat_modifier = 1250.0      # 12.5x XP
agility_modifier = 3000.0     # 30x XP
cooking_modifier = 3000.0     # 30x XP
# ... etc for all skills
```

**Application:**
```java
double modifier = Config.WOODCUTTING_MODIFICATION;
double xp = baseXP * modifier;
if (Config.DOUBLE_EXPERIENCE) {
    xp *= 2;
}
player.skills.addExperience(Skill.WOODCUTTING, xp);
```

### Activity System

**Location:** `game-server/src/main/java/com/osroyale/content/activity/`

#### Activity Base Class

```java
public abstract class Activity {
    protected abstract void start();
    protected abstract void finish();
    protected abstract void cleanup();
    protected abstract ActivityDeathType deathType();
}
```

#### Activity Types

**Minigames:**
- **Pest Control** - Group waves of pests
- **Barrows** - Boss sequence with puzzles
- **Fight Caves** - 63 waves culminating in Jad
- **Inferno** - Extreme challenge mode
- **Last Man Standing** - Battle royale PvP

**Bosses:**
- **Zulrah** - Pattern-based boss
- **Vorkath** - Dragon boss with mechanics
- **Kraken** - Tentacle minions
- **Cerberus** - Spirit summons
- **Godwars** - Multiple boss rooms

**Arenas:**
- **Duel Arena** - Player vs player with rules
- **Mage Arena** - God spell unlocks
- **Warrior Guild** - Token-based access

#### Activity Phases

```java
public enum ActivityPhase {
    LOBBY,      // Waiting for players
    STARTING,   // Countdown
    IN_PROGRESS,// Active gameplay
    ENDING,     // Victory/defeat
    CLEANUP     // Reset state
}
```

#### Lobby System

```java
LobbyManager.create(activityType, minPlayers, maxPlayers);

LobbyNode node = new LobbyNode(player, lobby);
lobby.add(node);

// Auto-start when threshold met
if (lobby.size() >= minPlayers) {
    lobby.start();
}
```

### Persistence Layer

**Location:** `game-server/src/main/java/com/osroyale/game/world/entity/mob/player/persist/`

#### Persistence Modes

**1. File-Based (PlayerPersistFile)**

```java
// Save location
Path saveDir = Paths.get("data", "profile", "save");
Path playerFile = saveDir.resolve(username + ".json");

// Format: JSON with Gson
JsonObject properties = new JsonObject();
for (PlayerJSONProperty prop : PROPERTIES) {
    properties.add(prop.label, gson.toJsonTree(prop.write(player)));
}
Files.write(playerFile, gson.toJson(properties).getBytes());
```

**2. Database (PlayerPersistDB)**

```java
// PostgreSQL via HikariCP
HikariDataSource pool = PostgreService.getConnectionPool();
Connection conn = pool.getConnection();

// Save query
String sql = "UPDATE players SET data = ? WHERE username = ?";
PreparedStatement stmt = conn.prepareStatement(sql);
stmt.setString(1, jsonData);
stmt.setString(2, username);
stmt.executeUpdate();
```

#### Saved Properties (100+ fields)

**Player State:**
- Username, password (Argon2 hashed)
- Position, appearance
- Play time, created date

**Inventory/Equipment:**
- Inventory items (28 slots)
- Equipment items (14 slots)
- Bank items (816 slots, 10 tabs)
- Looting bag, rune pouch

**Skills:**
- All 23 skill levels and XP
- Prestige levels per skill

**Combat:**
- Kill/death counts
- Kill streak
- Combat settings (autocast, fight type)

**Settings:**
- Client dimensions
- Brightness, zoom
- Chat modes (public, private, clan, trade)
- Game preferences (run toggle, accept aid, etc.)

**Content Progress:**
- Achievements
- Quest stages
- Slayer task/points/unlocks
- Collection log entries
- Activity statistics (LMS, Pest Control, etc.)

#### Save Triggers

```java
// Periodic save (every 5 minutes)
World.schedule(new PlayerSaveEvent());

// Logout save
player.save();

// Shutdown save
Runtime.getRuntime().addShutdownHook(() -> {
    World.get().save();
});
```

#### Async Saves (Virtual Threads)

```java
// Java 21 virtual threads for I/O
Thread.startVirtualThread(() -> {
    persistable.save(player);
    player.saved.set(true);
});
```

---

## Development Guide

### Adding New Content

#### 1. Create a Plugin

**Location:** `game-server/plugins/plugin/`

**Example: Custom NPC Shop**

```java
package plugin.click.npc;

import com.osroyale.game.plugin.PluginContext;
import com.osroyale.game.world.entity.mob.player.Player;
import com.osroyale.game.event.impl.NpcClickEvent;

public class MyShopPlugin extends PluginContext {

    @Override
    protected void firstClickNpc(Player player, NpcClickEvent event) {
        if (event.getNpc().id == 9999) {  // Your NPC ID
            openShop(player);
            event.cancel();
        }
    }

    private void openShop(Player player) {
        Shop shop = Shop.getShop("my-custom-shop");
        shop.open(player);
    }
}
```

**No registration needed!** The plugin is auto-discovered and loaded.

#### 2. Add a Command

```java
@Override
protected void handleCommand(Player player, CommandParser parser) {
    if (parser.matches("mycommand")) {
        player.message("My custom command executed!");

        if (parser.hasNext()) {
            String arg = parser.nextString();
            player.message("Argument: " + arg);
        }
    }
}
```

#### 3. Create a Minigame

**Extend Activity:**

```java
public class MyMinigame extends Activity {

    @Override
    protected void start() {
        // Initialize minigame state
        for (Player player : getPlayers()) {
            player.move(startPosition);
            player.message("Minigame started!");
        }
    }

    @Override
    protected void finish() {
        // Award rewards
        for (Player player : getPlayers()) {
            player.inventory.add(rewardItem);
            player.message("You won!");
        }
    }

    @Override
    protected void cleanup() {
        // Reset state
        getPlayers().clear();
    }

    @Override
    protected ActivityDeathType deathType() {
        return ActivityDeathType.SAFE;  // No item loss
    }
}
```

#### 4. Add a Custom Item

**Item Definition:**
```java
// Located in data/def/items.json or loaded from cache
ItemDefinition def = ItemDefinition.get(itemId);
def.setName("My Custom Item");
def.setExamine("A custom item description");
def.setNoteable(true);
def.setStackable(false);
def.setEquipmentType(EquipmentType.WEAPON);
```

**Item Click Handler:**
```java
@Override
protected void firstClickItem(Player player, ItemClickEvent event) {
    if (event.getItem().getId() == 12345) {
        player.message("You activated your custom item!");
        // Custom logic here
        event.cancel();
    }
}
```

### Testing

**Manual Testing:**
1. Start server: `./gradlew :game-server:run`
2. Start client: `./gradlew :game-client:run`
3. Login with test account
4. Use `::` commands for debugging

**Useful Debug Commands:**
- `::item <id> <amount>` - Spawn items
- `::npc <id>` - Spawn NPC
- `::tele <x> <y> [z]` - Teleport
- `::skill <name> <level>` - Set skill level
- `::bank` - Open bank anywhere
- `::master` - Max all skills

### Code Style

**Java Conventions:**
- Classes: `PascalCase`
- Methods: `camelCase`
- Constants: `UPPER_SNAKE_CASE`
- Packages: `lowercase`

**Kotlin Conventions:**
- Same as Java, plus:
- Extension functions: `fun Type.extensionName()`
- Data classes for DTOs

### Logging

```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

private static final Logger logger = LogManager.getLogger();

logger.info("Information message");
logger.warn("Warning message");
logger.error("Error message", exception);
logger.debug("Debug message");  // Only if Config.SERVER_DEBUG = true
```

---

## Configuration

### settings.toml

**Location:** `game-server/settings.toml`

#### Server Section

```toml
[server]
server_name = "Tarnish"
server_port = 43594
server_debug = false          # Enable debug logging
server_cycle_debug = false    # Log tick performance
parallel_game_engine = false  # Use parallel synchronizer
website_integration = false   # Enable forum login
```

#### World Section

```toml
[world]
type = "LIVE"  # LIVE, TEST, or LOCAL
```

- **LIVE:** Production mode
- **TEST:** Testing with relaxed constraints
- **LOCAL:** Development mode

#### Game Section

```toml
[game]
skull_time = 720              # Ticks to remain skulled (12 minutes)
log_player = true             # Log player actions
npc_bits = 16                 # NPC entity bits
npc_walking_radius = 5        # NPC wander distance
max_players = 2048            # Player capacity
max_npcs = 32000              # NPC capacity
max_bots = 10                 # Bot count

# Experience modifiers
combat_modifier = 1250.0      # 12.5x combat XP
agility_modifier = 3000.0     # 30x agility XP
# ... (all 18 skills)
```

#### Network Section

```toml
[network]
connection_limit = 3          # Simultaneous connections per IP
failed_login_attempts = 5     # Attempts before lockout
failed_login_timeout = 1      # Lockout duration (minutes)
login_threshold = 200         # Max logins per tick
logout_threshold = 200        # Max logouts per tick
client_packet_threshold = 30  # Max client packets per tick
server_packet_threshold = 1000 # Max server packets per tick
idle_timeout = 30             # Idle disconnect (seconds)
display_packets = true        # Log packet I/O
resource_leak_detection = "DISABLED"  # Netty leak detection
support_haproxy = false       # HAProxy PROXY protocol
ip_tos = "00010000"          # IP Type of Service (DSCP)

# RSA keys for login encryption
rsa_modulus = "10235303890025589152..."
rsa_exponent = "53925997225795133719..."
```

#### Database Section

```toml
[postgre]
postgre_url = ""              # JDBC URL (leave empty to disable)
postgre_user = ""             # Database username
postgre_pass = ""             # Database password
```

**Example (enabled):**
```toml
postgre_url = "jdbc:postgresql://localhost:5432/tarnish"
postgre_user = "tarnish_user"
postgre_pass = "secure_password"
```

#### Website Section

```toml
[website]
website_url = "https://osroyale.com"
forum_db_url = "jdbc:mysql://host:3306/forum_db"
forum_db_user = "forum_user"
forum_db_pass = "forum_pass"
```

#### Services Section

```toml
[services]
highscores_enabled = false    # Enable highscores API
```

#### Client Section

```toml
[client]
client_version = 12           # Client version check
```

**Version Mismatch:**
- If client version ≠ server version, login rejected with "Game updated" message

---

## Database & Persistence

### Database Schema (if using PostgreSQL)

**Players Table:**

```sql
CREATE TABLE players (
    id SERIAL PRIMARY KEY,
    username VARCHAR(12) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,  -- Argon2 hash
    created TIMESTAMP DEFAULT NOW(),
    last_login TIMESTAMP,
    play_time INTEGER DEFAULT 0,
    data JSONB  -- All player properties as JSON
);

CREATE INDEX idx_username ON players(username);
```

**Alternative:** Store individual fields instead of JSON blob for complex queries.

### Connection Pooling (HikariCP)

```java
HikariConfig config = new HikariConfig();
config.setDriverClassName("org.postgresql.Driver");
config.setJdbcUrl(Config.POSTGRE_URL);
config.setUsername(Config.POSTGRE_USER);
config.setPassword(Config.POSTGRE_PASS);
config.setMaximumPoolSize(50);
config.setConnectionTimeout(10_000);
config.setIdleTimeout(0);
config.setMaxLifetime(0);

HikariDataSource pool = new HikariDataSource(config);
```

**Usage:**
```java
try (Connection conn = PostgreService.getConnection()) {
    // Execute queries
}  // Auto-close connection back to pool
```

### Migration from Files to Database

**1. Configure PostgreSQL:**
```toml
[postgre]
postgre_url = "jdbc:postgresql://localhost:5432/tarnish"
postgre_user = "tarnish"
postgre_pass = "password"
```

**2. Enable Forum Integration:**
```toml
[server]
website_integration = true
```

**3. Migrate Existing Players:**
```java
// Read JSON files from data/profile/save/
// Insert into database using PlayerPersistDB
```

**4. Restart Server:**
- New logins/saves will use database
- Old file saves ignored

---

## Deployment

### Production Checklist

**1. Build Release JARs:**
```bash
./gradlew build -x test --no-daemon
```

**2. Configure Production Settings:**
```toml
[server]
server_debug = false
parallel_game_engine = true  # For multi-core servers

[world]
type = "LIVE"

[network]
display_packets = false
resource_leak_detection = "DISABLED"
```

**3. Database Setup:**
- Create production PostgreSQL database
- Configure connection in `settings.toml`
- Run migrations if needed

**4. Server Launch:**
```bash
java -jar game-server/build/libs/game-server-all.jar
```

**JVM Arguments (from build.gradle.kts):**
```
-XX:+UseZGC                    # ZGC garbage collector
-Xms8g -Xmx8g                  # 8GB heap
-XX:+UseStringDeduplication    # Reduce string memory
-XX:AutoBoxCacheMax=65535      # Cache boxed integers
--enable-preview               # Java preview features
```

**5. SwiftFUP Cache Server:**
```bash
# Start cache server on port (default)
cd swiftfup/server
./gradlew run
```

**6. Monitoring:**
- Watch `game-server/logs/` for errors
- Monitor tick performance with `server_cycle_debug = true`
- Check database connection pool metrics

### Reverse Proxy (Optional)

**HAProxy Configuration:**
```
listen tarnish
    bind 0.0.0.0:43594
    mode tcp
    option tcplog
    server server1 127.0.0.1:43595 send-proxy
```

**Enable in Tarnish:**
```toml
[network]
support_haproxy = true
```

### Firewall Rules

```bash
# Allow game port
sudo ufw allow 43594/tcp

# Allow cache server (SwiftFUP)
sudo ufw allow 43595/tcp
```

### Systemd Service (Linux)

**File:** `/etc/systemd/system/tarnish.service`

```ini
[Unit]
Description=Tarnish RSPS Server
After=network.target postgresql.service

[Service]
Type=simple
User=tarnish
WorkingDirectory=/opt/tarnish
ExecStart=/usr/bin/java -jar game-server-all.jar
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Commands:**
```bash
sudo systemctl enable tarnish
sudo systemctl start tarnish
sudo systemctl status tarnish
```

---

## Troubleshooting

### Common Issues

**1. Port Already in Use:**
```
java.net.BindException: Address already in use
```
**Solution:**
```bash
# Windows
netstat -ano | findstr :43594
taskkill /F /PID <PID>

# Linux
lsof -i :43594
kill -9 <PID>
```

**2. Client Can't Connect:**
- Verify server is running: Check console for "Server bound to port 43594"
- Check firewall: Ensure port 43594 is open
- Verify client version matches Config.CLIENT_VERSION

**3. Cache Files Missing:**
```
Error loading cache: File not found
```
**Solution:** Place cache files in `game-server/data/cache/`

**4. Database Connection Failed:**
```
HikariPool: Exception during pool initialization
```
**Solution:**
- Verify PostgreSQL is running
- Check connection URL, username, password in `settings.toml`
- Test connection: `psql -U tarnish_user -d tarnish`

**5. OutOfMemoryError:**
```
java.lang.OutOfMemoryError: Java heap space
```
**Solution:** Increase heap size in `build.gradle.kts`:
```kotlin
applicationDefaultJvmArgs = listOf("-Xmx16g")  // 16GB
```

### Debug Commands

**Enable Debug Mode:**
```toml
[server]
server_debug = true
```

**In-Game:**
- `::dev` - Toggle developer mode
- `::reloadplugins` - Reload plugin system
- `::gc` - Trigger garbage collection
- `::players` - List online players
- `::save` - Force save all data

---

## Additional Resources

### Project Links

- **GitHub:** https://github.com/Jire/tarnish
- **Forum:** https://rune-server.org/threads/tarnish-updated-public-release.706459/
- **RuneLite:** https://runelite.net/

### Related Projects

- **SwiftFUP:** Cache file server (see separate documentation)
- **OSRS Cache:** Official cache dumps

### Learning Resources

- **OSRS Wiki:** https://oldschool.runescape.wiki/
- **Netty Guide:** https://netty.io/wiki/user-guide.html
- **Kotlin Docs:** https://kotlinlang.org/docs/home.html

---

## Appendix

### File Locations Reference

**Server:**
- Main class: `game-server/src/main/java/com/osroyale/Main.java`
- Config: `game-server/src/main/java/com/osroyale/Config.java`
- Game engine: `game-server/src/main/java/com/osroyale/game/engine/GameThread.java`
- Plugins: `game-server/plugins/plugin/`
- Settings: `game-server/settings.toml`

**Client:**
- Main class: `game-client/src/main/java/com/osroyale/Client.java`
- Networking: `game-client/src/main/java/com/osroyale/BufferedConnection.java`

**Data:**
- Player saves: `game-server/data/profile/save/`
- Cache files: `game-server/data/cache/`
- Definitions: `game-server/data/def/`

### Key Configuration Values

| Setting | Default | Description |
|---------|---------|-------------|
| SERVER_PORT | 43594 | Server listen port |
| MAX_PLAYERS | 2048 | Player capacity |
| MAX_NPCS | 32000 | NPC capacity |
| TICK_MILLIS | 600 | Game tick interval (ms) |
| CONNECTION_LIMIT | 3 | Max connections per IP |
| IDLE_TIMEOUT | 30 | Idle disconnect (seconds) |
| CLIENT_VERSION | 12 | Client version check |

---

**End of Internal Documentation**

*Last Updated: 2025-12-26*
*Tarnish Version: Latest*
