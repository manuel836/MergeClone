# Fase 0 — Cimientos y esqueleto que camina: plan de implementación

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** El pulso del Galaxy Watch 7 viaja por BLE cifrado al iPhone, que lo escribe en Apple Salud sin duplicados; con CI, IPA y APK, simulador de enlace de punta a punta y las sondas de viabilidad listas para la ronda de pruebas reales.

**Architecture:** El reloj es periférico BLE (servidor GATT propio) y guarda todo en una cola Room antes de enviarlo; el iPhone es central CoreBluetooth con state restoration y escribe en HealthKit, y confirma con ACK. La lógica pura vive en `watch/core` (Kotlin/JVM) e `ios/Core` (paquete Swift), ambas probadas contra los mismos vectores binarios.

**Tech Stack:** Kotlin 2.4.21, AGP 9.5.1, Gradle 9.9.2, Compose for Wear OS 1.7.2 (Material 3), Room 2.8.5, Health Services 1.1.0; Swift 6.3 (Linux) / Xcode por defecto de `macos-latest`, SwiftUI, CoreBluetooth, HealthKit, XcodeGen 2.48.0; GitHub Actions; Python 3 (solo para validar vectores).

**Spec:** `docs/DISEÑO.md` (aprobado 2026-11-25). Fuentes técnicas: `docs/investigacion/plataformas-2026-11-25.md`.

## Global Constraints
- Nombre visible "Tándem"; paquete Android y bundle iOS `dev.personal.tandem`.
- Textos visibles (UI, docs, mensajes de error, PR) en español; código, identificadores y comentarios en inglés, comentando el porqué (el usuario no sabe Kotlin ni Swift).
- Sin permiso `INTERNET` en el reloj; sin red en el iPhone; sin analítica; el log nunca guarda texto de notificaciones.
- Wear OS: `minSdk 33`, `compileSdk 36`, `targetSdk 36`. iOS: deployment target 17.0, solo iPhone.
- UUIDs: servicio `2a03ae15-dedd-427a-b8f6-6f1de1097295`; `rx` (iPhone→reloj, write con respuesta) `b5d3d3d8-940b-4565-b411-f3db393f6c02`; `tx` (reloj→iPhone, notify) `03970326-27ac-4a47-830f-0539d90b7eec`. Ambas características exigen enlace cifrado.
- Enteros little-endian; tiempos en epoch ms UTC; CRC-32 IEEE 802.3 (el de zlib).
- SyncIdentifier de pulso: `tandem:hr:<epochMs>`; SyncVersion `1`.
- Proyecto iOS solo vía `ios/project.yml` (XcodeGen); `.xcodeproj` está en `.gitignore`.
- Ningún secreto en el repo; la firma del APK usa los secrets `TANDEM_KEYSTORE_B64`, `TANDEM_KEYSTORE_PASSWORD`, `TANDEM_KEY_ALIAS`, `TANDEM_KEY_PASSWORD`.
- Si una versión fijada arriba no resuelve en Gradle/SwiftPM, usar la estable más reciente de la misma serie y anotarlo en `docs/DECISIONES.md`.
- Reglas de calidad: `.claude/rules/ecc/` (common, kotlin, swift).

## Review Focus
1. **Reconexión a mitad de un mensaje fragmentado:** el reensamblador descarta el mensaje incompleto al desconectar y el reloj lo reenvía entero; nada se escribe dos veces. → prueba en Task 3 y Task 7 (`reassembler_resetDropsPartial`) y en Task 11 (escenario `drop-mid-message`).
2. **ACK perdido:** el reloj reenvía un RECORD ya escrito; el iPhone no duplica en Salud y vuelve a mandar ACK. → Task 4 (`session_duplicateRecordIsAckedWithoutWriting`) y Task 11.
3. **MTU mínimo (23):** todo mensaje cabe fragmentado con chunks de 12 bytes de payload útil. → vectores `frag-mtu23` (Task 2), Tasks 3 y 7.
4. **Reloj con horas de datos acumulados al reconectar** (p. ej. una noche sin iPhone): la cola se vacía en lotes sin bloquear ni perder orden. → Task 8 (`engine_drainsBacklogInOrderWithBoundedInFlight`).
5. **Permiso de Salud denegado en el iPhone:** no hay ACK, el reloj conserva los datos y la app lo dice en pantalla en español. → Task 4 (`session_writeFailureSendsNackAndKeepsRecord`) y Task 5 (estado en la UI).

---

## Estructura de archivos

```
scripts/verify.sh                         verificación local/CI; dice qué omitió
scripts/sim-e2e.sh                        corre el simulador de enlace (Task 11)
scripts/battery.sh                        mide % batería/h con adb (ronda final)
protocol/uuids.json                       UUIDs del servicio
protocol/vectors/*.json                   vectores compartidos
protocol/check_vectors.py                 valida los vectores (CRC, longitudes)
protocol/test_check_vectors.py
ios/Core/Package.swift                    paquete TandemCore + ejecutable tandem-sim-phone
ios/Core/Sources/TandemCore/{Frame,Fragmenter,Messages,HealthMapping,Ledger,PhoneSession,Log}.swift
ios/Core/Sources/tandem-sim-phone/main.swift
ios/Core/Tests/TandemCoreTests/*.swift
ios/project.yml                           XcodeGen
ios/App/{TandemApp,BleCentral,HealthKitWriter,FileLedgerStore,AppLog,AppModel}.swift
ios/App/Views/{WelcomeView,StatusView,HealthProbeView,LogView}.swift
ios/App/Tandem.entitlements, ios/App/Info.plist (generado desde project.yml)
watch/settings.gradle.kts, watch/gradle/libs.versions.toml, watch/build.gradle.kts
watch/core/  (Kotlin JVM)  dev.personal.tandem.core.{protocol,outbox,sim}
watch/app/   (Android)     dev.personal.tandem.{ble,health,data,service,ui,probe}
.github/workflows/{android.yml,swift-core.yml,ios-ipa.yml,e2e.yml,release.yml}
docs/{PROTOCOLO,INSTALACION,INVESTIGACION,PRUEBAS}.md, CHANGELOG.md
```

