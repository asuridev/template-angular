# Herramientas del agente

Este documento lista las herramientas de CLI disponibles para el agente más
allá de escribir código: validación visual de vistas/flujos y consulta de
documentación actualizada de librerías. Complementa a `CLAUDE.md` y
`ARCHITECTURE.md` — no repite sus convenciones de código.

## Playwright CLI

Disponible vía la skill `.claude/skills/playwright-cli/` (instrucciones y
referencias completas en su `SKILL.md`). Permite al agente abrir un
navegador real contra el dev server (`ng serve`), navegar, interactuar con
la vista y tomar snapshots — es la forma de obtener feedback real de una
vista recién creada o modificada, y de validar un flujo de usuario completo
de principio a fin.

**Esto es una herramienta de verificación manual/exploratoria del agente
durante el desarrollo, no un framework de pruebas del proyecto.** La
suite de pruebas automatizadas sigue siendo únicamente Karma + Jasmine
(`ARCHITECTURE.md §9`); usar Playwright CLI aquí no introduce Playwright
como dependencia de testing ni contradice esa regla.

## ctx7 CLI

Disponible vía `npx ctx7@latest`, para obtener documentación actualizada de
librerías, frameworks y SDKs (Angular, NgRx Signals, TanStack Query,
Tailwind, etc.) en lugar de depender de conocimiento de entrenamiento que
puede estar desactualizado. Flujo típico:

1. `npx ctx7@latest library "<nombre de la librería>" "<pregunta>"` — resuelve
   el ID de la librería (formato `/org/project`).
2. Elegir el mejor resultado por coincidencia de nombre, relevancia y
   reputación de la fuente.
3. `npx ctx7@latest docs <libraryId> "<pregunta>"` — obtiene la documentación
   puntual para responder.

Útil especialmente antes de usar APIs de NgRx Signals, TanStack Query
(`@tanstack/angular-query-experimental`) o Tailwind v4, ya documentadas en
`ARCHITECTURE.md`, cuando se necesite confirmar sintaxis o cambios de
versión recientes.
