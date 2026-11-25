# 第 7 章：Activity 組件的啟動過程 — 大綱

> **原書對應**：第 7 章〈Activity 组件的启动过程〉（p.400–450）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：本章為使用者空間應用框架與 `system_server`（ATMS / WMS / AMS），不涉及 Linux 核心驅動（底層 Binder IPC 詳見第 5 章，Zygote fork 細節留待第 11–12 章）
> **撰寫日期**：2026-11-25
> **狀態**：✅ 全章完成（`7.1`–`7.4` 共 4 節、2,055 行、29 個 Step、66 段程式碼、1,220 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、5 張 PlantUML 圖表皆已編譯並目視檢查通過）

---

## 一、原始碼專案清單與版本驗證

本章所有呼叫鏈皆位於單一核心專案 `platform/frameworks/base`，已於本機 `src/` 檢出並驗證對應版本：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 7 章涵蓋範圍 |
|---|---|---|---|
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163b` | **App / Client 端**：`core/java/android/app/{Activity.java,Instrumentation.java,ActivityTaskManager.java,IActivityTaskManager.aidl,ActivityClient.java,IActivityClientController.aidl,IApplicationThread.aidl,ActivityThread.java,ClientTransactionHandler.java}`、`core/java/android/app/servertransaction/{ClientTransaction.java,ClientTransactionItem.java,ActivityLifecycleItem.java,PauseActivityItem.java,LaunchActivityItem.java,ResumeActivityItem.java,TransactionExecutor.java,TransactionExecutorHelper.java}`<br>**`system_server` / WM & AM 端**：`services/core/java/com/android/server/wm/{ActivityTaskManagerService.java,ActivityStartController.java,ActivityStarter.java,ActivityTaskSupervisor.java,RootWindowContainer.java,Task.java,TaskFragment.java,ActivityRecord.java,ActivityClientController.java,ClientLifecycleManager.java,WindowProcessController.java}`、`services/core/java/com/android/server/am/{ActivityManagerService.java,ProcessList.java}` |

> **重大架構演進說明（2.3 → 17）**：
> 1. **職責拆離與視窗階層統一（AMS → ATMS + `wm/`）**：原書 2.3 中，所有 Activity 排程與堆疊管理都擠在 `com.android.server.am` 的 `ActivityManagerService` 與單一扁平陣列 `ActivityStack.mHistory`（存放 `HistoryRecord` 與 `TaskRecord`）裡。Android 10 起已徹底重構並整合至 **`com.android.server.wm`**——由 **`ActivityTaskManagerService`（ATMS）** 統管 Activity 與任務，`ActivityRecord`、`TaskFragment`、`Task`、`TaskDisplayArea`、`DisplayContent`、`RootWindowContainer` 全部納入統一的 **`WindowContainer` 樹狀視窗階層**，而 `ActivityManagerService`（AMS）退回專司進程、Service、Broadcast、Provider 與 OOM 調整。
> 2. **AIDL 化與三組 Binder 介面分離**：原書手寫的 `ActivityManagerProxy` / `ActivityManagerNative`（單一 `IActivityManager`）已全面改由 AIDL 編譯生成，並按職責切分為三個獨立介面：
>    - **`IActivityTaskManager`**（`ActivityTaskManager.getService()`）：發起 `startActivity` 等任務與啟動請求；
>    - **`IActivityClientController`**（`ActivityClient.getInstance()`，首次啟動時還會隨 `LaunchActivityItem` 夾帶下發省一次查詢）：承接 Activity 端回報 `activityPaused` / `activityStopped` 等生命週期完成事件；
>    - **`IActivityManager`**（`ActivityManager.getService()`）：新進程啟動後呼叫 `attachApplication` 註冊自身。
> 3. **統一事務管線取代散裝 IPC（`ClientLifecycleManager` + `ClientTransaction` + `TransactionExecutor`）**：原書 2.3 的 `IApplicationThread` 為每個狀態單獨定義 `schedulePauseActivity` / `scheduleLaunchActivity` / `scheduleResumeActivity` 等零散介面；Android 17 統一收斂為單一 `IApplicationThread.scheduleTransaction(ClientTransaction)`，由 **`ClientLifecycleManager`** 依行程聚合批次下發有序 `List<ClientTransactionItem>`（如 `PauseActivityItem`、或一次打包 `LaunchActivityItem` + `ResumeActivityItem`），客戶端由 **`TransactionExecutor`** 透過 `TransactionExecutorHelper` 自動補齊中間生命週期（`ON_CREATE → ON_START → ON_RESUME`）。
> 4. **異步進程啟動與「暫停舊 Activity + 預建新進程」並行化**：原書 2.3 必須等 Launcher 完整走完 Pause 回報 `activityPaused` 後，才在 `startSpecificActivityLocked` 同步呼叫 `AMS.startProcessLocked`；Android 17 在 `TaskFragment.resumeTopActivity` 剛排程完 `startPausing` 且發現 `!next.isProcessRunning()` 時，即透過 **`ATMS.startProcessAsync()`**（記錄至 `mStartingProcessActivities` 並以 `PooledLambda` 投遞至 `mH`，避免持 ATMS 鎖死鎖 AMS）**並行預先 fork 目標進程**，顯著縮短冷啟動延遲。

---

## 二、小節規劃與原書對照

原書第 7 章共 4 節，其中 7.2〈根 Activity 組件的啟動過程〉長達 34 頁、35 個 Step；依 `PLAN.md` 篇幅控管原則，將原書 7.2 依「暫停前／建程後」自然拆為 **`7.2`** 與 **`7.3`** 兩節，並將原書 7.3（同進程子 Activity）與 7.4（僅 2 頁的新進程子 Activity）整合為 **`7.4`** 一節形成完整對比，全章共 **4 個小節**（圖表皆以 PlantUML 繪製，存於 `ch07-activity/figs/`）：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `7.1-architecture-and-scenario.md` | 7.1 現代 Activity 啟動架構與三情景總覽（ATMS、`WindowContainer` 樹與 `ClientTransaction`） | 7.1（p.400–406） | **改寫為架構與情景定錨節**：釐清 2.3 → 17 的視窗樹結構（`RootWindowContainer` → `DisplayContent` → `TaskDisplayArea` → `Task` → `TaskFragment` → `ActivityRecord`）、三大 AIDL 介面分工與 `ClientLifecycleManager` 事務管線；保留原書「根 Activity 冷啟動 / 同進程子 Activity / 新進程子 Activity」三情景設計，並對照現代 `dumpsys activity activities` 與 `AndroidManifest.xml` 屬性判定規則 |
| `7.2-root-activity-request-and-pause.md` | 7.2 根 Activity 啟動（上）：啟動請求、`ActivityStarter` 建構任務與並行暫停／預建進程 | 7.2 Step 1–21（p.406–427） | **拆分並全面重追**：追蹤 Launcher 呼叫 `Activity.startActivity` → `Instrumentation.execStartActivity` → `ATMS.startActivity` → `ActivityStartController` / `ActivityStarter`（`execute` → `executeRequest` → `startActivityUnchecked` → `startActivityInner` → `setNewTask`）→ `RootWindowContainer.resumeFocusedTasksTopActivities` → `TaskFragment.resumeTopActivity` → `startPausing`（下發 `PauseActivityItem`）與**並行呼叫 `ATMS.startProcessAsync` 預建進程** → Launcher 執行 `PauseActivityItem` 並透過 `IActivityClientController.activityPaused` 觸發 `TaskFragment.completePause` |
| `7.3-root-activity-process-and-launch.md` | 7.3 根 Activity 啟動（下）：異步建程、`attachApplication` 與 `TransactionExecutor` 實例化 | 7.2 Step 22–35（p.427–440） | **拆分並全面重追**：追蹤 `ATMS.startProcessAsync` → `AMS.LocalService.startProcess` → `ProcessList.startProcessLocked`（界定與第 12 章的分工介面）→ 新進程 `ActivityThread.main()` → `AMS.attachApplicationLocked`（`bindApplication` + `mAtmInternal.attachApplication`）→ `RootWindowContainer.attachApplication`（消費 `mStartingProcessActivities`）→ `ActivityTaskSupervisor.realStartActivityLocked` / `tryRealStartActivityInner`（打包 `LaunchActivityItem` + `ResumeActivityItem` 立即下發）→ `TransactionExecutor.execute` 驅動 `handleLaunchActivity` / `performLaunchActivity`（`onCreate`）→ `cycleToPath`（`onStart`）→ `handleResumeActivity`（`onResume`） |
| `7.4-sub-activity-in-and-across-process.md` | 7.4 子 Activity 啟動：同進程復用與跨進程同任務（`Task` 與進程解耦模型） | 7.3 + 7.4（p.440–450） | **合併對照重追**：對比兩條子 Activity 啟動分支與 7.2–7.3 的核心差異點——(A) 同進程子 Activity：`ActivityStarter.startActivityInner` 不建新 `Task` 而呼叫 `addOrReparentStartingActivity(targetTask, ...)` 壓入既有任務，暫停完成後 `startSpecificActivity` 命中 `wpc != null && wpc.hasThread()` 直接走熱路徑 `realStartActivityLocked`；(B) 跨進程子 Activity：同樣復用 `sourceRecord` 所在 `Task`，卻因 `android:process` 不同導致 `getProcessController` 未命中而走 `startProcessAsync` + `attachApplication`，最終歸納 `Task` / `TaskFragment` / `WindowProcessController` 三層正交解耦設計 |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 7.1 現代 Activity 啟動架構與三情景總覽（`7.1-architecture-and-scenario.md`）
- **情景**：透過三種代表性啟動組合——(1) Launcher 帶 `FLAG_ACTIVITY_NEW_TASK` 冷啟動應用根 Activity（新 Task + 新進程）；(2) 根 Activity 啟動同進程子 Activity（同 Task + 同進程）；(3) 根 Activity 啟動獨立 `android:process` 子 Activity（同 Task + 新進程）——建立貫穿全章的觀察座標，並先拆解 Android 17 `wm/` 視窗容器階層與 `ClientTransaction` 基礎元件。
- **起點 → 終點**：`RootWindowContainer` → `DisplayContent` → `TaskDisplayArea` → `Task` → `TaskFragment` → `ActivityRecord` 樹狀關聯；`IActivityTaskManager` / `IActivityClientController` / `IActivityManager` / `IApplicationThread` 四組 Binder 邊界；`ClientLifecycleManager` → `ClientTransaction` → `ClientTransactionItem` 基礎結構。

### 7.2 根 Activity 啟動（上）：啟動請求、`ActivityStarter` 建構任務與並行暫停／預建進程（`7.2-root-activity-request-and-pause.md`）
- **情景**：使用者在 Launcher 點擊應用圖示，Launcher 以帶 `FLAG_ACTIVITY_NEW_TASK` 的 `Intent` 請求啟動目標應用的根 `MainActivity`（目標進程尚未運行），系統端建出 `ActivityRecord` 與新 `Task`，排程暫停 Launcher，並同時預先觸發異步建程，直到 Launcher 完成 `onPause()` 回報系統端。
- **起點 → 終點**：
  - 請求發起：`Activity.startActivity` → `Activity.startActivityForResult` → `Instrumentation.execStartActivity` → `ActivityTaskManager.getService().startActivity`（Binder IPC）
  - 系統端排程：`ATMS.startActivity` → `ATMS.startActivityAsUser` → `ActivityStartController.obtainStarter` → `ActivityStarter.execute` → `executeRequest`（解析 `ActivityInfo`、檢查權限與 BAL、建 `ActivityRecord`）→ `startActivityUnchecked` → `startActivityInner`（`computeLaunchingTaskFlags` → `getOrCreateRootTask` → `setNewTask` → `moveToFront`）→ `RootWindowContainer.resumeFocusedTasksTopActivities` → `Task.resumeTopActivityUncheckedLocked` → `TaskFragment.resumeTopActivity`
  - 暫停與並行預建進程：`TaskFragment.startPausing` → `schedulePauseActivity` → `ClientLifecycleManager.scheduleTransactionItem(PauseActivityItem)`；同時因 `!next.isProcessRunning()` 觸發 **`ATMS.startProcessAsync(next, ..., HOSTING_TYPE_NEXT_TOP_ACTIVITY)`**
  - Launcher 端暫停與回報：`ApplicationThread.scheduleTransaction` → `ClientTransactionHandler.scheduleTransaction` → `ActivityThread.H.EXECUTE_TRANSACTION` → `TransactionExecutor.execute` → `PauseActivityItem.execute`（`ActivityThread.handlePauseActivity` → `performPauseActivity` → `onPause`）→ `PauseActivityItem.postExecute` → `ActivityClient.getInstance().activityPaused` → `ActivityClientController.activityPaused` → `ActivityRecord.activityPaused` → `TaskFragment.completePause` → `RootWindowContainer.resumeFocusedTasksTopActivities`。

### 7.3 根 Activity 啟動（下）：異步建程、`attachApplication` 與 `TransactionExecutor` 實例化（`7.3-root-activity-process-and-launch.md`）
- **情景**：接續 7.2 節，系統端 `mH` 處理異步建程訊息，由 AMS / `ProcessList` 經 Zygote 建立目標應用進程；新進程進入 `ActivityThread.main()` 向 AMS 註冊 `attachApplication`，ATMS 取出待啟動的 `ActivityRecord` 並打包 `LaunchActivityItem` + `ResumeActivityItem` 立即送往新進程，最終在新進程主執行緒完成 `MainActivity` 實例化、`onCreate()`、`onStart()` 與 `onResume()`。
- **起點 → 終點**：
  - 異步建程：`ATMS.startProcessAsync`（寫入 `mStartingProcessActivities` + `PooledLambda` 投遞 `mH`）→ `AMS.LocalService.startProcess` → `AMS.startProcessLocked` → `ProcessList.startProcessLocked`（Zygote fork 交界點）
  - 新進程註冊：`ActivityThread.main()` → `ActivityThread.attach(false, startSeq)` → `ActivityManager.getService().attachApplication(mAppThread, startSeq)` → `AMS.attachApplicationLocked`（建立 `ProcessRecord`/`WindowProcessController` → `thread.bindApplication(...)` → `mAtmInternal.attachApplication(wpc)`）→ `ATMS.LocalService.attachApplication` → `RootWindowContainer.attachApplication`（從 `mStartingProcessActivities` 匹配取出 `r`）
  - 下發啟動事務：`ActivityTaskSupervisor.realStartActivityLocked`（檢查 `allPausedActivitiesComplete()`）→ `tryRealStartActivityInner`（建 `LaunchActivityItem` + `ResumeActivityItem`，首次夾帶 `IActivityClientController`）→ `ClientLifecycleManager.scheduleTransactionItems(..., shouldDispatchImmediately=true)` → `ClientTransaction.schedule()` → `IApplicationThread.scheduleTransaction`
  - 客戶端實例化與生命週期推進：`ActivityThread.H.EXECUTE_TRANSACTION` → `TransactionExecutor.execute` → `executeNonLifecycleItem(LaunchActivityItem)` → `ActivityThread.handleLaunchActivity` → `performLaunchActivity`（`LoadedApk` + `Instrumentation.newActivity` + `ContextImpl.createActivityContext` + `activity.attach` + `Instrumentation.callActivityOnCreate` → `MainActivity.onCreate`）→ `executeLifecycleItem(ResumeActivityItem)` → `cycleToPath(ON_RESUME, excludeLastState=true)`（自動補齊 `performStart` → `onStart`）→ `ResumeActivityItem.execute` → `ActivityThread.handleResumeActivity`（`performResumeActivity` → `onResume` + `wm.addView`）。

### 7.4 子 Activity 啟動：同進程復用與跨進程同任務（`7.4-sub-activity-in-and-across-process.md`）
- **情景**：
  - 情景 A（同進程）：已運行的 `MainActivity` 呼叫 `startActivity(intent)`（無 `FLAG_ACTIVITY_NEW_TASK`）啟動同進程的 `SubActivityInProcess`；
  - 情景 B（跨進程同任務）：`MainActivity` 呼叫 `startActivity(intent)`（無 `FLAG_ACTIVITY_NEW_TASK`）啟動宣告了獨立 `android:process` 的 `SubActivityInNewProcess`。
- **起點 → 終點**：
  - 任務復用共同分流：`ActivityStarter.startActivityInner` → `computeLaunchingTaskFlags`（`mInTask == null && mSourceRecord != null` 保留於來源任務）→ `newTask == false`、`mAddingToTask == true` → `addOrReparentStartingActivity(targetTask, "adding to task")` → 將新 `ActivityRecord` 加入同一 `Task` / `TaskFragment` 頂部；
  - 進程判定分流：`MainActivity` 完成 `PauseActivityItem` → `TaskFragment.completePause` → `resumeTopActivity` → `ActivityTaskSupervisor.startSpecificActivity` 查詢 `mService.getProcessController(r.processName, r.getUid())`：
    - 情景 A 命中 `wpc != null && wpc.hasThread()` → **直接進入 `realStartActivityLocked`**（熱路徑，零 Zygote fork）；
    - 情景 B 未命中（不同 `processName`）→ 進入 `mService.startProcessAsync`（實際早在 `resumeTopActivity` 暫停 `MainActivity` 時已並行預啟動）→ 新進程 `attachApplication` → `realStartActivityLocked`，呈現「同一 `Task` 跨兩個 `WindowProcessController`」結構。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| `ActivityManagerService.startActivity`（`server/am/ActivityManagerService.java`） | `ActivityTaskManagerService.startActivity` / `startActivityAsUser`（`server/wm/ActivityTaskManagerService.java`） | 改名搬家 + 拆分 | Activity 與 Task 管理由 AMS 獨立抽離至 ATMS，並併入 `com.android.server.wm` |
| `ActivityManagerProxy` / `ActivityManagerNative`（手寫 Binder 代理） | AIDL 生成 `IActivityTaskManager`、`IActivityClientController`、`IActivityManager` | 邏輯改變 + 拆分 | 改由 AIDL 編譯器生成，並將「啟動請求」、「客戶端回報（`activityPaused`）」與「進程註冊（`attachApplication`）」拆分至三組介面 |
| `ActivityStack`（單一線性 `ArrayList mHistory`） | `RootWindowContainer` + `TaskDisplayArea` + `Task` + `TaskFragment` + `ActivityStarter` + `ActivityTaskSupervisor` | 邏輯改變 + 拆分 | 舊 `ActivityStack` 已完全移除，拆分職責至 `ActivityStarter`（解析與任務決策）、`ActivityTaskSupervisor`（跨任務協調與進程啟動）與 `Task`/`TaskFragment`（視窗階層樹與狀態機） |
| `HistoryRecord` / `TaskRecord`（`server/am/`） | `ActivityRecord` / `Task`（繼承 `WindowContainer`，`server/wm/`） | 改名搬家 + 邏輯改變 | 成為視窗樹正式節點，統一管理生命週期、Z-order、轉場動畫（Shell Transitions）與多視窗配置 |
| `ProcessRecord` 直接同時管 AM 與 Activity | `ProcessRecord`（AM 端）+ `WindowProcessController`（WM 端鏡像控制器） | 拆分 | ATMS/WM 透過 `WindowProcessController` 操作進程執行緒與組態，避免直接搶佔 AMS 全域鎖 |
| `ActivityStack.startActivityMayWait` → `startActivityLocked` → `startActivityUncheckedLocked` | `ActivityStartController.obtainStarter` → `ActivityStarter.execute` → `executeRequest` → `startActivityUnchecked` → `startActivityInner` | 改名 + 邏輯改變 | 導入物件池（`Starter` 復用）、BAL（背景啟動限制）、SDK Sandbox 權限檢查與 `setNewTask` / `addOrReparentStartingActivity` 清晰分流 |
| `ActivityStack.startSpecificActivityLocked` 同步呼叫 `AMS.startProcessLocked` | `ActivityTaskSupervisor.startSpecificActivity` / `TaskFragment.resumeTopActivity` 呼叫 **`ATMS.startProcessAsync`**（投遞 `mH` + `mStartingProcessActivities`） | 邏輯改變 | 改為異步建程以避免 ATMS/AMS 鎖序死鎖，且在暫停前一個 Activity 的**同時就並行預先 fork 目標進程** |
| `ApplicationThreadProxy.schedulePauseActivity` / `scheduleLaunchActivity`（各自獨立 IPC） | `ClientLifecycleManager` → `ClientTransaction`（統一 `List<ClientTransactionItem>`：`PauseActivityItem`、`LaunchActivityItem`、`ResumeActivityItem`）→ `IApplicationThread.scheduleTransaction` | 邏輯改變 | 單一事務通道取代所有散裝生命週期 RPC，支援同一幀內批次合併（`mPendingTransactions`）與冷啟動立即下發（`shouldDispatchImmediately=true`） |
| `ActivityThread.H` 處理 `PAUSE_ACTIVITY` / `LAUNCH_ACTIVITY` 等獨立訊息 | `ActivityThread.H.EXECUTE_TRANSACTION`（`159`）→ `TransactionExecutor.execute` → `TransactionExecutorHelper` 狀態機推進 | 邏輯改變 | 由統一事務執行器自動計算並補齊中間生命週期（如 `LaunchActivityItem` → `ON_START` → `ResumeActivityItem`） |
| `ActivityManagerProxy.activityPaused` → `AMS.activityPaused` | `PauseActivityItem.postExecute` → `ActivityClient.getInstance().activityPaused` → `ActivityClientController.activityPaused` → `ActivityRecord.activityPaused` → `TaskFragment.completePause` | 改名搬家 + 邏輯改變 | 暫停完成回報改由 `PauseActivityItem.postExecute` 觸發並走獨立 `IActivityClientController` 介面 |
| —（無已知未確認項目） | — | — | 本章所有類別、路徑與方法簽名皆已於 `platform/frameworks/base`（`android-17.0.0_r1`，`94b4c163b`）實機驗證，0 筆 `TODO: 未確認` |