---

### Task 1: Esqueleto de verificación y vectores del protocolo

**Files:**
- Create: `scripts/verify.sh`, `protocol/uuids.json`, `protocol/vectors/frames.json`, `protocol/vectors/fragments.json`, `protocol/vectors/messages.json`, `protocol/check_vectors.py`, `protocol/test_check_vectors.py`, `docs/PROTOCOLO.md`

**Interfaces:**
- Produces: formato de vectores. `frames.json`: `[{name, version, type, msgId, payloadHex, encodedHex}]`. `fragments.json`: `[{name, encodedHex, mtu, fragmentsHex: [..]}]`. `messages.json`: `[{name, type, fields: {...}, payloadHex}]`. `verify.sh` acepta `--only <python|swift|android|e2e>` y termina con una tabla `paso | OK/FALLO/OMITIDO (motivo)`; sale con 1 si algo falla, 0 si todo pasó u omitió.

Trama (en `docs/PROTOCOLO.md`, español): `version u8 (=1) · type u8 · msgId u32 · payloadLen u32 · payload · crc32 u32` (CRC sobre todo lo anterior). Fragmento: `msgId u32 · index u16 · count u16 · chunk`, chunk ≤ `mtu - 3 - 8`. Tipos: `0x01 HELLO`, `0x02 HELLO_ACK`, `0x10 RECORD`, `0x11 ACK`, `0x12 NACK`, `0x20 PING`. Payloads: HELLO/HELLO_ACK = `protoMajor u8 · protoMinor u8 · maxMessage u32 · nameLen u8 · name utf8`; RECORD = `recordId u64 · kind u8 (1=hr) · count u16 · count×(epochMs i64 · bpm u16)`; ACK = `recordId u64`; NACK = `recordId u64 · reason u8 (1=healthDenied, 2=writeFailed, 3=malformed)`; PING = vacío. Reglas de compatibilidad: tipos desconocidos → NACK `malformed` sin cortar el enlace; versión de trama distinta → descartar y registrar.

- [ ] **Step 1: Escribir prueba que falla** — `protocol/test_check_vectors.py::test_all_vectors_valid` llama `check_all("protocol/vectors")` y espera `[]` errores; `test_detects_bad_crc` altera un byte de un `encodedHex` y espera 1 error con `"crc"` en el texto.
- [ ] **Step 2:** `python3 -m unittest protocol/test_check_vectors.py` → FALLA (no existe `check_vectors`).
- [ ] **Step 3:** Implementar `check_all(dir) -> list[str]` en `protocol/check_vectors.py` (recalcula CRC con `zlib.crc32`, longitudes y que la concatenación de fragmentos sin cabecera = `encodedHex`). Escribir los vectores: al menos `ping-empty`, `hello-watch` (nombre "Galaxy Watch7"), `record-hr-1`, `record-hr-100` (100 muestras), `ack`, `nack-health-denied`; fragmentos `frag-mtu23` (record-hr-1 con mtu 23) y `frag-mtu185`.
- [ ] **Step 4:** Crear `scripts/verify.sh` con el paso `python` (corre el unittest y `check_vectors.py`) y omite con motivo `swift`, `android`, `e2e` si faltan `swift` / `java`+`ANDROID_HOME` / ambos. Correr `bash scripts/verify.sh` → `python OK`, resto `OMITIDO`, salida 0.
- [ ] **Step 5:** Commit `feat(protocol): formato v1, vectores compartidos y verify.sh`.

### Task 2: CI base de Python y Swift

**Files:**
- Create: `.github/workflows/swift-core.yml`
- Test: el propio workflow

**Interfaces:**
- Produces: job `swift-core` en Ubuntu: `actions/checkout`, `swift-actions/setup-swift` (Swift 6.3), `python3 -m unittest protocol/test_check_vectors.py`, `swift test --package-path ios/Core`. Se dispara en `push` y `pull_request` con `paths: [ios/Core/**, protocol/**, .github/workflows/swift-core.yml]`.

- [ ] **Step 1:** Escribir el workflow (aún sin paquete Swift: el paso `swift test` se agrega en Task 3; aquí solo Python).
- [ ] **Step 2:** Validar sintaxis con `python3 -c "import yaml,sys;yaml.safe_load(open(sys.argv[1]))" .github/workflows/swift-core.yml` (instalar PyYAML en el entorno de desarrollo si falta, no en el repo).
- [ ] **Step 3:** Commit `ci: workflow de protocolo y núcleo Swift`.

### Task 3: TandemCore — trama y fragmentación (Swift)

