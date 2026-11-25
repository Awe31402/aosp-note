# 第 3 章：智能指標（`LightRefBase` / `RefBase` / `sp` / `wp`）— 大綱

> **原書對應**：第 3 章〈智能指针〉（p.57–79）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：本章為純使用者空間 C++ 基礎函式庫（`libutils`），不涉及 Linux 核心驅動
> **撰寫日期**：2026-11-25
> **狀態**：✅ 全章完成（`3.1`–`3.4` 共 4 節、34 個 Step、61 段程式碼、1,320 行逐行比對驗證 0 筆不符、0 筆 `TODO: 未確認`、6 張 PlantUML 圖表皆已編譯並目視檢查通過）

---

## 一、原始碼專案清單與版本驗證

本章所需專案皆已於本機 `src/` 檢出並驗證對應版本：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 3 章涵蓋範圍 |
|---|---|---|---|
| `platform/system/core` | `android-17.0.0_r1` | `545d248` | `libutils/Android.bp`、`libutils/include/utils/{LightRefBase.h,StrongPointer.h,RefBase.h,NativeHandle.h}`（前 3 檔為指向 `binder/include/utils/` 之 symlink）、`libutils/{LightRefBase.cpp,NativeHandle.cpp}`、`libutils/binder/Android.bp`、`libutils/binder/include/utils/{LightRefBase.h,StrongPointer.h,RefBase.h,AllocatorTracker.h}`、`libutils/binder/{RefBase.cpp,StrongPointer.cpp,RefBase_test.cpp,StrongPointer_test.cpp}` |
| `platform/frameworks/native` | `android-17.0.0_r1` | `ae266dc` | `libs/binder/{BpBinder.cpp,ProcessState.cpp,Android.bp}`（`OBJECT_LIFETIME_WEAK`、`attemptIncWeak`、`sp::force_set`、`ANDROID_UTILS_REF_BASE_DISABLE_IMPLICIT_CONSTRUCTION` 實戰對照） |
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163b` | `core/jni/android_util_Binder.cpp`（`BinderProxyNativeData` 使用 RBSAN `tryTag(OBJECT_TAG_JAVA_PROXY)` / `untag`）、`core/jni/com_android_internal_util_VirtualRefBasePtr.cpp` 與 `core/java/com/android/internal/util/VirtualRefBasePtr.java`（`VirtualLightRefBase` 跨 JNI 用例） |

> **重大架構演進說明（2.3 → 17）**：
> 1. **模組拆分與搬家**：原書 2.3 中 `LightRefBase`、`RefBase`、`sp`、`wp` 全塞在 `frameworks/base/include/utils/RefBase.h` 與 `frameworks/base/libs/utils/RefBase.cpp`；Android 17 已遷至 `platform/system/core/libutils/binder/`（由 `libutils_binder` 靜態併入 `libutils`，並拆為 `LightRefBase.h`、`StrongPointer.h`、`RefBase.h`、`AllocatorTracker.h` 四個標頭檔與各自 `.cpp`）。
> 2. **現代 C++ 原子記憶體序與移動語意**：全面以 C++11 `std::atomic<int32_t>` 取代舊 `android_atomic_inc/dec/cmpxchg`；嚴格採用 **relaxed 遞增 + release 遞減 + 最終釋放前 `atomic_thread_fence(acquire)`** 的 release-sequence 協議；`sp<T>` 新增移動建構／移動賦值（零引用計數開銷）與 `release()` 釋放所有權。
> 3. **反雙重擁有與編譯／執行期安全網**：
>    - 新增 `sp<T>::make(...)`、`sp<T>::makeNoThrow(...)`、`sp<T>::fromExisting(T*)`、`wp<T>::fromExisting(T*)` 與 `ANDROID_UTILS_REF_BASE_DISABLE_IMPLICIT_CONSTRUCTION` 巨集（`libutils` 與 `libbinder` 預設啟用），根除裸指標隱式轉 `sp`/`wp` 造成的 double-ownership；
>    - 在 `RefBase.h` 特化 `std::make_shared` / `std::make_unique` 並以 `static_assert` 編譯期封鎖混用；
>    - 執行期新增 `check_not_on_stack()`（棧物件偵測）、`BAD_STRONG`/`BAD_WEAK`（下溢／溢位偵測）、`sp_report_race()`（volatile 雙讀資料競爭啟發偵測）、`~RefBase()` 強計數非零 fatal 檢查，以及 **RBSAN**（`OBJECT_TAG_JAVA_PROXY` 標籤防護，防止物件仍在外部引用時遭銷毀）。
> 4. **生命週期語意收斂**：原書的 **`OBJECT_LIFETIME_FOREVER` 已完全移除**（只保留 `OBJECT_LIFETIME_STRONG` 與 `OBJECT_LIFETIME_WEAK`）；且對預設 `OBJECT_LIFETIME_STRONG` 物件，若從未建立強引用（`mStrong == INITIAL_STRONG_VALUE`），最後一個弱引用歸零時**不再自動 `delete impl->mBase`**（改印 `ALOGW` 警告，避免在建構子內暫時建立 `wp(this)` 導致提前析構）。
> 5. **可插拔配置器（Android 17 新增）**：`AllocatorTracker.h`（Copyright 2025）透過 `ANDROID_UTILS_CUSTOM_ALLOCATOR` 與 `ANDROID_NEW` / `ANDROID_NEW_NOTHROW` / `ANDROID_DELETE` 巨集支援自訂配置器（`libutils_binder_custom_allocator`），使 `sp::make` 與 `RefBase`/`weakref_impl` 的配置／釋放皆可被攔截管理。

---

## 二、小節規劃與原書對照

本章保留原書「輕量級指標 → 強指標 → 弱指標 → 實戰與驗證」的漸進式結構，規劃為 **4 個小節**（圖表皆以 PlantUML 繪製，存於 `ch03-smart-pointer/figs/`）：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `3.1-light-pointer.md` | 3.1 輕量級指標（`LightRefBase`、`VirtualLightRefBase` 與 `sp<>` 基礎） | 3.1.1 + 3.1.2（p.58–62） | **合併重寫並擴充**：原書 3.1.1 僅看 `LightRefBase` 與 `sp` 建構/解構、3.1.2 為自寫 `lightpointer.cpp`。17 合併為一節，深入 `LightRefBase<T>`（CRTP、`std::atomic` release/acquire fence、`incStrongRequireStrong`）、新增的 `VirtualLightRefBase`、`sp<T>` 完整所有權操作（`make`/`makeNoThrow`/`fromExisting`/move/`release`/operator=`sp_report_race`），並以 AOSP 17 真實類別 `NativeHandle`、`VirtualRefBasePtr` 及官方 `StrongPointer_test.cpp` 取代原書自寫範例 |
| `3.2-strong-pointer.md` | 3.2 強指標與 `RefBase` 生命週期（`RefBase`、`weakref_impl` 與記憶體序） | 3.2.1（p.63–69） | **保留並深度重寫**：完整追蹤 `RefBase`、`weakref_type`、`weakref_impl` 的記憶體布局與兩段式銷毀（物件本體 vs `weakref_impl`），涵蓋 `incStrong`（`INITIAL_STRONG_VALUE`、`check_not_on_stack`、`onFirstRef`）、`incStrongRequireStrong`、`forceIncStrong`、`decStrong`（release + acquire fence、`onLastStrongRef`、`ANDROID_DELETE`）、`~RefBase`（RBSAN 與 double-ownership 檢核）、`decWeak`（移除 `OBJECT_LIFETIME_FOREVER` 與 `INITIAL_STRONG_VALUE` 不再 `delete mBase` 的關鍵變化）以及 C++11 原子記憶體序與 `std::make_shared`/`make_unique` 編譯期封鎖 |
| `3.3-weak-pointer.md` | 3.3 弱指標與安全升級（`wp<>`、`promote`、`attemptIncStrong` 與 ABA 防護） | 3.2.2（p.69–75） | **保留並深度重寫**：追蹤 `wp<T>` 雙指標結構與三大實作不變量、`createWeak` / `wp::fromExisting`（`incWeakRequireWeak`）與 `~wp`；詳解 `wp::promote()` 如何改用 `result.set_pointer` + `attemptIncStrong(&result)`，逐分支剖析 `attemptIncStrong`（fast-path CAS、slow-path `OBJECT_LIFETIME_STRONG` 下的 `INITIAL_STRONG_VALUE` CAS 競態防禦 vs `OBJECT_LIFETIME_WEAK` 下的 `onIncStrongAttempted` 復活與並發 `onLastStrongRef` 補償）、`attemptIncWeak`，以及 `wp` 比較運算子如何以 `m_refs` 解決記憶體地址重用（ABA）問題 |
| `3.4-practice-and-safety.md` | 3.4 實戰情景、安全防線與測試驗證（`RefBase_test`、Binder 整合、RBSAN 與自訂配置器） | 3.2.3（p.75–79） | **大幅改寫為現代實戰節**：以 AOSP 17 官方 `RefBase_test.cpp`（`RefBaseBothLifetimes` 參數化對照、`ReplacedComparison`、`DoubleOwnershipDeath`、`StackOwnershipDeath`、`RacingDestructors` / `RacingPromotions`）取代原書自寫 `weightpointer.cpp`（原 `ForeverClass` 已不復存在）；並串接框架真實情景：`ProcessState` + `BpBinder` 的 `OBJECT_LIFETIME_WEAK` + `attemptIncWeak` + `force_set`、`BinderProxyNativeData` 的 RBSAN（`tryTag`/`untag`）、`DEBUG_REFS` + `ReferenceMover`（`renameRefId`）與 Android 17 `AllocatorTracker` |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 3.1 輕量級指標（`3.1-light-pointer.md`）
- **情景**：
  1. 呼叫端透過 `NativeHandle::create(handle, ownsHandle)` 以 `sp<NativeHandle>::make(...)` 建立 `LightRefBase` 衍生對象，經複製、移動（`std::move`）、`fromExisting` 與重新賦值後離開作用域，觀察計數歸零並安全析構底層 `native_handle_t`；
  2. 對照 Java 層 `VirtualRefBasePtr` 透過 JNI 直接對 `VirtualLightRefBase` 執行 `incStrong` / `decStrong`。
- **起點 → 終點**：
  - `NativeHandle::create` → `sp<T>::make`（`ANDROID_NEW` → `LightRefBase::LightRefBase()` 設 `mCount=0` → `LightRefBase::incStrong` `fetch_add(1, relaxed)`）→ `sp(const sp<T>&)` / `sp(sp<T>&&)` / `sp<T>::fromExisting`（`LightRefBase::incStrongRequireStrong`，若原值為 0 呼叫 `LightRefBase_reportIncStrongRequireStrongFailed` 中止）→ `sp::operator=`（volatile 雙讀 `m_ptr` + `sp_report_race()`）→ `sp::~sp()` → `LightRefBase::decStrong`（`fetch_sub(1, release) == 1` → `atomic_thread_fence(acquire)` → `delete static_cast<const T*>(this)` → `~NativeHandle()`）。
  - 另附 `VirtualRefBasePtr`（`nIncStrong`/`nDecStrong` → `VirtualLightRefBase` 虛解構子）與 `StrongPointer_test.cpp`（`move` / `Deleted` / `AssertStrongRefExists` / `release`）。

### 3.2 強指標與 `RefBase` 生命週期（`3.2-strong-pointer.md`）
- **情景**：呼叫端透過 `sp<Foo>::make(...)` 建立繼承 `RefBase` 的物件，歷經首次強引用初始化（`onFirstRef`）、多執行緒並發複製與釋放、`sp` 清空歸零觸發 `onLastStrongRef`，對比「預設強生命週期（`OBJECT_LIFETIME_STRONG`）」與「擴展弱生命週期（`OBJECT_LIFETIME_WEAK`）」下物件本體與 `weakref_impl` 兩階段銷毀的完整流程。
- **起點 → 終點**：
  - `sp<Foo>::make` → `RefBase::RefBase()`（`mRefs = ANDROID_NEW(weakref_impl, this)`，`mStrong = INITIAL_STRONG_VALUE`、`mWeak = 0`、`mFlags = OBJECT_LIFETIME_STRONG`）→ `RefBase::incStrong`（`mRefs->incWeak` → `mStrong.fetch_add(1, relaxed)` → `BAD_STRONG` 檢核 → `check_not_on_stack(this)` → `mStrong.fetch_sub(INITIAL_STRONG_VALUE, relaxed)` → `onFirstRef()`）
  - 釋放路徑：`sp::~sp()` → `RefBase::decStrong`（`mStrong.fetch_sub(1, release)` → `BAD_STRONG` 檢核 → 若 `c == 1`：`atomic_thread_fence(acquire)` → `onLastStrongRef(id)` → 若 `OBJECT_LIFETIME_STRONG` 則 `ANDROID_DELETE(RefBase, this)` 觸發 `RefBase::~RefBase()` 檢核 RBSAN `OBJECT_TAG_MASK` 與 `strongs`）→ `refs->decWeak(id)`（`mWeak.fetch_sub(1, release)` → 若 `c == 1`：`atomic_thread_fence(acquire)` → `OBJECT_LIFETIME_STRONG` 且曾經有強引用則 `ANDROID_DELETE(weakref_impl, impl)`，從未強引用則只印 `ALOGW`；`OBJECT_LIFETIME_WEAK` 則 `onLastWeakRef` + `ANDROID_DELETE(RefBase, impl->mBase)` → `~RefBase()` 釋放 `mRefs`）。

### 3.3 弱指標與安全升級（`3.3-weak-pointer.md`）
- **情景**：物件 A 持有物件 B 的 `sp<B>`，物件 B（或快取表）以 `wp<A>` 回指物件 A；當物件 B 需要存取 A 時呼叫 `wp<A>::promote()`，依序分析「仍有強引用（fast path）」、「從未強引用（`INITIAL_STRONG_VALUE` 競態 CAS）」、「強引用已歸零（`OBJECT_LIFETIME_STRONG` 升級失敗 vs `OBJECT_LIFETIME_WEAK` 呼叫 `onIncStrongAttempted` 復活與並發補償）」四種狀態，並驗證物件銷毀後同地址重用時 `wp == wp` / `wp == sp` 的比較行為。
- **起點 → 終點**：
  - 建立與釋放：`wp<T>(const sp<T>&)` → `RefBase::createWeak` → `weakref_type::incWeak`（或 `wp<T>::fromExisting` → `getWeakRefs()` → `incWeakRequireWeak`）；`wp::~wp()` → `weakref_type::decWeak`。
  - 升級：`wp<T>::promote()` → 建立空 `sp<T> result` → `m_refs->attemptIncStrong(&result)` → `incWeak` → fast-path `compare_exchange_weak` 迴圈 → slow-path 分流（`OBJECT_LIFETIME_STRONG` 檢查 `curCount <= 0` 或 CAS 遞增 `INITIAL_STRONG_VALUE`；`OBJECT_LIFETIME_WEAK` 呼叫 `onIncStrongAttempted(FIRST_INC_STRONG, id)` → `fetch_add(1)` → 若 `curCount != 0 && curCount != INITIAL_STRONG_VALUE` 補呼 `onLastStrongRef`）→ 若 `curCount == INITIAL_STRONG_VALUE` 扣除 `INITIAL_STRONG_VALUE`（注意：17 此處已**不再**呼叫 `onFirstRef()`，須對照原書差異！）→ `result.set_pointer(m_ptr)`。
  - 比較與弱升級：`wp::operator==` / `<`（優先比較 `m_refs` 與雙層鍵 `(m_ptr, m_refs)`）與 `weakref_type::attemptIncWeak`（CAS `mWeak > 0`）。

### 3.4 實戰情景、安全防線與測試驗證（`3.4-practice-and-safety.md`）
- **情景**：
  1. 沿著 `RefBase_test.cpp` 驗證 `OBJECT_LIFETIME_STRONG` / `OBJECT_LIFETIME_WEAK` 生命週期差異、地址重用比較、雙重擁有與棧物件 fatal 攔截，以及雙核並發 `RacingDestructors` / `RacingPromotions` 如何依賴 release-acquire 記憶體序；
  2. 沿著 Binder 與 JNI 真實呼叫鏈驗證高階機制：`ProcessState::getStrongProxyForHandle`（`attemptIncWeak` + `sp::force_set` + `BpBinder` `OBJECT_LIFETIME_WEAK`）、`BinderProxyNativeData` 的 RBSAN（`tryTag` / `untag`）、`DEBUG_REFS`（`trackMe` / `printRefs` / `ReferenceMover::move_references`）與 `AllocatorTracker`（`ANDROID_UTILS_CUSTOM_ALLOCATOR`）。
- **起點 → 終點**：
  - 測試驗證：`RefBaseBothLifetimes::{StrongMoves,NoStrongCountPromoteFromWeak,TagUntag,DieIfDeletedWithTag}`、`ReplacedComparison`、`DoubleOwnershipDeath`、`StackOwnershipDeath`、`RacingDestructors`、`RacingPromotions`、`CustomAllocator`。
  - 框架實戰：`ProcessState::getStrongProxyForHandle` → `e->refs->attemptIncWeak` → `result.force_set(b)`（`RefBase::forceIncStrong`）→ `BpBinder::onFirstRef` / `onLastStrongRef` / `~BpBinder`；以及 `javaObjectForIBinder` → `BinderProxyNativeData::tryTagObject` → `weakref_type::tryTag(OBJECT_TAG_JAVA_PROXY)` → `~BinderProxyNativeData` → `untag`（若未 `untag` 即銷毀則於 `RefBase::~RefBase` 觸發 `RBSAN: Object is deleted, but it is tagged`）。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

| 原書 2.3 符號 / 路徑 | Android 17 符號 / 路徑 | 變化類型 | 說明 |
|---|---|---|---|
| `frameworks/base/include/utils/RefBase.h`（單檔含 `LightRefBase`、`sp`、`RefBase`、`wp`） | `platform/system/core/libutils/binder/include/utils/{LightRefBase.h,StrongPointer.h,RefBase.h,AllocatorTracker.h}`（`libutils/include/utils/` 保留 symlink） | 搬家 + 拆分 | 抽離至 `system/core/libutils/binder/`（`libutils_binder`），拆分為 4 個標頭檔；`RefBase.h` 仍 `#include` `LightRefBase.h` 與 `StrongPointer.h` 維持相容 |
| `frameworks/base/libs/utils/RefBase.cpp` | `platform/system/core/libutils/binder/RefBase.cpp` + `StrongPointer.cpp` + `libutils/LightRefBase.cpp` | 搬家 + 拆分 | 錯誤報告函式抽離至 `StrongPointer.cpp`（`sp_report_race`）與 `LightRefBase.cpp`（`LightRefBase_reportIncStrongRequireStrongFailed`）減少內聯程式碼膨脹 |
| `volatile int32_t mCount` / `mStrong` / `mWeak` + `android_atomic_inc/dec/cmpxchg` | `std::atomic<int32_t>` + `fetch_add(relaxed)` / `fetch_sub(release)` + `atomic_thread_fence(acquire)` + `compare_exchange_weak` | 邏輯改變 | 改用標準 C++11 原子操作與精細記憶體序（relaxed 遞增、release 遞減、最終釋放前 acquire fence），建立嚴格 happens-before 保證 |
| `LightRefBase<T>` 僅含 `incStrong`/`decStrong` | 新增 `incStrongRequireStrong` 與 `VirtualLightRefBase`（非模板、帶虛擬解構子） | 新增 | 支援 `sp::fromExisting` 執行期檢核，並提供跨 JNI（`VirtualRefBasePtr`）使用的多型輕量級基底類 |
| `sp<T>` 僅支援裸指標與複製建構／賦值 | 新增 `sp<T>::make` / `makeNoThrow` / `fromExisting` / `cast` / `release`、移動建構／賦值、explicit `operator bool`、`sp_report_race()`、`ANDROID_UTILS_REF_BASE_DISABLE_IMPLICIT_CONSTRUCTION` | 新增 + 邏輯改變 | 推動以 `sp::make` 取代 `sp<T> = new T()`，封鎖隱式裸指標轉換與雙重擁有，並以移動語意免除無謂原子計數加減 |
| `RefBase::OBJECT_LIFETIME_FOREVER = 0x0003` | 已移除（僅剩 `OBJECT_LIFETIME_STRONG = 0x0000`、`OBJECT_LIFETIME_WEAK = 0x0001`） | 已移除 | 現代 `RefBase` 嚴禁手動管理生命週期；新增 `OBJECT_TAG_JAVA_PROXY = 0x10000`（RBSAN 位元遮罩）與 `mFlags` 共用欄位 |
| `weakref_type::decWeak` 在 `OBJECT_LIFETIME_STRONG` 且 `mStrong == INITIAL_STRONG_VALUE` 時執行 `delete impl->mBase` | 改為僅輸出 `ALOGW` 警告，**不再 `delete impl->mBase`**（`RefBase.cpp:606-616`） | 邏輯改變 | 避免在建構子內暫時建立 `wp(this)` 後提前銷毀尚未擁有強引用的物件 |
| `sp<T>::sp(T* p, weakref_type* refs)`（供 `wp::promote` 使用） | 已移除；`wp::promote()` 改建空 `sp<T> result`，呼叫 `m_refs->attemptIncStrong(&result)` 後 `result.set_pointer(m_ptr)` | 邏輯改變 | 簡化介面，並把 `result` 位址直接作為 `id` 傳入引用追蹤 |
| `attemptIncStrong` 中 `curCount == INITIAL_STRONG_VALUE` 時呼叫 `onFirstRef()`，且 `OBJECT_LIFETIME_STRONG` 未對 `INITIAL_STRONG_VALUE` 做 CAS 迴圈 | 17 對 `OBJECT_LIFETIME_STRONG` 的 `INITIAL_STRONG_VALUE` 加上 CAS 迴圈防禦並發析構；且扣除 `INITIAL_STRONG_VALUE` 後**不再呼叫 `onFirstRef()`**（`RefBase.cpp:668-724`） | 邏輯改變 | 修正多執行緒 `promote()` 與 `incStrong`/`decStrong` 競態，同時避免未預期於 `promote()` 觸發 `onFirstRef()` |
| `RefBase::~RefBase()` 僅檢查 `if (mRefs->mWeak == 0) delete mRefs;` | 依 `OBJECT_LIFETIME_MASK` 分流：先檢核 RBSAN `OBJECT_TAG_MASK`（非零 fatal）；`WEAK` 時若 `mWeak == 0` 釋放 `mRefs`；`STRONG` 時若 `mStrong == INITIAL_STRONG_VALUE` 印 `ALOGW` + CallStack，若 `mStrong != 0` 直接 `LOG_ALWAYS_FATAL`（Double owned） | 邏輯改變 + 新增 | 將 `weakref_impl` 釋放職責明確分離，並補上雙重擁有與 RBSAN 偵測 |
| `wp::operator==` 跨型別僅比 `m_ptr == o.m_ptr`（且無 `wp == sp` 安全比對） | `wp == wp` 一律比對 `m_refs == o.m_refs`；`wp == sp` 比對 `m_refs == o->getWeakRefs()`；`<`/`>` 先比 `m_ptr` 再以 `m_refs` 打破平手 | 邏輯改變 | 解決舊物件釋放後同地址配置新物件的 ABA 誤判問題，使 `wp` 可安全作為快取鍵 |
| 無對應機制（2.3 僅以 `new`/`delete` 直接配置） | `AllocatorTracker.h`（`ANDROID_UTILS_CUSTOM_ALLOCATOR`、`ANDROID_NEW`、`ANDROID_DELETE`）與 `std::make_shared`/`make_unique` `static_assert` 封鎖 | 新增 | 支援自訂配置器並於編譯期阻止把 `RefBase` 交給 STL 智能指標管理 |
| 原書自寫 `external/lightpointer`、`external/weightpointer` + `Android.mk` | `libutils/binder/{StrongPointer_test.cpp,RefBase_test.cpp}`、`libutils/NativeHandle.cpp`、`VirtualRefBasePtr` + `Android.bp`（Soong） | 取代 | 以 AOSP 17 內建單元測試與系統真實程式碼作為情境驗證依據 |
| —（無已知未確認項目） | — | — | 本章所有檔案路徑、函式簽名與行號皆已於 `android-17.0.0_r1` 實機驗證，0 筆 `TODO: 未確認` |
