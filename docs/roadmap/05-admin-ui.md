# Platform Admin UI

## Overview

The Platform Admin UI is a dedicated administrative interface for platform operators to manage the entire multi-tenant SaaS platform. It provides comprehensive oversight and control over users, organizations, subscriptions, and system health across all tenants.

**Target Users:** Platform operators, support engineers, operations team

**Tech Stack:** React 19 + Mantine UI 8 + TanStack Router + TanStack Query + TypeScript + Lingui i18n

**Architecture:** Separate SPA deployed independently from the main user-facing UI, communicates through API Gateway with elevated privileges

**Internationalization:** Full i18n support with Lingui for multi-language admin interface (English as base language)

**UI/UX Inspiration:**

- **Makerkit Admin Dashboard** - Modern SaaS admin interface with comprehensive metrics, user management, and subscription oversight
- **Magento 1 Admin Panel** - Rich data grids, advanced filtering, bulk actions, and detailed entity management
- **OroCommerce Admin** - Enterprise-grade UI with dashboards, widgets, advanced search, and workflow management
- **Design Goals:** Professional, data-dense, efficient workflows, keyboard shortcuts, responsive tables, real-time updates

**Current Status:**

- ✅ **Backend APIs:** User admin CRUD, tenant management, subscription queries, billing settings, plan catalog management
- 🚧 **Admin UI:** In development - dashboard, user management, organization management
- 📋 **Planned:** Advanced operator actions (ban/unban, impersonation), system monitoring, audit logs

**Authorization Model:**

- `PLATFORM_OPERATOR` authority bypasses all tenant restrictions for cross-tenant administrative operations
- Regular tenant authorities (`TENANT_OWNER`, `ADMIN`, `MEMBER`) are restricted to their tenant context

---

## API Implementation Status

### ✅ Implemented APIs

**User Administration:**

- `GET /api/v1/iam/operator/users` - List users with pagination
- `GET /api/v1/iam/operator/users/{id}` - Get user details
- `POST /api/v1/iam/operator/users` - Create user with temp password
- `PUT /api/v1/iam/operator/users/{id}` - Full user update
- `PATCH /api/v1/iam/operator/users/{id}` - Partial user update
- `DELETE /api/v1/iam/operator/users/{id}` - Delete user (cascade)

**Tenant Management:**

- `GET /api/v1/iam/tenants/{tenantKey}` - Get tenant (TENANT_OWNER)
- `PATCH /api/v1/iam/tenants/{tenantKey}/status` - Update status (TENANT_OWNER)
- `POST /api/v1/iam/tenants/{tenantKey}/retry-provisioning` - Retry provisioning (TENANT_OWNER)

**Invitation Management:**

- `POST /api/v1/iam/tenants/{tenantKey}/invitations` - Send invitation
- `GET /api/v1/iam/tenants/{tenantKey}/invitations` - List invitations
- `DELETE /api/v1/iam/tenants/{tenantKey}/invitations/{id}` - Revoke invitation

**Subscription Management:**

- `GET /api/v1/billing/subscriptions/{tenantKey}` - Get all subscriptions
- `GET /api/v1/billing/subscriptions/{tenantKey}/active` - Get active subscription
- `GET /api/v1/billing/subscriptions/me` - Get subscriptions for current subject

**Billing Settings:**

- `GET /api/v1/billing/settings/{tenantKey}` - Get billing settings
- `PATCH /api/v1/billing/settings/{tenantKey}` - Update billing settings

**Plan Catalog:**

- `GET /api/v1/billing/plans` - List active plans
- `GET /api/v1/billing/plans/{planCode}` - Get plan details
- `POST /api/v1/billing/plans` - Create plan (PLATFORM_OPERATOR)
- `PUT /api/v1/billing/plans/{planCode}` - Update plan (PLATFORM_OPERATOR)
- `DELETE /api/v1/billing/plans/{planCode}` - Deactivate plan (PLATFORM_OPERATOR)

### 📋 Planned APIs

**User Admin Actions:**

