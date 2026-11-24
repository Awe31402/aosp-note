# 第 2 章：AIDL HAL（硬體抽象層）— 大綱

> **原書對應**：第 2 章〈硬件抽象层〉（p.21–56）
> **AOSP 基準**：`refs/tags/android-17.0.0_r1`
> **Kernel 基準**：GKI `kernel/common` 分支 `android17-6.18`（Linux 6.18，檢出 commit `1e0de07`）
> **撰寫日期**：2026-11-24
> **狀態**：✅ 全章完成（`2.1-kernel-driver.md`–`2.5-app-client.md` 共 5 節，全數完成並通過逐行原始碼驗證）

---

## 一、原始碼專案清單與版本驗證

以下專案皆已對照 `android-17.0.0_r1` tag 與 GKI `android17-6.18` 分支實際檢出確認：

| 專案（Gitiles） | Tag / 分支 | Commit（短碼） | 第 2 章涵蓋範圍 |
|---|---|---|---|
| `kernel/common` | `android17-6.18`（GKI Linux 6.18） | `1e0de07` | `drivers/leds/{led-class.c,led-core.c,Kconfig,Makefile}`、`include/linux/leds.h` |
| `platform/hardware/interfaces` | `android-17.0.0_r1` | `0162af6` | `light/aidl/`（`ILights.aidl` 等介面、`default/` Rust 範例實作、`vts/functional/VtsHalLightTargetTest.cpp`）、`vibrator/aidl/`（`IVibrator.aidl`、`default/` C++ `libbinder_ndk` 範例實作與 Vendor APEX、`example_java_client/`） |
| `platform/hardware/libhardware` | `android-17.0.0_r1` | `97ee3e3` | `hardware.c`、`include/hardware/hardware.h`（保留對照 2.3 傳統 `hw_get_module` / `hw_module_t` 結構） |
| `platform/frameworks/native` | `android-17.0.0_r1` | `ae266dc` | `cmds/servicemanager/{ServiceManager.cpp,Access.cpp}`（VINTF 宣告與 SELinux 校驗）、`libs/binder/ndk/{service_manager.cpp,process.cpp,ibinder.cpp,stability.cpp}`、`libs/binder/Stability.cpp`、`services/vibratorservice/{VibratorHalController.cpp,VibratorHalWrapper.cpp,VibratorManagerHalController.cpp}` |
| `platform/frameworks/base` | `android-17.0.0_r1` | `94b4c163` | `services/java/com/android/server/SystemServer.java`、`services/core/java/com/android/server/lights/LightsService.java`、`services/core/java/com/android/server/vibrator/{VibratorManagerService.java,HalVibrator.java}`、`services/core/jni/{com_android_server_vibrator_VibratorController.cpp,com_android_server_vibrator_VibratorManagerService.cpp,onload.cpp}`、`core/java/android/hardware/lights/{LightsManager.java,SystemLightsManager.java,ILightsManager.aidl,LightsRequest.java}`、`core/java/android/app/SystemServiceRegistry.java` |

> **關於範例選取的說明**：
> 原書第 2 章以自建虛擬字元裝置 `freg` 貫穿「核心驅動 → C 驗證程式 → `libhardware` HAL `.so` → `SystemServer` 服務與 JNI → App」五層。在 Android 17 中，Project Treble 與 Stable AIDL HAL 已徹底取代同行程 `.so` 的 `libhardware`，且不同硬體子系統展現出兩種現代標準路徑：
> - **主線範例：`Lights`（`android.hardware.light.ILights`）**——從 Linux LED 子系統（`drivers/leds/led-class.c` `/sys/class/leds/`）→ VTS 驗證（`VtsHalLightTargetTest.cpp`）→ Stable AIDL HAL 介面與 Rust 範例服務（`light/aidl/`）→ `SystemServer` **Java 直連 AIDL HAL（零 JNI）**（`LightsService.java`）→ 框架 Manager 與 App（`SystemLightsManager` → `ILightsManager` → `LightsService` → `ILights`），結構最貼近原書 `freg` 讀寫硬體狀態的直覺且涵蓋全部五層。
> - **對照範例：`Vibrator`（`android.hardware.vibrator.IVibrator`）**——補足兩項極為常見的現代實務形態：**C++ `libbinder_ndk` HAL 服務實作與 Vendor APEX 打包**（`vibrator/aidl/default/`），以及 **SystemServer 經由 JNI + C++ `VibratorHalController` 呼叫 AIDL HAL**（`com_android_server_vibrator_VibratorController.cpp`），直接對照原書 2.4.3 的 JNI 層。

