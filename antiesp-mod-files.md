# Anti-ESP mod (Fabric 1.21.11) - paste these files into GitHub

How to use: in your GitHub repository tap **Add file > Create new file**. Type the file path exactly as shown in the heading (slashes make folders), paste the code below it, then tap **Commit changes**. There are 10 files.

When the last file is committed, open the **Actions** tab. The "build" run starts by itself. When it turns green, open it and download **antiesp-mod** from the Artifacts section. Unzip it and put the .jar in your `mods` folder together with Fabric API.

What it does: below Y=0 in the Overworld, blocks that are fully buried in solid rock are sent to clients as deepslate. Buried chests, spawners and other block entities are hidden too. Real blocks are sent back when a player comes within 10 blocks. Not included yet: hiding players and mobs.

To change settings, edit the numbers at the top of `Hider.java` (cutoff height) and `AntiEspMod.java` (reveal radius, check speed).

---

## 1. settings.gradle

```groovy
pluginManagement {
    repositories {
        maven {
            name = 'Fabric'
            url = 'https://maven.fabricmc.net/'
        }
        mavenCentral()
        gradlePluginPortal()
    }
}
```

## 2. build.gradle

```groovy
plugins {
    id 'fabric-loom' version '1.14-SNAPSHOT'
    id 'maven-publish'
}

version = project.mod_version
group = project.maven_group

base {
    archivesName = project.archives_base_name
}

dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    mappings loom.officialMojangMappings()
    modImplementation "net.fabricmc:fabric-loader:${project.loader_version}"
    modImplementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_version}"
}

processResources {
    inputs.property "version", project.version
    filesMatching("fabric.mod.json") {
        expand "version": project.version
    }
}

tasks.withType(JavaCompile).configureEach {
    it.options.release = 21
}

java {
    sourceCompatibility = JavaVersion.VERSION_21
    targetCompatibility = JavaVersion.VERSION_21
}
```

## 3. gradle.properties

```properties
org.gradle.jvmargs=-Xmx2G
org.gradle.parallel=true

minecraft_version=1.21.11
loader_version=0.18.1
fabric_version=0.141.1+1.21.11

mod_version=1.0.0
maven_group=com.example
archives_base_name=antiesp
```

## 4. .github/workflows/build.yml

```yaml
name: build
on:
  push:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: current
      - run: gradle build --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: antiesp-mod
          path: build/libs/*.jar
```

## 5. src/main/resources/fabric.mod.json

```json
{
  "schemaVersion": 1,
  "id": "antiesp",
  "version": "${version}",
  "name": "Anti ESP",
  "description": "Hides buried blocks below Y=0 until players get close.",
  "environment": "*",
  "entrypoints": {
    "main": ["com.example.antiesp.AntiEspMod"]
  },
  "mixins": ["antiesp.mixins.json"],
  "depends": {
    "fabricloader": ">=0.16.0",
    "minecraft": "~1.21.11",
    "java": ">=21",
    "fabric-api": "*"
  }
}
```

## 6. src/main/resources/antiesp.mixins.json

```json
{
  "required": true,
  "package": "com.example.antiesp.mixin",
  "compatibilityLevel": "JAVA_21",
  "mixins": [
    "BlockEntityInfoAccessor",
    "ChunkPacketDataMixin"
  ],
  "injectors": {
    "defaultRequire": 1
  }
}
```

## 7. src/main/java/com/example/antiesp/Hider.java

