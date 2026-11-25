# 第 4 章：`logd` 與 `liblog` 日誌系統 — 大綱

> **原書對應**：第 4 章〈Logger 日志系统〉（p.82–151）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **撰寫日期**：2026-11-24
> **狀態**：✅ 全章完成（`4.1`–`4.6` 共 6 節皆已撰寫並逐行比對原始碼驗證通過）

---

## 一、原始碼專案清單與版本驗證

以下專案皆已對照 `android-17.0.0_r1` tag 實際檢出確認：

| 專案（Gitiles） | Tag | Commit（短碼） | 第 4 章涵蓋範圍 |
|---|---|---|---|
| `platform/system/logging` | `android-17.0.0_r1` | `d35ba62` | `logd/`（`logd.rc`、`main.cpp`、`LogBuffer.h`、`SerializedLogBuffer.*`、`SerializedLogChunk.*`、`SerializedLogEntry.h`、`SerializedFlushToState.*`、`CompressionEngine.*`、`LogListener.*`、`LogReader.*`、`LogReaderList.*`、`LogReaderThread.*`、`CommandListener.*`、`LogKlog.*`、`LogAudit.*`、`LogdNativeService.*`、`LogPermissions.*`、`PruneList.*`）、`liblog/`（`include/android/log.h`、`include/log/*.h`、`include/private/android_logger.h`、`logger_write.cpp`、`logd_writer.cpp`、`pmsg_writer.cpp`、`logger_read.cpp`、`logd_reader.cpp`、`pmsg_reader.cpp`、`log_event_write.cpp`、`log_event_list.cpp`、`properties.cpp`、`logprint.cpp`、`event_tag_map.cpp`、`README.protocol.md`）、`logcat/`（`logcat.cpp`、`logcat_utils.h`、`process_names.*`、`logcat.proto`）、`rust/src/` |
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163` | `core/java/android/util/{Log.java,Slog.java,EventLog.java}`、`core/jni/{android_util_Log.cpp,android_util_Log.h,android_util_EventLog.cpp,eventlog_helper.h}` |

> **重大架構演進說明（2.3 → 17）**：
> 原書第 4 章所分析的 Linux 核心 Logger 驅動（`drivers/staging/android/logger.c` 及 `/dev/log/{main,events,radio,system}` 字元裝置）已於多年前從 Android 核心完全移除。Android 17 改採**純使用者空間守護行程 `logd` + Unix Domain Socket（`/dev/socket/{logdw,logdr,logd}`）+ zstd 壓縮環形緩衝區（`SerializedLogBuffer`）+ `/dev/pmsg0` 重啟留存**架構，且原始碼已從 `system/core` 獨立抽離至 `platform/system/logging`。本章完整重寫並對應原書 6 節結構，讓讀者能直接沿 Android 17 的實際呼叫鏈貫通寫入、緩衝壓縮、讀取授權、C/C++ 與 Java 介面及 `logcat` 工具。

---

## 二、小節規劃與原書對照

本章維持原書 6 大節（4.1–4.6）的教學骨架，全面改寫為 Android 17 的 `logd` + `liblog` 現代架構：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `4.1-log-format.md` | 4.1 日誌緩衝區與記錄格式 | 4.1（p.83–84） | **保留並大幅擴充**：從 2.3 的 3 類緩衝區與 v1 `logger_entry`（20 B）更新為 17 的 8 種 `log_id_t`、v4 `logger_entry`（28 B，含 `lid`/`uid`）、文字日誌與二進位事件日誌格式、`/dev/socket/logdw` 線上封包（`android_log_header_t`）與 `logd` 內部 `SerializedLogEntry` 格式 |
| `4.2-logd-daemon.md` | 4.2 `logd` 守護行程與壓縮緩衝區 | 4.2（p.84–100） | **全面重寫**：原書為核心驅動 `logger.c`；17 完整追蹤使用者空間 `/system/bin/logd` 的 `logd.rc`、三組 Unix socket、`main.cpp` 初始化、`LogListener`（`SO_PASSCRED`）→ `SerializedLogBuffer::Log`（`SerializedLogChunk` + zstd 壓縮 + `PruneList` 淘汰）、`LogReader` / `LogReaderThread` → `SerializedLogBuffer::FlushTo`（即時解壓縮與阻塞喚醒）、以及 `CommandListener` / `LogdNativeService` / `LogKlog` / `LogAudit` |
| `4.3-liblog.md` | 4.3 執行期日誌函式庫 `liblog` | 4.3（p.101–107） | **大幅改寫**：從 `logd_write.c` 直開 `/dev/log/*` 改為 `system/logging/liblog/`：`logger_write.cpp`（`__android_log_write_log_message` 與可擴充 logger callback、屬性過濾 `properties.cpp`）→ `logd_writer.cpp`（`LogdSocket` 雙 socket 阻塞/非阻塞分流、無鎖 CAS 連線、`dropped` 丟失回報）與 `pmsg_writer.cpp`（同步寫入 `/dev/pmsg0` 供重啟留存）；同時涵蓋 `logger_read.cpp` + `logd_reader.cpp` 讀取端 |
| `4.4-cpp-interface.md` | 4.4 C/C++ 日誌寫入介面 | 4.4（p.108–112） | **保留並更新**：標頭從 `cutils/log.h` 遷至 `log/log*.h` 與 NDK `android/log.h`；巨集從 `LOG*` 改名 `ALOG*`（含 `SLOG*`/`RLOG*`/`ALOG_ALWAYS_FATAL`）；詳解 `__android_log_print` / `__android_log_buf_print`、二進位事件寫入（`log_event_write.cpp` 與結構化 `log_event_list.cpp`），並簡述 `liblog` 的 Rust 綁定（`system/logging/rust/`） |
| `4.5-java-interface.md` | 4.5 Java 日誌寫入介面 | 4.5（p.112–117） | **保留並更新**：追蹤 `Log.java`（`println_native`、`@FastNative isLoggable`、`wtf` 與 `TerribleFailureHandler`）、`Slog.java`（`LOG_ID_SYSTEM`）、`EventLog.java`（`writeEvent`/`readEvents`）與 JNI 層 `android_util_Log.cpp`、`android_util_EventLog.cpp`（含 `eventlog_helper.h` 與 `android_log_context`） |
| `4.6-logcat.md` | 4.6 `logcat` 工具分析 | 4.6（p.118–151） | **大幅改寫**：從 2.3 手動 `select()` 多個 `/dev/log/*` 改為 `Logcat` C++ 類別 + `liblog` `android_logger_list_*` 串流讀取（由 `logd` 服務端依時間排序合併）；詳解基礎結構、`Run()` 命令列解析、讀取迴圈、`ProcessBuffer`（TEXT/BINARY/PROTO 三種輸出、`EventTagMap` 事件解碼、`std::regex`/UID/PID 過濾、`logprint.cpp` 格式化與 `RotateLogs()` 檔案輪轉） |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 4.1 日誌緩衝區與記錄格式（`4.1-log-format.md`）
- **情景**：梳理 Android 17 日誌系統在寫入線上、記憶體壓縮儲存與讀取輸出三階段所使用的全部緩衝區類型與二進位資料格式。
- **起點 → 終點**：`liblog/include/android/log.h`（`android_LogPriority`、`log_id_t` 8 種緩衝區）→ `liblog/include/private/android_logger.h`（`/dev/socket/logdw` 的 `android_log_header_t`、事件型別常數與 `android_event_*` 結構）→ `liblog/include/log/log_read.h`（`struct logger_entry` v4 28 bytes 與 `union log_msg`）→ `logd/SerializedLogEntry.h`（`logd` 內部壓縮塊條目格式）。

### 4.2 `logd` 守護行程與壓縮緩衝區（`4.2-logd-daemon.md`）
- **情景**：開機時 `init` 啟動 `/system/bin/logd` 建立三組 Unix Socket 與壓縮緩衝區；應用程式透過 `logdw` 寫入日誌並觸發 zstd 壓縮與淘汰；`logcat` 透過 `logdr` 驗證權限、即時解壓並串流讀取日誌。
- **起點 → 終點**：
  - 4.2.1 基礎結構與 `logd.rc`：`logd.rc` → `LogBuffer.h`、`SerializedLogBuffer.h`、`SerializedLogChunk.h`、`CompressionEngine.h`、`LogReaderList.h`、`LogReaderThread.h`；
  - 4.2.2 啟動初始化：`logd/main.cpp::main`（`DropPrivs`、`CreateLogBuffer` → `SerializedLogBuffer`、`LogReader`、`LogListener`、`CommandListener`、`LogAudit`、`LogKlog`、`readDmesg`、`LogdNativeService`、`SetProperty("logd.ready", "true")`）；
  - 4.2.3 寫入與壓縮淘汰：`LogListener::ThreadFunction`（`recvmsg` + `SO_PASSCRED` 取得 `ucred`）→ `SerializedLogBuffer::Log` → `SerializedLogChunk::Log` → 寫滿時 `FinishWriting()`（`ZstdCompressionEngine::Compress`）→ `MaybePrune()`（`PruneList` + `LogStatistics`）→ `reader_list_->NotifyNewLog()`；
  - 4.2.4 讀取與即時解壓：`LogReader::onDataAvailable`（`clientHasLogCredentials` 權限校驗與 UID 限制）→ `LogReaderThread` → `SerializedLogBuffer::FlushTo` / `SerializedFlushToState`（`IncReaderRefCount` 觸發 `ZstdCompressionEngine::Decompress`）→ `LogWriter::Write`（透過 `logdr` seqpacket 送出 `logger_entry`）；
  - 4.2.5 控制通道與授權：`CommandListener`（`/dev/socket/logd`）與 `LogdNativeService`（`ILogd::approve`/`decline`）。

### 4.3 執行期日誌函式庫 `liblog`（`4.3-liblog.md`）
- **情景**：任意原生行程呼叫 `liblog` 寫入一條日誌（或 `logcat` 透過 `liblog` 開啟緩衝區並讀取日誌）時，`liblog` 如何完成等級過濾、無鎖 socket 連線、雙路寫入（`logdw` + `/dev/pmsg0`）與讀取端協定封裝。
- **起點 → 終點**：
  - 寫入：`logger_write.cpp::__android_log_write_log_message` → `__android_log_logd_logger` → `write_to_log` → `logd_writer.cpp::LogdWrite`（`LogdSocket::BlockingSocket`/`NonBlockingSocket`、`writev`、`dropped` 計數回報）與 `pmsg_writer.cpp::PmsgWrite`（`/dev/pmsg0`）；搭配 `properties.cpp::__android_log_is_loggable_len`（系統屬性快取查表）；
  - 讀取：`logger_read.cpp::android_logger_list_alloc` / `android_logger_open` / `android_logger_list_read` → `logd_reader.cpp::logd_read`（連線 `/dev/socket/logdr` 送出查詢命令並讀回 `log_msg`）與 `pmsg_reader.cpp`。

### 4.4 C/C++ 日誌寫入介面（`4.4-cpp-interface.md`）
- **情景**：C/C++ 模組使用 `ALOGI` / `SLOGE` / `RLOGD` 與結構化事件 API 寫入各類日誌，追蹤巨集展開至 `liblog` 函式的完整路徑。
- **起點 → 終點**：`liblog/include/log/log.h` / `log_main.h` / `log_system.h` / `log_radio.h`（`ALOG*`/`SLOG*`/`RLOG*`/`ALOG_ALWAYS_FATAL`）→ `logger_write.cpp::__android_log_print` / `__android_log_buf_print` / `__android_log_buf_write` → `write_to_log`；二進位事件路徑：`log_event_write.cpp::__android_log_bwrite` 與 `log_event_list.cpp`（`android_log_create_context` / `android_log_write_int32` / `android_log_write_list`）；並簡述 `system/logging/rust/src/lib.rs` 綁定。

### 4.5 Java 日誌寫入介面（`4.5-java-interface.md`）
- **情景**：Java 應用程式或 `system_server` 呼叫 `Log.i`、`Log.wtf`、`Slog.w`、`EventLog.writeEvent` 時，從 Java 框架層經 JNI 落地到 `liblog`。
- **起點 → 終點**：
  - 文字日誌：`Log.java`（`i` / `wtf` / `isLoggable` / `println_native`）與 `Slog.java`（`LOG_ID_SYSTEM`）→ `core/jni/android_util_Log.cpp`（`android_util_Log_println_native` → `__android_log_buf_write`、`@FastNative android_util_Log_isLoggable` → `__android_log_is_loggable_len`）；
  - 事件日誌：`EventLog.java`（`writeEvent` / `readEvents`）→ `core/jni/android_util_EventLog.cpp`（`android_util_EventLog_writeEvent_*` 透過 `android_log_context` 寫入 `LOG_ID_EVENTS`、`android_util_EventLog_readEvents` 透過 `android_logger_list_*` 讀取）。

### 4.6 `logcat` 工具分析（`4.6-logcat.md`）
- **情景**：開發者在終端機執行 `logcat -b main,system -v threadtime MyTag:D *:S`（或帶 `-f`/`-r`/`-n` 輪轉、`-b events` 事件解碼、`--proto` 輸出），追蹤 `logcat` 從啟動、連線 `logd`、讀取、過濾到格式化輸出的全過程。
- **起點 → 終點**：`logcat/logcat.cpp::main` → `Logcat::Run`（解析選項、`SetLogFormat`、建立 `android_logger_list_alloc` + `android_logger_open`）→ 讀取主迴圈 `android_logger_list_read` → `Logcat::ProcessBuffer` → `liblog/logprint.cpp::android_log_processLogBuffer` / `android_log_processBinaryLogBuffer`（結合 `event_tag_map.cpp::EventTagMap`）→ `android_log_shouldPrintLine` + 正則/UID 過濾 → `android_log_formatLogLine` / `android_log_printLogLine` / `PrintToProto` → `WriteFully` 與 `RotateLogs()`。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| `kernel/goldfish/drivers/staging/android/logger.{c,h}`（`/dev/log/{main,events,radio,system}`） | `platform/system/logging/logd/`（`/system/bin/logd` + `/dev/socket/{logd,logdr,logdw}`） | 已移除並重寫 | 核心 Logger 驅動完全移除，改由使用者空間 `logd` 守護行程透過 Unix domain socket 集中管理 8 類日誌緩衝區 |
| `struct logger_entry`（v1，20 bytes，固定未壓縮環形緩衝區） | `struct logger_entry`（v4，28 bytes，含 `lid`/`uid`）+ `SerializedLogBuffer` / `SerializedLogChunk`（zstd 壓縮） | 邏輯改變 | `logd` 預設以 zstd 壓縮分塊儲存日誌，大幅節省 RAM 並支援 per-UID `PruneList` 淘汰與即時按需解壓 |
| `system/core/liblog/logd_write.c`（`__write_to_log_init` / `open("/dev/log/...")`） | `platform/system/logging/liblog/{logger_write.cpp,logd_writer.cpp,pmsg_writer.cpp,logd_reader.cpp}` | 改名搬家 + 邏輯改變 | 改為 `LogdSocket` 連線 `/dev/socket/logdw`（區分 `LOG_ID_SECURITY` 阻塞與一般非阻塞），同步寫入 `/dev/pmsg0` 支援跨重啟 pstore 留存，並提供統一 `android_logger_list_*` 讀取 API |
| `system/core/include/cutils/log.h`（`LOGV/D/I/W/E`） | `platform/system/logging/liblog/include/log/log*.h`（`ALOGV/D/I/W/E`）與 `include/android/log.h` | 改名搬家 | 巨集加上 `A` 前綴避免與 `math.h` 衝突，並透過 `liblog` 提供穩定 NDK C ABI 與 Rust 綁定 |
| `frameworks/base/core/java/android/util/{Log,Slog,EventLog}.java` + `core/jni/android_util_{Log,EventLog}.cpp` | 同路徑（`platform/frameworks/base`） | 邏輯改變 + 擴充 | `isLoggable` 改以 `@FastNative` 呼叫 `liblog` 屬性查表；`EventLog` 改用 `android_log_context` 列表編碼與 `android_logger_list_*` 讀取 |
| `system/core/logcat/logcat.cpp`（手寫 `select()` 多裝置 fd）+ `liblog/logprint.c` | `platform/system/logging/logcat/logcat.cpp`（`class Logcat` + `android_logger_list_read`）+ `liblog/logprint.cpp` | 搬家 + 邏輯改變 | 移除客戶端 `select()` 合併邏輯，改由 `logd` 服務端依序串流；新增 `--uid`/`--pid`/`-e` 正則、`--proto`、壓縮輸出與 `LogdNativeService` 授權審批 |
