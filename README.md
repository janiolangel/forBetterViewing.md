# AltPayNet HRIS Application Overview

Last updated: June 16, 2026

This document is the starting point for new contributors. Read it before making code changes. It explains what the system does, how the frontend and backend are split, where important files live, and how common HR workflows move through the application.

## 1. Product Summary

AltPayNet HRIS is an internal human resources platform. It supports employee self-service, HR administration, manager approvals, timekeeping, leave management, payroll-related records, benefits, and role/permission maintenance.

The application is split into:

- A Vue 3 frontend in `platform/`.
- Several Java Spring Boot backend services in `webservice/`.
- A Spring Cloud Gateway that routes browser/API traffic to backend services.
- A shared MySQL database for the current environment.
- RabbitMQ for notification-related messaging.
- NGINX for serving the frontend container and proxying `/api` traffic to the gateway.

At a high level:

```text
Browser
  -> platform NGINX
  -> /api
  -> apn-gateway
  -> apn-auth | apn-user | apn-workforce | apn-benefits
  -> MySQL / RabbitMQ / S3 where needed
```

## 2. Main User Roles

The platform uses JWT roles and permissions. The exact role/permission set is seeded and managed in `apn-auth`, but these are the practical roles you will see in the UI:

- Employee or user: clocks in/out, views profile, files leave, OT, WFH, and manual timekeeping requests.
- Manager: sees assigned team requests and can approve or reject relevant workflows.
- HR personnel/admin/superuser: manages employees, departments, benefits, payroll records, leave records, role permissions, and broader reports.

Important identity concepts:

- `username`: usually the login username/auth subject.
- `userId`: stable APN employee code stored in JWT claim `userId`; most business records use this value.
- `roles`: role authorities in the JWT, usually normalized as `ROLE_*`.
- `permissions`: permission authorities in the JWT, usually normalized as `PERM_*`.

## 3. Repository Layout

```text
altpaynet-chris/
  platform/                 Vue frontend
  webservice/               Backend services and docker compose
    apn-auth/               Auth/JWT/roles/permissions
    apn-gateway/            API gateway and Swagger aggregation
    apn-user/               Users, profiles, leave, notifications, org catalog
    apn-workforce/          Timekeeping, OT/WFH/manual requests, payroll
    apn-benefits/           Benefit definitions and eligibility
    apn-notifications/      Older/separate notification service scaffold
    docs/                   Backend setup/deployment docs
  docs/                     Top-level project docs and operational scripts
  scripts/                  Utility scripts
```

Useful docs already in the repo:

- Backend quick overview: `webservice/README.md`
- Frontend quick overview: `platform/README.md`
- Full local backend setup: `webservice/docs/DEV_ENV_FULL_STACK_SETUP.md`
- Deployment checklist: `docs/DEPLOYMENT_CHECKLIST.md`
- Environment variable reference: `docs/var_needed.md`

## 4. Frontend Overview

The frontend is in `platform/`.

Stack:

- Vue 3
- TypeScript
- Vite / rolldown-vite
- Pinia
- Vue Router
- PrimeVue and PrimeIcons
- TailwindCSS
- NGINX runtime container

Important frontend files:

```text
platform/src/main.ts                    App bootstrap
platform/src/App.vue                    Root shell and app-level UI
platform/src/router/index.ts            Router and route guard
platform/src/api/endpoints.ts           Central API URL registry
platform/src/utils/                     Shared frontend utilities
platform/src/stores/                    Pinia stores
platform/src/modules/                   Feature modules
platform/nginx.conf                     Runtime frontend/API proxy config
platform/entrypoint.sh                  NGINX env substitution and startup
```

### 4.1 Frontend Modules

Feature modules live under `platform/src/modules/`.

| Module | Purpose |
| --- | --- |
| `auth` | Login screen and auth views |
| `dashboard` | App shell, sidebar/layout, dashboard home, today status card |
| `employeeLogin` | Employee clock-in/clock-out view |
| `employeeManagement` | Employee list, department, group, job title, onboarding/offboarding |
| `profile` | Employee profile, attendance tab, leave balance, payroll, training, requests |
| `timekeeping` | HR/manager timekeeping, attendance logs, manual requests, OT, WFH |
| `reportsAndRequests` | Reports, leave management, request handling |
| `financesAndAssets` | Payroll and benefits management UI |
| `maintenance` | Role and permission management |
| `trainingManagement` | Training management views |

