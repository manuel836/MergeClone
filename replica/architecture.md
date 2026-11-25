# Arquitectura (replica-architect adaptado a apps nativas)

replica-architect asume web + Postgres + API HTTP. Aquí no hay servidor: el "esquema" son las tablas locales (Room en el reloj, ledger en el iPhone) y la "API" es el protocolo BLE. Detalle técnico completo en `docs/DISEÑO.md`.

## Stack
| capa | elección | por qué |
| --- | --- | --- |
| reloj | Kotlin + Compose for Wear OS (Material 3) | nativo; único camino para BLE + Health Services |
| datos del reloj | Room 2.8.x | cola persistente con confirmación |
| salud del reloj | Health Services (ExerciseClient, PassiveMonitoringClient, MeasureClient) | API oficial; Health Connect no existe en Wear OS |
| iPhone | SwiftUI + CoreBluetooth + HealthKit | nativo |
| lógica compartida iOS | paquete Swift `ios/Core` | se prueba en Linux sin Mac |
| proyecto iOS | XcodeGen (`ios/project.yml`) | sin .xcodeproj a mano |
| CI | GitHub Actions (Ubuntu; macOS solo para el IPA) | repo público: minutos gratis |

## Esquema local
```
watch (Room)
outbox(id INTEGER PK, kind TEXT CHECK IN ('hr','workout','route','sleep','daily'),
       payload BLOB, created_at INTEGER /*epoch ms UTC*/, attempts INTEGER, acked_at INTEGER NULL)
       index (acked_at, created_at)
iphone (archivo JSON o SQLite en el contenedor de la app)
ledger(sync_identifier TEXT PK, sync_version INTEGER, written_at)   -- espejo de lo escrito en HealthKit
app_filter(app_id TEXT PK, display_name TEXT, enabled BOOL, last_seen)
paired_watch(peripheral_uuid TEXT PK, name, bonded_at, protocol_version)
```

## "API" = mensajes del protocolo
| mensaje | sentido | qué hace | fase |
| --- | --- | --- | --- |
| HELLO / HELLO_ACK | ambos | versión de protocolo, MTU efectivo | 0 |
| RECORD (hr, workout, route, sleep, daily) | reloj → iPhone | registro de la cola | 0–3 |
| ACK(record_id) | iPhone → reloj | marcar enviado tras escribir en Salud | 0 |
| FILTER_SET | iPhone → reloj | lista de apps permitidas | 1 |
| FIND_PHONE / FIND_WATCH | ambos | hacer sonar | 4 |
| CAMERA_SHUTTER | reloj → iPhone | disparar | 4 |
Notificaciones y música no pasan por nuestro protocolo: el reloj las lee directo del iPhone (ANCS, AMS).

## Partes que muerden
- **Horas:** todo en epoch ms UTC; la zona horaria solo para mostrar.
- **Idempotencia:** un RECORD puede llegar dos veces; el iPhone usa SyncIdentifier = id estable del registro y no duplica.
- **Carreras:** ACK perdido → reenvío → el ledger lo absorbe.
- **Segundo plano:** iOS suspende la app; Samsung duerme apps. Servicio en primer plano en el reloj y state restoration en iOS.
- **Batería:** presupuesto medido por fase.

## Orden de construcción
1. Esqueleto: pulso en vivo reloj → iPhone → Salud (fase 0).
2. Must: notificaciones (1), entrenamientos (2), sueño y día (3).
3. Should/could: fase 4 y 5.
4. Arreglos de replica-entrepreneur: integrados en cada fase (ver `replica/fixes.md`).
