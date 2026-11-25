# Platform research: Galaxy Watch 7 (Wear OS) <-> iPhone 15 Pro Max (CoreBluetooth + HealthKit)

Research date: 2026-11-25. All claims are taken from the URLs cited next to them. Items marked **[UNCERTAIN]** have no official source or only weak (forum or third-party) sources. Items marked **[INFERENCE]** are my own reasoning from the cited facts.

---

## 0. Show-stoppers and big risks (read first)

| # | Risk | Severity | Evidence |
|---|---|---|---|
| R0 | **Samsung says the Galaxy Watch 7 does not support iOS.** It needs an Android 11+ phone with Galaxy Wearable and Google Play services. Setting the watch up, running Samsung Health sync and installing your APK all assume an Android host somewhere (another Android phone, or ADB sideloading over Wi-Fi). Your app must be fully standalone, and the iPhone sees the watch only as a generic BLE peripheral. | Critical (setup/ops) | https://www.samsung.com/uk/support/mobile-devices/what-phones-are-compatible-with-the-galaxy-watch/ , https://www.samsung.com/nz/support/mobile-devices/watch-and-phone-compatibility |
| R1 | **Sleep stages are not available to a third-party Wear OS app through official APIs.** Health Services has no sleep DataType. Passive monitoring exposes only `UserActivityState.USER_ACTIVITY_ASLEEP` (asleep/awake/exercise). Health Connect does not run on watches. Samsung Health Data SDK (the source of sleep stages) is a **phone** SDK that reads from Samsung Health on an Android phone. | Critical for "sleep stages" | sections 1.3 and 1.5 |
| R2 | **ANCS from Wear OS is possible in principle** (there is an MIT-licensed project, WearBridge, that does it), but it needs the watch to bond with the iPhone as an accessory and act as GATT **client**. Running that alongside your own GATT-server role is undocumented on Android. Google documents nothing about it, and Apple supports it only as a generic BLE accessory. | High | section 3 |
| R3 | **HealthKit under re-signing.** HealthKit works only if the re-signer's provisioning profile, which Apple signs and nobody can edit, allowlists `com.apple.developer.healthkit` **and** the re-signed binary claims it. That requires an explicit App ID with HealthKit enabled. Typical re-signing services use shared enterprise or developer certs, often with wildcard App IDs, which cannot carry HealthKit. FlekStore publishes no technical documentation on this. | High | section 6 |
| R4 | **Phone-call audio on the watch is not achievable.** Call audio would need the watch to be the HFP Hands-Free (HF) unit to the iPhone over Bluetooth Classic. On Android that role (`BluetoothHeadsetClient`) is `@SystemApi`/`@hide` and needs `BLUETOOTH_PRIVILEGED`. iOS gives third-party apps no API to the audio of a cellular call. Realistic scope: show incoming-call notifications through ANCS and accept/decline through ANCS positive/negative actions, with no audio. | Critical for "calls with audio" | section 3.6 |
| R5 | Wear OS version churn: the Galaxy Watch 7 runs One UI 8 Watch (Wear OS 6, API 36) as stable. One UI 9 Watch (Wear OS 7) is stable on the Watch 8 and still in beta on the Watch 7 as of mid-November 2026. | Medium | section 2.1 |

---

## 1. Health Services (androidx.health.services.client)

### 1.1 Version
- Current stable: **`androidx.health:health-services-client:1.1.0`**, released **September 23, 2026**. Changes since 1.0.0: debounced goals, exercise events (`GolfShotEvent`), running dynamics (stride length, ground contact time, vertical oscillation, vertical ratio), and `SWIMMING_LAP_COUNT_TOTAL`. Earlier: 1.0.0 stable on May 7, 2025, and `ELEVATION_GAIN_DAILY` added in 1.1.0-alpha03. No newer alpha is listed.
  https://developer.android.com/jetpack/androidx/releases/health
- The library minSdk is 30 (Wear OS 3+), per the release notes for 1.0.0-alpha03 on the same page.

### 1.2 ExerciseClient data types
The full DataType list, from the reference https://developer.android.com/reference/kotlin/androidx/health/services/client/data/DataType :
- Heart rate: `HEART_RATE_BPM`, `HEART_RATE_BPM_STATS`
- Distance: `DISTANCE`, `DISTANCE_TOTAL`, `DISTANCE_DAILY`, plus `FLAT_GROUND_DISTANCE`, `INCLINE_DISTANCE`, `DECLINE_DISTANCE` (each with a `_TOTAL` form)
- Calories: `CALORIES`, `CALORIES_TOTAL`, `CALORIES_DAILY`
- Pace/speed: `PACE`, `PACE_STATS`, `SPEED`, `SPEED_STATS`
- Location (route): `LOCATION` (`LocationData`). Needs `isGpsEnabled = true` in `ExerciseConfig`.
- Elevation/floors: `ABSOLUTE_ELEVATION`, `ABSOLUTE_ELEVATION_STATS`, `ELEVATION_GAIN`, `ELEVATION_GAIN_TOTAL`, `ELEVATION_GAIN_DAILY`, `ELEVATION_LOSS`, `ELEVATION_LOSS_TOTAL`, `FLOORS`, `FLOORS_TOTAL`, `FLOORS_DAILY`
- Steps: `STEPS`, `STEPS_TOTAL`, `STEPS_DAILY`, `STEPS_PER_MINUTE(_STATS)`, `WALKING_STEPS`, `RUNNING_STEPS`
- Other: `REP_COUNT`, `SWIMMING_STROKES`, `SWIMMING_LAP_COUNT`, `GOLF_SHOT_COUNT`, running dynamics, `ACTIVE_EXERCISE_DURATION_TOTAL`, `RESTING_EXERCISE_DURATION(_TOTAL)`
- **There is no sleep DataType.**

