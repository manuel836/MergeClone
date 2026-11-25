# Diseño: puente Galaxy Watch 7 ↔ iPhone

Estado: **borrador para tu aprobación** · 2026-11-25
Insumos: `replica/recon.md`, `replica/fixes.md`, `replica/architecture.md`, `replica/design/`, `docs/investigacion/plataformas-2026-11-25.md`.

## 1. Intención
Usar un Galaxy Watch 7 con un iPhone 15 Pro Max como si fueran pareja: que lleguen las notificaciones, que los entrenamientos, el sueño y las métricas del día terminen en Apple Salud, y luego el resto de lo que hace Merge. Para uso personal, sin depender de una suscripción ni de un servidor ajeno, y **más estable que Merge** en lo que más critican sus usuarios: desconexiones (71 reseñas), notificaciones que no llegan (58) y falta de sueño en Salud (20).

## 2. Objetivos, no objetivos y antiobjetivos
**Objetivos**
1. Notificaciones de todas las apps del iPhone, filtrables por app, que se pueden descartar; aceptar o rechazar llamadas.
2. Entrenamientos completos en Salud (pulso, distancia, calorías, ritmo, ruta GPS), **iniciados desde nuestra app del reloj** (una app de terceros no puede leer Samsung Health; lo aceptaste). Deportes: correr y caminar al aire libre, bicicleta al aire libre, interior (cinta, elíptica) y fuerza en gimnasio.
3. Sueño en Salud (sesiones `asleepUnspecified` y `inBed`) y métricas del día (pasos, distancia, pisos, calorías).
4. Reconexión sola, sin abrir apps, y ningún dato perdido ni duplicado.
5. Paridad razonable con Merge en lo demás (música, buscar, cámara, contactos).

**No objetivos:** Apple Watch con Android; un reloj y dos teléfonos; activación del reloj sin Android (opcional en fase 5); responder notificaciones (Merge tampoco lo hace en Wear OS y ANCS no lo permite); tiendas, cobros, cuentas, servidores, telemetría.

**Antiobjetivos** (aunque "funcione", sería un fracaso):
- Que se desconecte y haya que abrir la app para recuperarlo.
- Datos duplicados o inventados en Salud.
- Que el reloj no llegue al final del día por culpa del puente.
- Simular una función que la plataforma no permite (p. ej. audio de llamadas sin root).

## 3. Restricciones (con evidencia)
| Restricción | Consecuencia | Fuente |
| --- | --- | --- |
| Sin Mac | IPA compilado en GitHub Actions (macOS), sin firmar; firmas tú con FlekStore | tu entorno |
| Certificado comprado de FlekStore, tipo Development | **HealthKit solo funciona si su perfil lo incluye.** Primera prueba de la fase 0; si falla, me detengo. Plan B: cuenta propia de Apple Developer importada en FlekStore | investigación §6 |
| Health Services 1.1 no tiene tipo "sueño" | el reloj solo sabe "dormido/despierto"; sin fases (estimarlas es la fase 5) | investigación §1 |
| Health Connect no existe en Wear OS | todo sale de Health Services | investigación §1 |
| Audio de llamadas en el reloj exige el perfil manos libres (HFP), reservado a apps del sistema | **no viable** sin root; solo aceptar/rechazar por ANCS | investigación §3 |
| iOS no relanza una app que el usuario cerró deslizando | límite honesto, documentado en el README | Apple TN3115 |
| Wear OS 6 (API 36) en el Watch 7 | permisos `health.READ_HEART_RATE` y `health.READ_HEALTH_DATA_IN_BACKGROUND` | investigación §2 |
| Repo público | minutos de macOS gratis; aun así el IPA solo se compila si cambian `ios/` o `protocol/`, a mano o con tag | GitHub Docs |
| Claude Code corre en la nube | adb al reloj requiere tu PC | entorno |
| **Sin pruebas reales hasta el final** (decisión tuya) | todo se desarrolla y verifica con simulaciones; las pruebas en dispositivos se juntan en `docs/PRUEBAS.md` para una ronda final | tu respuesta 2026-11-25 |
| El reloj ya no está vinculado a Android | nuestro enlace no compite con Galaxy Wearable; la batería se mide contra un reloj sin otro teléfono | tu respuesta 2026-11-25 |

## 4. Enfoques considerados
| Enfoque | Qué implica | Veredicto |
| --- | --- | --- |
| **A. Reloj periférico, iPhone central** (elegido) | El reloj se anuncia con un servicio GATT propio; la app del iPhone se conecta con state restoration y reconexión sin límite de tiempo. Sobre el mismo enlace cifrado el reloj lee ANCS y AMS del iPhone. | Es el único que reconecta bien con la app del iPhone en segundo plano. |
| B. iPhone periférico, reloj central | La app del iPhone anuncia y el reloj la busca | En segundo plano iOS esconde sus anuncios (área de "overflow") y el reloj no la encuentra: empeora justo las desconexiones. Descartado. |
| C. Relé por internet | Servidor intermedio | Prohibido por alcance (sin servidores). Descartado. |
Elegido por el diseño base que diste y confirmado por la documentación. Riesgo abierto: que el Galaxy Watch 7 sea servidor GATT y cliente de ANCS a la vez sobre un mismo enlace (no documentado); se prueba en la fase 0 y, si falla, el plan es un segundo enlace en el que el reloj se conecta como central al iPhone ya vinculado solo para ANCS/AMS.