---

## 二、小節規劃與原書對照

本章維持原書由下而上貫穿五層的 5 大節骨架，全面改寫為 Android 17 的 GKI + Stable AIDL HAL 現代機制：

| 新檔名 | 小節標題 | 原書對應（頁碼） | 處理方式與理由 |
|---|---|---|---|
| `2.1-kernel-driver.md` | 2.1 核心驅動與 GKI 邊界：以 Linux LED 子系統為例 | 2.1（p.22–31）與 2.3.4（p.44–45） | **改寫**：原書直接修改內核編譯自建字元驅動 `freg`（devfs/procfs/sysfs）並以 `ueventd.rc` 開放權限；17 採 GKI + Vendor Kernel Module 架構，改追蹤核心標準 LED 類別驅動 `drivers/leds/led-class.c` / `led-core.c` 如何註冊 `/sys/class/leds/<name>/brightness` 並由 sysfs 節點對接 HAL，同時對比 `ueventd.rc` + SELinux 權限隔離 |
| `2.2-vts-validation.md` | 2.2 驗證 AIDL HAL：VTS 功能測試 | 2.2（p.32–33） | **改寫**：原書寫 C 程式直開 `/dev/freg` 驗證；17 在 Treble 與 SELinux 隔離下改由 VTS（Vendor Test Suite）對外驗證 HAL 契約。追蹤 `VtsHalLightTargetTest.cpp` 如何透過 `getAidlHalInstanceNames` 發現 VINTF 實例、`waitForDeclaredService<ILights>` 取得代理、執行 `getLights`/`setLightState`/`setLightEffects` 並檢查介面版本與異常碼 |
| `2.3-aidl-hal-service.md` | 2.3 AIDL HAL 介面、服務實作、VINTF 與載入驗證 | 2.3（p.34–45） | **全面重寫**：對照仍殘留於 17 的 `hardware/libhardware`（`hw_module_t` / `hw_get_module` `dlopen`），完整剖析現代 Stable AIDL HAL：介面定義（`@VintfStability`、`frozen: true`、`aidl_api/`）、雙語言實作與啟動（C++ `libbinder_ndk` `vibrator/aidl/default/` + Vendor APEX 與 Rust `libbinder_rs` `light/aidl/default/`）、以及取代 `hw_get_module` 的 **VINTF 宣告 + `ServiceManager::addService` 雙重驗證（`isVintfDeclared` + `Access::canAdd`）與 `libbinder_ndk` 橋接** |
| `2.4-system-service.md` | 2.4 系統服務層存取 AIDL HAL：Java 直連與 JNI 雙路徑 | 2.4（p.46–51） | **全面重寫**：追蹤 `SystemServer.java` 啟動 `LightsService` 與 `VibratorManagerService`；詳解 **路徑 A（Java 零 JNI 直連）**：`LightsService` 透過 `VintfHalCache` + `ServiceManager.waitForDeclaredService` + `ILights.Stub.asInterface` + `linkToDeath` 與 `@EnforcePermission` 暴露 `ILightsManager`；並對照 **路徑 B（JNI + C++ NDK Client）**：`VibratorManagerService` → `com_android_server_vibrator_VibratorController.cpp` → `VibratorHalController` 呼叫 `IVibrator` |
| `2.5-app-client.md` | 2.5 框架 Manager 與應用程式層存取硬體服務 | 2.5（p.52–56） | **改寫**：原書 App 直接呼叫隱藏 `ServiceManager.getService("freg")`；17 追蹤標準框架路徑：`Context.getSystemService(LightsManager.class)` → `SystemServiceRegistry` 建構 `SystemLightsManager` → `ServiceManager.getServiceOrThrow(Context.LIGHTS_SERVICE)` → `openSession()`（附 `IBinder token` 死亡回收）→ `requestLights(LightsRequest)` → `LightsService.LightsManagerBinderService.setLightStates` 權限與多 Session 優先權仲裁 → `ILights.setLightState()` → Vendor HAL 行程 → 核心驅動；並簡述 `vibrator/aidl/default/example_java_client/` 示範的特權客戶端對照 |

