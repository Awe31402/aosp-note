# 第 10 章：Content Provider 組件的實現原理 — 大綱

> **原書對應**：第 10 章〈Content Provider组件的实现原理〉（p.532–618）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：本章主體為使用者空間應用框架、`system_server`（AMS / `ContentProviderHelper` / `ContentService`）與原生 `libandroidfw`（`CursorWindow.cpp`），共享記憶體底層透過 `libcutils` 的 `ashmem_create_region` 銜接第 6 章的 GKI Linux 6.18 `memfd` / `ashmem_rust`
> **撰寫日期**：2026-11-27
> **狀態**：✅ 全章完成（`10.1`–`10.4` 共 2,308 行、28 個 Step、47 段程式碼、1,622 行逐行比對 0 筆不符、0 筆 `TODO: 未確認`、29 處 `〔與 2.3 差異〕`、5 張 PlantUML 圖表全數編譯並目視檢查通過）

---

## 一、原始碼專案清單與版本驗證

本章所有呼叫鏈（涵蓋 Java 框架、`system_server`、JNI 與原生 C++ `CursorWindow`）皆位於 `platform/frameworks/base`，已於本機 `src/` 檢出並驗證對應版本：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 10 章涵蓋範圍 |
|---|---|---|---|
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163b` | **App / Client 與 Provider 端**：`core/java/android/app/{ActivityThread.java,ContentProviderHolder.java,IActivityManager.aidl}`、`core/java/android/content/{ContentResolver.java,ContentProvider.java,ContentProviderNative.java,IContentProvider.java,IContentService.aidl}`、`core/java/android/database/{BulkCursorDescriptor.java,BulkCursorNative.java,BulkCursorToCursorAdaptor.java,CursorToBulkCursorAdaptor.java,CursorWindow.java,ContentObserver.java,ExecutorContentObserver.java,IContentObserver.aidl}`、`core/java/android/database/sqlite/{SQLiteCursor.java,SQLiteQuery.java}`<br>**`system_server` 端**：`services/core/java/com/android/server/am/{ActivityManagerService.java,ContentProviderHelper.java,ContentProviderRecord.java,ContentProviderConnection.java,ProviderMap.java,ProcessProviderRecord.java}`、`services/core/java/com/android/server/content/ContentService.java`<br>**JNI / Native 共享視窗**：`core/jni/android_database_CursorWindow.cpp`、`libs/androidfw/{CursorWindow.cpp,include/androidfw/CursorWindow.h}` |

> **重大架構演進說明（2.3 → 17）**：
> 1. **從單體 AMS 抽離為 `ContentProviderHelper` + 多使用者 `ProviderMap` + Stable/Unstable 連線雙計數**：原書 2.3 將所有 Provider 映射表與啟動發布邏輯全塞在 `ActivityManagerService.java` 內，且無穩定／不穩定引用區分（Provider 進程崩潰常連帶殺死客戶端進程）；Android 17 抽離為 **`ContentProviderHelper`**（`AMS.mCpHelper`）、區分 singleton 與 per-user 的 **`ProviderMap`**、進程側 **`ProcessProviderRecord`**，並以繼承 `Binder` 的 **`ContentProviderConnection`** 維護 `mStableCount` 與 `mUnstableCount`——`ContentResolver.query()` 先取 **`acquireUnstableProvider`**，遇 `DeadObjectException` 時呼叫 `unstableProviderDied()` 並改取 **`acquireProvider`（stable）**重試，大幅提升跨進程查詢容錯能力。
> 2. **`CursorWindow` 所有權翻轉與「16 KiB 內聯 → 延遲膨脹至 Ashmem/`memfd`」革命**：原書 2.3 由**客戶端**先配置固定 1 MiB Ashmem（經 `MemoryHeapBase`/`IMemory`）傳入 `bulkQuery()`，再二次 IPC 呼 `count()` 填充資料；Android 17 徹底反轉——由 **Provider 側**（`ContentProvider.Transport.query` → `CursorToBulkCursorAdaptor`）建好並填滿第一頁 `CursorWindow`，封裝為 **`BulkCursorDescriptor`** 單次 IPC 傳回客戶端 `BulkCursorToCursorAdaptor`。同時在原生層 `libs/androidfw/CursorWindow.cpp`，預設只配置 **`kInlineSize = 16384`（16 KiB）內聯 `calloc` 緩衝區**（序列化時直接緊縮寫進 Binder Parcel，零 fd、零 `mmap`），唯有資料超過 16 KiB 時才由 **`maybeInflate()`** 延遲建立 `ashmem_create_region`（在 Android 17 / API 37 預設切至 **`memfd`**，直連第 6 章）並以 `PROT_READ` 唯讀 fd 共享給客戶端！
> 3. **安全邊界與 `ContentService` 現代批次通知**：`IContentProvider` 所有方法改由 **`AttributionSource`** 傳遞完整權限歸因鏈（支援多跳代理歸責）並支援 `ICancellationSignal` 跨進程取消查詢；`ContentService` 強制 `checkContentProviderAccess` 檢查（API 26+ 越權拋 `SecurityException`），`notifyChange` 升級為 **`Uri[] uris` 批次陣列 + `int flags`**（`NOTIFY_INSERT`/`UPDATE`/`DELETE`/`SYNC_TO_NETWORK`），經 **`ObserverCollector`** 彙整後以 `oneway IContentObserver.onChangeEtc` 推播，並新增 **`ExecutorContentObserver`** 支援自訂執行緒池派發。

---

## 二、小節規劃與原書對照

本章完整保留原書 4 大節（10.1–10.4）的經典四部曲——架構與情景 → 啟動與發布 → 跨進程 `CursorWindow` 共享 → `ContentService` 更新通知（圖表皆以 PlantUML 繪製，存於 `ch10-content-provider/figs/`）：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `10.1-architecture-and-scenario.md` | 10.1 現代 Content Provider 架構、資料模型與核心情景（`ContentProviderHelper`、`ProviderMap` 與 Stable/Unstable 連線） | 10.1（p.532–557） | **改寫為架構與情景定錨節**：延續原書 `ArticlesProvider` 與 `Article` 雙應用實例，確立「按需啟動發布」、「跨進程 `CursorWindow` 查詢」與「`ContentObserver` 更新通知」三大情景；解剖 `ContentProviderHelper`、`ProviderMap`、`ContentProviderRecord`、`ContentProviderConnection`（`mStableCount`/`mUnstableCount`）、`ProcessProviderRecord`、`ContentProviderHolder` 及跨進程四組介面邊界 |
| `10.2-provider-launch-and-publish.md` | 10.2 Content Provider 組件的啟動、發布與連線計數過程（`acquireProvider`、`ContentProviderHelper` 與 `publishContentProviders`） | 10.2（p.558–580） | **保留並深度重追**：從 `ContentResolver.acquireUnstableProvider` / `acquireProvider` → `ApplicationContentResolver` → `ActivityThread.acquireProvider`（`acquireExistingProvider` 本地快取與引用計數）→ `AMS.getContentProvider` → `ContentProviderHelper.getContentProviderImpl`（`ProviderMap`、權限檢查、`mLaunchingProviders`、`startProcessLocked`、`incProviderCountLocked`、`cpr.wait()` 阻塞等待）→ 新進程 `attachApplication` → `bindApplication`（`ProviderInfoList`）→ `handleBindApplication` → `installContentProviders` / `installProvider`（`instantiateProvider`、`attachInfo`、`onCreate()`）→ `publishContentProviders`（`cpr.notifyAll()`）→ 客戶端 `installProvider` 登記並返回 |
| `10.3-data-sharing-cursor-window.md` | 10.3 Content Provider 的跨進程資料共享原理（`CursorWindow` 16 KiB 內聯延遲膨脹至 Ashmem/`memfd` 與 `BulkCursorDescriptor`） | 10.3（p.581–603） | **保留並徹底重寫現代路徑**：剖析 2.3 → 17 方向翻轉的共享模型，沿 `ContentResolver.query`（Unstable 優先 + `DeadObjectException` 降級 Stable 重試 + `CursorWrapperInner`）→ `ContentProviderProxy.query` → `ContentProviderNative.onTransact` → `ContentProvider.Transport.query` → `CursorToBulkCursorAdaptor.getBulkCursorDescriptor`（`SQLiteCursor.fillWindow` → 原生 `CursorWindow`）→ `CursorWindow::writeToParcel` / `maybeInflate()`（16 KiB 內聯緊縮 vs `ashmem_create_region`/`memfd` 唯讀 fd）→ `CursorWindow::createFromParcel` → `BulkCursorToCursorAdaptor.initialize` 與翻頁 `onMove()` 完整走通 |
| `10.4-content-observer-notification.md` | 10.4 Content Provider 的資料更新通知機制（`ContentService`、`ObserverNode` 樹與 `ContentObserver`） | 10.4（p.604–618） | **保留並深度重追雙子情景**：(1) 註冊觀察者：`ContentResolver.registerContentObserver` → `ContentObserver.getContentObserver`（`Transport`）→ `ContentService.registerContentObserver`（跨使用者與 `checkContentProviderAccess`）→ `ObserverNode.addObserverLocked`（URI 前綴樹 + `DeathRecipient`）；(2) 批次更新通知：`ContentResolver.notifyChange` → `ContentService.notifyChange` → `ObserverNode.collectObserversLocked` + `ObserverCollector` → `oneway IContentObserver.onChangeEtc` → `ContentObserver.dispatchChange`（`Handler` / `ExecutorContentObserver`）→ `onChange(...)` |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 10.1 現代 Content Provider 架構、資料模型與核心情景（`10.1-architecture-and-scenario.md`）
- **情景**：延續原書 `ArticlesProvider`（獨立進程的資料庫提供者）與 `Article`（消費端 UI 應用），先對照三大核心情景，再解剖 `system_server` 與應用進程雙側的資料結構與 IPC 契約。
- **起點 → 終點**：`AMS.mCpHelper`（`ContentProviderHelper`）→ `ProviderMap`（`mSingletonByName`/`ByClass` + `mProvidersByNamePerUser`/`ByClassPerUser`）→ `ContentProviderRecord` → `ContentProviderConnection`（`mStableCount`/`mUnstableCount`）→ `ProcessProviderRecord`；`ContentProviderHolder`；`IActivityManager` / `IContentProvider` / `IBulkCursor` / `IContentService` / `IContentObserver` 介面邊界。

### 10.2 Content Provider 組件的啟動、發布與連線計數過程（`10.2-provider-launch-and-publish.md`）
- **情景**：`Article` 進程首度存取尚未啟動的 `ArticlesProvider`（`content://shy.luo.providers.articles/...`），客戶端請求連線、AMS 孵化 Provider 進程並讓呼叫執行緒阻塞等待，直到新進程建立 Provider 實例並透過 `publishContentProviders` 喚醒客戶端。
- **起點 → 終點**：
  - 客戶端查快取與發起請求：`ContentResolver.acquireUnstableProvider` / `acquireProvider` → `ContextImpl.ApplicationContentResolver.acquireProvider` → `ActivityThread.acquireProvider`（`acquireExistingProvider` 未命中 `mProviderMap`）→ `ActivityManager.getService().getContentProvider(caller, pkg, auth, userId, stable)`
  - 系統端解析、冷啟與等待：`AMS.getContentProvider` → `ContentProviderHelper.getContentProvider` → `getContentProviderImpl`（`mProviderMap.getProviderByName` 未命中 → `PackageManager.resolveContentProvider` → `checkContentProviderPermissionLocked` → 建 `ContentProviderRecord` 加入 `mLaunchingProviders` → 查無進程呼叫 `startProcessLocked(..., HOSTING_TYPE_CONTENT_PROVIDER)` + `incProviderCountLocked` 建 `ContentProviderConnection` → 釋放 AMS 主鎖並在 `synchronized (cpr)` 中 `cpr.wait(...)`）
  - Provider 進程安裝與發布：`ActivityThread.main()` → `AMS.attachApplicationLocked`（`generateApplicationProvidersLocked` 收集 `ProviderInfoList` 隨 `bindApplication` 下發）→ `ActivityThread.handleBindApplication` → `installContentProviders` → `installProvider`（`AppComponentFactory.instantiateProvider`、`localProvider.attachInfo` → **`ArticlesProvider.onCreate()`**、`getIContentProvider()` 取得 `Transport`）→ `AMS.publishContentProviders` → `ContentProviderHelper.publishContentProviders`（填回 `cpr.mProvider`、移出 `mLaunchingProviders`、呼叫 **`cpr.notifyAll()`** 喚醒等待中的客戶端）
  - 客戶端安裝代理：`getContentProviderImpl` 返回 `ContentProviderHolder` → 客戶端 `ActivityThread.installProvider` 寫入本地 `mProviderMap` 與 `mProviderRefCountMap`（`ProviderRefCount` 維護 stable/unstable 計數）並返回 `IContentProvider` 代理。

