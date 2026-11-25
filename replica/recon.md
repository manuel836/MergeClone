# Mapa de recon: Merge (Wear OS ↔ iPhone)

Leído el 2026-11-25, solo de fuentes públicas (sitio, guías, tiendas). Sin descompilar ni copiar textos. Notas crudas con las 41 fuentes enlazadas: `replica/recon-fuentes.md`.
Adaptación a apps nativas: "pantallas" son pantallas del reloj (W) y del iPhone (I); no hay rutas web.

## Alcance (paso 1)
- **App y plataforma:** Merge, app de iPhone (id6698894276, v3.0.1, iOS 17.2+) + app de Wear OS (`com.mergecorp.merge`).
- **Porción:** reloj Wear OS con iPhone. Fuera: Apple Watch con Android, "un reloj y dos teléfonos", activación inalámbrica (fase 5 opcional), relé por internet.
- **Para quién:** uso personal del dueño de un Galaxy Watch 7 y un iPhone 15 Pro Max.

## Bucle central
Reloj emparejado una vez → se mantiene conectado solo → las notificaciones del iPhone llegan al reloj y los datos de salud del reloj llegan a Apple Salud.

## Pantallas
| ID | Pantalla | Cómo se llega | Propósito | Estados |
| --- | --- | --- | --- | --- |
| I01 | Bienvenida y permisos | primer arranque | Bluetooth, notificaciones, Salud | sin permiso, concedido, denegado |
| I02 | Buscar reloj | I01 → siguiente | lista de relojes cercanos | buscando, vacío, encontrado, error |
| I03 | Código de emparejamiento | tocar reloj | confirmar el mismo código en ambas pantallas | esperando, coincide, falló |
| I04 | Inicio / estado | app abierta | conexión, batería del reloj, última sincronización | conectado, reconectando, desconectado |
| I05 | Filtro de notificaciones | I04 → Notificaciones | activar/desactivar por app | vacío (aún no llegó ninguna), lista |
| I06 | Salud | I04 → Salud | qué tipos se envían, último envío, pendientes | sin permiso, al día, con cola |
| I07 | Registro (log) | I04 → Diagnóstico | log exportable | vacío, lleno |
| I08 | Ajustes | I04 → Ajustes | olvidar reloj, código, acerca de | — |
| W01 | Emparejar | primer arranque | anunciarse y mostrar código | anunciando, código, emparejado, error |
| W02 | Estado | app abierta | conexión, cola pendiente, batería | conectado, desconectado |
| W03 | Notificación | llega una | mostrar, descartar | texto, llamada entrante |
| W04 | Llamada entrante | llamada en el iPhone | aceptar / rechazar | sonando, contestada, perdida |
| W05 | Entrenamiento | lanzar | elegir tipo, en curso, resumen | GPS buscando, en curso, pausa, fin |
| W06 | Música | lanzar / tile | reproducir, pausar, saltar, volumen | sin reproducción, sonando |
| W07 | Buscar iPhone | lanzar | hacer sonar el iPhone | sonando, fuera de alcance |
| W08 | Cámara | lanzar | disparador | cámara abierta, cerrada |
| W09 | Tile + complicación | carátula | estado y accesos rápidos | — |

## Flujos
```
F01 Emparejar por primera vez
    W01 (anunciarse) -> I01 permisos -> I02 elegir reloj -> I03 mismo código en ambas pantallas -> I04 conectado
    Merge: 4–9 toques + ida a Ajustes de iOS. Meta: 5 toques, sin salir de la app.
    borde: código no coincide, reloj ya emparejado con otro, Bluetooth apagado, permiso denegado
F02 Recibir notificación
    iPhone recibe -> reloj W03 vibra -> descartar (se borra también en el iPhone)
    borde: app filtrada, reloj fuera de alcance, ráfaga de 50 notificaciones, texto largo, emoji
F03 Filtrar apps
    I05 -> desactivar app -> ya no llegan al reloj
F04 Llamada entrante
    W04 -> aceptar / rechazar; el audio sigue en el iPhone (ver límites)
F05 Entrenamiento
    W05 elegir tipo -> en curso (pulso, distancia, ritmo, GPS) -> fin -> cola -> iPhone escribe HKWorkout + ruta -> confirmación
    borde: iPhone lejos durante y al terminar, batería baja, GPS sin señal
F06 Sueño y métricas del día
    reloj registra -> cola -> iPhone escribe en Salud al reconectar, sin duplicados
F07 Reconexión
    se pierde el enlace -> al volver al alcance reconecta sola, aun con la app del iPhone en segundo plano
F08 Música, buscar, cámara
    W06/W07/W08 -> acción -> efecto en el iPhone
F09 Diagnóstico
    I07 -> exportar log -> compartir
```

## Componentes
Lista de permisos, tarjeta de estado de conexión, fila de app con interruptor, código de 6 dígitos grande, botones circulares del reloj (aceptar/rechazar), tarjeta de métrica, gráfico de ritmo cardiaco simple, toast de error, tile y complicación.

## Modelo de datos inferido
```
PendingRecord (reloj, Room)  id, type (hr|workout|route|sleep|daily), payload, created_at, attempts, acked
    evidencia: requisito de cola con confirmación; confianza: alta (es diseño propio)
AppFilter (iPhone)           bundle_id, display_name, enabled, last_seen
    evidencia: filtro por app (F2 en fuentes); confianza: alta
PairedWatch (iPhone)         peripheral_id, name, bonded_at, protocol_version
SyncLedger (iPhone)          sync_identifier, sync_version, written_at  -> evita duplicados en Salud
```

## Matriz de funciones
Ver `replica/features.csv`.

## Lo que no se puede clonar (o no aplica)
- Suscripción, paywall, cuentas, relé por internet, copia en iCloud, iconos de apps desde servidor: no aplican (sin servidores ni cobros).
- Siri ("Merge Assistant"): probablemente necesita perfil HFP o APIs no públicas; se evalúa en la fase 4.
- Responder notificaciones: Merge tampoco lo ofrece en Wear OS; ANCS solo permite acciones positiva/negativa.
- Audio de llamadas en el reloj: Merge lo logra en Samsung, presumiblemente como manos libres Bluetooth clásico (HFP). Viabilidad sin root a investigar en la fase 0/4.
- Activación sin teléfono Android: fuera de alcance (tu reloj ya está activado).

## Tamaño
Pantallas 17, flujos 9, entidades 4. Partes difíciles: (1) mantener el enlace BLE vivo con iOS en segundo plano y Samsung durmiendo apps; (2) ANCS/AMS desde Wear OS con cifrado y emparejamiento; (3) HealthKit con una firma de terceros. Tamaño: **L** (un trimestre a ritmo personal).
