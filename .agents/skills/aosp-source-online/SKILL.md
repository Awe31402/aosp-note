---
name: aosp-source-online
description: >-
  Fetch and browse Android Open Source Project (AOSP) source code files and
  directory listings online via android.googlesource.com (Gitiles) using curl,
  or convert cs.android.com URLs to Gitiles endpoints. Use this skill whenever
  you need to read AOSP source files, check directory contents, or inspect
  Android framework/system source code without a local AOSP checkout.
---

# Online AOSP Source Code Reader

Do **not** fetch `cs.android.com` directly with `curl`—it is a client-side JavaScript app that only returns a loader stub (`This site requires javascript.`). Always query `android.googlesource.com` (Gitiles) instead.

## 1. Read a Single Source File

Append `?format=TEXT` to the Gitiles file URL. The response is **base64-encoded**, so pipe it through `base64 -d`:

```bash
curl -s "https://android.googlesource.com/<project>/+/<ref>/<path>?format=TEXT" | base64 -d
```

Example (`Activity.java` on `main`):
```bash
curl -s "https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/Activity.java?format=TEXT" | base64 -d
```

## 2. List Directory Contents

Append `?format=JSON` to the Gitiles directory URL (keep the trailing `/`). The response starts with a 4-character XSSI protection prefix `)]}'` on the first line—strip it with `tail -n +2` before parsing with `jq`:

```bash
curl -s "https://android.googlesource.com/<project>/+/<ref>/<dir>/?format=JSON" \
  | tail -n +2 | jq -r '.entries[] | "\(.type)\t\(.name)"'
```

Example (`core/java/android/app/`):
```bash
curl -s "https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/?format=JSON" \
  | tail -n +2 | jq -r '.entries[] | "\(.type)\t\(.name)"'
```

## 3. Convert `cs.android.com` URLs to Gitiles URLs

| Component | `cs.android.com` | `android.googlesource.com` |
|---|---|---|
| Example | `https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/core/java/android/app/Activity.java` | `https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/core/java/android/app/Activity.java?format=TEXT` |
| Project | Path prefix under superproject (`frameworks/base`) | Separate Git repository (`platform/frameworks/base`) |
| Branch / Ref | `+/main:` | `+/refs/heads/main/` (or `+/refs/tags/<tag>/` or `+<commit>/`) |
| File Path | Remaining path inside the project | Same remaining path |

### Steps to Resolve the Git Project Name
1. Identify which Git repository owns the path (e.g., `frameworks/base`, `system/core`, `packages/apps/Settings`).
2. Most platform repositories are prefixed with `platform/` (e.g. `platform/frameworks/base`), while others sit at top-level paths (`kernel/...`, `device/...`, `toolchain/...`).
3. If unsure of the exact repository boundary or `name` attribute, query the AOSP manifest:
   ```bash
   curl -s "https://android.googlesource.com/platform/manifest/+/refs/heads/main/default.xml?format=TEXT" | base64 -d
   ```

## 4. Known Limitations & Workarounds

- **No full-text search or cross-references**: Gitiles only supports file reads and directory traversal. If you need symbol search across a repository, shallow-clone the specific repo and search locally with `rg` / `grep`:
  ```bash
  git clone --depth 1 https://android.googlesource.com/platform/frameworks/base
  ```
- **Commit history (`+log`) requires authentication**: Anonymous `curl` requests to `+log/...` return `403 Forbidden: Please sign in to view the history pages.` To inspect commit history, shallow-clone with `--depth N` and run `git log`.
- **Always decode `format=TEXT`**: Forgetting `| base64 -d` produces base64 text instead of source code.
