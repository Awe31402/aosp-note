# 筆記工程進度

> 任務定義見 [BRIEF.md](BRIEF.md)，執行規劃見 [PLAN.md](PLAN.md)。基準：AOSP `android-17.0.0_r1`；kernel：GKI `kernel/common` 分支 `android17-6.18`（Linux 6.18，5.1 基準 `028363aa3cfa`，5.4–5.10 基準 `1e0de07`）。
> 每章節奏：**檢查點 1 大綱** → **檢查點 2 第一節** → **全章完成**。每個檢查點都要等使用者確認才往下推進。

## 目前狀態

- **現在**：**第 5、2、4、6、3、7、8、9 章皆已全章完成**；第 9 章〈Android 系統廣播機制〉共 4 節（`9.1`–`9.4`，2,792 行、31 個 Step、73 段程式碼、1,978 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、32 處 `〔與 2.3 差異〕`、5 張 PlantUML 圖表全數編譯並目視檢查通過）。
- **下一步**：進入第 10 章〈Content Provider 組件的實現原理〉（`ContentProviderHelper` / `ContentProviderRecord` / `IContentProvider`），查證 `android-17.0.0_r1` 原始碼並撰寫 `ch10-content-provider/00-outline.md`（檢查點 1）。
- **待決事項**：
  - [x] 確認 17 對應的 GKI kernel 分支名稱：已確認為 `kernel/common` 的 `android17-6.18`（`028363aa3cfa` / `1e0de07`）
  - [x] 確認 `ch05-binder/00-outline.md`（檢查點 1 通過）
  - [x] 確認 `ch05-binder/5.1-driver.md` 的深度、篇幅與格式（檢查點 2 通過）
  - [x] 完成第 5 章全章 `5.1`–`5.10`
  - [x] 完成第 2 章（AIDL HAL）大綱與全章 `2.1`–`2.5`
  - [x] 完成第 4 章（`logd + liblog`）大綱與全章 `4.1`–`4.6`
  - [x] 完成第 6 章（`memfd / 現行 ashmem`）大綱與全章 `6.1`–`6.5`
  - [x] 完成第 3 章（智能指標）大綱與全章 `3.1`–`3.4`
  - [x] 完成第 7 章（Activity 啟動過程）大綱與全章 `7.1`–`7.4`
  - [x] 完成第 8 章（Service 啟動與綁定過程）大綱與全章 `8.1`–`8.3`
  - [x] 撰寫並確認 `ch09-broadcast/00-outline.md`（第 9 章檢查點 1）
  - [x] 確認 `ch09-broadcast/9.1-architecture-and-scenario.md`（第 9 章檢查點 2）
  - [x] 完成第 9 章全章 `9.1`–`9.4`
  - [ ] 撰寫並確認 `ch10-content-provider/00-outline.md`（第 10 章檢查點 1）

## 章節進度

狀態圖例：⬜ 未開始 · 📝 大綱撰寫中 · ✅1 大綱已確認 · ✍️ 正文撰寫中 · ✅2 第一節已確認 · ✅ 全章完成 · ⏭️ 跳過

| 順序 | 原書章 | 新章名 / 主題 | 處理方式 | 狀態 |
|---|---|---|---|---|
| 1 | 第 5 章 Binder 進程間通信系統 | Binder（driver、libbinder、servicemanager、AIDL） | 試寫章，重追呼叫鏈（5.1–5.10 全章完成） | ✅ |
| 2 | 第 2 章 硬體抽象層 | AIDL HAL | 改寫成新機制（2.1–2.5 全章完成） | ✅ |
| 3 | 第 4 章 Logger 日誌系統 | logd + liblog | 改寫成新機制（4.1–4.6 全章完成） | ✅ |
| 4 | 第 6 章 Ashmem 匿名共享記憶體 | memfd / 現行 ashmem | 改寫成新機制（6.1–6.5 全章完成） | ✅ |
| 5 | 第 3 章 智能指標 | 智能指標（RefBase / sp / wp） | 重追（3.1–3.4 全章完成） | ✅ |
| 6 | 第 7 章 Activity 組件的啟動過程 | Activity 啟動過程（ATMS / `ActivityStarter` / `ClientTransaction`） | 重追（7.1–7.4 全章完成） | ✅ |
| 7 | 第 8 章 Service 組件的啟動過程 | Service 啟動與綁定過程（`ActiveServices` / FGS / `ServiceDispatcher`） | 重追（8.1–8.3 全章完成） | ✅ |
| 8 | 第 9 章 Android 系統廣播機制 | 廣播機制（`BroadcastController` / `BroadcastQueueImpl` / `BroadcastProcessQueue`） | 重追（9.1–9.4 全章完成） | ✅ |
| 9 | 第 10 章 Content Provider 組件的實現原理 | — | 重追 | ⬜ |
| 10 | 第 11 章 Zygote 和 System 進程的啟動過程 | — | 重追 | ⬜ |
| 11 | 第 12 章 Android 應用程式進程的啟動過程 | — | 重追 | ⬜ |
| 12 | 第 13 章 Android 應用程式的消息處理機制 | — | 重追 | ⬜ |
| 13 | 第 14 章 Android 應用程式的鍵盤消息處理機制 | Input 系統 | 重追 | ⬜ |
| 14 | 第 15 章 Android 應用程式線程的消息循環模型 | — | 重追 | ⬜ |
| 15 | 第 16 章 Android 應用程式的安裝和顯示過程 | — | 重追 | ⬜ |
| — | 第 1 章 準備知識 | 如何取得 / 閱讀 17 原始碼（一頁） | 環境建置跳過 | ⏭️ |

