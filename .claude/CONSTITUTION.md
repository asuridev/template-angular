# Constitución

Este documento reúne las reglas **inviolables** del desarrollo de este
proyecto. No se omiten, no se reinterpretan caso a caso y no admiten
excepciones silenciosas — cualquier cambio a una de estas reglas requiere
actualizar explícitamente este archivo.

**Prioridad ante conflictos:** `CONSTITUTION.md` > `ARCHITECTURE.md` >
`CLAUDE.md`.

Cada regla incluye una referencia a la sección donde está explicada con
detalle y ejemplos de código.

## Estado y datos

1. **Nunca usar axios** — todo HTTP se hace vía `HttpClient` de Angular.
   (`ARCHITECTURE.md` §3, §9)
2. **NgRx SignalStore solo modela estado síncrono** de UI/sesión — nunca
   datos que provienen de una API. (`ARCHITECTURE.md` §2, §9)
3. **TanStack Query es el único mecanismo de estado asíncrono/de servidor y
   caché.** (`ARCHITECTURE.md` §3, §9)
4. **Los componentes nunca inyectan `HttpClient` ni un `*ApiService`
   directamente** — solo a través de TanStack Query (`queries/`) o del
   Store correspondiente. (`ARCHITECTURE.md` §2, §3)

## Componentes

5. **Solo componentes standalone** — nunca `NgModule`, nunca
   `standalone: true` explícito en el decorador. (`CLAUDE.md` — Angular
   Best Practices)
6. **`changeDetection: ChangeDetectionStrategy.OnPush` obligatorio** en todo
   componente. (`CLAUDE.md` — Components)
7. **Estado local vía signals** — nunca `mutate()`, usar `update()`/`set()`.
   (`CLAUDE.md` — State Management)
8. **Formularios siempre Reactive Forms** — nunca template-driven.
   (`CLAUDE.md` — Components)
9. **Nunca `ngClass`/`ngStyle`** — usar bindings `class`/`style`.
   (`CLAUDE.md` — Components)
10. **Nunca `@HostBinding`/`@HostListener`** — usar el objeto `host` del
    decorador. (`CLAUDE.md` — Angular Best Practices)

## Inyección de dependencias

11. **Siempre `inject()`** — nunca inyección por constructor.
    (`CLAUDE.md` — Services)
12. **Servicios singleton siempre `providedIn: 'root'`.**
    (`CLAUDE.md` — Services)
