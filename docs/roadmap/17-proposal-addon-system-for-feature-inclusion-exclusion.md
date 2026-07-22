# Proposal: Addon System for Feature Inclusion/Exclusion

## Objective

Design an addon system for foundation-ui-app that enables:

- Including/excluding features dynamically without core code changes
- Modular, self-contained feature modules
- Runtime addon discovery and registration
- Integration with existing architecture (FSD, entitlements, routing, navigation)

## Current Architecture Recap (from plugin-architecture-review.md)

The foundation-ui-app uses:

- **Feature-Sliced Design (FSD)** for layer separation
- **TanStack Router** for file-based routing
- **Mantine** for UI components
- **Zustand** for client state
- **TanStack Query** for server state
- **Lingui** for i18n
- **Entitlements/FeatureGates** for feature access control

Key strengths already in place:

1. Clear layer boundaries and public API pattern
2. Isolated features in src/features/
3. Existing configuration system (build-time, runtime, feature flags)
4. Entitlements system for feature gating

## Existing Solutions & Inspiration

We drew inspiration from the following existing solutions and best practices:

### 1. Plugin-Based Architecture (VS Code Style)
- **Source**: [TecoFize Blog: Building a Plugin System in React (Like VS Code Extensions)](https://tecofize.com/blogs/55/building-plugin-system-react-like-vscode-extensions/)
- **Key Takeaways**:
  - Core app should act as a stable host environment
  - Plugins should be dynamically loaded based on configuration
  - Communication through well-defined interfaces
  - Independent plugin development and deployment

### 2. React Hook-Based Plugin Systems
- **Source**: [CSDN Article: 如何设计一个‘插件化 React 系统’](https://blog.csdn.net/weixin_41455464/article/details/156160385)
- **Key Takeaways**:
  - Use React Hooks to inject logic into core components
  - Use Context API to share plugin registry and core services
  - Custom Hooks as plugin injection points

### 3. FSD Tooling
- **[eslint-fsd-plugin](https://github.com/bzuev/eslint-fsd-plugin)**: Enforces FSD path conventions
- **[fsd-forge](https://github.com/meybiz/fsd-forge)**: CLI for generating FSD structures (widgets, features, pages)

## Structure Improvements for Addon Support

### 1. Current Structure vs Proposed Structure

Current:

```
src/
├── app/
├── processes/
├── pages/
├── widgets/
├── features/
├── entities/
├── shared/
└── types/
```

Proposed (enhanced for addons):

```
src/
├── app/
│   ├── config/
│   │   ├── billing.ts
│   │   ├── index.ts
│   │   ├── addons.ts        # New: Addon configuration
│   │   ├── rollout-guards.ts
│   │   └── runtime-env.ts
│   ├── addons/              # New: Addon system core
│   │   ├── index.ts
│   │   ├── types.ts
│   │   ├── addon-registry.ts
│   │   ├── addon-loader.ts
│   │   ├── extension-points/ # New: Extension points for addons
│   │   │   ├── index.ts
│   │   │   ├── navigation.ts # Navigation extension point
│   │   │   └── widgets.ts    # Widget extension point
│   │   └── providers/        # New: Addon providers
│   │       └── AddonProvider.tsx
│   ├── app.tsx
│   ├── index.ts
│   └── theme.ts
├── processes/
├── pages/
├── widgets/
├── features/
├── entities/
├── shared/
├── addons/                  # New: Addon directory
│   └── [addon-id]/
│       ├── index.ts
│       ├── features/
│       ├── widgets/
│       ├── pages/
│       ├── api/
│       ├── types/
│       └── locales/
└── types/
```

### 2. Extension Points System

Extension points allow addons to contribute to UI without modifying core code.

```typescript
// src/app/addons/extension-points/navigation.ts
import { type ComponentType } from "react";

export interface NavItemExtension {
  id: string;
  label: string;
  to: string;
  icon: ComponentType<{ size?: number }>;
  order?: number;
  section?: "workspace" | "account";
  roles?: string[];
  feature?: string;
}

export interface NavigationExtensionPoint {
  registerNavItem(item: NavItemExtension): void;
  getNavItems(section: "workspace" | "account"): NavItemExtension[];
}

class NavigationExtensionPointImpl implements NavigationExtensionPoint {
  private items: NavItemExtension[] = [];

  registerNavItem(item: NavItemExtension): void {
    this.items.push(item);
  }

  getNavItems(section: "workspace" | "account"): NavItemExtension[] {
    return this.items
      .filter((item) => !item.section || item.section === section)
      .sort((a, b) => (a.order || 100) - (b.order || 100));
  }
}

export const navigationExtension = new NavigationExtensionPointImpl();
```

```typescript
// src/app/addons/extension-points/widgets.ts
import { type ComponentType } from "react";

export interface WidgetExtension {
  id: string;
  component: ComponentType<any>;
  location: "dashboard";
  order?: number;
}

export interface WidgetExtensionPoint {
  registerWidget(widget: WidgetExtension): void;
  getWidgets(location: "dashboard"): WidgetExtension[];
}

class WidgetExtensionPointImpl implements WidgetExtensionPoint {
  private widgets: WidgetExtension[] = [];

  registerWidget(widget: WidgetExtension): void {
    this.widgets.push(widget);
  }

  getWidgets(location: "dashboard"): WidgetExtension[] {
    return this.widgets
      .filter((w) => w.location === location)
      .sort((a, b) => (a.order || 100) - (b.order || 100));
  }
}

export const widgetExtension = new WidgetExtensionPointImpl();
```

### 3. Updated AppNav with Extension Points

Modify AppNav to use navigation extension point:

```typescript
// src/shared/ui/app-layout/app-nav.tsx (updated)
import { navigationExtension } from "@/app/addons/extension-points";

export function AppNav() {
  // ... existing code ...

  // Get addon nav items
  const addonWorkspaceItems = navigationExtension.getNavItems("workspace");
  const addonAccountItems = navigationExtension.getNavItems("account");

  // Merge with core items
  const allNavItems = [...navItems, ...addonWorkspaceItems];
  const allAccountItems = [...accountSubItems, ...addonAccountItems];

  // ... rest of the component ...
}
```

## Design Principles

1. **Backward Compatibility**: No breaking changes to existing code
2. **FSD Alignment**: Addons follow same FSD structure
3. **Opt-in**: Addons are optional, core app works without them
4. **Configuration Driven**: Feature inclusion/exclusion via config
5. **Type Safety**: Full TypeScript support
6. **Extension Points**: Addons contribute to UI via well-defined points

## Addon System Design

### 1. Addon Manifest Definition

```typescript
// src/app/addons/types.ts
export interface AddonManifest {
  id: string;
  name: string;
  version: string;
  description: string;
  author?: string;
  requires?: {
    core?: string;
    features?: string[];
  };
  permissions?: {
    features?: string[];
    roles?: string[];
  };
}

export interface Addon {
  manifest: AddonManifest;
  initialize?: (context: AddonContext) => void | Promise<void>;
  cleanup?: () => void;
}

export interface AddonContext {
  httpClient: typeof import("@/shared/api/http-client").httpClient;
  queryClient: typeof import("@/shared/lib/query-client").queryClient;
  session: {
    useSession: typeof import("@/processes/session").useSession;
    getAccessToken: typeof import("@/processes/session").getAccessToken;
    getTenantKey: typeof import("@/processes/session").getTenantKey;
  };
  i18n: typeof import("@lingui/core").i18n;
  extensions: {
    navigation: typeof import("./extension-points/navigation").navigationExtension;
    widgets: typeof import("./extension-points/widgets").widgetExtension;
  };
}
```

### 2. Addon Registry

```typescript
// src/app/addons/addon-registry.ts
export class AddonRegistry {
  private addons = new Map<string, Addon>();

  register(addon: Addon): void {
    if (this.addons.has(addon.manifest.id)) {
      throw new Error(`Addon ${addon.manifest.id} already registered`);
    }
    this.addons.set(addon.manifest.id, addon);
  }

  getAddon(id: string): Addon | undefined {
    return this.addons.get(id);
  }

  getAllAddons(): Addon[] {
    return Array.from(this.addons.values());
  }

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

### 3. Addon Configuration

Add addon configuration to the existing config system:

```typescript
// src/app/config/addons.ts
import { z } from "zod";

export const AddonConfigSchema = z.object({
  enabled: z.array(z.string()).default([]),
});

export type AddonConfig = z.infer<typeof AddonConfigSchema>;

export function getAddonConfig(): AddonConfig {
  // Read from environment or runtime config
  const enabledAddons = import.meta.env.VITE_ENABLED_ADDONS?.split(",") ?? [];
  return { enabled: enabledAddons };
}
```

### 4. Addon Loader

```typescript
// src/app/addons/addon-loader.ts
import { addonRegistry, type Addon } from "./addon-registry";
import { getAddonConfig } from "@/app/config";

// Static addon imports (for build-time inclusion)
const availableAddons: Record<string, () => Promise<{ default: Addon }>> = {
  // Example:
  // "project-management": () => import("@/addons/project-management"),
};

export async function loadAddons(): Promise<void> {
  const config = getAddonConfig();

  for (const addonId of config.enabled) {
    const loader = availableAddons[addonId];
    if (loader) {
      const { default: addon } = await loader();
      addonRegistry.register(addon);
    } else {
      console.warn(`Addon ${addonId} not found`);
    }
  }
}
```

### 5. Hybrid Routing Setup
- **Core routes**: Keep file-based (existing `src/pages/`)
- **Addon pages**: Add catch-all route under `/addons/` for dynamic rendering:

```typescript
// src/pages/_app/addons.tsx (new file)
import { createFileRoute, Outlet, useLocation } from "@tanstack/react-router";
import { addonRegistry } from "@/app/addons/addon-registry";
import { AuthGuard } from "@/shared/ui";
import { LoadingOverlay } from "@/shared/ui";
import { Suspense } from "react";

export const Route = createFileRoute("/_app/addons")({
  component: () => <Outlet />,
});

// Catch-all child route for addon pages
export const AddonCatchAllRoute = Route.createRoute({
  path: "/*",
  component: AddonRouteRenderer,
});

function AddonRouteRenderer() {
  const location = useLocation();
  const addons = addonRegistry.getAllAddons();
  
  // Match current path to addon routes
  const matchedRoute = addons
    .flatMap(p => p.routes || [])
    .find(route => route.path === location.pathname);

  if (!matchedRoute) {
    return <div>404 - Addon page not found</div>;
  }

  const Component = matchedRoute.component;

  // Use existing AuthGuard and FeatureGate for access control
  return (
    <AuthGuard>
      <Suspense fallback={<LoadingOverlay visible />}>
        <Component />
      </Suspense>
    </AuthGuard>
  );
}
```

### 6. Integration with App Bootstrap
Modify main.tsx and app.tsx to initialize addons:

```typescript
// src/main.tsx
async function bootstrap() {
  // ... existing MSW setup ...

  // Load and initialize addons
  const { loadAddons } = await import("@/app/addons/addon-loader");
  await loadAddons();

  // ... existing render code ...
}

// src/app/app.tsx
import { addonRegistry } from "./addons/addon-registry";
import { AddonContext } from "./addons/types";
import { navigationExtension } from "./addons/extension-points/navigation";
import { widgetExtension } from "./addons/extension-points/widgets";

export function App() {
  // ... existing state ...

  useEffect(() => {
    // Initialize addons after core systems are ready
    const initializeAddons = async () => {
      const context: AddonContext = {
        httpClient: (await import("@/shared/api/http-client")).httpClient,
        queryClient: (await import("@/shared/lib/query-client")).queryClient,
        session: {
          useSession: (await import("@/processes/session")).useSession,
          getAccessToken: (await import("@/processes/session")).getAccessToken,
          getTenantKey: (await import("@/processes/session")).getTenantKey,
        },
        i18n: (await import("@lingui/core")).i18n,
        extensions: {
          navigation: navigationExtension,
          widgets: widgetExtension,
        },
      };
      await addonRegistry.initializeAll(context);
    };

    initializeAddons().catch(console.error);
  }, []);

  // ... existing return ...
}
```

### 7. Feature Inclusion/Exclusion via Entitlements (Existing System)
Leverage the existing FeatureGate system for conditional rendering of addon features!

### 8. Addon Directory Structure

```
src/
├── addons/
│   ├── [addon-id]/
│   │   ├── index.ts              # Addon entry
│   │   ├── features/             # Addon-specific features (FSD)
│   │   ├── widgets/              # Addon widgets
│   │   ├── pages/                # Addon pages (if using dynamic routing)
│   │   ├── api/                  # Addon API modules
│   │   ├── types/                # Addon types
│   │   └── locales/              # Addon translations
│   ├── types.ts                  # Addon type definitions
│   ├── addon-registry.ts        # Registry implementation
│   ├── addon-loader.ts          # Addon loader
│   ├── extension-points/         # Extension points
│   └── providers/                # Addon providers
└── ... existing layers ...
```

### 9. Example Addon Implementation
```typescript
// src/addons/project-management/index.ts
import type { Addon } from "@/app/addons/types";
import { ProjectStatsWidget } from "./widgets/project-stats";

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
      component: () => import("./pages/projects").then(m => m.ProjectsPage),
      auth: true,
    },
  ],
  initialize: ({ extensions }) => {
    // Register nav item
    extensions.navigation.registerNavItem({
      id: "pm-projects",
      label: "Projects",
      to: "/_app/addons/projects",
      icon: IconChecklist,
      section: "workspace",
      order: 20,
    });

    // Register widget
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

### 10. Environment Variables

Add these to .env.example:

```env
# Comma-separated list of enabled addon IDs
VITE_ENABLED_ADDONS=
```

## Routing Strategy Decision: Approach 3 (Hybrid)
We've chosen the hybrid approach for addon routing:
- ✅ **Core routes** remain file-based (existing `src/pages/` structure)
- ✅ **Addons** register navigation items and widgets via extension points
- ✅ **Addon pages** use a catch-all route under `/addons/` for dynamic rendering
- ✅ **Entitlements/FeatureGate** handle access control for addon features

## Next Steps (Implementation Plan)
1. Create addon type definitions (src/app/addons/types.ts)
2. Implement addon registry (src/app/addons/addon-registry.ts)
3. Add addon config and loader
4. Implement extension points (navigation, widgets)
5. Add catch-all route for addon pages (src/pages/_app/addons.tsx)
6. Integrate addon initialization into app bootstrap
7. Update AppNav and dashboard to use extension points
8. Test with an example addon
9. Update architecture tests

## Performance Considerations

### 1. Bundle Size
- **Concern**: Including all addons in the main bundle could increase bundle size
- **Mitigation**: Use **dynamic imports** for addons (as shown in the example) — only load addons that are enabled via configuration
- **Tooling**: Use Vite's bundle analyzer to track addon bundle sizes

### 2. Initial Load Time
- **Concern**: Loading addons on app initialization could delay first render
- **Mitigation**:
  - Load addons **after** the core app is rendered and interactive
  - Use `Suspense` and `LoadingOverlay` for addon components (as shown in the example)
  - Prioritize loading critical addons first, lazy-load non-critical addons later

### 3. Route Matching
- **Concern**: Catch-all route matching could be slow with many addons
- **Mitigation**:
  - Keep the route matching logic simple (current linear search is fast for reasonable addon counts)
  - For large numbers of addons, pre-process routes into a `Map` for O(1) lookups

### 4. Extension Point Registrations
- **Concern**: Too many extension point registrations could cause delays
- **Mitigation**:
  - Keep registration logic lightweight (just pushing to an array)
  - Only register items when the addon is initialized (not on every render)

## Conclusion

This design allows us to:

- Enable/disable features via config
- Keep core app clean
- Maintain FSD architecture
- Leverage existing entitlements system
- Provide extension points for addons to contribute UI
- Provide a clear path for future feature modules
- Maintain good performance with dynamic imports and lazy loading
