# 如何在線上讀取 AOSP 原始碼（命令列 / 腳本）

## 結論

- **cs.android.com 無法直接抓取**：它是靠 JavaScript 渲染的網頁應用程式，用 curl 抓回來只有 JS 載入程式和 `This site requires javascript.`，沒有程式碼內容。
- **改用 android.googlesource.com（Gitiles）**：cs.android.com 上的 AOSP 程式碼都在這裡，加上 `?format=TEXT` 或 `?format=JSON` 就能用 curl 取得。

## 1. 讀取單一檔案

加上 `?format=TEXT` 會回傳 **base64 編碼**的原始檔，用 `base64 -d` 解碼：

```bash
curl -s "https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/Activity.java?format=TEXT" | base64 -d
```

## 2. 列出目錄內容

加上 `?format=JSON`。回應開頭有 `)]}'` 防 XSSI 前綴，要先去掉第一行：

```bash
curl -s "https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/?format=JSON" \
  | tail -n +2 | jq -r '.entries[] | "\(.type)\t\(.name)"'
```

## 3. 從 cs.android.com 網址換成 googlesource 網址

| 部分 | cs.android.com | android.googlesource.com |
|---|---|---|
| 範例 | `https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/app/Activity.java` | `https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/Activity.java?format=TEXT` |
| 專案 | 路徑的前段（`frameworks/base`）在 superproject 底下 | 獨立的 git 專案：`platform/frameworks/base` |
| 分支 | `+/main:` | `+/refs/heads/main/` |
| 檔案路徑 | 專案內剩下的路徑 | 相同 |

換算方法：
1. 找出檔案屬於哪個 git 專案（例如 `frameworks/base`、`system/core`、`packages/apps/Settings`）。專案清單可查 manifest：
   `https://android.googlesource.com/platform/manifest/+/refs/heads/main/default.xml?format=TEXT`
2. 專案名稱前面加上 `platform/`（以 manifest 裡的 `name` 屬性為準，有些專案不在 `platform/` 底下，例如 `kernel/...`、`device/...`）。
3. 分支用 `refs/heads/<branch>`，tag 用 `refs/tags/<tag>`（例如 `refs/tags/android-15.0.0_r1`），也可以直接放 commit hash。

## 4. 限制

- **沒有全文搜尋和交叉參照**：Gitiles 只能瀏覽目錄、讀檔案。要搜尋符號，請在本機 `repo sync` 或只 clone 需要的專案後用 `grep` / `rg`：
  ```bash
  git clone --depth 1 https://android.googlesource.com/platform/frameworks/base
  ```
- **commit 歷史（`+log`）要登入**：匿名用 curl 存取 `+log/...` 會得到 `403 Forbidden: Please sign in to view the history pages.` 要看歷史，請 clone 後用 `git log`。
- **TEXT 格式一定是 base64**，忘了解碼會看到一大串亂碼。