Some folders are present but are currently light scaffolds or less active, such as `announcementManagement`, `employeeRequirement`, and `recruitment`.

### 4.2 Routing

Routes are assembled in `platform/src/router/index.ts`.

Feature route files:

- `platform/src/modules/dashboard/router/index.ts`
- `platform/src/modules/employeeManagement/router/index.ts`
- `platform/src/modules/financesAndAssets/router/index.ts`
- `platform/src/modules/reportsAndRequests/router/index.ts`
- `platform/src/modules/maintenance/router/index.ts`

Public routes:

- `/login`
- `/employeeTimekeeping`

Authenticated routes generally render inside `DashboardLayout`.

Manager users are restricted by the frontend route guard to a smaller set of routes:

- `/dashboard`
- `/profile`
- `/employee-list`
- `/reports-and-requests`
- `/leave-management`
- `/employeeTimekeeping`

Do not rely only on frontend route guards for security. Backend services enforce authorization with JWT roles/permissions.

### 4.3 API Calls

All frontend endpoint strings should go through:

```text
platform/src/api/endpoints.ts
```

This keeps browser API calls relative to `/api`. The frontend container's NGINX proxy sends `/api/*` to `GATEWAY_URL`.

Examples:

- `endpoints.auth.login` -> `/api/auth/login`
- `endpoints.users.list` -> `/api/users`
- `endpoints.timekeeping.clockIn` -> `/api/timekeeping/clock-in`
- `endpoints.timekeeping.wfhRequestsPending` -> `/api/wfh/requests/pending`

Use `authedFetch` from `platform/src/utils/authedFetch.ts` for authenticated requests. It attaches the current token and handles auth behavior.

### 4.4 Pinia Stores

Stores live in `platform/src/stores/`.

| Store | Responsibility |
| --- | --- |
| `auth.ts` | Login, token persistence, refresh, user claims, logout |
| `dashboard.ts` | Dashboard data, today clock-in/out status, holidays |
| `timekeeping.ts` | Attendance records, manual/OT/WFH requests, exports |
| `employeeManagement.ts` | Employee CRUD, departments, groups, ranks, imports/exports |
| `profile.ts` | Profile-related local state |
| `payroll.ts` | Payroll API records |
| `benefitsManagement.ts` | Benefits API state |
| `notifications.ts` | Notification state |
| `networkStatus.ts` | Online/offline detection |
| `training.ts` | Training data |

When adding frontend behavior, prefer this flow:

```text
View/component
  -> Pinia store action
  -> endpoints.ts URL
  -> authedFetch
  -> backend endpoint
```

### 4.5 Shared Frontend Utilities

Shared utilities live in `platform/src/utils/`.

Important utilities:

- `dateTime.ts`: date keys, display dates, date ranges, time parsing, duration helpers.
- `downloads.ts`: response filename parsing, blob downloads, CSV downloads.
- `authedFetch.ts`: authenticated fetch wrapper.
- `storage.ts`: local storage JSON helpers.
- `roleLabels.ts`: role display helpers.
- `attachments.ts`: attachment URL helpers.
- `todayStatusStorage.ts`: persisted dashboard clock status.

Module-specific utilities live near the feature, for example:

- `platform/src/modules/timekeeping/utils/timekeepingUtils.ts`
- `platform/src/modules/timekeeping/utils/timekeepingViewHelpers.ts`
- `platform/src/modules/financesAndAssets/utils/payrollUtils.ts`
- `platform/src/modules/profile/utils/editProfileForm.ts`

Rule of thumb:

- Put generic reusable logic in `platform/src/utils`.
- Put feature-specific business formatting or mapping in that feature's `utils`.
- Do not duplicate date/time/download/status helpers if a shared util already exists.

## 5. Backend Overview

The backend is in `webservice/`.

Stack:

- Java 21
- Spring Boot
- Spring Security / OAuth2 resource server
- Spring Authorization Server in `apn-auth`
- Spring Cloud Gateway in `apn-gateway`
- JPA/Hibernate
- Flyway migrations
- MySQL
- RabbitMQ for notifications
- S3 for leave attachment storage where configured