---

## 三、各節情景與呼叫鏈（起點 → 終點）

### 2.1 核心驅動與 GKI 邊界：以 Linux LED 子系統為例（`2.1-kernel-driver.md`）
- **情景**：GKI 核心開機初始化 LED 類別子系統（`/sys/class/leds`），硬體供應商驅動模組向 LED 子系統註冊一顆實體 LED 裝置；當使用者空間 HAL 行程寫入 `/sys/class/leds/<name>/brightness` 時，核心 sysfs 回呼如何一路走到硬體驅動的亮度設定函式。
- **起點 → 終點**：`drivers/leds/led-class.c::leds_init`（註冊 `leds_class` 與 `led_groups` sysfs 屬性群組）→ 硬體模組呼叫 `led_classdev_register_ext`（建立 `/sys/class/leds/<name>` 與 `brightness`/`max_brightness`/`trigger` 節點）→ 使用者空間寫入 `brightness` → `brightness_store` → `led_set_brightness`（`led-core.c`）→ `led_set_brightness_nosleep` → `__led_set_brightness` / `set_brightness_work` → 驅動回呼 `led_cdev->brightness_set`；並對照 GKI `Kconfig`/`Makefile` 模組化邊界與 `ueventd.rc` + SELinux 權限隔離。

### 2.2 驗證 AIDL HAL：VTS 功能測試（`2.2-vts-validation.md`）
- **情景**：透過 AOSP VTS 測試執行檔 `VtsHalLightTargetTest` 在不觸碰任何 `/dev` 或 `/sys` 節點的前提下，自動發現裝置上宣告的所有 `ILights` AIDL HAL 實例並驗證其行為合約。
- **起點 → 終點**：`VtsHalLightTargetTest.cpp::main`（啟動 `ProcessState::self()->startThreadPool()`）→ `INSTANTIATE_TEST_SUITE_P` 呼叫 `android::getAidlHalInstanceNames(ILights::descriptor)`（從 VINTF 查詢所有實例）→ `LightsAidl::SetUp` 呼叫 `android::waitForDeclaredService<ILights>` → `lights->getLights(&supportedLights)` → `TEST_P(LightsAidl, TestSupported / TestLowPersistence / LightsAidlV3::TestEffects)` 呼叫 `setLightState` / `setLightEffects` 並驗證 `binder::Status`（`isOk()` / `EX_UNSUPPORTED_OPERATION` / `EX_ILLEGAL_ARGUMENT`）→ `TearDown` 復原背光。

