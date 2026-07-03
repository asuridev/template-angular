# Arquitectura

Este documento define las convenciones de arquitectura frontend para este
proyecto Angular: estructura de carpetas, gestión de estado, capa HTTP,
routing y organización de componentes. Complementa a `CLAUDE.md` (reglas de
estilo a nivel de componente/servicio) — lee ambos; este archivo no repite lo
que ya está cubierto ahí.

La referencia canónica de los patrones de estado descritos abajo es la
documentación oficial de NgRx Signals: https://ngrx.io/guide/signals

## 1. Estructura de carpetas y módulos

Feature-first. El código vive dentro del feature que lo posee; se promueve a
`shared/` solo la segunda vez que otro feature lo necesita. Evitar barrel
files (`index.ts`) salvo que exista una necesidad concreta.

```
src/app/
  core/                        # singletons de toda la app, registrados una vez en app.config.ts
    interceptors/
      auth-interceptor.ts
      error-interceptor.ts
      loading-interceptor.ts
    store/
      app.store.ts             # estado síncrono transversal sin feature dueño (tema, idioma, notificaciones globales)
  shared/                      # código cross-feature, reusado por 2+ features
    components/
      ui/                      # átomos: button.ts, input.ts, card.ts (variantes vía input())
    models/
    pipes/
    services/
  features/
    auth/
      auth.routes.ts
      components/
      guards/
        role-guard.ts
      layouts/
        auth-layout.ts         # shell del módulo, expone <router-outlet>, se carga vía loadComponent
      models/
        session-model.ts
      pages/
        login/
          login.ts
      pipes/
      queries/
        auth-queries.ts         # queryOptions() para injectQuery/injectMutation, envuelve auth-api.ts
      services/
        auth-api.ts            # <Feature>ApiService, solo envuelve HttpClient
      store/
        auth.store.ts
  app.config.ts
  app.routes.ts
```

**Naming:** seguir el estilo Angular v20 — sin sufijo de "tipo" en los nombres
de archivo (`auth-api.ts`, no `auth-api.service.ts`). Este proyecto ya sigue
esta convención (`app.ts`, no `app.component.ts`). Cuando el archivo sí
necesita una palabra que indique su tipo (guards, interceptors, modelos), se
usa guion — no punto — como separador (`role-guard.ts`, `auth-interceptor.ts`,
`session-model.ts`): es el separador por defecto de los schematics
`ng generate guard`/`ng generate interceptor` en Angular v20 (opción
`type-separator`, default `-`; `.` solo si se pasa explícitamente). **Única
excepción real:** mantener los sufijos con punto `.routes.ts` y `.store.ts`
— son convenciones ya establecidas del ecosistema Router / NgRx Signals, y ya
usadas por `app.routes.ts`.

Un feature solo necesita `layouts/` si tiene una "cáscara" visual propia
(nav, sidebar, contenedor). Si el feature es una sola página suelta, se omite
y la página se registra directamente como hija de la ruta lazy del feature
(ver §4).

## 2. Gestión de estado — NgRx Signals (estado síncrono) + TanStack Query (asíncrono)

**Decisión:** se separan responsabilidades entre dos mecanismos
especializados, cada uno idiomático en su terreno:

- **NgRx SignalStore** — únicamente estado **síncrono** de UI/cross-cutting
  (sesión, rol, flags de interfaz), vía `withState`/`withComputed`/
  `patchState`. No se usa para modelar datos que provienen de una API.