### 10.3 Content Provider 的跨進程資料共享原理（`10.3-data-sharing-cursor-window.md`）
- **情景**：`Article` 呼叫 `ContentResolver.query(uri, projection, queryArgs, cancellationSignal)` 跨進程查詢 `ArticlesProvider` 的 SQLite 資料庫，分別追蹤「結果 ≤ 16 KiB（內聯 Parcel 直傳）」與「結果 > 16 KiB（延遲膨脹為 Ashmem/`memfd` 唯讀零拷貝共享）」兩條路徑及跨視窗翻頁機制。
- **起點 → 終點**：
  - 客戶端防崩潰查詢入口：`ContentResolver.query`（`acquireUnstableProvider` → 嘗試 `unstableProvider.query(...)` → 若拋 `DeadObjectException` 則 `unstableProviderDied` 並降級 `acquireProvider` 重試 → `qCursor.getCount()` → 包裝為 `CursorWrapperInner` 確保 `close()` 時釋放連線）
  - 跨進程查詢與打包：`ContentProviderProxy.query`（建 `BulkCursorToCursorAdaptor`、跨進程 `QUERY_TRANSACTION` 傳 `AttributionSource` + `uri` + `queryArgs` + `IContentObserver` + `ICancellationSignal`）→ `ContentProviderNative.onTransact` → `ContentProvider.Transport.query`（`enforceReadPermission` → **`ArticlesProvider.query()`** → `SQLiteCursor`）→ `CursorToBulkCursorAdaptor` → `getBulkCursorDescriptor()`（觸發 `SQLiteCursor.getCount()` → `fillWindow()` → `SQLiteQuery.fillWindow()` 透過 JNI 寫入原生 `CursorWindow`）
  - 原生 `CursorWindow` 雙模態序列化：`BulkCursorDescriptor.writeToParcel` → `CursorWindow.writeToParcel` → `nativeWriteToParcel` → `CursorWindow::writeToParcel`（小視窗：`mAshmemFd == -1`，緊縮 `sizeInUse()` 直接 `parcel->writeInplace()`；大視窗：寫入期間觸發 `maybeInflate()` 呼叫 `ashmem_create_region` + `MappedFile::Create(RW)` + `ashmem_set_prot_region(PROT_READ)` 並搬移資料，序列化時 `parcel->writeDupFileDescriptor(mAshmemFd)`）
  - 客戶端映射與翻頁：`BulkCursorDescriptor.readFromParcel` → `CursorWindow::createFromParcel`（內聯 `malloc` 讀取，或 `F_DUPFD_CLOEXEC` + `MappedFile::Create(..., PROT_READ)` 唯讀映射，`mReadOnly = true`）→ `BulkCursorToCursorAdaptor.initialize(descriptor)`；當讀取列號超出當前視窗範圍時，`BulkCursorToCursorAdaptor.onMove` 透過 `IBulkCursor.getWindow(newPosition)` 向遠端重新填充新視窗。