### 2.3 AIDL HAL 介面、服務實作、VINTF 與載入驗證（`2.3-aidl-hal-service.md`）
- **情景**：對比 2.3 的 `libhardware` 同行程載入，在 Android 17 中從零串通一個獨立行程 AIDL HAL 的四個環節：凍結 Stable AIDL 介面定義 → Vendor HAL 服務端啟動註冊（C++ `libbinder_ndk` 與 Rust `libbinder_rs` 雙對照）→ `libbinder_ndk` 將服務送進 `servicemanager` → `servicemanager` 核對 VINTF manifest 與 SELinux 後放行。
- **起點 → 終點**：
  - 2.3.1 歷史對照：`hardware/libhardware/include/hardware/hardware.h`（`hw_module_t`/`hw_device_t`）與 `hardware.c::hw_get_module_by_class` → `load`（`dlopen`/`dlsym`）；
  - 2.3.2 介面定義：`hardware/interfaces/light/aidl/android/hardware/light/{ILights,HwLight,HwLightState,LightType}.aidl` 與 `Android.bp`（`stability: "vintf"`、`frozen: true`、`versions_with_info`、`aidl_api/`）；
  - 2.3.3 服務實作與啟動：
    - C++ NDK 路徑：`vibrator/aidl/default/{main.cpp,Vibrator.cpp,vibrator-default.rc,android.hardware.vibrator.xml,apex/Android.bp}`（`ABinderProcess_setThreadPoolMaxThreadCount(0)` → `ndk::SharedRefBase::make<Vibrator>()` → `AServiceManager_addService` → `ABinderProcess_joinThreadPool()`）；
    - Rust 路徑：`light/aidl/default/{main.rs,lights.rs,lights-default.rc,lights-default.xml,Android.bp}`（`BnLights::new_binder` → `binder::add_service` → `join_thread_pool`）；
  - 2.3.4 註冊與 VINTF 驗證：`libs/binder/ndk/service_manager.cpp::AServiceManager_addService` → `defaultServiceManager()->addService` → `cmds/servicemanager/ServiceManager.cpp::ServiceManager::addService` → `meetsDeclarationRequirements`（`Stability::requiresVintfDeclaration` → `isVintfDeclared` 查詢 `VintfObject::GetDeviceHalManifest()`）與 `Access::canAdd`（SELinux）。

### 2.4 系統服務層存取 AIDL HAL：Java 直連與 JNI 雙路徑（`2.4-system-service.md`）
- **情景**：開機階段 `SystemServer` 啟動系統服務並連線至位於 Vendor 行程的 AIDL HAL；比較 Java 直連（`LightsService`）與 JNI + C++ NDK（`VibratorManagerService`）兩種整合模式。
- **起點 → 終點**：
  - 2.4.1 服務啟動：`SystemServer.java::startBootstrapServices` / `startOtherServices` → `mSystemServiceManager.startService(LightsService.class)` 與 `VibratorManagerService.Lifecycle.class`；
  - 2.4.2 路徑 A（Java 直連 `ILights`）：`LightsService` 建構子 → `VintfHalCache::get`（`ServiceManager.waitForDeclaredService(ILights.DESCRIPTOR + "/default")` → `Binder.allowBlocking` → `ILights.Stub.asInterface` → `binder.linkToDeath`）→ `populateAvailableLightsFromHal`（檢查 `getInterfaceVersion() >= 3` 並呼叫 `getLights()`）→ `onStart()`（`publishLocalService` + `publishBinderService(Context.LIGHTS_SERVICE, mManagerService)`）→ `LightImpl::setLightUnchecked` 呼叫 `mVintfLights.get().setLightState(...)`；
  - 2.4.3 路徑 B（JNI + C++ NDK 呼叫 `IVibrator`）：`services/core/jni/onload.cpp` → `com_android_server_vibrator_VibratorManagerService.cpp` / `com_android_server_vibrator_VibratorController.cpp` → `frameworks/native/services/vibratorservice/{VibratorManagerHalController.cpp,VibratorHalController.cpp,VibratorHalWrapper.cpp}`（`AServiceManager_waitForService` → `IVibrator::fromBinder` → `doWithRetry` 與 `IVibratorCallback`）。

### 2.5 框架 Manager 與應用程式層存取硬體服務（`2.5-app-client.md`）
- **情景**：應用程式取得 `LightsManager`、建立受生命週期保護的 `LightsSession`、送出 `LightsRequest` 請求調整裝置燈光，並沿完整跨行程呼叫鏈追蹤至 `LightsService` 與 `ILights` AIDL HAL。
- **起點 → 終點**：App 呼叫 `context.getSystemService(LightsManager.class)` → `SystemServiceRegistry` 註冊的 `CachedServiceFetcher` 建立 `SystemLightsManager`（`ServiceManager.getServiceOrThrow(Context.LIGHTS_SERVICE)` → `ILightsManager.Stub.asInterface`）→ `openSession()`（`SystemLightsSession` 建立本地 `Binder` token 並呼叫 `mService.openSession(token, priority)`，`LightsService` 執行 `@EnforcePermission(CONTROL_DEVICE_LIGHTS)` 並對 `token.linkToDeath`）→ `session.requestLights(request)` → `sendStaticStates` → `mService.setLightStates(token, ids, states)` → `LightsService.LightsManagerBinderService.setLightStates` → `computeAndApplyLightConfigurationsLocked`（多 Session 優先權仲裁與系統靜音覆寫）→ `LightImpl.setColor` / `setLightUnchecked` → `mVintfLights.get().setLightState(...)`（跨行程至 `android.hardware.lights-service` Vendor 行程）；最後對照 `vibrator/aidl/default/example_java_client/example/vib/MyActivity.java` 說明直連 HAL 的邊界限制。