## 5. Cómo funciona

### 5.1 Repositorio
```
watch/            app Wear OS (Gradle, Kotlin)
ios/App/          app SwiftUI (fuentes)  · ios/project.yml (XcodeGen)
ios/Core/         paquete Swift: protocolo + mapeo a HealthKit, pruebas en Linux
protocol/         vectores de prueba compartidos (JSON + binario en hex)
docs/             DISEÑO, PROTOCOLO, INVESTIGACION, PRUEBAS, INSTALACION, DECISIONES, ENTREGA
replica/          salidas de replica
scripts/verify.sh verificación local (Git Bash) y en CI
.github/workflows android.yml, swift-core.yml, ios-ipa.yml
```

### 5.2 Reloj (Kotlin, Compose for Wear OS)
- **Servicio en primer plano** (tipos `connectedDevice` + `health`; `location` solo durante un entrenamiento) con notificación persistente discreta. Arranca al encender el reloj.
- **Periférico BLE:** anuncia el UUID de 128 bits del servicio propio y solicita el servicio ANCS para que iOS lo ofrezca. Servidor GATT con características que exigen cifrado.
- **Cliente ANCS/AMS** (fase 1/4) sobre el mismo enlace una vez vinculado.
- **Salud:** `MeasureClient` (pulso en vivo con la app abierta), `PassiveMonitoringClient` (pulso, pasos, distancia, pisos, calorías y estado dormido/despierto en segundo plano), `ExerciseClient` (entrenamientos con GPS).
- **Cola en Room (`outbox`):** todo dato se guarda primero; se marca enviado solo con el ACK del iPhone. Reintentos con espera creciente; se purga lo confirmado con más de 7 días.
- **Sueño:** el reloj convierte la secuencia dormido/despierto en sesiones (inicio, fin, interrupciones) con reglas probadas por TDD (umbral de siesta y fusión de despertares cortos: decisión del constructor, documentada).
- Todo el hardware (BLE, Health Services, reloj del sistema) detrás de interfaces con dobles de prueba.

### 5.3 iPhone (SwiftUI)
- **Central CoreBluetooth** con modo de fondo `bluetooth-central`, `CBCentralManagerOptionRestoreIdentifierKey` y `connect` pendiente sin tiempo límite: iOS despierta la app al volver el reloj, aunque esté suspendida. Se conecta con `CBConnectPeripheralOptionRequiresANCS` para que iOS comparta notificaciones con el reloj.
- **HealthKit:** escribe pulso, entrenamientos (`HKWorkoutBuilder` + `HKWorkoutRouteBuilder`), sueño y métricas. Cada muestra lleva `HKMetadataKeySyncIdentifier` = id estable del registro y `HKMetadataKeySyncVersion`: un reenvío no duplica y una corrección reemplaza. Solo tras escribir con éxito manda ACK.
- **Pantallas** (I01–I08 de `replica/recon.md`): bienvenida y permisos, emparejar, estado, filtro de notificaciones, salud, diagnóstico (log exportable con la hoja de compartir), ajustes.
- **ios/Core** (paquete Swift sin dependencias de Apple-only): codificador/decodificador del protocolo, fragmentación, reglas de mapeo a HealthKit expresadas como datos puros. Se prueba en Ubuntu.

### 5.4 Protocolo (detalle en `docs/PROTOCOLO.md`, fase 0)
- Servicio propio con UUID aleatorio de 128 bits; características: `rx` (escritura con respuesta, iPhone→reloj), `tx` (notify, reloj→iPhone), ambas con permiso de cifrado.
- Trama: `versión(1) · tipo(1) · id_mensaje(4) · índice_fragmento(2) · total_fragmentos(2) · longitud(2) · payload · CRC32(4)`, enteros little-endian. Fragmenta según el MTU negociado (mínimo 23).
- Mensajes: HELLO, RECORD, ACK, NACK, FILTER_SET, PING; los de la fase 4 se agregan con números nuevos sin romper los viejos (versión mayor solo si cambia la trama).
- **Vectores de prueba** en `protocol/vectors/*.json`: los mismos casos deben pasar en Kotlin y en Swift.

### 5.5 Seguridad y privacidad
- Vinculación con **LE Secure Connections, comparación numérica**: el sistema muestra el mismo código de 6 dígitos en el reloj y en el iPhone; sin código no hay datos (las características exigen enlace cifrado).
- Ningún permiso de internet en el reloj ni llamadas de red en el iPhone. Sin analítica ni telemetría. El log no guarda texto de notificaciones, solo metadatos.
- Keystore del APK creada una vez, fuera del repo, en secrets de Actions. El IPA sale con firma ad hoc (`codesign -s -`) y entitlements embebidos.