### 10.4 Content Provider 的資料更新通知機制（`10.4-content-observer-notification.md`）
- **情景**：`Article` 註冊 `ContentObserver` 監聽 `content://shy.luo.providers.articles/item`，當 `ArticlesProvider.insert()` 寫入新文章並呼叫 `ContentResolver.notifyChange()` 時，`ContentService` 遍歷 `ObserverNode` 前綴樹彙整並推播至觀察者。
- **起點 → 終點**：
  - 註冊觀察者：`ContentResolver.registerContentObserver` → `ContentObserver.getContentObserver()`（建立 `Transport extends IContentObserver.Stub`）→ `ContentResolver.getContentService().registerContentObserver(uri, notifyForDescendants, observer, userHandle, targetSdkVersion)` → `ContentService.registerContentObserver`（`handleIncomingUser` + `ActivityManagerInternal.checkContentProviderAccess`）→ `mRootNode.addObserverLocked(...)`（依 authority 與 path segments 遞迴建樹，掛入 `ObserverEntry` 並 `linkToDeath` 監控觀察者死亡）。
  - 發送與接收更新通知：`ArticlesProvider.insert()` → `ContentResolver.notifyChange(uri, observer, flags)` → `ContentService.notifyChange(Uri[] uris, ...)`（`checkContentProviderAccess` 驗證權限 → 遍歷每個 URI 呼叫 `mRootNode.collectObserversLocked(...)` 收集精確節點與 `notifyForDescendants` 祖先節點 → 匯入 `ObserverCollector` 依 `(observer, userId, selfChange, flags)` 合併批次 `Uri[]` → `collector.dispatch()`）→ `oneway IContentObserver.onChangeEtc(selfChange, uris, flags, userId)` → `ContentObserver.Transport.onChangeEtc` → `ContentObserver.dispatchChange`（`mHandler.post(...)` 或 `ExecutorContentObserver` 的 `mExecutor.execute(...)`）→ **`ContentObserver.onChange(selfChange, uris, flags)`**（向下相容轉呼 `onChange(selfChange, uri)`）。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| `ActivityManagerService.{mProvidersByName,mProvidersByClass,mLaunchingProviders,getContentProviderImpl,publishContentProviders}` | `com.android.server.am.ContentProviderHelper` + `ProviderMap`（singleton + per-user）+ `ProcessProviderRecord` | 搬家 + 拆分 | Provider 管理全面自單體 AMS 抽離至 `ContentProviderHelper`，支援多使用者與 Clone Profile 重導向 |