```java
package com.example.antiesp;

import net.minecraft.core.BlockPos;
import net.minecraft.core.Direction;
import net.minecraft.world.level.Level;
import net.minecraft.world.level.block.Blocks;
import net.minecraft.world.level.block.state.BlockState;
import net.minecraft.world.level.chunk.LevelChunk;
import net.minecraft.world.level.chunk.LevelChunkSection;
import net.minecraft.world.level.chunk.PalettedContainer;

public final class Hider {
    // Blocks below this height are hidden when buried.
    public static final int CUTOFF_Y = 0;
    // Overworld bottom (-64) and its first section index (-4).
    public static final int WORLD_MIN_Y = -64;
    public static final int WORLD_MIN_SECTION = -4;

    public static final BlockState FAKE = Blocks.DEEPSLATE.defaultBlockState();
    public static final ThreadLocal<LevelChunk> CURRENT = new ThreadLocal<>();

    private static final Direction[] DIRS = Direction.values();

    private Hider() {}

    private static boolean occludes(BlockState s) {
        return !s.isAir() && s.getFluidState().isEmpty() && s.canOcclude();
    }

    private static boolean canHide(BlockState s) {
        return !s.isAir() && s.getFluidState().isEmpty() && !s.is(Blocks.DEEPSLATE);
    }

    private static boolean isEnclosed(LevelChunk chunk, int lx, int y, int lz) {
        int minX = chunk.getPos().getMinBlockX();
        int minZ = chunk.getPos().getMinBlockZ();
        BlockPos.MutableBlockPos p = new BlockPos.MutableBlockPos();
        for (Direction d : DIRS) {
            int nx = lx + d.getStepX();
            int ny = y + d.getStepY();
            int nz = lz + d.getStepZ();
            if (nx < 0 || nx > 15 || nz < 0 || nz > 15) continue; // other chunk: assume solid
            if (ny < WORLD_MIN_Y) continue; // below the world: assume solid
            p.set(minX + nx, ny, minZ + nz);
            if (!occludes(chunk.getBlockState(p))) return false;
        }
        return true;
    }

    // Used for block entities: should the block at this spot have been hidden?
    public static boolean shouldHide(LevelChunk chunk, int lx, int y, int lz) {
        if (y >= CUTOFF_Y || y < WORLD_MIN_Y) return false;
        BlockPos.MutableBlockPos p = new BlockPos.MutableBlockPos(
                chunk.getPos().getMinBlockX() + lx, y, chunk.getPos().getMinBlockZ() + lz);
        return canHide(chunk.getBlockState(p)) && isEnclosed(chunk, lx, y, lz);
    }

    // Returns a copy of the section with buried blocks swapped for deepslate.
    public static LevelChunkSection transform(LevelChunk chunk, LevelChunkSection section) {
        if (chunk == null || chunk.getLevel().dimension() != Level.OVERWORLD) return section;

        LevelChunkSection[] all = chunk.getSections();
        int idx = -1;
        for (int i = 0; i < all.length; i++) {
            if (all[i] == section) {
                idx = i;
                break;
            }
        }
        if (idx < 0) return section;

        int baseY = (WORLD_MIN_SECTION + idx) * 16;
        if (baseY >= CUTOFF_Y || section.hasOnlyAir()) return section;

        PalettedContainer<BlockState> copy = null;
        for (int y = 0; y < 16; y++) {
            int wy = baseY + y;
            if (wy >= CUTOFF_Y) break;
            for (int x = 0; x < 16; x++) {
                for (int z = 0; z < 16; z++) {
                    BlockState st = section.getBlockState(x, y, z);
                    if (!canHide(st)) continue;
                    if (!isEnclosed(chunk, x, wy, z)) continue;
                    if (copy == null) copy = section.getStates().copy();
                    copy.getAndSetUnchecked(x, y, z, FAKE);
                }
            }
        }
        if (copy == null) return section;
        return new LevelChunkSection(copy, section.getBiomes());
    }
}
```

## 8. src/main/java/com/example/antiesp/AntiEspMod.java

```java
package com.example.antiesp;

import net.fabricmc.api.ModInitializer;
import net.fabricmc.fabric.api.event.lifecycle.v1.ServerTickEvents;
import net.minecraft.core.BlockPos;
import net.minecraft.network.protocol.Packet;
import net.minecraft.network.protocol.game.ClientGamePacketListener;
import net.minecraft.network.protocol.game.ClientboundBlockUpdatePacket;
import net.minecraft.server.level.ServerLevel;
import net.minecraft.server.level.ServerPlayer;
import net.minecraft.world.level.Level;
import net.minecraft.world.level.block.Blocks;
import net.minecraft.world.level.block.entity.BlockEntity;
import net.minecraft.world.level.block.state.BlockState;

import java.util.HashSet;
import java.util.Iterator;
import java.util.Map;
import java.util.WeakHashMap;

public class AntiEspMod implements ModInitializer {
    // Real blocks are sent to a player within this many blocks.
    public static final int REVEAL_RADIUS = 10;
    // How often to check, in ticks (20 ticks = 1 second).
    public static final int CHECK_INTERVAL = 10;

    private static final class Revealed {
        final HashSet<Long> positions = new HashSet<>();
    }

    private static final Map<ServerPlayer, Revealed> REVEALED = new WeakHashMap<>();

    @Override
    public void onInitialize() {
        ServerTickEvents.END_SERVER_TICK.register(server -> {
            if (server.getTickCount() % CHECK_INTERVAL != 0) return;
            for (ServerPlayer player : server.getPlayerList().getPlayers()) {
                revealAround(player);
            }
        });
    }

    private static void revealAround(ServerPlayer player) {
        if (!(player.level() instanceof ServerLevel level)) return;
        if (level.dimension() != Level.OVERWORLD) {
            REVEALED.remove(player);
            return;
        }

        BlockPos origin = player.blockPosition();
        Revealed rev = REVEALED.get(player);
        if (rev == null) {
            rev = new Revealed();
            REVEALED.put(player, rev);
        }

        // Forget far-away spots so they are re-sent if the chunk reloads later.
        int keep = REVEAL_RADIUS + 16;
        Iterator<Long> it = rev.positions.iterator();
        while (it.hasNext()) {
            long k = it.next();
            if (Math.abs(BlockPos.getX(k) - origin.getX()) > keep
                    || Math.abs(BlockPos.getZ(k) - origin.getZ()) > keep) {
                it.remove();
            }
        }

        int yMin = Math.max(Hider.WORLD_MIN_Y, origin.getY() - REVEAL_RADIUS);
        int yMax = Math.min(Hider.CUTOFF_Y - 1, origin.getY() + REVEAL_RADIUS);
        if (yMin > yMax) return;

        BlockPos.MutableBlockPos pos = new BlockPos.MutableBlockPos();
        for (int x = origin.getX() - REVEAL_RADIUS; x <= origin.getX() + REVEAL_RADIUS; x++) {
            for (int z = origin.getZ() - REVEAL_RADIUS; z <= origin.getZ() + REVEAL_RADIUS; z++) {
                for (int y = yMin; y <= yMax; y++) {
                    long key = BlockPos.asLong(x, y, z);
                    if (rev.positions.contains(key)) continue;
                    pos.set(x, y, z);
                    if (!level.hasChunkAt(pos)) continue;

                    rev.positions.add(key);
                    BlockState state = level.getBlockState(pos);
                    if (state.isAir() || state.is(Blocks.DEEPSLATE)) continue;

                    BlockPos fixed = pos.immutable();
                    player.connection.send(new ClientboundBlockUpdatePacket(fixed, state));

                    BlockEntity be = level.getBlockEntity(fixed);
                    if (be != null) {
                        Packet<ClientGamePacketListener> extra = be.getUpdatePacket();
                        if (extra != null) player.connection.send(extra);
                    }
                }
            }
        }
    }
}
```

