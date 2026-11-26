# 第 9 章：Android 系統廣播機制 — 大綱

> **原書對應**：第 9 章〈Android系统广播机制〉（p.494–531）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：本章為使用者空間應用框架與 `system_server`（AMS / `BroadcastController` / `BroadcastQueueImpl`），不涉及 Linux 核心驅動（底層 Binder IPC 詳見第 5 章，Zygote 建程細節留待第 12 章）
> **撰寫日期**：2026-11-26
> **狀態**：✅ 全章完成（`9.1`–`9.4` 共 2,792 行、31 個 Step、73 段程式碼、1,978 行逐行比對 0 筆不符、0 筆 `TODO: 未確認`、32 處 `〔與 2.3 差異〕`、5 張 PlantUML 圖表全數編譯並目視檢查通過）

---

## 一、原始碼專案清單與版本驗證

本章所有呼叫鏈皆位於 `platform/frameworks/base`，已於本機 `src/` 檢出並驗證對應版本：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 9 章涵蓋範圍 |
|---|---|---|---|
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163b` | **App / Client 端**：`core/java/android/content/{ContextWrapper.java,BroadcastReceiver.java,IIntentReceiver.aidl}`、`core/java/android/app/{ContextImpl.java,LoadedApk.java,ActivityThread.java,BroadcastOptions.java,BroadcastStickyCache.java,IActivityManager.aidl,IApplicationThread.aidl,ReceiverInfo.aidl}`<br>**`system_server` / AM 端**：`services/core/java/com/android/server/am/{ActivityManagerService.java,BroadcastController.java,BroadcastQueue.java,BroadcastQueueImpl.java,BroadcastProcessQueue.java,BroadcastRecord.java,BroadcastFilter.java,ReceiverList.java,ProcessReceiverRecord.java,BroadcastSkipPolicy.java,BroadcastConstants.java,BroadcastHistory.java}`（附設計文件 `BroadcastQueue.md`） |

> **重大架構演進說明（2.3 → 17）**：
> 1. **從單體 AMS 抽離為 `BroadcastController` + 單一 `BroadcastQueueImpl`（原 Modern Broadcast Queue）**：原書 2.3 將動態註冊表（`mRegisteredReceivers`、`mReceiverResolver`）、黏性廣播（`mStickyBroadcasts`）與排程佇列（`mParallelBroadcasts`、`mOrderedBroadcasts`）全部塞在 `ActivityManagerService.java` 內；Android 17 則已徹底拆分——註冊、解析、黏性快取與 `finishReceiver` 入口全數歸屬 **`BroadcastController`**，而排程派發由單一 **`BroadcastQueueImpl`** 承擔（舊版全域 FIFO 佇列與 fg/bg 雙佇列皆已移除，Android 14–15 的 `BroadcastQueueModernImpl` 在 17 已正式更名為唯一的 `BroadcastQueueImpl`）。
> 2. **從「全域無序／有序雙佇列」革命為「Per-Process 佇列（`BroadcastProcessQueue`）+ 並發槽位調度」**：Android 17 以 `mProcessQueues`（`SparseArray<BroadcastProcessQueue>`，同 UID 多進程以 `processNameNext` 串接）為每個潛在進程維護獨立佇列（內含 `mPendingUrgent`、`mPending`、`mPendingOffload` 三條子佇列）與動態計算的 `runnableAt` 時間戳；`mRunnableHead` 按 `runnableAt` 排序、`mRunning[]` 最多同時推進 `MAX_RUNNING_PROCESS_QUEUES` 個溫進程（加 `EXTRA_RUNNING_URGENT_PROCESS_QUEUES` 緊急保留槽），且全域**同時只允許 1 個 `mRunningColdStart` 冷啟動**，避免廣播風暴拖垮系統。
> 3. **單一 `BroadcastRecord` 追蹤全體接收者狀態與有序／優先級依賴**：每筆 `BroadcastRecord` 內建 `delivery[]` 狀態機（`DELIVERY_PENDING` / `SCHEDULED` / `DELIVERED` / `SKIPPED` / `TIMEOUT` / `FAILURE` / `DEFERRED`）與 **`blockedUntilBeyondCount[]`** 高水位陣列——有序廣播（`ordered`）或優先級分層廣播（`prioritized`）不再靠全域序列化阻塞所有進程，而是讓個別 `BroadcastProcessQueue` 檢查 `beyondCount >= blockedUntilBeyondCount[index]` 決定何時變成 `runnable`。無序動態接收者（`isAssumedDelivered == true`）在 `scheduleRegisteredReceiver` 送出後直接標記 `DELIVERY_DELIVERED`，不等待回報 `finishReceiver`。
> 4. **安全強化與現代管制**：動態註冊強制顯式標示 **`Context.RECEIVER_EXPORTED`** 或 **`Context.RECEIVER_NOT_EXPORTED`**（`DYNAMIC_RECEIVER_EXPLICIT_EXPORT_REQUIRED`，僅註冊受保護系統廣播或 `receiver == null` 讀黏性廣播除外）；進程動態接收者上限 **`MAX_RECEIVERS_ALLOWED_PER_APP`** 與獨立進程側記錄 **`ProcessReceiverRecord`**（`app.mReceivers`）；派發前經由 **`BroadcastSkipPolicy`**（涵蓋背景隱式靜態廣播限制、權限、AppOps、停止狀態與 `SaferIntentUtils.filterNonExportedComponents`）過濾；並整合 **CachedAppOptimizer 凍結／臨時解凍（`unfreezeTemporarily`）**、**Delivery Group 合併／去重**、**`BroadcastStickyCache`** 與 **`AnrTimer`** 雙階段 ANR 逾時管理。

---

## 二、小節規劃與原書對照

原書第 9 章分為 3 節（9.1 實例、9.2 動態註冊、9.3 廣播發送），其中 9.3 同時涵蓋無序動態派發並穿插說明有序與靜態接收者。由於 Android 17 的 `BroadcastQueueImpl` 在「無序溫進程派發」與「有序依賴／靜態 Manifest 冷啟動／`finishReceiver` + `AnrTimer` 閉環」之間有極為清晰的雙軌機制，本章規劃為 **4 節（9.1–9.4）**，兼顧原書主線與現代核心機制（圖表皆以 PlantUML 繪製，存於 `ch09-broadcast/figs/`）：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `9.1-architecture-and-scenario.md` | 9.1 現代廣播系統架構、資料模型與核心情景（`BroadcastController`、`BroadcastQueueImpl` 與 Per-Process 佇列） | 9.1（p.494–501） | **改寫為架構與情景定錨節**：延續原書 `Broadcounter` 實例（動態無序廣播），並補齊有序廣播與靜態 Manifest 接收者冷啟動對照情景；完整拆解 `BroadcastController`、`BroadcastQueueImpl`、`BroadcastProcessQueue`、`BroadcastRecord`、`ReceiverList` / `BroadcastFilter` / `ProcessReceiverRecord` 與三組 AIDL 介面（`IActivityManager`、`IApplicationThread`、`IIntentReceiver`） |
| `9.2-register-receiver.md` | 9.2 動態廣播接收者的註冊與註銷過程（`registerReceiver`、`ReceiverDispatcher` 與 `BroadcastController`） | 9.2（p.501–509） | **保留並深度重追**：從 `ContextWrapper.registerReceiver` → `ContextImpl.registerReceiverInternal`（`LoadedApk.getReceiverDispatcher` 建 `ReceiverDispatcher` / `InnerReceiver`；另說明 `BroadcastStickyCache` 快路徑）→ `AMS.registerReceiverWithFeature` → `BroadcastController.registerReceiverWithFeatureTraced`（`RECEIVER_EXPORTED`/`NOT_EXPORTED` 強制檢查、`mStickyBroadcasts` 匹配、`mRegisteredReceivers` / `MAX_RECEIVERS_ALLOWED_PER_APP` / `app.mReceivers`、`BroadcastFilter` 與 `mReceiverResolver.addFilter`、初始黏性廣播入隊），並簡述 `unregisterReceiver` 清理閉環 |
| `9.3-send-and-dispatch-broadcast.md` | 9.3 廣播的發送、排程與動態接收者派發過程（`sendBroadcast`、`BroadcastQueueImpl` 與 `ReceiverDispatcher`） | 9.3（p.509–531，動態無序主線） | **保留並深度重追**：沿 `CounterService.sendBroadcast` → `ContextImpl.sendBroadcast` → `AMS.broadcastIntentWithFeature` → `BroadcastController.broadcastIntentLockedTraced`（收集與合併靜／動態接收者、`SaferIntentUtils`、建 `BroadcastRecord`）→ `BroadcastQueueImpl.enqueueBroadcastLocked`（Delivery Group、拆散入各 `BroadcastProcessQueue`、`updateRunnableList`、`enqueueUpdateRunningList`）→ `MSG_UPDATE_RUNNING_LIST` → `updateRunningListLocked` → `promoteToRunningLocked` → `scheduleReceiverWarmLocked` → `dispatchReceivers`（`BroadcastSkipPolicy`、`assumeDelivered`、`thread.scheduleRegisteredReceiver`）→ `ActivityThread` → `InnerReceiver.performReceive` → `ReceiverDispatcher.performReceive` → `Args.getRunnable()` → `BroadcastReceiver.onReceive` |
| `9.4-ordered-manifest-and-finish.md` | 9.4 有序廣播、靜態 Manifest 冷啟動與 `finishReceiver` 閉環（`blockedUntilBeyondCount`、`scheduleReceiverColdLocked` 與 `AnrTimer`） | 9.3 延伸（p.511–526 提及之有序與靜態接收者分支） | **拆分獨立小節**：聚焦現代 Per-Process 佇列最關鍵的三大進階路徑——(1) 有序／優先級廣播如何透過 `blockedUntilBeyondCount[]` 與 `beyondCount` 跨進程協調、支援 `abortBroadcast` 與 `scheduleResultTo`；(2) 靜態 Manifest 接收者冷啟動（單槽 `mRunningColdStart` → `scheduleReceiverColdLocked` → `startProcessLocked` → `onApplicationAttachedLocked` → `thread.scheduleReceiver` → `ActivityThread.handleReceiver`）；(3) `PendingResult.finish()` / `goAsync()` → `AMS.finishReceiver` → `BroadcastQueueImpl.finishReceiverLocked` / `finishReceiverActiveLocked` 與 `BroadcastAnrTimer` 逾時處置 |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 9.1 現代廣播系統架構、資料模型與核心情景（`9.1-architecture-and-scenario.md`）
- **情景**：建立全章三大觀察情景——(A) `Broadcounter` 動態註冊接收者；(B) `CounterService` 發送無序廣播至已運行的動態接收者；(C) 有序廣播與靜態 `<receiver>` 觸發目標進程冷啟動並回報 `finishReceiver`——先畫出 `system_server` 與 App 雙側的架構與資料結構全貌。
- **起點 → 終點**：`AMS`（`mBroadcastController` / `mBroadcastQueue`）→ `BroadcastController`（`mRegisteredReceivers`、`mReceiverResolver`、`mStickyBroadcasts`）→ `ReceiverList` / `BroadcastFilter` / `ProcessReceiverRecord`；`BroadcastQueueImpl`（`mProcessQueues`、`mRunnableHead`、`mRunning[]`、`mRunningColdStart`、`mAnrTimer`）→ `BroadcastProcessQueue`（`mPendingUrgent` / `mPending` / `mPendingOffload`、`mActive`、`runnableAt`）→ `BroadcastRecord`（`receivers`、`delivery[]`、`blockedUntilBeyondCount[]`、`beyondCount`）；`IActivityManager` / `IApplicationThread` / `IIntentReceiver` 三組 AIDL 介面。

### 9.2 動態廣播接收者的註冊與註銷過程（`9.2-register-receiver.md`）
- **情景**：前景 Activity 呼叫 `registerReceiver(counterActionReceiver, filter, Context.RECEIVER_NOT_EXPORTED)` 訂閱自訂廣播（同時對照僅讀黏性廣播 `receiver == null` 的 `BroadcastStickyCache` 快路徑與缺少 exported 旗標拋出 `SecurityException` 的機制）。
- **起點 → 終點**：
  - 客戶端建立分發器：`ContextWrapper.registerReceiver` → `ContextImpl.registerReceiver` → `ContextImpl.registerReceiverInternal`（`LoadedApk.getReceiverDispatcher` 建 `ReceiverDispatcher` 與 `InnerReceiver`；或 `receiver == null` 命中 `BroadcastStickyCache.getIntent`）→ `ActivityManager.getService().registerReceiverWithFeature(...)`
  - 系統端檢核與登記：`AMS.registerReceiverWithFeature` → `BroadcastController.registerReceiverWithFeature` → `registerReceiverWithFeatureTraced`（`enforceNotIsolatedCaller`、`getRecordForAppLOSP`、`handleIncomingUser`、`mStickyBroadcasts` 收集與 `isProtectedBroadcast` 判斷、`DYNAMIC_RECEIVER_EXPLICIT_EXPORT_REQUIRED` 強制檢查、`mRegisteredReceivers` 查找或建立 `ReceiverList`、檢查 `MAX_RECEIVERS_ALLOWED_PER_APP` 與 `rl.app.mReceivers.addReceiver(rl)`、建立 `BroadcastFilter` 並加入 `rl` 與 `mReceiverResolver.addFilter`、若有匹配黏性廣播則組建 `BroadcastRecord` 入隊）
  - 註銷清理（簡述）：`ContextImpl.unregisterReceiver` → `LoadedApk.forgetReceiverDispatcher` → `BroadcastController.unregisterReceiver` → `removeReceiverLocked`。

### 9.3 廣播的發送、排程與動態接收者派發過程（`9.3-send-and-dispatch-broadcast.md`）
- **情景**：`CounterService` 呼叫 `sendBroadcast(intent)` 發送無序廣播給已註冊的動態接收者 `counterActionReceiver`，完整走過解析、Per-Process 入隊、並發調度槽派發與 App 主執行緒執行 `onReceive()`。
- **起點 → 終點**：
  - 發送與接收者解析：`ContextWrapper.sendBroadcast` → `ContextImpl.sendBroadcast`（`resolveTypeIfNeeded`、`prepareToLeaveProcess`、`broadcastIntentWithFeature`）→ `AMS.broadcastIntentWithFeature` → `BroadcastController.broadcastIntentWithFeature` → `broadcastIntentLocked` → `broadcastIntentLockedTraced`（驗證保護廣播／權限、`collectReceiverComponents` 查靜態接收者、`mReceiverResolver.queryIntent` 查動態接收者、依優先級合併為單一 `receivers` 清單、`SaferIntentUtils.filterNonExportedComponents`、建 `BroadcastRecord`、`queue.enqueueBroadcastLocked(r)`）
  - Per-Process 入隊與排程：`BroadcastQueueImpl.enqueueBroadcastLocked`（可凍結發送者延遲檢查、`applyDeliveryGroupPolicy`、`mSkipPolicy.shouldSkipAtEnqueueMessage`、逐接收者 `getOrCreateProcessQueue` → `queue.enqueueOrReplaceBroadcast` → `updateRunnableList` 插入 `mRunnableHead` → `enqueueUpdateRunningList` 投遞 `MSG_UPDATE_RUNNING_LIST`）→ `mLocalCallback` → `updateRunningListLocked`（依 `runnableAt` 掃描 `mRunnableHead`、檢查 `mRunning` 並發額度與 `isPendingUrgent`、`unfreezeTemporarily`、`promoteToRunningLocked`）
  - 溫進程派發與客戶端執行：`scheduleReceiverWarmLocked` → `shouldSkipReceiver`（`BroadcastSkipPolicy.shouldSkipMessage`）→ `dispatchReceivers`（`setDeliveryState(DELIVERY_SCHEDULED)`、`app.getOnewayThread().scheduleRegisteredReceiver(...)`、因 `assumeDelivered == true` 立刻 `finishReceiverActiveLocked(DELIVERY_DELIVERED)` 並續推下一筆）→ `ActivityThread.ApplicationThread.scheduleRegisteredReceiver`（`updateProcessState`、直呼 `InnerReceiver.performReceive`）→ `ReceiverDispatcher.performReceive`（建 `Args`、`mActivityThread.post(args.getRunnable())`）→ `Args` Lambda（`prepareToEnterProcess`、`receiver.setPendingResult(this)`、**`receiver.onReceive(mContext, intent)`**、`finish()`）。

### 9.4 有序廣播、靜態 Manifest 冷啟動與 `finishReceiver` 閉環（`9.4-ordered-manifest-and-finish.md`）
- **情景**：分別追蹤 (1) `sendOrderedBroadcast` 如何跨多個進程按優先級依序傳遞結果並允許 `abortBroadcast()` 與最終 `resultTo` 回調；(2) 靜態 `<receiver>` 命中尚未啟動的冷進程時如何透過單一冷啟動槽拉起進程並實例化執行；(3) 阻塞式接收者（有序／靜態／`goAsync()`）如何透過 `finishReceiver` 解除阻塞與取消 `BroadcastAnrTimer`。
- **起點 → 終點**：
  - 有序／優先級依賴：`BroadcastRecord` 建構時計算 `calculateBlockedUntilBeyondCount` → `BroadcastProcessQueue.updateRunnableAt`（若 `beyondCount < blockedUntilBeyondCount[index]` 標記 `REASON_BLOCKED` 暫不入 `mRunnableHead`）→ 前驅完成更新 `beyondCount` 喚醒後繼進程佇列 → 全部終結時 `scheduleResultTo(r)` 回傳結果至發送者。
  - 靜態接收者冷啟動與執行：`updateRunningListLocked` 發現 `!processWarm` → 搶佔全域唯一 `mRunningColdStart` 槽 → `promoteToRunningLocked` → `scheduleReceiverColdLocked`（`mService.startProcessLocked(..., HOSTING_TYPE_BROADCAST, ...)`）→ 新進程附著 `AMS.attachApplicationLocked` → `BroadcastQueueImpl.onApplicationAttachedLocked`（釋放 `mRunningColdStart`、`notifyStartedRunning`、`scheduleReceiverWarmLocked`）→ `dispatchReceivers`（啟動 `startDeliveryTimeoutLocked` / `mAnrTimer.start`、`thread.scheduleReceiver(...)`）→ `ActivityThread.ApplicationThread.scheduleReceiver` → `H.RECEIVER` → `handleReceiver`（`makeApplicationInner`、`AppComponentFactory.instantiateReceiver`、`receiver.setPendingResult(data)`、**`receiver.onReceive(context.getReceiverRestrictedContext(), data.intent)`**、`goAsync()` 延後或立即 `data.finish()`）。
  - 完成回報與 ANR 閉環：`BroadcastReceiver.PendingResult.finish()` → `sendFinished(IActivityManager)` → `AMS.finishReceiver` → `BroadcastController.finishReceiver` → `BroadcastQueueImpl.finishReceiverLocked`（寫回 `resultCode`/`resultData`/`resultExtras`/`resultAbort`、呼叫 `finishReceiverActiveLocked` 取消 `mAnrTimer`、若中止則批次標記剩餘接收者 `DELIVERY_SKIPPED`、檢查 `shouldRetire` 決定續派或 `demoteFromRunningLocked`；若逾時則 `BroadcastAnrTimer` 觸發 `DELIVERY_TIMEOUT` → `mService.appNotResponding`）。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| `ActivityManagerService.{registerReceiver,unregisterReceiver,broadcastIntent,broadcastIntentLocked,finishReceiver}` | `com.android.server.am.BroadcastController`（AMS 僅保留 Binder 入口委派至 `mBroadcastController`） | 搬家 + 拆分 | 註冊表、解析、黏性廣播與廣播發送前處理全數自單體 AMS 抽離至 `BroadcastController` |
| `AMS.mParallelBroadcasts` + `mOrderedBroadcasts` + `scheduleBroadcastsLocked` + `processNextBroadcast` | `BroadcastQueueImpl`（單一 `mBroadcastQueue`）+ `BroadcastProcessQueue`（`mProcessQueues`、`mRunnableHead`、`mRunning[]`、`mRunningColdStart`） | 邏輯徹底重寫 | 移除全域 FIFO 雙佇列；改為 Per-Process 獨立佇列、`runnableAt` 動態調度、溫進程多槽並發與單一冷啟動限流 |
| `AMS.mRegisteredReceivers` / `mReceiverResolver` / `mStickyBroadcasts` | `BroadcastController.{mRegisteredReceivers,mReceiverResolver,mStickyBroadcasts}` + `ProcessRecord.mReceivers`（`ProcessReceiverRecord`） | 搬家 + 擴充 | 移入 `BroadcastController`；進程側接收者集合獨立為 `ProcessReceiverRecord`，並增設 `MAX_RECEIVERS_ALLOWED_PER_APP` 上限 |
| `ActivityManagerProxy` / `ActivityManagerNative`（手寫 Binder） | AIDL 生成 `IActivityManager`（`registerReceiverWithFeature`、`broadcastIntentWithFeature`、`oneway finishReceiver`）與 `IIntentReceiver` | 邏輯改變 | 全面 AIDL 化；新增 `callingFeatureId`、`receiverId`、`flags`、`excludePermissions`、`excludePackages`、`BroadcastOptions`；`finishReceiver` 為 `oneway` |
| `ContextImpl.registerReceiver`（無 exported 旗標、無 sticky client cache） | `ContextImpl.registerReceiverInternal` + `BroadcastStickyCache` + `BroadcastController`（強制 `RECEIVER_EXPORTED` / `RECEIVER_NOT_EXPORTED`） | 邏輯改變 + 新增 | 非系統受保護廣播須顯式宣告匯出屬性否則拋 `SecurityException`；`receiver == null` 讀黏性廣播支援 `BroadcastStickyCache` 快取 |
| `BroadcastRecord`（僅 `nextReceiver` 與 `state` 單一游標） | `BroadcastRecord`（`delivery[]` 逐接收者狀態機、`blockedUntilBeyondCount[]`、`beyondCount`、`prioritized`、`deferUntilActive`） | 邏輯徹底重寫 | 支援跨進程並行派發與精準有序／優先級依賴控制，無須全域佇列串行等待 |
| `deliverToRegisteredReceiverLocked` / `processNextBroadcast` 內建過濾與逾時 | `BroadcastSkipPolicy`（入隊與派發雙階段檢查）+ `SaferIntentUtils` + `BroadcastAnrTimer`（`AnrTimer`） | 拆分 + 新增 | 權限、背景限制、exported 與套件可見性過濾獨立為 `BroadcastSkipPolicy`；ANR 計時改由 `AnrTimer` 統一管理 |
| `IApplicationThread.scheduleRegisteredReceiver`（7 參數）與 `scheduleReceiver`（6 參數） | `scheduleRegisteredReceiver`（12 參數，新增 `assumeDelivered`、`sendingUser`、`processState`、`sendingUid`、`sendingPackage`）、`scheduleReceiver`（12 參數）與批次 `scheduleReceiverList(List<ReceiverInfo>)` | 邏輯改變 + 新增 | 支援 `assumeDelivered` 免回報、`shareIdentity` 傳遞發送者身分、即時同步 `processState` 與 `ReceiverInfo` 批次介面 |
| `LoadedApk.ReceiverDispatcher.Args implements Runnable`（`run()` 直接實作） | `Args extends BroadcastReceiver.PendingResult`（`getRunnable()` 回傳 Lambda、`goAsync()` 支援、`mForgotten` 防護與 `prepareToEnterProcess`） | 邏輯改變 | `PendingResult` 成為獨立非同步控制柄（`goAsync()` 延後 `finish()`），`InnerReceiver` 透過 `ApplicationThread` 短路直呼多載 `performReceive` |
| `ActivityThread.handleReceiver`（`ClassLoader.loadClass().newInstance()`） | `ActivityThread.handleReceiver`（`AppComponentFactory.instantiateReceiver`、split/attribution/display Context、`sCurrentBroadcastIntent`、`getReceiverRestrictedContext`） | 邏輯改變 | 改由 `AppComponentFactory` 實例化並補齊多螢幕／權限歸屬 Context 與 `PendingResult` 生命週期 |
| —（無已知未確認項目） | — | — | 本章所有類別、路徑與方法簽名皆已於 `platform/frameworks/base`（`android-17.0.0_r1`，`94b4c163b`）實機驗證，0 筆 `TODO: 未確認` |