**Files:**
- Create: `ios/Core/Package.swift` (swift-tools 6.0, productos `TandemCore` y ejecutable `tandem-sim-phone` vacío por ahora), `ios/Core/Sources/TandemCore/Frame.swift`, `Fragmenter.swift`
- Test: `ios/Core/Tests/TandemCoreTests/FrameTests.swift`, `FragmenterTests.swift`, `VectorLoader.swift` (lee `../../protocol/vectors` relativo a `#filePath`)
- Modify: `.github/workflows/swift-core.yml` (añadir `swift test`), `scripts/verify.sh` (paso `swift`)

**Interfaces:**
- Produces:
  - `public struct Frame: Equatable, Sendable { public let type: UInt8; public let msgId: UInt32; public let payload: [UInt8] }`
  - `public enum FrameError: Error, Equatable { case tooShort, badVersion(UInt8), lengthMismatch, badCRC }`
  - `public func encodeFrame(_ f: Frame) -> [UInt8]`; `public func decodeFrame(_ bytes: [UInt8]) throws(FrameError) -> Frame`
  - `public func fragment(_ encoded: [UInt8], msgId: UInt32, mtu: Int) -> [[UInt8]]` (precondición `mtu >= 23`)
  - `public struct Reassembler { public init(); public mutating func accept(_ fragment: [UInt8]) -> [UInt8]?; public mutating func reset() }`

- [ ] **Step 1: Pruebas que fallan:** `test_encodeMatchesVectors` (todos los `frames.json`: `encodeFrame == encodedHex`), `test_decodeRoundTrip`, `test_decodeBadCRCThrows` (`.badCRC`), `test_decodeBadVersionThrows` (`.badVersion(2)`), `test_fragmentMatchesVectors` (`fragments.json`), `test_reassembleOutOfOrderFragments`, `test_reassembler_resetDropsPartial` (tras `reset()`, los fragmentos restantes del mensaje viejo no producen nada y un mensaje nuevo completo sí).
- [ ] **Step 2:** `swift test --package-path ios/Core` → FALLA.
- [ ] **Step 3:** Implementar. CRC-32 propio (tabla, polinomio `0xEDB88320`) porque Foundation en Linux no trae zlib de forma portable; documentarlo en un comentario.
- [ ] **Step 4:** `swift test --package-path ios/Core` → PASA; `bash scripts/verify.sh --only swift` → OK.
- [ ] **Step 5:** Commit `feat(core-ios): trama v1 y fragmentación`.

### Task 4: TandemCore — mensajes, mapeo a Salud, ledger y sesión del teléfono

**Files:**
- Create: `ios/Core/Sources/TandemCore/{Messages,HealthMapping,Ledger,PhoneSession,Log}.swift`
- Test: `MessagesTests.swift`, `HealthMappingTests.swift`, `LedgerTests.swift`, `PhoneSessionTests.swift`

**Interfaces:**
- Consumes: Task 3.
- Produces:
  - `public enum Message: Equatable, Sendable { case hello(Hello), helloAck(Hello), record(Record), ack(recordId: UInt64), nack(recordId: UInt64, reason: NackReason), ping }`; `encode() -> Frame`-compatible: `public func encodePayload(_ m: Message) -> (type: UInt8, payload: [UInt8])`, `public func decodeMessage(_ f: Frame) throws(MessageError) -> Message` (`MessageError.unknownType(UInt8)`, `.malformed`).
  - `public struct Hello { protoMajor: UInt8; protoMinor: UInt8; maxMessage: UInt32; name: String }`; `public struct Record { recordId: UInt64; kind: RecordKind; samples: [HeartRateSample] }`; `public enum RecordKind: UInt8 { case heartRate = 1 }`; `public struct HeartRateSample { epochMs: Int64; bpm: UInt16 }`; `public enum NackReason: UInt8 { case healthDenied = 1, writeFailed = 2, malformed = 3 }`.
  - `public struct HeartRateSampleSpec: Equatable { start: Date; end: Date; bpm: Double; syncIdentifier: String; syncVersion: Int }`; `public func heartRateSpecs(for r: Record) -> [HeartRateSampleSpec]` (start = end = epochMs).
  - `public protocol LedgerStore: Sendable { func contains(_ syncId: String, version: Int) async -> Bool; func insert(_ syncIds: [String], version: Int) async }`; `public actor InMemoryLedger: LedgerStore`.
  - `public protocol HealthSink: Sendable { func writeHeartRate(_ specs: [HeartRateSampleSpec]) async throws(HealthWriteError) }`; `public enum HealthWriteError: Error { case denied, failed(String) }`.
  - `public actor PhoneSession { public init(ledger: LedgerStore, health: HealthSink, log: LogSink); public func handle(_ m: Message) async -> [Message] }` — devuelve las respuestas a enviar.
  - `public protocol LogSink: Sendable { func log(_ level: LogLevel, _ message: String) }`.

- [ ] **Step 1: Pruebas que fallan:**
  - `test_messagesMatchVectors` (todo `messages.json` en ambos sentidos).
  - `test_unknownTypeIsReported` → `.unknownType(0x7F)`.
  - `test_heartRateSpecs_syncIdentifier` → para `epochMs 1764000000000`, `syncIdentifier == "tandem:hr:1764000000000"`, `syncVersion == 1`, `bpm == 72`.
  - `test_session_helloIsAnsweredWithHelloAck`.
  - `test_session_recordWritesThenAcks` → `health` recibe 1 llamada con N specs; respuesta `[.ack(recordId: 7)]`.
  - `test_session_duplicateRecordIsAckedWithoutWriting` → el mismo RECORD dos veces: 1 sola escritura, 2 ACK.
  - `test_session_partialDuplicateWritesOnlyNewSamples` → RECORD con 3 muestras de las que 2 ya están en el ledger: se escribe 1.
  - `test_session_writeFailureSendsNackAndKeepsRecord` → `.denied` ⇒ `[.nack(recordId:, reason: .healthDenied)]` y el ledger no cambia.
  - `test_session_logNeverContainsSampleValues`: el `LogSink` falso no recibe la cadena `"bpm"` con valores — solo conteos.