## 9. src/main/java/com/example/antiesp/mixin/ChunkPacketDataMixin.java

```java
package com.example.antiesp.mixin;

import com.example.antiesp.Hider;
import net.minecraft.network.FriendlyByteBuf;
import net.minecraft.network.protocol.game.ClientboundLevelChunkPacketData;
import net.minecraft.world.level.Level;
import net.minecraft.world.level.chunk.LevelChunk;
import net.minecraft.world.level.chunk.LevelChunkSection;
import org.spongepowered.asm.mixin.Final;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.Shadow;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Inject;
import org.spongepowered.asm.mixin.injection.Redirect;
import org.spongepowered.asm.mixin.injection.callback.CallbackInfo;
import org.spongepowered.asm.mixin.injection.callback.CallbackInfoReturnable;

import java.util.Iterator;
import java.util.List;

@Mixin(ClientboundLevelChunkPacketData.class)
public abstract class ChunkPacketDataMixin {

    @Shadow
    @Final
    private List<?> blockEntitiesData;

    // Remember which chunk is being turned into a packet.
    @Inject(method = "calculateChunkSize", at = @At("HEAD"))
    private static void antiesp$sizeHead(LevelChunk chunk, CallbackInfoReturnable<Integer> cir) {
        Hider.CURRENT.set(chunk);
    }

    @Inject(method = "extractChunkData", at = @At("HEAD"))
    private static void antiesp$extractHead(FriendlyByteBuf buf, LevelChunk chunk, CallbackInfo ci) {
        Hider.CURRENT.set(chunk);
    }

    // The packet buffer size must match what we actually write.
    @Redirect(method = "calculateChunkSize",
            at = @At(value = "INVOKE",
                    target = "Lnet/minecraft/world/level/chunk/LevelChunkSection;getSerializedSize()I"))
    private static int antiesp$size(LevelChunkSection section) {
        return Hider.transform(Hider.CURRENT.get(), section).getSerializedSize();
    }

    // Write the disguised section instead of the real one.
    @Redirect(method = "extractChunkData",
            at = @At(value = "INVOKE",
                    target = "Lnet/minecraft/world/level/chunk/LevelChunkSection;write(Lnet/minecraft/network/FriendlyByteBuf;)V"))
    private static void antiesp$write(LevelChunkSection section, FriendlyByteBuf buf) {
        Hider.transform(Hider.CURRENT.get(), section).write(buf);
    }

    // Drop block entities (chests, spawners...) that belong to hidden blocks.
    @Inject(method = "<init>(Lnet/minecraft/world/level/chunk/LevelChunk;)V", at = @At("TAIL"))
    private void antiesp$filterBlockEntities(LevelChunk chunk, CallbackInfo ci) {
        Hider.CURRENT.remove();
        if (chunk.getLevel().dimension() != Level.OVERWORLD) return;

        Iterator<?> it = this.blockEntitiesData.iterator();
        while (it.hasNext()) {
            BlockEntityInfoAccessor info = (BlockEntityInfoAccessor) it.next();
            int y = info.antiesp$getY();
            if (y >= Hider.CUTOFF_Y) continue;
            int packed = info.antiesp$getPackedXZ();
            int lx = (packed >> 4) & 15;
            int lz = packed & 15;
            if (Hider.shouldHide(chunk, lx, y, lz)) {
                it.remove();
            }
        }
    }
}
```

## 10. src/main/java/com/example/antiesp/mixin/BlockEntityInfoAccessor.java

```java
package com.example.antiesp.mixin;

import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.gen.Accessor;

@Mixin(targets = "net.minecraft.network.protocol.game.ClientboundLevelChunkPacketData$BlockEntityInfo")
public interface BlockEntityInfoAccessor {
    @Accessor("packedXZ")
    int antiesp$getPackedXZ();

    @Accessor("y")
    int antiesp$getY();
}
```

---

If the Actions build shows a red X, open the failed run, copy the red error lines, and send them to me. The most likely trouble spots are Minecraft class or method names that changed in 1.21.11, which are quick to fix once I see the error.
