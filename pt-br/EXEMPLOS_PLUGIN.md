# Exemplos de Plugin Hytale - Guia Prático

Este documento contém exemplos práticos de como desenvolver plugins para Hytale, baseado no projeto **FirstHytalePlugin**.

---

## Índice

1. [Estrutura do Plugin](#estrutura-do-plugin)
2. [Comandos Básicos](#comandos-básicos)
3. [Sistema de Teleporte](#sistema-de-teleporte)
4. [Sistema de Proteção de Áreas](#sistema-de-proteção-de-áreas)
5. [Padrões e Boas Práticas](#padrões-e-boas-práticas)
6. [Referência Rápida](#referência-rápida)

---

## Estrutura do Plugin

### Classe Principal do Plugin

Todo plugin Hytale precisa de uma classe que estende `JavaPlugin`:

```java
package me.alii;

import com.hypixel.hytale.server.core.command.system.CommandRegistry;
import com.hypixel.hytale.server.core.plugin.JavaPlugin;
import com.hypixel.hytale.server.core.plugin.JavaPluginInit;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

public class FirstPlugin extends JavaPlugin {

    public FirstPlugin(@NonNullDecl JavaPluginInit init) {
        super(init);
    }

    @Override
    protected void setup() {
        // Registra comandos e inicializa sistemas
        CommandRegistry commandRegistry = this.getCommandRegistry();
        
        // Registrar comandos
        commandRegistry.registerCommand(new MeuComando());
    }
}
```

### Arquivo manifest.json

O arquivo `src/main/resources/manifest.json` define as informações do plugin:

```json
{
    "id": "first-hytale-plugin",
    "name": "First Hytale Plugin",
    "version": "1.0.0",
    "description": "Plugin de exemplo para Hytale",
    "authors": [
        {
            "name": "Alii",
            "role": "Desenvolvedor"
        }
    ],
    "main": "me.alii.FirstPlugin"
}
```

---

## Comandos Básicos

### Comando Simples

O comando mais básico possível:

```java
package me.alii.commands;

import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

/**
 * Comando /hello - Envia uma mensagem simples
 */
public class HelloCommand extends CommandBase {

    public HelloCommand() {
        // (nome, descrição, requerJogador)
        super("hello", "Diz olá!", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        ctx.sendMessage(Message.raw("Olá, mundo!"));
    }
}
```

### Comando Apenas para Jogadores

```java
public class PlayerOnlyCommand extends CommandBase {

    public PlayerOnlyCommand() {
        super("info", "Mostra suas informações.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        // Verifica se é um jogador
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("Este comando só pode ser usado por jogadores!"));
            return;
        }

        // Obtém referência do jogador
        Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
        
        ctx.sendMessage(Message.raw("Você é um jogador válido!"));
    }
}
```

### Enviando Múltiplas Mensagens

```java
@Override
protected void executeSync(@NonNullDecl CommandContext ctx) {
    ctx.sendMessage(Message.raw("=== Menu de Ajuda ==="));
    ctx.sendMessage(Message.raw("/comando1 - Faz algo"));
    ctx.sendMessage(Message.raw("/comando2 - Faz outra coisa"));
    ctx.sendMessage(Message.raw(""));
    ctx.sendMessage(Message.raw("Dica: Use Tab para autocompletar!"));
}
```

---

## Sistema de Teleporte

O sistema de teleporte demonstra várias funcionalidades importantes da API.

### TeleportManager - Gerenciador Central

Padrão Singleton para gerenciar estados:

```java
package me.alii.teleport;

import java.util.*;

/**
 * Gerenciador central do sistema de teleporte.
 * Armazena requests de TPA, spawn e homes dos jogadores.
 */
public class TeleportManager {

    private static TeleportManager instance;

    // Armazena requests de TPA: <destinatário, remetente>
    private final Map<UUID, UUID> tpaRequests = new HashMap<>();

    // Armazena a localização do spawn global
    private double[] spawnLocation = null;

    // Armazena homes dos jogadores: <jogadorUUID, <nomeHome, coordenadas>>
    private final Map<UUID, Map<String, double[]>> playerHomes = new HashMap<>();

    // Armazena warps globais: <nomeWarp, coordenadas>
    private final Map<String, double[]> warps = new HashMap<>();

    // Tempo de expiração do TPA em milissegundos (30 segundos)
    private static final long TPA_EXPIRATION_TIME = 30000;
    private final Map<UUID, Long> tpaRequestTimes = new HashMap<>();

    private TeleportManager() {
        // Singleton privado
    }

    public static TeleportManager getInstance() {
        if (instance == null) {
            instance = new TeleportManager();
        }
        return instance;
    }

    // ==================== TPA SYSTEM ====================

    public void createTpaRequest(UUID sender, UUID target) {
        tpaRequests.put(target, sender);
        tpaRequestTimes.put(target, System.currentTimeMillis());
    }

    public Optional<UUID> getPendingTpaRequest(UUID target) {
        if (!tpaRequests.containsKey(target)) {
            return Optional.empty();
        }

        long requestTime = tpaRequestTimes.getOrDefault(target, 0L);
        if (System.currentTimeMillis() - requestTime > TPA_EXPIRATION_TIME) {
            removeTpaRequest(target);
            return Optional.empty();
        }

        return Optional.of(tpaRequests.get(target));
    }

    public void removeTpaRequest(UUID target) {
        tpaRequests.remove(target);
        tpaRequestTimes.remove(target);
    }

    // ==================== SPAWN SYSTEM ====================

    public void setSpawn(double x, double y, double z) {
        this.spawnLocation = new double[]{x, y, z};
    }

    public double[] getSpawn() {
        return spawnLocation;
    }

    public boolean hasSpawn() {
        return spawnLocation != null;
    }

    // ==================== HOME SYSTEM ====================

    public void setHome(UUID playerUUID, String homeName, double x, double y, double z) {
        playerHomes.computeIfAbsent(playerUUID, k -> new HashMap<>());
        playerHomes.get(playerUUID).put(homeName.toLowerCase(), new double[]{x, y, z});
    }

    public double[] getHome(UUID playerUUID, String homeName) {
        Map<String, double[]> homes = playerHomes.get(playerUUID);
        if (homes == null) return null;
        return homes.get(homeName.toLowerCase());
    }

    public Map<String, double[]> getHomes(UUID playerUUID) {
        return playerHomes.getOrDefault(playerUUID, new HashMap<>());
    }

    public boolean deleteHome(UUID playerUUID, String homeName) {
        Map<String, double[]> homes = playerHomes.get(playerUUID);
        if (homes == null) return false;
        return homes.remove(homeName.toLowerCase()) != null;
    }

    // ==================== WARP SYSTEM ====================

    public void setWarp(String warpName, double x, double y, double z) {
        warps.put(warpName.toLowerCase(), new double[]{x, y, z});
    }

    public double[] getWarp(String warpName) {
        return warps.get(warpName.toLowerCase());
    }

    public Map<String, double[]> getWarps() {
        return new HashMap<>(warps);
    }

    public boolean deleteWarp(String warpName) {
        return warps.remove(warpName.toLowerCase()) != null;
    }
}
```

### Comando /sethome - Salvar Posição

Demonstra como obter a posição do jogador:

```java
package me.alii.commands.teleport;

import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.math.vector.Vector3d;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import com.hypixel.hytale.server.core.modules.entity.component.TransformComponent;
import com.hypixel.hytale.server.core.universe.world.World;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;
import me.alii.teleport.TeleportManager;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

import java.util.UUID;

public class SetHomeCommand extends CommandBase {

    private static final int MAX_HOMES = 5;

    public SetHomeCommand() {
        super("sethome", "Salva sua posicao atual como home.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        // Verificação obrigatória
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("Este comando só pode ser usado por jogadores!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        
        // Obtém referências necessárias
        Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
        EntityStore entityStore = playerRef.getStore().getExternalData();
        World world = entityStore.getWorld();
        
        // TODO: Obter UUID real do jogador quando API disponível
        UUID playerUUID = UUID.fromString("00000000-0000-0000-0000-000000000001");
        String homeName = "home";

        // Verificar limite
        int currentHomes = manager.getHomes(playerUUID).size();
        boolean isUpdating = manager.getHome(playerUUID, homeName) != null;

        if (!isUpdating && currentHomes >= MAX_HOMES) {
            ctx.sendMessage(Message.raw("Você atingiu o limite de " + MAX_HOMES + " homes!"));
            ctx.sendMessage(Message.raw("Delete uma home com /delhome antes de criar outra."));
            return;
        }

        // IMPORTANTE: Executar na thread do mundo
        world.execute(() -> {
            // Obtém o componente de transformação
            TransformComponent transform = playerRef.getStore()
                .getComponent(playerRef, TransformComponent.getComponentType());
            
            // Obtém a posição
            Vector3d position = transform.getPosition();

            double x = position.x;
            double y = position.y;
            double z = position.z;

            // Salva a home
            manager.setHome(playerUUID, homeName, x, y, z);

            // Feedback ao jogador
            ctx.sendMessage(Message.raw("Home '" + homeName + "' salva com sucesso!"));
            ctx.sendMessage(Message.raw("Localização: X: " + (int)x + ", Y: " + (int)y + ", Z: " + (int)z));
            ctx.sendMessage(Message.raw("Use /home para teleportar de volta."));
        });
    }
}
```

### Comando /home - Teleportar

Demonstra como teleportar o jogador:

```java
package me.alii.commands.teleport;

import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.math.vector.Vector3d;
import com.hypixel.hytale.math.vector.Vector3f;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import com.hypixel.hytale.server.core.modules.entity.teleport.Teleport;
import com.hypixel.hytale.server.core.universe.world.World;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;
import me.alii.teleport.TeleportManager;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

import java.util.Map;
import java.util.UUID;

public class HomeCommand extends CommandBase {

    public HomeCommand() {
        super("home", "Teleporta para sua home salva.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("Este comando só pode ser usado por jogadores!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        
        Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
        EntityStore entityStore = playerRef.getStore().getExternalData();
        World world = entityStore.getWorld();
        
        UUID playerUUID = UUID.fromString("00000000-0000-0000-0000-000000000001");

        // Verifica se tem homes
        Map<String, double[]> homes = manager.getHomes(playerUUID);
        if (homes.isEmpty()) {
            ctx.sendMessage(Message.raw("Você não tem nenhuma home salva!"));
            ctx.sendMessage(Message.raw("Use /sethome para criar uma."));
            return;
        }

        // Obtém a home padrão
        double[] home = manager.getHome(playerUUID, "home");
        if (home == null) {
            ctx.sendMessage(Message.raw("Sua home padrão não existe!"));
            return;
        }

        // Teleporta na thread do mundo
        world.execute(() -> {
            // Cria o vetor de posição
            Vector3d homePosition = new Vector3d(home[0], home[1], home[2]);
            Vector3f rotation = new Vector3f();

            // Cria o componente de teleporte
            Teleport teleport = new Teleport(homePosition, rotation);
            
            // Adiciona o componente ao jogador (isso causa o teleporte)
            playerRef.getStore().addComponent(
                playerRef, 
                Teleport.getComponentType(), 
                teleport
            );

            ctx.sendMessage(Message.raw("Teleportado para home!"));
            ctx.sendMessage(Message.raw("X: " + (int)home[0] + ", Y: " + (int)home[1] + ", Z: " + (int)home[2]));
        });
    }
}
```

### Comando /warp - Listar e Teleportar

Demonstra lógica condicional:

```java
package me.alii.commands.teleport;

import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.math.vector.Vector3d;
import com.hypixel.hytale.math.vector.Vector3f;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import com.hypixel.hytale.server.core.modules.entity.teleport.Teleport;
import com.hypixel.hytale.server.core.universe.world.World;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;
import me.alii.teleport.TeleportManager;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

import java.util.Map;

public class WarpCommand extends CommandBase {

    public WarpCommand() {
        super("warp", "Lista as warps disponíveis ou teleporta para uma.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("Este comando só pode ser usado por jogadores!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        Map<String, double[]> warps = manager.getWarps();

        // Sem warps disponíveis
        if (warps.isEmpty()) {
            ctx.sendMessage(Message.raw("Não há warps disponíveis!"));
            ctx.sendMessage(Message.raw("Administradores podem criar warps com /setwarp"));
            return;
        }

        // Se tiver apenas uma warp, teleporta direto
        if (warps.size() == 1) {
            Map.Entry<String, double[]> entry = warps.entrySet().iterator().next();
            String warpName = entry.getKey();
            double[] loc = entry.getValue();
            
            Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
            EntityStore entityStore = playerRef.getStore().getExternalData();
            World world = entityStore.getWorld();
            
            world.execute(() -> {
                Vector3d warpPosition = new Vector3d(loc[0], loc[1], loc[2]);
                Vector3f rotation = new Vector3f();
                Teleport teleport = new Teleport(warpPosition, rotation);
                playerRef.getStore().addComponent(playerRef, Teleport.getComponentType(), teleport);
                
                ctx.sendMessage(Message.raw("Teleportado para warp '" + warpName + "'!"));
            });
        } else {
            // Lista todas as warps
            ctx.sendMessage(Message.raw("=== Warps Disponíveis ==="));
            for (Map.Entry<String, double[]> entry : warps.entrySet()) {
                double[] loc = entry.getValue();
                ctx.sendMessage(Message.raw(
                    entry.getKey() + " - X: " + (int)loc[0] + 
                    ", Y: " + (int)loc[1] + 
                    ", Z: " + (int)loc[2]
                ));
            }
        }
    }
}
```

### Comando /spawn - Teleporte para Spawn

```java
package me.alii.commands.teleport;

import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.math.vector.Vector3d;
import com.hypixel.hytale.math.vector.Vector3f;
import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import com.hypixel.hytale.server.core.modules.entity.teleport.Teleport;
import com.hypixel.hytale.server.core.universe.world.World;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;
import me.alii.teleport.TeleportManager;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

public class SpawnCommand extends CommandBase {

    public SpawnCommand() {
        super("spawn", "Teleporta para o spawn do servidor.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("Este comando só pode ser usado por jogadores!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        double[] spawn = manager.getSpawn();

        if (spawn == null) {
            ctx.sendMessage(Message.raw("O spawn ainda não foi definido!"));
            ctx.sendMessage(Message.raw("Administradores podem definir com /setspawn"));
            return;
        }

        Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
        EntityStore entityStore = playerRef.getStore().getExternalData();
        World world = entityStore.getWorld();

        world.execute(() -> {
            Vector3d spawnPosition = new Vector3d(spawn[0], spawn[1], spawn[2]);
            Vector3f rotation = new Vector3f();
            Teleport teleport = new Teleport(spawnPosition, rotation);
            playerRef.getStore().addComponent(playerRef, Teleport.getComponentType(), teleport);

            ctx.sendMessage(Message.raw("Teleportado para o spawn!"));
        });
    }
}
```

---

## Sistema de Proteção de Áreas

### ProtectionManager - Gerenciador de Regiões

```java
package me.alii.protection;

import java.util.*;

/**
 * Gerenciador de proteção de áreas (claims)
 */
public class ProtectionManager {

    private static ProtectionManager instance;

    private final Map<String, List<ProtectedRegion>> regions = new HashMap<>();
    private int defaultClaimRadius = 16;
    private int maxClaimsPerPlayer = 5;

    private ProtectionManager() {}

    public static ProtectionManager getInstance() {
        if (instance == null) {
            instance = new ProtectionManager();
        }
        return instance;
    }

    /**
     * Cria um claim rápido ao redor do jogador
     */
    public ProtectedRegion createQuickClaim(UUID owner, String world, int centerX, int centerZ) {
        int minX = centerX - defaultClaimRadius;
        int maxX = centerX + defaultClaimRadius;
        int minZ = centerZ - defaultClaimRadius;
        int maxZ = centerZ + defaultClaimRadius;

        // Verifica sobreposição
        if (hasOverlap(world, minX, minZ, maxX, maxZ)) {
            return null;
        }

        String regionId = "claim_" + owner.toString().substring(0, 8) + "_" + System.currentTimeMillis();
        ProtectedRegion region = new ProtectedRegion(regionId, owner, world, minX, minZ, maxX, maxZ);

        regions.computeIfAbsent(world, k -> new ArrayList<>()).add(region);
        return region;
    }

    /**
     * Verifica se há sobreposição com outras regiões
     */
    public boolean hasOverlap(String world, int minX, int minZ, int maxX, int maxZ) {
        List<ProtectedRegion> worldRegions = regions.get(world);
        if (worldRegions == null) return false;

        for (ProtectedRegion region : worldRegions) {
            if (region.overlaps(minX, minZ, maxX, maxZ)) {
                return true;
            }
        }
        return false;
    }

    /**
     * Obtém a região em uma posição
     */
    public ProtectedRegion getRegionAt(String world, int x, int z) {
        List<ProtectedRegion> worldRegions = regions.get(world);
        if (worldRegions == null) return null;

        for (ProtectedRegion region : worldRegions) {
            if (region.contains(x, z)) {
                return region;
            }
        }
        return null;
    }

    /**
     * Conta os claims de um jogador
     */
    public int getPlayerClaimCount(UUID playerUUID) {
        int count = 0;
        for (List<ProtectedRegion> worldRegions : regions.values()) {
            for (ProtectedRegion region : worldRegions) {
                if (region.getOwner().equals(playerUUID)) {
                    count++;
                }
            }
        }
        return count;
    }

    // Getters e Setters
    public int getDefaultClaimRadius() { return defaultClaimRadius; }
    public int getMaxClaimsPerPlayer() { return maxClaimsPerPlayer; }
}
```

### ProtectedRegion - Classe de Região

```java
package me.alii.protection;

import java.util.*;

/**
 * Representa uma região protegida no mundo
 */
public class ProtectedRegion {

    private final String id;
    private final UUID owner;
    private final String world;
    private final int minX, minZ, maxX, maxZ;
    private final Set<UUID> trustedPlayers = new HashSet<>();
    private final long createdAt;

    public ProtectedRegion(String id, UUID owner, String world, 
                          int minX, int minZ, int maxX, int maxZ) {
        this.id = id;
        this.owner = owner;
        this.world = world;
        this.minX = Math.min(minX, maxX);
        this.maxX = Math.max(minX, maxX);
        this.minZ = Math.min(minZ, maxZ);
        this.maxZ = Math.max(minZ, maxZ);
        this.createdAt = System.currentTimeMillis();
    }

    /**
     * Verifica se um ponto está dentro da região
     */
    public boolean contains(int x, int z) {
        return x >= minX && x <= maxX && z >= minZ && z <= maxZ;
    }

    /**
     * Verifica sobreposição com outra área
     */
    public boolean overlaps(int otherMinX, int otherMinZ, int otherMaxX, int otherMaxZ) {
        return !(otherMaxX < minX || otherMinX > maxX || 
                 otherMaxZ < minZ || otherMinZ > maxZ);
    }

    /**
     * Verifica se um jogador pode construir
     */
    public boolean canBuild(UUID playerUUID) {
        return owner.equals(playerUUID) || trustedPlayers.contains(playerUUID);
    }

    /**
     * Adiciona um jogador de confiança
     */
    public boolean trust(UUID playerUUID) {
        if (owner.equals(playerUUID)) return false;
        return trustedPlayers.add(playerUUID);
    }

    /**
     * Remove um jogador de confiança
     */
    public boolean untrust(UUID playerUUID) {
        return trustedPlayers.remove(playerUUID);
    }

    // Getters
    public String getId() { return id; }
    public UUID getOwner() { return owner; }
    public String getWorld() { return world; }
    public int getMinX() { return minX; }
    public int getMaxX() { return maxX; }
    public int getMinZ() { return minZ; }
    public int getMaxZ() { return maxZ; }
    public Set<UUID> getTrustedPlayers() { return new HashSet<>(trustedPlayers); }
    public long getCreatedAt() { return createdAt; }
}
```

### Comando /claim - Proteger Área

```java
package me.alii.commands.protection;

import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import me.alii.protection.ProtectedRegion;
import me.alii.protection.ProtectionManager;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

import java.util.UUID;

public class ClaimCommand extends CommandBase {

    public ClaimCommand() {
        super("claim", "Protege a área ao seu redor.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        ProtectionManager manager = ProtectionManager.getInstance();
        
        // TODO: Obter UUID real do jogador
        UUID playerUUID = UUID.fromString("00000000-0000-0000-0000-000000000001");
        
        // TODO: Obter posição real do jogador
        int x = 0, y = 64, z = 0;
        String worldName = "world";

        // Verifica limite de claims
        int currentClaims = manager.getPlayerClaimCount(playerUUID);
        int maxClaims = manager.getMaxClaimsPerPlayer();

        if (currentClaims >= maxClaims) {
            ctx.sendMessage(Message.raw("Você atingiu o limite de " + maxClaims + " claims!"));
            ctx.sendMessage(Message.raw("Use /unclaim para remover um claim antigo."));
            return;
        }

        // Cria o claim
        ProtectedRegion region = manager.createQuickClaim(playerUUID, worldName, x, z);

        if (region == null) {
            ctx.sendMessage(Message.raw("Não foi possível criar o claim!"));
            ctx.sendMessage(Message.raw("Pode haver sobreposição com outra área protegida."));
            return;
        }

        int radius = manager.getDefaultClaimRadius();

        ctx.sendMessage(Message.raw("=== Área Protegida! ==="));
        ctx.sendMessage(Message.raw("Raio de proteção: " + radius + " blocos"));
        ctx.sendMessage(Message.raw("De (" + region.getMinX() + ", " + region.getMinZ() + ")"));
        ctx.sendMessage(Message.raw("Até (" + region.getMaxX() + ", " + region.getMaxZ() + ")"));
        ctx.sendMessage(Message.raw("Claims: " + (currentClaims + 1) + "/" + maxClaims));
        ctx.sendMessage(Message.raw(""));
        ctx.sendMessage(Message.raw("Use /trust <jogador> para permitir construção"));
        ctx.sendMessage(Message.raw("Use /unclaim para remover a proteção"));
    }
}
```

---

## Padrões e Boas Práticas

### 1. Sempre Verificar se é Jogador

```java
@Override
protected void executeSync(@NonNullDecl CommandContext ctx) {
    if (!ctx.isPlayer()) {
        ctx.sendMessage(Message.raw("Este comando só pode ser usado por jogadores!"));
        return;
    }
    // ...
}
```

### 2. Executar na Thread do Mundo

Quando manipular entidades, sempre use `world.execute()`:

```java
World world = entityStore.getWorld();

world.execute(() -> {
    // Código que manipula entidades vai aqui
    TransformComponent transform = playerRef.getStore()
        .getComponent(playerRef, TransformComponent.getComponentType());
    // ...
});
```

### 3. Usar Padrão Singleton para Managers

```java
public class MeuManager {
    private static MeuManager instance;
    
    private MeuManager() { }
    
    public static MeuManager getInstance() {
        if (instance == null) {
            instance = new MeuManager();
        }
        return instance;
    }
}
```

### 4. Feedback Claro ao Usuário

```java
// ✅ Bom - Mensagens claras e informativas
ctx.sendMessage(Message.raw("=== Home Salva ==="));
ctx.sendMessage(Message.raw("Nome: " + homeName));
ctx.sendMessage(Message.raw("Localização: X: " + x + ", Y: " + y + ", Z: " + z));
ctx.sendMessage(Message.raw("Use /home para teleportar de volta."));

// ❌ Ruim - Mensagem vaga
ctx.sendMessage(Message.raw("OK"));
```

### 5. Validar Limites

```java
private static final int MAX_HOMES = 5;

// Verificar antes de adicionar
int currentHomes = manager.getHomes(playerUUID).size();
if (currentHomes >= MAX_HOMES) {
    ctx.sendMessage(Message.raw("Limite de " + MAX_HOMES + " homes atingido!"));
    return;
}
```

---

## Referência Rápida

### Imports Comuns

```java
// Comandos
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;

// Mensagens
import com.hypixel.hytale.server.core.Message;

// Entidades e Mundo
import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.server.core.universe.world.World;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;

// Componentes
import com.hypixel.hytale.server.core.modules.entity.component.TransformComponent;
import com.hypixel.hytale.server.core.modules.entity.teleport.Teleport;

// Vetores
import com.hypixel.hytale.math.vector.Vector3d;
import com.hypixel.hytale.math.vector.Vector3f;

// Plugin
import com.hypixel.hytale.server.core.plugin.JavaPlugin;
import com.hypixel.hytale.server.core.plugin.JavaPluginInit;
import com.hypixel.hytale.server.core.command.system.CommandRegistry;
```

### Snippets Úteis

#### Obter Posição do Jogador

```java
Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
World world = playerRef.getStore().getExternalData().getWorld();

world.execute(() -> {
    TransformComponent transform = playerRef.getStore()
        .getComponent(playerRef, TransformComponent.getComponentType());
    Vector3d position = transform.getPosition();
    
    double x = position.x;
    double y = position.y;
    double z = position.z;
});
```

#### Teleportar Jogador

```java
world.execute(() -> {
    Vector3d destination = new Vector3d(x, y, z);
    Vector3f rotation = new Vector3f(); // Manter rotação atual
    
    Teleport teleport = new Teleport(destination, rotation);
    playerRef.getStore().addComponent(
        playerRef, 
        Teleport.getComponentType(), 
        teleport
    );
});
```

#### Registrar Comandos no Plugin

```java
@Override
protected void setup() {
    CommandRegistry cmd = this.getCommandRegistry();
    
    cmd.registerCommand(new Comando1());
    cmd.registerCommand(new Comando2());
    cmd.registerCommand(new Comando3());
}
```

---

## Lista de Comandos do Projeto

| Comando | Descrição | Classe |
|---------|-----------|--------|
| `/tp` | Teleporte direto | `TpCommand` |
| `/tpa` | Request de teleporte | `TpaCommand` |
| `/tpaccept` | Aceitar TPA | `TpAcceptCommand` |
| `/tpdeny` | Recusar TPA | `TpDenyCommand` |
| `/spawn` | Ir para spawn | `SpawnCommand` |
| `/setspawn` | Definir spawn | `SetSpawnCommand` |
| `/home` | Ir para home | `HomeCommand` |
| `/sethome` | Definir home | `SetHomeCommand` |
| `/delhome` | Deletar home | `DelHomeCommand` |
| `/warp` | Listar/ir para warp | `WarpCommand` |
| `/setwarp` | Criar warp | `SetWarpCommand` |
| `/delwarp` | Deletar warp | `DelWarpCommand` |
| `/claim` | Proteger área | `ClaimCommand` |
| `/unclaim` | Remover proteção | `UnclaimCommand` |
| `/trust` | Confiar jogador | `TrustCommand` |
| `/untrust` | Remover confiança | `UntrustCommand` |
| `/claims` | Listar claims | `ClaimsCommand` |
| `/claiminfo` | Info do claim | `ClaimInfoCommand` |

---

## Estrutura de Diretórios Recomendada

```
src/main/java/
└── me/alii/
    ├── FirstPlugin.java          # Classe principal
    ├── commands/
    │   ├── FirstCommand.java     # Comando exemplo
    │   ├── teleport/             # Comandos de teleporte
    │   │   ├── TpCommand.java
    │   │   ├── HomeCommand.java
    │   │   └── ...
    │   └── protection/           # Comandos de proteção
    │       ├── ClaimCommand.java
    │       └── ...
    ├── teleport/
    │   └── TeleportManager.java  # Gerenciador de teleporte
    └── protection/
        ├── ProtectionManager.java
        └── ProtectedRegion.java

src/main/resources/
└── manifest.json                 # Manifest do plugin
```

---

*Documentação criada para o projeto FirstHytalePlugin*
