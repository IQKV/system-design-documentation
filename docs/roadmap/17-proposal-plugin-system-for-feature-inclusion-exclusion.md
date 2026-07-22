# Proposal: Plugin System for Feature Inclusion/Exclusion

## Objective

Design a plugin system for foundation-ui-app that enables:

- Including/excluding features dynamically without core code changes
- Modular, self-contained feature modules
- Runtime plugin discovery and registration
- Integration with existing architecture (FSD, entitlements, routing, navigation)

## Current Architecture Recap (from foundation-ui-app/PLUGIN_ARCHITECTURE_REVIEW.md)

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

## Structure Improvements for Plugin Support

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

Proposed (enhanced for plugins):

```
src/
├── app/
│   ├── config/
│   │   ├── billing.ts
│   │   ├── index.ts
│   │   ├── plugins.ts        # New: Plugin configuration
│   │   ├── rollout-guards.ts
│   │   └── runtime-env.ts
│   ├── plugins/              # New: Plugin system core
│   │   ├── index.ts
│   │   ├── types.ts
│   │   ├── plugin-registry.ts
│   │   ├── plugin-loader.ts
│   │   ├── extension-points/ # New: Extension points for plugins
│   │   │   ├── index.ts
│   │   │   ├── navigation.ts # Navigation extension point
│   │   │   └── widgets.ts    # Widget extension point
│   │   └── providers/        # New: Plugin providers
│   │       └── PluginProvider.tsx
│   ├── app.tsx
│   ├── index.ts
│   └── theme.ts
├── processes/
├── pages/
├── widgets/
├── features/
├── entities/
├── shared/
├── plugins/                  # New: Plugin directory
│   └── [plugin-id]/
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

Extension points allow plugins to contribute to UI without modifying core code.

```typescript
// src/app/plugins/extension-points/navigation.ts
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
// src/app/plugins/extension-points/widgets.ts
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
import { navigationExtension } from "@/app/plugins/extension-points";

