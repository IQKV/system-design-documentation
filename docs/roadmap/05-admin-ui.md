# Platform Admin UI

## Overview

The Platform Admin UI is a dedicated administrative interface for platform operators to manage the entire multi-tenant SaaS platform. It provides comprehensive oversight and control over users, organizations, subscriptions, and system health across all tenants.

**Target Users:** Platform operators, support engineers, operations team

**Tech Stack:** React 19 + Mantine UI 8 + TanStack Router + TanStack Query + TypeScript + Lingui i18n

**Architecture:** Separate SPA deployed independently from the main user-facing UI, communicates through API Gateway with elevated privileges

**Internationalization:** Full i18n support with Lingui for multi-language admin interface (English as base lamguage)

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
- `GET /api/v1/iam/admin/users` - List users with pagination
- `GET /api/v1/iam/admin/users/{id}` - Get user details
- `POST /api/v1/iam/admin/users` - Create user with temp password
- `PUT /api/v1/iam/admin/users/{id}` - Full user update
- `PATCH /api/v1/iam/admin/users/{id}` - Partial user update
- `DELETE /api/v1/iam/admin/users/{id}` - Delete user (cascade)

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
- `POST /api/v1/iam/admin/users/{id}/ban` - Ban user (PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/users/{id}/unban` - Unban user (PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/users/{id}/impersonate` - Impersonate user (PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/users/{id}/unlock` - Unlock account (PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/users/{id}/verify-email` - Verify email (PLATFORM_OPERATOR)
- `GET /api/v1/iam/admin/users/{id}/memberships` - Get memberships (PLATFORM_OPERATOR)
- `GET /api/v1/iam/admin/users/{id}/activity` - Get activity log (PLATFORM_OPERATOR)

**Tenant Admin Actions:**
- `GET /api/v1/iam/admin/tenants` - List all tenants (paginated, PLATFORM_OPERATOR)
- `GET /api/v1/iam/admin/tenants/{tenantKey}` - Get tenant (platform operator view, PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/tenants/{tenantKey}/suspend` - Suspend tenant (PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/tenants/{tenantKey}/unsuspend` - Unsuspend tenant (PLATFORM_OPERATOR)
- `DELETE /api/v1/iam/admin/tenants/{tenantKey}` - Delete tenant (PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/tenants/{tenantKey}/transfer-ownership` - Transfer ownership (PLATFORM_OPERATOR)
- `GET /api/v1/iam/admin/tenants/{tenantKey}/export` - Export data (GDPR, PLATFORM_OPERATOR)

**Subscription Admin Actions:**
- `GET /api/v1/billing/admin/subscriptions` - List all subscriptions (PLATFORM_OPERATOR)
- `POST /api/v1/billing/admin/subscriptions/{id}/change-plan` - Change plan (PLATFORM_OPERATOR)
- `POST /api/v1/billing/admin/subscriptions/{id}/cancel` - Cancel subscription (PLATFORM_OPERATOR)
- `POST /api/v1/billing/admin/subscriptions/{id}/reactivate` - Reactivate (PLATFORM_OPERATOR)
- `POST /api/v1/billing/admin/subscriptions/{id}/apply-discount` - Apply discount (PLATFORM_OPERATOR)
- `POST /api/v1/billing/admin/subscriptions/{id}/extend-trial` - Extend trial (PLATFORM_OPERATOR)

**System Administration:**
- `GET /api/v1/iam/admin/dashboard/metrics` - Platform metrics (PLATFORM_OPERATOR)
- `GET /api/v1/iam/admin/audit-log` - Audit trail (PLATFORM_OPERATOR)
- `GET /api/v1/iam/admin/system/health` - Service health (PLATFORM_OPERATOR)
- `GET /api/v1/iam/admin/system/jobs` - Background jobs (PLATFORM_OPERATOR)
- `POST /api/v1/iam/admin/system/jobs/{jobName}/trigger` - Trigger job (PLATFORM_OPERATOR)

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

### 1. Dashboard & Metrics

**Overview Dashboard** — Real-time platform health and key metrics (inspired by Makerkit admin)

| Metric                  | Description                                                      | Visualization       |
| ----------------------- | ---------------------------------------------------------------- | ------------------- |
| Active Users            | Total active users across all tenants (last 30 days)             | Number card + trend |
| Total Organizations     | Count of all organizations by status (ACTIVE, SUSPENDED, etc.)   | Number card + chart |
| Active Subscriptions    | Count of active paid subscriptions                               | Number card + trend |
| Trial Accounts          | Organizations currently in trial period with expiry countdown    | Number card + list  |
| Revenue Metrics         | MRR, ARR, churn rate (from Stripe data)                          | Charts + trends     |
| System Health           | Service status, database connections, queue depth                | Status indicators   |
| Recent Activity         | Latest signups, subscription changes, support tickets            | Activity feed       |
| Growth Metrics          | New signups, conversion rate, retention rate                     | Charts + trends     |

**Dashboard Features (Magento/OroCommerce-inspired):**
- **Customizable Widgets:** Drag-and-drop dashboard widgets
- **Date Range Selector:** Quick filters (today, last 7/30/90 days, custom range)
- **Real-time Updates:** WebSocket-based live metrics
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

### 2. User Management

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
- **Delete Account:** Permanently deletes user and all memberships (cascade) - `DELETE /api/v1/iam/admin/users/{id}`
- **Create User:** Create user with random temporary password - `POST /api/v1/iam/admin/users`
- **Update User:** Full or partial profile update - `PUT/PATCH /api/v1/iam/admin/users/{id}`

**Platform Operator Actions (To Be Implemented):**
- **Ban User:** Suspend account with reason and duration (temporary/permanent)
- **Unban User:** Restore suspended account
- **Verify Email:** Manually verify email address
- **Unlock Account:** Clear failed login attempts
- **Revoke All Sessions:** Global signout for user
- **Impersonate User:** Login as user for support purposes (with audit trail)
- **Send Notification:** Send direct email to user

---

### 3. Organization Management

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

### 4. Subscription & Billing Management

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

### 5. Platform Actions & Tools

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

### 6. System Administration

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
- User admin endpoints (`/api/v1/iam/admin/users`) should be secured with `PLATFORM_OPERATOR` authority
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
- **Tenant-Scoped Authorities:** `TENANT_OWNER`, `ADMIN`, `MEMBER` are restricted to their tenant context
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
GET    /api/v1/iam/admin/users                    # List users (paginated)
GET    /api/v1/iam/admin/users/{id}               # Get user by ID
POST   /api/v1/iam/admin/users                    # Create user
PUT    /api/v1/iam/admin/users/{id}               # Replace user (full update)
PATCH  /api/v1/iam/admin/users/{id}               # Partial update user
DELETE /api/v1/iam/admin/users/{id}               # Delete user
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
GET    /api/v1/iam/admin/dashboard/metrics        # Platform-wide metrics

# User Admin Actions (extend existing)
POST   /api/v1/iam/admin/users/{id}/ban           # Ban user
POST   /api/v1/iam/admin/users/{id}/unban         # Unban user
POST   /api/v1/iam/admin/users/{id}/impersonate   # Impersonate user
POST   /api/v1/iam/admin/users/{id}/unlock        # Unlock account
POST   /api/v1/iam/admin/users/{id}/verify-email  # Manually verify email
GET    /api/v1/iam/admin/users/{id}/memberships   # Get user's org memberships
GET    /api/v1/iam/admin/users/{id}/activity      # Get user activity log

# Organization Admin (extend existing)
GET    /api/v1/iam/admin/tenants                  # List all tenants (paginated)
GET    /api/v1/iam/admin/tenants/{tenantKey}      # Get tenant details (platform operator view)
POST   /api/v1/iam/admin/tenants/{tenantKey}/suspend      # Suspend organization
POST   /api/v1/iam/admin/tenants/{tenantKey}/unsuspend    # Unsuspend organization
DELETE /api/v1/iam/admin/tenants/{tenantKey}      # Delete organization
POST   /api/v1/iam/admin/tenants/{tenantKey}/transfer-ownership  # Transfer ownership
GET    /api/v1/iam/admin/tenants/{tenantKey}/export       # Export org data (GDPR)
GET    /api/v1/iam/admin/tenants/{tenantKey}/members      # List org members
GET    /api/v1/iam/admin/tenants/{tenantKey}/activity     # Get org activity log

# Subscription Admin (extend existing)
GET    /api/v1/billing/admin/subscriptions        # List all subscriptions (paginated)
GET    /api/v1/billing/admin/subscriptions/{id}   # Get subscription details
POST   /api/v1/billing/admin/subscriptions/{id}/change-plan    # Change plan
POST   /api/v1/billing/admin/subscriptions/{id}/cancel         # Cancel subscription
POST   /api/v1/billing/admin/subscriptions/{id}/reactivate     # Reactivate subscription
POST   /api/v1/billing/admin/subscriptions/{id}/apply-discount # Apply discount
POST   /api/v1/billing/admin/subscriptions/{id}/extend-trial   # Extend trial

# System Administration
GET    /api/v1/iam/admin/audit-log                # Global audit trail
GET    /api/v1/iam/admin/system/health            # Service health status
GET    /api/v1/iam/admin/system/jobs              # Background job status
POST   /api/v1/iam/admin/system/jobs/{jobName}/trigger  # Trigger job manually
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
- **@lingui/macro** - Compile-time message extraction
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
import { Trans, useLingui } from '@lingui/react';
import { msg } from '@lingui/macro';

function UserList() {
  const { _ } = useLingui();
  
  return (
    <div>
      <h1><Trans>User Management</Trans></h1>
      <Button>
        {_(msg`Create User`)}
      </Button>
      <p>
        <Trans>
          Total users: {userCount}
        </Trans>
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
// Store in backend (operator preferences)
PATCH /api/v1/iam/admin/operators/me/preferences
{
  "locale": "ru"
}

// Store in localStorage (fallback)
localStorage.setItem('admin-locale', 'ru');
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
  style: 'currency',
  currency: 'USD'
});
formatter.format(1234.56); // $1,234.56 (en) / 1 234,56 $ (fr)

// Date formatting
const dateFormatter = new Intl.DateTimeFormat(locale, {
  year: 'numeric',
  month: 'long',
  day: 'numeric'
});
dateFormatter.format(new Date()); // December 15, 2024 (en) / 15 décembre 2024 (fr)

// Relative time
const rtf = new Intl.RelativeTimeFormat(locale, { numeric: 'auto' });
rtf.format(-1, 'day'); // yesterday (en) / вчера (ru)
```

**Mantine Integration:**
- Mantine components automatically respect locale for date pickers
- Number inputs use locale-specific formatting
- Currency inputs with proper symbol placement

### Pluralization

**Lingui Plural Support:**
```tsx
import { Plural } from '@lingui/macro';

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
  en: () => import('./locales/en/messages'),
  ru: () => import('./locales/ru/messages'),
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
  locales: ['en', 'ru', 'uk', 'de', 'es', 'fr', 'it', 'pt', 'ja', 'zh-CN'],
  sourceLocale: 'en',
  catalogs: [
    {
      path: 'src/locales/{locale}/messages',
      include: ['src'],
      exclude: ['**/node_modules/**']
    }
  ],
  format: 'po'
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
- ✅ User admin CRUD API (`/api/v1/iam/admin/users`)
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
