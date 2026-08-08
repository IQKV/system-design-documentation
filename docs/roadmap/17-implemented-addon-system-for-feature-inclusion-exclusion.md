# Implemented: Addon System for Feature Inclusion/Exclusion

> **Status**: Infrastructure complete, no addon implementations yet  
> **Proposal ref**: `17-proposal-addon-system-for-feature-inclusion-exclusion.md`  
> **Codebase**: `foundation-ui-app`

---

## Summary

The addon system described in the proposal has been fully implemented in `foundation-ui-app`. All core infrastructure — types, registry, loader, extension points, routing, and bootstrap integration — is in place and working. The `src/addons/` slot directory exists but contains only a `.gitkeep`; no concrete addons have been written yet.

---

## What Was Implemented

### Directory Structure (as built)

```
src/
├── addons/                              ← Addon slot (empty — only .gitkeep)
├── app/
│   ├── addons/                          ← Addon system core
│   │   ├── types.ts
│   │   ├── addon-registry.ts
│   │   ├── addon-loader.ts
│   │   ├── index.ts
│   │   └── extension-points/
│   │       ├── index.ts
│   │       ├── navigation.ts
│   │       └── widgets.ts
│   ├── config/
│   │   ├── addons.ts                    ← AddonConfigSchema + getAddonConfig()
│   │   ├── billing.ts
│   │   ├── rollout-guards.ts
│   │   ├── runtime-env.ts
│   │   └── index.ts
│   ├── app.tsx
│   └── ...
├── features/
│   └── manage-billing/
│       ├── model/
│       │   ├── entitlements-context.tsx
│       │   └── use-entitlements.ts
│       └── ui/
│           └── feature-gate.tsx
├── pages/
│   ├── _app.tsx
│   └── _app/
│       ├── addons.tsx                   ← Parent route (Outlet pass-through)
│       └── addons/
│           └── $addonPath.tsx           ← Catch-all for addon pages
└── shared/
    └── ui/
        └── app-layout/
            └── app-nav.tsx              ← Reads navigationExtension.getNavItems()
```

The `app/addons/providers/AddonProvider.tsx` directory described in the proposal was **not created** — it was found unnecessary (see deviations below).

---

## System Architecture

There are two independent, complementary inclusion/exclusion mechanisms:

| Mechanism                  | Scope                                        | Driven by                             |
| -------------------------- | -------------------------------------------- | ------------------------------------- |
| Addon system               | Entire feature modules (nav, pages, widgets) | `VITE_ENABLED_UI_ADDONS` env var      |
| Entitlements / FeatureGate | Individual UI elements                       | User's subscription plan from backend |

---

## Addon System — End-to-End Flow

### 1. Configuration (`src/app/config/addons.ts`)

```typescript
export function getAddonConfig(): AddonConfig {
  const raw: string =
    (typeof window !== "undefined" && (window as any)["VITE_ENABLED_UI_ADDONS"]) ||
    import.meta.env.VITE_ENABLED_UI_ADDONS ||
    "";
  const enabledAddons = raw
    .split(",")
    .map((s) => s.trim())
    .filter(Boolean);
  return { enabled: enabledAddons };
}
```

Reads a comma-separated list of addon IDs from **either**:

- `window.VITE_ENABLED_UI_ADDONS` — runtime injection (e.g. a `config.js` served separately, allows container-level override without rebuild)
- `import.meta.env.VITE_ENABLED_UI_ADDONS` — build-time `.env` value

Priority goes to the runtime window value. The result is Zod-validated:

```typescript
export const AddonConfigSchema = z.object({
  enabled: z.array(z.string()).default([]),
});
```

`.env.example` declares: `VITE_ENABLED_UI_ADDONS=`

This is a **hybrid runtime/build-time** config — a refinement over the proposal, which only described build-time `import.meta.env` reading.

---

### 2. Available Addons Whitelist (`src/app/addons/addon-loader.ts`)

```typescript
type AvailableAddons = Record<string, (() => Promise<{ default: Addon }>) | string>;

const availableAddons: AvailableAddons = {
  // "project-management": () => import("@/addons/project-management"),
  // "@company/my-external-addon": "@company/my-external-addon",
};
```

The whitelist maps an addon ID to either:

- **A function loader** — `() => Promise<{ default: Addon }>` for local addons under `src/addons/`
- **A package name string** — for external npm packages, resolved via `await import(packageName)`

