You are an expert in TypeScript, Angular, and scalable web application development. You write maintainable, performant, and accessible code following Angular and TypeScript best practices.

## Arquitectura

La estructura de carpetas, la gestión de estado (NgRx Signals), la capa
HTTP/interceptors, la composición de routing y las convenciones de
organización de componentes de este proyecto están documentadas en
@.claude/ARCHITECTURE.md. Léelo antes de crear cualquier feature module nuevo.

## Constitución

Las reglas listadas en @.claude/CONSTITUTION.md son innegociables: no se
omiten ni se reinterpretan caso a caso. Ante cualquier conflicto aparente
entre este archivo, @.claude/ARCHITECTURE.md y @.claude/CONSTITUTION.md,
esta última tiene siempre prioridad.

## Herramientas

Las herramientas de CLI disponibles para el agente (Playwright CLI para
feedback visual/validación de flujos, ctx7 para documentación de
librerías) están documentadas en @.claude/TOOLS.md.

## TypeScript Best Practices

- Use strict type checking
- Prefer type inference when the type is obvious
- Avoid the `any` type; use `unknown` when type is uncertain

## Angular Best Practices

- Always use standalone components over NgModules
- Must NOT set `standalone: true` inside Angular decorators. It's the default.
- Use signals for state management
- Implement lazy loading for feature routes
- Do NOT use the `@HostBinding` and `@HostListener` decorators. Put host bindings inside the `host` object of the `@Component` or `@Directive` decorator instead
- Use `NgOptimizedImage` for all static images.
  - `NgOptimizedImage` does not work for inline base64 images.
- This project uses zoneless change detection
  (`provideZonelessChangeDetection()`) — never import `zone.js`, inject
  `NgZone`, or write code that relies on automatic/implicit change
  detection outside of signals and `OnPush`

## Components

- Keep components small and focused on a single responsibility
- Use `input()` and `output()` functions instead of decorators
- Use `computed()` for derived state
- Set `changeDetection: ChangeDetectionStrategy.OnPush` in `@Component` decorator
- Prefer inline `template`/`styles` for small, simple components. Use separate
  files (`templateUrl`/`styleUrl`) once the template/styles grow in
  complexity, to improve separation of concerns
- Naming: no `Component` suffix on the class (`UserProfile`, not
  `UserProfileComponent`), matching the file's own no-type-suffix convention
  (see `ARCHITECTURE.md` §1). The `.ts`, template, and style files share the
  same base name; with multiple style files, append descriptive words (e.g.
  `user-profile-settings.css`)
- Prefer Reactive forms instead of Template-driven ones
- Do NOT use `ngClass`, use `class` bindings instead
- Do NOT use `ngStyle`, use `style` bindings instead

## State Management

- Use signals for local component state
- Use `computed()` for derived state
- Keep state transformations pure and predictable
- Do NOT use `mutate` on signals, use `update` or `set` instead

## Templates

- Keep templates simple and avoid complex logic
- Use native control flow (`@if`, `@for`, `@switch`) instead of `*ngIf`, `*ngFor`, `*ngSwitch`
- Use the async pipe to handle observables

## Styling

- This project uses Tailwind CSS (v4) as its only styling solution, wired
  via `@tailwindcss/postcss` (see `src/styles.css`, `.postcssrc.json`)
- Use Tailwind utility classes for all styling; do not introduce other CSS
  frameworks or component libraries (Bootstrap, Material,
  styled-components, etc.)

## Services

- Design services around a single responsibility
- Use the `providedIn: 'root'` option for singleton services
- Use the `inject()` function instead of constructor injection