## 已確認的決定

| 日期 | 決定 |
|---|---|
| 2026-11-24 | 用途：個人學習筆記 |
| 2026-11-24 | 基準版本：`android-17.0.0_r1`（固定 tag，不用 `main`） |
| 2026-11-24 | 深度：C（完整重寫，詳細程度跟原書相同） |
| 2026-11-24 | 順序：第 5 章試寫 → 第 2、4、6 章改寫成新機制 → 其餘依原書順序；第 1 章跳過 |
| 2026-11-24 | 輸出：繁體中文 Markdown，每節結尾附時序圖，每個 Step 標〔與 2.3 差異〕 |
| 2026-11-24 | Kernel 基準：GKI `kernel/common` 分支 `android17-6.18`（Linux 6.18，commit `028363aa3cfa`） |
| 2026-11-25 | 圖表工具：改用 PlantUML 繪製（各章節 `figs/*.puml` + 白底高對比 `.png`，Markdown 引用 `.png`），取代內嵌 Mermaid |

## 工作日誌

- **2026-11-24**：透過 /define-task 完成任務定義，建立 `BRIEF.md`、`progress.md`。
- **2026-11-24**：建立 `PLAN.md`（工作流程、原始碼取得策略、第 5 章計畫）。其中的路徑和 kernel 分支尚未驗證。
- **2026-11-24**：完成第 5 章檢查點 1：驗證 `android-17.0.0_r1` manifest 與 GKI `kernel/common` `android17-6.18` 分支，淺 clone 相關專案至 `src/`，並完成 `ch05-binder/00-outline.md`（含原書 → 17 對照表），等待確認。
- **2026-11-24**：檢查點 1 通過；完成第 5 章檢查點 2 試寫節 `ch05-binder/5.1-driver.md`（涵蓋 5.1.1–5.1.5、三級鎖、binderfs、`binder_alloc` 無核心 VA 映射與 LRU shrinker，所有程式碼片段均已對照本機 clone 逐行驗證），等待使用者校準深度與篇幅。
- **2026-11-24**：檢查點 2 通過；完成第 5 章 `ch05-binder/5.2-libbinder.md`（涵蓋 `IBinder`/`IInterface`/`BnInterface`/`BpInterface`、`IMPLEMENT_META_INTERFACE` 白名單、`BBinder`（`PackedData` + `Extras`）、`BpRefBase`/`BpBinder`（`PrivateAccessor`、`variant<BinderHandle, RpcHandle>`、`Stability::check`、per-UID 水位）、`ProcessState`（`pthread_atfork`、`mHandleToObject`）、`IPCThreadState`（TLS、`transact`/`waitForResponse`/`talkWithDriver`/`executeCommand`）、`Parcel` 物件扁平化與 `libbinder_ndk`/`libbinder_rs`/RPC Binder 生態定位，並逐行比對原始碼驗證完畢）。
- **2026-11-24**：完成第 5 章 `ch05-binder/5.3-aidl-example.md`（以 `ICppJavaTests.aidl`、`generate_cpp.cpp` 與官方 golden output 解析 `aidl --lang=cpp` 四檔生成碼、`binder::Status`、`DO_NOT_DIRECTLY_USE_ME_IMPLEMENT_META_INTERFACE`、`TransactionCodeData`、`enforceNoDataAvail`、`Delegator`，並對照 `aidl_test_service.cpp`、`main_gpuservice.cpp`、`aidl_test_client` 與 NDK `codelab/client/testClient.cpp`，逐行比對驗證 0 筆不符）。
- **2026-11-24**：完成第 5 章剩餘全部小節 `5.4-refcount.md`、`5.5-death-notification.md`、`5.6-servicemanager-start.md`、`5.7-servicemanager-proxy.md`、`5.8-service-start.md`、`5.9-service-proxy.md`、`5.10-java-interface.md`；第 5 章全章 10 節（15,291 行、311 段程式碼）全數通過逐行原始碼驗證（0 筆不符、0 筆 `TODO: 未確認`、10 張 Mermaid 時序圖全數通過語法檢查），狀態推進至 ✅ 全章完成。
- **2026-11-24**：完成第 2 章〈AIDL HAL〉全章：`ch02-aidl-hal/00-outline.md` 與 `2.1-kernel-driver.md`–`2.5-app-client.md` 共 5 節（6,007 行、109 段程式碼、6 張 Mermaid 圖），以 `android.hardware.light`（GKI LED 子系統 + Rust HAL + VTS + Java 零 JNI `LightsService` + `SystemLightsManager`）為主線並對照 `android.hardware.vibrator`（C++ `libbinder_ndk` HAL + Vendor APEX + JNI `VibratorController`）與傳統 `libhardware`，全數通過逐行原始碼驗證（0 筆不符、0 筆 `TODO: 未確認`），狀態推進至 ✅ 全章完成。
- **2026-11-25**：完成第 4 章〈`logd` 與 `liblog` 日誌系統〉全章：`ch04-logd/00-outline.md` 與 `4.1-log-format.md`–`4.6-logcat.md` 共 6 節（10,018 行、159 段原始碼引用、46 處 `〔與 2.3 差異〕` 對照、各節附完整時序／結構圖），以 `platform/system/logging`（`d35ba62`）與 `platform/frameworks/base`（`94b4c163`）為基準，完整重寫日誌緩衝區與格式、`logd` 守護行程（`SerializedLogBuffer` + zstd 壓縮 + `PruneList` + `LogReaderThread` + `LogKlog`/`LogAudit`/`LogdNativeService`）、`liblog` 雙通道讀寫（`LogdSocket` + `/dev/pmsg0`）、C/C++/Rust 與 Java/JNI 介面及 `logcat` 工具分析，全數通過逐行與行號範圍原始碼驗證（0 筆不符、0 筆 `TODO: 未確認`），狀態推進至 ✅ 全章完成。
- **2026-11-25**：將 `aosp-note` 內第 2、4、5 章全部 27 張 Mermaid 圖表（21 張時序圖、3 張架構圖、3 張類別圖）改寫為 PlantUML（`.puml` + 白底高對比 `.png` 存於各章節 `figs/`），Markdown 改為圖片引用，並同步更新 `BRIEF.md` 與 `PLAN.md` 寫作規範。
- **2026-11-25**：完成第 6 章〈`memfd` 與現行 Ashmem 匿名共享記憶體系統〉檢查點 1 大綱 `ch06-memfd-ashmem/00-outline.md`（驗證 Linux 6.18 `mm/memfd.c` + Rust `ashmem_rust` 驅動、`libcutils/ashmem-dev.cpp` API 37 預設切 `memfd`、`MemoryHeapBase`/`IMemory`、`SharedMemory`/`MemoryFile` 與 Binder `binder_translate_fd`），等待確認。
- **2026-11-25**：第 6 章檢查點 1 通過；完成檢查點 2 試寫節 `ch06-memfd-ashmem/6.1-kernel-memfd-ashmem.md`（16 個 Step、30 段原始碼逐行驗證 0 筆不符、0 筆 `TODO: 未確認`，附 PlantUML 架構圖 `figs/6.1-arch.png` 與時序圖 `figs/6.1-seq.png`），等待確認深度與篇幅。
- **2026-11-25**：第 6 章檢查點 2 通過；完成第 6 章剩餘全部小節 `6.2-libcutils-ndk.md`、`6.3-cpp-interface.md`、`6.4-java-interface.md`、`6.5-sharing-mechanism.md`；第 6 章全章 5 節（2,932 行、36 個 Step、68 段程式碼、1,825 行逐行比對 0 筆不符、0 筆 `TODO: 未確認`、8 張 PlantUML 圖表全數編譯驗證），狀態推進至 ✅ 全章完成。
- **2026-11-25**：完成第 3 章〈智能指標（`LightRefBase` / `RefBase` / `sp` / `wp`）〉檢查點 1 大綱 `ch03-smart-pointer/00-outline.md`（對照原書 p.57–79 與 `platform/system/core/libutils/binder/`、`platform/frameworks/native/libs/binder/`、`platform/frameworks/base/core/jni/` 於 `android-17.0.0_r1` 驗證，整理為 `3.1`–`3.4` 四節與完整 2.3 → 17 對照表），等待確認。
- **2026-11-25**：第 3 章檢查點 1 通過；完成檢查點 2 第一節 `ch03-smart-pointer/3.1-light-pointer.md`（10 個 Step、15 段原始碼、350 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`，附 PlantUML 類別圖 `figs/3.1-class.png` 與時序圖 `figs/3.1-seq.png`），等待確認深度與篇幅。
- **2026-11-25**：第 3 章檢查點 2 通過；完成第 3 章剩餘全部小節 `3.2-strong-pointer.md`、`3.3-weak-pointer.md`、`3.4-practice-and-safety.md`；第 3 章全章 4 節（2,280 行、34 個 Step、61 段程式碼、1,320 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、6 張 PlantUML 圖表全數編譯並目視檢查通過），狀態推進至 ✅ 全章完成。
- **2026-11-25**：完成第 7 章〈Activity 組件的啟動過程〉檢查點 1 大綱 `ch07-activity/00-outline.md`（對照原書 p.400–450 與 `platform/frameworks/base` `android-17.0.0_r1` `94b4c163b` 驗證 ATMS、`ActivityStarter`、`TaskFragment`、`ClientLifecycleManager`、`ClientTransaction`、`TransactionExecutor` 與並行 `startProcessAsync`，規劃為 `7.1`–`7.4` 四節並附完整 2.3 → 17 對照表），等待確認。
- **2026-11-25**：第 7 章檢查點 1 通過；完成檢查點 2 第一節 `ch07-activity/7.1-architecture-and-scenario.md`（7 個 Step、17 段原始碼、246 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`，附 PlantUML 架構圖 `figs/7.1-arch.png` 與三情景總覽時序圖 `figs/7.1-seq.png`），等待確認深度與篇幅。
- **2026-11-25**：第 7 章檢查點 2 通過；完成第 7 章剩餘全部小節 `7.2-root-activity-request-and-pause.md`、`7.3-root-activity-process-and-launch.md`、`7.4-sub-activity-in-and-across-process.md`；第 7 章全章 4 節（2,055 行、29 個 Step、66 段程式碼、1,220 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、5 張 PlantUML 圖表全數編譯並目視檢查通過），狀態推進至 ✅ 全章完成。
- **2026-11-25**：完成第 8 章〈Service 組件的啟動過程〉檢查點 1 大綱 `ch08-service/00-outline.md`（對照原書 p.451–493 與 `platform/frameworks/base` `android-17.0.0_r1` `94b4c163b` 驗證 `ActiveServices`、`ServiceMap`、`ProcessServiceRecord`、FGS 與背景啟動管制、`bindServiceInstance`、`bindToken` 與 `LoadedApk.ServiceDispatcher`，規劃為 `8.1`–`8.3` 三節並附完整 2.3 → 17 對照表），等待確認。
- **2026-11-25**：第 8 章檢查點 1 通過；完成檢查點 2 第一節 `ch08-service/8.1-architecture-and-scenario.md`（6 個 Step、13 段原始碼、153 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`，附 PlantUML 類別圖 `figs/8.1-class.png` 與雙情景總覽時序圖 `figs/8.1-seq.png`），等待確認深度與篇幅。
- **2026-11-25**：第 8 章檢查點 2 通過；完成第 8 章剩餘全部小節 `8.2-start-service-new-process.md`、`8.3-bind-service-in-process.md`；第 8 章全章 3 節（2,416 行、28 個 Step、62 段程式碼、1,569 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、4 張 PlantUML 圖表全數編譯並目視檢查通過），狀態推進至 ✅ 全章完成。
- **2026-11-26**：完成第 9 章〈Android 系統廣播機制〉檢查點 1 大綱 `ch09-broadcast/00-outline.md`（對照原書 p.494–531 與 `platform/frameworks/base` `android-17.0.0_r1` `94b4c163b` 驗證 `BroadcastController`、已更名為唯一實作的 `BroadcastQueueImpl`、Per-Process `BroadcastProcessQueue`、`BroadcastRecord.blockedUntilBeyondCount[]`、`BroadcastSkipPolicy`、`BroadcastStickyCache`、`RECEIVER_EXPORTED`/`NOT_EXPORTED` 與 `AnrTimer`，規劃為 `9.1`–`9.4` 四節並附完整 2.3 → 17 對照表），等待確認。
- **2026-11-26**：第 9 章檢查點 1 通過；完成檢查點 2 第一節 `ch09-broadcast/9.1-architecture-and-scenario.md`（516 行、6 個 Step、18 段原始碼、312 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`，附 PlantUML 架構類別圖 `figs/9.1-arch.png` 與三情景總覽時序圖 `figs/9.1-seq.png` 並目視檢查通過），等待確認深度與篇幅。
- **2026-11-26**：第 9 章檢查點 2 通過；完成第 9 章剩餘全部小節 `9.2-register-receiver.md`、`9.3-send-and-dispatch-broadcast.md`、`9.4-ordered-manifest-and-finish.md`；第 9 章全章 4 節（2,792 行、31 個 Step、73 段程式碼、1,978 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、32 處 `〔與 2.3 差異〕`、5 張 PlantUML 圖表全數編譯並目視檢查通過），狀態推進至 ✅ 全章完成。
