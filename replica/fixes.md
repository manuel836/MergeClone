# Qué odian, qué falta y qué arreglamos

Muestra: **335 reseñas reales** (321 App Store en 8 países, 11 Google Play, 3 Hacker News; Reddit bloqueó el acceso). Fechas 2023-09 a 2026-11. Clasificación en `replica/feedback.md` (temas propios en `replica/themes.json`). Citas literales; el feed de Apple no da enlace por reseña, así que el enlace es la página de reseñas del país.

## 1. Lo que odian (sin precio ni cobros, que aquí no aplican)
1. **Desconexiones y emparejamiento** — 71 reseñas, 9 fuentes. "Not stable enough connection" ([us](https://apps.apple.com/us/app/id6698894276?see-all=reviews)); "Pésima conexión" ([mx](https://apps.apple.com/mx/app/id6698894276?see-all=reviews)).
2. **Notificaciones que no llegan, llegan tarde o solo de algunas apps** — 58 reseñas, 10 fuentes. "Spotty notifications" ([us](https://apps.apple.com/us/app/id6698894276?see-all=reviews)).
3. **Instalación complicada y paywall al final** — 22 reseñas, 8 fuentes. "Horrible to set up, then you have to pay" ([us](https://apps.apple.com/us/app/id6698894276?see-all=reviews)).
4. **Hay que mantener la app abierta en segundo plano** — 7 reseñas, 5 fuentes. "You have to keep the Merge App open and running in the background. But still it sometimes disconnects." ([us](https://apps.apple.com/us/app/id6698894276?see-all=reviews)).
5. **Batería** (reloj y iPhone) — 12 reseñas, 5 fuentes. "Great functionality but expensive and drains watch battery." ([us](https://apps.apple.com/us/app/id6698894276?see-all=reviews)).
(Precio: 153 reseñas; cobros: 17. No aplican a un proyecto personal sin cobros.)

## 2. Lo que falta
1. **Sueño y otras métricas a Apple Salud** — 20 reseñas, 7 fuentes. "I wish the sleep data and other galaxy features like BMI would sync to Apple Health." ([us](https://apps.apple.com/us/app/id6698894276?see-all=reviews)); "Sleep and other health metrics are not synced accurately." ([ca](https://apps.apple.com/ca/app/id6698894276?see-all=reviews)).
2. **Contestar llamadas en el reloj** — 27 reseñas, 7 fuentes (en algunos modelos no funciona). "Wish we could answer iPhone calls on the watch." ([us](https://apps.apple.com/us/app/id6698894276?see-all=reviews)).
3. **Clima y hora** — 6 reseñas, 2 fuentes (*delgado*).

## 3. Lo no resuelto
- **Datos de salud duplicados o imprecisos** cuando conviven otras integraciones (pocas reseñas, *delgado*).
- **Diagnóstico**: el "troubleshooter" no ayuda; el usuario no sabe por qué falla (reseña de Google Play, *delgado*).

## 4. Plan de arreglos (evidencia × costo)
| # | Arreglo | Tamaño | Fase | Evidencia |
| --- | --- | --- | --- | --- |
| 1 | Reconexión automática con *state restoration* de CoreBluetooth: la app de iOS se relanza sola tras ser suspendida; reconexión pendiente sin tiempo límite. Pruebas de 24 h en `docs/PRUEBAS.md`. | M | 0 | desconexiones (71), segundo plano (7) |
| 2 | Cola persistente en el reloj (Room) con confirmación del iPhone: nada se pierde por estar lejos. | M | 0 | sincronización y datos (13) |
| 3 | Notificaciones por ANCS directas del sistema (no dependen de que la app del iPhone esté viva) + filtro por app. | M | 1 | notificaciones (58) |
| 4 | Sueño a Salud (`asleepUnspecified` si no hay fases) sin duplicados (SyncIdentifier/SyncVersion). | M | 3 | salud y sueño (20) |
| 5 | Presupuesto de batería del reloj medido en cada fase (% por hora). | S | todas | batería (12) |
| 6 | Pantalla de estado y log exportable en ambos lados. | S | 0 | diagnóstico (*delgado*) |
| 7 | Emparejamiento dentro de la app, con código en ambas pantallas, sin ir a Ajustes de iOS. | S | 0 | instalación (22) |

Estas filas ya están en `replica/features.csv` (reconexión, log, sueño).

## 5. El ángulo
Proyecto personal, sin venta: el ángulo solo guía el nombre y el tono.
> Para quien usa un Galaxy Watch con iPhone y está harto de que se desconecte, este puente **se reconecta solo, no pierde datos y manda también el sueño a Salud**, sin cuentas ni suscripción.
Evidencia: desconexiones (71 reseñas, 9 fuentes) + salud y sueño (20, 7).