---

## 四、附錄：原書（Android 2.3）→ Android 17 對照表

### 4.1 檔案路徑與架構元件對照

| 原書 2.3 路徑 / 元件 | Android 17 路徑 / 元件 | 變化類型 | 說明 |
|---|---|---|---|
| `kernel/goldfish/drivers/freg/freg.c`（內建字元驅動） | `kernel/common`: `drivers/leds/{led-class.c,led-core.c}` + Vendor Kernel Module（`.ko`） | 邏輯改變 + 架構重構 | GKI 要求核心本體與供應商驅動分離，優先複用標準 Linux 子系統與 sysfs（`/sys/class/leds/`） |
| `external/freg/freg.c`（直開 `/dev/freg` 的 C 測試程式） | `platform/hardware/interfaces`: `light/aidl/vts/functional/VtsHalLightTargetTest.cpp` | 已移除並改寫 | 不再從系統層直觸驅動裝置檔，改以 VTS 測試框架透過 VINTF + Binder 驗證 AIDL HAL 介面契約 |
| `hardware/libhardware/include/hardware/hardware.h`、`hardware.c`（`hw_module_t`/`hw_get_module`） | `platform/hardware/libhardware/`（已退為歷史相容）→ 改由 `platform/hardware/interfaces/<hal>/aidl/` + `libbinder_ndk` / `libbinder_rs` 取代 | 棄用並重寫 | 從 `dlopen` 載入 `system_server` 同行程 `.so`，演進為獨立 Vendor 行程（或 Vendor APEX）提供 `@VintfStability` Stable AIDL 服務 |
| `system/core/rootdir/ueventd.rc`（開放 `/dev/freg 0666`） | `ueventd.rc` + SELinux policy + `lights-default.rc`（`class hal`、`user`/`group`）+ VINTF XML（`lights-default.xml`） | 邏輯改變 + 擴充 | 嚴禁 `0666` 全開；硬體節點僅授權給對應 Vendor HAL 行程，並由 `ServiceManager` 強制檢驗 VINTF 與 SELinux |
| `frameworks/base/services/java/com/android/server/FregService.java` + `com_android_server_FregService.cpp`（必經 JNI） | `LightsService.java`（Java 直連 `ILights`，**無 JNI**）或 `VibratorManagerService.java` + `com_android_server_vibrator_VibratorController.cpp` + `VibratorHalController.cpp`（JNI + NDK） | 邏輯改變 | Java 系統服務可直接呼叫 `@VintfStability` Java AIDL 代理（免寫 JNI）；需 C++ 複用或低延遲時才經 JNI 呼叫 `libbinder_ndk` |
| `frameworks/base/services/java/com/android/server/SystemServer.java`（`ServerThread::run` 手動 `ServiceManager.addService`） | `SystemServer.java` 經 `mSystemServiceManager.startService(LightsService.class)` + `LightsService::onStart()` 呼叫 `publishBinderService` / `publishLocalService` | 邏輯改變 | 引入 `SystemService` 生命週期與 `onBootPhase`，同時區分同行程 `LocalServices` 與跨行程 Binder 服務 |
| App 直接呼叫 `ServiceManager.getService("freg")` | `Context.getSystemService(LightsManager.class)` → `SystemServiceRegistry` → `SystemLightsManager` → `ILightsManager`（含 `openSession(token)` 與 `@EnforcePermission`） | 邏輯改變 | 透過公開／系統 Manager 封裝隱藏 Binder 細節，並以 `IBinder` session token + `linkToDeath` 管理多客戶端仲裁與資源自動回收 |