Each backend service follows a common shape:

```text
src/main/java/com/altpaynet/<service>/
  application/       Business services
  domain/            Entities/domain models
  infrastructure/    Repositories, config, clients
  presentation/      Controllers and HTTP DTOs
src/main/resources/
  application*.properties
  db/migration/      Flyway migrations
```

Some packages use `web` instead of `presentation`, especially `apn-auth`.

### 5.1 Service Responsibilities

| Service | Local port | Responsibility |
| --- | ---: | --- |
| `apn-gateway` | 8080 | Routes external API traffic and aggregates Swagger/OpenAPI |
| `apn-auth` | 8081 | Login, refresh tokens, JWT signing, roles, permissions, auth users |
| `apn-user` | 8082 | Employee users, profiles, org catalog, leave requests, notifications |
| `apn-workforce` | 8083 | Timekeeping, attendance, OT/WFH/manual approvals, payroll records, holidays |
| `apn-benefits` | 8084 | Benefits definitions, benefit eligibility/enrollments, leave benefit data |
| MySQL | 3306 | Shared database for local/dev |
| RabbitMQ | 5672 / 15672 | Messaging and management UI |

Gateway route contracts are summarized in `webservice/README.md`:

- `/auth/**` and `/oauth2/**` -> `apn-auth`
- `/users/**` -> `apn-user`
- `/benefits/**` -> `apn-benefits`
- `/timekeeping/**`, `/overtime/**`, `/wfh/**`, `/payroll/**` -> `apn-workforce`

### 5.2 Auth Service

Path:

```text
webservice/apn-auth/
```

Main responsibilities:

- Authenticate users.
- Issue JWT access tokens and refresh tokens.
- Include `userId`, `roles`, and `permissions` in JWT claims.
- Manage roles and permissions.
- Provide JWKS for resource services to verify tokens.

Main controllers:

- `AuthController`
- `RoleManagementAdminController`
- `PermissionQueryController`

Important endpoints through gateway:

- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `GET /oauth2/jwks`
- `GET /api/auth/admin/users`
- `GET /api/auth/admin/roles`
- `GET /api/auth/admin/permissions`
- `PUT /api/auth/admin/roles/{roleName}/permissions`
- `PUT /api/auth/admin/users/{userId}/role`

Related frontend areas:

- `platform/src/stores/auth.ts`
- `platform/src/modules/auth/`
- `platform/src/modules/maintenance/`

### 5.3 User Service

Path:

```text
webservice/apn-user/
```

Main responsibilities:

- Employee user records and profile data.
- Department, group, rank, and membership catalog.
- User provisioning and validation.
- Leave filing and approval workflow.
- Leave balances and leave history.
- Leave attachments.
- Notifications.

Main controllers:

- `UserController`
- `UserCatalogController`
- `UserNotificationController`
- `UserProvisioningController`
- `UserValidationController`

Important endpoint groups:

- `/api/users`
- `/api/users/profile`
- `/api/users/profile/context`
- `/api/users/department`
- `/api/users/department/memberships`
- `/api/users/group`
- `/api/users/rank`
- `/api/users/leaves`
- `/api/users/leaves/pending`
- `/api/users/leaves/save`
- `/api/users/requests/{requestId}/approve`
- `/api/users/requests/{requestId}/reject`
- `/api/users/notifications`

Important tables/entities:

- `users`
- `APNEmployee`
- `APNDepartment`
- `APNEmployeeDepartment`
- `APNGroup`
- `APNRole`
- `user_change_requests`
- `user_change_approvals`
- `leave_requests`
- `user_notifications`

Related frontend areas:

- Employee list and org management: `platform/src/modules/employeeManagement/`
- Profile and leave balance: `platform/src/modules/profile/`
- Leave management: `platform/src/modules/reportsAndRequests/views/LeaveManagementView.vue`
- Notifications: `platform/src/stores/notifications.ts`

### 5.4 Workforce Service

Path:

```text
webservice/apn-workforce/
```

Main responsibilities:

- Attendance/timekeeping records.
- Clock-in and clock-out.
- Late, undertime, incomplete, present status derivation.
- Manual timekeeping correction requests.
- Overtime requests.
- Work-from-home requests.
- Manager approval/rejection for workforce requests.
- Payroll records.
- Holidays.
- Timekeeping CSV exports.

