# PatchBukkit (Patched Edition)

A plugin for [PumpkinMC](https://pumpkinmc.org/) that adds support for [PaperMC](https://papermc.io/), [Spigot](https://www.spigotmc.org/), and [Bukkit](https://dev.bukkit.org/) plugins.

> **Fork Info & Fixes Summary:**
> Bản fork này chứa các bản sửa lỗi quan trọng (bugfixes) và tối ưu hóa tương thích cho PumpkinMC 0.2.0+ cũng như các môi trường phần cứng CPU thế hệ cũ (`x86-64-v2`).

---

## 🛠️ Summary of Applied Fixes (Các lỗi đã khắc phục)

### 1. Sửa lỗi nhận diện thư mục gốc Server (`get_server_root` trong `directories.rs`)
* **Hiện tượng:** Trước đây hàm `directories::get_base_directory` sử dụng `server.get_data_folder().parent()`. Trong môi trường thực tế (như container Pterodactyl hoặc khi Pumpkin nạp đường dẫn tương đối), logic này dẫn đến việc thư mục `/patchbukkit/` bị sinh sai vị trí (bị ném vào `plugins/patchbukkit`) hoặc crash do không tìm được thư mục cha tuyệt đối.
* **Khắc phục:** Viết lại hàm `get_server_root`, tự động dò ngược lên cây thư mục cha để tìm dấu hiệu nhận diện server root chuẩn (`pumpkin.toml` hoặc thư mục `plugins/`), đảm bảo thư mục `/patchbukkit/` và `/patchbukkit/patchbukkit-plugins/` luôn được tạo chính xác tại root của server.

### 2. Đồng bộ hóa Protobuf & Rust Native Callbacks (`proto/` & `native_callbacks/`)
* **Hiện tượng:** Mã nguồn gốc bị lỗi không thể biên dịch do không đồng bộ giữa định nghĩa Protobuf và Rust callbacks với PumpkinMC API mới:
  * `SendRawPacketRequest` gây lỗi `unresolved import` do phương thức `player.client.try_enqueue_packet_data` không còn tồn tại trong Pumpkin API.
  * `SpawnWorldEntityRequest` trong `world.rs` truy cập các trường không tồn tại (`entity_uuid`, `entity_id`).
* **Khắc phục:**
  * Đồng bộ và làm sạch lại `bridge.proto`, `entity.proto` và `world.proto`.
  * Loại bỏ các RPC / Callbacks không còn tương thích (`SendRawPacket`, `RemoveEntity`).
  * Tự động sinh UUID hợp lệ (`uuid::Uuid::new_v4()`) khi khởi tạo entity trong world.

### 3. Tối ưu hóa Cargo Build & Theo dõi thay đổi Proto (`Cargo.toml` & `build/main.rs`)
* **Khắc phục:**
  * Thiết lập `codegen-units = 1` trong profile `[profile.release]` để tối ưu hóa code generation và tương thích tốt nhất khi xuất thư viện chia sẻ FFI (`.so`).
  * Bổ sung chỉ thị `cargo::rerun-if-changed=../proto` vào build script để Cargo tự động phát hiện và compile lại Rust code mỗi khi có chỉnh sửa trong thư mục proto.

### 4. Khắc phục lỗi Crash `SIGILL` trên CPU đời cũ (`x86-64-v2` Support)
* **Hiện tượng:** Các bản build phát hành chính thức trên GitHub Actions của upstream mặc định nhắm vào tập lệnh `x86-64-v3` (AVX2/BMI2). Khi chạy trên các vi xử lý đời cũ hơn (chẳng hạn như Intel Core i3-3217U Ivy Bridge), tiến trình sẽ lập tức bị crash với tín hiệu lỗi phần cứng `SIGILL` (Illegal Instruction) ngay khi `dlopen`.
* **Khắc phục:** Hỗ trợ biên dịch nhắm vào kiến trúc `x86-64-v2` tương thích rộng rãi:
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
