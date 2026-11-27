# 第 13 章：Android 應用程式的消息處理機制 — 大綱

> **原書對應**：第 13 章〈Android应用程序的消息处理机制〉（p.653–674）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：本章主體為使用者空間 Java 層（`Looper` / `MessageQueue` / `Handler` / `Message`）、JNI 橋接層（`NativeMessageQueue`）與 C++ 層（`libutils/Looper`）；底層透過 Linux `eventfd(2)` 與 `epoll_create1(2)` / `epoll_ctl(2)` / `epoll_wait(2)` 系統呼叫實現執行緒休眠、喚醒與多工 FD 事件監聽
> **撰寫日期**：2026-11-27
> **狀態**：✅ 全章完成（`13.1`–`13.4` 共 1,735 行、20 個 Step、30 段原始碼、1,013 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、20 處 `〔與 2.3 差異〕`、5 張 PlantUML 圖表全數編譯並目視檢查通過）

---

## 一、原始碼專案清單與版本驗證

本章所有呼叫鏈（涵蓋 Soong 編譯期 `MessageQueue` 選型、三套 `MessageQueue` 實作、`Looper`、`Handler`、`Message`、`NativeMessageQueue` JNI 與 `libutils/Looper`）皆已於本機 `src/` 檢出並驗證對應版本：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 13 章涵蓋範圍 |
|---|---|---|---|
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163b` | **編譯期選型**：`core/java/Android.bp`（`messagequeue-gen`，依 `release_package_messagequeue_implementation` 選出 `android/os/MessageQueue.java`，預設 `CombinedMessageQueue`）<br>**Java 層消息框架**：`core/java/android/os/Looper.java`（`prepare`、`prepareMainLooper`、`loop`、`loopOnce`、`Observer`、`LooperDoctor`、`showSlowLog`）、`core/java/android/os/Handler.java`（建構子、`createAsync`、`sendMessage*`、`enqueueMessage`、`dispatchMessage`、`dispatchMessageImpl`、`handleMessage`、`post`）、`core/java/android/os/Message.java`（`obtain`、`recycleUnchecked`、`FLAG_IN_USE`、`FLAG_ASYNCHRONOUS`、Concurrent 模式下停用 `sPool`）<br>**三套 `MessageQueue` 實作**：`core/java/android/os/CombinedMessageQueue/MessageQueue.java`（預設雙模式：Legacy 單鎖鏈表 + Concurrent `ConcurrentSkipListSet` / `StackNode` CAS）、`core/java/android/os/CombinedDeliMessageQueue/MessageQueue.java`（雙模式：Legacy + 無鎖 DeliQueue `MessageStack` Treiber Stack + 雙 Min-Heap + `WaitState` 64-bit CAS，附 `README.md`）、`core/java/android/os/LegacyMessageQueue/MessageQueue.java`（純傳統單鎖鏈表）<br>**JNI 橋接層**：`core/jni/android_os_MessageQueue.h`、`core/jni/android_os_MessageQueue.cpp`（`MessageQueue`、`NativeMessageQueue`、`WeakLooperCallback`、`nativeInit`/`nativeDestroy`/`nativePollOnce`/`nativeWake`/`nativeSetFileDescriptorEvents`/`nativeSetSkipEpollWaitForZeroTimeout`） |
| `platform/system/core` | `android-17.0.0_r1` | `545d248` | **C++ 層 `Looper`**：`libutils/include/utils/Looper.h`、`libutils/Looper.cpp`（`Looper::Looper`、`rebuildEpollLocked`、`pollOnce`、`pollInner`、`wake`、`awoken`、`addFd`、`setSkipEpollWaitForZeroTimeout`、`mMessageEnvelopes`、`mRequestsGeneration`） |

> **重大架構演進說明（2.3 → 17）**：
> 1. **`MessageQueue` 全面進入「雙模式併發／無鎖」時代（`Android.bp` 三實作並存 + `USE_NEW_MESSAGEQUEUE`）**：原書 2.3 的 `MessageQueue.java` 只有單一 `synchronized (this)` 保護的有序單向鏈表，容易因低優先權執行緒持鎖入隊導致高優先權 UI 主執行緒發生優先權反轉與掉幀。Android 17 在 `core/java/Android.bp` 透過 `messagequeue-gen` 提供三套實作——預設為 **`CombinedMessageQueue`**（執行期依 `USE_NEW_MESSAGEQUEUE`〔API > 36 即 Android 17 啟用〕、`Flags.useConcurrentMessageQueueInApps()` 或核心進程 `UserHandle.isCore()` 切換至雙 `ConcurrentSkipListSet` + 無鎖 `StackNode` 狀態機，舊 App 則維持 Legacy 模式以相容反射存取 `mMessages` 欄位），並額外提供下一代無鎖 **`CombinedDeliMessageQueue`**（Treiber Stack + 雙 Min-Heap + 64-bit `mWaitState` CAS，與第 12 章 12.3 節 `--use-deliqueue=` 銜接）以及純 **`LegacyMessageQueue`**。同時在 Legacy 模式下也加入了 `mLast` 尾指針與 `mAsyncMessageCount` 加速常見的 O(1) 尾插。
> 2. **Sync Barrier（同步屏障）與 `msg.target == null` 語意反轉**：原書 2.3 沒有同步屏障概念，並以 `msg.target == null` 作為「退出消息」讓 `Looper.loop()` 返回；Android 17 中，退出改由 `MessageQueue.next()` 直接回傳 **`null`**，而 **`msg.target == null`** 徹底轉為 **Sync Barrier（`postSyncBarrier()` / `removeSyncBarrier()`）**——遇到屏障時跳過所有普通同步消息，優先派發帶有 `FLAG_ASYNCHRONOUS` 的非同步消息（例如 `Choreographer` VSync 繪製回呼與 `Handler.createAsync()`）。
> 3. **Native `Looper` 由 `pipe()` 改為 `eventfd()`、序號防 Stale Response 與零超時跳過 `epoll_wait`**：`Looper.cpp` 由 `frameworks/base/libs/utils/` 搬移至 `platform/system/core/libutils/`；喚醒機制從占用兩個 FD 的 `pipe()`（`mWakeReadPipeFd` / `mWakeWritePipeFd` 寫入 `'W'`）改為單一 `eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC)`（`mWakeEventFd` 寫／讀 8 位元組 `uint64_t`），`epoll_create()` 改為 `epoll_create1(EPOLL_CLOEXEC)`，並以單調遞增 `SequenceNumber`（`WAKE_EVENT_FD_SEQ = 1`）搭配 `mRequestsGeneration` 避免 FD 關閉重用導致的過期事件誤派發。此外，Android 17 新增 `nativeSetSkipEpollWaitForZeroTimeout()`，當 `timeoutMillis == 0` 且沒有額外註冊 FD（`mRequests.empty()`）時直接略過 `epoll_wait()` 系統呼叫以壓低每幀延遲。
> 4. **64 位元 `mPtr` 引用計數保護、可觀測性與 `Message` 物件池在併發模式下停用**：Java `MessageQueue.mPtr` 由 32 位元 `int` 升級為 64 位元 `long`，C++ `NativeMessageQueue` 繼承 `RefBase` + `LooperCallback`（支援 `setFileDescriptorEvents()` 回呼 Java `dispatchEvents()` 與跨 JNI 例外延遲重拋 `mExceptionObj`），Concurrent 模式更以 `incrementMptrRefs()` / `decrementMptrRefs()` 防止 `quit()` 釋放 `NativeMessageQueue` 與他執行緒 `nativeWake()` 競態。`Looper.loop()` 拆出 `loopOnce()` 整合 `Observer`、`LooperDoctor`、`SlowDelivery`/`SlowDispatch`、Perfetto SDK 追蹤與 `Binder.clearCallingIdentity()` 校驗；且當 `MessageQueue.getUseConcurrent()` 為真時，`Message.obtain()` 與 `recycleUnchecked()` **停用全域 `sPool` 物件池**，避免無鎖佇列遍歷時遭遇 CAS ABA 問題。

---

## 二、小節規劃與原書對照

本章完整保留原書 4 大節（13.1–13.4）的結構——「建立消息隊列 → 消息循環 → 消息發送 → 消息處理與 IdleHandler」，在每節中以預設 `CombinedMessageQueue`（同時對照 Legacy 單鎖鏈表與 Concurrent 併發實作，並於關鍵處補充 `CombinedDeliMessageQueue` 無鎖 DeliQueue 對照）完整呈現 Android 17 的全貌（圖表皆以 PlantUML 繪製，存於 `ch13-message-queue/figs/`）：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `13.1-create-message-queue.md` | 13.1 建立執行緒消息隊列（`Looper.prepare`、`MessageQueue` 三實作選型、`NativeMessageQueue` 與 `eventfd` / `epoll`） | 13.1（p.653–658） | **保留並深度重寫**：對應原書架構圖與建構鏈路——`Looper.prepareMainLooper` / `prepare(quitAllowed)` → `new Looper` → `new MessageQueue(quitAllowed)`（解析 `Android.bp` `messagequeue-gen` 三實作與 `computeUseConcurrent()` / `USE_NEW_MESSAGEQUEUE`、`setSkipEpollWaitForZeroTimeout`）→ `android_os_MessageQueue_nativeInit`（`jlong mPtr` + `incStrong`）→ `NativeMessageQueue` → `libutils/Looper`（`eventfd(0, EFD_NONBLOCK \| EFD_CLOEXEC)` + `rebuildEpollLocked` `epoll_create1(EPOLL_CLOEXEC)` + `WAKE_EVENT_FD_SEQ`） |
| `13.2-message-loop.md` | 13.2 執行緒消息循環過程（`Looper.loop` / `loopOnce`、`MessageQueue.next` 雙模式、`nativePollOnce` 與 `Looper::pollInner`） | 13.2（p.658–663） | **保留並重追現代雙模式與原生迴圈**：對應原書 Step 1–Step 7——`Looper.loop` / `loopOnce` → `MessageQueue.next`（解剖 `nextLegacy()` 同步屏障與 `mBlocked`、`nextConcurrent()` / `nextMessage()` 雙 `ConcurrentSkipListSet` 與 `mStackStateTimedPark`，並對照 `CombinedDeliMessageQueue` 的 `nextDeliQueue()` `heapSweep` / `drainFreelist` / `WaitState` CAS）→ `Binder.flushPendingCommands` → `nativePollOnce` → `NativeMessageQueue::pollOnce`（例外延遲重拋）→ `Looper::pollOnce` / `pollInner`（零超時略過 `epoll_wait`、`WAKE_EVENT_FD_SEQ` → `awoken()` 讀 `uint64_t`、原生 `mMessageEnvelopes` 與 `LooperCallback` / `NativeMessageQueue::handleEvent`） |
| `13.3-send-message.md` | 13.3 執行緒消息發送過程（`Handler.sendMessage`、`enqueueMessage` 雙模式、Sync Barrier 與 `eventfd` 喚醒） | 13.3（p.663–668） | **保留並補齊併發入隊與 Sync Barrier**：對應原書 Step 1–Step 5——`Handler` 建構子與 `Handler.createAsync` → `sendMessage` / `sendMessageDelayed` / `sendMessageAtTime` → `Handler.enqueueMessage`（`target`、`workSourceUid`、`FLAG_ASYNCHRONOUS`）→ `MessageQueue.enqueueMessage`（① `enqueueMessageLegacy` 的 `mLast` O(1) 尾插與 Sync Barrier 非同步喚醒規則；② `enqueueMessageConcurrent` 的無鎖 `StackNode` CAS 與 `concurrentWake` / `incrementMptrRefs`；③ 對照 `CombinedDeliMessageQueue` 的 Treiber Stack `pushMessage` + `mWaitState` CAS）→ `postSyncBarrier` / `removeSyncBarrier` → `nativeWake` → `NativeMessageQueue::wake` → `Looper::wake`（寫 `uint64_t inc = 1` 至 `mWakeEventFd`） |
| `13.4-handle-message-and-idle.md` | 13.4 執行緒消息處理與空閒處理器（`Looper.loopOnce`、`Handler.dispatchMessage`、`Message` 池化演進與 `IdleHandler`） | 13.4（p.668–674） | **保留並重追現代派發監控與 `Message` 回收**：對應原書 Step 1–Step 3 與 `IdleHandler`——`Looper.loopOnce`（`msg == null` 退出語意、Perfetto 追蹤、`Observer`、`ThreadLocalWorkSource`、`LooperDoctor`、`SlowDelivery`/`SlowDispatch`、`Binder.clearCallingIdentity` 核驗）→ `Handler.dispatchMessage` / `dispatchMessageImpl`（`msg.callback.run()` → `mCallback.handleMessage` → `handleMessage`）→ `Message.recycleUnchecked` 與 `Message.obtain`（剖析為何 Concurrent/DeliQueue 停用 `sPool` 避免 ABA 問題）→ `IdleHandler`（`addIdleHandler`/`removeIdleHandler`/`queueIdle` 於 Legacy 與 Concurrent 模式下的觸發條件、快照執行與 `keep == false` 移除） |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 13.1 建立執行緒消息隊列（`13.1-create-message-queue.md`）
- **情景**：應用程式主執行緒（承接 12.3 節 `ActivityThread.main()`）呼叫 `Looper.prepareMainLooper()`，或工作執行緒（如 `HandlerThread`）呼叫 `Looper.prepare()`，為該執行緒建立跨越 Java、JNI 與 C++ 三層的消息隊列與 `epoll` + `eventfd` 監聽基礎設施。
- **起點 → 終點**：
  - Java 層 `Looper` 與 `MessageQueue` 建構：`Looper.prepareMainLooper()` → `Looper.prepare(false)`（檢查 `sThreadLocal.get() == null`）→ `new Looper(false)` → `new MessageQueue(false)`（`getUseConcurrent()` → `computeUseConcurrent()` 依 `USE_NEW_MESSAGEQUEUE` / `Flags` / `UserHandle.isCore()` 鎖定雙模式；呼叫 `nativeInit()` 寫入 `long mPtr`；呼叫 `setSkipEpollWaitForZeroTimeout(mPtr)`）
  - JNI 橋接建立：`android_os_MessageQueue_nativeInit()`（`new NativeMessageQueue()`、`incStrong(env)`、回傳 `jlong` 指標；`NativeMessageQueue` 建構子透過 `Looper::getForThread()` / `new Looper(false)` / `Looper::setForThread()` 綁定 C++ `thread_local static sp<Looper> gThreadLocalLooper`）
  - C++ 層 `eventfd` 與 `epoll` 初始化：`libutils/Looper.cpp::Looper::Looper(false)`（`mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC))`）→ `rebuildEpollLocked()`（`mEpollFd.reset(epoll_create1(EPOLL_CLOEXEC))`、`epoll_ctl(EPOLL_CTL_ADD, mWakeEventFd, EPOLLIN, data.u64 = WAKE_EVENT_FD_SEQ)`）+ `Looper::setSkipEpollWaitForZeroTimeout()`。

### 13.2 執行緒消息循環過程（`13.2-message-loop.md`）
- **情景**：執行緒建立消息隊列後呼叫 `Looper.loop()` 進入永駐事件迴圈；當尚無到期消息時，執行緒主動沖刷待處理 Binder 命令並在 `epoll_wait()` 上睡眠讓出 CPU，直到被 `eventfd` 喚醒、到達下一條延遲消息時間或有註冊 FD 事件就緒。
- **起點 → 終點**：
  - Java 層迴圈與取訊：`Looper.loop()`（檢查 `myLooper()`、`Binder.clearCallingIdentity()`、讀 `getThresholdOverride()`）→ `for (;;) loopOnce(...)` → `MessageQueue.next()`：
    - Legacy 模式 `nextLegacy()`：檢查 `mPtr == 0` → 必要時 `Binder.flushPendingCommands()` → `nativePollOnce(ptr, nextPollTimeoutMillis)` → `synchronized (this)` 遇 `msg.target == null`（Sync Barrier）則跳過同步消息掃描 `msg.isAsynchronous()`；到期則出隊返回，未到期則算 `nextPollTimeoutMillis` 並標記 `mBlocked = true`
    - Concurrent 模式 `nextConcurrent()`：`nextMessage(false, false)` 掃描 `StackNode` 並取出 `mPriorityQueue` / `mAsyncPriorityQueue`（`ConcurrentSkipListSet`）最早到期消息，必要時寫入 `mStackStateTimedPark` 協調喚醒；並對照 `CombinedDeliMessageQueue.nextDeliQueue()` 的 `mStack.heapSweep()` / `drainFreelist()` / `mWaitState` CAS
  - JNI 與 C++ 層 `epoll_wait` 阻塞與喚醒：`MessageQueue.nativePollOnce(ptr, timeoutMillis)` → `android_os_MessageQueue_nativePollOnce()` → `NativeMessageQueue::pollOnce(env, obj, timeoutMillis)`（暫存 `mPollEnv`/`mPollObj`、結束後若 `mExceptionObj != NULL` 則 `env->Throw`）→ `Looper::pollOnce()` → `Looper::pollInner()`（校正 `mNextMessageUptime`；若 `!sSkipEpollWaitIfPossible || originalTimeout != 0 || hasRequests` 才呼叫 `epoll_wait(epollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis)`；遇到 `seq == WAKE_EVENT_FD_SEQ` 且 `EPOLLIN` 時呼叫 `awoken()` 讀取 8 位元組 `uint64_t`；接著派發到期原生 `mMessageEnvelopes` 與 `mResponses` 中的 `LooperCallback::handleEvent`，含 `NativeMessageQueue::handleEvent` 回呼 Java `MessageQueue.dispatchEvents`）。

### 13.3 執行緒消息發送過程（`13.3-send-message.md`）
- **情景**：另一條執行緒（例如 Binder 執行緒收到 AMS 的 `IApplicationThread` 呼叫，或背景工作執行緒完成資料載入）透過目標執行緒的 `Handler` 發送普通消息、延遲消息或非同步消息，或系統（如 `ViewRootImpl`）透過 `postSyncBarrier()` 插入同步屏障，將消息寫入目標 `MessageQueue` 並在必要時透過 `eventfd` 喚醒沉睡中的目標執行緒。
- **起點 → 終點**：
  - `Handler` 封裝與入隊入口：`new Handler(Looper)` / `Handler.createAsync(Looper)` → `Handler.sendMessage()` → `sendMessageDelayed()` → `sendMessageAtTime()`（或 `sendMessageAtFrontOfQueue()` 設 `when = 0`）→ `Handler.enqueueMessage(queue, msg, uptimeMillis)`（設定 `msg.target = this`、`msg.workSourceUid = ThreadLocalWorkSource.getUid()`、若 `mAsynchronous` 則 `msg.setAsynchronous(true)`）→ `queue.enqueueMessage(msg, uptimeMillis)`
  - `MessageQueue` 雙模式入隊與喚醒判定：
    - Legacy 模式 `enqueueMessageLegacy()`：`synchronized (this)` 檢查 `isInUse()` / `mQuitting`、`markInUse()`、`incAndTraceMessageCount()`；頭插（`p == null || when == 0 || when < p.when`）設 `needWake = mBlocked`；尾／中插運用 `mLast` 尾指針 O(1) 尾插，僅在隊頭為 Sync Barrier（`p.target == null`）且新消息為首條非同步消息時設 `needWake = true`
    - Concurrent 模式 `enqueueMessageConcurrent()` → `enqueueMessageUnchecked()`：透過 `VarHandle` CAS 將 `MessageNode` 壓入無鎖 `mState` 堆疊，依原頂端狀態節點（`sStackStateActive` / `sStackStateParked` / `mStackStateTimedPark`）判定是否呼叫 `concurrentWake()`（經 `incrementMptrRefs()` 保護 `mPtr` 後呼叫 `nativeWake`）；並對照 `CombinedDeliMessageQueue` 的 `mStack.pushMessage(msg)` + `sWaitState.compareAndSet()` 與 `postSyncBarrier()` / `removeSyncBarrier()`
  - JNI 與 C++ 層 `eventfd` 喚醒：`MessageQueue.nativeWake(mPtr)` → `android_os_MessageQueue_nativeWake()` → `NativeMessageQueue::wake()` → `Looper::wake()`（`uint64_t inc = 1; TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)))`，喚醒阻塞在 `epoll_wait()` 的目標執行緒）。

### 13.4 執行緒消息處理與空閒處理器（`13.4-handle-message-and-idle.md`）
- **情景**：目標執行緒自 `MessageQueue.next()` 取得一筆到期 `Message` 後，在 `Looper.loopOnce()` 內完成追蹤、監控與身分校驗並透過 `Handler.dispatchMessage()` 分派執行，結束後回收消息物件；當隊列暫時沒有到期消息、即將再次進入 `epoll_wait()` 休眠前，則依序執行已註冊的 `IdleHandler.queueIdle()` 處理空閒工作。
- **起點 → 終點**：
  - `Looper.loopOnce()` 監控與派發主幹：`Message msg = me.mQueue.next()`（若 `msg == null` 則直接返回 `false` 結束 `Looper.loop()`）→ Perfetto `message_queue_receive` 追蹤 → `me.mLogging` → `sObserver.messageDispatchStarting()` → `ThreadLocalWorkSource.setUid(msg.workSourceUid)` → `LooperDoctor.startMessageTimer()` → `msg.target.dispatchMessage(msg)` → `Thread.interrupted()`（依旗標清理中斷位）→ `observer.messageDispatched()` → `finally` 停止 `LooperDoctor` 計時、還原 `WorkSource`、結束 `Trace` → `showSlowLog()`（Slow delivery / Slow dispatch 告警）→ `Binder.clearCallingIdentity()` 校驗身分未被汙染 → `msg.recycleUnchecked()`
  - `Handler` 三級派發：`Handler.dispatchMessage()` → `dispatchMessageImpl()`：① 若 `msg.callback != null`（例如 `Handler.post(Runnable)`）執行 `handleCallback(msg)`（`msg.callback.run()`）；② 否則若 `mCallback != null` 呼叫 `mCallback.handleMessage(msg)`，返回 `true` 則攔截結束；③ 否則呼叫子類覆寫的 `handleMessage(msg)`
  - `Message` 回收與物件池演進：`Message.recycleUnchecked()` 與 `Message.obtain()`（Legacy 模式歸還長度上限 50 的 `sPool` 鏈表；Concurrent / DeliQueue 模式下 `MessageQueue.getUseConcurrent() == true`，停用 `sPool` 僅呼叫 `clearReferenceFields()` + `markInUse()` 以根絕 CAS ABA 問題）
  - 空閒消息處理器（`IdleHandler`）：`MessageQueue.addIdleHandler()` / `removeIdleHandler()` → 於 `nextLegacy()` / `nextConcurrent()` 首次無到期消息時（`pendingIdleHandlerCount < 0` 且隊列空或隊頭尚未到期）複製至 `mPendingIdleHandlers` 陣列，在鎖外依序呼叫 `idler.queueIdle()`，回傳 `false` 則自動移除；跑完後重置 `pendingIdleHandlerCount = 0` 並將 `nextPollTimeoutMillis` 清為 `0`，立即重新掃描隊列以免遺漏期間抵達的新消息。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| 單一 `frameworks/base/core/java/android/os/MessageQueue.java`（單鎖有序單向鏈表） | `core/java/Android.bp` `messagequeue-gen` 依旗標選自 `CombinedMessageQueue`（預設雙模式）、`CombinedDeliMessageQueue`（雙模式 + 無鎖 DeliQueue）或 `LegacyMessageQueue` | 架構重構 | 預設 `CombinedMessageQueue` 對核心進程與 Android 17+ App 啟用雙 `ConcurrentSkipListSet` + 無鎖 `StackNode`；另提供無鎖 DeliQueue（Treiber Stack + 雙 Min-Heap + `WaitState` CAS）；Legacy 保留相容反射 `mMessages` |
| `MessageQueue.mPtr`（`int` 32 位元裸指標） | `MessageQueue.mPtr`（`long` 64 位元）+ `NativeMessageQueue` 繼承 `RefBase`（`incStrong`/`decStrong`）+ Concurrent `incrementMptrRefs()` | 邏輯改變 + 安全強化 | 支援 64 位元位址與引用計數生命週期管理，防止 `quit()` 釋放與併發 `nativeWake()` 造成 UAF |
| `msg.target == null` 表示「退出消息」（於 `Looper.loop()` 判斷） | `MessageQueue.next()` 返回 **`null`** 表示退出；**`msg.target == null`** 改表示 **Sync Barrier**（`postSyncBarrier` / `removeSyncBarrier`） | 語意反轉 + 新增 | 同步屏障可阻擋普通同步消息並讓 `FLAG_ASYNCHRONOUS` 非同步消息優先插隊（如 VSync 繪製） |
| `MessageQueue.enqueueMessage()`（O(n) 從頭掃描插入） | Legacy 模式新增 `mLast` 尾指針 O(1) 尾插與 `mAsyncMessageCount`；Concurrent 模式無鎖 `StackNode` CAS；DeliQueue 模式無鎖 Treiber Stack `pushMessage()` + `mWaitState` CAS | 演算法重構 | 消除鎖競爭與 UI 執行緒優先權反轉，大幅提升高併發入隊吞吐量 |
| `Message.obtain()` / `recycle()` 一律使用 `sPool` 單向鏈表物件池 | Legacy 模式維持 `sPool`（上限 50）；Concurrent / DeliQueue 模式下 `MessageQueue.getUseConcurrent()` 為真，停用 `sPool` 直接 `new Message()`、`recycleUnchecked()` 僅清欄位並標記 `FLAG_IN_USE` | 邏輯改變 | 防止併發移除執行緒與物件重用產生 CAS ABA 問題；現代 ART GC 成本已大幅降低 |
| `frameworks/base/libs/utils/Looper.cpp`（`pipe()` 雙 FD `mWakeReadPipeFd`/`mWakeWritePipeFd` + `epoll_create()` + 裸 `data.fd`） | `platform/system/core/libutils/Looper.cpp`（`eventfd(0, EFD_NONBLOCK \| EFD_CLOEXEC)` 單一 `mWakeEventFd` + `epoll_create1(EPOLL_CLOEXEC)` + 64-bit `SequenceNumber` + `mRequestsGeneration` + `setSkipEpollWaitForZeroTimeout()`） | 搬家 + 邏輯重寫 | 省 1 個 FD、消除關閉重用 FD 的 Stale Response、零超時無註冊 FD 時跳過 `epoll_wait()` 系統呼叫 |
| `NativeMessageQueue`（僅包 `pollOnce` / `wake`） | `NativeMessageQueue : public MessageQueue, public LooperCallback`（新增 `setFileDescriptorEvents`、`handleEvent`、`WeakLooperCallback`、`mExceptionObj` 跨 JNI 延遲重拋） | 邏輯擴充 | 支援 Java 層 `OnFileDescriptorEventListener` 與原生回呼例外安全傳回 Java 層 |
| `Looper.loop()` 內直接取訊與 `dispatchMessage` | `Looper.loop()` 抽離 `loopOnce()`，加入 `Binder.clearCallingIdentity()` 校驗、`Observer`、`LooperDoctor`、`ThreadLocalWorkSource`、`SlowDelivery`/`SlowDispatch` 與 Perfetto SDK 追蹤 | 邏輯擴充 | 強化跨進程身分安全與卡頓／耗時診斷可觀測性 |
| `Handler()` 無參建構子為主流，無非同步模式 | `Handler()` 無參建構子已 `@Deprecated`；新增 `Handler.createAsync(Looper)`、`mAsynchronous` 與 `dispatchMessageImpl()` | API 演進 | 防止未顯式指定 `Looper` 造成隱性錯誤，並為同步屏障提供標準非同步 Handler 建構介面 |
| —（無已知未確認項目） | — | — | 本章所有類別、建構規則、檔案路徑與方法簽名皆已於 `platform/frameworks/base`（`94b4c163b`）與 `platform/system/core`（`545d248`）實機驗證，0 筆 `TODO: 未確認` |
