# 第 6 章：`memfd` 與現行 Ashmem 匿名共享記憶體系統 — 大綱

> **原書對應**：第 6 章〈Ashmem匿名共享内存系统〉（p.335–400）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：GKI `kernel/common` 分支 `android17-6.18`（Linux 6.18，commit `028363aa3cfa`）
> **撰寫日期**：2026-11-25
> **狀態**：✅ 全章完成（`6.1`–`6.5` 共 5 節、36 個 Step、68 段程式碼、8 張 PlantUML 圖表皆已撰寫並逐行比對原始碼驗證通過，0 筆不符）

---

## 一、原始碼專案清單與版本驗證

本章所需專案皆已於本機 `src/` 檢出並驗證對應版本：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 6 章涵蓋範圍 |
|---|---|---|---|
| `kernel/common` | `android17-6.18` | `028363aa3` | `mm/memfd.c`、`include/uapi/linux/memfd.h`、`drivers/staging/android/{Kconfig,Makefile,ashmem.h,ashmem.rs,ashmem_range.rs,ashmem_shrinker.rs,ashmem_toggle.rs,shmem.rs,ashmem_exports.c}`、`security/selinux/{hooks.c,include/security.h,include/policycap.h,include/policycap_names.h}`、`drivers/android/binder.c`（`binder_translate_fd` / `BINDER_TYPE_FD`） |
| `platform/system/core` | `android-17.0.0_r1` | `545d248` | `libcutils/{ashmem-dev.cpp,ashmem-internal.h,include/cutils/ashmem.h}`、`init/devices.cpp`（`/dev/ashmem<boot_id>` 複製）、`rootdir/ueventd.rc` |
| `platform/frameworks/native` | `android-17.0.0_r1` | `ae266dc` | `libs/binder/{IMemory.cpp,MemoryHeapBase.cpp,MemoryBase.cpp,MemoryDealer.cpp}`、`libs/binder/include/binder/{IMemory.h,MemoryHeapBase.h,MemoryBase.h,MemoryDealer.h}`、`libs/binder/tests/binderMemoryHeapBaseUnitTest.cpp` |
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163b` | `native/android/sharedmem.cpp`（NDK `ASharedMemory_*`）、`core/java/android/os/{SharedMemory.java,SharedMemory.aidl,MemoryFile.java}`、`core/jni/{android_os_SharedMemory.cpp,android_os_MemoryFile.cpp}`、`core/java/android/service/notification/NotificationRankingUpdate.java`、`core/java/com/android/internal/os/ApplicationSharedMemory.java`、`core/jni/com_android_internal_os_ApplicationSharedMemory.cpp` |

> **重大架構演進說明（2.3 → 17）**：
> 1. **核心雙軌與交接**：原書 2.3 的 C 語言驅動 `drivers/staging/android/ashmem.c` 在 GKI 6.18 已完全移除，改由 **Rust 實作的 `ashmem_rust` 模組**（`ashmem.rs` + `ashmem_range.rs` + `ashmem_shrinker.rs` + `ashmem_toggle.rs` + `shmem.rs`，`Kconfig` 明定 `depends on SHMEM && RUST`）接手；與此同時，Android 自 Android 10 起逐步棄用 `ashmem` 與 pin/unpin 回收語意，並以 Linux 主線 **`memfd`（`memfd_create` + `F_ADD_SEALS`）** 取而代之。
> 2. **Android 17 的預設切換分水嶺**：在 `libcutils/ashmem-dev.cpp::use_memfd()` 中，只要核心啟用 SELinux `memfd_class` policy capability、裝置滿足 `ro.vendor.api_level >= 202604`（VSR-3.5.2-004）且 App 滿足 `targetSdk >= 37`（即 Android 17），`ashmem_create_region()` 便**預設改走 `memfd_create`**；僅舊版 App／相容路徑仍回退至 `/dev/ashmem<boot_id>`（由 `ueventd` 附加隨機 `boot_id` 防止硬編碼直開 `/dev/ashmem`）。
> 3. **Java 與 NDK 現代 API**：原書的 `MemoryFile` 已退居為 **`android.os.SharedMemory`** 的薄包裝（且 `allowPurging` 標記 `@Deprecated`）；原生 NDK 則提供穩定的 **`ASharedMemory_*`** C API（`libandroid`），Java 層 `SharedMemory` 直接實作 `Parcelable` 並以 `DirectByteBuffer` + `Cleaner` + `VMRuntime.registerNativeAllocation` 管理映射與原生記憶體壓力。

---

## 二、小節規劃與原書對照

本章維持原書 5 大節（6.1–6.5）的教學骨架，全面改寫為 Android 17 的 `memfd` + Rust 版 Ashmem 雙軌現代架構（每節圖表依新準則以 PlantUML 繪製，存於 `ch06-memfd-ashmem/figs/`）：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `6.1-kernel-memfd-ashmem.md` | 6.1 核心層匿名共享記憶體：`memfd` 與 Rust 版 Ashmem 驅動 | 6.1（p.336–353） | **全面重寫**：原書為 C 版 `ashmem.c`；17 併行剖析 Linux 6.18 主線 `mm/memfd.c`（`memfd_create`、`shmem_file_setup`、`F_ADD_SEALS`、SELinux `memfd_class`）與 Rust 重寫的 `drivers/staging/android/ashmem*.rs`（初始化、`AshmemToggleMisc`、`open`/`mmap`、`Area::pin`/`unpin`、`Shrinker` 回收） |
| `6.2-libcutils-ndk.md` | 6.2 執行期函式庫 `libcutils` 與 NDK `ASharedMemory` 存取介面 | 6.2（p.353–357） | **大幅改寫並擴充**：從單純包裝 `/dev/ashmem` 改為 `ashmem-dev.cpp` 雙後端分流（`use_memfd()` 在 API 37 預設切 `memfd`、`/dev/ashmem<boot_id>`、`is_memfd_fd`、`F_SEAL_FUTURE_WRITE` 保護、pin/unpin 棄用為 no-op），並補上 NDK `ASharedMemory_*`（`sharedmem.cpp`） |
| `6.3-cpp-interface.md` | 6.3 匿名共享記憶體的 C++ 存取介面（`IMemoryHeap` / `MemoryHeapBase` / `MemoryBase` / `MemoryDealer`） | 6.3（p.357–378） | **保留並更新**：類別從 `frameworks/base/libs/binder/` 遷至 `frameworks/native/libs/binder/`；詳解 `MemoryHeapBase` 新增的 `FORCE_MEMFD` / `MEMFD_ALLOW_SEALING`、`BpMemoryHeap` + `HeapCache` 去重映射、`BpMemory::getMemory` ILP32 邊界檢查與 `MemoryDealer`（`SimpleBestFitAllocator`），並以 `binderMemoryHeapBaseUnitTest.cpp` 取代原書自寫測試程式 |
| `6.4-java-interface.md` | 6.4 匿名共享記憶體的 Java 存取介面（`SharedMemory` 與 `MemoryFile`） | 6.4（p.378–394） | **大幅改寫**：以現代核心 `SharedMemory`（`android_os_SharedMemory.cpp`、`isRegionReadOnly`、`DirectByteBuffer`、`Closer`/`Unmapper`、`MemoryRegistration`、`Parcelable`）為主線，說明舊 `MemoryFile` 如何退化為薄包裝，並以 AOSP 17 實際系統用例（`NotificationRankingUpdate`、`ApplicationSharedMemory`）取代原書自寫 App 範例 |
| `6.5-sharing-mechanism.md` | 6.5 跨行程共享原理（Binder FD 傳遞與 `shmem` 頁快取映射） | 6.5（p.394–400） | **保留並深化**：以第 5 章 Binder 驅動為基礎，追蹤 `Parcel::writeFileDescriptor`（`BINDER_TYPE_FD`）→ `binder_translate_fd`（`fget_raw` → `security_binder_transfer_file` → `fd_install`）→ 雙行程 `mmap(MAP_SHARED)` 共享同一 `shmem` inode 實體頁框，並解析 `F_SEAL_FUTURE_WRITE`／`prot_mask` 與 SELinux `memfd_class` 的單寫多讀安全保證 |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 6.1 核心層匿名共享記憶體：`memfd` 與 Rust 版 Ashmem 驅動（`6.1-kernel-memfd-ashmem.md`）
- **情景**：
  1. 現代路徑：Android 17 App 呼叫 `memfd_create` 建立匿名共享記憶體並加上大小與唯寫封印；
  2. 相容路徑：開機時核心載入 Rust 版 `ashmem_rust` 模組並由 `ueventd` 建立 `/dev/ashmem<boot_id>`，舊行程開啟、設定大小與權限、`mmap` 並執行 `ASHMEM_UNPIN`／記憶體壓力回收。
- **起點 → 終點**：
  - `memfd` 路徑：`SYSCALL_DEFINE2(memfd_create)` → `sanitize_flags` → `alloc_name` → `alloc_file`（`shmem_file_setup` + `SHMEM_FL_MEMFD` + `security_inode_init_security_anon`）→ `fd_install`；以及 `memfd_fcntl(F_ADD_SEALS)` → `memfd_add_seals`；
  - Rust Ashmem 路徑：`AshmemModule::init`（`MiscDeviceRegistration::register("ashmem")` + 3 個 `AshmemToggleMisc` + `ASHMEM_SHRINKER`）→ `ueventd` `devices.cpp` 複製 `/dev/ashmem<boot_id>` → `AshmemMisc::open` → `ioctl(ASHMEM_SET_NAME / SET_SIZE / SET_PROT_MASK)` → `AshmemMisc::mmap`（`ShmemFile::new` → `shmem::zero_setup` → `shmem::set_file`）→ `Ashmem::pin_unpin`（`Area::pin`/`unpin`）→ `Shrinker::scan_objects` → `AshmemGuard::free_lru` → `ShmemFile::punch_hole`。

### 6.2 執行期函式庫 `libcutils` 與 NDK `ASharedMemory` 存取介面（`6.2-libcutils-ndk.md`）
- **情景**：原生 C/C++ 或 NDK 開發者呼叫 `ashmem_create_region` / `ASharedMemory_create` 建立共享記憶體，寫入資料後呼叫 `ashmem_set_prot_region(fd, PROT_READ)` / `ASharedMemory_setProt` 鎖定唯讀，另一端再以 `ashmem_valid` / `ashmem_get_size_region` / `ASharedMemory_getSize` 檢驗。
- **起點 → 終點**：`native/android/sharedmem.cpp::ASharedMemory_{create,getSize,setProt,dupFromJava}` → `libcutils/ashmem-dev.cpp::ashmem_create_region` → `use_memfd()`（4 條件判定）→ `__memfd_create_region`（`syscall(__NR_memfd_create)` + `ftruncate` + `F_ADD_SEALS(F_SEAL_GROW | F_SEAL_SHRINK)`）或 `__ashmem_create_region`（`get_ashmem_device_path` 讀 `boot_id` + `open` + `ASHMEM_SET_NAME/SIZE`）→ `ashmem_set_prot_region`（`is_memfd_fd` 分流 `memfd_set_prot_region(F_SEAL_FUTURE_WRITE)` vs `ioctl(ASHMEM_SET_PROT_MASK)`）→ `ashmem_pin_region`/`ashmem_unpin_region`（`do_pin` 棄用警告與 memfd no-op）→ `ashmem_get_size_region`（`fstat` vs `ASHMEM_GET_SIZE`）。

### 6.3 匿名共享記憶體的 C++ 存取介面（`6.3-cpp-interface.md`）
- **情景**：Native Server 建立 `MemoryHeapBase`（含 `FORCE_MEMFD` 與 `READ_ONLY` 模式）並透過 `MemoryDealer` 切分子塊 `MemoryBase`，經 Binder 傳回 `sp<IMemory>` 給 Client；Client 首次存取指標時透明完成跨行程 `mmap` 並由 `HeapCache` 去重快取。
- **起點 → 終點**：`MemoryHeapBase::MemoryHeapBase(size, flags, name)`（`FORCE_MEMFD` 走 `memfd_create_region` + `F_ADD_SEALS`，否則走 `ashmem_create_region` + `mapfd`）→ `MemoryDealer::allocate`（`SimpleBestFitAllocator` → `Allocation : MemoryBase`）→ Client 呼叫 `IMemory::pointer()` → `BpMemory::getMemory()`（Binder `GET_MEMORY` → `BnMemory::onTransact` 回傳 `sp<IBinder>` + `int64_t offset` + `uint64_t size` + 邊界檢查）→ `BpMemoryHeap::getBase()` → `assertMapped()` → `HeapCache::find_heap` + `assertReallyMapped()`（Binder `HEAP_ID` → `BnMemoryHeap::onTransact` 寫入 `fd` → `fcntl(F_DUPFD_CLOEXEC)` → `mmap(MAP_SHARED)`）→ 物件釋放或 Server 死亡時 `HeapCache::binderDied` / `free_heap` 清理映射。

### 6.4 匿名共享記憶體的 Java 存取介面（`6.4-java-interface.md`）
- **情景**：
  1. Java 服務透過 `SharedMemory.create` 建立共享記憶體、`mapReadWrite` 寫入資料、`setProtect(PROT_READ)` 轉唯讀，再直接作為 `Parcelable` 傳給另一行程以 `mapReadOnly` 零拷貝讀取（如 `NotificationRankingUpdate`）；
  2. 對照 `MemoryFile` 的現代實作與 `ApplicationSharedMemory`（`system_server` → 全 App 唯讀共享記憶體）。
- **起點 → 終點**：`SharedMemory.create` → `nCreate`（`android_os_SharedMemory.cpp::SharedMemory_nCreate` → `ashmem_create_region`）→ `new SharedMemory(fd)`（`nGetSize` + `MemoryRegistration` 呼叫 `VMRuntime.registerNativeAllocation` + `Cleaner(Closer)`）→ `map`（`Os.mmap(MAP_SHARED)` + `DirectByteBuffer` + `Unmapper`）→ `setProtect` / `isRegionReadOnly`（`F_GET_SEALS` 優先、`ASHMEM_GET_PROT_MASK` 回退）→ `writeToParcel` / `CREATOR.createFromParcel`；並對照 `MemoryFile`（薄包裝 `SharedMemory` + `@FastNative native_pin`）與 `ApplicationSharedMemory`。

### 6.5 跨行程共享原理（`6.5-sharing-mechanism.md`）
- **情景**：行程 A 建立 `memfd`（或 `ashmem`）共享記憶體、寫入資料並施加 `F_SEAL_FUTURE_WRITE`（或 `PROT_READ`），透過 Binder 交易把 fd 傳送給行程 B；行程 B 收到新 fd 後執行 `mmap(MAP_SHARED)` 並直接讀取實體記憶體內容，且任何嘗試以 `PROT_WRITE` 映射或 `write()` 寫入都會被核心攔截。
- **起點 → 終點**：行程 A `Parcel::writeFileDescriptor`（`BINDER_TYPE_FD`）→ `binder.c::binder_transaction` → `binder_translate_fd`（`fget_raw` → `security_binder_transfer_file` → `binder_deferred_fd_close` 與 `binder_do_fd_close` / `fd_install` 在行程 B `files_struct` 配置新 fd 指向同一 `struct file`）→ 行程 B `Parcel::readFileDescriptor` → `mmap(MAP_SHARED)` → `shmem_mmap` / `AshmemMisc::mmap`（驗證 `F_SEAL_FUTURE_WRITE` 或 `prot_mask` 與 SELinux `memfd_class`）→ 缺頁時 `shmem_fault` 將兩行程頁表指向同一 `struct inode` / `address_space` 的實體頁框（`struct folio`）。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| `kernel/goldfish/drivers/staging/android/ashmem.c`（C 驅動） | `kernel/common/mm/memfd.c` + `kernel/common/drivers/staging/android/ashmem.rs`（`ashmem_rust`） | 邏輯改變 + 語言重寫 | 2.3 C 版 `ashmem.c` 已完全移除；GKI 6.18 中 Ashmem 改由 Rust 重寫（`depends on SHMEM && RUST`），並以 Linux 主線 `memfd` 作為 Android 17（API 37）預設匿名共享記憶體後端 |
| `/dev/ashmem`（固定裝置路徑） | `memfd_create()`（無裝置節點）或 `/dev/ashmem<boot_id>`（`system/core/init/devices.cpp`） | 邏輯改變 | `ueventd` 複製 `/dev/ashmem<boot_id>` 阻止 App 硬編碼直開 `/dev/ashmem`；API 37+ 預設直接走 `memfd_create` 系統呼叫 |
| `ASHMEM_PIN` / `ASHMEM_UNPIN` + `ashmem_shrink`（核心主動回收） | `ashmem_range.rs` + `ashmem_shrinker.rs`（僅相容層）；`memfd` 路徑為 no-op（Android Q 起棄用） | 已棄用（memfd 移除） | `libcutils/ashmem-dev.cpp::do_pin` 印出棄用警告且對 `memfd` 直接回傳 0；Android 改推 `onTrimMemory` 主動釋放 |
| `ASHMEM_SET_PROT_MASK`（單純 prot_mask） | `memfd`: `fcntl(F_ADD_SEALS, F_SEAL_GROW \| F_SEAL_SHRINK \| F_SEAL_FUTURE_WRITE)`；`ashmem`: `Ashmem::set_prot_mask` + `AshmemToggle{Read,Exec}` | 邏輯改變 | `memfd` 透過 Linux file sealing 不可逆地禁止擴縮與後續可寫映射，並配合 SELinux `memfd_class` 權限控管 |
| `system/core/libcutils/ashmem-dev.c` | `platform/system/core/libcutils/ashmem-dev.cpp` + `platform/frameworks/base/native/android/sharedmem.cpp` | 改名 + 邏輯改變 + 新增 | 改寫為 C++ 並內建 `use_memfd()` 雙後端分流；另於 NDK 新增穩定公開 API `ASharedMemory_*` |
| `frameworks/base/libs/binder/{IMemory,MemoryHeapBase,MemoryBase,MemoryDealer}.cpp` | `platform/frameworks/native/libs/binder/{IMemory,MemoryHeapBase,MemoryBase,MemoryDealer}.cpp` | 搬家 + 邏輯改變 | 遷至 `frameworks/native`；`MemoryHeapBase` 新增 `FORCE_MEMFD`/`MEMFD_ALLOW_SEALING`；`BpMemory::getMemory` 改用 64-bit offset/size 並加 ILP32 越界防護；`BpMemoryHeap` 改用 `std::atomic<int32_t> mHeapId` 與 `F_DUPFD_CLOEXEC` |
| `frameworks/base/core/java/android/os/MemoryFile.java` + `android_os_MemoryFile.cpp` | `SharedMemory.java` + `android_os_SharedMemory.cpp`（主角）；`MemoryFile.java`（薄包裝） | 新增 + 邏輯改變 | 現代 Java 層改以 `SharedMemory`（`Parcelable`、`DirectByteBuffer`、`Cleaner`、`MemoryRegistration`、`isRegionReadOnly`）為核心；`MemoryFile` 內部委派給 `SharedMemory` 且 `allowPurging` 已棄用 |
| `binder_transaction` 直接呼叫 `task_get_unused_fd_flags` + `task_fd_install`（即時安裝目標行程 fd） | `kernel/common/drivers/android/binder.c::binder_translate_fd`（延遲安裝 `binder_txn_fd_fixup` + `binder_do_fd_close`） | 邏輯改變 | 現代 Binder 驅動不再直接操作目標行程 `files_struct`，改以 fixup 清單在接收端執行緒讀取交易時才安裝 fd，避免遠端行程 FD 表競爭與洩漏 |
| —（無已知未確認項目） | — | — | 本章所有候選檔案與符號皆已於 `android-17.0.0_r1` 與 `android17-6.18` 驗證存在，0 筆 `TODO: 未確認` |
