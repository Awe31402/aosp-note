# 任務簡報：以 android-17.0.0_r1 重寫《Android系統源代碼情景分析（第三版）》

> 確認日期：2026-11-24
> 原書：羅升陽《Android系統源代碼情景分析（第三版）》，電子工業出版社 2017，本目錄下 PDF（849 頁，基準 Android 2.3）

## 1. Purpose（目的）

自己研讀現代 AOSP（`android-17.0.0_r1`）用的**個人學習筆記**。沿用原書的方法和章節骨架：先給一個具體情景，再沿著呼叫鏈一步步讀原始碼。

**成功標準**：讀完一章之後，能**脫離筆記，自己在 17 的原始碼裡重走一遍那個情景**，並且知道哪些地方跟原書 2.3 相同、哪些已經改變。

深度：**C 等級（完整重寫）**，詳細程度跟原書相同。

## 2. Situation（背景）

- 原書基準是 **Android 2.3**。之後的重大變化：Dalvik → ART、kernel Logger 驅動 → logd、legacy HAL → HIDL → AIDL HAL、Binder 驅動演進（binderfs、`/dev/binder` / `/dev/hwbinder` / `/dev/vndbinder`）、servicemanager 改寫並改用 AIDL、Project Treble 的 system/vendor 分離、WindowManager / Input / PackageManager 重構。**照搬原書的呼叫鏈幾乎一定會錯。**
- **原始碼只能透過 Gitiles 線上取得**（見 `.agents/skills/aosp-source-online`、`how-to-view-source-online.md`），不能全文搜尋。要追符號時，用 `git clone --depth 1 --branch android-17.0.0_r1 <project>` 把需要的專案抓到本機再 grep。本機沒有完整的 AOSP 樹。
- **Kernel 部分**（binder、ashmem、memfd）用 17 對應的 GKI `kernel/common` 分支，實際分支名稱在第 5 章檢查點 1 確認。
- 讀者有 Android 底層經驗，**不需要解釋 C++/Java 語法或 Linux 基礎**。

## 3. Inputs（資料來源）

**主要（作為程式碼和行為的依據）**
1. AOSP `android-17.0.0_r1` 原始碼。每段程式碼都要標註**專案 + 路徑 + 函式名**，而且必須是實際讀到的內容。
2. 對應的 GKI kernel 原始碼（`kernel/common`）。

**次要（只用來參考架構和寫法，不作為程式碼依據）**
3. 原書 PDF：參考情景設計、Step 拆法、觀念說明，並標注原書章節和頁碼。
4. source.android.com 官方文件。

**不能用**
- 部落格、CSDN、二手教學文章。
- 憑記憶寫出的程式碼片段。
- g720 / 任何 vendor BSP。

**找不到時的規則**：無法在原始碼中證實的函式、路徑、行為或版本演進說法，一律標記 `TODO: 未確認` 並留空，**不補上看似合理的內容，不猜版本號**。

## 4. Limits（限制）

- 不改方法論：保持「情景 → Step 1、2、3… → 原始碼 + 解說」，不寫成 API 手冊或架構總覽。
- 不逐版追蹤歷史：只寫 2.3 和 17 兩個端點的對比，中間版本只在關鍵轉折點提一句，而且要有證據。
- 不寫第 1 章的環境建置（模擬器、LineageOS、TWRP、刷機），最多一頁「如何取得 / 閱讀 17 原始碼」。
- 不碰 vendor / SoC 實作，只用 AOSP 和 GKI 公開的程式碼。
- 不新增原書沒有的主題（SurfaceFlinger、Camera、Telephony、Mainline 等），除非某章的情景確實需要。
- 不貼大段、與情景無關的程式碼：只貼呼叫鏈上用得到的部分，其餘用 `...` 省略，並附完整路徑。
- 不大量照抄原書文字：觀念說明用自己的話重寫並標注頁碼。

## 5. Done-when（完成條件與檢查點）

**輸出格式**
- 繁體中文 Markdown，放在 `aosp-note/` 下：
  ```
  aosp-note/
    README.md            ← 總目錄、版本基準、進度表
    chXX-<topic>/
      00-outline.md      ← 本章大綱
      X.Y-<section>.md
  ```
- 每一小節開頭標注：情景說明、對應原書章節/頁碼、涉及的專案與 tag。
- 每個 Step 的格式：`專案/路徑::函式名` → 程式碼片段 → 解說 → **〔與 2.3 差異〕**（不變 / 改名搬家 / 邏輯改變 / 已移除 / 新增）。
- 每一節結尾附一張 Mermaid 時序圖，總結整條呼叫鏈。

**範圍與順序**
1. **第 5 章 Binder** 試寫。
2. 第 2、4、6 章改寫成新機制：AIDL HAL、logd + liblog、memfd / 現行 ashmem，章名跟著換掉。
3. 其餘章節（3、7–16）依原書順序。
4. 第 1 章跳過，或只寫一頁「如何取得 / 閱讀 17 原始碼」。

**每一章的節奏**
1. **檢查點 1：大綱**（`00-outline.md`）。內容包括小節清單、每節的情景、呼叫鏈的起點和終點、與原書小節的對應，以及使用的 kernel 分支。**確認之後才開始寫正文。**
2. **檢查點 2：第一個小節**。確認深度、篇幅、格式之後，再寫全章。
3. **全章完成**：沒有未標記的猜測，所有程式碼片段都可以在 17 的原始碼中找到。

**目前進度**：見 [progress.md](progress.md)。