Main controllers:

- `TimekeepingController`
- `ManualTimekeepingRequestController`
- `OvertimeFilingController`
- `WorkFromHomeFilingController`
- `PayrollController`
- `HolidayController`

Important endpoint groups:

- `/api/timekeeping`
- `/api/timekeeping/clock-in`
- `/api/timekeeping/clock-out`
- `/api/timekeeping/manual-requests`
- `/api/overtime/requests`
- `/api/wfh/requests`
- `/api/payroll`
- `/api/workforce/holidays`

Important tables/entities:

- `APNDEmpAttendance`
- `workforce_change_requests`
- `workforce_change_approvals`
- `payroll_records`
- `apn_holiday`

Important business files:

- `DefaultTimekeepingService.java`
- `DefaultWorkforceRequestService.java`
- `DefaultWorkforceCatalogService.java`
- `DefaultPayrollService.java`
- `TimekeepingResponse.java`

Related frontend areas:

- Employee clocking: `platform/src/modules/employeeLogin/`
- Dashboard today status: `platform/src/stores/dashboard.ts`
- HR/manager timekeeping: `platform/src/modules/timekeeping/`
- Profile attendance tab: `platform/src/modules/profile/components/tabs/ProfileAttendanceTab.vue`
- Payroll UI: `platform/src/modules/financesAndAssets/`

### 5.5 Benefits Service

Path:

```text
webservice/apn-benefits/
```

Main responsibilities:

- Benefit definitions.
- Eligibility/enrollment checks.
- Leave benefit information for employees.

Main controller:

- `BenefitController`

Important endpoint groups:

- `/api/benefits`
- `/api/benefits/eligible`
- `/api/benefits/my-leave`

Important tables/entities:

- `benefit`
- `benefit_enrollment`

Related frontend areas:

- `platform/src/stores/benefitsManagement.ts`
- `platform/src/modules/financesAndAssets/`
- `platform/src/modules/profile/components/tabs/ProfileLeaveBalanceTab.vue`

### 5.6 Gateway Service

Path:

```text
webservice/apn-gateway/
```

Main responsibilities:

- Validate JWTs for protected routes.
- Route API requests to backend services.
- Serve aggregated Swagger/OpenAPI docs.
- Apply CORS rules.

Important files:

- `GatewayRouteService.java`
- `SecurityConfig.java`
- `SwaggerUiFallbackController.java`

Local Swagger:

- `http://localhost:8080/swagger-ui.html`
- `http://localhost:8080/swagger-ui/index.html`

## 6. Core Business Workflows

### 6.1 Login and Authorization

1. User submits credentials from `LoginView.vue`.
2. Frontend calls `POST /api/auth/login`.
3. `apn-auth` validates credentials and returns access/refresh tokens.
4. Frontend decodes/stores token data in `auth.ts`.
5. Later API calls use `authedFetch`, which attaches the bearer token.
6. Gateway and resource services validate the JWT.
7. Backend authorization uses roles/permissions mapped from JWT claims.

Common claims to check when debugging:

- `sub`
- `userId`
- `roles`
- `permissions`
- `exp`
- `jti`

### 6.2 Employee Creation and User Provisioning

Employee management flows usually touch both `apn-user` and `apn-auth`.

Typical path:

1. HR/admin creates or imports employee data in the frontend.
2. Frontend calls user service endpoints such as `/api/users/direct` or `/api/users/bulk-import`.
3. `apn-user` stores profile/catalog data.
4. When required, `apn-user` calls `apn-auth` to provision an auth user.
5. The auth user receives role and identity data needed for login/JWT claims.

Relevant frontend:

- `platform/src/modules/employeeManagement/views/EmployeeListView.vue`
- `platform/src/stores/employeeManagement.ts`

Relevant backend:

- `UserController`
- `UserProvisioningController`
- `AuthServiceClient`
- `AuthController`

### 6.3 Leave Filing and Approval

Leave workflow is split between pending requests and finalized leave records.

Important rule:

- `user_change_requests` stores pending leave requests.
- `leave_requests` stores finalized approved/rejected leave output.

Typical path:

