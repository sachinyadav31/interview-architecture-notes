# 17: Angular frontend architecture

## Applications
| App | Angular | Users |
|---|---|---|
| `eprocure-business-portal` | 18.2 | Buyers: requisitions, POs, invoices, eSourcing, contracts, DocuSign, catalogs, coupons, inventory, insights/dashboards |
| `supplier-portal` | 18.2 | Suppliers: catalogs/products, RFQ responses, POs, invoices, contacts/events |
| `ep-tenant-portal` | 13 → **18** (my upgrade, Jun 2026) | Platform admin: tenants, modules, languages, pricing, themes |
| `identity-management-ui-lib` | 18 library | Shared login, OTP, user/role management, published as `@inclufy/iam-ui-lib` |

UI kit: **PrimeNG 18** (Aura theme), Bootstrap 5, Angular Material in places, chart.js, `ngx-translate` (translations served from `ep-translation-service`), `@ng-idle` (auto logout on inactivity), `ng-otp-input` (MFA).

## Structure (business portal)
- **Feature modules, lazy loaded** (`loadChildren`): `e-sourcing`, `invoice`, `contract`, `catalogue`, `coupon`, `inventory`, `insights`, `settings`, `iam`… so the initial bundle stays small.
- `shared/` → `_interceptors/` and `_config/`:
  - **`TokenInterceptor`**: adds `Authorization: Bearer <token>` from `IamAuthUtils`; normalises GET params (removes empty `sort`, forces `isDeleted=false` filter).
  - **`HttpErrorInterceptor`**: central error toasts, handles 401 → redirect to login, 403 → access denied page.
  - **`LoaderInterceptor`**: counts in-flight requests to show/hide a global spinner.
- **Route guards** (`canActivate`) check the login state and **ACLs** (menu items and buttons hidden if the user lacks the ACL resource; the backend still enforces it).
- Environments: `environment.ts / .sit.ts / .prod.ts`; `proxy.conf.json` routes `/api` → gateway `:8200` locally.

## The Angular 13 → 18 upgrade (my work)
Same path as the supplier portal:
- Angular/CLI 18.2, TypeScript 5.4, zone.js 0.14, PrimeNG 18 + `@primeng/themes`, ngx-translate 16.
- `tsconfig`: target ES2022, **`useDefineForClassFields: false`** (keeps the old class-field semantics, which matters for decorators and DI).
- `angular.json`: `browserTarget` → `buildTarget`, removed the PrimeNG resources CSS (themes are now JS presets), removed `defaultProject`.
- `app.module`: `providePrimeNG({theme: Aura})`, `provideHttpClient(withInterceptorsFromDi())` (keeps class-based interceptors working), `provideAnimationsAsync()`.
- API changes: `InputTextareaModule` → `TextareaModule`, `PrimeNGConfig` → `PrimeNG`, `p-radioButton label` → `inputId` + `<label>`, table row events use `$event.data`, `@` must be escaped in templates (the new control-flow syntax).
- Result: builds clean on Angular 18; only optional-chain warnings left.

## Interview Q&A

**Q: How does change detection work? What is OnPush?**
A: Zone.js patches async APIs and triggers a check from the root down. With `ChangeDetectionStrategy.OnPush` a component is checked only when an `@Input` reference changes, an event fires inside it, an async pipe emits, or you call `markForCheck()`. That gives big performance wins for lists. Signals (v16+) allow finer-grained updates and eventually zoneless apps.

**Q: How do interceptors work, and in what order?**
A: Every `HttpClient` call passes through the interceptor chain in the order they're provided (request) and in reverse for the response. Requests are immutable, so you `clone()` them. We had token → loader → error.

**Q: switchMap vs mergeMap vs concatMap vs exhaustMap?**
A: `switchMap` cancels the previous inner observable (type-ahead search). `mergeMap` runs all in parallel. `concatMap` queues them in order (sequential saves). `exhaustMap` ignores new ones while one is running (prevents login double-submit).

**Q: How do you avoid memory leaks with subscriptions?**
A: `async` pipe, `takeUntilDestroyed()` (v16+) or `takeUntil(destroy$)`, and avoiding nested subscribes.

**Q: Lazy loading: why and how?**
A: Split feature modules (or standalone routes) with `loadChildren: () => import(...)`, so code downloads only when the route is visited. Smaller initial bundle, faster first load. Add preloading strategies for likely next routes.

**Q: Standalone components and signals: have you used them?**
A: Our big portals are NgModule-based (we kept modules during the upgrade to limit risk). Standalone components drop NgModules (`imports` on the component). Signals are reactive values (`signal`, `computed`, `effect`) that Angular tracks precisely. I'd adopt them gradually for new features.

**Q: How do you handle auth in an SPA?**
A: Token interceptor + route guards + a central 401 handler with refresh logic, idle timeout (`@ng-idle`), and hiding UI by ACL. The backend still enforces every rule; UI checks are only for user experience.

**Q: How did you do i18n?**
A: `ngx-translate` with JSON keys loaded from our translation service per tenant and language (that's why the gateway whitelists `/translation?translationKey=login_page`: the login page needs labels before the user has a token).
