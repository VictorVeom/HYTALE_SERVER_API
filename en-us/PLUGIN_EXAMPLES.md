# Hytale Plugin Examples - Practical Guide

This document contains practical examples of how to develop plugins for Hytale, based on the **FirstHytalePlugin** project.

---

## Table of Contents

1. [Plugin Structure](#plugin-structure)
2. [Basic Commands](#basic-commands)
3. [Teleport System](#teleport-system)
4. [Area Protection System](#area-protection-system)
5. [Patterns and Best Practices](#patterns-and-best-practices)
6. [Quick Reference](#quick-reference)

---

## Plugin Structure

### Main Plugin Class

Every Hytale plugin needs a class that extends `JavaPlugin`:

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
        // Register commands and initialize systems
        CommandRegistry commandRegistry = this.getCommandRegistry();
        
        // Register commands
        commandRegistry.registerCommand(new MyCommand());
    }
}
```

### manifest.json File

The `src/main/resources/manifest.json` file defines plugin information:

```json
{
    "id": "first-hytale-plugin",
    "name": "First Hytale Plugin",
    "version": "1.0.0",
    "description": "Example plugin for Hytale",
    "authors": [
        {
            "name": "Alii",
            "role": "Developer"
        }
    ],
    "main": "me.alii.FirstPlugin"
}
```

---

## Basic Commands

### Simple Command

The most basic command possible:

```java
package me.alii.commands;

import com.hypixel.hytale.server.core.Message;
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;
import org.checkerframework.checker.nullness.compatqual.NonNullDecl;

/**
 * /hello command - Sends a simple message
 */
public class HelloCommand extends CommandBase {

    public HelloCommand() {
        // (name, description, requiresPlayer)
        super("hello", "Says hello!", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        ctx.sendMessage(Message.raw("Hello, world!"));
    }
}
```

### Player-Only Command

```java
public class PlayerOnlyCommand extends CommandBase {

    public PlayerOnlyCommand() {
        super("info", "Shows your information.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        // Check if sender is a player
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("This command can only be used by players!"));
            return;
        }

        // Get player reference
        Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
        
        ctx.sendMessage(Message.raw("You are a valid player!"));
    }
}
```

### Sending Multiple Messages

```java
@Override
protected void executeSync(@NonNullDecl CommandContext ctx) {
    ctx.sendMessage(Message.raw("=== Help Menu ==="));
    ctx.sendMessage(Message.raw("/command1 - Does something"));
    ctx.sendMessage(Message.raw("/command2 - Does something else"));
    ctx.sendMessage(Message.raw(""));
    ctx.sendMessage(Message.raw("Tip: Use Tab for autocomplete!"));
}
```

---

## Teleport System

The teleport system demonstrates several important API features.

### TeleportManager - Central Manager

Singleton pattern for state management:

```java
package me.alii.teleport;

import java.util.*;

/**
 * Central manager for the teleport system.
 * Stores TPA requests, spawn, and player homes.
 */
public class TeleportManager {

    private static TeleportManager instance;

    // Stores TPA requests: <recipient, sender>
    private final Map<UUID, UUID> tpaRequests = new HashMap<>();

    // Stores global spawn location
    private double[] spawnLocation = null;

    // Stores player homes: <playerUUID, <homeName, coordinates>>
    private final Map<UUID, Map<String, double[]>> playerHomes = new HashMap<>();

    // Stores global warps: <warpName, coordinates>
    private final Map<String, double[]> warps = new HashMap<>();

    // TPA expiration time in milliseconds (30 seconds)
    private static final long TPA_EXPIRATION_TIME = 30000;
    private final Map<UUID, Long> tpaRequestTimes = new HashMap<>();

    private TeleportManager() {
        // Private singleton constructor
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

### /sethome Command - Save Position

Demonstrates how to get the player's position:

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
        super("sethome", "Saves your current position as home.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        // Required verification
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("This command can only be used by players!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        
        // Get required references
        Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
        EntityStore entityStore = playerRef.getStore().getExternalData();
        World world = entityStore.getWorld();
        
        // TODO: Get real player UUID when API available
        UUID playerUUID = UUID.fromString("00000000-0000-0000-0000-000000000001");
        String homeName = "home";

        // Check limit
        int currentHomes = manager.getHomes(playerUUID).size();
        boolean isUpdating = manager.getHome(playerUUID, homeName) != null;

        if (!isUpdating && currentHomes >= MAX_HOMES) {
            ctx.sendMessage(Message.raw("You have reached the limit of " + MAX_HOMES + " homes!"));
            ctx.sendMessage(Message.raw("Delete a home with /delhome before creating another."));
            return;
        }

        // IMPORTANT: Execute on the world thread
        world.execute(() -> {
            // Get the transform component
            TransformComponent transform = playerRef.getStore()
                .getComponent(playerRef, TransformComponent.getComponentType());
            
            // Get position
            Vector3d position = transform.getPosition();

            double x = position.x;
            double y = position.y;
            double z = position.z;

            // Save the home
            manager.setHome(playerUUID, homeName, x, y, z);

            // Player feedback
            ctx.sendMessage(Message.raw("Home '" + homeName + "' saved successfully!"));
            ctx.sendMessage(Message.raw("Location: X: " + (int)x + ", Y: " + (int)y + ", Z: " + (int)z));
            ctx.sendMessage(Message.raw("Use /home to teleport back."));
        });
    }
}
```

### /home Command - Teleport

Demonstrates how to teleport the player:

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
        super("home", "Teleports to your saved home.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("This command can only be used by players!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        
        Ref<EntityStore> playerRef = ctx.senderAsPlayerRef();
        EntityStore entityStore = playerRef.getStore().getExternalData();
        World world = entityStore.getWorld();
        
        UUID playerUUID = UUID.fromString("00000000-0000-0000-0000-000000000001");

        // Check if player has homes
        Map<String, double[]> homes = manager.getHomes(playerUUID);
        if (homes.isEmpty()) {
            ctx.sendMessage(Message.raw("You don't have any saved homes!"));
            ctx.sendMessage(Message.raw("Use /sethome to create one."));
            return;
        }

        // Get default home
        double[] home = manager.getHome(playerUUID, "home");
        if (home == null) {
            ctx.sendMessage(Message.raw("Your default home doesn't exist!"));
            return;
        }

        // Teleport on the world thread
        world.execute(() -> {
            // Create position vector
            Vector3d homePosition = new Vector3d(home[0], home[1], home[2]);
            Vector3f rotation = new Vector3f();

            // Create teleport component
            Teleport teleport = new Teleport(homePosition, rotation);
            
            // Add component to player (this causes the teleport)
            playerRef.getStore().addComponent(
                playerRef, 
                Teleport.getComponentType(), 
                teleport
            );

            ctx.sendMessage(Message.raw("Teleported to home!"));
            ctx.sendMessage(Message.raw("X: " + (int)home[0] + ", Y: " + (int)home[1] + ", Z: " + (int)home[2]));
        });
    }
}
```

### /warp Command - List and Teleport

Demonstrates conditional logic:

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
        super("warp", "Lists available warps or teleports to one.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("This command can only be used by players!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        Map<String, double[]> warps = manager.getWarps();

        // No warps available
        if (warps.isEmpty()) {
            ctx.sendMessage(Message.raw("No warps available!"));
            ctx.sendMessage(Message.raw("Administrators can create warps with /setwarp"));
            return;
        }

        // If there's only one warp, teleport directly
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
                
                ctx.sendMessage(Message.raw("Teleported to warp '" + warpName + "'!"));
            });
        } else {
            // List all warps
            ctx.sendMessage(Message.raw("=== Available Warps ==="));
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

### /spawn Command - Teleport to Spawn

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
        super("spawn", "Teleports to the server spawn.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        if (!ctx.isPlayer()) {
            ctx.sendMessage(Message.raw("This command can only be used by players!"));
            return;
        }

        TeleportManager manager = TeleportManager.getInstance();
        double[] spawn = manager.getSpawn();

        if (spawn == null) {
            ctx.sendMessage(Message.raw("Spawn has not been set yet!"));
            ctx.sendMessage(Message.raw("Administrators can set it with /setspawn"));
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

            ctx.sendMessage(Message.raw("Teleported to spawn!"));
        });
    }
}
```

---

## Area Protection System

### ProtectionManager - Region Manager

```java
package me.alii.protection;

import java.util.*;

/**
 * Manager for area protection (claims)
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
     * Creates a quick claim around the player
     */
    public ProtectedRegion createQuickClaim(UUID owner, String world, int centerX, int centerZ) {
        int minX = centerX - defaultClaimRadius;
        int maxX = centerX + defaultClaimRadius;
        int minZ = centerZ - defaultClaimRadius;
        int maxZ = centerZ + defaultClaimRadius;

        // Check for overlap
        if (hasOverlap(world, minX, minZ, maxX, maxZ)) {
            return null;
        }

        String regionId = "claim_" + owner.toString().substring(0, 8) + "_" + System.currentTimeMillis();
        ProtectedRegion region = new ProtectedRegion(regionId, owner, world, minX, minZ, maxX, maxZ);

        regions.computeIfAbsent(world, k -> new ArrayList<>()).add(region);
        return region;
    }

    /**
     * Checks for overlap with other regions
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
     * Gets the region at a position
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
     * Counts a player's claims
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

    // Getters and Setters
    public int getDefaultClaimRadius() { return defaultClaimRadius; }
    public int getMaxClaimsPerPlayer() { return maxClaimsPerPlayer; }
}
```

### ProtectedRegion - Region Class

```java
package me.alii.protection;

import java.util.*;

/**
 * Represents a protected region in the world
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
     * Checks if a point is inside the region
     */
    public boolean contains(int x, int z) {
        return x >= minX && x <= maxX && z >= minZ && z <= maxZ;
    }

    /**
     * Checks overlap with another area
     */
    public boolean overlaps(int otherMinX, int otherMinZ, int otherMaxX, int otherMaxZ) {
        return !(otherMaxX < minX || otherMinX > maxX || 
                 otherMaxZ < minZ || otherMinZ > maxZ);
    }

    /**
     * Checks if a player can build
     */
    public boolean canBuild(UUID playerUUID) {
        return owner.equals(playerUUID) || trustedPlayers.contains(playerUUID);
    }

    /**
     * Adds a trusted player
     */
    public boolean trust(UUID playerUUID) {
        if (owner.equals(playerUUID)) return false;
        return trustedPlayers.add(playerUUID);
    }

    /**
     * Removes a trusted player
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

### /claim Command - Protect Area

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
        super("claim", "Protects the area around you.", false);
    }

    @Override
    protected void executeSync(@NonNullDecl CommandContext ctx) {
        ProtectionManager manager = ProtectionManager.getInstance();
        
        // TODO: Get real player UUID
        UUID playerUUID = UUID.fromString("00000000-0000-0000-0000-000000000001");
        
        // TODO: Get real player position
        int x = 0, y = 64, z = 0;
        String worldName = "world";

        // Check claim limit
        int currentClaims = manager.getPlayerClaimCount(playerUUID);
        int maxClaims = manager.getMaxClaimsPerPlayer();

        if (currentClaims >= maxClaims) {
            ctx.sendMessage(Message.raw("You have reached the limit of " + maxClaims + " claims!"));
            ctx.sendMessage(Message.raw("Use /unclaim to remove an old claim."));
            return;
        }

        // Create the claim
        ProtectedRegion region = manager.createQuickClaim(playerUUID, worldName, x, z);

        if (region == null) {
            ctx.sendMessage(Message.raw("Could not create the claim!"));
            ctx.sendMessage(Message.raw("There may be overlap with another protected area."));
            return;
        }

        int radius = manager.getDefaultClaimRadius();

        ctx.sendMessage(Message.raw("=== Area Protected! ==="));
        ctx.sendMessage(Message.raw("Protection radius: " + radius + " blocks"));
        ctx.sendMessage(Message.raw("From (" + region.getMinX() + ", " + region.getMinZ() + ")"));
        ctx.sendMessage(Message.raw("To (" + region.getMaxX() + ", " + region.getMaxZ() + ")"));
        ctx.sendMessage(Message.raw("Claims: " + (currentClaims + 1) + "/" + maxClaims));
        ctx.sendMessage(Message.raw(""));
        ctx.sendMessage(Message.raw("Use /trust <player> to allow building"));
        ctx.sendMessage(Message.raw("Use /unclaim to remove protection"));
    }
}
```

---

## Patterns and Best Practices

### 1. Always Check if Sender is Player

```java
@Override
protected void executeSync(@NonNullDecl CommandContext ctx) {
    if (!ctx.isPlayer()) {
        ctx.sendMessage(Message.raw("This command can only be used by players!"));
        return;
    }
    // ...
}
```

### 2. Execute on World Thread

When manipulating entities, always use `world.execute()`:

```java
World world = entityStore.getWorld();

world.execute(() -> {
    // Entity manipulation code goes here
    TransformComponent transform = playerRef.getStore()
        .getComponent(playerRef, TransformComponent.getComponentType());
    // ...
});
```

### 3. Use Singleton Pattern for Managers

```java
public class MyManager {
    private static MyManager instance;
    
    private MyManager() { }
    
    public static MyManager getInstance() {
        if (instance == null) {
            instance = new MyManager();
        }
        return instance;
    }
}
```

### 4. Clear User Feedback

```java
// ✅ Good - Clear and informative messages
ctx.sendMessage(Message.raw("=== Home Saved ==="));
ctx.sendMessage(Message.raw("Name: " + homeName));
ctx.sendMessage(Message.raw("Location: X: " + x + ", Y: " + y + ", Z: " + z));
ctx.sendMessage(Message.raw("Use /home to teleport back."));

// ❌ Bad - Vague message
ctx.sendMessage(Message.raw("OK"));
```

### 5. Validate Limits

```java
private static final int MAX_HOMES = 5;

// Check before adding
int currentHomes = manager.getHomes(playerUUID).size();
if (currentHomes >= MAX_HOMES) {
    ctx.sendMessage(Message.raw("Limit of " + MAX_HOMES + " homes reached!"));
    return;
}
```

---

## Quick Reference

### Common Imports

```java
// Commands
import com.hypixel.hytale.server.core.command.system.CommandContext;
import com.hypixel.hytale.server.core.command.system.basecommands.CommandBase;

// Messages
import com.hypixel.hytale.server.core.Message;

// Entities and World
import com.hypixel.hytale.component.Ref;
import com.hypixel.hytale.server.core.universe.world.World;
import com.hypixel.hytale.server.core.universe.world.storage.EntityStore;

// Components
import com.hypixel.hytale.server.core.modules.entity.component.TransformComponent;
import com.hypixel.hytale.server.core.modules.entity.teleport.Teleport;

// Vectors
import com.hypixel.hytale.math.vector.Vector3d;
import com.hypixel.hytale.math.vector.Vector3f;

// Plugin
import com.hypixel.hytale.server.core.plugin.JavaPlugin;
import com.hypixel.hytale.server.core.plugin.JavaPluginInit;
import com.hypixel.hytale.server.core.command.system.CommandRegistry;
```

### Useful Snippets

#### Get Player Position

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

#### Teleport Player

```java
world.execute(() -> {
    Vector3d destination = new Vector3d(x, y, z);
    Vector3f rotation = new Vector3f(); // Keep current rotation
    
    Teleport teleport = new Teleport(destination, rotation);
    playerRef.getStore().addComponent(
        playerRef, 
        Teleport.getComponentType(), 
        teleport
    );
});
```

#### Register Commands in Plugin

```java
@Override
protected void setup() {
    CommandRegistry cmd = this.getCommandRegistry();
    
    cmd.registerCommand(new Command1());
    cmd.registerCommand(new Command2());
    cmd.registerCommand(new Command3());
}
```

---

## Project Command List

| Command | Description | Class |
|---------|-------------|-------|
| `/tp` | Direct teleport | `TpCommand` |
| `/tpa` | Teleport request | `TpaCommand` |
| `/tpaccept` | Accept TPA | `TpAcceptCommand` |
| `/tpdeny` | Deny TPA | `TpDenyCommand` |
| `/spawn` | Go to spawn | `SpawnCommand` |
| `/setspawn` | Set spawn | `SetSpawnCommand` |
| `/home` | Go to home | `HomeCommand` |
| `/sethome` | Set home | `SetHomeCommand` |
| `/delhome` | Delete home | `DelHomeCommand` |
| `/warp` | List/go to warp | `WarpCommand` |
| `/setwarp` | Create warp | `SetWarpCommand` |
| `/delwarp` | Delete warp | `DelWarpCommand` |
| `/claim` | Protect area | `ClaimCommand` |
| `/unclaim` | Remove protection | `UnclaimCommand` |
| `/trust` | Trust player | `TrustCommand` |
| `/untrust` | Remove trust | `UntrustCommand` |
| `/claims` | List claims | `ClaimsCommand` |
| `/claiminfo` | Claim info | `ClaimInfoCommand` |

---

## Recommended Directory Structure

```
src/main/java/
└── me/alii/
    ├── FirstPlugin.java          # Main class
    ├── commands/
    │   ├── FirstCommand.java     # Example command
    │   ├── teleport/             # Teleport commands
    │   │   ├── TpCommand.java
    │   │   ├── HomeCommand.java
    │   │   └── ...
    │   └── protection/           # Protection commands
    │       ├── ClaimCommand.java
    │       └── ...
    ├── teleport/
    │   └── TeleportManager.java  # Teleport manager
    └── protection/
        ├── ProtectionManager.java
        └── ProtectedRegion.java

src/main/resources/
└── manifest.json                 # Plugin manifest
```

---

*Documentation created for the FirstHytalePlugin project*