1. Employee files leave from profile/leave UI.
2. Frontend calls `/api/users/leaves/save`.
3. `apn-user` creates a pending maker-checker request.
4. Manager/approver loads pending requests.
5. Manager approves or rejects using `/api/users/requests/{id}/approve` or `/reject`.
6. Approved/rejected result becomes visible in leave history and related reports.

Attachments:

- Leave attachments are handled by `AttachmentStorageService`.
- S3 storage is implemented by `S3AttachmentStorageService`.
- Frontend preview uses `AttachmentPreviewModal.vue`.

### 6.4 Clock-In and Clock-Out

Employee clocking uses `apn-workforce`.

Typical path:

1. Employee opens `/employeeTimekeeping` or dashboard timekeeping card.
2. Frontend calls `/api/timekeeping/clock-in` or `/api/timekeeping/clock-out`.
3. Workforce service resolves identity from JWT `userId`.
4. `DefaultTimekeepingService` creates or updates `APNDEmpAttendance`.
5. Frontend refreshes today status and attendance records.

Attendance status concepts:

- `Present`: complete day with enough rendered work time.
- `Late`: clock-in beyond the schedule threshold.
- `Incomplete`: clock-in without clock-out.
- `Undertime`: completed work below required minutes or recorded undertime.
- `Absent` or `No Log`: no attendance record for that date.

Schedule rules in the current service:

- `FLEXI`: never late.
- `SEMI_FLEXI`: on time from 07:00 through 09:30.
- `STRICT`: currently late after 09:00.

### 6.5 Manual Timekeeping, Overtime, and WFH Requests

These workflows use maker-checker style pending requests in `apn-workforce`.

Shared concepts:

- Employee files request.
- Request is saved in `workforce_change_requests`.
- Assigned department manager can view pending requests.
- Manager approves or rejects.
- Approval data is tracked in `workforce_change_approvals`.

Manual timekeeping:

- Endpoint group: `/api/timekeeping/manual-requests`
- Approval applies correction to `APNDEmpAttendance`.

Overtime:

- Endpoint group: `/api/overtime/requests`
- Approved overtime contributes to rendered minutes/export calculations.

Work from home:

- Endpoint group: `/api/wfh/requests`
- Pending WFH approvals are visible to the assigned manager.

Relevant frontend:

- `platform/src/modules/timekeeping/views/timekeepingView.vue`
- `platform/src/modules/timekeeping/views/components/TimekeepingDailyTab.vue`
- `platform/src/modules/timekeeping/views/components/TimekeepingOvertimeTab.vue`
- `platform/src/modules/timekeeping/views/components/TimekeepingWfhTab.vue`
- `platform/src/stores/timekeeping.ts`

### 6.6 Payroll

Payroll records are handled by `apn-workforce` and shown in `financesAndAssets`.

Typical features:

- List payroll records.
- Add/update/delete payroll records.
- Import payroll records.
- Generate/export frontend template data.

Important files:

- `webservice/apn-workforce/.../PayrollController.java`
- `webservice/apn-workforce/.../DefaultPayrollService.java`
- `platform/src/stores/payroll.ts`
- `platform/src/modules/financesAndAssets/views/financesAndAssetsView.vue`
- `platform/src/modules/financesAndAssets/utils/payrollUtils.ts`

### 6.7 Benefits

Benefits are managed by `apn-benefits`.

Typical path:

1. Frontend loads benefits through `benefitsManagement.ts`.
2. Admin/HR creates or updates benefits.
3. Employee-specific benefit data can be shown in profile/benefits UI.

## 7. Database and Migrations

The services use Flyway migrations in each service's resources:

```text
webservice/apn-auth/src/main/resources/db/migration
webservice/apn-user/src/main/resources/db/migration
webservice/apn-workforce/src/main/resources/db/migration
webservice/apn-benefits/src/main/resources/db/migration
```

Local compose uses a shared MySQL database:

```text
Database: apn_chris_db
User: apn_chris
Default local password: apn_chris_pwd
```

Migration guidance:

- Add schema changes as new Flyway migration files.
- Do not edit already-applied migrations for shared environments.
- Keep migration names descriptive.
- If a migration needs seed data, make it idempotent when possible.
- Be careful with legacy table names such as `APNEmployee`, `APNDepartment`, and `APNDEmpAttendance`; casing matters in some environments.

## 8. Local Development

