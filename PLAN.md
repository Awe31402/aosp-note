# 執行規劃

> 任務定義見 [BRIEF.md](BRIEF.md)，逐章狀態見 [progress.md](progress.md)。這份文件寫的是**怎麼做**：工作流程、工具、驗證方式，以及第 5 章的具體計畫。
> 本文中的檔案路徑、專案名稱、kernel 分支都是**依經驗列出的候選，尚未對 `android-17.0.0_r1` 驗證**，一律標記〔待確認〕，開工時逐一查證後再改掉標記。

---

## 一、通用工作流程（每章適用）

### 階段 0：準備
1. 讀原書該章（PDF 分段讀，每次最多 20 頁），整理出：
   - 原書的情景清單（每節要回答什麼問題）
   - 原書的呼叫鏈骨架（Step 1…N 的函式序列）
   - 原書的核心觀念（之後要用自己的話重寫並標注頁碼）
2. 把原書的每個類別/函式對應到 17：還在嗎？搬到哪裡？改名了嗎？做成一張**對照表**（大綱的附錄）。

### 階段 1：大綱（檢查點 1）
產出 `chXX-<topic>/00-outline.md`，內容：
- 新的小節清單，每節附**情景說明**和**呼叫鏈的起點 → 終點**
- 與原書小節的對應（保留 / 合併 / 拆分 / 新增 / 刪除，附理由）
- 需要的原始碼專案清單（已驗證存在於 17 tag）
- 原書 → 17 的類別/函式對照表
- 已知的 `TODO: 未確認` 項目

→ **停下來等使用者確認。**

### 階段 2：第一節（檢查點 2）
照下面的「小節寫作流程」寫第一節。→ **停下來等使用者確認深度、篇幅、格式。** 確認後的寫法就成為後續所有小節的範本。

### 階段 3：全章
依大綱逐節完成。每寫完一節就更新 `progress.md`。

### 小節寫作流程
1. 從情景起點開始，**每一步都實際讀原始碼**（Gitiles 或本機 clone），確認下一個被呼叫的函式。
2. 每個 Step 的寫法：
   ```markdown
   #### Step N：`ClassName::method`
   **位置**：`<project>/<path>` 〔android-17.0.0_r1〕

   ```cpp
   // 只貼呼叫鏈上用得到的部分，其餘用 ... 省略
   ```

   解說……

   > 〔與 2.3 差異〕邏輯改變：……（原書 p.XXX）
   ```
3. 小節結尾附一張 PlantUML 時序圖總結呼叫鏈（原始檔存於該章節 `figs/<節號>-seq.puml`，架構圖為 `-arch.puml`、類別圖為 `-class.puml`，以 `java -Djava.awt.headless=true -DPLANTUML_LIMIT_SIZE=8192 -jar /usr/share/plantuml/plantuml.jar -tpng -charset UTF-8` 編譯為白底高對比 `.png`，Markdown 以 `![...](figs/xxx.png)` 引用，確保深淺主題皆清晰可讀；中文字型固定 `Noto Sans CJK TC`，注意以 `~__`／`~--` 跳脫 C 符號與 CLI 旗標避免觸發 Creole 底線／刪除線）。
4. **自我檢查**（寫完一節就做）：
   - 每段程式碼都能對應到實際檔案？用 grep 在本機 clone 中逐段比對。
   - 有沒有沒標來源的斷言？有就補來源，否則改成 `TODO: 未確認`。
   - 〔與 2.3 差異〕的判斷是否有原書頁碼作證據？

---

## 二、原始碼取得策略

| 情境 | 做法 |
|---|---|
| 讀單一檔案、看目錄 | Gitiles：`https://android.googlesource.com/<project>/+/refs/tags/android-17.0.0_r1/<path>?format=TEXT`，再 `base64 -d` |
| 要搜尋符號、追呼叫者 | 淺 clone 該專案到 `aosp-note/src/`：`git clone --depth 1 --branch android-17.0.0_r1 https://android.googlesource.com/<project> src/<project>`，再用 `grep -rn` |
| 確認專案名稱 | 讀 17 tag 的 `platform/manifest/default.xml` |
| Kernel | 淺 clone `kernel/common` 的 `android17-6.18` 分支（已驗證，commit `028363aa3cfa`）到 `src/kernel/common` |

- **本機 clone 優先**：一章會讀幾十到上百個檔案，clone 一次之後搜尋比較快，也方便逐段驗證程式碼。磁碟空間充足（約 357G 可用）。
- `src/` 只放原始碼，不放筆記；可以隨時刪掉重抓。
- 筆記中引用的路徑一律寫**專案內的相對路徑 + tag**，不寫本機路徑。
- 本機沒有 `rg`，用 `grep -rn`。

---

## 三、第 5 章 Binder 的具體計畫

### 3.1 原書第 5 章結構（p.152–334）

| 原書小節 | 主題 |
|---|---|
| 5.1 | Binder 驅動程式：資料結構、初始化、open、mmap、核心緩衝區管理 |
| 5.2 | Binder 進程間通信庫（libbinder） |
| 5.3 | Binder 進程間通信應用實例 |
| 5.4 | Binder 物件引用計數技術（本地 / 實體 / 引用 / 代理物件的生命週期） |
| 5.5 | Binder 物件死亡通知機制 |
| 5.6 | Service Manager 的啟動過程 |
| 5.7 | Service Manager 代理物件的取得過程 |
| 5.8 | Service 組件的啟動過程（註冊 Service、啟動 Binder 線程池） |
| 5.9 | Service 代理物件的取得過程 |
| 5.10 | Binder 進程間通信機制的 Java 介面 |