- [ ] **Step 2:** `swift test --package-path ios/Core` → FALLA.
- [ ] **Step 3:** Implementar (Swift 6, concurrencia estricta; ver skill `swift-concurrency-6-2` de ECC y `swift-protocol-di-testing` para los dobles).
- [ ] **Step 4:** `swift test --package-path ios/Core` → PASA.
- [ ] **Step 5:** Commit `feat(core-ios): mensajes, mapeo a HealthKit, ledger y sesión`.

### Task 5: App de iOS + IPA con la prueba de HealthKit (entrega temprana)

**Files:**
- Create: `ios/project.yml`, `ios/App/Tandem.entitlements`, `ios/App/TandemApp.swift`, `ios/App/AppModel.swift`, `ios/App/HealthKitWriter.swift`, `ios/App/FileLedgerStore.swift`, `ios/App/AppLog.swift`, `ios/App/Views/{WelcomeView,HealthProbeView,LogView}.swift`, `.github/workflows/ios-ipa.yml`

**Interfaces:**
- Consumes: Task 4 (`HealthSink`, `LedgerStore`, `LogSink`, `HeartRateSampleSpec`).
- Produces:
  - `final class HealthKitWriter: HealthSink` — pide autorización de escritura de `heartRate` (y, para fases futuras, nada más todavía); guarda `HKQuantitySample` con `HKMetadataKeySyncIdentifier`/`HKMetadataKeySyncVersion`.
  - `actor FileLedgerStore: LedgerStore` — JSON en `Application Support/ledger.json`.
  - `final class AppLog: LogSink` — anillo de 5 000 líneas en `Application Support/log.txt`; `exportURL() -> URL`.
  - `HealthProbeView`: botón **"Probar HealthKit"** → pide permiso, escribe 1 muestra de pulso de prueba (70 lpm, ahora, syncId `tandem:probe:<epochMs>`) y muestra uno de: "✅ HealthKit funciona: se guardó una muestra de prueba en Salud", "❌ La firma no incluye HealthKit (entitlement ausente)", "⚠️ Permiso denegado: actívalo en Ajustes > Salud > Acceso a datos". Detecta entitlement ausente por el error `HKError.errorAuthorizationDenied`/`errorHealthDataUnavailable` y por `HKHealthStore.isHealthDataAvailable()`; el texto exacto del error va al log.
  - `project.yml`: target `Tandem`, `PRODUCT_BUNDLE_IDENTIFIER: dev.personal.tandem`, `INFOPLIST_KEY_CFBundleDisplayName: Tándem`, deployment 17.0, `UIBackgroundModes: [bluetooth-central]`, `NSBluetoothAlwaysUsageDescription`, `NSHealthUpdateUsageDescription`, `NSHealthShareUsageDescription` (en español), entitlements `com.apple.developer.healthkit: true`, `com.apple.developer.healthkit.access: []`, dependencia local `Core` → `TandemCore`.
  - `ios-ipa.yml` (macOS, `workflow_dispatch` + `push` tags `v*` + `push`/`pull_request` con `paths: [ios/**, protocol/**, .github/workflows/ios-ipa.yml]`): instala XcodeGen 2.48.0 (Homebrew), `xcodegen generate`, `xcodebuild -scheme Tandem -configuration Release -sdk iphoneos -derivedDataPath build CODE_SIGNING_ALLOWED=NO build`, copia a `Payload/`, `codesign --force --sign - --entitlements ios/App/Tandem.entitlements Payload/Tandem.app`, **verifica** con `codesign -d --entitlements :- Payload/Tandem.app | grep -q com.apple.developer.healthkit` (falla el job si falta), zip → `Tandem-unsigned.ipa`, sube artefacto.

- [ ] **Step 1:** Pruebas de lo que sí es lógica (en `ios/Core`, no en la app): ninguna nueva; la app es pegamento. Verificación = CI de macOS.
- [ ] **Step 2:** Escribir archivos; `xcodegen` no corre en Linux: validar `project.yml` con `python3 -c "import yaml;yaml.safe_load(open('ios/project.yml'))"`.
- [ ] **Step 3:** Push; correr `ios-ipa.yml` → artefacto `Tandem-unsigned.ipa` y paso de verificación de entitlements en verde. Si falla, usar `superpowers:systematic-debugging` con el log del job.
- [ ] **Step 4:** Publicar **pre-release `v0.0.1-hk`** con el IPA y avisar al usuario con los pasos de la prueba (van en `docs/PRUEBAS.md`, prueba P0-1). **No esperar** su resultado para seguir; si informa fallo, detenerse según `docs/DISEÑO.md`.
- [ ] **Step 5:** Commit `feat(ios): app mínima con prueba de HealthKit e IPA en CI`.

### Task 6: Proyecto Gradle del reloj y CI de Android