- **TanStack Query** (`@tanstack/angular-query-experimental`) — todo estado
  **asíncrono/de servidor**: fetch, caché, mutaciones e invalidación, vía
  `injectQuery`/`injectMutation`. Es el único mecanismo para este tipo de
  estado (ver detalle y ejemplos en §3, subsección "Estado asíncrono y
  caché — TanStack Query"). No se usa `rxMethod`+`tapResponse` sobre el
  store para llamadas HTTP.

**Cuándo crear un store:** estado síncrono compartido por 2 o más
componentes/rutas (sesión, preferencias, flags). Nunca para datos que vienen
de una API — eso es responsabilidad de TanStack Query. El estado trivial de
un solo componente sigue siendo una señal local (según `CLAUDE.md`) — no
crear un store para eso.

**Root-provided vs. feature-provided:**
- `{ providedIn: 'root' }` para estado cross-cutting que debe sobrevivir la
  navegación (ej. `AuthStore` — sesión/usuario/rol).
- Feature-provided (sin `providedIn`, inyectado vía el arreglo `providers`
  de la ruta del feature) para estado que debe destruirse al salir del
  feature.

**Regla obligatoria:** los componentes nunca inyectan `HttpClient` ni un
`*ApiService` directamente. Para estado de servidor usan
`injectQuery`/`injectMutation` a través de las `queryOptions` definidas en
`queries/` (ver §3). Para estado síncrono, inyectan el Store y llaman a sus
métodos.

```typescript
export const AuthStore = signalStore(
  { providedIn: 'root' },
  withState({ user: null as User | null }),
  withComputed(({ user }) => ({
    isAuthenticated: computed(() => user() !== null),
  })),
  withMethods((store) => ({
    setUser(user: User | null): void {
      patchState(store, { user });
    },
    clear(): void {
      patchState(store, { user: null });
    },
  }))
);
```

`setUser` se invoca típicamente desde el `onSuccess` de una mutación de
TanStack Query (ver ejemplo en §3) — el store nunca dispara la llamada HTTP
por sí mismo, solo recibe el resultado y lo guarda como estado síncrono.

Para slices de estado síncrono reutilizados entre varios stores, usar
`signalStoreFeature(withState(...), withComputed(...), withMethods(...))`.

**Un store realmente transversal (sin feature dueño) vive en
`core/store/<name>.store.ts`**, no dentro de `features/`. Es la otra cara
del caso `AuthStore`: `AuthStore` es `providedIn: 'root'` pero su dominio
es el feature `auth`, así que su archivo vive en
`features/auth/store/auth.store.ts` (ver §1). Cuando el estado síncrono no
pertenece al dominio de ningún feature — ej. tema visual, idioma activo,
un banner de notificaciones global — no hay feature al que asignarle el
archivo, y por eso vive en `core/store/` en su lugar. Sigue siendo
`providedIn: 'root'` igual que cualquier store cross-cutting (ver regla de
arriba).

## 3. Capa HTTP

- **Servicios API inyectables**, uno por feature: `<Feature>ApiService`,
  `providedIn: 'root'`, archivo `features/<name>/services/<name>-api.ts`.
  Responsabilidad única: envolver llamadas a `HttpClient` y mapear DTOs a
  modelos. Sin lógica de negocio ni estado — eso vive en el store.
- **Interceptors funcionales**, registrados una sola vez en `app.config.ts`
  vía `provideHttpClient(withInterceptors([...]))`. Nota: `provideHttpClient()`
  todavía no se llama en ningún lugar de este proyecto — es wiring nuevo.

```typescript
// core/interceptors/auth-interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthStore).token();
  if (!token) return next(req);
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};

// core/interceptors/error-interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 401) inject(Router).navigateByUrl('/login');
      return throwError(() => normalizeApiError(err));
    })
  );

// core/interceptors/loading-interceptor.ts
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loading = inject(LoadingIndicatorService);
  loading.show();
  return next(req).pipe(finalize(() => loading.hide()));
};
```

```typescript
// app.config.ts
providers: [
  provideHttpClient(
    withInterceptors([authInterceptor, errorInterceptor, loadingInterceptor])
  ),
  // ...providers existentes
]
```

**Función de cada interceptor** (resumen):
- `auth-interceptor.ts` — adjunta el token de autenticación a cada request
  saliente, leyéndolo del `AuthStore`, para que ningún `*ApiService` tenga
  que agregarlo manualmente.
- `error-interceptor.ts` — centraliza el manejo de errores HTTP (ej. redirige
  a `/login` en un 401, normaliza el formato del error) antes de que llegue
  al consumidor de la petición (TanStack Query u otro).
- `loading-interceptor.ts` — lleva la cuenta de peticiones en curso para
  mostrar/ocultar un indicador de carga global, sin que cada componente
  gestione su propio spinner.

### Estado asíncrono y caché — TanStack Query

TanStack Query no reemplaza los interceptors anteriores, los complementa:
los interceptors operan a nivel de petición HTTP; TanStack Query opera un
nivel más arriba, gestionando caché, reintentos, invalidación y el ciclo de
vida de cada consulta/mutación.

- **Setup**, en `app.config.ts` (aún no existe, es wiring nuevo, igual que
  `provideHttpClient`):
```typescript
// app.config.ts
providers: [
  provideTanStackQuery(new QueryClient()),
  // ...providers existentes (incluye provideHttpClient con interceptors)
]
```
- **Convención**: por feature, un archivo `queries/<name>-queries.ts` con
  una clase `@Injectable({ providedIn: 'root' })` que expone métodos
  `queryOptions(...)`, consumiendo el `<Feature>ApiService` correspondiente
  (nunca `HttpClient` directamente).

```typescript
// features/auth/queries/auth-queries.ts
@Injectable({ providedIn: 'root' })
export class AuthQueries {
  private authApi = inject(AuthApiService);

  session() {
    return queryOptions({
      queryKey: ['auth', 'session'],
      queryFn: () => this.authApi.getSession(),
    });
  }
}
```

- **Uso en componentes**: a diferencia del estado síncrono (§2), aquí sí se
  permite llamar `injectQuery`/`injectMutation` directamente desde el
  componente — TanStack Query ya encapsula caché, reintentos e invalidación,
  no requiere pasar por un store intermedio.

```typescript
// features/auth/pages/login/login.ts
export class Login {
  private authApi = inject(AuthApiService);
  private queryClient = inject(QueryClient);
  private authStore = inject(AuthStore);

  loginMutation = injectMutation(() => ({
    mutationFn: (credentials: Credentials) => this.authApi.login(credentials),
    onSuccess: (user) => {
      this.authStore.setUser(user);                              // estado síncrono → NgRx
      this.queryClient.invalidateQueries({ queryKey: ['auth'] }); // caché → TanStack Query
    },
  }));
}
```

## 4. Routing

- Cada feature posee un `<feature>.routes.ts` que exporta un arreglo
  `Routes`, compuesto de forma perezosa (lazy) desde la raíz vía
  `loadChildren`.
- Los guards funcionales (`CanActivateFn`) se componen y anidan siguiendo un
  patrón en capas: `authGuard → roleGuard → guard de dominio`.
- `route.data` transporta metadata que adapta el layout (el equivalente
  Angular del `handle` de React Router).

```typescript
// app.routes.ts
export const routes: Routes = [
  {
    path: 'auth',
    loadChildren: () => import('./features/auth/auth.routes').then((m) => m.AUTH_ROUTES),
  },
];

// features/admin/admin.routes.ts — composición de guards anidados
export const ADMIN_ROUTES: Routes = [
  {
    path: '',
    canActivate: [authGuard],
    children: [
      {
        path: '',
        canActivate: [roleGuard('admin')],
        children: [
          { path: '', canActivate: [mfaGuard], component: AdminDashboard, data: { authWide: true } },
        ],
      },
    ],
  },
];

// role-guard.ts — guard funcional parametrizado
export const roleGuard = (role: string): CanActivateFn => () =>
  inject(AuthStore).user()?.role === role || inject(Router).createUrlTree(['/forbidden']);
```

### Layouts de módulo y lazy loading

- Cada feature con shell visual propio define su layout en
  `features/<name>/layouts/<name>-layout.ts`. Es un componente standalone
  con `<router-outlet>` en su template, para renderizar las páginas hijas.
- El layout se registra como ruta padre del feature usando `loadComponent`
  (no `component`), y cada página hija también usa `loadComponent`
  individualmente — así cada página es su propio chunk descargado bajo
  demanda, no solo el feature completo.
- Los guards (ejemplo anterior) se aplican a nivel de ruta (en el padre del
  layout, o en cada hija); el layout solo se encarga del shell visual, nunca
  de la protección de acceso.

```typescript
// features/auth/auth.routes.ts
export const AUTH_ROUTES: Routes = [
  {
    path: '',
    loadComponent: () => import('./layouts/auth-layout').then((m) => m.AuthLayout),
    children: [
      { path: 'login', loadComponent: () => import('./pages/login/login').then((m) => m.Login) },
      { path: 'register', loadComponent: () => import('./pages/register/register').then((m) => m.Register) },
    ],
  },
];
```

```typescript
// features/auth/layouts/auth-layout.ts
@Component({
  selector: 'app-auth-layout',
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [RouterOutlet],
  template: `
    <div class="auth-shell">
      <router-outlet />
    </div>
  `,
})
export class AuthLayout {}
```

## 5. Componentes / diseño atómico

- `shared/components/ui/` contiene los átomos (Button, Input, Card…), cada
  uno exponiendo variantes vía `input()` (ej.
  `variant: input<'primary' | 'secondary'>()`), con las clases Tailwind de
  marca encapsuladas dentro del propio átomo — nunca repetidas en templates
  de features.
- **Ante un nuevo tratamiento visual de un componente ya existente
  (compartido o de feature), la solución es agregar una nueva variante**
  (vía su `input()` de variante, como en el ejemplo anterior) — nunca
  editar el template del componente para añadir clases Tailwind
  condicionadas a un caso puntual. Esto no afecta la excepción de layout
  del bullet siguiente: un consumidor sigue pudiendo aplicar utilidades de
  spacing (`mt-4`, `gap-2`, etc.) sobre el host del componente; lo que se
  prohíbe es tocar las clases internas de apariencia/marca del propio
  componente.
- Regla de promoción (ver §1): un componente específico de un feature se
  mueve a `shared/` solo en su segundo reuso entre features.
- Una página por carpeta: `pages/<page-name>/<page-name>.ts`.
- Los templates de feature usan los átomos compartidos para controles de
  formulario/botones — nada de duplicar literales tipo
  `<input class="border rounded px-3...">` cuando ya existe un wrapper
  compartido. Solo se permiten utilidades de layout (flex/grid/gap/spacing)
  inline en templates de feature.

## 6. Modelos / interfaces

- Carpeta `models/` por feature, sufijo `-model.ts` (guion, no punto — ver
  regla de naming en §1), ej. `session-model.ts`.
- Los tipos pequeños de uso local quedan inline en el servicio que los usa.
- Se promueven a `shared/models/` solo al ser reusados por 2 o más features.

## 7. Pipes

- Carpeta `pipes/` por feature, para pipes de uso exclusivo de ese feature.
- Se promueven a `shared/pipes/` solo al ser reusados por 2 o más features
  (misma regla de promoción que §1/§6).
- **Naming:** sin sufijo de tipo (`format-currency.ts`, no
  `format-currency.pipe.ts`), consistente con la convención de §1
  (excepción solo para `.routes.ts` y `.store.ts`).
- Los pipes son standalone por default — no se declaran en ningún
  `NgModule`, se importan directamente en el arreglo `imports` del
  componente que los usa.

## 8. Configuración de entorno

- `src/environments/environment.ts`, `environment.qa.ts`,
  `environment.production.ts`, cableados vía `fileReplacements` por
  configuración de build en `angular.json`
  (`architect.build.configurations`). Nada de esto existe aún — es setup
  nuevo.
- Los servicios API leen `environment.apiUrl`, nunca `process.env` ni
  variables globales ad-hoc.

## 9. Testing

Mantener los defaults de Angular CLI: Karma + Jasmine, `*.spec.ts` colocado
junto a cada archivo fuente (ya es el patrón de `app.spec.ts`). No se
introduce ningún framework o convención nueva aquí.

Playwright CLI (ver `TOOLS.md`) es una herramienta de verificación manual
del agente durante el desarrollo (feedback visual, validación de flujos)
— no introduce Playwright como framework de pruebas automatizadas del
proyecto.

## 10. Reglas explícitas de la stack elegida

- **Sin axios en ningún lugar** — solo `HttpClient`, vía los servicios API
  respaldados por interceptors (§3). Se estandariza completamente en
  `HttpClient` por DI, testabilidad (`HttpTestingController`) y composición
  de interceptors.
- **TanStack Query es la única capa de estado asíncrono/caché de servidor**
  (`@tanstack/angular-query-experimental`, §3). NgRx SignalStore no debe
  usarse para modelar datos que provienen de una API — solo estado síncrono
  de UI/sesión (§2).
- Las llamadas a un servicio API (§3) para datos de servidor se hacen
  únicamente a través de TanStack Query (`queries/` +
  `injectQuery`/`injectMutation`), nunca directamente ni encapsuladas en el
  store. El store solo orquesta estado síncrono (§2). Esa disciplina de
  capas (componente → TanStack Query / Store → servicio → `HttpClient`) es
  obligatoria en todo el proyecto.
- **Tailwind CSS (v4) es la única librería de estilos** — wireado vía
  `@import "tailwindcss"` en `src/styles.css` y el plugin
  `@tailwindcss/postcss` en `.postcssrc.json`. No se introducen otros
  frameworks/librerías CSS ni CSS plano fuera de utilidades Tailwind (salvo
  el bloque `<style>` de host ya presente en componentes generados por el
  CLI, ver §5).
- **Proyecto zoneless** — `provideZonelessChangeDetection()` está activo en
  `app.config.ts` desde el scaffold inicial. No se importa `zone.js`, no se
  inyecta `NgZone`, y la UI se actualiza únicamente vía signals +
  `ChangeDetectionStrategy.OnPush` (ver `CLAUDE.md` — Angular Best
  Practices).