Currently empty (only example comments). This is the only place to wire in new addons.

---

### 3. `loadAddons()` mechanics

```typescript
export async function loadAddons(): Promise<void> {
  const config = getAddonConfig();
  for (const addonId of config.enabled) {
    const loader = availableAddons[addonId];
    if (!loader) {
      console.warn(`Addon "${addonId}" not found in availableAddons`);
      continue;
    }
    try {
      let addon: Addon;
      if (typeof loader === "function") {
        const module = await loader();
        addon = module.default;
      } else {
        const module = await import(/* @vite-ignore */ loader);
        addon = module.default;
      }
      addonRegistry.register(addon);
    } catch (err) {
      console.error(`Failed to load addon "${addonId}":`, err);
      // continues — one broken addon does not break the app
    }
  }
}
```

Resilient: a failed addon import logs an error and skips, not crashing app startup.

---

### 4. `AddonRegistry`

```typescript
export class AddonRegistry {
  private addons = new Map<string, Addon>();

  register(addon: Addon): void {
    if (this.addons.has(addon.manifest.id)) {
      throw new Error(`Addon ${addon.manifest.id} already registered`);
    }
    this.addons.set(addon.manifest.id, addon);
  }

  getAddon(id: string): Addon | undefined { ... }
  getAllAddons(): Addon[] { ... }

  async initializeAll(context: AddonContext): Promise<void> {
    for (const addon of this.getAllAddons()) {
      if (addon.initialize) {
        await addon.initialize(context);
      }
    }
  }
}

export const addonRegistry = new AddonRegistry();
```

Module-level singleton. Initialization is sequential and awaited.

---

### 5. `Addon` Contract (`src/app/addons/types.ts`)

```typescript
export interface Addon {
  manifest: AddonManifest;
  initialize?: (context: AddonContext) => void | Promise<void>;
  cleanup?: () => void;
  routes?: Array<{
    path: string;
    component: () => Promise<{ default: ComponentType<any> }>;
    auth?: boolean;
  }>;
}
```

`routes` is included in the real `Addon` interface (the proposal had it in the example addon but not in the initial type definition).

The `AddonContext` passed to `initialize`:

```typescript
export interface AddonContext {
  httpClient: ...;
  queryClient: ...;
  session: { useSession, getAccessToken, getTenantKey };
  i18n: ...;
  extensions: {
    navigation: NavigationExtensionPoint;
    widgets: WidgetExtensionPoint;
  };
}
```

---

### 6. Extension Points

Both are **module-level singletons** — not React context, not class instances requiring instantiation.

**Navigation** (`src/app/addons/extension-points/navigation.ts`):

```typescript
registerNavItem(item: NavItemExtension): void
getNavItems(section: "workspace" | "account"): NavItemExtension[]
```

`NavItemExtension` fields: `id`, `label`, `to`, `icon` (ComponentType), `order?`, `section?`, `roles?`, `feature?`

Items are filtered by section and sorted by `order` (default 100) on read.

**Widgets** (`src/app/addons/extension-points/widgets.ts`):

```typescript
registerWidget(widget: WidgetExtension): void
getWidgets(location: "dashboard"): WidgetExtension[]
```

Only `"dashboard"` location is supported. Widgets are sorted by `order` on read.

---

### 7. Bootstrap Sequence (`src/main.tsx`) — Key Deviation from Proposal

```typescript
async function bootstrap() {
  // ... MSW setup ...

  const { loadAddons, addonRegistry, navigationExtension, widgetExtension } =
    await import("@/app/addons");

  await loadAddons();

  await addonRegistry.initializeAll({
    httpClient,
    queryClient,
    session: { useSession, getAccessToken, getTenantKey },
    i18n,
    extensions: { navigation: navigationExtension, widgets: widgetExtension },
  });

  root.render(<App />);
}
```

**Addon initialization runs in `main.tsx` before `root.render()`, not inside `useEffect` in `App`.**

This is a deliberate deviation from the proposal. The reason is documented directly in `src/app/app.tsx`:

> _"Addon initialization is handled in main.tsx bootstrap() before React mounts, so extension points (nav items, widgets) are already populated on first render — avoiding a race condition where AppNav's useMemo runs before addon nav items are registered."_

The `AddonProvider` wrapper proposed for `app.tsx` was not needed as a result.

---