**Files:**
- Create: `watch/settings.gradle.kts`, `watch/build.gradle.kts`, `watch/gradle/libs.versions.toml`, `watch/gradle/wrapper/*` (wrapper 9.9.2), `watch/gradlew`, `watch/gradlew.bat`, `watch/core/build.gradle.kts` (Kotlin JVM, JUnit 5), `watch/app/build.gradle.kts` (Android app, `namespace`/`applicationId` `dev.personal.tandem`, Compose, sin `INTERNET`), `watch/app/src/main/AndroidManifest.xml` (`<uses-feature android:name="android.hardware.type.watch"/>`, app standalone `com.google.android.wearable.standalone=true`), `.github/workflows/android.yml`
- Modify: `scripts/verify.sh` (paso `android`: `./gradlew :core:test :app:testDebugUnitTest :app:lintDebug`)

**Interfaces:**
- Produces: `android.yml` (Ubuntu, JDK 21): `./gradlew :core:test :app:testDebugUnitTest :app:lintDebug :app:assembleRelease`; si los 4 secrets existen, decodifica la keystore a `$RUNNER_TEMP` y firma; si no, sube el APK sin firmar y escribe en el resumen del job "APK sin firmar: faltan los secrets (ver docs/INSTALACION.md)". `release.yml` (tags `v*`) **exige** los secrets y adjunta `Tandem-<tag>.apk` y el IPA del `ios-ipa.yml` al Release (pre-release si el tag contiene `-`).

- [ ] **Step 1:** Prueba trivial que falla en `:core` (`ProtocolVersionTest.currentVersionIsOne` sobre `const val PROTOCOL_VERSION`).
- [ ] **Step 2:** `cd watch && ./gradlew :core:test` → FALLA; implementar la constante → PASA.
- [ ] **Step 3:** `./gradlew :app:assembleDebug :app:lintDebug` → OK (instalar Android SDK en el entorno de desarrollo fuera del repo si falta; anotar cómo en `docs/INSTALACION.md`).
- [ ] **Step 4:** `bash scripts/verify.sh --only android` → OK.
- [ ] **Step 5:** Commit `build(watch): proyecto Gradle, CI de Android y release`.

### Task 7: watch/core — trama y fragmentación (Kotlin)

**Files:**
- Create: `watch/core/src/main/kotlin/dev/personal/tandem/core/protocol/{Frame,Fragmenter,Crc32}.kt`
- Test: `watch/core/src/test/kotlin/.../protocol/{FrameTest,FragmenterTest,Vectors}.kt` (lee `../../protocol/vectors`)

**Interfaces:**
- Produces: `data class Frame(val type: UByte, val msgId: UInt, val payload: ByteArray)` (equals/hashCode por contenido); `sealed class FrameError : Exception` (`TooShort`, `BadVersion(v)`, `LengthMismatch`, `BadCrc`); `fun encodeFrame(f: Frame): ByteArray`; `fun decodeFrame(bytes: ByteArray): Frame` (lanza `FrameError`); `fun fragment(encoded: ByteArray, msgId: UInt, mtu: Int): List<ByteArray>`; `class Reassembler { fun accept(fragment: ByteArray): ByteArray?; fun reset() }`.

- [ ] **Step 1:** Mismas pruebas que Task 3 con nombres Kotlin (`encodeMatchesVectors`, `decodeBadCrcThrows`, `fragmentMatchesVectors`, `reassembleOutOfOrder`, `reassembler_resetDropsPartial`).
- [ ] **Step 2:** `./gradlew :core:test` → FALLA.
- [ ] **Step 3:** Implementar (usar `java.util.zip.CRC32`).
- [ ] **Step 4:** `./gradlew :core:test` → PASA.
- [ ] **Step 5:** Commit `feat(core-watch): trama v1 y fragmentación`.

### Task 8: watch/core — mensajes y motor de la cola

**Files:**
- Create: `.../core/protocol/Messages.kt`, `.../core/outbox/{OutboxEntry,OutboxStore,SyncEngine,Transport,Clock}.kt`
- Test: `MessagesTest.kt`, `SyncEngineTest.kt`, fakes en `src/test/.../fakes/`

**Interfaces:**
- Consumes: Task 7.
- Produces:
  - `sealed interface Message` espejo de Task 4 (`Hello`, `HelloAck`, `Record(recordId: ULong, kind: RecordKind, samples: List<HeartRateSample>)`, `Ack`, `Nack`, `Ping`); `fun encodeMessage(m: Message, msgId: UInt): Frame`; `fun decodeMessage(f: Frame): Message` (lanza `UnknownType`, `Malformed`).
  - `data class OutboxEntry(val id: Long, val kind: RecordKind, val payload: ByteArray, val createdAtMs: Long, val attempts: Int, val ackedAtMs: Long?)`
  - `interface OutboxStore { suspend fun enqueue(kind: RecordKind, payload: ByteArray, nowMs: Long): Long; suspend fun pending(limit: Int): List<OutboxEntry>; suspend fun markAcked(id: Long, nowMs: Long); suspend fun bumpAttempt(id: Long); suspend fun purgeAckedOlderThan(cutoffMs: Long) }`
  - `interface Transport { val connected: StateFlow<Boolean>; val maxPayload: StateFlow<Int>; suspend fun send(bytes: ByteArray); val incoming: Flow<ByteArray> }`
  - `interface Clock { fun nowMs(): Long }`
  - `class SyncEngine(store: OutboxStore, transport: Transport, clock: Clock, maxInFlight: Int = 4, ackTimeoutMs: Long = 10_000)` con `suspend fun run()` (cancelable). Reglas: al conectar envía HELLO; envía pendientes en orden de `createdAtMs` con como mucho `maxInFlight` sin ACK; ACK → `markAcked`; NACK o timeout → `bumpAttempt` y reintento con espera `min(2^attempts s, 300 s)`; al desconectar, todo lo en vuelo vuelve a pendiente; purga acked > 7 días una vez por conexión.
  - `fun heartRateBatch(samples: List<HeartRateSample>): ByteArray` (payload de RECORD sin recordId; el recordId es el `id` de la cola).