### 3.2 預期在 17 會遇到的主要差異（全部〔待確認〕）
- **驅動**：binderfs、多個 binder domain（binder / hwbinder / vndbinder）、`binder_alloc` 從 binder.c 拆出、可能存在的 Rust binder 實作。
- **libbinder**：`ProcessState` / `IPCThreadState` 仍存在但已大幅修改；可能有 `libbinder_ndk`、`libbinder_rs`、RPC binder（socket 上的 binder）。
- **servicemanager**：從 C 寫的 `service_manager.c` 改成 C++ + AIDL 介面（`IServiceManager.aidl`），可能加上 lazy service、VINTF 宣告檢查。
- **AIDL**：原書手寫 `IXxx` / `BnXxx` / `BpXxx`；現在由 aidl 編譯器產生，要追產生出來的程式碼長什麼樣。
- **Java 層**：`Binder`、`BinderProxy`、`ServiceManager` 仍在，JNI 層 `android_util_Binder.cpp`。

### 3.3 第 5 章原始碼專案（已於檢查點 1 驗證）

| 專案 | 基準 | 用途 |
|---|---|---|
| `platform/frameworks/native` | `android-17.0.0_r1` (`ae266dc`) | `libs/binder/`（libbinder、`IServiceManager.aidl`、`BackendUnifiedServiceManager`）、`cmds/servicemanager/`、`services/gpuservice/` |
| `platform/system/tools/aidl` | `android-17.0.0_r1` (`9a94664`) | AIDL 編譯器（`generate_cpp.cpp`、`generate_java_binder.cpp`、`tests/golden_output/`） |
| `platform/frameworks/base` | `android-17.0.0_r1` (`94b4c16`) | `core/java/android/os/`（`Binder.java`、`BinderProxy.java`、`ServiceManager.java`、`ServiceManagerNative.java`）、`core/jni/android_util_Binder.cpp`、`android_os_ServiceManagerNative.cpp` |
| `platform/system/core` | `android-17.0.0_r1` (`545d248`) | `rootdir/init.rc`（掛載 `/dev/binderfs` 與 `/dev/*binder` symlink） |
| `kernel/common` | `android17-6.18` (`028363aa3cfa`) | `drivers/android/{binder.c,binder_internal.h,binder_alloc.c,binder_alloc.h,binderfs.c,binder_pick.c}`、`include/uapi/linux/android/binder.h` |

### 3.4 第 5 章大綱的執行步驟（檢查點 1 之前要做的事）
1. 讀原書第 5 章各節開頭與小結（不精讀全文），確認每節的情景和呼叫鏈骨架。
2. 從 17 tag 的 manifest 確認上表的專案名稱，並淺 clone。
3. 確認 17 對應的 GKI kernel 分支，淺 clone `kernel/common`。
4. 建立「原書 → 17」對照表：原書提到的主要結構體和函式（`binder_proc`、`binder_thread`、`binder_node`、`binder_ref`、`binder_buffer`、`binder_transaction`、`ProcessState`、`IPCThreadState`、`svcmgr_handler`……）在 17 各自的位置和狀態。
5. 規劃新的小節。初步想法（待對照表完成後調整）：
   - 驅動部分保留原書順序（資料結構 → 初始化 → open → mmap → 緩衝區），加上 binderfs 和 binder domain。
   - servicemanager 改寫成 C++/AIDL 版本的啟動和註冊/查詢流程。
   - 「應用實例」改用 AIDL 寫，並追 aidl 產生的 Bn/Bp 程式碼。
   - 評估是否新增「RPC binder」或「Rust binder」小節；**原則上不加**（Limits：不新增原書沒有的主題），只在對照表中註記。
6. 寫出 `ch05-binder/00-outline.md`，更新 `progress.md`，**停下來等確認**。

### 3.5 風險與對策

| 風險 | 對策 |
|---|---|
| Gitiles 不能搜尋，追呼叫者很慢 | 一開始就淺 clone 需要的專案 |
| 17 的 tag 剛發布，某些專案的 tag 可能不完整 | 從 manifest 確認每個專案的 revision；缺的話記到 `progress.md` 的待決事項 |
| Kernel 版本不明，binder 驅動行為隨版本變化 | 檢查點 1 時就確定分支，之後整章固定使用 |
| C 等級篇幅很大，單一小節就可能很長 | 檢查點 2 時跟使用者校準篇幅，必要時把小節拆成多個檔案 |
| 憑記憶寫出不存在的函式或行為 | 每節的自我檢查都用 grep 逐段比對；無法證實就標 `TODO: 未確認` |

---

## 四、跨對話接續的方式
- 開新對話時：讀 `BRIEF.md` → `progress.md` → `PLAN.md`，從 `progress.md` 的「下一步」接續。
- 每個工作階段結束前：更新 `progress.md` 的目前狀態、章節進度表、工作日誌。
- 如果決定有變（例如改了格式或範圍）：同時更新 `BRIEF.md` 和 `progress.md` 的「已確認的決定」。
