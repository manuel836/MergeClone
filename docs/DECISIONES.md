# Decisiones y supuestos

Registro de decisiones, supuestos y conflictos entre herramientas. Lo más reciente, arriba.

| Fecha | Tema | Decisión | Por qué |
| --- | --- | --- | --- |
| 2026-11-25 | Diseño aprobado | `docs/DISEÑO.md` aprobado; nombre **Tándem**; prueba temprana de HealthKit: sí; sin pruebas reales del resto hasta el final; no se espera OK de dispositivos entre fases. | Respuestas del usuario. |
| 2026-11-25 | Un plan por fase | Superpowers pide un plan por subsistema que funcione solo; se escribe el plan de la fase 0 ahora y el de cada fase al empezarla, cada uno para tu aprobación salvo que digas lo contrario. | Las fases dependen de lo aprendido en la anterior. |
| 2026-11-25 | Rama `main` | Creada desde un commit inicial vacío (autorizado por el usuario, opción B) y unida a la rama de trabajo con un merge, sin reescribir historia. PR #1 en borrador. | El repo estaba vacío y el PR necesitaba una base. |
| 2026-11-25 | Dónde corre Claude Code | Esta sesión corre en un contenedor en la nube, no en la PC con Windows. El trabajo de repo, docs, CI e investigación se hace aquí; lo que toca el reloj (adb) necesita tu PC. | Es el entorno real de la sesión. |
| 2026-11-25 | Plugins sin cargar en esta sesión | Los tres plugins están instalados en el contenedor, pero la sesión no los cargó (se instalaron a mitad de sesión). Mientras tanto, sigo sus SKILL.md leyéndolos del disco, sin modificarlos. | La recarga de plugins solo ocurre al iniciar sesión. |
| 2026-11-25 | Origen de Superpowers | Quedó instalado como `superpowers@anthropic-plugin-directory` (es el mismo plugin, obra/superpowers). `.claude/settings.json` declara `superpowers@claude-plugins-official` como pediste. | En el primer intento el marketplace oficial no lo encontraba; no dupliqué la instalación. |
| 2026-11-25 | Hooks de ECC | Perfil **minimal** por decisión del usuario: solo los hooks esenciales de ciclo de vida y seguridad. Queda en `.claude/settings.json` (`env`: `ECC_HOOKS_ENABLED=true`, `ECC_HOOK_PROFILE=minimal`), que tiene prioridad sobre la preferencia del plugin y viaja con el repo; también se guardó en la configuración local del plugin. | Decisión tuya (2026-11-25). |
| 2026-11-25 | Reglas de ECC | Copiadas sin cambios de `rules/common`, `rules/swift`, `rules/kotlin` a `.claude/rules/ecc/`, sin correr el instalador. | Pedido explícito; crédito en THIRD-PARTY-NOTICES.md. |
| 2026-11-25 | replica en apps nativas | Donde replica supone web (Playwright, dominio, capturas de rutas), se adapta: "pantallas" = pantallas del reloj y del iPhone; pruebas E2E = pruebas instrumentadas/manuales en `docs/PRUEBAS.md`. | replica está pensado para apps web. |
