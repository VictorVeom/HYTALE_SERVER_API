# Documentação da API do Hytale Server

## Índice

1. [Visão Geral](#visão-geral)
2. [Estrutura do JAR](#estrutura-do-jar)
3. [Sistema de Plugins](#sistema-de-plugins)
4. [Sistema de Comandos](#sistema-de-comandos)
5. [Sistema de Entidades](#sistema-de-entidades)
6. [Sistema de Eventos](#sistema-de-eventos)
7. [Sistema de Mundos](#sistema-de-mundos)
8. [Sistema de Inventário](#sistema-de-inventário)
9. [Sistema de Permissões](#sistema-de-permissões)
10. [Matemática e Vetores](#matemática-e-vetores)
11. [Componentes de Entidades](#componentes-de-entidades)
12. [Sistema de Tarefas](#sistema-de-tarefas)
13. [Mensagens e Comunicação](#mensagens-e-comunicação)
14. [Geração de Mundo](#geração-de-mundo)
15. [Módulos Builtin](#módulos-builtin)

---

## Visão Geral

O `HytaleServer.jar` é a biblioteca principal para desenvolvimento de plugins no servidor Hytale. Esta documentação foi gerada através da análise das classes contidas no JAR.

### Pacotes Principais

| Pacote | Descrição |
|--------|-----------|
| `com.hypixel.hytale.server.core` | Classes principais do servidor |
| `com.hypixel.hytale.server.core.plugin` | Sistema de plugins |
| `com.hypixel.hytale.server.core.command` | Sistema de comandos |
| `com.hypixel.hytale.server.core.entity` | Entidades do jogo |
| `com.hypixel.hytale.server.core.universe.world` | Sistema de mundos |
| `com.hypixel.hytale.event` | Sistema de eventos |
| `com.hypixel.hytale.math` | Matemática e vetores |
| `com.hypixel.hytale.component` | Sistema de componentes ECS |
| `com.hypixel.hytale.builtin` | Plugins e sistemas builtin |

---

## Estrutura do JAR

### Classes de Servidor Principal

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

### Constantes do Servidor

- `DEFAULT_PORT` - Porta padrão do servidor
- `SCHEDULED_EXECUTOR` - Executor para tarefas agendadas
- `METRICS_REGISTRY` - Registro de métricas

---

## Sistema de Plugins

### Hierarquia de Classes

```
PluginBase (abstrato)
└── JavaPlugin (abstrato)
    └── Seu Plugin
```

### PluginBase

Classe base para todos os plugins do Hytale.

```java
public abstract class PluginBase implements CommandOwner {
    // Métodos de Ciclo de Vida
    protected void setup0()      // Chamado durante setup
    protected void setup()       // Override para configuração inicial
    protected void start0()      // Chamado durante inicialização
    protected void start()       // Override para lógica de início
    protected void shutdown0(boolean)  // Chamado durante shutdown
    protected void shutdown()    // Override para cleanup

    // Getters de Registros
    getCommandRegistry() : CommandRegistry
    getEventRegistry() : EventRegistry
    getEntityRegistry() : EntityRegistry
    getTaskRegistry() : TaskRegistry
    getAssetRegistry() : AssetRegistry
    getBlockStateRegistry() : BlockStateRegistry
    getEntityStoreRegistry() : ComponentRegistryProxy<EntityStore>
    getChunkStoreRegistry() : ComponentRegistryProxy<ChunkStore>
    getClientFeatureRegistry() : ClientFeatureRegistry

    // Informações do Plugin
    getName() : String
    getLogger() : HytaleLogger
    getIdentifier() : PluginIdentifier
    getManifest() : PluginManifest
    getDataDirectory() : Path
    getState() : PluginState

    // Permissões
    getBasePermission() : String

    // Estado
    isEnabled() : boolean
    isDisabled() : boolean

    // Configuração
    withConfig(BuilderCodec<T>) : Config<T>
    withConfig(String, BuilderCodec<T>) : Config<T>
}
```

### JavaPlugin

Extensão de PluginBase para plugins Java.

```java
public abstract class JavaPlugin extends PluginBase {
    getFile() : Path           // Arquivo JAR do plugin
    getClassLoader() : PluginClassLoader
    getType() : PluginType     // Retorna PluginType.JAVA
}
```

### PluginManager

Gerenciador central de plugins.

```java
public class PluginManager {
    // Usado internamente pelo servidor para carregar/descarregar plugins
}
```

### Estados de Plugin (PluginState)

- `LOADING` - Plugin está carregando
- `SETUP` - Plugin em fase de setup
- `STARTING` - Plugin iniciando
- `ENABLED` - Plugin ativo
- `DISABLING` - Plugin desabilitando
- `DISABLED` - Plugin desabilitado

---

## Sistema de Comandos

### Hierarquia de Classes

```
AbstractCommand
└── CommandBase (abstrato)
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

Classe base para comandos síncronos.

```java
public abstract class CommandBase extends AbstractCommand {
    // Construtores
    CommandBase(String name, String description)
    CommandBase(String name, String description, boolean requiresPlayer)
    CommandBase(String name)

    // Método Principal - Override este método
    protected abstract void executeSync(CommandContext ctx);
}
```

### CommandContext

Contexto de execução de um comando.

```java
public final class CommandContext {
    // Obter Argumentos
    <T> T get(Argument<?, T> arg)
    String[] getInput(Argument<?, ?> arg)
    boolean provided(Argument<?, ?> arg)

    // Informações do Sender
    CommandSender sender()
    boolean isPlayer()
    <T extends CommandSender> T senderAs(Class<T> type)
    Ref<EntityStore> senderAsPlayerRef()

    // Comunicação
    void sendMessage(Message message)

    // Outros
    String getInputString()
    AbstractCommand getCalledCommand()
}
```

### Message

Classe para criar mensagens formatadas.

```java
public class Message {
    // Criação de Mensagens
    static Message empty()
    static Message translation(String key)
    static Message raw(String text)
    static Message parse(String text)
    static Message join(Message... messages)

    // Formatação
    Message bold(boolean)
    Message italic(boolean)
    Message monospace(boolean)
    Message color(String hex)
    Message color(Color color)
    Message link(String url)

    // Parâmetros
    Message param(String key, String value)
    Message param(String key, boolean value)
    Message param(String key, double value)
    Message param(String key, int value)
    Message param(String key, long value)
    Message param(String key, float value)
    Message param(String key, Message value)

    // Composição
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

### Tipos de Argumentos (ArgTypes)

```java
public class ArgTypes {
    // Tipos Básicos
    static SingleArgumentType<String> STRING
    static SingleArgumentType<Integer> INTEGER
    static SingleArgumentType<Float> FLOAT
    static SingleArgumentType<Double> DOUBLE
    static SingleArgumentType<Boolean> BOOLEAN
    static SingleArgumentType<Long> LONG

    // Tipos de Jogo
    static SingleArgumentType<Player> PLAYER
    static SingleArgumentType<World> WORLD
    static SingleArgumentType<ItemStack> ITEM

    // Tipos de Posição
    static MultiArgumentType<RelativeIntPosition> BLOCK_POSITION
    static MultiArgumentType<RelativeDoublePosition> POSITION
    static MultiArgumentType<RelativeVector3i> VECTOR3I
    static MultiArgumentType<RelativeChunkPosition> CHUNK_POSITION
    static MultiArgumentType<RelativeDirection> DIRECTION

    // Outros
    static <E extends Enum<E>> EnumArgumentType<E> enumArg(Class<E>)
}
```

### Definindo Argumentos

```java
public abstract class Argument<ArgType, DataType> {
    // Tipos de Argumentos
    RequiredArg<T>    // Argumento obrigatório
    OptionalArg<T>    // Argumento opcional
    DefaultArg<T>     // Argumento com valor padrão
    FlagArg           // Flag booleana
}
```

### Exemplo de Comando

```java
public class MeuComando extends CommandBase {
    
    private static final RequiredArg<String> ARG_NOME = 
        new RequiredArg<>("nome", ArgTypes.STRING);
    
    private static final OptionalArg<Integer> ARG_QUANTIDADE =
        new OptionalArg<>("quantidade", ArgTypes.INTEGER, 1);
    
    public MeuComando() {
        super("meucomando", "Descrição do comando", false);
        addArgument(ARG_NOME);
        addArgument(ARG_QUANTIDADE);
    }
    
    @Override
    protected void executeSync(CommandContext ctx) {
        String nome = ctx.get(ARG_NOME);
        int quantidade = ctx.get(ARG_QUANTIDADE);
        
        ctx.sendMessage(Message.raw("Nome: " + nome + ", Qtd: " + quantidade));
    }
}
```

---

## Sistema de Entidades

### Hierarquia de Entidades

```
Entity (abstrato)
└── LivingEntity (abstrato)
    ├── Player
    └── ... outras entidades vivas

BlockEntity
ProjectileComponent
```

### Entity

Classe base para todas as entidades.

```java
public abstract class Entity implements Component<EntityStore> {
    // Constantes
    static int UNASSIGNED_ID
    static BuilderCodec<Entity> CODEC

    // Propriedades
    int getNetworkId()
    UUID getUuid()
    String getLegacyDisplayName()
    World getWorld()
    boolean wasRemoved()

    // Transformação
    TransformComponent getTransformComponent()
    void setTransformComponent(TransformComponent)
    void moveTo(Ref<EntityStore>, double x, double y, double z, ComponentAccessor<EntityStore>)

    // Lifecycle
    void loadIntoWorld(World)
    void unloadFromWorld()
    boolean remove()
    void markNeedsSave()

    // Referência
    Ref<EntityStore> getReference()
    void setReference(Ref<EntityStore>)
    void clearReference()
    Holder<EntityStore> toHolder()

    // Colisão
    boolean isCollidable()
    boolean isHiddenFromLivingEntity(...)
}
```

### LivingEntity

Entidade viva com inventário e estatísticas.

```java
public abstract class LivingEntity extends Entity {
    // Inventário
    Inventory getInventory()
    Inventory setInventory(Inventory)
    Inventory setInventory(Inventory, boolean sendToClient)

    // Estatísticas
    StatModifiersManager getStatModifiersManager()

    // Queda
    double getCurrentFallDistance()
    void setCurrentFallDistance(double)

    // Respiração
    boolean canBreathe(...)

    // Durabilidade de Itens
    boolean canDecreaseItemStackDurability(...)
    boolean canApplyItemStackPenalties(...)
    ItemStackSlotTransaction decreaseItemStackDurability(...)
    ItemStackSlotTransaction updateItemStackDurability(...)

    // Rede
    void invalidateEquipmentNetwork()
    boolean consumeEquipmentNetworkOutdated()
}
```

### Player

Classe de jogador.

```java
public class Player extends LivingEntity implements CommandSender, PermissionHolder, MetricProvider {
    // Constantes
    static int DEFAULT_VIEW_RADIUS_CHUNKS
    static long RESPAWN_INVULNERABILITY_TIME_NANOS
    static long MAX_TELEPORT_INVULNERABILITY_MILLIS
    static ComponentType<EntityStore, Player> getComponentType()

    // Inicialização
    void init(UUID uuid, PlayerRef playerRef)
    void copyFrom(Player other)

    // Managers
    WindowManager getWindowManager()
    PageManager getPageManager()
    HudManager getHudManager()
    HotbarManager getHotbarManager()
    WorldMapTracker getWorldMapTracker()

    // Conexão e Rede
    PacketHandler getPlayerConnection()
    int getClientViewRadius()
    void setClientViewRadius(int)
    int getViewRadius()
    void sendInventory()

    // Dados do Jogador
    PlayerConfigData getPlayerConfigData()
    PlayerRef getPlayerRef()
    String getDisplayName()

    // Game Mode
    GameMode getGameMode()
    static void setGameMode(Ref<EntityStore>, GameMode, ComponentAccessor<EntityStore>)
    static void initGameMode(Ref<EntityStore>, ComponentAccessor<EntityStore>)

    // Spawn e Respawn
    boolean isFirstSpawn()
    void setFirstSpawn(boolean)
    boolean hasSpawnProtection()
    long getSinceLastSpawnNanos()
    void setLastSpawnTimeNanos(long)
    static Transform getRespawnPosition(Ref<EntityStore>, String, ComponentAccessor<EntityStore>)

    // Permissões
    boolean hasPermission(String permission)
    boolean hasPermission(String permission, boolean defaultValue)

    // Comunicação
    void sendMessage(Message message)

    // Montaria
    int getMountEntityId()
    void setMountEntityId(int)

    // Movimento
    void applyMovementStates(...)
    void addLocationChange(...)
    void resetVelocity(Velocity)
    void processVelocitySample(...)

    // Blocos
    boolean isOverrideBlockPlacementRestrictions()
    void setOverrideBlockPlacementRestrictions(...)
    void configTriggerBlockProcessing(...)

    // Estado
    void resetManagers(Holder<EntityStore>)
    boolean isWaitingForClientReady()
    void startClientReadyTimeout()
    void handleClientReady(boolean)

    // Pickup
    void notifyPickupItem(...)

    // Salvamento
    CompletableFuture<Void> saveConfig(World, Holder<EntityStore>)
}
```

---

## Sistema de Eventos

### Classes Base

```java
// Interface base para eventos
public interface IBaseEvent<KeyType> { }

// Evento síncrono
public interface IEvent<KeyType> extends IBaseEvent<KeyType> { }

// Evento assíncrono
public interface IAsyncEvent<KeyType> extends IBaseEvent<KeyType> { }

// Evento que pode ser cancelado
public interface ICancellable {
    boolean isCancelled()
    void setCancelled(boolean)
}

// Evento processado
public interface IProcessedEvent {
    void markProcessed()
    boolean isProcessed()
}
```

### EventRegistry

Registro de eventos do plugin.

```java
public class EventRegistry extends Registry<EventRegistration<?, ?>> implements IEventRegistry {
    // Registro de Eventos Síncronos
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

    // Registro de Eventos Assíncronos
    <E extends IAsyncEvent<Void>> EventRegistration<Void, E> registerAsync(
        Class<? super E> eventClass,
        Function<CompletableFuture<E>, CompletableFuture<E>> handler
    )

    // Registro Global (recebe todos os eventos do tipo)
    <K, E extends IBaseEvent<K>> EventRegistration<K, E> registerGlobal(
        Class<? super E> eventClass,
        Consumer<E> handler
    )

    // Registro de Eventos Não Processados
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
    MONITOR  // Apenas para observação, não deve modificar
}
```

### Eventos de Jogador

```java
// Jogador conectou
class PlayerConnectEvent implements IAsyncEvent<Void>

// Jogador desconectou
class PlayerDisconnectEvent implements IEvent<Void>

// Setup de desconexão
class PlayerSetupDisconnectEvent implements IEvent<Void>

// Jogador adicionado ao mundo
class AddPlayerToWorldEvent implements IAsyncEvent<Void>

// Jogador removido do mundo
class DrainPlayerFromWorldEvent implements IEvent<Void>

// Interação do jogador
class PlayerInteractEvent implements IEvent<Void>

// Movimento do mouse
class PlayerMouseMotionEvent implements IEvent<Void>

// Jogador craftou
class PlayerCraftEvent implements IEvent<Void>
```

### Eventos de Entidade

```java
// Entidade removida
class EntityRemoveEvent implements IEvent<Void>

// Entidade usou bloco
class LivingEntityUseBlockEvent implements IEvent<Void>

// Inventário mudou
class LivingEntityInventoryChangeEvent implements IEvent<Void>
```

### Eventos de Mundo

```java
// Mundo adicionado
class AddWorldEvent implements IEvent<Void>

// Mundo removido
class RemoveWorldEvent implements IEvent<Void>

// Mundo iniciado
class StartWorldEvent implements IEvent<Void>

// Todos os mundos carregados
class AllWorldsLoadedEvent implements IEvent<Void>

// Chunk carregado
class ChunkEvent implements IEvent<Void>

// Chunk descarregado
class ChunkUnloadEvent implements IEvent<Void>

// Chunk salvo
class ChunkSaveEvent implements IEvent<Void>

// Fase da lua mudou
class MoonPhaseChangeEvent implements IEvent<Void>
```

### Eventos de Permissão

```java
// Permissão do jogador mudou
class PlayerPermissionChangeEvent implements IEvent<Void>
    class PermissionsAdded
    class PermissionsRemoved
    class GroupAdded
    class GroupRemoved

// Grupo do jogador mudou
class PlayerGroupEvent implements IEvent<Void>
    class Added
    class Removed

// Permissão de grupo mudou
class GroupPermissionChangeEvent implements IEvent<Void>
    class Added
    class Removed
```

### Eventos de Dano

```java
// Kill feed
class KillFeedEvent implements IEvent<Void>
    class Display
    class KillerMessage
    class DecedentMessage
```

### Exemplo de Uso de Eventos

```java
public class MeuPlugin extends JavaPlugin {
    
    @Override
    protected void start() {
        // Registrar evento de conexão
        getEventRegistry().register(
            PlayerConnectEvent.class,
            this::onPlayerConnect
        );
        
        // Registrar evento com prioridade
        getEventRegistry().register(
            EventPriority.HIGH,
            PlayerDisconnectEvent.class,
            this::onPlayerDisconnect
        );
    }
    
    private void onPlayerConnect(PlayerConnectEvent event) {
        getLogger().info("Jogador conectou!");
    }
    
    private void onPlayerDisconnect(PlayerDisconnectEvent event) {
        getLogger().info("Jogador desconectou!");
    }
}
```

---

## Sistema de Mundos

### World

Classe principal de mundo.

```java
public class World extends TickingThread implements Executor, IWorldChunks, IMessageReceiver {
    // Constantes
    static float SAVE_INTERVAL
    static String DEFAULT
    static MetricsRegistry<World> METRICS_REGISTRY

    // Informações Básicas
    String getName()
    Path getSavePath()
    HytaleLogger getLogger()
    boolean isAlive()

    // Configuração
    WorldConfig getWorldConfig()
    DeathConfig getDeathConfig()
    GameplayConfig getGameplayConfig()

    // Tempo
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

    // Entidades
    Entity getEntity(UUID uuid)
    Ref<EntityStore> getEntityRef(UUID uuid)
    <T extends Entity> T spawnEntity(T entity, Vector3d position, Vector3f rotation)
    <T extends Entity> T addEntity(T entity, Vector3d position, Vector3f rotation, AddReason reason)

    // Jogadores
    List<Player> getPlayers()
    int getPlayerCount()
    Collection<PlayerRef> getPlayerRefs()
    void trackPlayerRef(PlayerRef)
    void untrackPlayerRef(PlayerRef)
    CompletableFuture<PlayerRef> addPlayer(PlayerRef)
    CompletableFuture<PlayerRef> addPlayer(PlayerRef, Transform)
    CompletableFuture<Void> drainPlayersTo(World target)

    // Comunicação
    void sendMessage(Message message)

    // Stores
    ChunkStore getChunkStore()
    EntityStore getEntityStore()
    ChunkLightingManager getChunkLighting()
    WorldMapManager getWorldMapManager()
    WorldPathConfig getWorldPathConfig()

    // Eventos e Notificações
    EventRegistry getEventRegistry()
    WorldNotificationHandler getNotificationHandler()

    // Features do Cliente
    Map<ClientFeature, Boolean> getFeatures()
    boolean isFeatureEnabled(ClientFeature)
    void registerFeature(ClientFeature, boolean)
    void broadcastFeatures()

    // Compass e Mapa
    boolean isCompassUpdating()
    void setCompassUpdating(boolean)

    // Inicialização e Shutdown
    CompletableFuture<World> init()
    void stopIndividualWorld()

    // Execução
    void execute(Runnable task)
    void consumeTaskQueue()
}
```

### WorldConfig

Configuração de um mundo.

```java
public class WorldConfig {
    // Sub-classes
    class ChunkConfig { }
    
    // Configurações específicas do mundo
}
```

---

## Sistema de Inventário

### Inventory

```java
public class Inventory {
    // Métodos de inventário do jogador
}
```

### ItemStack

Representa um item empilhado.

```java
public class ItemStack {
    // Codec
    static BuilderCodec<ItemStack> CODEC

    // Propriedades
    // ... métodos de manipulação de itens

    // Inner classes
    class Metadata { }
}
```

### ItemContainer

Container de itens genérico.

```java
public interface ItemContainer {
    // Métodos de container
    
    // Evento de mudança
    class ItemContainerChangeEvent implements IEvent<Void> { }
    
    // Dados temporários
    class TempItemData { }
}
```

### Tipos de Containers

- `SimpleItemContainer` - Container simples
- `EmptyItemContainer` - Container vazio
- `DelegateItemContainer` - Container delegado
- `ItemStackItemContainer` - Container baseado em ItemStack
- `CombinedItemContainer` - Container combinado

### Filtros de Slots

```java
// Tipos de filtros
class SlotFilter { }
class TagFilter { }
class ArmorSlotAddFilter { }
class NoDuplicateFilter { }
class ResourceFilter { }
class ItemSlotFilter { }
```

---

## Sistema de Permissões

### PermissionHolder

Interface para entidades com permissões.

```java
public interface PermissionHolder {
    boolean hasPermission(String permission)
    boolean hasPermission(String permission, boolean defaultValue)
}
```

### HytalePermissions

Classe de permissões padrão.

```java
public class HytalePermissions {
    // Constantes de permissões do sistema
}
```

### PermissionProvider

Interface para provedores de permissão.

```java
public interface PermissionProvider {
    // Métodos de consulta de permissão
}
```

---

## Matemática e Vetores

### Vector3d

Vetor 3D com precisão double.

```java
public class Vector3d {
    // Constantes
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

    // Campos
    double x, y, z

    // Construtores
    Vector3d()
    Vector3d(Vector3d other)
    Vector3d(Vector3i other)
    Vector3d(double x, double y, double z)
    Vector3d(float pitch, float yaw)
    Vector3d(Random random, double scale)

    // Getters e Setters
    double getX(), setX(double)
    double getY(), setY(double)
    double getZ(), setZ(double)

    // Atribuição
    Vector3d assign(Vector3d)
    Vector3d assign(double value)
    Vector3d assign(double[] values)
    Vector3d assign(double x, double y, double z)

    // Operações Matemáticas
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

    // Distância
    double distanceTo(Vector3d)
    double distanceTo(Vector3i)
    double distanceTo(double x, double y, double z)
    double distanceSquaredTo(Vector3d)
    double distanceSquaredTo(double x, double y, double z)

    // Normalização e Comprimento
    Vector3d normalize()
    double length()
    double squaredLength()
    Vector3d setLength(double length)
    Vector3d clampLength(double maxLength)

    // Rotação
    Vector3d rotateX(float angle)
    Vector3d rotateY(float angle)
    Vector3d rotateZ(float angle)

    // Arredondamento
    Vector3d floor()
    Vector3d ceil()
    Vector3d clipToZero(double threshold)

    // Verificações
    boolean closeToZero(double threshold)
    boolean isInside(int sizeX, int sizeY, int sizeZ)
    boolean isFinite()

    // Conversão
    Vector3i toVector3i()
    Vector3f toVector3f()
    Vector3d clone()

    // Métodos Estáticos
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

Vetor 3D com precisão float.

```java
public class Vector3f {
    float x, y, z
    
    // Construtores e métodos similares ao Vector3d
    // Usado principalmente para rotações
}
```

### Vector3i

Vetor 3D com precisão inteira.

```java
public class Vector3i {
    int x, y, z
    
    // Construtores e métodos similares
    // Usado para posições de blocos
}
```

### Transform

Representa transformação (posição + rotação).

```java
public class Transform {
    // Combina posição (Vector3d) e rotação (Vector3f)
}
```

### Location

Representa uma localização no mundo.

```java
public class Location {
    // Mundo + Posição
}
```

### Outros Vetores

- `Vector2d` - Vetor 2D double
- `Vector2i` - Vetor 2D int
- `Vector2l` - Vetor 2D long
- `Vector3l` - Vetor 3D long
- `Vector4d` - Vetor 4D double

---

## Componentes de Entidades

### TransformComponent

Componente de transformação (posição e rotação).

```java
public class TransformComponent implements Component<EntityStore> {
    // Codec
    static BuilderCodec<TransformComponent> CODEC
    static ComponentType<EntityStore, TransformComponent> getComponentType()

    // Construtores
    TransformComponent()
    TransformComponent(Vector3d position, Vector3f rotation)

    // Posição
    Vector3d getPosition()
    void setPosition(Vector3d position)
    void teleportPosition(Vector3d position)

    // Rotação
    Vector3f getRotation()
    void setRotation(Vector3f rotation)
    void teleportRotation(Vector3f rotation)

    // Transformação Completa
    Transform getTransform()

    // Chunk
    WorldChunk getChunk()
    Ref<ChunkStore> getChunkRef()
    void setChunkLocation(Ref<ChunkStore>, WorldChunk)
    void markChunkDirty(ComponentAccessor<EntityStore>)

    // Rede
    ModelTransform getSentTransform()

    // Clone
    TransformComponent clone()
}
```

### Outros Componentes de Entidade

```java
// Componentes disponíveis em com.hypixel.hytale.server.core.modules.entity.component

class BoundingBox          // Caixa de colisão
class DisplayNameComponent // Nome de exibição
class ModelComponent       // Modelo 3D
class HeadRotation         // Rotação da cabeça
class DynamicLight         // Luz dinâmica
class AudioComponent       // Áudio
class Invulnerable         // Invulnerabilidade
class Intangible           // Intangibilidade
class EntityScaleComponent // Escala da entidade
class PositionDataComponent // Dados de posição
class CollisionResultComponent // Resultado de colisão
class SnapshotBuffer       // Buffer de snapshot
class FromPrefab           // Origem de prefab
class FromWorldGen         // Origem de geração de mundo
class WorldGenId           // ID de geração
class PropComponent        // Componente de prop
class NewSpawnComponent    // Novo spawn
class Interactable         // Interagível
class RespondToHit         // Resposta a hit
class RotateObjectComponent // Rotação de objeto
class MovementAudioComponent // Áudio de movimento
class ActiveAnimationComponent // Animação ativa
class PersistentModel      // Modelo persistente
class PersistentDynamicLight // Luz dinâmica persistente
class HiddenFromAdventurePlayers // Oculto de jogadores adventure
```

---

## Sistema de Tarefas

### TaskRegistry

Registro de tarefas agendadas.

```java
public class TaskRegistry extends Registry<TaskRegistration> {
    // Registrar tarefa de CompletableFuture
    TaskRegistration registerTask(CompletableFuture<Void> task)

    // Registrar tarefa agendada
    TaskRegistration registerTask(ScheduledFuture<Void> task)
}
```

### Exemplo de Uso

```java
public class MeuPlugin extends JavaPlugin {
    
    @Override
    protected void start() {
        // Tarefa única
        CompletableFuture<Void> tarefa = CompletableFuture.runAsync(() -> {
            // Código da tarefa
        });
        getTaskRegistry().registerTask(tarefa);
        
        // Tarefa agendada
        ScheduledFuture<Void> tarefaAgendada = HytaleServer.SCHEDULED_EXECUTOR
            .scheduleAtFixedRate(() -> {
                // Código executado periodicamente
            }, 0, 1, TimeUnit.SECONDS);
        getTaskRegistry().registerTask(tarefaAgendada);
    }
}
```

---

## Geração de Mundo

### Pacote Principal

```
com.hypixel.hytale.builtin.hytalegenerator
```

### Componentes de Geração

#### Density Nodes

Nós para cálculo de densidade de terreno:

- `ConstantValueDensity` - Valor constante
- `Noise2dDensity` - Ruído 2D
- `Noise3dDensity` - Ruído 3D
- `SumDensity` - Soma de densidades
- `MultiplierDensity` - Multiplicação
- `MinDensity`, `MaxDensity` - Mínimo/Máximo
- `ClampDensity` - Limitação
- `GradientDensity` - Gradiente
- `SelectorDensity` - Seleção
- `CacheDensity` - Cache
- `TerrainDensity` - Densidade de terreno
- E muitos outros...

#### Material Providers

Provedores de materiais para geração:

- `MaterialProvider` - Interface base
- `ConstantMaterialProviderAsset`
- `WeightedMaterialProviderAsset`
- `FieldFunctionMaterialProviderAsset`
- `StripedMaterialProviderAsset`
- `TerrainDensityMaterialProviderAsset`

#### Biomes

Sistema de biomas:

- `BiomeAsset` - Asset de bioma
- `BiomeType` - Tipo de bioma
- `BiomeMap` - Mapa de biomas
- `SimpleBiomeMap` - Implementação simples

#### Props

Sistema de props (vegetação, decorações):

- `Prop` - Classe base
- `PrefabProp` - Prop de prefab
- `ClusterProp` - Cluster de props
- `DensityProp` - Prop baseado em densidade
- `BoxProp` - Prop de caixa
- `ColumnProp` - Prop de coluna

---

## Módulos Builtin

### Plugins Builtin Incluídos

O HytaleServer.jar inclui diversos plugins builtin:

#### HytaleGenerator
Gerador de mundo oficial do Hytale.

#### Teleporter
Sistema de teleporte.

```java
// Componentes
class Teleporter

// Interações
class TeleporterInteraction

// Páginas
class TeleporterSettingsPage
```

#### Memories (Memórias)
Sistema de memórias do jogo.

```java
// Componentes
class PlayerMemories
class Memory

// Páginas
class MemoriesPage
```

#### Portals
Sistema de portais.

```java
// Componentes
class PortalDevice
class VoidEvent
class VoidSpawner

// Interações
class EnterPortalInteraction
class ReturnPortalInteraction
```

#### Beds (Camas)
Sistema de camas e sono.

```java
// Recursos
class WorldSleep
class WorldSlumber

// Componentes
class PlayerSleep
class SleepTracker

// Interações
class BedInteraction
```

#### Deployables
Sistema de itens deployáveis (torretas, armadilhas).

```java
// Componentes
class DeployableComponent
class DeployableOwnerComponent
class DeployableProjectileComponent

// Configurações
class DeployableTurretConfig
class DeployableTrapConfig
class DeployableAoeConfig
```

#### Ambience
Sistema de ambiente e música.

```java
// Componentes
class AmbienceTracker
class AmbientEmitterComponent

// Recursos
class AmbienceResource
```

#### Creative Hub
Hub criativo.

```java
// Interações
class HubPortalInteraction

// Comandos
class HubCommand
```

---

## Comandos Builtin

O servidor inclui diversos comandos nativos:

### Comandos de Mundo

- `/world` - Gerenciamento de mundos
- `/worldconfig` - Configuração de mundo
- `/block` - Manipulação de blocos
- `/time` - Controle de tempo

### Comandos de Entidade

- `/spawn` - Spawn de entidades
- `/teleport` - Teleporte
- `/gamemode` - Modo de jogo

### Comandos de Permissão

- `/op` - Operador
- `/perm` - Permissões

### Comandos de Acesso

- `/whitelist` - Lista branca
- `/ban` - Banimento
- `/unban` - Desbanir

### Comandos de Debug

- `/debug` - Debugging
- `/hitbox` - Visualização de hitbox

### Comandos de Plugin

- `/plugin` - Gerenciamento de plugins

### Comandos de Interação

- `/interaction` - Sistema de interações

---

## Estrutura de Manifest

O arquivo `manifest.json` deve estar em `src/main/resources/`:

```json
{
    "id": "meu-plugin",
    "name": "Meu Plugin",
    "version": "1.0.0",
    "description": "Descrição do plugin",
    "authors": [
        {
            "name": "Autor",
            "role": "Desenvolvedor"
        }
    ],
    "main": "com.exemplo.MeuPlugin"
}
```

---

## Exemplo Completo de Plugin

```java
package com.exemplo;

import com.hypixel.hytale.server.core.plugin.JavaPlugin;
import com.hypixel.hytale.server.core.plugin.JavaPluginInit;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.event.events.player.PlayerConnectEvent;
import com.hypixel.hytale.event.EventPriority;

public class MeuPlugin extends JavaPlugin {

    public MeuPlugin(JavaPluginInit init) {
        super(init);
    }

    @Override
    protected void setup() {
        getLogger().info("Plugin em setup!");
    }

    @Override
    protected void start() {
        getLogger().info("Plugin iniciado!");
        
        // Registrar comandos
        getCommandRegistry().register(new MeuComando());
        
        // Registrar eventos
        getEventRegistry().register(
            EventPriority.NORMAL,
            PlayerConnectEvent.class,
            this::onPlayerConnect
        );
    }

    @Override
    protected void shutdown() {
        getLogger().info("Plugin desligando!");
    }

    private void onPlayerConnect(PlayerConnectEvent event) {
        getLogger().info("Jogador conectou ao servidor!");
    }

    // Comando interno
    public static class MeuComando extends CommandBase {
        
        public MeuComando() {
            super("meucomando", "Um comando de exemplo", false);
        }

        @Override
        protected void executeSync(CommandContext ctx) {
            ctx.sendMessage(Message.raw("Comando executado com sucesso!"));
        }
    }
}
```

---

## Referências Adicionais

### Pacotes de Terceiros Incluídos

O JAR inclui algumas bibliotecas de terceiros:

- `javax.annotation` - Anotações JSR-305
- `org.bouncycastle` - Criptografia

### Logging

O sistema usa `HytaleLogger`:

```java
HytaleLogger logger = getLogger();
logger.info("Mensagem informativa");
logger.warn("Aviso");
logger.error("Erro");
logger.debug("Debug");
```

### Codecs

O sistema usa codecs para serialização:

```java
// BuilderCodec para objetos complexos
BuilderCodec<MeuObjeto> CODEC = BuilderCodec.builder()
    .field("campo", ...)
    .build();
```

---

## Notas Importantes

1. **API Não Oficial**: Esta documentação foi gerada por análise do bytecode. A API pode mudar sem aviso.

2. **Sistema ECS**: O Hytale usa um sistema Entity-Component-System. Familiarize-se com `Component`, `Ref`, `Holder`, `ComponentAccessor`.

3. **Assíncrono**: Muitas operações são assíncronas e retornam `CompletableFuture`.

4. **Thread Safety**: Operações de mundo devem ser executadas na thread do mundo usando `world.execute(Runnable)`.

5. **Salvamento**: Chame `markNeedsSave()` para garantir que alterações sejam persistidas.

---

*Documentação gerada automaticamente através de análise do HytaleServer.jar*
*Versão do JAR analisado: Compatível com Java 25*
