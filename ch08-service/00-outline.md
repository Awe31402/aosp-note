# 第 8 章：Service 組件的啟動過程 — 大綱

> **原書對應**：第 8 章〈Service 组件的启动过程〉（p.451–493）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：本章為使用者空間應用框架與 `system_server`（AMS / `ActiveServices`），不涉及 Linux 核心驅動（底層 Binder IPC 詳見第 5 章，Zygote 建程細節留待第 12 章）
> **撰寫日期**：2026-11-25
> **狀態**：✅ 全章完成（`8.1`–`8.3` 共 2,416 行、28 個 Step、62 段程式碼、1,569 行逐行比對 0 筆不符、0 筆 `TODO: 未確認`、18 處 `〔與 2.3 差異〕`、4 張 PlantUML 圖表全數編譯並目視檢查通過）

---

## 一、原始碼專案清單與版本驗證

本章所有呼叫鏈皆位於 `platform/frameworks/base`，已於本機 `src/` 檢出並驗證對應版本：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 8 章涵蓋範圍 |
|---|---|---|---|
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163b` | **App / Client 端**：`core/java/android/content/ContextWrapper.java`、`core/java/android/app/{ContextImpl.java,LoadedApk.java,Service.java,ActivityThread.java,IActivityManager.aidl,IApplicationThread.aidl,IServiceConnection.aidl}`<br>**`system_server` / AM 端**：`services/core/java/com/android/server/am/{ActivityManagerService.java,ActiveServices.java,ServiceRecord.java,IntentBindRecord.java,AppBindRecord.java,ConnectionRecord.java,ProcessServiceRecord.java}` |

> **重大架構演進說明（2.3 → 17）**：
> 1. **從單體 AMS 抽離為獨立子系統 `ActiveServices`（但未併入 ATMS/WM）**：與第 7 章 Activity 遷往 `com.android.server.wm` 不同，Service 在 Android 17 仍歸屬 `com.android.server.am`，但原本散落在 `ActivityManagerService.java` 內的數千行 Service 邏輯已徹底抽離至 **`ActiveServices.java`**，搭配按多使用者切分的 **`ServiceMap`** 與進程側 **`ProcessServiceRecord`**（`app.mServices`）；當 Activity 綁定 Service 時，則透過 `mAtmInternal.getServiceConnectionsHolder(token)` 跨服務取得 `ActivityServiceConnectionsHolder<ConnectionRecord>`，避免 AMS 直接操作 WM 的 `ActivityRecord`。
> 2. **顯式 Intent 強制、背景啟動限制與前台服務（FGS）合規管制**：原書 2.3 允許任意隱式 Intent 在背景無限制 `startService`；Android 17 強制 `validateServiceIntent`（隱式 Intent 拋出 `IllegalArgumentException`）、導入 **`startForegroundService()`**（`fgRequired`）、在 `ActiveServices.startServiceLocked` 嚴格執行背景啟動限制（`bgLaunch` / `BackgroundServiceStartNotAllowedException`）與 FGS 啟動白名單／型別超時檢查（`setFgsRestrictionLocked`、`SHORT_SERVICE` 等）。
> 3. **仍保留專屬 `IApplicationThread` RPC，而非 `ClientTransaction`**：不同於 Activity 全部收斂為 `scheduleTransaction(ClientTransaction)`，Android 17 的 Service 生命週期仍走 `IApplicationThread.aidl` 上的 `oneway` 專屬方法——**`scheduleCreateService`**、**`scheduleServiceArgs`**（以 `ParceledListSlice` 批次傳遞多筆 `StartItem`）與 **`scheduleBindService`**（新增 `IBinder bindToken` 精準識別該次 `IntentBindRecord`，而非單靠 `Intent` 比對）。
> 4. **現代綁定強化（`bindServiceInstance`、64 位元旗標、`Executor` 與 `IBinderSession`）**：`ContextImpl.bindServiceCommon` 改呼叫 `IActivityManager.bindServiceInstance(...)`（支援 64 位元 `long flags`、`Context.BindServiceFlags`、Isolated/SDK Sandbox 具名實例 `instanceName`），`LoadedApk.ServiceDispatcher` 支援傳入自訂 **`Executor`** 取代主執行緒 `Handler`，而 `IServiceConnection.connected(name, service, session, dead)` 新增了 `IBinderSession` 與 `dead` 旗標。

---

## 二、小節規劃與原書對照

本章維持原書 3 大節（8.1–8.3）的經典結構，完整對照 Android 17 實作並補齊原書 8.2 未詳追的 `onStartCommand`（`sendServiceArgsLocked` → `scheduleServiceArgs`）與 FGS／背景啟動管制（圖表皆以 PlantUML 繪製，存於 `ch08-service/figs/`）：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `8.1-architecture-and-scenario.md` | 8.1 現代 Service 管理架構、資料模型與雙情景總覽（`ActiveServices`、四層綁定模型與 FGS） | 8.1（p.451–459） | **改寫為架構與情景定錨節**：建立全章「新進程 `startService`/`startForegroundService`」與「同進程 `bindService`」雙情景；拆解 `ActiveServices`、`ServiceMap`、`ProcessServiceRecord` 與四層核心資料結構（`ServiceRecord`、`IntentBindRecord`、`AppBindRecord`、`ConnectionRecord`），以及 `IActivityManager` / `IApplicationThread` / `IServiceConnection` 三組 AIDL 介面與 `ActivityServiceConnectionsHolder` 跨鎖設計 |
| `8.2-start-service-new-process.md` | 8.2 Service 在新進程中的啟動過程（`startService` / `startForegroundService`、`ActiveServices`、`onCreate` 與 `onStartCommand`） | 8.2（p.459–471） | **保留並深度重追**：從 `ContextWrapper.startService` / `startForegroundService` → `ContextImpl.startServiceCommon` → `AMS.startService` → `ActiveServices.startServiceLocked`（`retrieveServiceLocked`、背景啟動與 FGS 管制）→ `startServiceInnerLocked`（`pendingStarts`）→ `bringUpServiceLocked`（查無進程 → `startProcessLocked` + `mPendingServices`）→ 新進程 `attachApplication` → `ActiveServices.attachApplicationLocked` → `realStartServiceLocked`（`SyncBatchSession`、`bumpServiceExecutingLocked`、`scheduleCreateService`、`sendServiceArgsLocked`）→ `ActivityThread` 執行 `handleCreateService`（`onCreate`）與 `handleServiceArgs`（`onStartCommand`）並回報 `serviceDoneExecuting` |
| `8.3-bind-service-in-process.md` | 8.3 Service 的綁定與回調過程（`bindService`、`ServiceDispatcher`、`publishService` 與 `onServiceConnected`） | 8.3（p.471–493） | **保留並深度重追**：從 `ContextWrapper.bindService` → `ContextImpl.bindServiceCommon`（`LoadedApk.getServiceDispatcher` 建 `ServiceDispatcher` / `InnerConnection`）→ `AMS.bindServiceInstance` → `ActiveServices.bindServiceLocked`（`ActivityServiceConnectionsHolder`、建立 `IntentBindRecord` / `AppBindRecord` / `ConnectionRecord`、`BIND_AUTO_CREATE` 命中既有進程直走 `realStartServiceLocked` → `onCreate`）→ `requestServiceBindingsLocked` / `requestServiceBindingLocked`（`scheduleBindService` + `bindToken`）→ `ActivityThread.handleBindService`（`onBind` → `publishService`）→ `ActiveServices.publishServiceLocked` → `InnerConnection.connected` → `ServiceDispatcher.connected` → `RunConnection` → `doConnected`（`DeathMonitor` + `ServiceConnection.onServiceConnected`） |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 8.1 現代 Service 管理架構、資料模型與雙情景總覽（`8.1-architecture-and-scenario.md`）
- **情景**：透過兩大代表情景——(1) 呼叫端以 `startService` / `startForegroundService` 在尚未啟動的新進程中拉起遠端 Service；(2) Activity 以 `bindService` 在同進程中綁定本地 Service 並取得其 `IBinder` 介面——建立全章觀察座標，先解剖系統端與客戶端的核心結構關係。
- **起點 → 終點**：`AMS.mServices`（`ActiveServices`）→ `mServiceMap`（per-user `ServiceMap`）→ `ServiceRecord` → `IntentBindRecord` → `AppBindRecord` → `ConnectionRecord`（四層綁定圖譜）；`ProcessRecord.mServices`（`ProcessServiceRecord`）；`ActivityServiceConnectionsHolder`；`IActivityManager` / `IApplicationThread` / `IServiceConnection` 三組 AIDL 契約。

### 8.2 Service 在新進程中的啟動過程（`8.2-start-service-new-process.md`）
- **情景**：應用程式呼叫 `startService(intent)`（並對照 `startForegroundService(intent)`）請求啟動宣告了獨立 `android:process` 且尚未運行的 Service，系統核驗權限與背景／FGS 規則、孵化新進程，最終在新進程中依序完成 `Service.onCreate()` 與 `Service.onStartCommand()`。
- **起點 → 終點**：
  - 請求發起：`ContextWrapper.startService` → `ContextImpl.startService` / `startForegroundService` → `ContextImpl.startServiceCommon`（`validateServiceIntent` + `ActivityManager.getService().startService` + 錯誤轉例外）
  - 系統端檢核與建程：`AMS.startService` → `ActiveServices.startServiceLocked`（`retrieveServiceLocked`、`setFgsRestrictionLocked`、`bgLaunch` 檢核）→ `startServiceInnerLocked`（建立 `ServiceRecord.StartItem` 加入 `r.pendingStarts`）→ `bringUpServiceLocked` / `bringUpServiceInnerLocked`（`getProcessRecordLocked == null` → `mAm.startProcessLocked` + `mPendingServices.add(r)`）
  - 新進程附著與派發：`ActivityThread.main()` → `AMS.attachApplicationLocked` → `finishAttachApplicationInner` → `ActiveServices.attachApplicationLocked`（消費 `mPendingServices`）→ `realStartServiceLocked`（`r.setProcess`、`ProcessServiceRecord.startService`、`bumpServiceExecutingLocked` 設 ANR 超時、OOM adj 批次更新、`thread.scheduleCreateService`、`requestServiceBindingsLocked`、`sendServiceArgsLocked` → `thread.scheduleServiceArgs`）
  - 新進程實例化與啟動：`ActivityThread.H`（`CREATE_SERVICE`）→ `handleCreateService`（`AppComponentFactory.instantiateService` + `ContextImpl.createAppContext` + `service.attach` + **`service.onCreate()`** + `mServices.put(token, service)` + `serviceDoneExecuting`）→ `ActivityThread.H`（`SERVICE_ARGS`）→ `handleServiceArgs`（**`service.onStartCommand(...)`** + `serviceDoneExecuting`）。

### 8.3 Service 的綁定與回調過程（`8.3-bind-service-in-process.md`）
- **情景**：已運行的 Activity 透過 `bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)` 綁定同進程內的 `CounterService`（尚未建立），系統在同進程建立 Service、執行 `onBind()` 取回 `IBinder`，經由 `publishService` 與 `IServiceConnection` 回調至客戶端 `ServiceConnection.onServiceConnected()`。
- **起點 → 終點**：
  - 客戶端準備分發器：`ContextWrapper.bindService` → `ContextImpl.bindService` → `ContextImpl.bindServiceCommon`（`validateServiceIntent` → `LoadedApk.getServiceDispatcher` 建 `ServiceDispatcher` 與 `InnerConnection` → `ActivityManager.getService().bindServiceInstance`）
  - 系統端登記綁定與建立 Service：`AMS.bindServiceInstance` → `ActiveServices.bindServiceLocked`（`mAtmInternal.getServiceConnectionsHolder(token)` → `retrieveServiceLocked` → `ServiceRecord.retrieveAppBindingLocked` 建 `IntentBindRecord`/`AppBindRecord` → `new ConnectionRecord` → 掛入 `mServiceConnections` 與 `ActivityServiceConnectionsHolder` → `BIND_AUTO_CREATE` 呼叫 `bringUpServiceLocked` 命中同進程 `app != null && thread != null` → `realStartServiceLocked` → `scheduleCreateService` → `CounterService.onCreate()`）
  - 請求綁定與發布 `IBinder`：`requestServiceBindingsLocked` → `requestServiceBindingLocked`（`b.requested = true` → `app.getThread().scheduleBindService(r, b.token, ...)`）→ `ActivityThread.H`（`BIND_SERVICE`）→ `handleBindService`（**`CounterService.onBind(intent)`** → `ActivityManager.getService().publishService(token, bindToken, binder)` → `serviceDoneExecuting`）
  - 回調至客戶端：`AMS.publishService` → `ActiveServices.publishServiceLocked`（設 `b.binder = service`、`b.received = true` → 遍歷 `AppBindRecord.connections` 呼叫 `c.conn.connected(r.name, service, session, false)`）→ `LoadedApk.ServiceDispatcher.InnerConnection.connected` → `ServiceDispatcher.connected`（透過 `mActivityExecutor` 或 `mActivityThread.post(new RunConnection(...))`）→ `RunConnection.run` → `ServiceDispatcher.doConnected`（掛 `DeathMonitor` 並呼叫 **`ServiceConnection.onServiceConnected(name, service)`**）。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| `ActivityManagerService.{startServiceLocked,bringUpServiceLocked,realStartServiceLocked,bindService,requestServiceBindingsLocked,publishService}` | `platform/frameworks/base/services/core/java/com/android/server/am/ActiveServices.java` | 搬家 + 拆分 | Service 排程邏輯全數從單體 AMS 抽離至 `ActiveServices`，AMS 僅保留 Binder 入口委派 |
| `ActivityManagerService.mServices`（單一全域 Map） | `ActiveServices.mServiceMap`（`SparseArray<ServiceMap>`，per-user）+ `ProcessRecord.mServices`（`ProcessServiceRecord`） | 邏輯改變 + 拆分 | 支援多使用者隔離，並把進程持有的 Service 集合與綁定資訊獨立至 `ProcessServiceRecord` |
| `ActivityManagerProxy` / `ActivityManagerNative`（手寫 Binder） | AIDL 生成 `IActivityManager`（`startService`、`bindServiceInstance`、`publishService`）與 `IServiceConnection`（`connected(..., IBinderSession, boolean dead)`） | 邏輯改變 | 全面 AIDL 化；`bindService` 升級為 `bindServiceInstance`（64 位元 `long flags`、支援具名實例與 SDK Sandbox） |
| `ContextImpl.startService`（允許隱式 Intent、無背景啟動限制） | `ContextImpl.startService` / `startForegroundService` → `startServiceCommon`（`validateServiceIntent` + `fgRequired` + `BackgroundServiceStartNotAllowedException` / `ForegroundServiceStartNotAllowedException`） | 邏輯改變 + 新增 | 強制顯式 Intent，新增前台服務啟動管道與嚴格背景啟動／FGS 型別超時管制 |
| `HistoryRecord.connections`（AMS 直接存取 Activity 物件） | `mAm.mAtmInternal.getServiceConnectionsHolder(token)` → `ActivityServiceConnectionsHolder<ConnectionRecord>` | 邏輯改變 + 拆分 | 因應 Activity 遷至 ATMS/WM，透過 `ActivityTaskManagerInternal` 橋接解綁生命週期，避免 AM/WM 鎖死鎖 |
| `IApplicationThread.scheduleBindService(token, intent, rebind)` 與 `publishService(token, intent, service)` | `scheduleBindService(token, bindToken, intent, rebind, processState, bindSeq)` 與 `publishService(token, bindToken, service)` | 邏輯改變 | 新增 `IBinder bindToken` 與 `bindSeq` 精準綁定 `IntentBindRecord`，避免以 `Intent.filterEquals` 搜尋失配 |
| `IApplicationThread.scheduleServiceArgs(token, startId, flags, args)`（單筆派發） | `scheduleServiceArgs(IBinder token, in ParceledListSlice args)`（批次派發） | 邏輯改變 | 透過 `ParceledListSlice` 單次 IPC 合併派送多筆 `ServiceStartArgs` |
| `LoadedApk.ServiceDispatcher`（僅支援 `Handler` 派發） | `LoadedApk.ServiceDispatcher`（同時支援 `Handler` 與 `Executor`、`DeathMonitor` 與 `IBinderSession`） | 擴充 | 支援 `Context.bindService(intent, flags, executor, conn)` 在自訂執行緒池接收 `onServiceConnected` |
| `ActivityManagerService.attachApplicationLocked` 直接呼叫 `realStartServiceLocked` | `AMS.finishAttachApplicationInner` → `mServices.attachApplicationLocked(app, processName)` → `ActiveServices.realStartServiceLocked` | 改名 + 邏輯改變 | 納入兩階段進程附著流程，並以 `SyncBatchSession` 批次更新 OOM adj 後再發送 `scheduleCreateService` |
| —（無已知未確認項目） | — | — | 本章所有類別、路徑與方法簽名皆已於 `platform/frameworks/base`（`android-17.0.0_r1`，`94b4c163b`）實機驗證，0 筆 `TODO: 未確認` |