Exercise guide: https://developer.android.com/health-and-fitness/guides/health-services/active-data
- Supported types vary by device and exercise. Query `ExerciseTypeCapabilities.supportedDataTypes` before requesting.
- Run the workout in a foreground service with `android:foregroundServiceType="health|location"`. Use `location` alone only if you target Wear OS 4 or lower.
- `BatchingMode.HEART_RATE_5_SECONDS` gives more frequent HR delivery with the screen off, for example to stream HR to a phone. Check it against `capabilities.supportedBatchingModeOverrides`.
- Data-point timestamps are relative to boot. Convert them with `bootInstant = now - elapsedRealtime`.
- Only one exercise runs at a time across all apps. Another app (for example Samsung Health) can end yours.

### 1.3 PassiveMonitoringClient
Guide: https://developer.android.com/health-and-fitness/guides/health-services/monitor-background
- Passive types use the `_DAILY` DataTypes (`STEPS_DAILY`, `DISTANCE_DAILY`, `CALORIES_DAILY`, `FLOORS_DAILY`, `ELEVATION_GAIN_DAILY`) plus `HEART_RATE_BPM`. Check `capabilities.supportedDataTypesPassiveMonitoring`. The guide's example uses `HEART_RATE_BPM` and `STEPS_DAILY`.
- Background delivery: subclass `PassiveListenerService` and declare it with `android:permission="com.google.android.wearable.healthservices.permission.PASSIVE_DATA_BINDING"`. Data arrives in **batches**.
- Registrations do **not** survive a reboot. Re-register from `BOOT_COMPLETED` by way of WorkManager, because Health Services can take 10 seconds or more to acknowledge.
- Activity state: request `ACTIVITY_RECOGNITION`, set `setShouldUserActivityInfoBeRequested(true)`, and handle `onUserActivityInfoReceived()`. Values (https://developer.android.com/reference/kotlin/androidx/health/services/client/data/UserActivityState): `USER_ACTIVITY_ASLEEP`, `USER_ACTIVITY_EXERCISE`, `USER_ACTIVITY_PASSIVE`, `USER_ACTIVITY_UNKNOWN`.
  - So a third-party app gets **only an asleep/awake transition** (with `stateChangeTime`, which may be in the past). It gets no sleep stages or sessions. You can build an "asleepUnspecified" session from ASLEEP→PASSIVE transitions. **[INFERENCE]**

### 1.4 Permissions (Wear OS 5.x vs 6)
- Permissions page: https://developer.android.com/health-and-fitness/health-services/permissions
  - `HEART_RATE_BPM` needs `android.permission.health.READ_HEART_RATE`. That applies on API 36 (Wear OS 6) and higher. On API 35 and lower it is `BODY_SENSORS`.
  - Steps, distance, calories, floors, elevation gain/loss, pace, speed and `UserActivityState` need `ACTIVITY_RECOGNITION`.
  - `LOCATION` and `ABSOLUTE_ELEVATION` need `ACCESS_FINE_LOCATION`.
  - Background body-sensor access from passive monitoring: on API 36+, `android.permission.health.READ_HEALTH_DATA_IN_BACKGROUND`. On API 33–35, `BODY_SENSORS` plus `BODY_SENSORS_BACKGROUND`.
  - Manifest migration: `<uses-permission android:name="android.permission.BODY_SENSORS" android:maxSdkVersion="35"/>`, `<uses-permission android:name="android.permission.health.READ_HEART_RATE"/>`, and the same pattern for BACKGROUND / `READ_HEALTH_DATA_IN_BACKGROUND`.
- Wear OS 6 behaviour changes: https://developer.android.com/training/wearables/versions/6/changes
  - Wear OS 6 is based on Android 16 (API 36). `BODY_SENSORS` maps to `READ_HEART_RATE`, plus `READ_OXYGEN_SATURATION` and `READ_SKIN_TEMPERATURE` on supported devices. `BODY_SENSORS_BACKGROUND` maps to `READ_HEALTH_DATA_IN_BACKGROUND`.
  - Apps targeting 35 or lower get `READ_HEART_RATE` requested automatically on Wear OS 6.

### 1.5 Sleep stages on Galaxy Watch: other routes
- **Health Connect on the watch: not available.** Samsung's FAQ says the Health Connect app "does not support Wear OS devices". Galaxy Watch data goes to Samsung Health on the **phone**, and Samsung Health syncs sleep to Health Connect on the phone.
  https://developer.samsung.com/health/health-connect-faq.html
  Google's Health Connect getting-started guide covers phones only (Android 14 framework module, or the APK on Android 13 and lower), and no Wear OS support is documented. https://developer.android.com/health-and-fitness/guides/health-connect/develop/get-started
- **Samsung Health Data SDK:** reads Samsung Health data, including Sleep, on an **Android phone** (Android 10+, Samsung Health 6.30.2+, no emulator).
  https://developer.samsung.com/health/data/overview.html , https://developer.samsung.com/health/data/guide/introduction.html
  - The sleep codelab (phone app) exposes sessions with stages **Light, REM, Deep, Awake**, plus duration, sleep score, and SpO2/skin temperature where supported. https://developer.samsung.com/codelab/health/sleep-data.html
  - Testing uses Samsung Health "developer mode". Distribution requires a **partner request** with the package name and SHA-256 signature, otherwise you get an auth exception. https://developer.samsung.com/health/data/faq.html
  - It does not run on the watch and is not an iOS SDK. With an iPhone-only setup, **this route does not exist** unless you also keep an Android phone paired to the watch and running Samsung Health. **[INFERENCE]**
  - The older Samsung Health (Android) SDK is deprecated as of July 31, 2025, per search summaries of Samsung pages. **[UNCERTAIN wording]**
- **Samsung Health Sensor SDK (on the watch):** gives raw trackers on Galaxy Watch4 and later: accelerometer, PPG, ECG, HR with IBI, SpO2, skin temperature, BIA, EDA. It provides **no sleep staging**, so you would have to build your own sleep classifier. https://developer.samsung.com/health/sensor/overview.html
- **Bottom line:** for the iPhone, the realistic sources are "asleep/awake" from `UserActivityState` and, optionally, your own HR- or motion-based heuristic, written to HealthKit as `asleepUnspecified` + `inBed`. Writing stages would mean inventing them.

---

## 2. Wear OS platform

### 2.1 Galaxy Watch 7 OS
- Stable One UI 8 Watch (**Wear OS 6, API 36**) has been on the Galaxy Watch 7 since October 2025, with security updates through 2026.
  https://www.androidcentral.com/wearables/samsung-galaxy-watch/wear-os-6-galaxy-watch-7 , https://9to5google.com/2025/10/14/samsung-one-ui-8-galaxy-watch-7-rollout/
- One UI 9 Watch (Wear OS 7, Android 17 based) rolled out stable to the Galaxy Watch 8 in November 2026. For the Galaxy Watch 7 there is a **beta** (September 2026), and Samsung says compatible watches get it by the end of 2026.
  https://www.androidauthority.com/galaxy-watch-8-one-ui-9-wear-os-7-stable-update-3732792/ , https://www.sammobile.com/news/samsung-launches-galaxy-watch-7-one-ui-9-beta-program/
- Wear OS 7 developer notes: Live Updates replace Ongoing Activities for local publishing, and there is a new "Wear Workout Tracker". https://android-developers.googleblog.com/2026/05/whats-new-wear-os-7.html
- Recommendation: target API 36, test on API 36, and expect API 37 soon. **[INFERENCE]**

### 2.2 Foreground service types
https://developer.android.com/develop/background-work/services/fgs/service-types
- `connectedDevice`: needs `FOREGROUND_SERVICE_CONNECTED_DEVICE` and **at least one** of the runtime permissions `BLUETOOTH_CONNECT`, `BLUETOOTH_ADVERTISE` or `BLUETOOTH_SCAN` granted.
- `health`: needs `FOREGROUND_SERVICE_HEALTH` and one of `HIGH_SAMPLING_RATE_SENSORS` (manifest), `BODY_SENSORS` (API ≤35), `READ_HEART_RATE`, `READ_SKIN_TEMPERATURE`, `READ_OXYGEN_SATURATION` or `ACTIVITY_RECOGNITION`. These are while-in-use permissions, so you **cannot start a body-sensor health FGS from the background** without `READ_HEALTH_DATA_IN_BACKGROUND` (API 36) or `BODY_SENSORS_BACKGROUND` (API 33–35).
- `location`: needs `FOREGROUND_SERVICE_LOCATION`, location enabled, and coarse or fine location granted. You can't start it from the background without `ACCESS_BACKGROUND_LOCATION`.
- `dataSync`, `mediaPlayback`, `phoneCall` and `mediaProjection` can't be launched from `BOOT_COMPLETED` when you target Android 15+.
- One service can combine types: `health|location|connectedDevice`.

### 2.3 BLE on Wear OS
- Permissions (Android 12+): `BLUETOOTH_SCAN` (optionally with `neverForLocation`), `BLUETOOTH_ADVERTISE` to make the device discoverable/advertise, and `BLUETOOTH_CONNECT` to talk to paired devices. All are runtime permissions under "Nearby devices". https://developer.android.com/develop/connectivity/bluetooth/bt-permissions
- `BluetoothLeAdvertiser` (API 21+) requires `BLUETOOTH_ADVERTISE`. https://developer.android.com/reference/android/bluetooth/le/BluetoothLeAdvertiser
- The Android BLE guide says a device can be peripheral, central, or both, and that older chipsets may not support peripheral mode. https://developer.android.com/guide/topics/connectivity/bluetooth-le
- **[UNCERTAIN]** No Wear OS- or Samsung-specific official doc confirms or denies `BluetoothLeAdvertiser` or `BluetoothGattServer` for third-party apps on the Galaxy Watch 7. Check `BluetoothAdapter.isMultipleAdvertisementSupported()` and `getBluetoothLeAdvertiser() != null` at runtime, and do a hardware spike first.
- Background: Wear OS applies Doze and App Standby. Use WorkManager for deferred work, and keep the BLE link alive with a foreground service (`connectedDevice`). https://developer.android.com/training/wearables/data/network-access
- Alternative: the Companion Device Manager (CDM) presence API helps an app stay alive while a companion is in range. It is documented for phones, and its availability on Wear OS is unverified **[UNCERTAIN]**. https://developer.android.com/develop/connectivity/bluetooth/companion-device-pairing
- **[INFERENCE]** Android peripheral advertising uses rotating private addresses. iOS can reconnect reliably to a stable `CBPeripheral.identifier` only after the devices bond and exchange IRKs. Plan for bonding.

---

## 3. ANCS and AMS (watch as client of the iPhone)

### 3.1 Specs (Apple archive)
- ANCS intro: https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleNotificationCenterServiceSpecification/Introduction/Introduction.html
- ANCS spec: https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleNotificationCenterServiceSpecification/Specification/Specification.html
- ANCS appendix (enums): https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleNotificationCenterServiceSpecification/Appendix/Appendix.html
- AMS intro: https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleMediaService_Reference/Introduction/Introduction.html
- AMS spec: https://developer.apple.com/library/archive/documentation/CoreBluetooth/Reference/AppleMediaService_Reference/Specification/Specification.html

### 3.2 UUIDs
- ANCS service `7905F431-B5CE-4E99-A40F-4B1E122D00D0`
  - Notification Source `9FBF120D-6301-42D9-8C58-25E699A21DBD` (notify, mandatory)
  - Control Point `69D1D8F3-45E1-49A8-9821-9BBDFDAAD9D9` (write with response)
  - Data Source `22EAC6E9-24D6-4BB5-BE44-B36ACE7C7BFB` (notify)
- AMS service `89D3502B-0F36-433A-8EF4-C502AD55F8DC`
  - Remote Command `9B3C81D8-57B1-4A8A-B8DF-0E56F7CA51C2` (write, notify)
  - Entity Update `2F7CABCE-808D-411F-9A0C-BB92BA96C102` (write with response, notify)
  - Entity Attribute `C6B2F38C-23AB-46D8-A6AB-A3A870BBD5D7` (read, write)
- Enums from the appendix:
  - CategoryID: 0 Other, **1 IncomingCall**, 2 MissedCall, 3 Voicemail, 4 Social, 5 Schedule, 6 Email, 7 News, 8 HealthAndFitness, 9 BusinessAndFinance, 10 Location, 11 Entertainment
  - EventID: 0 Added, 1 Modified, 2 Removed
  - EventFlags: bit0 Silent, bit1 Important, bit2 PreExisting, bit3 PositiveAction, bit4 NegativeAction
  - CommandID: 0 GetNotificationAttributes, 1 GetAppAttributes, 2 PerformNotificationAction
  - ActionID: 0 Positive, 1 Negative
  - NotificationAttributeID 0–7: AppIdentifier, Title, Subtitle, Message, MessageSize, Date, PositiveActionLabel, NegativeActionLabel
  - Numbers are little-endian and strings are UTF-8.

### 3.3 Roles, security, solicitation
- The iPhone is the publisher, NP for ANCS and MS for AMS. The accessory is the client, NC or MR. Apple: "An accessory acting as a GATT client is free to access and use other services provided by the iOS device." (ANCS/AMS intro pages)
- "All these characteristics require authorization for access" (ANCS spec). In practice, iOS answers with Insufficient Authentication until the link is encrypted and bonded, and that triggers pairing. Sources: Apple bluetooth-dev list https://lists.apple.com/archives/bluetooth-dev/2013/Jul/msg00046.html and TI E2E https://e2e.ti.com/support/wireless-connectivity/bluetooth-group/bluetooth/f/bluetooth-forum/991838/cc2640r2f-q1-ancs-authentication **[forum-level sources]**
- ANCS/AMS "is not guaranteed to always be present". Subscribe to Service Changed (spec pages).
- **Solicitation:** the convention is that the accessory advertises the ANCS UUID in the "List of 128-bit Service Solicitation UUIDs" AD type, which makes the service visible to the accessory after bonding. The current Apple Accessory Design Guidelines section was **not retrievable**, so this is **[UNCERTAIN]**. Separately, CoreBluetooth offers `CBConnectPeripheralOptionRequiresANCS`, which lets an iOS app that connects to the accessory ask iOS to require ANCS on that link. That is useful in your setup because your own iOS app is the central. https://developer.apple.com/documentation/corebluetooth/cbconnectperipheraloptionrequiresancs
- **Same link, both roles:** GATT client/server roles are independent of link-layer central/peripheral, and both devices can host a GATT server on one connection. Example: the watch is a GATT server for your custom service and also a GATT client of ANCS on the same LE link. Android exposes `BluetoothGattServer` (server) and `BluetoothDevice.connectGatt()` (client) separately. **[UNCERTAIN]** Whether `connectGatt()` to an iPhone that is already connected to your `BluetoothGattServer` reuses the same ACL link is undocumented. It normally does on the Android stack, but verify on the device. Apple mailing-list discussion: https://lists.apple.com/archives/bluetooth-dev/2013/Nov/msg00050.html

### 3.4 Evidence that Wear OS can be an ANCS client
- **WearBridge** (MIT, Kotlin/Compose): the Wear OS watch is the BLE central and GATT client, bonds with the iPhone (the user accepts the iOS pairing dialog), and subscribes to ANCS. Tested on a Pixel Watch 2 (Wear OS 5). A Galaxy Watch is listed as "secondary" and a OnePlus Watch 2 was community-tested. It has an incoming-call overlay. Quick reply is still pending. https://github.com/k97/WearBridge
- Older: glass_ble_ancs (Google Glass), and MohammadAG's 2015 Android Wear ANCS demo (press only).
- Non-Android reference implementations: Nordic nRF Connect SDK ANCS client https://nrfconnectdocs.nordicsemi.com/ncs/latest/nrf/libraries/bluetooth/services/ancs_client.html , ancs4linux https://github.com/pzmarzly/ancs4linux

### 3.5 Practical design implication [INFERENCE]
Because your iOS app is already a central connected to the watch, the simplest path is this. The iOS app connects with `CBConnectPeripheralOptionRequiresANCS: true`. The watch's custom characteristics require encryption, so pairing happens once. The watch then discovers ANCS/AMS on the iPhone over the same bonded link with `connectGatt`. Validate this in a spike before committing.

### 3.6 Calls with audio
- ANCS gives call notifications (CategoryID 1) and positive/negative actions, which map to accept/decline. It carries no audio (ANCS spec and appendix).
- Audio would need HFP. On Android the HF role (`BluetoothHeadsetClient`) is a hidden system API that needs `BLUETOOTH_PRIVILEGED`, so it is not available to Play or sideloaded apps on stock firmware. AOSP: https://android.googlesource.com/platform/system/bt/+/91f38752fa70188a33f07342d55c08125f1b0f98/binder/android/bluetooth/IBluetoothHeadsetClient.aidl . Public `BluetoothHeadset` is the AG side only: https://developer.android.com/reference/android/bluetooth/BluetoothHeadset
- iOS: there is no public API for third-party access to native cellular call audio (Apple DTS on the forums). https://developer.apple.com/forums/thread/841340
- **Conclusion:** no call audio on the watch. **[High confidence, forum-plus-AOSP sourcing]**

---

## 4. CoreBluetooth (iOS central)

- Background mode: `UIBackgroundModes` = `bluetooth-central`. In the background the app can discover, connect and interact, and it is woken for delegate events. `AllowDuplicates` is ignored, and the scan interval grows when all scanning apps are in the background.
  https://developer.apple.com/library/archive/documentation/NetworkingInternetWeb/Conceptual/CoreBluetooth_concepts/CoreBluetoothBackgroundProcessingForIOSApps/PerformingTasksWhileYourAppIsInTheBackground.html
- `connect(_:options:)`: "Attempts to connect to a peripheral don't time out." Cancel with `cancelPeripheralConnection`. Deallocating the peripheral also cancels.
  https://developer.apple.com/documentation/corebluetooth/cbcentralmanager/connect(_:options:)
  A pending connect to a known (bonded) peripheral is the recommended auto-reconnect pattern (background guide above).
- State preservation and restoration: pass `CBCentralManagerOptionRestoreIdentifierKey`, implement `centralManager(_:willRestoreState:)`, and re-set peripheral delegates. In scene-based apps `launchOptions` is nil, so persist the UID yourself and pass the same UID every launch.
  https://developer.apple.com/documentation/corebluetooth/cbcentralmanageroptionrestoreidentifierkey
- Relaunch rules: TN3115. A **user force-quit means no relaunch.** Apple DTS describes an exception for AccessorySetupKit apps on iOS 26+, and a forum report disputes it **[UNCERTAIN]**.
  https://developer.apple.com/documentation/technotes/tn3115-bluetooth-state-restoration-app-relaunch-rules , https://developer.apple.com/forums/thread/840468
- Connection parameters and advertising (for the watch as peripheral): advertise at 20 ms for 30 s, then one of 152.5, 211.25, 318.75, 417.5, 546.25, 760, 852.5, 1022.5 or 1285 ms. Connection interval min ≥15 ms, supervision timeout 2–6 s. https://developer.apple.com/library/archive/qa/qa1931/_index.html
- Pairing and encryption: on the peripheral side, encryption-required permissions make the link encrypt and pair. `CBAttributePermissions` documents `readEncryptionRequired` / `writeEncryptionRequired`. https://developer.apple.com/documentation/corebluetooth/cbattributepermissions . On Android the equivalent is `BluetoothGattCharacteristic.PERMISSION_READ_ENCRYPTED(_MITM)`.
  - iPhone pairing is P-256 / LE Secure Connections capable. https://support.apple.com/guide/security/sec82597d97e/web
  - Numeric comparison (a 6-digit code on both devices) is used only with LE Secure Connections when both sides have DisplayYesNo-class I/O. https://www.bluetooth.com/blog/bluetooth-pairing-part-4/
  - **[UNCERTAIN]** What iOS shows depends on the IO capabilities the Android stack advertises. A watch with a display usually triggers a numeric-comparison style dialog on both devices, and Just Works triggers a plain "Pair / Cancel" alert. I found no official Apple page that describes the iOS dialog. Test it.

---

## 5. HealthKit (writing)

- Setup: add the HealthKit capability. Xcode then adds `healthkit` to `UIRequiredDeviceCapabilities`, which you can remove if HealthKit is optional. Call `HKHealthStore.isHealthDataAvailable()` and keep one long-lived `HKHealthStore`. Entitlements are `com.apple.developer.healthkit` and `com.apple.developer.healthkit.access` (the latter only for clinical records/verifiable). The Info.plist keys are `NSHealthUpdateUsageDescription` (required for writing) and `NSHealthShareUsageDescription` (required for reading).
  https://developer.apple.com/documentation/healthkit/setting-up-healthkit , https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.healthkit , https://developer.apple.com/documentation/bundleresources/information-property-list/nshealthupdateusagedescription
- Heart rate: `HKQuantityTypeIdentifier.heartRate`, unit count/time (`count/min`). The optional metadata `HKMetadataKeyHeartRateMotionContext` can be set by any app. https://developer.apple.com/documentation/healthkit/hkquantitytypeidentifier/heartrate
- Workouts on iOS: use `HKWorkoutBuilder(healthStore:configuration:device:)`, then `beginCollection` → `add(samples)` → `addWorkoutEvents` → `endCollection` → `finishWorkout`. `HKLiveWorkoutBuilder` is watchOS only. https://developer.apple.com/documentation/healthkit/hkworkoutbuilder
- Route: get `HKWorkoutRouteBuilder` from `workoutBuilder.seriesBuilder(for: HKSeriesType.workoutRoute())`, call `insertRouteData([CLLocation])`, then `finishRoute(with: workout, metadata:)` after the workout is saved. Apple discourages direct init. https://developer.apple.com/documentation/healthkit/hkworkoutroutebuilder
- Sleep: `HKCategoryType(.sleepAnalysis)` with `HKCategoryValueSleepAnalysis` values `inBed`, `awake`, `asleepCore`, `asleepDeep`, `asleepREM` and `asleepUnspecified` (iOS 16+). `asleep` is deprecated. Stage samples may overlap the inBed sample but must not overlap each other. https://developer.apple.com/documentation/healthkit/hkcategoryvaluesleepanalysis , https://developer.apple.com/documentation/healthkit/hkcategoryvaluesleepanalysis/asleepunspecified
- Steps, distance, flights and active energy use the quantity identifiers `stepCount`, `distanceWalkingRunning`, `flightsClimbed` and `activeEnergyBurned` (the standard `HKQuantityTypeIdentifier` set; individual pages not fetched).
- **Dedup and update:** `HKMetadataKeySyncIdentifier` (String) **must** be paired with `HKMetadataKeySyncVersion` (NSNumber). When you save an object with the same sync identifier, HealthKit replaces the existing object if the new version is greater. It also replaces the object inside its workout or correlation. https://developer.apple.com/documentation/healthkit/hkmetadatakeysyncidentifier , https://developer.apple.com/documentation/healthkit/hkmetadatakeysyncversion
  - **[INFERENCE]** Use deterministic IDs such as `watch-hr-<epochMillis>` with version 1 for idempotent re-sends after BLE reconnects.

---

## 6. Signing, re-signing and HealthKit

- Provisioning profile semantics (TN3125): every entitlement the app claims in its code signature must be in the profile's allowlist. The profile is signed by Apple and cannot be edited. Modern systems trust `DER-Encoded-Profile`. https://developer.apple.com/documentation/technotes/tn3125-inside-code-signing-provisioning-profiles
  - Apple DTS: "you can't edit a provisioning profile", and DTS does not support re-signing .ipa files. https://developer.apple.com/forums/thread/769038
- Which programs offer HealthKit: per Apple's capability table, **ADP ✓, ADEP (Enterprise) ✓, free Apple Developer ✓**. Background Modes are ✓/✓/✓. https://developer.apple.com/help/account/reference/supported-capabilities-ios
- HealthKit is an app-specific capability, so it needs an **explicit App ID**. Wildcard App IDs cannot enable such capabilities. https://developer.apple.com/library/archive/qa/qa1713/_index.html , https://developer.apple.com/help/account/identifiers/register-an-app-id
- Symptom when it is missing: `Missing com.apple.developer.healthkit entitlement` (HKError code 4), seen when the profile has it but the re-signed binary does not, or the other way round. https://docs.median.co/discuss/6a99d3dc52ac1e308d5f3a2a
- **Conclusion for FlekStore/ESign-type services [INFERENCE + UNCERTAIN]:** HealthKit works only if (a) the certificate's team has an explicit App ID matching your (possibly rewritten) bundle ID with HealthKit enabled, (b) the profile allowlists `com.apple.developer.healthkit`, and (c) the re-signer writes that entitlement into the binary signature.
  - Shared enterprise certs generally don't do (a) for arbitrary user IPAs. A **personal certificate imported into FlekStore** (your own $99 ADP account, or a free account through other tools) can, if you create the App ID with HealthKit yourself.
- FlekStore facts: a web/app sideloading store. You either import your own developer certificate or buy one from FlekStore (from about $15.99). It uses a UDID profile and warns that purchased certs can be revoked. No public technical docs exist on entitlements or custom-IPA support. https://macobserver.com/?p=228763 , https://flekstore.com/ , https://www.trustpilot.com/review/flekstore.com — **[UNCERTAIN]** whether FlekStore preserves custom entitlements. Verify with `codesign -d --entitlements :- Payload/App.app` on a re-signed IPA (you can only do that with a Mac or CI), or inspect the result on-device through HealthKit authorization behaviour.
- **Building unsigned or ad-hoc on GitHub Actions:**
  - Ad-hoc signing uses the pseudo-identity `-` (`codesign -s -`). Xcode's "Sign to Run Locally" is ad-hoc signing (Apple DTS). https://developer.apple.com/documentation/security/seccodesignatureflags/adhoc , https://developer.apple.com/forums/thread/723669
  - `CODE_SIGN_ENTITLEMENTS` is the path to the entitlements file. `CODE_SIGN_IDENTITY` is the cert common name. https://developer.apple.com/documentation/xcode/build-settings-reference
  - **[INFERENCE, standard practice; `CODE_SIGNING_ALLOWED` is not in Apple's build-settings reference]**:
    - `xcodebuild ... CODE_SIGNING_ALLOWED=NO` produces an **unsigned** .app with **no entitlements embedded**. The re-signer must then supply them, which many do from the new profile's allowlist.
    - Safer: build with `CODE_SIGNING_ALLOWED=NO`, then run `codesign --force --sign - --entitlements App.entitlements Payload/App.app` on the macOS runner. Sign nested frameworks first. Zip `Payload/` into a `.ipa`.
    - The result carries the entitlements blob for re-signers that copy entitlements from the binary.

---

## 7. XcodeGen

- Latest release: **2.48.0** (Nov 19, 2026; year inferred from the Xcode 27.2 / Swift 6.4 references). It needs macOS 14 and Swift 6.1 to run, and adds experimental support for Xcode 27.2's JSON project format (`options.projectFileFormat: xcproj`). 2.47.0 added the `xcode26_3`/`xcode27_0` formats and defaults `xcodeVersion` to 16.0.
  https://github.com/yonaskolb/XcodeGen/releases , https://github.com/yonaskolb/XcodeGen/blob/master/CHANGELOG.md
- Spec: https://github.com/yonaskolb/XcodeGen/blob/master/Docs/ProjectSpec.md
  - target `info: {path, properties}` generates Info.plist and sets `INFOPLIST_FILE`
  - `entitlements: {path, properties}` generates `.entitlements` and sets `CODE_SIGN_ENTITLEMENTS`
  - top-level `packages:` with `path:` declares a local package
  - target `dependencies: - package: Name` (with optional `product:` / `products:`)

```yaml
name: WatchBridge
options:
  bundleIdPrefix: com.example
  deploymentTarget: { iOS: "17.0" }
packages:
  BridgeCore:
    path: Packages/BridgeCore          # local Swift package (dir with Package.swift)
targets:
  WatchBridge:
    type: application
    platform: iOS
    sources: [App]
    dependencies:
      - package: BridgeCore
      - sdk: HealthKit.framework
      - sdk: CoreBluetooth.framework
    info:
      path: App/Info.plist
      properties:
        UILaunchScreen: {}
        UIBackgroundModes: [bluetooth-central]
        NSBluetoothAlwaysUsageDescription: "Connects to your Galaxy Watch."
        NSHealthShareUsageDescription: "Reads workouts you saved."
        NSHealthUpdateUsageDescription: "Saves heart rate, workouts and sleep from your watch."
        UIRequiredDeviceCapabilities: [healthkit]
    entitlements:
      path: App/WatchBridge.entitlements
      properties:
        com.apple.developer.healthkit: true
        com.apple.developer.healthkit.access: []
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.example.watchbridge
        CODE_SIGNING_ALLOWED: NO
```
(`NSBluetoothAlwaysUsageDescription` is the standard CoreBluetooth key. It was not separately fetched here.)

---

## 8. GitHub Actions billing (late 2026)

- Standard runner per-minute prices: Linux 2-core $0.006, Windows $0.010, **macOS 3/4-core (M1 or Intel) $0.062**. Larger runners are not free for public repos. https://docs.github.com/en/billing/reference/actions-runner-pricing
- Standard GitHub-hosted runners are **free in public repositories**, and that use does not reduce the included quota. GitHub Free personal accounts get **$12/month of included compute** (expressed as money, not minutes). Artifact storage is 500 MB on Free, cache 10 GB per repo, and self-hosted runners are free. https://docs.github.com/en/billing/concepts/product-billing/github-actions
  - **[INFERENCE]** On a private repo that is about 193 macOS minutes a month ($12 / $0.062). The old "10x multiplier" model is not described in current docs.
- 2026 changes: hosted runner prices were cut by up to 39% from January 1, 2026 (https://github.blog/changelog/2026-01-01-reduced-pricing-for-github-hosted-runners-usage/). The planned $0.002/min fee for self-hosted runners (from March 2026) was **postponed** (https://github.blog/changelog/2025-12-16-coming-soon-simpler-pricing-and-a-better-experience-for-github-actions/). Current docs still list self-hosted as free.
- **Recommendation:** use a public repo, or accept roughly 15–20 private macOS builds a month at about 10 minutes each. **[INFERENCE]**

---

## 9. Current stable versions (as of 2026-11-25)

| Item | Version | Source |
|---|---|---|
| Kotlin | 2.4.21 (Oct 8, 2026) | https://kotlinlang.org/docs/releases.html |
| Android Gradle Plugin | 9.5.1 (9.5 line; min/default Gradle 9.6.0, JDK 17, max API 37) | https://developer.android.com/build/releases/gradle-plugin |
| Gradle | 9.9.2 (Nov 19, 2026) | https://gradle.org/releases/ |
| Wear Compose (`androidx.wear.compose`) | 1.7.2 stable (Nov 18, 2026); 1.8.0-alpha04 | https://developer.android.com/jetpack/androidx/releases/wear-compose |
| Compose UI/Foundation | 1.13.0 | https://developer.android.com/jetpack/androidx/versions |
| Room | 2.8.5 (`androidx.room`); Room 3: 3.0.3 (`androidx.room3`) | https://developer.android.com/jetpack/androidx/versions |
| Health Services client | 1.1.0 | https://developer.android.com/jetpack/androidx/releases/health |
| Health Connect client (phone only) | 1.1.0 stable, 1.2.0-alpha06 | https://developer.android.com/jetpack/androidx/versions |
| WorkManager | 2.12.0 | https://developer.android.com/jetpack/androidx/versions |
| `macos-latest` runner | macOS 26 arm64 (image 20261117), **default Xcode 26.6** (also 26.0.1–26.5) | https://github.com/actions/runner-images , https://github.com/actions/runner-images/blob/main/images/macos/macos-26-arm64-Readme.md |
| Swift on runner default | Swift 6.3 (Xcode 26.6 bundles Swift 6.3 and the iOS 26.5 SDK) | https://developer.apple.com/documentation/xcode-release-notes/xcode-26_6-release-notes |
| Latest Xcode overall | Xcode 27.2 (Swift 6.4, iOS 27.2 SDK); 27.3 beta 2. Xcode 27 is available on runners as a public preview (`xcode-27` label) | https://developer.apple.com/documentation/xcode-release-notes/xcode-27_2-release-notes , https://github.com/actions/runner-images |

Note: `macos-latest` moves gradually (1–2 months), so pin `macos-26` and select Xcode explicitly (`sudo xcode-select -s /Applications/Xcode_26.6.app`). https://github.com/actions/runner-images