- [ ] **Step 1: Pruebas que fallan:** `messagesMatchVectors`; `engine_sendsHelloOnConnect`; `engine_marksAckedOnlyAfterAck`; `engine_nackRetriesWithBackoff` (reloj falso: 2 s, 4 s); `engine_inFlightReturnsToPendingOnDisconnect`; `engine_drainsBacklogInOrderWithBoundedInFlight` (500 entradas, nunca más de 4 sin ACK, orden estricto); `engine_purgesAckedOlderThan7Days`. Usar `kotlinx-coroutines-test` (`runTest`, tiempo virtual).
- [ ] **Step 2:** `./gradlew :core:test` → FALLA.
- [ ] **Step 3:** Implementar.
- [ ] **Step 4:** `./gradlew :core:test` → PASA.
- [ ] **Step 5:** Commit `feat(core-watch): mensajes y motor de cola con ACK`.

### Task 9: App del reloj — cola Room, pulso, BLE periférico y servicio

**Files:**
- Create: `watch/app/src/main/kotlin/dev/personal/tandem/data/{OutboxDb,OutboxDao,RoomOutboxStore}.kt`, `.../health/{HeartRateSource,HealthServicesHeartRateSource,PassiveHeartRateService}.kt`, `.../ble/{GattPeripheral,BleTransport,Uuids}.kt`, `.../service/{BridgeService,BootReceiver}.kt`, manifiesto
- Test: `watch/app/src/test/.../data/RoomOutboxStoreTest.kt` (Robolectric, Room en memoria), `.../health/HeartRateBatcherTest.kt`