### 5.6 CI y verificación
- `android.yml` (Ubuntu): compilar, pruebas unitarias, lint, APK firmado como artefacto.
- `swift-core.yml` (Ubuntu): `swift test` del paquete `ios/Core` + vectores.
- `ios-ipa.yml` (macOS): XcodeGen + `xcodebuild` sin firmar + firma ad hoc con entitlements; solo con cambios en `ios/` o `protocol/`, a mano o con tag `v*`.
- `scripts/verify.sh`: corre lo que exista en la máquina (Gradle, Swift, Python para vectores) y **dice explícitamente qué omitió**. En tu PC con Windows lo normal es que Swift se omita (lo cubre el CI). Nada está terminado sin verify.sh y el CI en verde.
- Releases por tag con APK e IPA adjuntos.

### 5.7 Batería
Medición en cada fase con `adb shell dumpsys batterystats` y % por hora en reposo conectado. **Presupuesto inicial: ≤ 1,5 %/h extra conectado en reposo; entrenamiento con GPS fuera del presupuesto** (lo domina el GPS). Si una fase se pasa, se ajusta antes de cerrarla.

### 5.8 Diseño visual
Tokens propios en `replica/design/tokens.json` (acento verde azulado, sin relación con Merge), contraste AA verificado (18 pares, 0 fallos). iOS usa la fuente del sistema y Dynamic Type; el reloj, Material 3 for Wear OS con fondo negro. Todo el texto en español.

### 5.9 Simulación (sin dispositivos hasta el final)
Como no habrá pruebas reales hasta completar todo, la verificación descansa en tres capas:
1. **Pruebas unitarias con dobles** de BLE, Health Services, HealthKit y reloj del sistema (TDD).
2. **Vectores de protocolo compartidos**: los mismos bytes deben codificar/decodificar igual en Kotlin y Swift.
3. **Simulador de enlace de punta a punta en el CI**: la lógica del reloj corre en la JVM y la de `ios/Core` en Swift sobre Linux, conectadas por un transporte de prueba (socket local) que implementa la misma interfaz que el BLE real, con MTU pequeño, pérdidas, desconexiones y reenvíos inyectados. Prueba flujos completos: pulso → cola → envío → "escritura en Salud" simulada → ACK; reconexión; duplicados.
Lo que ninguna simulación cubre (radio real, cifrado y vinculación, permisos de iOS/Wear OS, ANCS/AMS reales, segundo plano, batería, firma) queda en `docs/PRUEBAS.md` con pasos exactos.

**Mi recomendación:** aunque el resto espere, haz al inicio una sola prueba real de 5 minutos: instalar el IPA de la fase 0 firmado con FlekStore y tocar "Probar HealthKit". Si tu certificado no incluye HealthKit, las fases 2 y 3 no sirven tal como están, y es mejor saberlo antes de construirlas. Si prefieres no hacerlo, sigo y ese riesgo queda abierto hasta el final.

## 6. Fases
Como las definiste (0 a 5), cada una en su rama y PR. **Cambio por tu decisión de no probar en dispositivos hasta el final:** cada fase se cierra con CI y verify.sh en verde, revisión y escaneo de ECC y un Release con APK e IPA; no espero tu OK de dispositivos entre fases, sino que sigo con la siguiente. Las pruebas reales de todas las fases se hacen al final con `docs/PRUEBAS.md`. Los Releases por fase se marcan como *pre-release*. Fase 0 = cimientos, CI, keystore, `docs/INSTALACION.md`, pulso en vivo a Salud y las pruebas de viabilidad (HealthKit con tu firma, reconexión en segundo plano, ANCS/AMS desde el Galaxy, estado dormido en Health Services) → `docs/INVESTIGACION.md`.

**Puntos de parada acordados** (se activan cuando se haga la prueba real correspondiente; hasta entonces `docs/INVESTIGACION.md` marca cada punto como "según documentación, pendiente de prueba real"):
- HealthKit no funciona con tu firma → me detengo y te aviso; tú decides el plan B (cuenta propia de Apple Developer u otra).
- El reloj no entrega el estado "dormido" (o lo entrega poco fiable) → el sueño de la fase 3 queda como **no viable** en `docs/INVESTIGACION.md` y te aviso; no se inventan sesiones con un estimador propio salvo que lo pidas (fase 5).
Funciones marcadas desde ya: **audio de llamadas en el reloj = no viable**; **Siri desde el reloj = probablemente no viable** (se confirma en la fase 4).

## 7. Nombre
Tres propuestas (ninguna contiene "Merge"; uso personal, sin búsqueda de marcas):
1. **Lazo** — corto, en español, "lo que une".
2. **Tándem** — dos que trabajan juntos.
3. **Órbita** — el reloj gira alrededor del teléfono.
Elige uno; mientras tanto el código usa el identificador neutro `bridge` (paquete `dev.personal.bridge`, bundle `dev.personal.bridge`).

## 8. Lo que queda a criterio del constructor
Reglas exactas de sesiones de sueño, intervalos de reintento, formato interno de payloads por tipo, diseño visual fino de cada pantalla, estructura de módulos Gradle. Cada decisión se anota en `docs/DECISIONES.md`.