export function AppNav() {
  // ... existing code ...

  // Get plugin nav items
  const pluginWorkspaceItems = navigationExtension.getNavItems("workspace");
  const pluginAccountItems = navigationExtension.getNavItems("account");

  // Merge with core items
  const allNavItems = [...navItems, ...pluginWorkspaceItems];
  const allAccountItems = [...accountSubItems, ...pluginAccountItems];

  // ... rest of the component ...
}
```

## Design Principles

1. **Backward Compatibility**: No breaking changes to existing code
2. **FSD Alignment**: Plugins follow same FSD structure
3. **Opt-in**: Plugins are optional, core app works without them
4. **Configuration Driven**: Feature inclusion/exclusion via config
5. **Type Safety**: Full TypeScript support
6. **Extension Points**: Plugins contribute to UI via well-defined points

## Plugin System Design

### 1. Plugin Manifest Definition

```typescript
// src/app/plugins/types.ts
export interface PluginManifest {
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

export interface Plugin {
  manifest: PluginManifest;
  initialize?: (context: PluginContext) => void | Promise<void>;
  cleanup?: () => void;
}

export interface PluginContext {
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

### 2. Plugin Registry

```typescript
// src/app/plugins/plugin-registry.ts
export class PluginRegistry {
  private plugins = new Map<string, Plugin>();

  register(plugin: Plugin): void {
    if (this.plugins.has(plugin.manifest.id)) {
      throw new Error(`Plugin ${plugin.manifest.id} already registered`);
    }
    this.plugins.set(plugin.manifest.id, plugin);
  }

  getPlugin(id: string): Plugin | undefined {
    return this.plugins.get(id);
  }

  getAllPlugins(): Plugin[] {
    return Array.from(this.plugins.values());
  }

  async initializeAll(context: PluginContext): Promise<void> {
    for (const plugin of this.getAllPlugins()) {
      if (plugin.initialize) {
        await plugin.initialize(context);
      }
    }
  }
}

export const pluginRegistry = new PluginRegistry();
```

### 3. Plugin Configuration

Add plugin configuration to the existing config system:

```typescript
// src/app/config/plugins.ts
import { z } from "zod";

export const PluginConfigSchema = z.object({
  enabled: z.array(z.string()).default([]),
});

export type PluginConfig = z.infer<typeof PluginConfigSchema>;

export function getPluginConfig(): PluginConfig {
  // Read from environment or runtime config
  const enabledPlugins = import.meta.env.VITE_ENABLED_PLUGINS?.split(",") ?? [];
  return { enabled: enabledPlugins };
}
```

### 4. Plugin Loader

```typescript
// src/app/plugins/plugin-loader.ts
import { pluginRegistry, type Plugin } from "./plugin-registry";
import { getPluginConfig } from "@/app/config";

// Static plugin imports (for build-time inclusion)
const availablePlugins: Record<string, () => Promise<{ default: Plugin }>> = {
  // Example:
  // "project-management": () => import("@/plugins/project-management"),
};

export async function loadPlugins(): Promise<void> {
  const config = getPluginConfig();

  for (const pluginId of config.enabled) {
    const loader = availablePlugins[pluginId];
    if (loader) {
      const { default: plugin } = await loader();
      pluginRegistry.register(plugin);
    } else {
      console.warn(`Plugin ${pluginId} not found`);
    }
  }
}
```

### 5. Integration with App Bootstrap

Modify main.tsx and app.tsx to initialize plugins:

```typescript
// src/main.tsx
async function bootstrap() {
  // ... existing MSW setup ...

  // Load and initialize plugins
  const { loadPlugins } = await import("@/app/plugins/plugin-loader");
  await loadPlugins();

  // ... existing render code ...
}

// src/app/app.tsx
import { pluginRegistry } from "./plugins/plugin-registry";
import { PluginContext } from "./plugins/types";
import { navigationExtension } from "./plugins/extension-points/navigation";
import { widgetExtension } from "./plugins/extension-points/widgets";

export function App() {
  // ... existing state ...

  useEffect(() => {
    // Initialize plugins after core systems are ready
    const initializePlugins = async () => {
      const context: PluginContext = {
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
      await pluginRegistry.initializeAll(context);
    };

    initializePlugins().catch(console.error);
  }, []);

  // ... existing return ...
}
```

### 6. Feature Inclusion/Exclusion via Entitlements (Existing System)

Leverage the existing FeatureGate system for conditional rendering of plugin features!

### 7. Plugin Directory Structure

```
src/
├── plugins/
│   ├── [plugin-id]/
│   │   ├── index.ts              # Plugin entry
│   │   ├── features/             # Plugin-specific features (FSD)
│   │   ├── widgets/              # Plugin widgets
│   │   ├── pages/                # Plugin pages (if using dynamic routing)
│   │   ├── api/                  # Plugin API modules
│   │   ├── types/                # Plugin types
│   │   └── locales/              # Plugin translations
│   ├── types.ts                  # Plugin type definitions
│   ├── plugin-registry.ts        # Registry implementation
│   ├── plugin-loader.ts          # Plugin loader
│   ├── extension-points/         # Extension points
│   └── providers/                # Plugin providers
└── ... existing layers ...
```

### 8. Example Plugin Implementation

```typescript
// src/plugins/project-management/index.ts
import type { Plugin } from "@/app/plugins/types";
import { ProjectStatsWidget } from "./widgets/project-stats";

const plugin: Plugin = {
  manifest: {
    id: "project-management",
    name: "Project Management",
    version: "1.0.0",
    description: "Project management features",
  },
  initialize: ({ extensions }) => {
    // Register nav item
    extensions.navigation.registerNavItem({
      id: "pm-projects",
      label: "Projects",
      to: "/projects",
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

export default plugin;
```

### 9. Environment Variables

Add these to .env.example:

```env
# Comma-separated list of enabled plugin IDs
VITE_ENABLED_PLUGINS=
```

## Next Steps (Implementation Plan)

1. Create plugin type definitions (src/app/plugins/types.ts)
2. Implement plugin registry (src/app/plugins/plugin-registry.ts)
3. Add plugin config and loader
4. Implement extension points (navigation, widgets)
5. Integrate plugin initialization into app bootstrap
6. Update AppNav and dashboard to use extension points
7. Test with an example plugin
8. Update architecture tests

## Conclusion

This design allows us to:

- Enable/disable features via config
- Keep core app clean
- Maintain FSD architecture
- Leverage existing entitlements system
- Provide extension points for plugins to contribute UI
- Provide a clear path for future feature modules