**Interfaces:**
- Consumes: Task 8 (`OutboxStore`, `Transport`, `SyncEngine`, `heartRateBatch`).
- Produces:
  - `RoomOutboxStore : OutboxStore` (tabla `outbox`, índice `(acked_at, created_at)`).
  - `interface HeartRateSource { fun live(): Flow<HeartRateSample> }`; `HealthServicesHeartRateSource` usa `MeasureClient` (con la app/servicio activo); `PassiveHeartRateService : PassiveListenerService` registra `DataType.HEART_RATE_BPM` en segundo plano.
  - `class HeartRateBatcher(windowMs: Long = 60_000)`: junta muestras y emite un lote por minuto → `OutboxStore.enqueue`.
  - `BleTransport : Transport` sobre `BluetoothGattServer`: servicio con los UUIDs de Global Constraints, permisos `PERMISSION_READ_ENCRYPTED`/`PERMISSION_WRITE_ENCRYPTED`, CCCD en `tx`, MTU de `onMtuChanged`, fragmenta con `fragment()` y reensambla con `Reassembler`; `reset()` al desconectar. Anuncia el UUID del servicio y **solicita ANCS** (`7905F431-B5CE-4E99-A40F-4B1E122D00D0`) en los datos de anuncio.
  - `BridgeService`: servicio en primer plano (`connectedDevice|health`), notificación persistente "Tándem activo", arranca `SyncEngine.run()`; `BootReceiver` lo inicia en `BOOT_COMPLETED`.
  - Permisos: `BLUETOOTH_ADVERTISE`, `BLUETOOTH_CONNECT`, `android.permission.health.READ_HEART_RATE`, `android.permission.health.READ_HEALTH_DATA_IN_BACKGROUND`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_CONNECTED_DEVICE`, `FOREGROUND_SERVICE_HEALTH`, `RECEIVE_BOOT_COMPLETED`, `POST_NOTIFICATIONS`, `WAKE_LOCK`. Sin `INTERNET`.

- [ ] **Step 1: Pruebas que fallan:** `roomStore_pendingOrderedByCreatedAt`, `roomStore_markAckedExcludesFromPending`, `roomStore_survivesReopen` (archivo temporal), `batcher_emitsOneBatchPerWindow`, `batcher_flushOnStop`, `manifest_hasNoInternetPermission` (lee el manifiesto fusionado de `build/intermediates` o, más simple, parsea `src/main/AndroidManifest.xml`).
- [ ] **Step 2:** `./gradlew :app:testDebugUnitTest` → FALLA.
- [ ] **Step 3:** Implementar. Lo no probable en JVM (GATT, Health Services reales) queda detrás de las interfaces y se documenta en `docs/PRUEBAS.md` (P0-3, P0-4).
- [ ] **Step 4:** `./gradlew :app:testDebugUnitTest :app:lintDebug` → PASA.
- [ ] **Step 5:** Commit `feat(watch): cola Room, pulso, periférico BLE y servicio en primer plano`.

### Task 10: Pantallas del reloj, central BLE del iPhone y estado

**Files:**
- Create (reloj): `.../ui/{MainActivity,PairingScreen,StatusScreen,StatusViewModel}.kt`, strings en `res/values/strings.xml` (español)
- Create (iPhone): `ios/App/BleCentral.swift`, `ios/App/Views/StatusView.swift`; Modify `AppModel.swift`, `TandemApp.swift`
- Test: `watch/app/src/test/.../ui/StatusViewModelTest.kt`; `ios/Core/Tests/TandemCoreTests/ConnectionStateTests.swift`
- Create (Core): `ios/Core/Sources/TandemCore/ConnectionState.swift`

**Interfaces:**
- Consumes: Tasks 4, 8, 9.
- Produces:
  - Core: `public enum ConnectionState: Equatable { case unpaired, scanning, connecting, connected(since: Date), reconnecting(lastSeen: Date), bluetoothOff, unauthorized }`; `public func statusText(_ s: ConnectionState, now: Date) -> String` (español: "Conectado al reloj desde hace 5 min", "Buscando el reloj…", "Bluetooth apagado: actívalo en el Centro de control", "Sin permiso de Bluetooth: actívalo en Ajustes > Tándem").
  - `final class BleCentral: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate` con `CBCentralManagerOptionRestoreIdentifierKey: "dev.personal.tandem.central"`; en `willRestoreState` recupera el periférico y resuscribe; tras conectar guarda el `identifier` y en cada desconexión llama `connect(_:options:)` de nuevo (sin timeout) con `CBConnectPeripheralOptionRequiresANCS: true`; busca por UUID de servicio; negocia `maximumWriteValueLength(for: .withResponse)`; pasa bytes a `Reassembler` → `decodeFrame` → `PhoneSession.handle` → respuestas fragmentadas a `rx`.
  - Reloj: `StatusViewModel(engineState: StateFlow<EngineState>, pending: Flow<Int>)` → `StatusUi(title, detail, pendingCount)`; textos "Conectado", "Esperando al iPhone", "N datos pendientes".
- [ ] **Step 1: Pruebas que fallan:** `test_statusText_connectedMinutes` ("Conectado al reloj desde hace 5 min" con `since = now - 300`), `test_statusText_bluetoothOff`; Kotlin `statusVm_showsPendingCount` (3 → "3 datos pendientes"), `statusVm_disconnectedText`.
- [ ] **Step 2:** Correr ambas suites → FALLA.
- [ ] **Step 3:** Implementar; la UI SwiftUI y Compose no lleva más lógica que la de estos modelos.
- [ ] **Step 4:** `swift test --package-path ios/Core` y `./gradlew :app:testDebugUnitTest` → PASA; `ios-ipa.yml` en verde.
- [ ] **Step 5:** Commit `feat: estado de conexión, central BLE con restauración y pantallas`.

### Task 11: Simulador de enlace de punta a punta

**Files:**
- Create: `watch/core/src/main/kotlin/dev/personal/tandem/core/sim/{SimWatchMain,TcpTransport,FaultyTransport}.kt` (+ `application` en `watch/core/build.gradle.kts`, `mainClass` `dev.personal.tandem.core.sim.SimWatchMainKt`), `ios/Core/Sources/tandem-sim-phone/main.swift`, `scripts/sim-e2e.sh`, `.github/workflows/e2e.yml`
- Modify: `scripts/verify.sh` (paso `e2e`, requiere `swift` y `java`)

**Interfaces:**
- Consumes: Tasks 4, 8.
- Produces:
  - `tandem-sim-phone --port <p> --health ok|denied --out result.json`: escucha TCP, usa `PhoneSession` con `InMemoryLedger` y un `HealthSink` en memoria; al cerrar escribe `{"written": <n>, "unique": <n>, "duplicatesWritten": <n>, "acks": <n>, "nacks": <n>}`.
  - `SimWatchMain --port <p> --scenario <name> --samples <n>`: usa `SyncEngine` + `InMemoryOutboxStore` + `TcpTransport` envuelto en `FaultyTransport(mtu, dropEvery, disconnectAfterBytes)` (cada envío TCP lleva un prefijo de longitud u16 para simular paquetes BLE).
  - Escenarios: `happy` (mtu 185), `mtu23`, `drop-mid-message` (desconecta a mitad de un mensaje fragmentado y reconecta), `lost-ack` (el teléfono procesa pero el ACK se pierde una vez), `backlog` (2 000 muestras), `health-denied` (espera 0 escritos, NACK, cola intacta).
  - `scripts/sim-e2e.sh` corre cada escenario y verifica con `python3`: `unique == samples`, `duplicatesWritten == 0` (y para `health-denied`: `written == 0` y el reloj termina con todo pendiente).
- [ ] **Step 1:** Escribir `sim-e2e.sh` con las aserciones de todos los escenarios → falla (no existen los binarios).
- [ ] **Step 2:** Implementar ambos lados.
- [ ] **Step 3:** `bash scripts/sim-e2e.sh` → todos los escenarios `OK`; `e2e.yml` (Ubuntu, JDK 21 + Swift 6.3) en verde.
- [ ] **Step 4:** Commit `test: simulador de enlace reloj↔iPhone con fallas inyectadas`.

### Task 12: Sondas de viabilidad para la ronda real

**Files:**
- Create: `watch/app/src/main/kotlin/dev/personal/tandem/probe/{ProbeScreen,AdvertisingProbe,AncsProbe,SleepStateProbe}.kt`; `ios/App/Views/DiagnosticsView.swift` (une `HealthProbeView` y `LogView`)
- Test: `watch/app/src/test/.../probe/ProbeReportTest.kt`

**Interfaces:**
- Produces: `data class ProbeResult(val id: String, val status: Status /* VIABLE, PARTIAL, NOT_VIABLE, NOT_RUN */, val evidence: String)`; `fun formatReport(results: List<ProbeResult>, wearOsVersion: String, nowMs: Long): String` (texto en español, también va a logcat con la etiqueta `TandemProbe` para copiarlo a `docs/INVESTIGACION.md`).
  - `AdvertisingProbe`: `BluetoothAdapter.isMultipleAdvertisementSupported` y si arranca un anuncio de prueba.
  - `AncsProbe`: con el iPhone conectado y vinculado, `connectGatt` al dispositivo central, descubre servicios, busca ANCS `7905F431-…` y se suscribe a *Notification Source* `9FBF120D-6301-42D9-8C58-25E699A21DBD`; cuenta notificaciones en 60 s. Igual para AMS `89D3502B-0F36-433A-8EF4-C502AD55F8DC` (solo descubrimiento).
  - `SleepStateProbe`: `PassiveMonitoringClient.getCapabilitiesAsync()` → si `supportsUserActivityState` y registra `UserActivityInfo`; guarda transiciones a `USER_ACTIVITY_ASLEEP` durante una noche.
- [ ] **Step 1: Prueba que falla:** `report_listsAllProbesInSpanish` (contiene "Anuncio BLE", "ANCS", "AMS", "Estado dormido" y "Wear OS 6") y `report_marksNotRun`.
- [ ] **Step 2–4:** FALLA → implementar → `./gradlew :app:testDebugUnitTest` PASA.
- [ ] **Step 5:** Commit `feat(watch): sondas de viabilidad (anuncio, ANCS, AMS, sueño)`.

### Task 13: Documentación de la fase

**Files:**
- Create: `docs/INSTALACION.md`, `docs/INVESTIGACION.md`, `docs/PRUEBAS.md`, `scripts/battery.sh`, `CHANGELOG.md`; Modify: `README.md` (crear, versión inicial corta), `docs/DECISIONES.md`

Contenido exigido:
- `INSTALACION.md` (Windows, paso a paso): instalar JDK 21, Android SDK platform-tools (adb), Python 3 y Git for Windows; crear la keystore **una vez** con `keytool -genkeypair -v -keystore tandem.jks -alias tandem -keyalg RSA -keysize 4096 -validity 10000` fuera del repo, respaldo, convertir a base64 con PowerShell `[Convert]::ToBase64String([IO.File]::ReadAllBytes("tandem.jks"))` y cargar los 4 secrets en GitHub (Settings > Secrets and variables > Actions); emparejar depuración inalámbrica del reloj (Ajustes > Opciones de desarrollador > Depuración inalámbrica > Emparejar con código → `adb pair IP:PUERTO`, luego `adb connect IP:PUERTO`); verificar la versión con `adb shell getprop ro.build.version.release` y `adb shell getprop ro.build.version.sdk`; instalar con `adb install -r Tandem-*.apk`; descargar el IPA del Release y firmarlo en FlekStore (Signer > importar IPA > firmar > instalar).
- `INVESTIGACION.md`: tabla de viabilidad de la fase 0 (HealthKit con firma de FlekStore, reconexión en segundo plano, ANCS, AMS, estado dormido, anuncio BLE en el Galaxy, audio de llamadas = no viable, Siri = probable no viable) con columnas `punto | según documentación | estado | evidencia/enlace`; estado inicial "pendiente de prueba real" salvo los no viables; enlaces oficiales de `docs/investigacion/plataformas-2026-11-25.md`.
- `PRUEBAS.md`: P0-1 HealthKit (5 min, al inicio), P0-2 emparejamiento con código de 6 dígitos, P0-3 pulso en vivo llega a Salud sin duplicados (comparar en Salud > Ritmo cardiaco > Mostrar todos los datos), P0-4 reconexión 24 h (alejarse 10 min, reiniciar el iPhone, bloquear el iPhone 8 h), P0-5 sondas (copiar el informe `TandemProbe` de logcat), P0-6 batería (`scripts/battery.sh` 2 h en reposo conectado; presupuesto ≤ 1,5 %/h extra). Cada prueba: requisitos, pasos numerados, resultado esperado, dónde anotar el resultado.
- `scripts/battery.sh`: `adb shell dumpsys batterystats --reset`, espera N minutos, lee el nivel inicial/final de `dumpsys battery` y escribe `%/h`.
- `CHANGELOG.md`: sección `## v0.1.0 — Fase 0` en español.