### 8.1 Backend Full Stack

From `webservice/`:

```bash
docker compose --env-file .env.local up -d --build
```

Useful health checks:

```text
Gateway:   http://localhost:8080/health
Auth:      http://localhost:8081/actuator/health
User:      http://localhost:8082/actuator/health
Workforce: http://localhost:8083/actuator/health
Benefits:  http://localhost:8084/actuator/health
RabbitMQ:  http://localhost:15672
```

Use `webservice/docs/DEV_ENV_FULL_STACK_SETUP.md` for the complete setup guide.

### 8.2 Frontend

From `platform/`:

```bash
npm ci
npm run dev
```

Default dev URL:

```text
http://localhost:5173
```

Production build:

```bash
npm run build
```

Current local note:

- The installed Vite version requests Node `20.19+` or `22.12+`.
- Node `20.14.0` may print a warning.
- In this workspace, sandboxed Vite builds can hit Windows `spawn EPERM`; rerunning outside the sandbox has completed successfully.

## 9. Testing and Verification

Frontend:

```bash
cd platform
.\node_modules\.bin\vue-tsc.cmd -b
npm run build
```

Backend service tests:

```bash
cd webservice/apn-workforce
.\mvnw.cmd test
```

Run a focused backend test:

```bash
cd webservice/apn-workforce
.\mvnw.cmd test -Dtest=DefaultTimekeepingServiceTests
```

Other services have their own Maven wrappers:

```text
webservice/apn-auth/mvnw.cmd
webservice/apn-user/mvnw.cmd
webservice/apn-workforce/mvnw.cmd
webservice/apn-benefits/mvnw.cmd
webservice/apn-gateway/mvnw.cmd
```

Before opening or merging a change, try to verify the smallest affected unit plus the frontend type check if the UI changed.

## 10. Debugging Guide

### 10.1 Login Problems

Check:

- Frontend endpoint: `platform/src/api/endpoints.ts`
- Auth store: `platform/src/stores/auth.ts`
- Gateway route to auth service.
- Auth service logs.
- JWT claims: `userId`, `roles`, `permissions`, `exp`.
- Browser network tab: status code and response body.

Common causes:

- Wrong gateway URL.
- Expired token and failed refresh.
- Missing JWT `userId`.
- Missing permissions for a backend `@PreAuthorize` check.

### 10.2 Request Not Visible to Manager

Check:

- Is the request pending?
- Is the manager assigned to the requester's department?
- Does the request use the expected maker `userId`?
- Does backend `canView` include the current actor identifiers?
- Is the frontend hiding rows due to search/date/status filters?

Relevant backend:

- `DefaultWorkforceRequestService`
- `DefaultWorkforceCatalogService`
- `DefaultUserRequestService`
- `DefaultUserCatalogService`

Relevant frontend:

- `platform/src/stores/timekeeping.ts`
- `platform/src/modules/timekeeping/views/timekeepingView.vue`
- `platform/src/modules/reportsAndRequests/views/LeaveManagementView.vue`

### 10.3 Attendance Status Looks Wrong

Check:

- Backend `DefaultTimekeepingService`.
- `TimekeepingResponse`.
- Frontend attendance helpers in `platform/src/modules/timekeeping/utils/`.
- Profile attendance rollup in `ProfileAttendanceTab.vue`.
- Schedule type: `FLEXI`, `SEMI_FLEXI`, or `STRICT`.
- Stored `late`, `timeIn`, `timeOut`, and `undertime` values.

### 10.4 API Works Directly But Not From Browser

Check:

- Browser calls should use `/api`, not internal service hostnames.
- `platform/src/api/endpoints.ts` normalizes `VITE_API_BASE_URL`.
- `platform/nginx.conf` proxies `/api/*`.
- `GATEWAY_URL` is set correctly in the frontend container.
- Gateway CORS settings allow the frontend origin.

## 11. Coding Conventions

Frontend:

- Use `endpoints.ts` for API URLs.
- Use `authedFetch` for authenticated calls.
- Use Pinia stores for API-backed shared state.
- Put generic helpers in `platform/src/utils`.
- Put feature-only helpers in the feature module's `utils`.
- Keep display formatting separate from stored/API values.
- Keep route-level access checks in the router, but enforce real security on the backend.
- Run `vue-tsc` after TypeScript/Vue changes.