### 8. Navigation Integration (`src/shared/ui/app-layout/app-nav.tsx`)

```typescript
const addonNavItems: NavItem[] = navigationExtension.getNavItems("workspace").map((item) => {
  const IconComponent = item.icon;
  return { label: item.label, icon: <IconComponent size={16} />, to: item.to };
});

const navItems: NavItem[] = [...baseNavItems, ...addonNavItems];
```

Because extension points are pre-populated before first render, this synchronous call is safe. Addon items are appended after base items.

> **Partial implementation note**: Only the `"workspace"` section is merged. The `"account"` section extension point exists and addons can register items into it, but `AppNav` does not yet merge `getNavItems("account")` into `accountSubItems`.

---

### 9. Addon Routing (`src/pages/_app/addons/`)

**`src/pages/_app/addons.tsx`** — parent route, just renders `<Outlet />`.

**`src/pages/_app/addons/$addonPath.tsx`** — catch-all that renders addon pages:

```typescript
function AddonPage() {
  const currentPath = routerState.location.pathname;
  const [Component, setComponent] = useState<React.ComponentType<any> | null>(null);

  useEffect(() => {
    const findRoute = async () => {
      const addons = addonRegistry.getAllAddons();
      for (const addon of addons) {
        for (const route of addon.routes ?? []) {
          if (route.path === currentPath) {
            const loaded = await route.component();
            setComponent(() => loaded.default);
            return;
          }
        }
      }
    };
    findRoute().catch(console.error);
  }, [currentPath]);

  if (!Component) return <LoadingOverlay visible />;
  return <Component />;
}
```

- Route matching is exact string equality on `route.path === currentPath`
- Components are lazy-loaded on demand via the `component()` loader declared in the addon
- Authentication is **not** re-applied inside this component — it is enforced by the `/_app` parent route's `beforeLoad` guard
- The `AuthGuard` wrapper proposed in the catch-all was not added (the parent route already covers it)

---

## Entitlements / FeatureGate System

Unchanged from pre-proposal state, used as-is.

`EntitlementsProvider` wraps all authenticated routes in `_app.tsx`:

```tsx
<EntitlementsProvider>
  <AppLayout>
    <Outlet />
  </AppLayout>
</EntitlementsProvider>
```

It fetches plan entitlements from the backend via TanStack Query. For personal workspaces in multi-tenant mode it uses `DEFAULT_PERSONAL_WORKSPACE_FEATURES` (hard-coded, no API call).

**`<FeatureGate feature="...">` component**:

```tsx
<FeatureGate feature="advanced_analytics" showUpgradePrompt>
  <AdvancedAnalyticsPanel />
</FeatureGate>
```

Hooks: `useHasFeature(featureCode)`, `useQuota("maxUsers" | "maxProjects")`.

Feature codes are `snake_case` strings matching backend `PlanEntitlement.features` keys.

---

## Other Configuration-Driven Feature Flags

The runtime-env config system (`src/app/config/runtime-env.ts`) drives additional build/runtime inclusion switches, following the same hybrid `window.*` / `import.meta.env.*` pattern:

| Variable                    | Values                           | Effect                                                                                                          |
| --------------------------- | -------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `VITE_ROLLOUT_MODE`         | `MULTI_TENANT` / `SINGLE_TENANT` | Shows/hides Billing nav, Team nav, Org settings, Tenant Switcher; activates `guardMultiTenantRoute()` redirects |
| `VITE_ENABLE_MAGIC_LINK`    | `true` / `false`                 | Toggles magic-link auth UI                                                                                      |
| `VITE_PAYMENT_GATEWAY_TYPE` | `STRIPE` / `LEMON_SQUEEZY`       | Selects payment integration                                                                                     |
| `VITE_DEMO_MODE`            | `true` / `false`                 | Enables demo-mode behaviour                                                                                     |

These flags are not part of the addon system but form a related layer of feature control.

---

## Full Flow: Config → Rendered UI