| `ContentProviderRecord.clients`（單純 `HashSet<ProcessRecord>`，無穩定度區分） | `ContentProviderRecord.mConnections`（`ArrayList<ContentProviderConnection>`，含 `mStableCount` / `mUnstableCount`）+ `IActivityManager.{refContentProvider,unstableProviderDied}` | 邏輯改變 + 新增 | 引入 Stable / Unstable 雙引用計數；`ContentResolver.query` 優先使用 Unstable 連線，Provider 死亡時可重試而不連坐殺死客戶端 |
| `ActivityManagerProxy` / `ActivityManagerNative`（手寫 Binder） | AIDL 生成 `IActivityManager`（`getContentProvider`、`publishContentProviders`、`refContentProvider`、`oneway removeContentProvider`） | 邏輯改變 | 全面 AIDL 化；新增 `callingPackage`、`userId`、`stable` 與 `oneway` 釋放 |
| `IContentProvider.bulkQuery(uri, ..., CursorWindow, ...)`（客戶端建視窗傳給 Provider） | `IContentProvider.query(AttributionSource, uri, projection, Bundle queryArgs, ICancellationSignal)` → 回傳 `BulkCursorDescriptor` | 邏輯徹底反轉 | 改由 Provider 端建構並填滿 `CursorWindow`，連同 `IBulkCursor`、`columnNames`、`count` 打包為 `BulkCursorDescriptor` 單次 IPC 傳回；支援 `AttributionSource` 與 `ICancellationSignal` |
| `CursorWindow` 固定配置 1 MiB `MemoryHeapBase` / `IMemory`（`core/jni/CursorWindow.cpp`） | `libs/androidfw/CursorWindow.cpp`（預設 `kInlineSize = 16384` 內聯 `calloc`，超限時 `maybeInflate()` 轉 `ashmem_create_region` / `memfd` + `PROT_READ` fd） | 搬家 + 邏輯徹底重寫 | 小結果集直寫 Binder Parcel 省去 fd 與 `mmap`；大結果集才膨脹至匿名共享記憶體（API 37 預設走 `memfd`）並設唯讀保護 |
| `ActivityThread.installProvider`（`ClassLoader.loadClass().newInstance()`） | `ActivityThread.installProvider`（`AppComponentFactory.instantiateProvider`、split Context、`mProviderRefCountMap` 區分 stable/unstable） | 邏輯改變 | 改由 `AppComponentFactory` 實例化並追蹤本機與系統雙層引用計數 |
| `IContentService.registerContentObserver(uri, desc, observer)` / `notifyChange(uri, observer, self, syncToNetwork)` | `IContentService.registerContentObserver(uri, desc, observer, userHandle, targetSdkVersion)` / `notifyChange(Uri[] uris, ..., int flags, int userId, int targetSdkVersion, String callingPackage)` | 邏輯改變 + 擴充 | 支援多使用者、`checkContentProviderAccess` 強制權限檢查、`Uri[]` 批次通知與細粒度 `NOTIFY_*` 旗標 |
| `ContentService.notifyChange` 逐筆直呼 `IContentObserver.onChange(boolean)` | `ObserverCollector` 彙整 + `oneway IContentObserver.onChangeEtc(selfUpdate, Uri[], flags, userId)` + `ExecutorContentObserver` | 邏輯改變 + 新增 | 減少重複 IPC 次數並支援帶 `Uri` 集合／操作旗標與 `Executor` 非同步回調 |
| —（無已知未確認項目） | — | — | 本章所有類別、路徑與方法簽名皆已於 `platform/frameworks/base`（`android-17.0.0_r1`，`94b4c163b`）實機驗證，0 筆 `TODO: 未確認` |
