# Foundation UI Platform Admin - Refactoring Plan

## 1. Architectural Foundation

This project already uses **Feature-Sliced Design (FSD)** as its architectural pattern (evidenced by `src/architecture.test.ts`). We will preserve and strengthen this pattern, strictly following the official [UI Coding Guidelines](../system-design-documentation/docs/coding-guidelines/ui.md].

## 2. Current State Assessment

### What's working well:

- ✅ Clear separation of layers (app/processes/pages/features/entities/shared)
- ✅ Centralized test selectors (`src/shared/lib/test-selectors.ts`)
- ✅ Well-organized API layer (`src/shared/api/`)
- ✅ Mantine + TanStack integration is solid
- ✅ i18n with Lingui is properly set up

### Areas for improvement (aligning with official guidelines):

1. **Empty widgets layer**: `src/widgets/` has only a `.gitkeep` and `index.ts` - no actual widgets
2. **Inline components in pages**: The admin dashboard page has multiple inline components that belong in widgets
3. **Underutilized entities layer**: `src/entities/` has only a `.gitkeep` - business entities aren't formally structured here
4. **Page title setup**: Current `page-title.ts` is a single file, not following the guideline's structured folder with `PageTitle` component, `usePageTitle` hook, and `ADMIN_TITLE` constant
5. **Query client location**: Guidelines say `queryClient` should be in `src/shared/lib/`, but it's currently in `src/app/app.tsx`
6. **Dashboard page title**: The admin dashboard doesn't use the proper `<PageTitle>` component with `ADMIN_TITLE`
7. **Inline utility functions**: The dashboard has inline functions (`getRefundStatusColor`, `getSeverityColor`) that should be moved to shared utilities
8. **Missing design tokens structure**: Guidelines mention `src/shared/lib/design-tokens/`, which is not present

## 3. Detailed Refactoring Steps (Aligned with Official Guidelines)

### 3.1. Fix Page Title Structure

- Refactor `src/shared/lib/page-title.ts` into a proper folder structure:
  ```
  shared/lib/page-title/
  ├── index.ts          # Public API
  ├── constants.ts      # APP_TITLE, ADMIN_TITLE
  ├── page-title.tsx    # <PageTitle> React component
  └── use-page-title.ts # usePageTitle hook
  ```
- Implement exactly as specified in the UI guidelines
- Update existing pages to use the new structure
- Add `appTitle={ADMIN_TITLE}` to all `/admin/*` routes

### 3.2. Move Query Client to Shared Lib

- Move `queryClient` definition from `src/app/app.tsx` to `src/shared/lib/query-client.ts`
- Export it from `shared/lib/index.ts`
- Update `app.tsx` to import it from there
- Pass it to both router context and QueryClientProvider as per guidelines

### 3.3. Extract Dashboard Components to Widgets

Create widgets for dashboard components currently in `src/pages/admin/index.tsx`. Each widget follows FSD conventions with proper public API:

- `widgets/dashboard-card-group/` (contains CardGroup component)
- `widgets/dashboard-sub-card/` (contains SubCard component)
- `widgets/dashboard-stat-value/` (contains StatValue and WidgetStatValue)
- `widgets/dashboard-tenant-signup-chart/` (contains TenantSignupChartCard)
- `widgets/dashboard-subscription-breakdown/`
- `widgets/dashboard-recent-critical-audit/`
- `widgets/dashboard-recent-refunds/`
- `widgets/dashboard-suspended-tenants/`
- Each widget has:
  - `index.ts` (public API)
  - `ui/` folder with the component(s)
  - `model/` folder if needed for any widget-specific logic
- Update `src/pages/admin/index.tsx` to import these widgets
- Add proper `<PageTitle>` to the dashboard page with `appTitle={ADMIN_TITLE}`

### 3.4. Extract Shared Utility Functions

- Create `shared/lib/date-utils.ts`: Centralize dayjs configuration and extensions
- Create `shared/lib/color-utils.ts`: Move `getRefundStatusColor` and `getSeverityColor` here
- Create `shared/lib/design-tokens/` (structure per guidelines, if not already present)
- Export all new utilities from `shared/lib/index.ts`
- Update dashboard to use these shared utilities

### 3.5. Strengthen the Entities Layer

Create entity structures for core business objects, each following FSD conventions:

- `entities/user/`
- `entities/tenant/`
- `entities/subscription/`
- `entities/announcement/`
- `entities/invitation/`
- Each entity has:
  - `index.ts` (public API)
  - `types.ts` (entity type definitions)
  - `model/` (optional: Zustand stores, business logic)
  - `ui/` (optional: entity-specific UI components)
  - `api/` (optional: API functions specific to this entity)

### 3.6. Validate Architecture

- Run `pnpm test:arch` to ensure FSD constraints are met
- Run `pnpm lint` to check for any linting issues
- Run `pnpm type-check` to ensure TypeScript is happy
- Run `pnpm build` to verify everything compiles correctly

### 3.7. Preserve Existing Functionality

- **Critical**: All existing features, components, and logic must remain fully functional after refactoring
- Test all pages and interactions to ensure nothing is broken

## 4. Benefits of This Refactoring

1. **Full guideline alignment**: The codebase will strictly follow the official UI coding guidelines, ensuring consistency with the tenant app
2. **Better reusability**: Widgets and utilities can be reused across multiple pages
3. **Cleaner pages**: Pages focus on routing and composition, not implementation details
4. **Clearer separation of concerns**: Entities encapsulate business logic, widgets handle UI composition
5. **Easier testing**: Smaller, more focused components are easier to test
6. **Scalability**: Adding new features will follow a clear, established pattern
7. **Better maintainability**: Structured page titles, shared query client, and centralized utilities make the codebase easier to maintain

## 5. Next Steps

1. Fix page title structure first (smallest change, highest guideline alignment impact)
2. Move query client to shared lib
3. Extract shared utility functions
4. Implement widget extraction (highest impact for code organization)
5. Then work on entities layer
6. Validate all changes with tests (`pnpm test:arch`, `pnpm lint`, `pnpm type-check`, `pnpm build`)