- `POST /api/v1/iam/operator/users/{id}/ban` - Ban user (PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/users/{id}/unban` - Unban user (PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/users/{id}/impersonate` - Impersonate user (PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/users/{id}/unlock` - Unlock account (PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/users/{id}/verify-email` - Verify email (PLATFORM_OPERATOR)
- `GET /api/v1/iam/operator/users/{id}/memberships` - Get memberships (PLATFORM_OPERATOR)
- `GET /api/v1/iam/operator/users/{id}/activity` - Get activity log (PLATFORM_OPERATOR)

**Tenant Admin Actions:**

- `GET /api/v1/iam/operator/tenants` - List all tenants (paginated, PLATFORM_OPERATOR)
- `GET /api/v1/iam/operator/tenants/{tenantKey}` - Get tenant (platform operator view, PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/tenants/{tenantKey}/suspend` - Suspend tenant (PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/tenants/{tenantKey}/unsuspend` - Unsuspend tenant (PLATFORM_OPERATOR)
- `DELETE /api/v1/iam/operator/tenants/{tenantKey}` - Delete tenant (PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/tenants/{tenantKey}/transfer-ownership` - Transfer ownership (PLATFORM_OPERATOR)
- `GET /api/v1/iam/operator/tenants/{tenantKey}/export` - Export data (GDPR, PLATFORM_OPERATOR)

**Subscription Admin Actions:**

- `GET /api/v1/billing/operator/subscriptions` - List all subscriptions (PLATFORM_OPERATOR)
- `POST /api/v1/billing/operator/subscriptions/{id}/change-plan` - Change plan (PLATFORM_OPERATOR)
- `POST /api/v1/billing/operator/subscriptions/{id}/cancel` - Cancel subscription (PLATFORM_OPERATOR)
- `POST /api/v1/billing/operator/subscriptions/{id}/reactivate` - Reactivate (PLATFORM_OPERATOR)
- `POST /api/v1/billing/operator/subscriptions/{id}/apply-discount` - Apply discount (PLATFORM_OPERATOR)
- `POST /api/v1/billing/operator/subscriptions/{id}/extend-trial` - Extend trial (PLATFORM_OPERATOR)

**System Administration:**

- `GET /api/v1/iam/operator/dashboard/metrics` - Platform metrics (PLATFORM_OPERATOR)
- `GET /api/v1/iam/operator/audit-log` - Audit trail (PLATFORM_OPERATOR)
- `GET /api/v1/iam/operator/system/health` - Service health (PLATFORM_OPERATOR)
- `GET /api/v1/iam/operator/system/jobs` - Background jobs (PLATFORM_OPERATOR)
- `POST /api/v1/iam/operator/system/jobs/{jobName}/trigger` - Trigger job (PLATFORM_OPERATOR)

---

## Design Principles & UX Patterns

### Inspired by Best-in-Class Admin Interfaces

**From Makerkit Admin:**

- Clean, modern SaaS aesthetic
- Real-time metrics with live updates
- Comprehensive user and subscription management
- Revenue-focused dashboards
- Trial conversion tracking
- Customer lifecycle visualization

**From Magento 1 Admin:**

- Powerful data grids with advanced filtering
- Column management and customization
- Saved grid states and bookmarks
- Bulk actions with progress indicators
- Inline editing capabilities
- Keyboard shortcuts for power users
- Mass update operations
- Export functionality on every grid

**From OroCommerce Admin:**

- Enterprise-grade UI patterns
- Tabbed detail views with rich information
- Activity streams and timelines
- Contextual actions and workflows
- Advanced search with saved filters
- Dashboard widgets and customization
- Tag and categorization system
- Notes and internal communication

### Core UX Principles

1. **Efficiency First:**
   - Minimize clicks to complete common tasks
   - Keyboard shortcuts for all major actions
   - Bulk operations for repetitive tasks
   - Quick actions on hover
   - Inline editing where appropriate

2. **Data Density:**
   - Show maximum relevant information without clutter
   - Customizable columns and views
   - Collapsible sections for optional details
   - Progressive disclosure of complexity

3. **Discoverability:**
   - Clear navigation hierarchy
   - Command palette (Cmd+K) for quick access
   - Contextual help and tooltips
   - Breadcrumbs for deep navigation
   - Search everywhere functionality

4. **Feedback & Confirmation:**
   - Immediate feedback for all actions
   - Confirmation dialogs for destructive operations
   - Progress indicators for long-running tasks
   - Success/error notifications
   - Undo capability where possible

5. **Performance:**
   - Optimistic UI updates
   - Skeleton loading states
   - Virtual scrolling for large lists
   - Debounced search and filters
   - Cached data with smart invalidation

6. **Consistency:**
   - Uniform component usage
   - Consistent action patterns
   - Standard keyboard shortcuts
   - Predictable navigation
   - Unified color coding for statuses

---

## Core Features

> **Phase scope guide:** Feature sections are tagged with their target phase. Phase 2 MVP is the initial implementation target — build only what is tagged `Phase 2 MVP`. Advanced grid features (drag-and-drop, AND/OR filters, saved bookmarks, inline editing, undo/redo) are Phase 3–4 scope and must not be built during Phase 2 even if the feature section describes them in full detail.

### 1. Dashboard & Metrics `Phase 2 MVP`

**Overview Dashboard** — Real-time platform health and key metrics (inspired by Makerkit admin)

| Metric               | Description                                                    | Visualization       |
| -------------------- | -------------------------------------------------------------- | ------------------- |
| Active Users         | Total active users across all tenants (last 30 days)           | Number card + trend |
| Total Organizations  | Count of all organizations by status (ACTIVE, SUSPENDED, etc.) | Number card + chart |
| Active Subscriptions | Count of active paid subscriptions                             | Number card + trend |
| Trial Accounts       | Organizations currently in trial period with expiry countdown  | Number card + list  |
| Revenue Metrics      | MRR, ARR, churn rate (from Stripe data)                        | Charts + trends     |
| System Health        | Service status, database connections, queue depth              | Status indicators   |
| Recent Activity      | Latest signups, subscription changes, support tickets          | Activity feed       |
| Growth Metrics       | New signups, conversion rate, retention rate                   | Charts + trends     |

**Dashboard Features (Magento/OroCommerce-inspired):**

- **Customizable Widgets:** Drag-and-drop dashboard widgets
- **Date Range Selector:** Quick filters (today, last 7/30/90 days, custom range)
- **Real-time Updates:** SSE-based live metrics (Phase 4 — requires backend `/api/v1/iam/operator/dashboard/metrics/stream` endpoint)
- **Export Capabilities:** Export any widget data to CSV/Excel/PDF
- **Saved Views:** Save custom dashboard configurations
- **Comparison Mode:** Compare metrics across different time periods
- **Drill-down:** Click any metric to see detailed breakdown

**Filters:**

- Date range selector (last 7/30/90 days, custom range)
- Tenant status filter (all, active, suspended, trial)
- Subscription tier filter
- Region/timezone filter (if multi-region)

**Actions:**

- Export metrics to CSV/Excel
- Schedule automated reports (daily/weekly/monthly email)
- Configure alert thresholds
- Create custom dashboard views
- Share dashboard snapshots with team

---

### 2. User Management `Phase 2 MVP`

**User List** — Comprehensive view of all platform users (Magento-style data grid)

**Table Columns:**

- User ID (UUID)
- Email
- Username
- Email Verified (✓/✗)
- Account Status (Active, Locked, Suspended, Deleted)
- Organizations (count + list)
- Last Login
- Created At
- Actions

**Advanced Grid Features (Magento/OroCommerce-inspired):**

- **Column Management:** Show/hide columns, reorder, resize
- **Advanced Filtering:** Multi-condition filters with AND/OR logic
- **Saved Filters:** Save frequently used filter combinations
- **Inline Editing:** Quick edit user fields without opening detail view
- **Mass Actions:** Select multiple users for bulk operations
- **Export:** Export filtered results to CSV/Excel
- **Bookmarks:** Save grid state (filters, sorting, columns)
- **Quick Search:** Real-time search across all visible columns

**Search & Filters:**

- **Quick Search:** Search by email, username, user ID (with autocomplete)
- **Advanced Filters:**
  - Account status (multi-select)
  - Email verification status
  - Organization membership (by tenant key or name)
  - Last login date range
  - Creation date range
  - Has active subscription (yes/no)
  - Number of organizations (range)
- **Filter Presets:** "Recently Active", "Unverified Emails", "Locked Accounts", "Multi-Org Users"

**Bulk Actions:**

- Export selected users to CSV
- Send bulk email notifications
- Bulk suspend/unsuspend
- Bulk unlock accounts
- Add to organization (bulk invite)
- Tag users for follow-up

**User Detail View:**

**Tabbed Interface (OroCommerce-style):**

**Tab 1: Profile Information**

- User ID, email, username
- Email verification status with resend verification option
- Account creation date
- Last login timestamp
- Failed login attempts counter
- Account lock status and expiry
- Profile photo/avatar
- Timezone and locale preferences

**Tab 2: Organization Memberships**

- List of all organizations user belongs to
- Authority/role in each organization (TENANT_OWNER, ADMIN, MEMBER)
- Join date per organization
- Last activity per organization
- Quick navigation to organization details
- Remove from organization action

**Tab 3: Authentication History**

- Login history (last 100 logins with IP, user agent, timestamp, location)
- Active sessions with device info and "Revoke" action
- Token revocation history
- Password reset history
- Failed login attempts log
- MFA status and device list

**Tab 4: Billing Information**

- Associated Stripe customer IDs (per tenant)
- Active subscriptions across organizations
- Payment history with invoice links
- Billing email addresses
- Payment methods on file
- Lifetime value (LTV)

**Tab 5: Activity Log**

- Audit trail of user actions (paginated, filterable)
- Organization changes
- Permission changes
- Support ticket history
- API usage statistics
- Feature usage tracking

**Tab 6: Notes & Tags**

- Internal operator notes (not visible to user)
- Tags for categorization
- Support ticket references
- Follow-up reminders

**Platform Operator Actions (Current Implementation):**

- **Delete Account:** Permanently deletes user and all memberships (cascade) - `DELETE /api/v1/iam/operator/users/{id}`
- **Create User:** Create user with random temporary password - `POST /api/v1/iam/operator/users`
- **Update User:** Full or partial profile update - `PUT/PATCH /api/v1/iam/operator/users/{id}`

**Platform Operator Actions (To Be Implemented):**

- **Ban User:** Suspend account with reason and duration (temporary/permanent)
- **Unban User:** Restore suspended account
- **Verify Email:** Manually verify email address
- **Unlock Account:** Clear failed login attempts
- **Revoke All Sessions:** Global signout for user
- **Impersonate User:** Login as user for support purposes (with audit trail)
- **Send Notification:** Send direct email to user

---

### 3. Organization Management `Phase 2 MVP`

**Organization List** — All tenants across the platform (Magento-style grid)

**Table Columns:**

- Tenant Key (8-char NanoID)
- Organization Name
- Status (PROVISIONING, ACTIVE, SUSPENDED, DELETED, PROVISIONING_FAILED)
- Owner (email + link to user)
- Member Count
- Subscription Status (Trial, Active, Past Due, Canceled)
- MRR (Monthly Recurring Revenue)
- Created At
- Last Activity
- Actions

**Advanced Grid Features:**

- **Column Management:** Customizable columns
- **Advanced Filtering:** Multi-condition filters
- **Saved Views:** "Active Trials", "Past Due", "High Value", "Recently Created"
- **Inline Actions:** Quick suspend/unsuspend without opening detail
- **Color Coding:** Status-based row highlighting
- **Sorting:** Multi-column sorting
- **Pagination:** Configurable page size (20/50/100/200)

**Search & Filters:**

- **Quick Search:** Search by tenant key, organization name, owner email
- **Advanced Filters:**
  - Status (multi-select with color indicators)
  - Subscription status (multi-select)
  - Creation date range
  - Member count range (e.g., 1-5, 6-20, 21-50, 51+)
  - Last activity date
  - MRR range
  - Plan type
  - Has payment method (yes/no)
  - Trial expiring soon (next 7/14/30 days)
- **Filter Presets:** "Needs Attention", "High Value Customers", "Churning Risk", "New This Month"

**Bulk Actions:**

- Export organizations to CSV/Excel
- Bulk suspend/unsuspend
- Send bulk notifications to owners
- Apply discount code to multiple orgs
- Tag organizations
- Assign to support agent

**Organization Detail View:**

**Tabbed Interface (OroCommerce-style with sidebar navigation):**

**Tab 1: Overview**

- Tenant Key (with copy button)
- Organization Name (editable inline)
- Status with status history timeline
- Creation date and provisioning details
- Provisioning details (schema name, migration status)
- Last activity timestamp
- Database schema size and growth chart
- Quick stats: Members, Subscriptions, Storage Used, API Calls
- Owner information with quick actions

**Tab 2: Members**

- Data grid with all members
- Columns: Name, Email, Authority, Join Date, Last Active, Actions
- Invitation history (pending, accepted, expired, revoked)
- Member activity timeline
- Inline actions: Change role, Remove member
- Bulk actions: Change roles, Send notifications
- Add member / Send invitation button
- Resend invitations
- Export member list

**Tab 3: Subscription & Billing**

- Current subscription plan (with upgrade/downgrade actions)
- Subscription status and period
- Billing cycle visualization (timeline)
- Billing settings (company name, billing email, tax ID)
- Stripe customer ID with direct link to Stripe dashboard
- Payment history table (sortable, filterable)
- Invoice list with download links and status
- Upcoming invoice preview
- Subscription change history with audit trail
- Payment method on file
- Billing alerts and notifications

**Tab 4: Usage & Limits**

- Current plan limits (users, storage, API calls, features)
- Usage metrics vs limits with progress bars
- Feature flags and entitlements (toggle view)
- Usage trends over time (charts)
- API usage breakdown by endpoint
- Storage breakdown by type
- Overage alerts and notifications
- Usage forecast

**Tab 5: Activity & Audit**

- Organization activity log (filterable, searchable)
- Member changes timeline
- Subscription changes timeline
- Support tickets linked to this org
- System events (provisioning, suspension, migrations)
- API access logs
- Export activity log

**Tab 6: Settings & Configuration**

- Organization settings
- Feature flags (enable/disable features)
- API keys and webhooks
- Custom domain settings
- Branding/white-label settings
- Data retention policies
- Backup and restore options

**Tab 7: Notes & Support**

- Internal operator notes
- Support ticket history
- Tags and categories
- Assigned support agent
- Follow-up reminders
- Customer health score

**Platform Operator Actions (Current Implementation):**

- **Retry Provisioning:** For organizations in PROVISIONING_FAILED status - `POST /api/v1/iam/tenants/{tenantKey}/retry-provisioning` (TENANT_OWNER)
- **Update Status:** Change tenant status - `PATCH /api/v1/iam/tenants/{tenantKey}/status` (TENANT_OWNER)

**Platform Operator Actions (To Be Implemented):**

- **Suspend Organization:** Temporarily disable with reason
- **Unsuspend Organization:** Restore suspended organization
- **Delete Organization:** Soft delete with data retention (GDPR-compliant)
- **Force Schema Migration:** Run pending database migrations
- **Transfer Ownership:** Change organization owner
- **Export Data:** Generate data export for organization (GDPR)
- **Impersonate Owner:** Login as organization owner for support

---

### 4. Subscription & Billing Management `Phase 3`

**Subscription List** — Global view of all subscriptions

**Table Columns:**

- Subscription ID (Stripe sub_xxx)
- Organization/User (depending on rollout mode)
- Plan Name
- Status (Active, Trialing, Past Due, Canceled, Unpaid)
- MRR (Monthly Recurring Revenue)
- Current Period (start - end)
- Next Billing Date
- Cancel at Period End (✓/✗)
- Created At
- Actions

**Search & Filters:**

- Search by subscription ID, organization, customer email
- Filter by status
- Filter by plan
- Filter by billing period (monthly, annual)
- Filter by MRR range
- Filter by next billing date range

**Metrics:**

- Total MRR/ARR
- Subscription distribution by plan
- Churn rate
- Trial conversion rate
- Average subscription lifetime

**Subscription Detail View:**

**Subscription Information:**

- Subscription ID with link to Stripe
- Organization/User details
- Plan details (name, price, billing period)
- Status and status history
- Current period dates
- Next billing date
- Cancellation details (if applicable)

**Payment History:**

- Invoice list with status
- Payment method details
- Failed payment attempts
- Refund history

**Platform Operator Actions (Current Implementation):**

- **View Subscriptions:** Query tenant subscriptions from local cache - `GET /api/v1/billing/subscriptions/{tenantKey}` (TENANT_OWNER)
- **View Active Subscription:** Get active subscription - `GET /api/v1/billing/subscriptions/{tenantKey}/active` (TENANT_OWNER)
- **View Billing Settings:** Get billing configuration - `GET /api/v1/billing/settings/{tenantKey}` (TENANT_OWNER)
- **Update Billing Settings:** Partial update of billing info - `PATCH /api/v1/billing/settings/{tenantKey}` (TENANT_OWNER)

**Platform Operator Actions (To Be Implemented):**

- **Change Plan:** Upgrade/downgrade with proration
- **Cancel Subscription:** Immediate or end-of-period
- **Reactivate Subscription:** Restore canceled subscription
- **Apply Discount:** Add coupon or discount code
- **Extend Trial:** Add trial days
- **Pause Subscription:** Temporarily pause billing
- **Update Payment Method:** Change payment details
- **Issue Refund:** Process refund with reason
- **Send Invoice:** Manually send invoice email

**Plan Catalog Management (Current Implementation):**

**Plan List:**

- Plan Code (unique identifier)
- Display Name
- Billing Period (MONTHLY, ANNUAL)
- Price (in minor units, e.g., cents)
- Currency (ISO 4217, default USD)
- Scope (TENANT, USER)
- Feature Set (JSON)
- Active Status
- Actions

**Plan Actions (Current):**

- **List Plans:** `GET /api/v1/billing/plans` - List all active plans (authenticated users)
- **Get Plan:** `GET /api/v1/billing/plans/{planCode}` - Get plan details (authenticated users)
- **Create Plan:** `POST /api/v1/billing/plans` - Create new plan (PLATFORM_OPERATOR)
- **Update Plan:** `PUT /api/v1/billing/plans/{planCode}` - Update plan details (PLATFORM_OPERATOR)
- **Deactivate Plan:** `DELETE /api/v1/billing/plans/{planCode}` - Soft delete by setting active=false (PLATFORM_OPERATOR)

**Plan Actions (To Be Implemented):**

- View subscriptions using plan
- Duplicate plan for quick creation

---

### 5. Platform Actions & Tools `Phase 2 MVP (basic) → Phase 4 (impersonation)`

**User Actions:**

**Ban/Unban User:**

- Reason selection (abuse, payment failure, terms violation, other)
- Custom reason text
- Duration (temporary with date picker, permanent)
- Notification option (send email to user)
- Audit trail entry

**Delete Account:**

- Confirmation dialog with impact warning
- Data retention policy display
- Option to export user data before deletion
- Cascade options (remove from organizations, delete billing data)
- GDPR compliance notes
- Audit trail entry

**Impersonation:**

- User selection with search
- Reason for impersonation (required)
- Session duration limit (default 30 minutes)
- Audit trail with full session recording
- Visual indicator in UI when impersonating
- Exit impersonation button
- Restrictions: cannot change password, cannot delete account, cannot access billing

**Organization Actions:**

**Suspend/Unsuspend Organization:**

- Reason selection
- Custom reason text
- Notification options (notify owner, notify all members)
- Impact warning (users lose access, data preserved)
- Audit trail entry

**Delete Organization:**

- Confirmation with organization name verification
- Impact assessment (member count, data size, active subscriptions)
- Data export option before deletion
- Cascade options (cancel subscriptions, remove members)
- GDPR compliance notes
- Audit trail entry

**Retry Provisioning:**

- Available for organizations in PROVISIONING_FAILED status
- Shows provisioning error details
- Option to force schema recreation
- Real-time provisioning progress
- Automatic status update on completion

---

### 6. System Administration `Phase 4`

**Service Health:**

- Service status dashboard (IAM, Gateway, Billing)
- Database connection pool status
- RabbitMQ queue depths and consumer status
- Redis cache hit rates (if applicable)
- API response time metrics
- Error rate monitoring

**Background Jobs:**

- ShedLock job status and last execution
- Job execution history
- Failed job retry queue
- Manual job trigger capability

**Configuration:**

- Platform rollout mode display (MULTI_TENANT, SINGLE_TENANT)
- Feature flags management
- System-wide settings
- Maintenance mode toggle

**Audit Log:**

- Global audit trail of all platform operator actions
- Filterable by operator user, action type, date range
- Export capability
- Retention policy display

---

## Technical Architecture

### Authentication & Authorization

**Platform Operator Authentication:**

- Platform operators have `PLATFORM_OPERATOR` authority in their JWT
- MFA required for all platform operator accounts
- Session timeout: 30 minutes of inactivity
- IP whitelist support for additional security
- Separate authentication flow with elevated security requirements

**Token Storage Strategy:**

- Access token stored in **memory only** (JavaScript variable in `operator-session` Zustand store) — never in `localStorage` or `sessionStorage`, never in a non-httpOnly cookie.
- Refresh token stored in an **httpOnly, Secure, SameSite=Strict cookie** — inaccessible to JavaScript, immune to XSS.
- On page reload, the access token is gone from memory; the Axios interceptor detects the missing token, uses the httpOnly refresh cookie to silently obtain a new access token, and restores the session transparently.
- On explicit logout, the server invalidates the refresh token and the client clears the Zustand store. The httpOnly cookie is cleared server-side via `Set-Cookie` with `Max-Age=0`.
- This strategy means the admin UI has no persistent token in JavaScript-accessible storage — XSS cannot steal a long-lived credential.

**Current Authorization Model:**

```
User Authorities (per tenant):
├── TENANT_OWNER    # Full tenant management within their tenant
├── ADMIN           # User management, invitations within their tenant
└── MEMBER          # Basic access within their tenant

Platform Authority:
└── PLATFORM_OPERATOR  # Full platform access, bypasses all tenant restrictions
```

**Authority Hierarchy & Bypass Logic:**

```
Authorization Check Flow:
1. Check if user has PLATFORM_OPERATOR authority
   └─> YES: Bypass tenant context validation, allow cross-tenant operations
   └─> NO: Continue to tenant-scoped checks

2. Check tenant-scoped authority (TENANT_OWNER, ADMIN, MEMBER)
   └─> Validate JWT tenant claim matches requested tenant
   └─> Enforce authority-based permissions within tenant
```

**Current Implementation:**

- `PLATFORM_OPERATOR` authority required for plan catalog mutations (POST/PUT/DELETE)
- `TENANT_OWNER` authority required for tenant management endpoints (when not platform operator)
- `TENANT_OWNER` or `ADMIN` authority required for invitation management (when not platform operator)
- User admin endpoints (`/api/v1/iam/operator/users`) should be secured with `PLATFORM_OPERATOR` authority
- Tenant context validation enforced via JWT claims and path variable matching **for non-platform users**

**Planned Enhancement - Platform Authority Bypass:**

```java
// Example authorization logic to be implemented
@PreAuthorize("hasAuthority('PLATFORM_OPERATOR') or " +
              "(hasAuthority('TENANT_OWNER') and @tenantSecurity.matchesCurrent(#tenantKey))")
public TenantResponse getTenant(@PathVariable String tenantKey) {
    // PLATFORM_OPERATOR can access any tenant
    // TENANT_OWNER can only access their own tenant
}
```

**Security Notes:**

- **Platform Authority:** `PLATFORM_OPERATOR` bypasses tenant isolation for all administrative operations
- **Tenant-Scoped Authorities:** `TENANT_OWNER`, `PLAFORM_OPERATOR`, `MEMBER` are restricted to their tenant context
- **Cross-Tenant Protection:** Non-platform users cannot access other tenants' data (enforced via `TenantContextMismatchException`)
- **Audit Trail:** All platform-level operations must be logged with operator identity for compliance
- **JWT Claims:** Platform operators have special JWT claims indicating their elevated privileges

**Platform Operator Authorities:**

```
PLATFORM_OPERATOR (bypasses all tenant restrictions)
├── VIEW_USERS              # View all users across all tenants
├── MANAGE_USERS            # Create, update, delete, ban, unlock users
├── VIEW_ORGANIZATIONS      # View all tenants/organizations
├── MANAGE_ORGANIZATIONS    # Suspend, delete, transfer ownership
├── VIEW_BILLING            # View all subscriptions and billing data
├── MANAGE_BILLING          # Modify subscriptions, apply discounts
├── MANAGE_PLAN_CATALOG     # Create, update, deactivate plans (implemented)
├── IMPERSONATE_USER        # Login as any user for support
├── VIEW_AUDIT_LOG          # Access global audit trail
├── VIEW_METRICS            # View platform-wide metrics
├── VIEW_SYSTEM_HEALTH      # Monitor service health
└── MANAGE_SYSTEM           # System configuration, job management
```

**Implementation Strategy:**

1. **Service Layer Authorization:**

```java
// Check for platform operator authority first, then fall back to tenant-scoped
if (hasPlatformOperatorAuthority(jwt)) {
    // Bypass tenant validation - platform operator can access any tenant
    return service.getAnyTenant(tenantKey);
} else if (hasTenantAuthority(jwt, tenantKey)) {
    // Enforce tenant context matching for regular users
    return service.getTenantForOwner(tenantKey, jwt);
} else {
    throw new AccessDeniedException();
}
```

2. **Gateway Context Propagation:**

- Platform operators get special header: `X-Platform-Operator: true`
- Downstream services check this header to bypass tenant isolation
- Regular users never get this header (stripped by gateway)
- Gateway validates `PLATFORM_OPERATOR` authority before adding header

3. **Audit Requirements:**

- All platform operator actions logged with:
  - Operator user ID and email
  - Target tenant/user
  - Action performed
  - Timestamp and IP address
  - Reason (for sensitive operations like impersonation, deletion)

---

### API Integration

**Current API Endpoints (Implemented):**

**IAM Service - User Admin:**

```
GET    /api/v1/iam/operator/users                    # List users (paginated)
GET    /api/v1/iam/operator/users/{id}               # Get user by ID
POST   /api/v1/iam/operator/users                    # Create user
PUT    /api/v1/iam/operator/users/{id}               # Replace user (full update)
PATCH  /api/v1/iam/operator/users/{id}               # Partial update user
DELETE /api/v1/iam/operator/users/{id}               # Delete user
```

**IAM Service - Tenant Management:**

```
GET    /api/v1/iam/tenants/{tenantKey}            # Get tenant by key (TENANT_OWNER)
PATCH  /api/v1/iam/tenants/{tenantKey}/status     # Update tenant status (TENANT_OWNER)
POST   /api/v1/iam/tenants/{tenantKey}/retry-provisioning  # Retry provisioning (TENANT_OWNER)
```

**IAM Service - Invitations:**

```
POST   /api/v1/iam/tenants/{tenantKey}/invitations           # Send invitation (TENANT_OWNER/ADMIN)
GET    /api/v1/iam/tenants/{tenantKey}/invitations           # List invitations (TENANT_OWNER/ADMIN)
DELETE /api/v1/iam/tenants/{tenantKey}/invitations/{id}      # Revoke invitation (TENANT_OWNER/ADMIN)
GET    /api/v1/iam/invitations/{token}                       # Preview invitation (public)
POST   /api/v1/iam/invitations/{token}/accept                # Accept invitation (public)
```

**Billing Service - Subscriptions:**

```
GET    /api/v1/billing/subscriptions/{tenantKey}/active      # Get active subscription (TENANT_OWNER)
GET    /api/v1/billing/subscriptions/{tenantKey}             # Get all subscriptions (TENANT_OWNER)
GET    /api/v1/billing/subscriptions/me/active               # Get active for current subject
GET    /api/v1/billing/subscriptions/me                      # Get all for current subject
```

**Billing Service - Billing Settings:**

```
GET    /api/v1/billing/settings/{tenantKey}       # Get billing settings (TENANT_OWNER)
PATCH  /api/v1/billing/settings/{tenantKey}       # Update billing settings (TENANT_OWNER)
```

**Billing Service - Plan Catalog:**

```
GET    /api/v1/billing/plans                      # List all active plans (authenticated)
GET    /api/v1/billing/plans/{planCode}           # Get plan by code (authenticated)
POST   /api/v1/billing/plans                      # Create plan (PLATFORM_OPERATOR)
PUT    /api/v1/billing/plans/{planCode}           # Update plan (PLATFORM_OPERATOR)
DELETE /api/v1/billing/plans/{planCode}           # Deactivate plan (PLATFORM_OPERATOR)
```

**Platform Operator API Endpoints (To Be Implemented):**

```
# Dashboard & Metrics
GET    /api/v1/iam/operator/dashboard/metrics        # Platform-wide metrics

# User Admin Actions (extend existing)
POST   /api/v1/iam/operator/users/{id}/ban           # Ban user
POST   /api/v1/iam/operator/users/{id}/unban         # Unban user
POST   /api/v1/iam/operator/users/{id}/impersonate   # Impersonate user
POST   /api/v1/iam/operator/users/{id}/unlock        # Unlock account
POST   /api/v1/iam/operator/users/{id}/verify-email  # Manually verify email
GET    /api/v1/iam/operator/users/{id}/memberships   # Get user's org memberships
GET    /api/v1/iam/operator/users/{id}/activity      # Get user activity log

# Organization Admin (extend existing)
GET    /api/v1/iam/operator/tenants                  # List all tenants (paginated)
GET    /api/v1/iam/operator/tenants/{tenantKey}      # Get tenant details (platform operator view)
POST   /api/v1/iam/operator/tenants/{tenantKey}/suspend      # Suspend organization
POST   /api/v1/iam/operator/tenants/{tenantKey}/unsuspend    # Unsuspend organization
DELETE /api/v1/iam/operator/tenants/{tenantKey}      # Delete organization
POST   /api/v1/iam/operator/tenants/{tenantKey}/transfer-ownership  # Transfer ownership
GET    /api/v1/iam/operator/tenants/{tenantKey}/export       # Export org data (GDPR)
GET    /api/v1/iam/operator/tenants/{tenantKey}/members      # List org members
GET    /api/v1/iam/operator/tenants/{tenantKey}/activity     # Get org activity log

# Subscription Admin (extend existing)
GET    /api/v1/billing/operator/subscriptions        # List all subscriptions (paginated)
GET    /api/v1/billing/operator/subscriptions/{id}   # Get subscription details
POST   /api/v1/billing/operator/subscriptions/{id}/change-plan    # Change plan
POST   /api/v1/billing/operator/subscriptions/{id}/cancel         # Cancel subscription
POST   /api/v1/billing/operator/subscriptions/{id}/reactivate     # Reactivate subscription
POST   /api/v1/billing/operator/subscriptions/{id}/apply-discount # Apply discount
POST   /api/v1/billing/operator/subscriptions/{id}/extend-trial   # Extend trial

# System Administration
GET    /api/v1/iam/operator/audit-log                # Global audit trail
GET    /api/v1/iam/operator/system/health            # Service health status
GET    /api/v1/iam/operator/system/jobs              # Background job status
POST   /api/v1/iam/operator/system/jobs/{jobName}/trigger  # Trigger job manually

# Operator Preferences
GET    /api/v1/iam/operator/operators/me/preferences # Get operator preferences (locale, UI settings)
PATCH  /api/v1/iam/operator/operators/me/preferences # Save operator preferences

# Dashboard Streaming (Phase 4)
GET    /api/v1/iam/operator/dashboard/metrics/stream # SSE stream for live metric updates (EventSource)
```

### UI Components (Mantine)

**Key Components:**

- **DataTable** (mantine-datatable) for all list views with sorting, filtering, pagination, column management
- **Modal** for confirmation dialogs and detail views
- **Drawer** for side panels and quick actions
- **Notifications** for action feedback (toast notifications)
- **Charts** (mantine-charts with Recharts) for metrics visualization
- **Badge** for status indicators with color coding
- **ActionIcon** and **Menu** for row actions and bulk actions
- **Tabs** for detail view sections
- **DateRangePicker** for date filters
- **Select** and **MultiSelect** for filters with search
- **TextInput** with search icon for search fields
- **Spotlight** (Cmd+K) for global search and quick actions
- **Timeline** for activity feeds and history
- **Progress** for usage metrics and limits
- **Skeleton** for loading states
- **Empty State** components for no data scenarios

**Layout:**

- **AppShell** with collapsible navigation sidebar
- **Header** with global search, notifications, user menu
- **Breadcrumb** navigation for deep hierarchies
- **Responsive design** for tablet/desktop (mobile not prioritized)
- **Dark mode** support (optional)
- **Keyboard shortcuts** for power users (Magento-inspired)

**Advanced UI Features (Magento/OroCommerce-inspired):**

- **Grid State Persistence:** Remember column order, filters, sorting per user
- **Bookmarks:** Save and share grid configurations
- **Quick Actions:** Hover actions on table rows
- **Inline Editing:** Edit fields without opening detail view
- **Batch Operations:** Progress indicator for bulk actions
- **Contextual Help:** Tooltips and help text throughout
- **Undo/Redo:** For destructive actions (with timeout)
- **Keyboard Navigation:** Full keyboard support for grids and forms
- **Command Palette:** Cmd+K for quick navigation and actions

### State Management

**TanStack Query:**

- Server state caching for all API data
- Automatic refetch on window focus
- Optimistic updates for mutations
- Pagination and infinite scroll support

**Zustand:**

- UI state (sidebar collapsed, active filters)
- Platform operator session state
- Impersonation mode indicator
- User preferences (theme, locale, grid settings)

**nuqs:**

- URL-based filter state for shareable links
- Pagination state in URL

**Locale Management:**

- User locale preference stored in browser localStorage
- Fallback to browser language
- Per-operator locale setting (persisted in backend)
- Real-time locale switching without page reload

---

## Internationalization (i18n)

### Overview

The Platform Admin UI provides full internationalization support using **Lingui** for multi-language admin interfaces, enabling platform operators worldwide to use the admin panel in their preferred language.

**Supported Languages (Initial):**

- 🇬🇧 English (en) - Default
- 🇷🇺 Russian (ru)
- 🇺🇦 Ukrainian (uk)
- 🇩🇪 German (de)
- 🇪🇸 Spanish (es)
- 🇫🇷 French (fr)
- 🇮🇹 Italian (it)
- 🇵🇹 Portuguese (pt)
- 🇯🇵 Japanese (ja)
- 🇨🇳 Chinese Simplified (zh-CN)

### Implementation with Lingui

**Technology Stack:**

- **@lingui/core** - Core i18n functionality
- **@lingui/react** - React integration with Trans component and useLingui hook
- **@lingui/core/macro** and **@lingui/react/macro** - Compile-time message extraction (v6 macro paths)
- **@lingui/cli** - Message extraction and compilation tools
- **PO format** - Industry-standard translation format

**Message Extraction:**

```bash
# Extract messages from source code
pnpm messages:extract

# Compile PO catalogs to TypeScript
pnpm messages:compile
```

**Usage in Components:**

```tsx
// Lingui v6 — always use /macro import paths
import { Trans, useLingui } from "@lingui/react/macro";
import { msg } from "@lingui/core/macro";

function UserList() {
  const { _ } = useLingui();

  return (
    <div>
      <h1>
        <Trans>User Management</Trans>
      </h1>
      <Button>{_(msg`Create User`)}</Button>
      <p>
        <Trans>Total users: {userCount}</Trans>
      </p>
    </div>
  );
}
```

### Locale Detection & Selection

**Detection Strategy:**

1. Check operator's saved preference (from backend API)
2. Check browser localStorage (`admin-locale`)
3. Detect browser language (`navigator.language`)
4. Fallback to English (en)

**Locale Switcher:**

- Language selector in header/user menu
- Shows current language with flag icon
- Dropdown with all available languages
- Instant switching without page reload
- Saves preference to backend and localStorage

**Locale Persistence:**

```typescript
// Store in localStorage (primary client-side persistence)
localStorage.setItem("admin-locale", "ru");

// Store in backend (operator preferences — planned, Phase 2)
// PATCH /api/v1/iam/operator/operators/me/preferences
// { "locale": "ru" }
// Note: this endpoint is in the planned API list and not yet implemented.
// Until available, localStorage is the sole persistence mechanism.
```

### Translation Coverage

**Fully Translated Elements:**

- Navigation menu and breadcrumbs
- Dashboard metrics and labels
- Data grid headers and filters
- Form labels and placeholders
- Validation error messages
- Success/error notifications
- Confirmation dialogs
- Help text and tooltips
- Status labels and badges
- Action button labels
- Empty state messages
- Loading indicators

**Dynamic Content:**

- User-generated content (org names, user names) - NOT translated
- System-generated messages - Translated
- Email addresses, IDs, technical data - NOT translated
- Dates and numbers - Localized using Intl API

### Number & Date Formatting

**Intl API Integration:**

```typescript
// Number formatting (currency, percentages)
const formatter = new Intl.NumberFormat(locale, {
  style: "currency",
  currency: "USD",
});
formatter.format(1234.56); // $1,234.56 (en) / 1 234,56 $ (fr)

// Date formatting
const dateFormatter = new Intl.DateTimeFormat(locale, {
  year: "numeric",
  month: "long",
  day: "numeric",
});
dateFormatter.format(new Date()); // December 15, 2024 (en) / 15 décembre 2024 (fr)

// Relative time
const rtf = new Intl.RelativeTimeFormat(locale, { numeric: "auto" });
rtf.format(-1, "day"); // yesterday (en) / вчера (ru)
```

**Mantine Integration:**

- Mantine components automatically respect locale for date pickers
- Number inputs use locale-specific formatting
- Currency inputs with proper symbol placement

### Pluralization

**Lingui Plural Support:**

```tsx
// Lingui v6 — Plural is imported from @lingui/react/macro
import { Plural } from "@lingui/react/macro";

<Plural
  value={userCount}
  one="# user"
  other="# users"
/>

// Russian example (complex pluralization)
<Plural
  value={userCount}
  one="# пользователь"
  few="# пользователя"
  many="# пользователей"
  other="# пользователей"
/>
```

### RTL (Right-to-Left) Support

**Planned for Arabic/Hebrew:**

- Automatic layout flip for RTL languages
- Mantine RTL support via `dir="rtl"` on root element
- Mirror icons and layouts
- Adjust text alignment
- Flip navigation and menus

### Translation Workflow

**For Developers:**

1. Write code with `<Trans>` and `msg` macros
2. Run `pnpm messages:extract` to extract messages
3. Commit updated PO files to repository
4. Run `pnpm messages:compile` before build

**For Translators:**

1. Receive PO files from repository
2. Translate using PO editor (Poedit, Lokalise, Crowdin)
3. Return translated PO files
4. Developers commit translations and rebuild

**Translation Tools:**

- **Poedit** - Desktop PO editor
- **Lokalise** - Cloud translation management (optional)
- **Crowdin** - Community translation platform (optional)
- **GitHub Actions** - Automated extraction on PR

### Context & Comments

**Providing Context for Translators:**

```tsx
// Add context for ambiguous terms
<Trans context="user action">Delete</Trans>
<Trans context="database operation">Delete</Trans>

// Add comments for translators
<Trans comment="Button label for creating a new user">
  Create User
</Trans>
```

### Locale-Specific Features

**Date/Time Display:**

- 12-hour vs 24-hour format based on locale
- Week starts on Sunday (US) vs Monday (EU)
- Date format: MM/DD/YYYY (US) vs DD/MM/YYYY (EU) vs YYYY-MM-DD (ISO)

**Currency Display:**

- Symbol placement: $100 (US) vs 100€ (EU)
- Decimal separator: 1,234.56 (US) vs 1.234,56 (EU)
- Thousand separator: 1,234 (US) vs 1 234 (FR)

**Name Formatting:**

- First name + Last name (Western)
- Last name + First name (Eastern)
- Honorifics and titles

### Performance Optimization

**Lazy Loading Catalogs:**

```typescript
// Load only active locale
const catalogs = {
  en: () => import("./locales/en/messages"),
  ru: () => import("./locales/ru/messages"),
  // ... other locales
};

// Dynamic import on locale change
const loadCatalog = async (locale: string) => {
  const catalog = await catalogs[locale]();
  i18n.load(locale, catalog.messages);
  i18n.activate(locale);
};
```

**Bundle Size:**

- Only active locale loaded initially
- Other locales loaded on-demand
- Compiled catalogs are minified
- Tree-shaking removes unused messages

### Testing i18n

**Test Coverage:**

- Snapshot tests for each locale
- Pluralization edge cases
- Number/date formatting
- RTL layout (when implemented)
- Missing translation fallbacks

**Pseudo-localization:**

- Test locale with extended characters
- Verify UI handles longer text
- Check for hardcoded strings
- Validate layout flexibility

### Configuration

**lingui.config.ts:**

```typescript
export default {
  locales: ["en", "ru", "uk", "de", "es", "fr", "it", "pt", "ja", "zh-CN"],
  sourceLocale: "en",
  catalogs: [
    {
      path: "src/locales/{locale}/messages",
      include: ["src"],
      exclude: ["**/node_modules/**"],
    },
  ],
  format: "po",
};
```

### Migration Path

**Phase 1: Core UI (Current)**

- Navigation and menus
- Dashboard labels
- Common actions (Create, Edit, Delete, Save, Cancel)
- Status labels

**Phase 2: Data Grids**

- Column headers
- Filter labels
- Bulk action labels
- Empty states

**Phase 3: Forms & Validation**

- Form labels and placeholders
- Validation messages
- Help text

**Phase 4: Advanced Features**

- Complex workflows
- Contextual help
- Onboarding content
- Documentation links

**Phase 5: Additional Languages**

- Add more languages based on operator demand
- Community translations
- Professional translation services for critical languages

---

## Security Considerations

**Access Control:**

- Admin UI accessible only to users with `PLATFORM_OPERATOR` authority
- Separate authentication flow from main user UI with elevated security
- MFA enforcement for all platform operator accounts
- IP whitelist support for additional security

**Platform Authority Bypass:**

- `PLATFORM_OPERATOR` bypasses all tenant isolation checks
- Gateway propagates `X-Platform-Operator: true` header for platform operators
- Downstream services check platform operator authority before enforcing tenant context validation
- Regular users cannot spoof platform headers (stripped by gateway)

**Tenant Context Validation (for non-platform users):**

- JWT `X-Tenant-ID` claim must match requested tenant in path/query
- Service layer throws `TenantContextMismatchException` (HTTP 403) on mismatch
- Database queries automatically scoped to tenant schema via MyBatis interceptor

**Audit & Compliance:**

- Complete audit trail of all platform operator actions
- Platform operator operations logged with elevated privilege indicator
- Impersonation sessions fully logged with reason and duration
- GDPR-compliant data export and deletion
- Data retention policies enforced

**Rate Limiting:**

- Platform operator API endpoints rate-limited per platform operator
- Stricter limits on destructive actions (delete, ban)
- Platform operators have higher rate limits than tenant owners

**Sensitive Actions:**

- Confirmation dialogs for all destructive actions
- Re-authentication required for high-risk actions (delete, impersonate)
- Reason required for ban, suspend, delete actions
- Two-person rule for critical operations (optional, configurable)

---

## Implementation Phases

### Phase 1: Foundation (Current State)

**Completed:**

- ✅ User admin CRUD API (`/api/v1/iam/operator/users`)
- ✅ Tenant management API (get, status update, retry provisioning)
- ✅ Invitation management API (send, list, revoke, accept)
- ✅ Subscription query API (local cache, tenant-scoped)
- ✅ Billing settings API (get, update)
- ✅ Plan catalog API with PLATFORM_OPERATOR authority
- ✅ Multi-mode support (TENANT/USER scoped subscriptions)

**In Progress:**

- 🚧 Admin UI scaffolding with Mantine UI
- 🚧 Dashboard metrics aggregation
- 🚧 User list and detail views
- 🚧 i18n setup with Lingui (10 languages)

### Phase 2: Core Admin Features (MVP)

- Dashboard with basic metrics (active users, orgs, subscriptions, trials)
- User list with search and filters
- User detail view with profile, memberships, activity
- Organization list with search and filters
- Organization detail view with members, billing, usage
- Basic platform operator actions (ban, suspend, delete) with confirmation dialogs
- Audit log viewer

### Phase 3: Billing & Subscriptions

- Subscription management UI
- Plan catalog management UI
- Billing settings management
- Payment history and invoices
- Subscription lifecycle actions (change plan, cancel, reactivate)

### Phase 4: Advanced Features

- Impersonation with full audit trail
- Advanced metrics and reporting
- Bulk actions (bulk ban, bulk suspend, bulk export)
- System health monitoring dashboard
- Background job management UI
- Real-time notifications for critical events

### Phase 5: Automation & Intelligence

- Automated alerts and notifications
- Anomaly detection (unusual activity, spike in signups)
- Predictive analytics (churn prediction, usage forecasting)
- Automated remediation workflows
- Custom dashboard widgets
- Scheduled reports via email

---

## Future Enhancements

**Planned Features (Makerkit/Enterprise Admin-inspired):**

- **Advanced Analytics Dashboard:**
  - Cohort analysis
  - Funnel visualization
  - Customer segmentation
  - Predictive churn modeling
  - Revenue forecasting
- **Workflow Automation:**
  - Automated actions based on triggers
  - Email sequences for trial users
  - Auto-suspend on payment failure
  - Escalation rules for support
- **Custom Reports:**
  - Report builder with drag-and-drop
  - Scheduled report delivery
  - Custom SQL queries (for advanced users)
  - Report templates library
- **Multi-admin Role Support:**
  - Read-only operator
  - Billing operator (billing access only)
  - Support operator (limited user management)
  - Custom role builder
- **Admin Activity Dashboard:**
  - Track operator actions
  - Operator performance metrics
  - Audit compliance reports
- **Tenant Usage Analytics:**
  - Feature adoption tracking
  - API usage patterns
  - Storage growth analysis
  - User engagement metrics
- **Cost Analysis & Optimization:**
  - Infrastructure cost per tenant
  - Profitability analysis
  - Resource optimization recommendations
  - Cost allocation reports
- **Automated Compliance Reporting:**
  - GDPR compliance dashboard
  - SOC 2 audit trails
  - Data retention reports
  - Privacy request tracking
- **Integration Hub:**
  - Support ticketing systems (Zendesk, Intercom)
  - Communication platforms (Slack, Teams)
  - Analytics platforms (Mixpanel, Amplitude)
  - CRM systems (Salesforce, HubSpot)
- **Mobile App:**
  - Critical admin actions on mobile
  - Push notifications for alerts
  - Quick user lookup
  - Subscription management
- **AI-Powered Features:**
  - Anomaly detection (unusual activity, spike in signups)
  - Churn prediction
  - Support ticket auto-categorization
  - Smart recommendations for operators

---

## Related Documentation

- [Platform Architecture](./architecture.md)
- [Platform Capabilities](./capabilities.md)
- [Tenancy Model](./tenancy.md)
- [Scalability](./scalability.md)

---

## Frontend Architecture Rules

> Strict design and architecture rules for the Platform Admin UI codebase.
> These rules are enforced by `src/architecture.test.ts` on every CI build and must not be bypassed.

---

### Layer Structure — Feature-Sliced Design (FSD)

The Admin UI strictly follows [Feature-Sliced Design](https://feature-sliced.design/). Layers are ordered top → bottom. A layer may only import from layers **below** it. Cross-layer imports must go through the public API (`index.ts`) — never import internal files directly.

```
src/
├── app/          # Bootstrap, providers, router, theme
├── pages/        # Route-level components (TanStack Router file-based)
├── widgets/      # Self-contained page sections with own data fetching
├── features/     # Reused or complex user-facing flows
├── entities/     # Domain models, API hooks, status display components
├── processes/    # Reserved — cross-cutting flows (currently unused)
└── shared/       # Infrastructure primitives with zero business logic
    ├── api/      # Axios instance and interceptors only
    ├── lib/      # Utilities, query client, i18n setup, URL state helpers
    ├── ui/       # Generic UI components (no domain knowledge)
    └── types/    # Shared TypeScript types
```

**Dependency direction is strictly enforced.** `widgets` may import from `features`, `entities`, and `shared`. `features` may import from `entities` and `shared`. `entities` may import from `shared` only. `shared` has no internal layer dependencies.

---

### Layer Responsibilities

#### `app/`

- Wires all providers: `MantineProvider`, `QueryClientProvider`, `I18nProvider`, `ModalsProvider`, `RouterProvider`.
- Defines the Mantine theme and design tokens.
- Initializes the Lingui locale on startup.
- Contains runtime environment config (`config/runtime-env.ts`).
- **Must not** contain any business logic or domain-specific code.

#### `pages/`

- One file per route, following TanStack Router file-based conventions.
- Pages are **thin** — they mount one or two widgets and pass route params. No data fetching, no business logic.
- Maximum complexity allowed in a page: reading route params/search params and passing them to widgets.
- File naming: kebab-case, dynamic segments use `$param` prefix (`$userId.tsx`, `$tenantKey.tsx`).

```
pages/
├── __root.tsx           # AppShell layout, auth guard
├── index.tsx            # Redirects to /dashboard
├── dashboard.tsx
├── users/
│   ├── index.tsx        # Mounts <UserTable />
│   └── $userId.tsx      # Mounts <UserDetail userId={userId} />
├── organizations/
│   ├── index.tsx
│   └── $tenantKey.tsx
├── subscriptions/
│   ├── index.tsx
│   └── $subscriptionId.tsx
├── plans/
│   ├── index.tsx
│   └── $planCode.tsx
├── audit-log.tsx
└── system.tsx
```

#### `widgets/`

- Self-contained page sections. Each widget owns its data fetching, loading states, and error handling.
- Widgets compose entities and features. They may contain internal sub-components (dialogs, row actions, tab panels) that are **not exported** — internal files are not part of the public API.
- Action dialogs (ban, suspend, delete, etc.) live **inside the widget** as internal files until they are needed in more than one widget. Only then are they promoted to `features/`.
- Each widget must have an `index.ts` that exports only what consumers need.

```
widgets/
├── app-shell/           # Sidebar nav, header, breadcrumbs, command palette (Cmd+K)
├── dashboard-overview/  # Metrics cards, charts, date range selector
├── user-table/          # DataTable + filters + bulk actions + inline action dialogs
├── user-detail/         # Tabbed detail view (profile, memberships, auth, billing, activity)
├── org-table/
├── org-detail/
├── subscription-table/
├── plan-table/
├── audit-log-table/
└── system-health/       # Service health, job status, background jobs
```

#### `features/`

A slice is promoted to `features/` only when **both** conditions are met:

1. It is used in more than one widget **or** it has a multi-step flow with its own state.
2. It cannot be simplified to a single dialog + single mutation.

Current qualifying features:

```
features/
├── auth/                # Platform operator login, MFA, session bootstrap
└── impersonate-user/    # Multi-step: reason input → session start → UI indicator → exit
```

Everything else (ban dialog, suspend dialog, delete confirmation, plan create form) starts as a widget-internal file. **Do not pre-emptively create feature slices.**

#### `entities/`

- One slice per backend domain: `user`, `organization`, `subscription`, `plan`, `audit-log`, `operator-session`.
- Each entity slice contains: API hooks (`api.ts`), domain types and enums (`model.ts`), and optionally a status display component (`ui.tsx`).
- Entity UI components are **stateless display only** — they receive props and render. No mutations, no dialogs, no actions.
- The `operator-session` entity holds the current operator's JWT state and impersonation state in a Zustand store.

```
entities/
├── user/
│   ├── api.ts           # useUsers, useUser, useCreateUser, useUpdateUser, useDeleteUser
│   ├── model.ts         # User type, UserStatus enum, status → color map
│   ├── ui.tsx           # UserStatusBadge
│   └── index.ts
├── organization/
│   ├── api.ts
│   ├── model.ts         # Tenant type, TenantStatus enum
│   ├── ui.tsx           # OrgStatusBadge
│   └── index.ts
├── subscription/
│   ├── api.ts
│   ├── model.ts
│   ├── ui.tsx           # SubscriptionStatusBadge
│   └── index.ts
├── plan/
│   ├── api.ts
│   ├── model.ts
│   └── index.ts
├── audit-log/
│   ├── api.ts
│   ├── model.ts
│   └── index.ts
└── operator-session/
    ├── model.ts         # OperatorSession type, ImpersonationSession type
    ├── store.ts         # Zustand store — current operator, impersonation state
    └── index.ts
```

**Entity files are flat** — no sub-folders inside an entity slice. If a single file grows beyond ~200 lines, split by concern (`api.ts`, `model.ts`, `ui.tsx`) but keep them at the same level.

#### `shared/`

- `shared/api/` — Axios instance and interceptors only. No query hooks, no domain types.
- `shared/lib/` — Infrastructure utilities:
  - `query.ts` — `queryClient` singleton and query key factories per domain.
  - `i18n.ts` — Lingui setup and `loadCatalog` function.
  - `date.ts` — `Intl` formatters for date, currency, and relative time.
  - `url-state.ts` — `nuqs` helpers: `usePaginationState`, `useFilterState`.
- `shared/ui/` — Generic components with **zero domain knowledge**: `DataTable` wrapper, `ConfirmDialog`, `PageHeader`, `StatusBadge` (generic), `EmptyState`, `LoadingOverlay`.
- `shared/types/` — `PaginatedResponse<T>`, `ApiError`, `SortOrder`, `GenericDataResponse<T>`.

**The hard rule for `shared/`:** if a component or utility knows what a `TenantStatus`, `UserStatus`, or `PLATFORM_OPERATOR` is — it does not belong in `shared/`. Move it to the appropriate entity.

---

### Public API (Barrel Exports)

Every slice and every `shared/` segment **must** expose an `index.ts`. Consumers import from the `index.ts` only — never from internal files.

```ts
// ✅ Correct
import { UserStatusBadge, useUser } from "@/entities/user";
import { UserTable } from "@/widgets/user-table";

// ❌ Wrong — importing internal files directly
import { UserStatusBadge } from "@/entities/user/ui";
import { useUser } from "@/entities/user/api";
```

This is enforced by `architecture.test.ts`. A missing `index.ts` fails CI.

---

### Component Placement Decision Rules

Use this decision tree when adding a new component:

```
Is it a pure display primitive with no domain knowledge?
  YES → shared/ui/

Does it display domain state (status, badge, avatar) with no actions?
  YES → entities/{domain}/ui.tsx

Is it a single dialog or form tied to one mutation, used in one widget?
  YES → widget-internal file (not exported from index.ts)

Is it a single dialog or form used in MORE THAN ONE widget?
  YES → features/{action-name}/

Is it a multi-step flow with its own state, or cross-cutting (auth, impersonation)?
  YES → features/{flow-name}/

Does it fetch its own data and compose multiple entities/features?
  YES → widgets/{section-name}/
```

---

### Naming Conventions

| Context                           | Convention                    | Example                             |
| --------------------------------- | ----------------------------- | ----------------------------------- |
| Page files                        | kebab-case                    | `user-settings.tsx`, `$userId.tsx`  |
| Widget / feature / entity folders | kebab-case                    | `user-table/`, `impersonate-user/`  |
| `shared/ui` component folders     | kebab-case                    | `confirm-dialog/`, `page-header/`   |
| React components                  | PascalCase                    | `UserStatusBadge`, `OrgDetailTabs`  |
| Hooks                             | `use` prefix, camelCase       | `useUsers`, `useOrgDetail`          |
| Zustand stores                    | `use` prefix + `Store` suffix | `useOperatorSessionStore`           |
| Query key factories               | `{domain}Keys`                | `userKeys`, `orgKeys`               |
| Zod schemas                       | `{purpose}Schema`             | `banUserSchema`, `createPlanSchema` |
| Test files                        | `.test.ts(x)` co-located      | `user-table.test.tsx`               |

**Component naming encodes placement:**

| Name pattern                                | Lives in                   |
| ------------------------------------------- | -------------------------- |
| `*StatusBadge`, `*Avatar`, `*Label`         | `entities/{domain}/ui.tsx` |
| `*Table`, `*Detail`, `*Panel`, `*Overview`  | `widgets/`                 |
| `*Dialog`, `*Form`, `*Drawer` (single use)  | widget-internal            |
| `*Flow`, `*Wizard`, `*Session`              | `features/`                |
| `*Button`, `*Card`, `*EmptyState` (generic) | `shared/ui/`               |

---

### State Management Rules

**TanStack Query** owns all server state. Do not duplicate API data in Zustand.

**Zustand** owns client-only UI state:

- Current operator session and decoded JWT claims (`operator-session` entity store).
- Impersonation session state (active, target user, reason, expiry).
- Grid preferences (column visibility, saved filters) — persisted to `localStorage`.

**URL state via `nuqs`** owns shareable UI state:

- Pagination (`page`, `pageSize`).
- Active filters (status, date range, search term).
- Active tab in detail views.

**Rule:** if state needs to survive a page refresh and be shareable via URL → `nuqs`. If it's ephemeral UI state within a session → Zustand. If it's server data → TanStack Query.

---

### Data Fetching Rules

- All API functions live in `entities/{domain}/api.ts`. Never call `apiClient` directly from a widget or page.
- Query keys are defined in `shared/lib/query.ts` as factory functions:

```ts
export const userKeys = {
  all: ["users"] as const,
  list: (filters: UserFilters) => [...userKeys.all, "list", filters] as const,
  detail: (id: string) => [...userKeys.all, "detail", id] as const,
};
```

- After every mutation, invalidate the relevant query keys — do not manually update the cache unless a performance measurement justifies it.
- Use `queryClient.ensureQueryData()` in TanStack Router loaders for route-level prefetching.
- Optimistic updates are allowed only in widgets where the latency is user-perceptible and the rollback logic is straightforward.

---

### Forms and Validation Rules

- Every form has a Zod schema defined first. The TypeScript type is derived from the schema with `z.infer<typeof schema>`.
- Schemas live in the `model/` segment of the feature, or inline in the widget file if the form is widget-internal.
- Use `@mantine/form` with `mantine-form-zod-resolver` for Mantine-native forms.
- Use `react-hook-form` with `@hookform/resolvers/zod` for complex multi-step forms in `features/`.
- Validation error messages must be explicit strings — never rely on Zod's default messages in user-facing UI.

---

### i18n Rules

- Every user-visible string must be wrapped in `<Trans>` or `_(msg\`...\`)` from LinguiJS.
- Never hardcode UI strings — this includes button labels, table column headers, status labels, error messages, and empty state text.
- Dynamic content that is not translated: user-generated names, email addresses, IDs, technical keys.
- Dates and numbers use `Intl` formatters from `shared/lib/date.ts` — never format them manually.
- Locale catalogs are lazy-loaded per locale. Only the active locale is in the initial bundle.

---

### Accessibility Rules

- All interactive elements must be keyboard-accessible.
- All data tables must support keyboard navigation (arrow keys, Enter to open detail).
- All action dialogs must trap focus and return focus to the trigger element on close.
- All status badges and icons must have accessible labels (`aria-label` or visible text).
- Color alone must never be the only indicator of status — always pair color with a text label or icon.
- Confirmation dialogs for destructive actions must have a clearly labeled cancel option as the default focus target.

---

### Security Rules (Frontend)

- The `PLATFORM_OPERATOR` authority check is performed on every protected route via the auth guard in `__root.tsx`. Non-operator users are redirected to the login page.
- JWT claims are read from the decoded token in `operator-session` store — never from `localStorage` directly.
- Impersonation state is visually indicated in the `app-shell` widget at all times when active. No action that modifies the impersonated user's password, billing, or account deletion is permitted during impersonation — these actions are disabled in the UI and rejected by the API.
- All destructive actions (delete, ban, suspend) require a confirmation dialog with explicit user input (typing the resource name or selecting a reason).
- Re-authentication is required before high-risk actions (impersonation start, account deletion). This is enforced by the API and reflected in the UI with a password confirmation step.

---

### Architecture Test Coverage

`src/architecture.test.ts` enforces the following at CI time:

| Rule                                      | How enforced                                           |
| ----------------------------------------- | ------------------------------------------------------ |
| All FSD layers exist                      | `statSync` check on each layer directory               |
| All slices have `index.ts`                | Directory scan of `features/`, `widgets/`, `entities/` |
| `shared/ui` has `index.ts`                | Direct check                                           |
| Features have `ui/` and `model/` segments | Directory scan                                         |
| Page files use kebab-case                 | Regex on filenames                                     |
| `shared/ui` folders use kebab-case        | Regex on folder names                                  |

Run with: `pnpm test:arch`

Any new slice added to `features/`, `widgets/`, or `entities/` must have an `index.ts` before the PR is merged. The architecture test will fail CI otherwise.

---

### Loading States & Transition Rules

The Admin UI uses three loading mechanisms with strictly defined responsibilities. Using the wrong one for a given context is a UX bug.

#### The Three Mechanisms

| Mechanism                          | When it shows                                                                 | What it covers                   |
| ---------------------------------- | ----------------------------------------------------------------------------- | -------------------------------- |
| **Full-page spinner**              | App bootstrap only — initial locale load before any route renders             | Entire viewport                  |
| **Top progress bar** (`nprogress`) | Route navigation — from link click until the new route is fully rendered      | Navigation feedback only         |
| **Skeleton**                       | First data load for a specific section — when there is no cached data to show | The content area of that section |

These three must never be mixed for the same event. A route navigation shows the progress bar — it never additionally shows a skeleton or spinner. A widget's first load shows a skeleton — it never shows a spinner inside the content area.

#### Full-Page Spinner

- Used **only once** in the application lifecycle: during app bootstrap while the initial locale catalog is loading.
- Implemented in `app.tsx` via the `isInitialLoading` state gate.
- Must not be reused for any other purpose. No widget, feature, or page may render a full-viewport spinner.

#### Top Progress Bar (nprogress)

- Fires on every route transition via `router.subscribe("onBeforeLoad")` and `router.subscribe("onLoad")`.
- Covers the perceived latency of navigation — the user always gets immediate feedback that something is happening.
- Route loaders use `pendingMs` to suppress the `pendingComponent` for fast cached navigations. The progress bar still runs — it is the only visible indicator for sub-300ms transitions.
- Do not manually call `nprogress.start()` or `nprogress.complete()` anywhere outside the router subscription. Progress bar control belongs to the router, not to individual components.

#### Skeletons

Skeletons are the primary loading UI for data-driven content. The following rules are strict:

**When to use a skeleton:**

- A widget or section is mounting for the first time and has no cached data (`isLoading === true`, which means `isFetching && !data`).
- A route's `pendingComponent` — shown only when the loader takes longer than `pendingMs`.

**When NOT to use a skeleton:**

- Refetching with existing data already on screen (`isFetching && !!data`) — use opacity dimming instead (see below).
- Inside a dialog or drawer that is opening — the dialog itself is the loading indicator; show a skeleton only if the dialog content requires a separate async fetch.
- For mutations — never show a skeleton while a mutation is in flight.

**Skeleton anatomy rules:**

1. **Match real layout dimensions exactly.** A skeleton must have the same height, spacing, and column structure as the real content it replaces. If the real table row is 52px tall with 8 columns, the skeleton row is 52px tall with 8 columns. Layout shift when data arrives is a bug.

2. **Skeleton rows count matches the default page size.** If the table defaults to 20 rows per page, the skeleton renders 20 rows. Do not use an arbitrary number like 5 or 3.

3. **Never skeletonize chrome.** Page headers, breadcrumbs, tab bars, sidebar navigation, and action buttons above the table render immediately — they are never replaced with skeletons. Only the data content area is skeletonized.

4. **Tab bars render immediately; tab panel content is skeletonized.** In detail views (user detail, org detail), the tab list is always visible. The skeleton lives inside the active tab panel only.

5. **One skeleton component per widget.** Each widget that has a loading state owns a co-located `*-skeleton.tsx` file. Skeletons are not shared across widgets — they are too layout-specific.

6. **Skeletons are not animated beyond Mantine's default pulse.** Do not add custom shimmer animations or staggered reveals. The default Mantine `Skeleton` pulse is sufficient and consistent.

**Skeleton file convention:**

```
widgets/
└── user-table/
    ├── user-table.tsx
    ├── user-table-skeleton.tsx   ← co-located, not exported from index.ts
    └── index.ts                  ← exports UserTable only
```

The skeleton is an internal implementation detail of the widget. It is never imported from outside.

**Skeleton example — table widget:**

```tsx
// widgets/user-table/user-table-skeleton.tsx
export function UserTableSkeleton() {
  return (
    <Stack gap={1}>
      <Skeleton height={40} radius={0} />
      {Array.from({ length: 20 }).map((_, i) => (
        <Skeleton key={i} height={52} radius={0} />
      ))}
    </Stack>
  );
}

// widgets/user-table/user-table.tsx
export function UserTable() {
  const { data, isLoading, isFetching } = useQuery({ ... });

  if (isLoading) return <UserTableSkeleton />;

  return (
    <Box style={{ opacity: isFetching ? 0.6 : 1, transition: "opacity 150ms ease" }}>
      <DataTable rows={data.items} ... />
    </Box>
  );
}
```

**Skeleton example — detail view with tabs:**

```tsx
// widgets/user-detail/user-detail.tsx
export function UserDetail({ userId }: { userId: string }) {
  const { data, isLoading } = useQuery(userKeys.detail(userId), ...);

  return (
    <Tabs defaultValue="profile">
      <Tabs.List>
        {/* Tab bar always renders — never skeletonized */}
        <Tabs.Tab value="profile"><Trans>Profile</Trans></Tabs.Tab>
        <Tabs.Tab value="memberships"><Trans>Memberships</Trans></Tabs.Tab>
        <Tabs.Tab value="activity"><Trans>Activity</Trans></Tabs.Tab>
      </Tabs.List>

      <Tabs.Panel value="profile">
        {isLoading ? <ProfileTabSkeleton /> : <ProfileTab user={data} />}
      </Tabs.Panel>
      {/* other panels */}
    </Tabs>
  );
}
```

#### Refetch State — Opacity Dimming

When data is already on screen and a refetch is triggered (filter change, pagination, sort change, window focus refetch), the UI must not replace the existing content with a skeleton. Instead, dim the content area:

```tsx
<Box style={{ opacity: isFetching ? 0.6 : 1, transition: "opacity 150ms ease" }}>
  {/* existing content stays visible */}
</Box>
```

The `150ms ease` transition prevents the opacity change itself from being jarring. The user sees the table slightly fade, then sharpen when new data arrives — no layout shift, no skeleton flash.

Use `placeholderData: keepPreviousData` from TanStack Query on all list queries to ensure `data` is never `undefined` during a refetch:

```ts
import { keepPreviousData } from "@tanstack/react-query";

const { data, isFetching } = useQuery({
  queryKey: userKeys.list(filters),
  queryFn: () => fetchUsers(filters),
  placeholderData: keepPreviousData,
});
```

Without `keepPreviousData`, changing a filter causes `data` to become `undefined` momentarily, which triggers `isLoading: true` and shows the skeleton. With it, the previous page's data stays in place until the new data arrives.

#### Route Loader Pattern

Every route that fetches data must use a loader with `ensureQueryData` and `pendingMs`:

```ts
export const Route = createFileRoute("/users/$userId")({
  loader: ({ context: { queryClient }, params }) =>
    queryClient.ensureQueryData({
      queryKey: userKeys.detail(params.userId),
      queryFn: () => fetchUser(params.userId),
    }),
  pendingComponent: UserDetailSkeleton,
  pendingMs: 300,
  component: UserDetailPage,
});
```

- `ensureQueryData` returns immediately on cache hit — no loader delay, no pending state shown.
- `pendingMs: 300` suppresses `pendingComponent` for the first 300ms — cached navigations show nothing but the progress bar.
- `pendingComponent` is the widget's skeleton — the same component used for `isLoading` inside the widget.

#### Summary Decision Table

| Situation                               | Correct UI response                                  |
| --------------------------------------- | ---------------------------------------------------- |
| App first paint, locale loading         | Full-page spinner                                    |
| Navigating to a new route               | Top progress bar only                                |
| Route loader taking > 300ms (cold load) | Top progress bar + route `pendingComponent` skeleton |
| Widget first mount, no cache            | Skeleton (content area only, chrome stays)           |
| Filter / sort / page change             | Opacity dim on existing content                      |
| Mutation in flight                      | Button loading state, no content change              |
| Mutation complete, query invalidated    | Opacity dim during background refetch                |
| Background refetch (window focus)       | No visible indicator (silent)                        |

---

### Guards & Authorization Rules

Guards protect routes, data, and actions. They are implemented at three distinct levels with strictly defined responsibilities. Mixing levels or using the wrong mechanism for a given concern is an architecture bug.

---

#### Core Rule: Guards Live in `beforeLoad`, Never in Components

Route protection is never implemented as a wrapper component (`<AuthGuard>`, `<RequireAuth>`, etc.). Wrapper components render first, then redirect — this causes a flash of protected content and triggers unnecessary data fetching before the redirect fires.

TanStack Router's `beforeLoad` runs synchronously before the component tree mounts and before the route loader executes. A `throw redirect(...)` inside `beforeLoad` aborts the navigation entirely — nothing renders, nothing fetches.

```ts
// ✅ Correct — beforeLoad guard
export const Route = createFileRoute("/users")({
  beforeLoad: ({ context }) => {
    if (!context.session.isAuthenticated) {
      throw redirect({ to: "/login", search: { redirect: location.href } });
    }
  },
});

// ❌ Wrong — component wrapper guard
function UsersPage() {
  const session = useSession();
  if (!session.isAuthenticated) return <Navigate to="/login" />;  // renders first, then redirects
  return <UserTable />;
}
```

---

#### Three Guard Levels

**Level 1 — Root route (`__root.tsx`): authentication + platform operator authority**

Covers the entire application tree. Every route under `__root.tsx` is protected. The login route lives outside this tree as a sibling, not a child.

```ts
// pages/__root.tsx
export const Route = createRootRouteWithContext<{
  queryClient: QueryClient;
  session: OperatorSession;
}>()({
  beforeLoad: ({ context, location }) => {
    if (!context.session.isAuthenticated) {
      throw redirect({
        to: "/login",
        search: { redirect: location.href },
      });
    }
    if (!context.session.hasPlatformOperatorAuthority) {
      throw redirect({ to: "/unauthorized" });
    }
  },
  component: RootLayout,
});
```

This is the only place that checks `isAuthenticated` and `hasPlatformOperatorAuthority`. Do not repeat these checks in child routes.

**Level 2 — Route group layouts: sub-authority checks**

Route groups (TanStack Router layout routes) protect logical sections that require a specific operator authority beyond the base `PLATFORM_OPERATOR`. Only add a level-2 guard when a section genuinely has a narrower permission requirement.

```ts
// pages/system/_layout.tsx
export const Route = createFileRoute("/system/_layout")({
  beforeLoad: ({ context }) => {
    if (!context.session.authorities.includes("MANAGE_SYSTEM")) {
      throw redirect({ to: "/dashboard" });
    }
  },
});
```

Current sections that warrant a level-2 guard:

| Route group          | Required authority    |
| -------------------- | --------------------- |
| `/system`            | `MANAGE_SYSTEM`       |
| `/plans` (mutations) | `MANAGE_PLAN_CATALOG` |
| `/audit-log`         | `VIEW_AUDIT_LOG`      |

Read-only sections (`VIEW_*` authorities) that all platform operators have do not need a level-2 guard — the root guard is sufficient.

**Level 3 — Route loaders: data-level 403 handling**

When a loader fetches a specific resource and the API returns 403, the loader catches it and redirects rather than rendering an error page. This handles edge cases where the operator's session authority changed mid-session or a resource was deleted between navigation.

```ts
// pages/users/$userId.tsx
export const Route = createFileRoute("/users/$userId")({
  loader: async ({ context: { queryClient }, params }) => {
    try {
      return await queryClient.ensureQueryData({
        queryKey: userKeys.detail(params.userId),
        queryFn: () => fetchUser(params.userId),
      });
    } catch (error) {
      if (isApiError(error) && error.status === 403) {
        throw redirect({ to: "/users" });
      }
      if (isApiError(error) && error.status === 404) {
        throw notFound();
      }
      throw error;
    }
  },
  pendingComponent: UserDetailSkeleton,
  pendingMs: 300,
  component: UserDetailPage,
});
```

Level-3 guards handle API-driven authorization failures. They do not re-check JWT claims — that is level 1's responsibility.

---

#### Router Context Carries Session State

Guards read session state from router context, not directly from the Zustand store. This keeps guards testable and makes the dependency explicit.

```ts
// app.tsx — wire session into router context
const router = createRouter({
  routeTree,
  context: {
    queryClient,
    session: useOperatorSessionStore.getState(),
  },
  defaultPreload: "intent",
});

// Keep router context in sync when session changes (login, logout, token refresh)
useOperatorSessionStore.subscribe((session) => {
  router.invalidate();
});
```

When the session store changes (token refreshed, operator logs out), `router.invalidate()` re-runs all active `beforeLoad` functions with the updated context. This means logout automatically triggers the root guard and redirects to `/login` without any additional logic.

---

#### Redirect-Back Pattern

Always preserve the operator's intended destination when redirecting to login. After successful authentication, redirect back to the original URL.

```ts
// pages/__root.tsx — capture intended destination
beforeLoad: ({ context, location }) => {
  if (!context.session.isAuthenticated) {
    throw redirect({
      to: "/login",
      search: { redirect: location.href },
    });
  }
},
```

```ts
// pages/login.tsx — typed search params
export const Route = createFileRoute("/login")({
  validateSearch: z.object({
    redirect: z.string().catch("/dashboard"),
  }),
  beforeLoad: ({ context }) => {
    // Already authenticated — skip login page
    if (context.session.isAuthenticated) {
      throw redirect({ to: "/dashboard" });
    }
  },
  component: LoginPage,
});
```

```ts
// features/auth/login-form.tsx — redirect after success
const { redirect: redirectTo } = Route.useSearch();
const navigate = useNavigate();

const onLoginSuccess = () => {
  navigate({ to: redirectTo, replace: true });
};
```

The `replace: true` prevents the login page from appearing in the browser history — the back button goes to wherever the operator was before the session expired, not back to the login page.

---

#### Impersonation Constraints — Action Level, Not Route Level

During an impersonation session, the operator can navigate to any route. Impersonation does not restrict navigation — it restricts specific mutations. This is an action-level concern, not a route guard.

Blocked actions are defined as a static set in the `operator-session` entity store. Widgets consume a `canPerformAction` selector to conditionally disable or hide controls.

```ts
// entities/operator-session/store.ts
const BLOCKED_DURING_IMPERSONATION = new Set([
  "change-password",
  "delete-account",
  "update-billing",
  "revoke-all-sessions",
  "impersonate-user", // cannot impersonate while already impersonating
]);

export const useOperatorSessionStore = create<OperatorSessionStore>()(
  immer((set, get) => ({
    isImpersonating: false,
    impersonationTarget: null,

    canPerformAction: (action: string): boolean => {
      if (!get().isImpersonating) return true;
      return !BLOCKED_DURING_IMPERSONATION.has(action);
    },
  })),
);
```

```tsx
// widget-internal usage
function UserActionsMenu({ user }: { user: User }) {
  const canPerformAction = useOperatorSessionStore((s) => s.canPerformAction);

  return (
    <Menu>
      <Menu.Item
        disabled={!canPerformAction("delete-account")}
        color="red"
        leftSection={<IconTrash size={14} />}
      >
        <Trans>Delete Account</Trans>
      </Menu.Item>
      <Menu.Item disabled={!canPerformAction("change-password")}>
        <Trans>Reset Password</Trans>
      </Menu.Item>
    </Menu>
  );
}
```

Rules for impersonation constraints:

- Disabled actions render as disabled (`disabled` prop), not hidden — the operator must see that the action exists but is unavailable during impersonation.
- The `app-shell` widget renders a persistent impersonation banner when `isImpersonating` is true. This banner is never dismissible.
- The API enforces these constraints independently. The frontend constraint is UX feedback, not a security boundary.

---

#### Token Expiry — The Interceptor Guard

JWT expiry mid-session is handled in the Axios response interceptor in `shared/api/`. This is a guard on every API call, not just on navigation. It is the only place that handles 401 responses.

```ts
// shared/api/client.ts
let isRefreshing = false;
let refreshQueue: Array<(token: string) => void> = [];

apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as AxiosRequestConfig & { _retry?: boolean };

    if (error.response?.status === 401 && !originalRequest._retry) {
      if (isRefreshing) {
        // Queue concurrent requests while refresh is in flight
        return new Promise((resolve) => {
          refreshQueue.push((token) => {
            originalRequest.headers!["Authorization"] = `Bearer ${token}`;
            resolve(apiClient(originalRequest));
          });
        });
      }

      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const newToken = await useOperatorSessionStore.getState().refreshToken();
        refreshQueue.forEach((cb) => cb(newToken));
        refreshQueue = [];
        originalRequest.headers!["Authorization"] = `Bearer ${newToken}`;
        return apiClient(originalRequest);
      } catch {
        // Refresh failed — session is dead
        refreshQueue = [];
        useOperatorSessionStore.getState().clearSession();
        // router.invalidate() triggers root beforeLoad → redirects to /login
        router.invalidate();
        return Promise.reject(error);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  },
);
```

Key points:

- The refresh queue prevents multiple concurrent 401s from triggering multiple refresh attempts — only one refresh runs, all other requests wait for it.
- On refresh success, all queued requests retry with the new token transparently. The operator never sees an error.
- On refresh failure, `clearSession()` + `router.invalidate()` is the only logout path. Do not call `navigate("/login")` directly from the interceptor — let the root `beforeLoad` guard handle the redirect so the current URL is preserved as the redirect target.
- The `_retry` flag prevents infinite retry loops if the refreshed token is also rejected.

---

#### Guard Summary

| Concern                                 | Mechanism                                                  | Location                                             |
| --------------------------------------- | ---------------------------------------------------------- | ---------------------------------------------------- |
| Not authenticated                       | `beforeLoad` + `throw redirect`                            | `pages/__root.tsx`                                   |
| Not `PLATFORM_OPERATOR`                 | `beforeLoad` + `throw redirect`                            | `pages/__root.tsx`                                   |
| Sub-authority (e.g. `MANAGE_SYSTEM`)    | `beforeLoad` + `throw redirect`                            | Route group layout                                   |
| API 403 on resource load                | Loader `try/catch` + `throw redirect`                      | Individual route loader                              |
| API 404 on resource load                | Loader `try/catch` + `throw notFound()`                    | Individual route loader                              |
| Token expired mid-session               | Axios interceptor → silent refresh → `router.invalidate()` | `shared/api/client.ts`                               |
| Impersonation action blocks             | `canPerformAction` selector → `disabled` prop              | Widget-internal, reads from `operator-session` store |
| Already authenticated visiting `/login` | `beforeLoad` + `throw redirect` to `/dashboard`            | `pages/login.tsx`                                    |

---

### E2E Testing — Data Attributes

The Admin UI uses a semantic `data-test-*` attribute vocabulary for E2E test targeting. These attributes are the only stable, i18n-safe, refactor-resistant way to locate elements in Playwright tests. CSS selectors, text content, and element roles are not used as primary locators — they break on style changes, translations, and component restructuring.

All `data-test-*` attributes are stripped from the production bundle at build time and have zero runtime cost in production.

---

#### Attribute Vocabulary

Six attributes cover all targeting needs. Each has a single, well-defined responsibility.

| Attribute           | Purpose                                                    | Value format                                        |
| ------------------- | ---------------------------------------------------------- | --------------------------------------------------- |
| `data-test-section` | Marks a widget or page region boundary                     | kebab-case noun: `user-table`, `app-sidebar`        |
| `data-test-action`  | Marks an interactive element by what it does               | kebab-case verb-noun: `ban-user`, `confirm-ban`     |
| `data-test-entity`  | Marks a row or detail view as representing a domain entity | domain name: `user`, `organization`, `subscription` |
| `data-test-id`      | The entity's real ID from the API                          | raw ID value: `usr_abc123`, `a3f7k2m9`              |
| `data-test-status`  | Current status value on an entity row or badge             | enum value: `ACTIVE`, `BANNED`, `SUSPENDED`         |
| `data-test-state`   | Current UI state of a section                              | `loading`, `fetching`, `ready`, `empty`, `error`    |
| `data-test-count`   | Numeric value on summary/pagination elements               | integer as string: `"42"`                           |

`data-testid` (the React Testing Library default) is reserved for elements that do not fit any of the above categories. It is the last resort, not the default.

---

#### `data-test-section` — Region Boundaries

Every major widget and layout region gets a `data-test-section`. This scopes all child locators and prevents false matches across sections.

```tsx
// widgets/app-shell/app-shell.tsx
<nav data-test-section="app-sidebar">...</nav>
<header data-test-section="app-header">...</header>

// widgets/user-table/user-table.tsx
<Box data-test-section="user-table" data-test-state={tableState}>
  ...
</Box>

// widgets/user-detail/user-detail.tsx
<div data-test-section="user-detail">...</div>

// widgets/dashboard-overview/dashboard-overview.tsx
<div data-test-section="dashboard-metrics">...</div>

// features/impersonate-user/impersonation-banner.tsx
<div data-test-section="impersonation-banner">...</div>
```

Rules:

- One `data-test-section` per widget root element — not on every internal div.
- Sections are never nested. A widget does not put `data-test-section` on its internal sub-components.
- The section attribute is always on the outermost element of the widget.

---

#### `data-test-action` — Interactive Elements

Every button, input, select, and link that a test needs to interact with gets `data-test-action`. The value describes what the action does, not what the element is.

```tsx
// Table toolbar
<Button data-test-action="create-user">...</Button>
<Button data-test-action="export-csv">...</Button>
<Button data-test-action="bulk-suspend">...</Button>
<TextInput data-test-action="search-users" />
<Select data-test-action="filter-status" />
<Select data-test-action="filter-subscription-status" />

// Row actions (scoped inside data-test-entity row)
<Menu.Item data-test-action="ban-user">...</Menu.Item>
<Menu.Item data-test-action="unban-user">...</Menu.Item>
<Menu.Item data-test-action="delete-user">...</Menu.Item>
<Menu.Item data-test-action="impersonate-user">...</Menu.Item>

// Dialog actions
<Button data-test-action="confirm-ban">...</Button>
<Button data-test-action="confirm-delete">...</Button>
<Button data-test-action="confirm-suspend">...</Button>
<Button data-test-action="cancel-dialog">...</Button>

// Impersonation
<Button data-test-action="exit-impersonation">...</Button>
<Button data-test-action="start-impersonation">...</Button>

// Pagination
<Button data-test-action="next-page">...</Button>
<Button data-test-action="prev-page">...</Button>
```

Rules:

- Action names are verb-noun pairs in kebab-case: `ban-user`, `create-plan`, `export-csv`.
- Confirmation dialog actions are prefixed with `confirm-`: `confirm-ban`, `confirm-delete`.
- Cancel/close actions on dialogs always use `cancel-dialog` — consistent across all dialogs.
- Do not use `data-test-action` on disabled elements — Playwright can locate them by the action name and assert `disabled` state separately.

---

#### `data-test-entity` + `data-test-id` + `data-test-status` — Entity Rows

Every table row and detail view root that represents a domain entity gets all three attributes. This is what makes row-level targeting reliable and independent of row position.

```tsx
// widgets/user-table/user-table.tsx — table row
<Table.Tr
  key={user.id}
  data-test-entity="user"
  data-test-id={user.id}
  data-test-status={user.status}
>
  <Table.Td>...</Table.Td>
</Table.Tr>

// widgets/org-table/org-table.tsx
<Table.Tr
  data-test-entity="organization"
  data-test-id={org.tenantKey}
  data-test-status={org.status}
>

// widgets/subscription-table/subscription-table.tsx
<Table.Tr
  data-test-entity="subscription"
  data-test-id={subscription.id}
  data-test-status={subscription.status}
>

// widgets/user-detail/user-detail.tsx — detail view root
<div
  data-test-section="user-detail"
  data-test-entity="user"
  data-test-id={user.id}
  data-test-status={user.status}
>
```

`data-test-status` on the row root is the source of truth for status assertions. The visible badge text is translated and must never be used for assertions.

```tsx
// entities/user/ui.tsx — status badge also carries the attribute
<Badge color={userStatusColor(user.status)} data-test-status={user.status}>
  <Trans>{userStatusLabel(user.status)}</Trans>
</Badge>
```

```ts
// Playwright — language-independent status assertion
await expect(
  page.locator(`[data-test-entity="user"][data-test-id="${userId}"] [data-test-status]`),
).toHaveAttribute("data-test-status", "BANNED");
```

---

#### `data-test-state` — Section UI State

The `data-test-state` attribute on a section root reflects the current loading/data state of that section. Tests use it to wait for sections to be ready before interacting.

```tsx
// widgets/user-table/user-table.tsx
const tableState = isLoading ? "loading" : isFetching ? "fetching" : data?.items.length === 0 ? "empty" : "ready";

<Box
  data-test-section="user-table"
  data-test-state={tableState}
>
  {isLoading ? <UserTableSkeleton /> : <DataTable ... />}
</Box>
```

Valid state values:

| Value      | Meaning                                                |
| ---------- | ------------------------------------------------------ |
| `loading`  | First load, no cached data, skeleton is shown          |
| `fetching` | Refetching with existing data visible (opacity dimmed) |
| `ready`    | Data loaded and displayed                              |
| `empty`    | Query succeeded but returned zero results              |
| `error`    | Query failed, error state shown                        |

```ts
// Playwright — always wait for ready before interacting
await page.locator('[data-test-section="user-table"][data-test-state="ready"]').waitFor();
```

This replaces fragile `waitForTimeout` calls and makes tests resilient to variable network latency.

---

#### `data-test-count` — Numeric Assertions

Summary and pagination elements that display counts get `data-test-count` with the numeric value as a string attribute. This avoids parsing translated text like "42 users" or "Showing 1–20 of 42".

```tsx
// Pagination summary
<Text data-test-count="total-results">{totalItems}</Text>
<Text data-test-count="selected-rows">{selectedCount}</Text>
<Text data-test-count="current-page">{currentPage}</Text>

// Dashboard metric cards
<Text data-test-count="active-users">{metrics.activeUsers}</Text>
<Text data-test-count="active-organizations">{metrics.activeOrgs}</Text>
<Text data-test-count="active-subscriptions">{metrics.activeSubscriptions}</Text>
```

```ts
// Playwright — assert count without parsing text
await expect(page.locator('[data-test-count="total-results"]')).toHaveAttribute(
  "data-test-count",
  "42",
);
// or
const count = await page
  .locator('[data-test-count="total-results"]')
  .getAttribute("data-test-count");
expect(Number(count)).toBeGreaterThan(0);
```

---

#### Production Build — Strip All Test Attributes

All `data-test-*` attributes are removed from the production bundle. They must not appear in production HTML.

Install the SWC plugin:

```bash
pnpm add -D babel-plugin-react-remove-properties
```

Configure in `vite.config.ts`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react-swc";

export default defineConfig(({ mode }) => ({
  plugins: [
    react({
      plugins:
        mode === "production"
          ? [
              [
                "babel-plugin-react-remove-properties",
                {
                  properties: [
                    "data-test-section",
                    "data-test-action",
                    "data-test-entity",
                    "data-test-id",
                    "data-test-status",
                    "data-test-state",
                    "data-test-count",
                    "data-testid",
                  ],
                },
              ],
            ]
          : [],
    }),
  ],
}));
```

Verify the strip works before shipping: `pnpm build && grep -r "data-test" dist/` must return no results.

---

#### Playwright Page Object Pattern

Page objects wrap the attribute vocabulary into readable, maintainable test helpers. One page object per major widget or page.

```ts
// e2e/pages/users.page.ts
import { type Page, expect } from "@playwright/test";

export class UsersPage {
  constructor(private readonly page: Page) {}

  // Section locators
  get section() {
    return this.page.locator('[data-test-section="user-table"]');
  }

  // Wait helpers
  async waitForReady() {
    await this.section
      .and(this.page.locator('[data-test-state="ready"]'))
      .waitFor({ timeout: 10_000 });
  }

  async waitForEmpty() {
    await this.section.and(this.page.locator('[data-test-state="empty"]')).waitFor();
  }

  // Toolbar interactions
  async search(term: string) {
    await this.page.locator('[data-test-action="search-users"]').fill(term);
    await this.waitForReady();
  }

  async filterByStatus(status: string) {
    await this.page.locator('[data-test-action="filter-status"]').selectOption(status);
    await this.waitForReady();
  }

  // Row targeting
  row(userId: string) {
    return this.page.locator(`[data-test-entity="user"][data-test-id="${userId}"]`);
  }

  rowStatus(userId: string) {
    return this.row(userId).locator("[data-test-status]");
  }

  rowAction(userId: string, action: string) {
    return this.row(userId).locator(`[data-test-action="${action}"]`);
  }

  // Composite actions
  async banUser(userId: string) {
    await this.rowAction(userId, "ban-user").click();
    await this.page.locator('[data-test-action="confirm-ban"]').click();
    await expect(this.rowStatus(userId)).toHaveAttribute("data-test-status", "BANNED");
  }

  async deleteUser(userId: string) {
    await this.rowAction(userId, "delete-user").click();
    await this.page.locator('[data-test-action="confirm-delete"]').click();
    await expect(this.row(userId)).not.toBeVisible();
  }

  // Count assertions
  async expectTotalResults(count: number) {
    await expect(this.page.locator('[data-test-count="total-results"]')).toHaveAttribute(
      "data-test-count",
      String(count),
    );
  }
}
```

Test reads like a spec:

```ts
// e2e/users.spec.ts
test("operator bans a user and status updates immediately", async ({ page }) => {
  const usersPage = new UsersPage(page);
  await page.goto("/users");
  await usersPage.waitForReady();
  await usersPage.banUser("usr_abc123");
});

test("search filters the user list", async ({ page }) => {
  const usersPage = new UsersPage(page);
  await page.goto("/users");
  await usersPage.waitForReady();
  await usersPage.search("alice@example.com");
  await usersPage.expectTotalResults(1);
});
```

---

#### Rules Summary

- Use `data-test-section` on every widget root — one per widget, never nested.
- Use `data-test-action` on every interactive element a test needs to click or fill — named by what it does, not what it is.
- Use `data-test-entity` + `data-test-id` + `data-test-status` on every entity row and detail view root.
- Use `data-test-state` on every data-driven section to enable reliable `waitFor` in tests.
- Use `data-test-count` on numeric summary elements to avoid parsing translated text.
- Never use translated text content as a Playwright locator — it breaks on locale change.
- Never use CSS class names as Playwright locators — they break on style refactors.
- Never use element position (`nth-child`, `first`, `last`) as a primary locator — use `data-test-id` instead.
- All `data-test-*` attributes are stripped in production builds — verify with `grep` after every build pipeline change.
- Page objects are the only place Playwright locator strings are written — tests call page object methods, never raw locators.
