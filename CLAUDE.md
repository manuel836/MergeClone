# CLAUDE.md

## Objetivo
Puente directo por Bluetooth LE entre un Galaxy Watch 7 (Wear OS) y un iPhone 15 Pro Max, de uso personal:
notificaciones, entrenamientos, sueño y métricas a Apple Salud, y el resto de funciones tipo Merge.
Sin servidores, cuentas, cobros, tiendas ni telemetría. Nombre del producto: pendiente (ver docs/DISEÑO.md).

## Reglas
- Todo lo que lee el usuario (UI, docs, mensajes, commits de cara a él, PRs) va en **español**; el código, identificadores y comentarios técnicos, en **inglés**. El usuario no sabe Kotlin ni Swift: comenta el porqué.
- Herramientas: **replica** decide QUÉ (alcance, paridad, salidas en `replica/`); **Superpowers** decide CÓMO (brainstorming → worktrees → writing-plans → subagent-driven-development → TDD → requesting-code-review → verification-before-completion → finishing-a-development-branch); **ECC** decide CALIDAD y SEGURIDAD (reglas en `.claude/rules/ecc/`, revisores code-reviewer, kotlin-reviewer, kotlin-build-resolver, security-reviewer; `/code-review`, `/security-scan`, `/quality-gate` antes de cerrar cada rama). No usar replica-build/backend/launch/deploy ni el planner de ECC.
- Conflictos entre herramientas y supuestos → `docs/DECISIONES.md`. Cambios de hooks de ECC: los decide el usuario.
- No copiar código, textos, logos ni imágenes de Merge; sin descompilar. Solo funciones y flujos.
- Documentación oficial vigente antes de programar; enlazarla en `docs/INVESTIGACION.md`. Nada de APIs de memoria.
- TDD en toda la lógica; hardware detrás de interfaces con dobles de prueba. Lo no automatizable → `docs/PRUEBAS.md`.
- Ningún secreto en el repo (keystore y claves solo como secrets de Actions). No cambiar la visibilidad del repo.
- iOS: proyecto generado con XcodeGen (`ios/project.yml`); nunca editar `.xcodeproj` a mano.
- Cada fase en su rama y PR; al cerrarla, esperar el OK del usuario tras probar en sus dispositivos.

## Cómo verificar
Nada está terminado sin las dos cosas en verde:
1. `bash scripts/verify.sh` (en Windows con Git Bash; también en Linux/CI).
2. El CI de GitHub Actions del PR.