Backend:

- Keep controllers thin.
- Put business rules in application services.
- Put persistence in repositories/entities.
- Use DTO records for HTTP request/response payloads.
- Resolve identity from JWT claims, especially `userId`.
- Use `@PreAuthorize` for permission checks.
- Add or update focused tests for behavior changes.
- Use Flyway for schema changes.

General:

- Prefer small, focused changes.
- Do not duplicate date, time, download, and status helpers.
- Avoid changing unrelated files during a bug fix.
- When fixing a bug, add a test or at least document the verification path.

## 12. Important Paths by Task

Login/auth:

```text
platform/src/stores/auth.ts
platform/src/modules/auth/views/LoginView.vue
webservice/apn-auth/src/main/java/com/altpaynet/auth/
webservice/apn-gateway/src/main/java/com/altpaynet/gateway/
```

Employee management:

```text
platform/src/modules/employeeManagement/
platform/src/stores/employeeManagement.ts
webservice/apn-user/src/main/java/com/altpaynet/user/
```

Profile:

```text
platform/src/modules/profile/views/profileView.vue
platform/src/modules/profile/components/tabs/
webservice/apn-user/src/main/java/com/altpaynet/user/application/DefaultUserProfileService.java
```

Leave:

```text
platform/src/modules/profile/components/tabs/ProfileLeaveBalanceTab.vue
platform/src/modules/reportsAndRequests/views/LeaveManagementView.vue
webservice/apn-user/src/main/java/com/altpaynet/user/application/DefaultUserRequestService.java
```

Timekeeping:

```text
platform/src/modules/employeeLogin/
platform/src/modules/timekeeping/
platform/src/stores/timekeeping.ts
webservice/apn-workforce/src/main/java/com/altpaynet/workforce/application/DefaultTimekeepingService.java
```

OT/WFH/manual requests:

```text
platform/src/modules/timekeeping/views/timekeepingView.vue
platform/src/modules/profile/components/tabs/ProfileAttendanceTab.vue
webservice/apn-workforce/src/main/java/com/altpaynet/workforce/application/DefaultWorkforceRequestService.java
```

Payroll:

```text
platform/src/modules/financesAndAssets/
platform/src/stores/payroll.ts
webservice/apn-workforce/src/main/java/com/altpaynet/workforce/application/DefaultPayrollService.java
```

Benefits:

```text
platform/src/stores/benefitsManagement.ts
webservice/apn-benefits/src/main/java/com/altpaynet/benefits/
```

Roles and permissions:

```text
platform/src/modules/maintenance/
webservice/apn-auth/src/main/java/com/altpaynet/auth/web/RoleManagementAdminController.java
webservice/apn-auth/src/main/java/com/altpaynet/auth/web/PermissionQueryController.java
```

## 13. Suggested First Week for Interns

Day 1:

- Read this document.
- Run the frontend locally.
- Read `platform/src/api/endpoints.ts`, `platform/src/stores/auth.ts`, and `platform/src/router/index.ts`.
- Log in and click through dashboard, profile, timekeeping, leave management, and employee list.

Day 2:

- Run the backend stack with Docker Compose.
- Open Swagger through the gateway.
- Trace one login request and one authenticated request in the browser Network tab.

Day 3:

- Pick one workflow and trace it end to end:
  - Leave filing and approval, or
  - Clock-in/clock-out, or
  - WFH request and approval.

Day 4:

- Make a tiny frontend-only change using existing patterns.
- Run `vue-tsc` and build.

Day 5:

- Pair on a backend bug or small feature.
- Add a focused test if behavior changes.

## 14. Glossary

APN code:

- Stable employee/user code carried in JWT `userId`.

Maker-checker:

- Pattern where the employee/requester creates a pending request and a manager/checker approves or rejects it.

Rendered minutes:

- Worked minutes plus approved overtime minutes where applicable.

Pending request:

- A request waiting for approval/rejection.

Finalized leave:

- Leave request that has been approved or rejected and written to leave history.

Gateway:

- Spring Cloud Gateway service that validates/routes API traffic.

Resource service:

- Backend service that accepts JWTs and serves business APIs, such as user, workforce, and benefits.

Flyway:

- Database migration tool used by backend services.