```
.env / window.VITE_ENABLED_UI_ADDONS
        │
        ▼
getAddonConfig()          Zod-validated, comma-split to string[]
        │
        ▼
loadAddons()              in main.tsx bootstrap, before React renders
  ├── look up each ID in availableAddons{}
  ├── dynamic-import the addon module (function or package string)
  └── addonRegistry.register(addon)
        │
        ▼
addonRegistry.initializeAll(context)
  └── addon.initialize(context)
      ├── extensions.navigation.registerNavItem(...)
      └── extensions.widgets.registerWidget(...)
        │
        ▼
root.render(<App />)
        │
        ├── AppNav (first render, extension points already populated)
        │   └── navigationExtension.getNavItems("workspace")
        │       → merged into navItems → rendered as nav links
        │
        └── /_app/addons/$addonPath (on navigation to addon route)
            └── AddonPage
                → scans addon.routes[] for path match
                → lazy-loads matched component
                → renders it
```

---

## Deviations from the Proposal

| Aspect                         | Proposal                                            | Implemented                                                                                                           |
| ------------------------------ | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Addon init location            | `useEffect` in `App` component                      | `await` in `main.tsx` before `root.render()` — eliminates render race condition                                       |
| `AddonProvider` wrapper        | Proposed (`app/addons/providers/AddonProvider.tsx`) | Not created — unnecessary given bootstrap-before-render approach                                                      |
| Runtime config override        | Not described                                       | `window.VITE_ENABLED_UI_ADDONS` takes priority over build-time env, enabling container-level override without rebuild |
| External package addons        | Not described                                       | `availableAddons` value can be a package name string; `loadAddons` handles `await import(packageName)`                |
| `routes` in `Addon` interface  | Present in example addon only                       | Included in the `Addon` TypeScript interface in `types.ts`                                                            |
| Account-section nav extension  | `addonAccountItems` merged into `accountSubItems`   | Extension point exists and works; `AppNav` only merges workspace section (account section wiring not yet done)        |
| `AuthGuard` in catch-all route | Proposed inside `AddonRouteRenderer`                | Not present — auth enforced by parent `/_app` `beforeLoad` guard                                                      |
| `src/addons/` content          | Addon subdirectories                                | Only `.gitkeep` — infrastructure exists, no addons written yet                                                        |

---

## Writing a New Addon

1. Create `src/addons/<addon-id>/index.ts` exporting a default `Addon` object
2. Register it in `src/app/addons/addon-loader.ts`:
   ```typescript
   const availableAddons: AvailableAddons = {
     "<addon-id>": () => import("@/addons/<addon-id>"),
   };
   ```
3. Enable it: `VITE_ENABLED_UI_ADDONS=<addon-id>` in `.env` or at runtime via `window.VITE_ENABLED_UI_ADDONS`

Addon structure follows FSD:

```
src/addons/<addon-id>/
├── index.ts         ← Addon entry — exports default Addon object
├── features/        ← Addon-specific features
├── widgets/         ← Dashboard widgets
├── pages/           ← Page components referenced in routes[]
├── api/             ← API modules
├── types/
└── locales/         ← Translations
```

Example addon (from proposal, fully compatible with the real `Addon` interface):

```typescript
import type { Addon } from "@/app/addons/types";

const addon: Addon = {
  manifest: {
    id: "project-management",
    name: "Project Management",
    version: "1.0.0",
    description: "Project management features",
  },
  routes: [
    {
      path: "/_app/addons/projects",
      component: () => import("./pages/projects").then((m) => ({ default: m.ProjectsPage })),
      auth: true,
    },
  ],
  initialize: ({ extensions }) => {
    extensions.navigation.registerNavItem({
      id: "pm-projects",
      label: "Projects",
      to: "/_app/addons/projects",
      icon: IconChecklist,
      section: "workspace",
      order: 20,
    });
    extensions.widgets.registerWidget({
      id: "pm-stats",
      component: ProjectStatsWidget,
      location: "dashboard",
      order: 50,
    });
  },
};

export default addon;
```

---

## Known Gaps / Next Steps

- **No addon implementations** — `src/addons/` is empty; the first concrete addon needs to be written to validate the full flow end-to-end
- **Account nav section not wired** — `AppNav` does not merge `getNavItems("account")` into `accountSubItems`
- **Widget extension point not consumed** — the dashboard page does not yet call `widgetExtension.getWidgets("dashboard")` to render registered widgets
- **Route matching is linear O(n)** — adequate for small addon counts; for many addons, pre-build a `Map<string, route>` as noted in the proposal's performance section
- **No `cleanup()` call site** — `AddonRegistry` does not call `addon.cleanup()` on unmount/hot-reload; relevant if addons register side effects
- **Architecture test** — `src/architecture.test.ts` should be updated to assert addon layer boundaries
