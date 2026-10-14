# PatchBukkit (Patched Edition)

A plugin for [PumpkinMC](https://pumpkinmc.org/) that adds support for [PaperMC](https://papermc.io/), [Spigot](https://www.spigotmc.org/), and [Bukkit](https://dev.bukkit.org/) plugins.

> **Fork Info & Fixes Summary:**
> This fork incorporates critical bugfixes and compatibility enhancements for running PatchBukkit on PumpkinMC 0.2.0+, as well as ensuring support for legacy CPU architectures (`x86-64-v2`).

---

## 🛠️ Summary of Applied Fixes

### 1. Fixed Server Root Directory Detection (`get_server_root` in `directories.rs`)
* **Problem:** Originally, `directories::get_base_directory` relied on `server.get_data_folder().parent()`. In production environments (such as Pterodactyl container setups or when Pumpkin loads relative paths), this caused the `/patchbukkit/` folder to be generated in the wrong location (nested inside `plugins/patchbukkit`) or panicked due to failing to resolve the absolute parent directory.
* **Resolution:** Re-implemented `get_server_root` to recursively traverse parent directories looking for standard server root indicators (`pumpkin.toml` or the `plugins/` directory). This guarantees that `/patchbukkit/` and `/patchbukkit/patchbukkit-plugins/` are always created directly in the true root of the server.

### 2. Synchronized Protobuf Definitions & Rust Native Callbacks (`proto/` & `native_callbacks/`)
* **Problem:** Upstream source had build-breaking mismatches between Protobuf definitions and the latest PumpkinMC 0.2.0+ API:
  * `SendRawPacketRequest` caused an `unresolved import` error because `player.client.try_enqueue_packet_data` was removed in Pumpkin's API.
  * `SpawnWorldEntityRequest` in `world.rs` attempted to access non-existent fields (`entity_uuid`, `entity_id`).
* **Resolution:**
  * Cleaned up and synchronized `bridge.proto`, `entity.proto`, and `world.proto`.
  * Removed deprecated RPC endpoints and callbacks (`SendRawPacket`, `RemoveEntity`).
  * Automatically generate a valid UUID (`uuid::Uuid::new_v4()`) when spawning entities in the world.

### 3. Optimized Cargo Build Profile & Proto Change Tracking (`Cargo.toml` & `build/main.rs`)
* **Resolution:**
  * Configured `codegen-units = 1` under `[profile.release]` to optimize code generation and provide cleaner FFI dynamic shared library (`.so`) link behavior.
  * Added `cargo::rerun-if-changed=../proto` to `build/main.rs` so Cargo automatically detects and recompiles Protobuf changes without requiring manual rebuilds.

### 4. Resolved Hardware `SIGILL` Crash on Legacy CPUs (`x86-64-v2` Support)
* **Problem:** Upstream CI/CD releases target `x86-64-v3` (requiring AVX2/BMI2 instructions) by default. When deployed on older hardware (such as 3rd-generation Intel Core i3-3217U Ivy Bridge CPUs), the binary crashed immediately with a hardware `SIGILL` (Illegal Instruction) signal upon `dlopen`.
* **Resolution:** Provided cross-compilation instructions and flags targeting `x86-64-v2` for maximum hardware compatibility:
  ```bash
  RUSTFLAGS="-C target-cpu=x86-64-v2" cargo zigbuild --target x86_64-unknown-linux-gnu.2.31 --release
  ```

---

## Installation

1. **Requirement**: Install Java 25 or newer.
2. **Download**: Grab the library matching your operating system and CPU architecture from Releases.
3. **Deploy**: Place the downloaded file into your PumpkinMC `plugins/` directory. (Run PumpkinMC once to generate this folder if it doesn't exist).
4. **Initialize**: Restart PumpkinMC. This creates a new `patchbukkit/` directory in your server root.
5. **Add Plugins**: Drop your `.jar` plugin files (Paper/Spigot/Bukkit) into the newly created `patchbukkit/patchbukkit-plugins/` folder and restart.

## Development & Building

1. Make sure Protobuf (`protoc`), Rust, and Java 25 are installed.
2. Build Java runtime:
   ```bash
   cd java && ./gradlew jar && cd ..
   ```
3. Build Rust native library:
   ```bash
   cd rust && cargo build --release
   ```
4. Copy the generated native library from `rust/target/release/` to your PumpkinMC `plugins/` directory.

There is also an [architecture guide](ARCHITECTURE.md) available.
