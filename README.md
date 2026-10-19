# 🛠️ Paper Dev Toolkit

[![CI Build](https://github.com/croxln/paper-dev-toolkit/actions/workflows/ci.yml/badge.svg)](https://github.com/croxln/paper-dev-toolkit/actions/workflows/ci.yml)
[![Java 21+](https://img.shields.io/badge/Java-21%2B-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://adoptium.net/)
[![PaperMC](https://img.shields.io/badge/PaperMC-1.21.4-1A1A1A?style=for-the-badge&logo=minecraft&logoColor=white)](https://papermc.io/)
[![Leaf / Folia](https://img.shields.io/badge/Leaf%20%2F%20Folia-Compatible-3B82F6?style=for-the-badge)](https://github.com/PaperMC/Folia)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

A high-performance server foundation library for modern **Leaf, Folia, and PaperMC (1.21+)** environments.

Focuses on solving the two most critical backend challenges of modern multi-threaded server architectures: **cross-region thread scheduling** and **scoped, reversible memory cooldowns**, without forcing cookie-cutter UI templates.

---

## ⚡ Core Engine Features

### 1. 🌐 Universal ThreadBridge (`ThreadBridge`)
Ensures every entity, player, and block operation executes strictly on the thread that owns it across Leaf, Folia, and traditional Paper.
- **Region-Safe Execution:** Eliminates `RegionizedServer` threading violations.
- **Player & Entity Safety:** `run(player, action)` and `runOrRetired(player, action, fallback)`.
- **Auto-Cleanup on Reload:** Implements `AutoCloseable` with automatic task tracking (`tasks.remove(...)`) to prevent orphaned timer tasks on `/reload` or plugin disable.

### 2. ⏱️ Scoped & Reversible Cooldowns (`CooldownService`)
Pure in-memory, high-throughput cooldown and rate-limiting engine.
- **Reversible Tokens:** If a teleport fails, player cancels, or payment is aborted, `token.revert()` rolls back the exact previous cooldown and decrement usage count.
- **Multi-Scope Queries:** Check cooldowns globally (`"global"`) or per-feature/world (`remainingAny(uuid, "world_nether", "global")`).
- **Usage Limiting:** Built-in `isLocked(player, maxUses)` for finite use features.
- **Persistence Snapshots:** Direct `snapshot()` and `restore()` methods to persist active cooldowns to SQLite/MySQL/YAML across server reboots.

### 3. 🧩 Native 1.21.4 Data Components (`ItemComponentBuilder`)
Full native support for modern 1.20.5+ / 1.21 item components without NMS:
- Custom stack sizes (`maxStackSize(64)`), fire immunity (`fireResistant(true)`), edible item properties, and wearable slot binding.

---

## 📦 Installation

### Gradle (Composite Build - Recommended for local plugin suites)

In your plugin's `settings.gradle.kts`:
```kotlin
includeBuild("../paper-dev-toolkit")
```
In your plugin's `build.gradle.kts`:
```kotlin
dependencies {
    implementation("io.github.croxln:paper-dev-toolkit:1.0.0")
}
```

---

## 🚀 Practical Examples

### 1. ThreadBridge: Safe Regional Operations on Leaf / Folia

```java
import io.github.croxln.toolkit.thread.ThreadBridge;

public final class MyPlugin extends JavaPlugin {
    private ThreadBridge threadBridge;

    @Override
    public void onEnable() {
        this.threadBridge = new ThreadBridge(this);
    }

    @Override
    public void onDisable() {
        // Automatically cancels all running tasks
        this.threadBridge.close();
    }

    public void healPlayerSafely(Player player) {
        // Runs on player's regional thread even if called from an async thread
        threadBridge.run(player, () -> {
            player.setHealth(player.getMaxHealth());
            player.sendMessage("§aHealed!");
        });
    }
}
```

---

### 2. CooldownService: Scoped & Reversible Action Cooldowns

```java
import io.github.croxln.toolkit.cooldown.CooldownService;
import java.time.Duration;

public final class TeleportManager {
    private final CooldownService cooldowns = new CooldownService();

    public void requestTeleport(Player player) {
        // Check if player is on cooldown
        if (cooldowns.isCoolingDown(player.getUniqueId(), "teleport")) {
            long remainingSec = cooldowns.remaining(player.getUniqueId(), "teleport").toSeconds();
            player.sendMessage("§cPlease wait " + remainingSec + "s before teleporting again.");
            return;
        }

        // Start 30s cooldown and receive a rollback token
        CooldownService.Token token = cooldowns.start(player.getUniqueId(), "teleport", Duration.ofSeconds(30));

        // If something fails (e.g. player moves or damage taken), revert immediately!
        boolean teleportSuccess = tryFindSafeLocation(player);
        if (!teleportSuccess) {
            token.revert();
            player.sendMessage("§cTeleport failed! Cooldown was restored.");
        }
    }
}
```

---

## 📄 License
Released under the [MIT License](LICENSE).