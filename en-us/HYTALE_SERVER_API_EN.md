# Hytale Server API Documentation

## Table of Contents

1. [Overview](#overview)
2. [JAR Structure](#jar-structure)
3. [Plugin System](#plugin-system)
4. [Command System](#command-system)
5. [Entity System](#entity-system)
6. [Event System](#event-system)
7. [World System](#world-system)
8. [Inventory System](#inventory-system)
9. [Permission System](#permission-system)
10. [Math and Vectors](#math-and-vectors)
11. [Entity Components](#entity-components)
12. [Task System](#task-system)
13. [Messages and Communication](#messages-and-communication)
14. [World Generation](#world-generation)
15. [Builtin Modules](#builtin-modules)

---

## Overview

The `HytaleServer.jar` is the main library for Hytale server plugin development. This documentation was generated through analysis of the classes contained in the JAR.

### Main Packages

| Package | Description |
|---------|-------------|
| `com.hypixel.hytale.server.core` | Core server classes |
| `com.hypixel.hytale.server.core.plugin` | Plugin system |
| `com.hypixel.hytale.server.core.command` | Command system |
| `com.hypixel.hytale.server.core.entity` | Game entities |
| `com.hypixel.hytale.server.core.universe.world` | World system |
| `com.hypixel.hytale.event` | Event system |
| `com.hypixel.hytale.math` | Math and vectors |
| `com.hypixel.hytale.component` | ECS component system |
| `com.hypixel.hytale.builtin` | Builtin plugins and systems |

---

## JAR Structure

### Main Server Classes

```
com.hypixel.hytale.server.core.HytaleServer
├── getEventBus() : EventBus
├── getPluginManager() : PluginManager
├── getCommandManager() : CommandManager
├── getConfig() : HytaleServerConfig
├── shutdownServer()
├── isBooting() : boolean
├── isBooted() : boolean
├── isShuttingDown() : boolean
├── get() : HytaleServer (static, singleton)
└── getServerName() : String
```

### Server Constants

- `DEFAULT_PORT` - Default server port
- `SCHEDULED_EXECUTOR` - Executor for scheduled tasks
- `METRICS_REGISTRY` - Metrics registry

---

## Plugin System

### Class Hierarchy

```
PluginBase (abstract)
└── JavaPlugin (abstract)
    └── Your Plugin
```

### PluginBase

Base class for all Hytale plugins.

```java
public abstract class PluginBase implements CommandOwner {
    // Lifecycle Methods
    protected void setup0()      // Called during setup
    protected void setup()       // Override for initial configuration
    protected void start0()      // Called during initialization
    protected void start()       // Override for start logic
    protected void shutdown0(boolean)  // Called during shutdown
    protected void shutdown()    // Override for cleanup

    // Registry Getters
    getCommandRegistry() : CommandRegistry
    getEventRegistry() : EventRegistry
    getEntityRegistry() : EntityRegistry
    getTaskRegistry() : TaskRegistry
    getAssetRegistry() : AssetRegistry
    getBlockStateRegistry() : BlockStateRegistry
    getEntityStoreRegistry() : ComponentRegistryProxy<EntityStore>
    getChunkStoreRegistry() : ComponentRegistryProxy<ChunkStore>
    getClientFeatureRegistry() : ClientFeatureRegistry

    // Plugin Information
    getName() : String
    getLogger() : HytaleLogger
    getIdentifier() : PluginIdentifier
    getManifest() : PluginManifest
    getDataDirectory() : Path
    getState() : PluginState

    // Permissions
    getBasePermission() : String

    // State
    isEnabled() : boolean
    isDisabled() : boolean

    // Configuration
    withConfig(BuilderCodec<T>) : Config<T>
    withConfig(String, BuilderCodec<T>) : Config<T>
}
```

### JavaPlugin

Extension of PluginBase for Java plugins.

```java
public abstract class JavaPlugin extends PluginBase {
    getFile() : Path           // Plugin JAR file
    getClassLoader() : PluginClassLoader
    getType() : PluginType     // Returns PluginType.JAVA
}
```

### PluginManager

Central plugin manager.

```java
public class PluginManager {
    // Used internally by the server to load/unload plugins
}
```

### Plugin States (PluginState)

- `LOADING` - Plugin is loading
- `SETUP` - Plugin in setup phase
- `STARTING` - Plugin starting
- `ENABLED` - Plugin active
- `DISABLING` - Plugin disabling
- `DISABLED` - Plugin disabled

---

## Command System

### Class Hierarchy

```
AbstractCommand
└── CommandBase (abstract)
    ├── AbstractPlayerCommand
    ├── AbstractWorldCommand
    ├── AbstractTargetEntityCommand
    ├── AbstractTargetPlayerCommand
    ├── AbstractAsyncCommand
    ├── AbstractAsyncPlayerCommand
    ├── AbstractAsyncWorldCommand
    └── AbstractCommandCollection
```

### CommandBase

Base class for synchronous commands.

```java
public abstract class CommandBase extends AbstractCommand {
    // Constructors
    CommandBase(String name, String description)
    CommandBase(String name, String description, boolean requiresPlayer)
    CommandBase(String name)

    // Main Method - Override this method
    protected abstract void executeSync(CommandContext ctx);
}
```

### CommandContext

Command execution context.

```java
public final class CommandContext {
    // Get Arguments
    <T> T get(Argument<?, T> arg)
    String[] getInput(Argument<?, ?> arg)
    boolean provided(Argument<?, ?> arg)

    // Sender Information
    CommandSender sender()
    boolean isPlayer()
    <T extends CommandSender> T senderAs(Class<T> type)
    Ref<EntityStore> senderAsPlayerRef()

    // Communication
    void sendMessage(Message message)

    // Others
    String getInputString()
    AbstractCommand getCalledCommand()
}
```

### Message

Class for creating formatted messages.

```java
public class Message {
    // Message Creation
    static Message empty()
    static Message translation(String key)
    static Message raw(String text)
    static Message parse(String text)
    static Message join(Message... messages)

    // Formatting
    Message bold(boolean)
    Message italic(boolean)
    Message monospace(boolean)
    Message color(String hex)
    Message color(Color color)
    Message link(String url)

    // Parameters
    Message param(String key, String value)
    Message param(String key, boolean value)
    Message param(String key, double value)
    Message param(String key, int value)
    Message param(String key, long value)
    Message param(String key, float value)
    Message param(String key, Message value)

    // Composition
    Message insert(Message child)
    Message insert(String text)
    Message insertAll(Message... children)
    Message insertAll(List<Message> children)

    // Getters
    String getRawText()
    String getMessageId()
    String getColor()
    List<Message> getChildren()
    FormattedMessage getFormattedMessage()
}
```

### Argument Types (ArgTypes)

```java
public class ArgTypes {
    // Basic Types
    static SingleArgumentType<String> STRING
    static SingleArgumentType<Integer> INTEGER
    static SingleArgumentType<Float> FLOAT
    static SingleArgumentType<Double> DOUBLE
    static SingleArgumentType<Boolean> BOOLEAN
    static SingleArgumentType<Long> LONG

    // Game Types
    static SingleArgumentType<Player> PLAYER
    static SingleArgumentType<World> WORLD
    static SingleArgumentType<ItemStack> ITEM

    // Position Types
    static MultiArgumentType<RelativeIntPosition> BLOCK_POSITION
    static MultiArgumentType<RelativeDoublePosition> POSITION
    static MultiArgumentType<RelativeVector3i> VECTOR3I
    static MultiArgumentType<RelativeChunkPosition> CHUNK_POSITION
    static MultiArgumentType<RelativeDirection> DIRECTION

    // Others
    static <E extends Enum<E>> EnumArgumentType<E> enumArg(Class<E>)
}
```

### Defining Arguments

```java
public abstract class Argument<ArgType, DataType> {
    // Argument Types
    RequiredArg<T>    // Required argument
    OptionalArg<T>    // Optional argument
    DefaultArg<T>     // Argument with default value
    FlagArg           // Boolean flag
}
```

### Command Example

```java
public class MyCommand extends CommandBase {
    
    private static final RequiredArg<String> ARG_NAME = 
        new RequiredArg<>("name", ArgTypes.STRING);
    
    private static final OptionalArg<Integer> ARG_AMOUNT =
        new OptionalArg<>("amount", ArgTypes.INTEGER, 1);
    
    public MyCommand() {
        super("mycommand", "Command description", false);
        addArgument(ARG_NAME);
        addArgument(ARG_AMOUNT);
    }
    
    @Override
    protected void executeSync(CommandContext ctx) {
        String name = ctx.get(ARG_NAME);
        int amount = ctx.get(ARG_AMOUNT);
        
        ctx.sendMessage(Message.raw("Name: " + name + ", Amount: " + amount));
    }
}
```

---

## Entity System

### Entity Hierarchy

```
Entity (abstract)
└── LivingEntity (abstract)
    ├── Player
    └── ... other living entities

BlockEntity
ProjectileComponent
```

### Entity

Base class for all entities.

```java
public abstract class Entity implements Component<EntityStore> {
    // Constants
    static int UNASSIGNED_ID
    static BuilderCodec<Entity> CODEC

    // Properties
    int getNetworkId()
    UUID getUuid()
    String getLegacyDisplayName()
    World getWorld()
    boolean wasRemoved()

    // Transform
    TransformComponent getTransformComponent()
    void setTransformComponent(TransformComponent)
    void moveTo(Ref<EntityStore>, double x, double y, double z, ComponentAccessor<EntityStore>)

    // Lifecycle
    void loadIntoWorld(World)
    void unloadFromWorld()
    boolean remove()
    void markNeedsSave()

    // Reference
    Ref<EntityStore> getReference()
    void setReference(Ref<EntityStore>)
    void clearReference()
    Holder<EntityStore> toHolder()

    // Collision
    boolean isCollidable()
    boolean isHiddenFromLivingEntity(...)
}
```

### LivingEntity

Living entity with inventory and stats.

```java
public abstract class LivingEntity extends Entity {
    // Inventory
    Inventory getInventory()
    Inventory setInventory(Inventory)
    Inventory setInventory(Inventory, boolean sendToClient)

    // Stats
    StatModifiersManager getStatModifiersManager()

    // Fall
    double getCurrentFallDistance()
    void setCurrentFallDistance(double)

    // Breathing
    boolean canBreathe(...)

    // Item Durability
    boolean canDecreaseItemStackDurability(...)
    boolean canApplyItemStackPenalties(...)
    ItemStackSlotTransaction decreaseItemStackDurability(...)
    ItemStackSlotTransaction updateItemStackDurability(...)

    // Network
    void invalidateEquipmentNetwork()
    boolean consumeEquipmentNetworkOutdated()
}
```

### Player

Player class.

```java
public class Player extends LivingEntity implements CommandSender, PermissionHolder, MetricProvider {
    // Constants
    static int DEFAULT_VIEW_RADIUS_CHUNKS
    static long RESPAWN_INVULNERABILITY_TIME_NANOS
    static long MAX_TELEPORT_INVULNERABILITY_MILLIS
    static ComponentType<EntityStore, Player> getComponentType()

    // Initialization
    void init(UUID uuid, PlayerRef playerRef)
    void copyFrom(Player other)

    // Managers
    WindowManager getWindowManager()
    PageManager getPageManager()
    HudManager getHudManager()
    HotbarManager getHotbarManager()
    WorldMapTracker getWorldMapTracker()

    // Connection and Network
    PacketHandler getPlayerConnection()
    int getClientViewRadius()
    void setClientViewRadius(int)
    int getViewRadius()
    void sendInventory()

    // Player Data
    PlayerConfigData getPlayerConfigData()
    PlayerRef getPlayerRef()
    String getDisplayName()

    // Game Mode
    GameMode getGameMode()
    static void setGameMode(Ref<EntityStore>, GameMode, ComponentAccessor<EntityStore>)
    static void initGameMode(Ref<EntityStore>, ComponentAccessor<EntityStore>)

    // Spawn and Respawn
    boolean isFirstSpawn()
    void setFirstSpawn(boolean)
    boolean hasSpawnProtection()
    long getSinceLastSpawnNanos()
    void setLastSpawnTimeNanos(long)
    static Transform getRespawnPosition(Ref<EntityStore>, String, ComponentAccessor<EntityStore>)

    // Permissions
    boolean hasPermission(String permission)
    boolean hasPermission(String permission, boolean defaultValue)

    // Communication
    void sendMessage(Message message)

    // Mount
    int getMountEntityId()
    void setMountEntityId(int)

    // Movement
    void applyMovementStates(...)
    void addLocationChange(...)
    void resetVelocity(Velocity)
    void processVelocitySample(...)

    // Blocks
    boolean isOverrideBlockPlacementRestrictions()
    void setOverrideBlockPlacementRestrictions(...)
    void configTriggerBlockProcessing(...)

    // State
    void resetManagers(Holder<EntityStore>)
    boolean isWaitingForClientReady()
    void startClientReadyTimeout()
    void handleClientReady(boolean)

    // Pickup
    void notifyPickupItem(...)

    // Saving
    CompletableFuture<Void> saveConfig(World, Holder<EntityStore>)
}
```

---

## Event System

### Base Classes

```java
// Base interface for events
public interface IBaseEvent<KeyType> { }

// Synchronous event
public interface IEvent<KeyType> extends IBaseEvent<KeyType> { }

// Asynchronous event
public interface IAsyncEvent<KeyType> extends IBaseEvent<KeyType> { }

// Cancellable event
public interface ICancellable {
    boolean isCancelled()
    void setCancelled(boolean)
}

// Processed event
public interface IProcessedEvent {
    void markProcessed()
    boolean isProcessed()
}
```

### EventRegistry

Plugin event registry.

```java
public class EventRegistry extends Registry<EventRegistration<?, ?>> implements IEventRegistry {
    // Register Synchronous Events
    <E extends IBaseEvent<Void>> EventRegistration<Void, E> register(
        Class<? super E> eventClass, 
        Consumer<E> handler
    )

    <E extends IBaseEvent<Void>> EventRegistration<Void, E> register(
        EventPriority priority,
        Class<? super E> eventClass, 
        Consumer<E> handler
    )

    <K, E extends IBaseEvent<K>> EventRegistration<K, E> register(
        Class<? super E> eventClass, 
        K key,
        Consumer<E> handler
    )

    // Register Asynchronous Events
    <E extends IAsyncEvent<Void>> EventRegistration<Void, E> registerAsync(
        Class<? super E> eventClass,
        Function<CompletableFuture<E>, CompletableFuture<E>> handler
    )

    // Global Registration (receives all events of the type)
    <K, E extends IBaseEvent<K>> EventRegistration<K, E> registerGlobal(
        Class<? super E> eventClass,
        Consumer<E> handler
    )

    // Register Unhandled Events
    <K, E extends IBaseEvent<K>> EventRegistration<K, E> registerUnhandled(
        Class<? super E> eventClass,
        Consumer<E> handler
    )
}
```

### EventPriority

```java
public enum EventPriority {
    HIGHEST,
    HIGH,
    NORMAL,
    LOW,
    LOWEST,
    MONITOR  // For observation only, should not modify
}
```

### Player Events

```java
// Player connected
class PlayerConnectEvent implements IAsyncEvent<Void>

// Player disconnected
class PlayerDisconnectEvent implements IEvent<Void>

// Disconnect setup
class PlayerSetupDisconnectEvent implements IEvent<Void>

// Player added to world
class AddPlayerToWorldEvent implements IAsyncEvent<Void>

// Player removed from world
class DrainPlayerFromWorldEvent implements IEvent<Void>

// Player interaction
class PlayerInteractEvent implements IEvent<Void>

// Mouse movement
class PlayerMouseMotionEvent implements IEvent<Void>

// Player crafted
class PlayerCraftEvent implements IEvent<Void>
```

### Entity Events

```java
// Entity removed
class EntityRemoveEvent implements IEvent<Void>

// Entity used block
class LivingEntityUseBlockEvent implements IEvent<Void>

// Inventory changed
class LivingEntityInventoryChangeEvent implements IEvent<Void>
```

### World Events

```java
// World added
class AddWorldEvent implements IEvent<Void>

// World removed
class RemoveWorldEvent implements IEvent<Void>

// World started
class StartWorldEvent implements IEvent<Void>

// All worlds loaded
class AllWorldsLoadedEvent implements IEvent<Void>

// Chunk loaded
class ChunkEvent implements IEvent<Void>

// Chunk unloaded
class ChunkUnloadEvent implements IEvent<Void>

// Chunk saved
class ChunkSaveEvent implements IEvent<Void>

// Moon phase changed
class MoonPhaseChangeEvent implements IEvent<Void>
```

### Permission Events

```java
// Player permission changed
class PlayerPermissionChangeEvent implements IEvent<Void>
    class PermissionsAdded
    class PermissionsRemoved
    class GroupAdded
    class GroupRemoved

// Player group changed
class PlayerGroupEvent implements IEvent<Void>
    class Added
    class Removed

// Group permission changed
class GroupPermissionChangeEvent implements IEvent<Void>
    class Added
    class Removed
```

### Damage Events

```java
// Kill feed
class KillFeedEvent implements IEvent<Void>
    class Display
    class KillerMessage
    class DecedentMessage
```

### Event Usage Example

```java
public class MyPlugin extends JavaPlugin {
    
    @Override
    protected void start() {
        // Register connection event
        getEventRegistry().register(
            PlayerConnectEvent.class,
            this::onPlayerConnect
        );
        
        // Register event with priority
        getEventRegistry().register(
            EventPriority.HIGH,
            PlayerDisconnectEvent.class,
            this::onPlayerDisconnect
        );
    }
    
    private void onPlayerConnect(PlayerConnectEvent event) {
        getLogger().info("Player connected!");
    }
    
    private void onPlayerDisconnect(PlayerDisconnectEvent event) {
        getLogger().info("Player disconnected!");
    }
}
```

---

## World System

### World

Main world class.

```java
public class World extends TickingThread implements Executor, IWorldChunks, IMessageReceiver {
    // Constants
    static float SAVE_INTERVAL
    static String DEFAULT
    static MetricsRegistry<World> METRICS_REGISTRY

    // Basic Information
    String getName()
    Path getSavePath()
    HytaleLogger getLogger()
    boolean isAlive()

    // Configuration
    WorldConfig getWorldConfig()
    DeathConfig getDeathConfig()
    GameplayConfig getGameplayConfig()

    // Time
    int getDaytimeDurationSeconds()
    int getNighttimeDurationSeconds()
    long getTick()
    boolean isTicking()
    void setTicking(boolean)
    boolean isPaused()
    void setPaused(boolean)
    void setTps(int)
    static void setTimeDilation(float, ComponentAccessor<EntityStore>)

    // Chunks
    WorldChunk loadChunkIfInMemory(long key)
    WorldChunk getChunkIfInMemory(long key)
    WorldChunk getChunkIfLoaded(long key)
    WorldChunk getChunkIfNonTicking(long key)
    CompletableFuture<WorldChunk> getChunkAsync(long key)
    CompletableFuture<WorldChunk> getNonTickingChunkAsync(long key)

    // Entities
    Entity getEntity(UUID uuid)
    Ref<EntityStore> getEntityRef(UUID uuid)
    <T extends Entity> T spawnEntity(T entity, Vector3d position, Vector3f rotation)
    <T extends Entity> T addEntity(T entity, Vector3d position, Vector3f rotation, AddReason reason)

    // Players
    List<Player> getPlayers()
    int getPlayerCount()
    Collection<PlayerRef> getPlayerRefs()
    void trackPlayerRef(PlayerRef)
    void untrackPlayerRef(PlayerRef)
    CompletableFuture<PlayerRef> addPlayer(PlayerRef)
    CompletableFuture<PlayerRef> addPlayer(PlayerRef, Transform)
    CompletableFuture<Void> drainPlayersTo(World target)

    // Communication
    void sendMessage(Message message)

    // Stores
    ChunkStore getChunkStore()
    EntityStore getEntityStore()
    ChunkLightingManager getChunkLighting()
    WorldMapManager getWorldMapManager()
    WorldPathConfig getWorldPathConfig()

    // Events and Notifications
    EventRegistry getEventRegistry()
    WorldNotificationHandler getNotificationHandler()

    // Client Features
    Map<ClientFeature, Boolean> getFeatures()
    boolean isFeatureEnabled(ClientFeature)
    void registerFeature(ClientFeature, boolean)
    void broadcastFeatures()

    // Compass and Map
    boolean isCompassUpdating()
    void setCompassUpdating(boolean)

    // Initialization and Shutdown
    CompletableFuture<World> init()
    void stopIndividualWorld()

    // Execution
    void execute(Runnable task)
    void consumeTaskQueue()
}
```

### WorldConfig

World configuration.

```java
public class WorldConfig {
    // Sub-classes
    class ChunkConfig { }
    
    // World-specific configurations
}
```

---

## Inventory System

### Inventory

```java
public class Inventory {
    // Player inventory methods
}
```

### ItemStack

Represents a stacked item.

```java
public class ItemStack {
    // Codec
    static BuilderCodec<ItemStack> CODEC

    // Properties
    // ... item manipulation methods

    // Inner classes
    class Metadata { }
}
```

### ItemContainer

Generic item container.

```java
public interface ItemContainer {
    // Container methods
    
    // Change event
    class ItemContainerChangeEvent implements IEvent<Void> { }
    
    // Temporary data
    class TempItemData { }
}
```

### Container Types

- `SimpleItemContainer` - Simple container
- `EmptyItemContainer` - Empty container
- `DelegateItemContainer` - Delegate container
- `ItemStackItemContainer` - ItemStack-based container
- `CombinedItemContainer` - Combined container

### Slot Filters

```java
// Filter types
class SlotFilter { }
class TagFilter { }
class ArmorSlotAddFilter { }
class NoDuplicateFilter { }
class ResourceFilter { }
class ItemSlotFilter { }
```

---

## Permission System

### PermissionHolder

Interface for entities with permissions.

```java
public interface PermissionHolder {
    boolean hasPermission(String permission)
    boolean hasPermission(String permission, boolean defaultValue)
}
```

### HytalePermissions

Default permissions class.

```java
public class HytalePermissions {
    // System permission constants
}
```

### PermissionProvider

Interface for permission providers.

```java
public interface PermissionProvider {
    // Permission query methods
}
```

---

## Math and Vectors

### Vector3d

3D vector with double precision.

```java
public class Vector3d {
    // Constants
    static Vector3d ZERO
    static Vector3d UP, DOWN
    static Vector3d FORWARD, BACKWARD
    static Vector3d RIGHT, LEFT
    static Vector3d NORTH, SOUTH, EAST, WEST
    static Vector3d ALL_ONES
    static Vector3d MIN, MAX
    static Vector3d[] BLOCK_SIDES
    static Vector3d[] BLOCK_EDGES
    static Vector3d[] BLOCK_CORNERS
    static Vector3d[] CARDINAL_DIRECTIONS

    // Fields
    double x, y, z

    // Constructors
    Vector3d()
    Vector3d(Vector3d other)
    Vector3d(Vector3i other)
    Vector3d(double x, double y, double z)
    Vector3d(float pitch, float yaw)
    Vector3d(Random random, double scale)

    // Getters and Setters
    double getX(), setX(double)
    double getY(), setY(double)
    double getZ(), setZ(double)

    // Assignment
    Vector3d assign(Vector3d)
    Vector3d assign(double value)
    Vector3d assign(double[] values)
    Vector3d assign(double x, double y, double z)

    // Math Operations
    Vector3d add(Vector3d)
    Vector3d add(Vector3i)
    Vector3d add(double x, double y, double z)
    Vector3d add(double value)
    Vector3d addScaled(Vector3d v, double scale)
    Vector3d subtract(Vector3d)
    Vector3d subtract(double x, double y, double z)
    Vector3d negate()
    Vector3d scale(double factor)
    Vector3d scale(Vector3d factors)
    Vector3d cross(Vector3d)
    double dot(Vector3d)

    // Distance
    double distanceTo(Vector3d)
    double distanceTo(Vector3i)
    double distanceTo(double x, double y, double z)
    double distanceSquaredTo(Vector3d)
    double distanceSquaredTo(double x, double y, double z)

    // Normalization and Length
    Vector3d normalize()
    double length()
    double squaredLength()
    Vector3d setLength(double length)
    Vector3d clampLength(double maxLength)

    // Rotation
    Vector3d rotateX(float angle)
    Vector3d rotateY(float angle)
    Vector3d rotateZ(float angle)

    // Rounding
    Vector3d floor()
    Vector3d ceil()
    Vector3d clipToZero(double threshold)

    // Checks
    boolean closeToZero(double threshold)
    boolean isInside(int sizeX, int sizeY, int sizeZ)
    boolean isFinite()

    // Conversion
    Vector3i toVector3i()
    Vector3f toVector3f()
    Vector3d clone()

    // Static Methods
    static Vector3d max(Vector3d a, Vector3d b)
    static Vector3d min(Vector3d a, Vector3d b)
    static Vector3d lerp(Vector3d a, Vector3d b, double t)
    static Vector3d lerpUnclamped(Vector3d a, Vector3d b, double t)
    static Vector3d directionTo(Vector3d from, Vector3d to)
    static Vector3d add(Vector3d a, Vector3d b)
    static String formatShortString(Vector3d v)
}
```

### Vector3f

3D vector with float precision.

```java
public class Vector3f {
    float x, y, z
    
    // Constructors and methods similar to Vector3d
    // Used mainly for rotations
}
```

### Vector3i

3D vector with integer precision.

```java
public class Vector3i {
    int x, y, z
    
    // Constructors and methods similar
    // Used for block positions
}
```

### Transform

Represents transformation (position + rotation).

```java
public class Transform {
    // Combines position (Vector3d) and rotation (Vector3f)
}
```

### Location

Represents a location in the world.

```java
public class Location {
    // World + Position
}
```

### Other Vectors

- `Vector2d` - 2D double vector
- `Vector2i` - 2D int vector
- `Vector2l` - 2D long vector
- `Vector3l` - 3D long vector
- `Vector4d` - 4D double vector

---

## Entity Components

### TransformComponent

Transform component (position and rotation).

```java
public class TransformComponent implements Component<EntityStore> {
    // Codec
    static BuilderCodec<TransformComponent> CODEC
    static ComponentType<EntityStore, TransformComponent> getComponentType()

    // Constructors
    TransformComponent()
    TransformComponent(Vector3d position, Vector3f rotation)

    // Position
    Vector3d getPosition()
    void setPosition(Vector3d position)
    void teleportPosition(Vector3d position)

    // Rotation
    Vector3f getRotation()
    void setRotation(Vector3f rotation)
    void teleportRotation(Vector3f rotation)

    // Full Transform
    Transform getTransform()

    // Chunk
    WorldChunk getChunk()
    Ref<ChunkStore> getChunkRef()
    void setChunkLocation(Ref<ChunkStore>, WorldChunk)
    void markChunkDirty(ComponentAccessor<EntityStore>)

    // Network
    ModelTransform getSentTransform()

    // Clone
    TransformComponent clone()
}
```

### Other Entity Components

```java
// Components available in com.hypixel.hytale.server.core.modules.entity.component

class BoundingBox          // Collision box
class DisplayNameComponent // Display name
class ModelComponent       // 3D model
class HeadRotation         // Head rotation
class DynamicLight         // Dynamic light
class AudioComponent       // Audio
class Invulnerable         // Invulnerability
class Intangible           // Intangibility
class EntityScaleComponent // Entity scale
class PositionDataComponent // Position data
class CollisionResultComponent // Collision result
class SnapshotBuffer       // Snapshot buffer
class FromPrefab           // Prefab origin
class FromWorldGen         // World generation origin
class WorldGenId           // Generation ID
class PropComponent        // Prop component
class NewSpawnComponent    // New spawn
class Interactable         // Interactable
class RespondToHit         // Hit response
class RotateObjectComponent // Object rotation
class MovementAudioComponent // Movement audio
class ActiveAnimationComponent // Active animation
class PersistentModel      // Persistent model
class PersistentDynamicLight // Persistent dynamic light
class HiddenFromAdventurePlayers // Hidden from adventure players
```

---

## Task System

### TaskRegistry

Scheduled task registry.

```java
public class TaskRegistry extends Registry<TaskRegistration> {
    // Register CompletableFuture task
    TaskRegistration registerTask(CompletableFuture<Void> task)

    // Register scheduled task
    TaskRegistration registerTask(ScheduledFuture<Void> task)
}
```

### Usage Example

```java
public class MyPlugin extends JavaPlugin {
    
    @Override
    protected void start() {
        // Single task
        CompletableFuture<Void> task = CompletableFuture.runAsync(() -> {
            // Task code
        });
        getTaskRegistry().registerTask(task);
        
        // Scheduled task
        ScheduledFuture<Void> scheduledTask = HytaleServer.SCHEDULED_EXECUTOR
            .scheduleAtFixedRate(() -> {
                // Periodically executed code
            }, 0, 1, TimeUnit.SECONDS);
        getTaskRegistry().registerTask(scheduledTask);
    }
}
```

---

## World Generation

### Main Package

```
com.hypixel.hytale.builtin.hytalegenerator
```

### Generation Components

#### Density Nodes

Nodes for terrain density calculation:

- `ConstantValueDensity` - Constant value
- `Noise2dDensity` - 2D noise
- `Noise3dDensity` - 3D noise
- `SumDensity` - Sum of densities
- `MultiplierDensity` - Multiplication
- `MinDensity`, `MaxDensity` - Minimum/Maximum
- `ClampDensity` - Clamping
- `GradientDensity` - Gradient
- `SelectorDensity` - Selection
- `CacheDensity` - Cache
- `TerrainDensity` - Terrain density
- And many more...

#### Material Providers

Material providers for generation:

- `MaterialProvider` - Base interface
- `ConstantMaterialProviderAsset`
- `WeightedMaterialProviderAsset`
- `FieldFunctionMaterialProviderAsset`
- `StripedMaterialProviderAsset`
- `TerrainDensityMaterialProviderAsset`

#### Biomes

Biome system:

- `BiomeAsset` - Biome asset
- `BiomeType` - Biome type
- `BiomeMap` - Biome map
- `SimpleBiomeMap` - Simple implementation

#### Props

Props system (vegetation, decorations):

- `Prop` - Base class
- `PrefabProp` - Prefab prop
- `ClusterProp` - Prop cluster
- `DensityProp` - Density-based prop
- `BoxProp` - Box prop
- `ColumnProp` - Column prop

---

## Builtin Modules

### Included Builtin Plugins

The HytaleServer.jar includes several builtin plugins:

#### HytaleGenerator
Official Hytale world generator.

#### Teleporter
Teleport system.

```java
// Components
class Teleporter

// Interactions
class TeleporterInteraction

// Pages
class TeleporterSettingsPage
```

#### Memories
Game memories system.

```java
// Components
class PlayerMemories
class Memory

// Pages
class MemoriesPage
```

#### Portals
Portal system.

```java
// Components
class PortalDevice
class VoidEvent
class VoidSpawner

// Interactions
class EnterPortalInteraction
class ReturnPortalInteraction
```

#### Beds
Bed and sleep system.

```java
// Resources
class WorldSleep
class WorldSlumber

// Components
class PlayerSleep
class SleepTracker

// Interactions
class BedInteraction
```

#### Deployables
Deployable items system (turrets, traps).

```java
// Components
class DeployableComponent
class DeployableOwnerComponent
class DeployableProjectileComponent

// Configurations
class DeployableTurretConfig
class DeployableTrapConfig
class DeployableAoeConfig
```

#### Ambience
Ambience and music system.

```java
// Components
class AmbienceTracker
class AmbientEmitterComponent

// Resources
class AmbienceResource
```

#### Creative Hub
Creative hub.

```java
// Interactions
class HubPortalInteraction

// Commands
class HubCommand
```

---

## Builtin Commands

The server includes several native commands:

### World Commands

- `/world` - World management
- `/worldconfig` - World configuration
- `/block` - Block manipulation
- `/time` - Time control

### Entity Commands

- `/spawn` - Entity spawning
- `/teleport` - Teleport
- `/gamemode` - Game mode

### Permission Commands

- `/op` - Operator
- `/perm` - Permissions

### Access Commands

- `/whitelist` - Whitelist
- `/ban` - Ban
- `/unban` - Unban

### Debug Commands

- `/debug` - Debugging
- `/hitbox` - Hitbox visualization

### Plugin Commands

- `/plugin` - Plugin management

### Interaction Commands

- `/interaction` - Interaction system

---

## Manifest Structure

The `manifest.json` file must be in `src/main/resources/`:

```json
{
    "id": "my-plugin",
    "name": "My Plugin",
    "version": "1.0.0",
    "description": "Plugin description",
    "authors": [
        {
            "name": "Author",
            "role": "Developer"
        }
    ],
    "main": "com.example.MyPlugin"
}
```

---

## Complete Plugin Example

```java
package com.example;

import com.hypixel.hytale.server.core.plugin.JavaPlugin;
import com.hypixel.hytale.server.core.plugin.JavaPluginInit;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.event.events.player.PlayerConnectEvent;
import com.hypixel.hytale.event.EventPriority;

public class MyPlugin extends JavaPlugin {

    public MyPlugin(JavaPluginInit init) {
        super(init);
    }

    @Override
    protected void setup() {
        getLogger().info("Plugin in setup!");
    }

    @Override
    protected void start() {
        getLogger().info("Plugin started!");
        
        // Register commands
        getCommandRegistry().register(new MyCommand());
        
        // Register events
        getEventRegistry().register(
            EventPriority.NORMAL,
            PlayerConnectEvent.class,
            this::onPlayerConnect
        );
    }

    @Override
    protected void shutdown() {
        getLogger().info("Plugin shutting down!");
    }

    private void onPlayerConnect(PlayerConnectEvent event) {
        getLogger().info("Player connected to the server!");
    }

    // Internal command
    public static class MyCommand extends CommandBase {
        
        public MyCommand() {
            super("mycommand", "An example command", false);
        }

        @Override
        protected void executeSync(CommandContext ctx) {
            ctx.sendMessage(Message.raw("Command executed successfully!"));
        }
    }
}
```

---

## Additional References

### Included Third-Party Packages

The JAR includes some third-party libraries:

- `javax.annotation` - JSR-305 Annotations
- `org.bouncycastle` - Cryptography

### Logging

The system uses `HytaleLogger`:

```java
HytaleLogger logger = getLogger();
logger.info("Informational message");
logger.warn("Warning");
logger.error("Error");
logger.debug("Debug");
```

### Codecs

The system uses codecs for serialization:

```java
// BuilderCodec for complex objects
BuilderCodec<MyObject> CODEC = BuilderCodec.builder()
    .field("field", ...)
    .build();
```

---

## Important Notes

1. **Unofficial API**: This documentation was generated by bytecode analysis. The API may change without notice.

2. **ECS System**: Hytale uses an Entity-Component-System. Familiarize yourself with `Component`, `Ref`, `Holder`, `ComponentAccessor`.

3. **Asynchronous**: Many operations are asynchronous and return `CompletableFuture`.

4. **Thread Safety**: World operations must be executed on the world thread using `world.execute(Runnable)`.

5. **Saving**: Call `markNeedsSave()` to ensure changes are persisted.

---

*Documentation automatically generated through HytaleServer.jar analysis*
*Analyzed JAR version: Compatible with Java 25*
