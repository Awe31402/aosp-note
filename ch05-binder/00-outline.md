# 第 5 章：Binder 進程間通信系統 — 大綱（檢查點 1）

> **原書對應**：第 5 章〈Binder 进程间通信系统〉（p.152–334）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：GKI `kernel/common` 分支 `android17-6.18`（Linux 6.18，固定檢出 commit `028363aa3cfa`）
> **撰寫日期**：2026-11-24
> **狀態**：檢查點 1 已確認 → 檢查點 2（`5.1-driver.md`）已完成，待確認深度與格式

---

## 一、原始碼專案清單與版本驗證

以下專案皆已對照 `platform/manifest`（`refs/tags/android-17.0.0_r1`）的 `default.xml` 及 `android.googlesource.com` 實測確認存在：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 5 章涵蓋範圍 |
|---|---|---|---|
| `kernel/common` | `android17-6.18`（GKI Linux 6.18） | `028363aa3cfa` | `drivers/android/{binder.c,binder_internal.h,binder_alloc.c,binder_alloc.h,binderfs.c,binder_pick.c}`、`include/uapi/linux/android/binder.h` |
| `platform/frameworks/native` | `android-17.0.0_r1` | `ae266dc` | `libs/binder/`（libbinder、`IServiceManager.aidl`、`BackendUnifiedServiceManager`、`JavaBBinderBase`）、`cmds/servicemanager/`、`services/gpuservice/main_gpuservice.cpp` |
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c16` | `core/java/android/os/{Binder.java,BinderProxy.java,ServiceManager.java,ServiceManagerNative.java}`、`core/java/com/android/internal/os/BinderInternal.java`、`core/jni/{android_util_Binder.cpp,android_os_ServiceManagerNative.cpp}` |
| `platform/system/tools/aidl` | `android-17.0.0_r1` | `9a94664` | `generate_cpp.cpp`、`generate_java_binder.cpp`、`codelab/`、`tests/golden_output/`（官方預先產生的 C++ / Java AIDL 骨架碼） |
| `platform/system/core` | `android-17.0.0_r1` | `545d248` | `rootdir/init.rc`（掛載 `/dev/binderfs` 與建立 `/dev/{binder,hwbinder,vndbinder}` 符號連結） |

> **GKI 分支確認依據**：
> 1. `source.android.com` 官方 GKI 文件標明 Android 17 對應 `android17-6.18`（Linux 6.18）。
> 2. `android-17.0.0_r1` manifest 已收錄 `kernel/prebuilts/6.18/{arm64,x86_64}`。
> 3. `git ls-remote` 確認 `kernel/common` 存在 `refs/heads/android17-6.18`（`028363aa3cfa34ff91b31494770815c593b98af9`）。
>
> **關於 Rust Binder / RPC Binder / `libbinder_ndk` / `libbinder_rs` 的取捨**：
> `android17-6.18` 同時包含 C 版（`CONFIG_ANDROID_BINDER_IPC`，`drivers/android/binder.c`）與 Rust 版（`CONFIG_ANDROID_BINDER_IPC_RUST`，`drivers/android/binder/`）驅動，開機透過 `binder_pick.c`（核心參數 `binder.impl=c|rust`，預設 `c`）擇一載入；`libs/binder` 亦包含 RPC Binder（Socket 傳輸）、NDK（`libs/binder/ndk/`）與 Rust（`libs/binder/rust/`）綁定。依簡報 Limits 原則（不新增原書沒有的主題），本章**以預設 C 版驅動與 `libbinder` C++ / Java 為主線**，上述新機制僅在架構定位與對照表中標註，不另開獨立小節。

---

## 二、小節規劃與原書對照

本章維持原書 10 大節（5.1–5.10）的情景式骨架，針對 Android 17 的重大重構調整各節內容與實例：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `5.1-driver.md` | 5.1 Binder 驅動程式 | 5.1（p.153–190） | **保留並大幅改寫**：加入 binderfs 與多 domain（`binder`/`hwbinder`/`vndbinder`）、三級細粒度自旋鎖、`binder_alloc` 獨立模組、無核心 VA 映射之按需裝頁與 LRU shrinker 回收機制 |
| `5.2-libbinder.md` | 5.2 Binder 進程間通信庫（libbinder） | 5.2（p.191–195） | **保留並更新**：涵蓋 `ProcessState`、`IPCThreadState`、`IBinder`/`BBinder`/`BpBinder`/`IInterface` 新貌、`IMPLEMENT_META_INTERFACE` 手動白名單限制，簡述與 NDK/Rust/RPC Binder 的關係 |
| `5.3-aidl-example.md` | 5.3 Binder 進程間通信應用實例（AIDL C++） | 5.3（p.196–203） | **改寫**：原書手寫 `IFregService`/`Bn`/`Bp`，但 17 的 `IMPLEMENT_META_INTERFACE` 已以 `static_assert` 禁止新增手寫介面；改以 AIDL 定義服務並追 `aidl --lang=cpp` 產生的 `I<Name>`/`Bn<Name>`/`Bp<Name>`（使用 `system/tools/aidl/tests/golden_output/` 真實輸出與 `gpuservice` 啟動骨架） |
| `5.4-refcount.md` | 5.4 Binder 物件引用計數技術 | 5.4（p.204–219） | **保留四層結構並更新**：維持本地 / 實體 / 引用 / 代理物件生命週期四子節，更新為細粒度鎖（`inner_lock`/`outer_lock`/`node->lock`）、`tmp_refs` 與 `dbitmap` handle 管理 |
| `5.5-death-notification.md` | 5.5 Binder 物件死亡通知機制 | 5.5（p.220–231） | **保留三階段情景**：註冊 / 發送 / 註銷死亡通知，更新 `binder_deferred_release` / `binder_node_release` 與 `BpBinder` 實作（註記 17 另新增的凍結通知 `FREEZE_NOTIFICATION`） |
| `5.6-servicemanager-start.md` | 5.6 ServiceManager 的啟動過程 | 5.6（p.232–245） | **大幅重寫**：原書為獨立 C 程式（`service_manager.c` + `binder.c`）；17 改為 C++ + `libbinder` + AIDL `BnServiceManager` + `utils/Looper`（`setupPolling`）+ `timerfd`（Lazy Service）+ `BINDER_SET_CONTEXT_MGR_EXT`（SELinux SID） |
| `5.7-servicemanager-proxy.md` | 5.7 ServiceManager 代理物件的取得過程 | 5.7（p.246–251） | **重寫**：從直接建立 `BpServiceManager` 改為 `defaultServiceManager()` → `CppBackendShim` → `BackendUnifiedServiceManager`（含 `BinderCacheWithInvalidation`）→ AIDL `android::os::BpServiceManager`（handle 0） |
| `5.8-service-start.md` | 5.8 Service 組件的啟動過程 | 5.8（p.252–298） | **保留兩階段，改寫呼叫鏈**：以真實系統服務 `GpuService`（`main_gpuservice.cpp`）取代原書自建 `FregService`；完整追蹤 `addService` 跨行程交易（含 `BR_TRANSACTION_SEC_CTX`、SELinux `Access::canAdd`、`linkToDeath`）與執行緒池啟動（`startThreadPool` / `joinThreadPool` / `BR_SPAWN_LOOPER`） |
| `5.9-service-proxy.md` | 5.9 Service 代理物件的取得過程 | 5.9（p.299–307） | **保留情景，改寫呼叫鏈**：追蹤 `getService` → `getService2`（`ServiceWithMetadata`）、Client 端快取命中與回填（`BinderCacheWithInvalidation`）、Server 端 `ServiceManager::tryGetService`（含 `Access::canFind` 與 Lazy Service `tryStartService`）與核心 `binder_translate_handle` |
| `5.10-java-interface.md` | 5.10 Binder 進程間通信機制的 Java 介面 | 5.10（p.308–334） | **保留五子節，更新 JNI 與 AIDL 橋接**：揭示 17 的關鍵改動——Java `ServiceManagerProxy` 經由 JNI `getNativeServiceManager()` 複用 C++ `BackendUnifiedServiceManager`；`BinderProxy` 獨立成檔並以 `ProxyMap` + `NativeAllocationRegistry` 管理；`JavaBBinder` 拆為 `JavaBBinderBase` + `JavaBBinderExt` |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 5.1 Binder 驅動程式（`5.1-driver.md`）
- **5.1.1 基礎資料結構與鎖模型**（對應原書 5.1.1，p.154–172）
  - **情景**：梳理驅動內外資料結構（`binder_work`、`binder_node`、`binder_ref`/`binder_ref_data`、`binder_ref_death`、`binder_proc`、`binder_thread`、`binder_transaction`、`binder_buffer`、`binder_alloc`、`binder_device`/`binder_context`/`binderfs_info`、`flat_binder_object`、`binder_transaction_data`/`_secctx`、`BR_*`/`BC_*`）及取代全域 `binder_lock` 的三級鎖序（`outer_lock` → `node->lock` → `inner_lock`）。
- **5.1.2 Binder 裝置初始化與 binderfs 掛載**（對應原書 5.1.2，p.172–173）
  - **情景**：系統開機時核心初始化 Binder 驅動並由 `init` 掛載 `/dev/binderfs`、建立多組獨立 context 的裝置節點。
  - **起點 → 終點**：`kernel/common/drivers/android/binder.c::binder_init` → `binder_alloc_shrinker_init` / `init_binderfs`（`binderfs.c`）→ `system/core/rootdir/init.rc` 執行 `mount binder binder /dev/binderfs` → `binderfs_fill_super` → `binderfs_binder_device_create` 建立 `binder`/`hwbinder`/`vndbinder` 並建立 `/dev/*` symlink。
- **5.1.3 Binder 裝置檔的開啟過程**（對應原書 5.1.3，p.173–174）
  - **情景**：使用者空間行程首次初始化 `ProcessState` 並開啟 `/dev/binder`（或 `/dev/vndbinder`）。
  - **起點 → 終點**：`frameworks/native/libs/binder/ProcessState.cpp::open_driver`（`open`、`BINDER_VERSION`、`BINDER_SET_MAX_THREADS`、`BINDER_ENABLE_ONEWAY_SPAM_DETECTION`）→ `drivers/android/binder.c::binder_open`（建立 `binder_proc`、初始化 `dbitmap` 與鎖、由 `is_binderfs_device` 綁定 `binder_context`、`binder_alloc_init`、掛入 `binder_procs`）。
- **5.1.4 Binder 裝置檔的記憶體映射過程**（對應原書 5.1.4，p.174–180）
  - **情景**：`ProcessState` 開啟裝置後呼叫 `mmap(BINDER_VM_SIZE)` 為該行程配置交易接收緩衝區位址空間。
  - **起點 → 終點**：`ProcessState::ProcessState`（`mmap` 1MB − 2 pages）→ `drivers/android/binder.c::binder_mmap`（權限檢查、設定 `VM_DONTCOPY | VM_MIXEDMAP` 與 `binder_vm_ops`）→ `drivers/android/binder_alloc.c::binder_alloc_mmap_handler`（分配 `alloc->pages` 陣列與單一初始空閒 `binder_buffer`、設 `free_async_space = buffer_size / 2`；**不配置核心 VA、不預先分配實體頁**）。
- **5.1.5 核心緩衝區管理**（對應原書 5.1.5，p.181–190）
  - **情景**：一次交易在目標行程中配置接收緩衝區、從發送端拷貝資料、交給目標行程讀取，以及目標行程以 `BC_FREE_BUFFER` 歸還緩衝區。
  - **起點 → 終點**：
    - 分配：`binder_alloc_new_buf` → `binder_alloc_new_buf_locked`（從 `free_buffers` 紅黑樹切分）→ `binder_install_buffer_pages` → `binder_install_single_page`（`alloc_page` + `vm_insert_page`）/ `binder_lru_freelist_del`；
    - 拷貝：`binder_alloc_copy_user_to_buffer` / `binder_alloc_copy_to_buffer`（透過 `kmap_local_page` 逐頁寫入目標 `struct page`）；
    - 查詢與釋放：`binder_alloc_prepare_to_free` → `binder_alloc_free_buf` → `binder_free_buf_locked`（合併相鄰空閒塊，將未使用頁加入 `binder_lru_freelist_add`，待系統記憶體壓力由 shrinker `binder_alloc_free_page` 真正釋放）。

### 5.2 Binder 進程間通信庫（`5.2-libbinder.md`）
- **情景**：理解使用者空間 `libbinder` 的核心類別設計，作為後續實例與追蹤呼叫鏈的基礎。
- **起點 → 終點**：`ProcessState`（單例管理 fd/mmap/執行緒池上限/handle 表與 `pthread_atfork` 防護）→ `IPCThreadState`（每執行緒 TLS、`mIn`/`mOut` Parcel、與驅動互動）→ `IBinder` 類別族（`BBinder`/`BnInterface` vs `BpBinder`/`BpRefBase`/`BpInterface`、`IInterface`、`DECLARE_META_INTERFACE` / `IMPLEMENT_META_INTERFACE` 白名單檢查）→ `Parcel` 物件扁平化概覽。

### 5.3 Binder 進程間通信應用實例（`5.3-aidl-example.md`）
- **情景**：在 Android 17 中以 AIDL 定義服務介面並解析 `aidl` 編譯器為 C++ 產生的完整程式碼，搭配原生服務與客戶端的註冊/呼叫流程。
- **起點 → 終點**：
  - 介面定義與生成：`system/tools/aidl/generate_cpp.cpp` 與官方 golden output（`tests/golden_output/aidl-cpp-java-test-interface-cpp-source/` 的 `ICppJavaTests.h`、`BnCppJavaTests.h`、`BpCppJavaTests.h`、`ICppJavaTests.cpp`）→ 展示 `I<Name>`、`Bp<Name>`（序列化並呼叫 `remote()->transact`）、`Bn<Name>::onTransact`（反序列化並分派實作方法）、`I<Name>Default` 與 `I<Name>Delegator`；
  - 服務啟動與客戶端呼叫骨架：對照 `frameworks/native/services/gpuservice/main_gpuservice.cpp`（C++ `libbinder`）與 `system/tools/aidl/codelab/`（`IHello.aidl` 與 NDK `testClient.cpp`）。

### 5.4 Binder 物件引用計數技術（`5.4-refcount.md`）
- **情景**：追蹤跨行程共享一個 Binder 服務物件時，四層物件的強弱引用計數如何聯動。
- **起點 → 終點**：
  - 5.4.1 本地物件 `BBinder`：`binder_thread_read` 寫入 `BR_INCREFS`/`BR_ACQUIRE`/`BR_RELEASE`/`BR_DECREFS` → `IPCThreadState::executeCommand` 增減 `RefBase` 計數並回傳 `BC_INCREFS_DONE`/`BC_ACQUIRE_DONE` → `binder_thread_write` 清除 `pending_*_ref`；
  - 5.4.2 實體物件 `binder_node`：`binder_new_node` → `binder_inc_node_nilocked` / `binder_dec_node_nilocked`（`internal_strong_refs`、`local_strong_refs`、`local_weak_refs`、`tmp_refs` 與 `binder_work` `BINDER_WORK_NODE`）；
  - 5.4.3 引用物件 `binder_ref`：`binder_get_ref_for_node_olocked`（以 `dbitmap` 分配 `desc`）→ `binder_inc_ref_olocked` / `binder_dec_ref_olocked` → `binder_cleanup_ref`；
  - 5.4.4 代理物件 `BpBinder`：`ProcessState::getStrongProxyForHandle` → `BpBinder::create` → `onFirstRef`（`incStrongHandle` / `BC_ACQUIRE`）→ `onLastStrongRef`（`decStrongHandle` / `BC_RELEASE`）→ `onIncStrongAttempted`（`attemptIncStrongHandle` / `BC_ATTEMPT_ACQUIRE`）→ `~BpBinder`（`decWeakHandle` / `BC_DECREFS`）。

### 5.5 Binder 物件死亡通知機制（`5.5-death-notification.md`）
- **情景**：Client 註冊死亡監聽，於 Server 異常終止時收到回呼，或在不需要時主動註銷。
- **起點 → 終點**：
  - 5.5.1 註冊：`BpBinder::linkToDeath` → `IPCThreadState::requestDeathNotification`（`BC_REQUEST_DEATH_NOTIFICATION`）→ `binder_thread_write` 配置 `binder_ref_death`（若目標已死則立刻投遞 `BINDER_WORK_DEAD_BINDER`）；
  - 5.5.2 發送：Server 終止關閉 `/dev/binder` → `binder_release` → `binder_deferred_release` → `binder_node_release` 遍歷 `node->refs` 將 `death->work` 排入各 Client 的 `todo` → Client `binder_thread_read` 回傳 `BR_DEAD_BINDER` → `IPCThreadState::executeCommand` → `BpBinder::sendObituary` → `DeathRecipient::binderDied` → 回覆 `BC_DEAD_BINDER_DONE`；
  - 5.5.3 註銷：`BpBinder::unlinkToDeath` → `IPCThreadState::clearDeathNotification`（`BC_CLEAR_DEATH_NOTIFICATION`）→ `binder_thread_write` → `BR_CLEAR_DEATH_NOTIFICATION_DONE` 釋放 `binder_ref_death`。

### 5.6 ServiceManager 的啟動過程（`5.6-servicemanager-start.md`）
- **情景**：開機時 `init` 依 `servicemanager.rc` 啟動 `/system/bin/servicemanager`，完成 Context Manager 註冊並進入事件迴圈。
- **起點 → 終點**：
  - 5.6.1 開啟與映射：`cmds/servicemanager/main.cpp::main` → `ProcessState::initWithDriver(driver)` → `setThreadPoolMaxThreadCount(0)` → `setCallRestriction(FATAL_IF_NOT_ONEWAY)`；
  - 5.6.2 註冊為 Context Manager：建立 `ServiceManager`（`os::BnServiceManager` + `DeathRecipient`）→ `setRequestingSid(true)` → 自註冊 `"manager"` → `IPCThreadState::setTheContextObject(manager)` → `ProcessState::becomeContextManager()`（`BINDER_SET_CONTEXT_MGR_EXT` + `FLAT_BINDER_FLAG_TXN_SECURITY_CTX`）→ `binder.c::binder_ioctl_set_ctx_mgr` 建立 `context->binder_context_mgr_node`；
  - 5.6.3 進入事件迴圈：`Looper::prepare` → `BinderCallback::setupTo`（`IPCThreadState::setupPolling` 寫入 `BC_ENTER_LOOPER` 並將 binder fd 加入 epoll）→ `ClientCallbackCallback::setupTo`（5 秒 `timerfd`）→ `SetProperty("servicemanager.ready", "true")` → `looper->pollAll(-1)` → `BinderCallback::handleEvent` → `IPCThreadState::handlePolledCommands`。

### 5.7 ServiceManager 代理物件的取得過程（`5.7-servicemanager-proxy.md`）
- **情景**：原生 C++ 行程呼叫 `defaultServiceManager()` 取得可與 ServiceManager 對話的代理物件。
- **起點 → 終點**：`libs/binder/IServiceManager.cpp::defaultServiceManager` → `CppBackendShim` 包裝 `getBackendUnifiedServiceManager()`（`libs/binder/BackendUnifiedServiceManager.cpp`）→ `ProcessState::self()->getContextObject(nullptr)` → `ProcessState::getStrongProxyForHandle(0)` → `BpBinder::create(0)` → `interface_cast<android::os::IServiceManager>` 建立 AIDL 生成的 `android::os::BpServiceManager` → 封裝進 `BackendUnifiedServiceManager`（附帶 `BinderCacheWithInvalidation`）。

### 5.8 Service 組件的啟動過程（`5.8-service-start.md`）
- **情景**：以 `GpuService`（`frameworks/native/services/gpuservice/main_gpuservice.cpp`）為例，將原生 Service 註冊到 ServiceManager 並啟動執行緒池。
- **起點 → 終點**：
  - 5.8.1 註冊 Service：`sm->addService` → `CppBackendShim::addService` → `BackendUnifiedServiceManager::addService` → AIDL `BpServiceManager::addService`（寫入 `Parcel::writeStrongBinder`）→ `BpBinder::transact` → `IPCThreadState::transact` → `talkWithDriver`（`BC_TRANSACTION`）→ `binder_transaction`（查 `context->binder_context_mgr_node`、呼叫 `binder_translate_binder` 建立 `binder_node` 與 ServiceManager 的 `binder_ref`、附加 `security_ctx`、喚醒 ServiceManager）→ ServiceManager `looper->pollAll` 醒來 → `handlePolledCommands` → `executeCommand(BR_TRANSACTION_SEC_CTX)` → `the_context_object->transact` → `BnServiceManager::onTransact` → `ServiceManager::addService`（`Access::canAdd` SELinux 權限檢查、記錄到 `mNameToService`、對服務 `linkToDeath`、通知 `IServiceCallback`）→ `sendReply`（`BC_REPLY`）→ `binder_transaction` 寫回並喚醒 Server → `waitForResponse` 收到 `BR_REPLY`；
  - 5.8.2 啟動 Binder 執行緒池：`ProcessState::setThreadPoolMaxThreadCount` → `startThreadPool` → `spawnPooledThread(true)` → `PoolThread::threadLoop` → `IPCThreadState::joinThreadPool(true)`（`BC_ENTER_LOOPER`）與主執行緒 `joinThreadPool`；併追蹤驅動在執行緒耗盡時下發 `BR_SPAWN_LOOPER` 的路徑。

### 5.9 Service 代理物件的取得過程（`5.9-service-proxy.md`）
- **情景**：Client 呼叫 `defaultServiceManager()->getService(name)` 查詢已註冊的 Service 並取得遠端代理物件。
- **起點 → 終點**：`CppBackendShim::getService` → `realGetService` → `BackendUnifiedServiceManager::getService2`（先查 `returnIfCached`）→ 未命中則走 AIDL `BpServiceManager::getService2` → `servicemanager` 的 `ServiceManager::getService2` → `tryGetService`（`Access::canFind`、查 `mNameToService`、找不到且已宣告時透過 `tryStartService` 啟動 Lazy Service）→ 將 `ServiceWithMetadata`（含 `BpBinder`）寫入 reply → `binder_transaction` 呼叫 `binder_translate_handle` 為 Client 建立指向 Server `binder_node` 的 `binder_ref` → Client 端 `Parcel::readStrongBinder` 建立 `BpBinder(handle)` → `BackendUnifiedServiceManager::updateCache`（非 lazy 服務時註冊 `linkToDeath` 寫入快取）→ `interface_cast<I<Name>>` 包裝為 `Bp<Name>`。

### 5.10 Binder 進程間通信機制的 Java 介面（`5.10-java-interface.md`）
- **情景**：從 Java 層完整貫通 ServiceManager 代理取得、AIDL Java 介面、Java 服務註冊、代理取得與遠端呼叫。
- **起點 → 終點**：
  - 5.10.1 ServiceManager 的 Java 代理取得：`ServiceManager.getIServiceManager()` → `ServiceManagerNative.asInterface(Binder.allowBlocking(BinderInternal.getContextObject()))` → `ServiceManagerProxy` 建構子呼叫 `getNativeServiceManager()`（`core/jni/android_os_ServiceManagerNative.cpp` → `libs/binder/IServiceManagerFFI.cpp::getJavaServicemanagerImpl...` → `getBackendUnifiedServiceManager()`）→ `javaObjectForIBinder` 建構 `BinderProxy` → `IServiceManager.Stub.asInterface`；
  - 5.10.2 Java 服務介面的定義和解析：對照 `system/tools/aidl/generate_java_binder.cpp` 與 `tests/golden_output/aidl-cpp-java-test-interface-java-source/` 解析 `I<Name>`、`Stub`、`Stub.Proxy`；
  - 5.10.3 Java 服務的啟動過程：`Binder` 建構子（`getNativeBBinderHolder()` 建立 `JavaBBinderHolder` 並交由 `NativeAllocationRegistry` 管理）→ `ServiceManager.addService` → `Parcel.writeStrongBinder` → `android_util_Binder.cpp::ibinderForJavaObject` 透過 `JavaBBinderHolder::get` 建立 `JavaBBinderExt`（繼承 `JavaBBinderBase` / `BBinder`）→ 進入底層註冊；
  - 5.10.4 Java 服務代理物件取得：`ServiceManager.getService`（先查 `sCache`，再呼叫 `rawGetService` → `ServiceManagerProxy.getService2` → C++ `BackendUnifiedServiceManager`）→ `javaObjectForIBinder` 透過 `BinderProxy.getInstance`（`ProxyMap` + `BinderProxyNativeData`）回傳 `BinderProxy` → `I<Name>.Stub.asInterface` 包裝成 `Stub.Proxy`；
  - 5.10.5 Java 服務的呼叫過程：`Stub.Proxy.<method>` → `BinderProxy.transact` → `transactNative`（`android_os_BinderProxy_transact`）→ `BpBinder::transact` → 核心驅動 → Server 端 `IPCThreadState::executeCommand` → `JavaBBinderExt::onTransact` → JNI 呼叫 `Binder.execTransact` → `execTransactInternal` → `Stub.onTransact` → 服務實作 → reply 返回。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

### 4.1 檔案路徑對照

| 原書 2.3 路徑 | Android 17 路徑（專案 + 相對路徑） | 變化類型 | 說明 |
|---|---|---|---|
| `kernel/goldfish/drivers/staging/android/binder.c` | `kernel/common`: `drivers/android/binder.c` + `binder_internal.h` + `binder_alloc.c` + `binder_alloc.h` + `binderfs.c` + `binder_pick.c` | 改名搬家 + 拆分 | 移出 staging 進入主線 `drivers/android/`；緩衝區分配器拆為 `binder_alloc.*`；新增 `binderfs.c` 與 `binder_pick.c`（另有 `drivers/android/binder/` Rust 實作） |
| `kernel/goldfish/drivers/staging/android/binder.h` | `kernel/common`: `include/uapi/linux/android/binder.h`（及 `binderfs.h`） | 改名搬家 | UAPI 標頭移至 `include/uapi/linux/android/` |
| `frameworks/base/include/binder/*` | `platform/frameworks/native`: `libs/binder/include/binder/*` | 搬家 | `libbinder` 從 `frameworks/base` 獨立遷入 `frameworks/native` |
| `frameworks/base/libs/binder/*` | `platform/frameworks/native`: `libs/binder/*` | 搬家 + 擴充 | 新增 `BackendUnifiedServiceManager.cpp`、`aidl/android/os/IServiceManager.aidl`、`IServiceManagerFFI.cpp`、`JavaBBinderBase.cpp`、`LazyServiceRegistrar.cpp`、`ndk/`、`rust/`、`Rpc*.cpp` |
| `frameworks/base/cmds/servicemanager/{service_manager.c,binder.c,binder.h}` | `platform/frameworks/native`: `cmds/servicemanager/{main.cpp,ServiceManager.cpp,ServiceManager.h,Access.cpp,servicemanager.rc,vndservicemanager.rc}` | 已移除並重寫 | 原 C 語言直呼 `ioctl` 的實作完全移除，改寫為 C++ + `libbinder` + AIDL `BnServiceManager` |
| `frameworks/base/core/java/android/os/Binder.java`（含內部類 `BinderProxy`） | `platform/frameworks/base`: `core/java/android/os/Binder.java` + `core/java/android/os/BinderProxy.java` | 拆分 | `BinderProxy` 拆為獨立公開類別，引入 `ProxyMap` 與 `NativeAllocationRegistry` |
| `frameworks/base/core/java/android/os/ServiceManager.java`、`ServiceManagerNative.java` | 同路徑（`platform/frameworks/base`） | 邏輯改變 | `ServiceManagerProxy` 改透過 JNI 呼叫 `BackendUnifiedServiceManager` + AIDL `IServiceManager` |
| `frameworks/base/core/jni/android_util_Binder.cpp` | `platform/frameworks/base`: `core/jni/android_util_Binder.cpp` + `core/jni/android_os_ServiceManagerNative.cpp` | 邏輯改變 + 新增 | 新增 `android_os_ServiceManagerNative.cpp` 橋接 Java ServiceManager；`JavaBBinder` 改為 `JavaBBinderExt : JavaBBinderBase` |
| `system/core/rootdir/init.rc` | 同路徑（`platform/system/core`） | 邏輯改變 | 新增掛載 `/dev/binderfs` 並建立 `/dev/{binder,hwbinder,vndbinder}` symlink |

### 4.2 核心驅動結構體與函式對照

| 原書 2.3 符號 | Android 17（`android17-6.18`）狀態 | 變化類型 | 說明 |
|---|---|---|---|
| `binder_lock()`（單一全域 mutex） | `binder_proc_lock`（`outer_lock`）、`binder_node_lock`（`node->lock`）、`binder_inner_proc_lock`（`inner_lock`） | 已移除 / 邏輯改變 | 改為三級細粒度自旋鎖，嚴格鎖序 `outer → node → inner` |
| `struct binder_proc`（內含 `buffer`/`vma`/`pages`/`free_buffers`/`allocated_buffers`） | `binder_internal.h::struct binder_proc` + `binder_alloc.h::struct binder_alloc` | 拆分 + 邏輯改變 | 記憶體分配欄位全移入 `proc->alloc`；新增 `context`、`dmap`（`dbitmap`）、`is_frozen`/`freeze_wait`、`inner_lock`/`outer_lock`、`waiting_threads` |
| `struct binder_buffer`（以彈性陣列 `data[0]` 內嵌於共享緩衝區） | `binder_alloc.h::struct binder_buffer`（獨立 `kzalloc`，以 `unsigned long user_data` 指向使用者位址） | 邏輯改變 | 不再將 metadata 放在共享記憶體中；新增 `extra_buffers_size`、`clear_on_free`、`oneway_spam_suspect` |
| `struct binder_node` / `struct binder_ref` / `struct binder_thread` / `struct binder_transaction` | 均移至 `binder_internal.h` | 邏輯改變 | `binder_node` 新增 `lock`、`tmp_refs`、`txn_security_ctx`、`sched_policy`/`inherit_rt`；`binder_ref` 計數抽為 `struct binder_ref_data` 並增 `freeze`；`binder_transaction` 新增 `security_ctx`、`fd_fixups`、`binder_priority` |
| （無） | `struct binder_device`、`struct binder_context`、`struct binderfs_info` | 新增 | 支援多 binder domain（`/dev/binder`、`/dev/hwbinder`、`/dev/vndbinder`）與 binderfs 隔離，各 device 獨立維護 `binder_context_mgr_node` |
| `binder_init`（註冊單一 `misc_register(&binder_miscdev)`） | `binder.c::binder_init` → `binder_alloc_shrinker_init` + `init_binderfs`（非 binderfs 時才依 `binder_devices_param` 呼叫 `init_binder_device`） | 邏輯改變 | Android 預設啟用 `CONFIG_ANDROID_BINDERFS`，由 `init.rc` 掛載時於 `binderfs_fill_super` 建立預設裝置 |
| `binder_open` | `binder.c::binder_open` | 邏輯改變 | 透過 `is_binderfs_device` 綁定所屬 `binder_context`、初始化 `dbitmap` 與雙鎖、呼叫 `binder_alloc_init` |
| `binder_mmap`（呼叫 `get_vm_area` + `binder_update_page_range` 預配 1 頁） | `binder.c::binder_mmap` → `binder_alloc.c::binder_alloc_mmap_handler` | 邏輯改變 | 移除 `get_vm_area` 與核心 VA 映射；mmap 時不分配任何實體頁，僅建立 `alloc->pages` 陣列與初始空閒 `binder_buffer` |
| `binder_alloc_buf` / `binder_free_buf` / `binder_buffer_lookup` / `binder_update_page_range` | `binder_alloc_new_buf` / `binder_alloc_free_buf` / `binder_alloc_prepare_to_free` / `binder_install_buffer_pages` + `binder_install_single_page` + `binder_lru_freelist_{add,del}` + `binder_alloc_free_page` | 改名搬家 + 邏輯改變 | 拆入 `binder_alloc.c`；實體頁按需裝入並以 LRU freelist + kernel shrinker 延遲回收；資料讀寫改用 `kmap_local_page` |
| `BINDER_SET_CONTEXT_MGR` | `BINDER_SET_CONTEXT_MGR_EXT`（優先，可帶 `FLAT_BINDER_FLAG_TXN_SECURITY_CTX`）與 `BINDER_SET_CONTEXT_MGR`（相容回退） | 擴充 | 對應 `binder_ioctl_set_ctx_mgr`，並與 `BR_TRANSACTION_SEC_CTX` 配合傳遞呼叫端 SELinux context |
| `binder_inc_node` / `binder_dec_node` / `binder_inc_ref` / `binder_dec_ref` | `binder_inc_node_nilocked` / `binder_dec_node_nilocked`、`binder_inc_ref_olocked` / `binder_dec_ref_olocked`、`binder_inc_ref_for_node` / `binder_dec_ref_for_handle` | 改名 + 邏輯改變 | 函式命名明確標記所需鎖狀態（`_ilocked` / `_olocked` / `_nilocked`） |
| `binder_release` → `binder_deferred_release` | 同名函式（含 `binder_node_release`、`binder_alloc_deferred_release`） | 邏輯改變 | 流程骨架保留，改以細粒度鎖與 `tmp_refs` 安全清理 |

### 4.3 使用者空間（`libbinder` / `servicemanager` / Java / JNI）對照

| 原書 2.3 符號 | Android 17（`android-17.0.0_r1`）狀態 | 變化類型 | 說明 |
|---|---|---|---|
| `ProcessState::self()` | `ProcessState::self()` + `ProcessState::initWithDriver(driver)` | 擴充 | 支援指定 `/dev/binder` 或 `/dev/vndbinder`；`BINDER_VM_SIZE = 1MB - 2*PAGE_SIZE`、`DEFAULT_MAX_BINDER_THREADS = 15`；加入 `pthread_atfork` 防 fork 後誤用 |
| `IPCThreadState::{self,transact,joinThreadPool,talkWithDriver,executeCommand}` | 同名函式保留，另增 `setupPolling` / `handlePolledCommands` / `setTheContextObject` / `getCallingSid` / `freeze` | 擴充 | 支援與 `Looper` 整合（供 `servicemanager` 單執行緒事件迴圈使用）及 SELinux SID / 行程凍結 |
| `BpBinder`（僅持有 `int32_t mHandle`） | `BpBinder`（`std::variant<BinderHandle, RpcHandle> mHandle`，工廠方法 `BpBinder::create`） | 邏輯改變 | 同時抽象核心 Binder 與 RPC Binder；加入 `BpBinder` 數量追蹤（Watermark）與凍結狀態回呼 |
| `IMPLEMENT_META_INTERFACE` | 加上 `static_assert(internal::allowedManualInterface(NAME))` | 邏輯改變 | 僅允許既有白名單，新介面一律強制使用 AIDL 產生 |
| `svcmgr_handler` / `binder_loop` / `do_add_service` / `do_find_service` | `ServiceManager::{addService,getService2,checkService2,...}`（繼承 `os::BnServiceManager`）+ `Looper::pollAll` + `BinderCallback` + `ClientCallbackCallback` | 已移除並重寫 | 完全改用 AIDL `IServiceManager.aidl`、SELinux `Access`、VINTF 查詢與 Lazy Service 機制 |
| `defaultServiceManager()` → `interface_cast<IServiceManager>(getContextObject)` | `defaultServiceManager()` → `CppBackendShim(getBackendUnifiedServiceManager())` → AIDL `android::os::BpServiceManager` | 邏輯改變 | 透過 `BackendUnifiedServiceManager` 統一提供 client-side 快取（`BinderCacheWithInvalidation`）與 `getService2` |
| `BinderService<T>` | 仍存在於 `BinderService.h`，但已標註 `WARNING: deprecated - DO NOT USE` | 棄用 | 現代服務（如 `main_gpuservice.cpp`）直接呼叫 `defaultServiceManager()->addService`、`startThreadPool`、`joinThreadPool` |
| Java `ServiceManagerProxy`（直接 `mRemote.transact` 手動打包） | `ServiceManagerProxy` 呼叫 `native getNativeServiceManager()`（`android_os_ServiceManagerNative.cpp` → `IServiceManagerFFI.cpp` → `BackendUnifiedServiceManager`）並交給 AIDL `IServiceManager.Stub.asInterface` | 邏輯改變 | Java 與 C++ 共用同一套 `BackendUnifiedServiceManager` 與 client-side cache |
| JNI `JavaBBinder` + `JavaBBinderHolder` | `JavaBBinderExt : android::internal::JavaBBinderBase` + `JavaBBinderHolder` | 改名 + 拆分 | 基底抽入 `frameworks/native/libs/binder/JavaBBinderBase.cpp` |
| Java `BinderProxy`（`Binder.java` 內部類） | 獨立 `BinderProxy.java`（含 `ProxyMap` 與 `BinderProxyNativeData`） | 搬家 + 邏輯改變 | 以 `NativeAllocationRegistry` 取代 `finalize()` 管理原生生命週期 |

---

## 五、已知 `TODO: 未確認` 項目

1. 第 5.1 節涵蓋資料結構、binderfs 初始化、open、mmap 與 `binder_alloc` 緩衝區管理，C 等級完整重寫篇幅較大；將於**檢查點 2** 寫完 5.1 後與使用者校準：保留為單一 `5.1-driver.md`，或拆為 `5.1a-driver-core.md` 與 `5.1b-driver-alloc.md`。