- [ ] **Step 1:** Escribir los documentos.
- [ ] **Step 2:** `bash scripts/verify.sh` → todo OK u OMITIDO con motivo.
- [ ] **Step 3:** Commit `docs: instalación, investigación, pruebas y changelog de la fase 0`.

### Task 14: Cierre de la fase

- [ ] **Step 1:** `replica-test` (adaptado: casos por flujo F01/F02-pulso/F07/F09 y errores por severidad) → `replica/test-fase0.md`.
- [ ] **Step 2:** `replica-diff` → `replica/diff-fase0.md` y actualizar la columna `clone` de `replica/features.csv`.
- [ ] **Step 3:** Revisión con `superpowers:requesting-code-review` usando los agentes de ECC `code-reviewer`, `kotlin-reviewer`, `security-reviewer` (y `kotlin-build-resolver` si algo no compila); comandos de ECC `code-review`, `security-scan`, `quality-gate`. Corregir hallazgos reales.
- [ ] **Step 4:** `superpowers:verification-before-completion`: `bash scripts/verify.sh` en verde + todos los workflows del PR en verde (`android`, `swift-core`, `e2e`, `ios-ipa`).
- [ ] **Step 5:** `superpowers:finishing-a-development-branch`; tag `v0.1.0-fase0` → Release *pre-release* con APK (firmado si hay secrets) e IPA. Aviso al usuario y paso a la fase 1.
