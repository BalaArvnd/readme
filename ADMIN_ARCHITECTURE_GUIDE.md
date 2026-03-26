# Zopping Admin Dashboard — Complete Architecture Guide

## 1. What Is This Project?

**Zopping** is a **multi-tenant SaaS e-commerce platform** (think Shopify for India). This repo is the **admin dashboard** — the control panel where store owners and staff manage orders, products, customers, delivery, marketing, and more.

- **Tech**: React 16 SPA, Create React App (CRA) based
- **Scale**: 21 feature domains, 70+ reusable components, integrates with **26 microservices**
- **Users**: Store owners, staff with varying permission levels, enterprise clients

---

## 2. High-Level Architecture

```
Browser
  └── index.js (Bugsnag error boundary + BrowserRouter)
        └── App.js (Session init, notifications, analytics, GTM, Faro)
              └── router.js (Permission-based routing + lazy loading)
                    ├── PublicPage (Login, Signup, ForgotPassword)
                    └── PrivateRoute → AuthenticatedPage
                          ├── NavBar (top bar)
                          ├── Menu (sidebar)
                          └── Page Content (21 feature domains)
                                └── API Layer → 26 Backend Microservices
```

**Interview talking point**: "It's a classic SPA admin panel with a microservices backend. The frontend acts as an API gateway aggregator — a single React app consuming 26 different microservices through a unified API client."

---

## 3. Project Structure (What Goes Where)

| Directory | Purpose | Interview Keywords |
|-----------|---------|-------------------|
| `src/config/` | Environment config (API hosts, keys, Firebase) | Environment management |
| `src/containers/` | Layout wrappers + route guards (App, PrivateRoute, AuthenticatedPage) | Container/Presentational pattern |
| `src/components/` | 70+ reusable UI components (Form, Table, Modal, etc.) | Component library |
| `src/pages/` | 21 feature domains (orders, catalogue, delivery, etc.) | Feature-based organization |
| `src/lib/` | Core infrastructure (API client, auth, i18n, storage) | Service layer |
| `src/hooks/` | Custom hooks (notifications, push notifications) | React Hooks |
| `src/utils/` | Pure utility functions | Helpers |
| `src/services/` | External integrations (Firebase) | Third-party services |

---

## 4. State Management — No Redux!

This is a **big interview discussion point**. The project deliberately avoids Redux/MobX and uses:

1. **Component-level `setState`** — Each page/component manages its own state
2. **React Context API** — For cross-cutting concerns (enterprise mode, store selection, trip IDs, payment modes)
3. **localStorage** — For session persistence (auth token, user, org, permissions)
4. **Window CustomEvents** — For cross-component communication (`storeChanged`, `breadCrumb:updated`)
5. **Utility helpers** in `src/lib/stateManagement/` — `updateStateRecursively()`, `getNestedState()`, `compareValues()`

**Why this matters for interviews**: You can discuss trade-offs:
- **Pros**: Simpler mental model, no boilerplate, no action/reducer ceremony, faster to onboard
- **Cons**: No single source of truth, prop drilling in deep trees, harder to debug state changes, no time-travel debugging
- "For an admin dashboard where most pages are independent CRUD views, the trade-off was reasonable — each page fetches its own data and manages its own lifecycle"

---

## 5. API Layer — The `API` Class

`src/lib/api/index.js` — Custom Axios wrapper

```javascript
// Usage pattern across the app:
const api = new API({ url: "/order-service/order" });
const response = await api.get({ storeId: 123 });
```

**Key features**:
- **Service Host Mapping**: Maps 26 microservice prefixes (e.g., `order-service`, `catalogue-service`) to the Go backend host
- **Auto auth injection**: Attaches JWT Bearer token from localStorage
- **Organization ID injection**: For multi-tenancy
- **Request cancellation**: Built-in `CancelToken` support (prevents memory leaks on unmount)
- **API versioning**: Custom `X-API-VERSION` headers per service
- **Language headers**: For i18n responses

**26 Microservices Consumed**:
account-service, billing-service, blog-service, catalogue-service, communication-service, customer-service, logistics-service, media-service, order-service, promo-service, ticket-service, website-service, public-service, offer-service, inventory-service, config-service, analytics-service, report-service, auth-service, wallet-service, bulk-processing-service, gateway-internal, credit-service, partner-integrations, master-service, reporting-aggregator-service

**Interview talking point**: "We built a thin abstraction over Axios that handles auth injection, multi-tenancy headers, and service routing — essentially a client-side API gateway pattern."

---

## 6. Authentication & Authorization

### Auth Flow
1. User logs in → JWT stored in localStorage
2. Session data saved: `{ user, organization, billingSettings, permissions, stores }`
3. Every API call auto-attaches `Authorization: Bearer <token>`
4. On 401 → redirect to `/user/logout` → clear session

### Permission System (RBAC)
`src/lib/auth/index.js` — **Endpoint-level permissions**

```javascript
// Permissions are stored as:
{ "order-service/order": { "GET": true, "POST": true, "PUT": false, "DELETE": false } }

// Checked via:
hasPermissions("order-service", "order", "PUT") // → false
```

### Route Protection
`src/containers/PrivateRoute/index.jsx` — Two-layer access control:
1. **Endpoint permissions**: Can the user access the underlying API?
2. **Extension flags**: Is the feature enabled for this organization? (e.g., `Marketing`, `HrManagement`, `AbandonedCart`)

```javascript
// Router defines required permissions per route:
{ slug: "orders", endpoints: ["order-service/order"], extensions: [], href: "orders/orders" }
```

**Interview talking point**: "We implemented endpoint-level RBAC — permissions mirror the backend API structure, so frontend route access is tied directly to what API calls the user is allowed to make. This prevents UI/API permission drift."

---

## 7. Routing & Code Splitting

`src/containers/App/router.js` — React Router v4

- **Public routes**: `/login`, `/signUp`, `/forgot-password`, `/verify`
- **Private routes**: Everything else, wrapped in `<PrivateRoute>`
- **Lazy loading**: All major page sections use `React.lazy()` with `Suspense`
  ```javascript
  const Operations = LazyComponent(() => import(/* webpackChunkName: "operations" */ '../../pages/operations'))
  ```
- **Prefetching**: Critical chunks use `/* webpackPrefetch: true */`

**Interview talking point**: "We use route-based code splitting with webpack magic comments for named chunks and prefetching. Each feature domain is a separate bundle, so the initial load only includes the dashboard."

---

## 8. Component Architecture

### The Form System — Custom BaseForm
`src/components/Form/index.js` — Class-based form framework

Every form in the app extends `BaseForm`:
```javascript
class OrderForm extends BaseForm {
  // generateStateMappers() auto-binds state ↔ inputs
  // Built-in validation (ALWAYS, ONCHANGE, ONBLUR, ONSUBMIT)
  // isFormValid(), beforeSubmit(), getState()
}
```

**23 input types**: Text, Select, Phone, Checkbox, Radio, Toggle, DateRangePicker, ImageUpload, RichTextEditor (Quill), ProductSearch, CategorySearch, and more.

### The ListingPage Pattern
`src/containers/ListingPage/index.js` — Reusable CRUD container

Most pages follow this pattern:
```javascript
<ListingPage
  api={{ url: "/order-service/order", transform: (r) => r.data }}
  primaryKey="referenceNumber"
  tableProperties={TableView}
  form={{ component: OrderForm }}
  filters={{ component: FilterForm }}
/>
```

This is a **huge architectural pattern** — one container handles:
- Data fetching + pagination
- CRUD operations (create/read/update/delete)
- Filtering and search
- Loading states
- Store-dependent data refresh

**Interview talking point**: "We built a declarative ListingPage abstraction — you just pass it an API config, table definition, and form component, and it handles the entire CRUD lifecycle. This reduced boilerplate across 50+ listing pages."

### Modal System
`src/components/Popup/` — Portal-based modals with:
- ESC key handling
- Click-outside-to-close
- Body scroll lock
- Focus management

### Table Component
`src/components/Table/index.js` — Compositional pattern:
```javascript
<Table>
  <Header>...</Header>
  <Row>
    <Cell>...</Cell>
  </Row>
</Table>
```

### Notification System
- **Toast notifications**: `useNotificationManager` hook — manages toast queue with auto-dismiss
- **Push notifications**: `usePushNotifications` hook — Firebase Cloud Messaging with token caching (7-day TTL), retry with exponential backoff
- **Audio alerts**: Plays notification sound for new orders

---

## 9. Styling Approach

- **Plain CSS** with **CSS Custom Properties** (variables)
- **No CSS-in-JS**, no Tailwind, no SCSS
- Design tokens in `src/containers/App/tokens.css`
- Primary color: `#4ab819` (green), Error: `#fa1f4b`
- One CSS file per component (co-located)
- Font: Open Sans

**CSS Variables**:
```css
--primary-color: #4ab819;
--primary-text-color: #2b3238;
--secondary-text-color: #80959d;
--border-color: #dadee0;
--bg-color: #fbfcfc;
--color-error: #fa1f4b;
--border-radius: 6px;
--transition: 0.2s ease;
```

---

## 10. Observability & Monitoring Stack

| Tool | Purpose | Integration Point |
|------|---------|-------------------|
| **Bugsnag** | Error tracking | ErrorBoundary component wraps entire app |
| **Grafana Faro** | Real User Monitoring (RUM) | `src/initialize.js` with OpenTelemetry |
| **Google Tag Manager** | Analytics events | Injected in App.js |
| **Google Analytics** | Page tracking | Via GTM |
| **Firebase Cloud Messaging** | Push notifications | `src/services/firebaseService.js` |

**Error Boundary Features**:
- Bugsnag integration for automatic error capture
- Chunk loading error detection with auto-reload (3 retries)
- Faro analytics error reporting
- Filters out auth errors (401, 403) from reporting

**Interview talking point**: "We have a layered observability stack — Bugsnag for error tracking with React Error Boundaries, Grafana Faro with OpenTelemetry for distributed tracing and RUM, and GTM for business analytics."

---

## 11. Internationalization (i18n)

`src/lib/translator/index.js`

- **4 languages**: English, Hindi, Arabic, Kannada
- Uses **MessageFormat** library for pluralization/parameterization
- Language stored in localStorage, loaded on app init
- Dynamic dataset loading per language
- `getMessage('key', { count: 5 })` API

---

## 12. Multi-Tenancy & Multi-Store

This is architecturally significant:
- **Organization-level**: Each tenant (org) has its own settings, extensions, and billing
- **Store-level**: One org can have multiple stores; the StoreSelector lets users switch
- **Store change propagation**: Uses `window.dispatchEvent(new CustomEvent("storeChanged"))` — all listening components re-fetch data
- **API injection**: `storeId` and `organizationId` auto-injected into API calls

---

## 13. Firebase Integration

`src/services/firebaseService.js` — `FirebaseNotificationService` class

- **Token Management**: FCM token with 7-day cache TTL
- **Retry Logic**: Exponential backoff (3 attempts) for token fetch
- **Service Worker**: Custom SW at `/admin/firebase-messaging-sw.js`
- **Foreground Handling**: `setupForegroundHandler(callback)` for in-app notifications
- **Backend Registration**: `sendTokenToBackend(token, userId)` for device tracking

---

## 14. Build & Deployment

- **Build Tool**: Create React App (react-scripts 3.4.1)
- **Node Version**: 16.14.2 (pinned in `.nvmrc`)
- **Docker**: Production Dockerfile for containerized deployment
- **Versioning**: Semantic-release for automated version bumps
- **CI/CD**: GitHub Actions workflows
- **Hosting**: Firebase hosting support + Docker deployment
- **Storybook**: Component documentation on port 9009

---

## 15. Key Interview Discussion Topics

### Architecture Decisions You Should Be Ready to Discuss

1. **"Why no Redux?"** — Admin dashboards are mostly independent CRUD pages. Component state + Context was sufficient. Trade-off: simplicity vs. centralized debugging.

2. **"How do you handle cross-cutting state?"** — React Context for global flags, localStorage for persistence, Window CustomEvents for inter-component communication.

3. **"How does the permission system work?"** — Endpoint-level RBAC mirroring the backend API structure, plus extension-based feature flags. Two-layer check at route level.

4. **"How do you handle code splitting?"** — Route-based lazy loading with React.lazy + Suspense. Named webpack chunks with prefetching for critical paths.

5. **"Describe your API architecture"** — Single React app consuming 26 microservices through a unified API client class. Auto-handles auth, multi-tenancy, versioning, and cancellation.

6. **"How do you handle errors?"** — React Error Boundaries + Bugsnag at the top level. Chunk loading failures auto-retry (3 attempts). Auth errors (401/403) filtered from error reporting.

7. **"What's your form strategy?"** — Custom BaseForm class with 23 input types, built-in validation pipeline (ALWAYS/ONCHANGE/ONBLUR/ONSUBMIT), and auto state binding via `generateStateMappers()`.

8. **"How do you manage real-time features?"** — Firebase Cloud Messaging for push notifications with token caching (7-day TTL), retry logic with exponential backoff, and WebSocket support via `wss://events.zopsmart.com/ws/store`.

### Technical Depth Questions You Might Get

- **Performance**: Code splitting, lazy loading, request cancellation on unmount, debounced layout updates
- **Security**: JWT in localStorage (discuss trade-offs vs. httpOnly cookies), HTTPS redirect, reCAPTCHA, permission checks on both frontend and backend
- **Testing**: CodeceptJS E2E tests with Puppeteer (not unit tests — this is worth acknowledging honestly)
- **DevOps**: Docker deployment, semantic versioning, CI validation of MR titles, Firebase hosting
- **Monitoring**: OpenTelemetry instrumentation, Faro RUM, Bugsnag error boundaries

### What's "Legacy" (Be Honest About It)

- React **16** (not 18) — no concurrent features, no automatic batching
- React Router **v4** (not v6) — older API, no data loaders
- **Class components** still prevalent (BaseForm, ListingPage, many pages)
- **CRA-based** (not Vite/Next.js) — slower builds, no SSR
- No unit/integration tests (only E2E)
- axios **0.19** (old version)

Being honest about tech debt shows maturity. Frame it as: "I understand the trade-offs and what a migration path would look like."

---

## 16. Quick Reference — Key File Paths

| Pattern | File |
|---------|------|
| App entry | `src/index.js` |
| Root component | `src/containers/App/App.js` |
| Router | `src/containers/App/router.js` |
| API client | `src/lib/api/index.js` |
| Auth library | `src/lib/auth/index.js` |
| Route guard | `src/containers/PrivateRoute/index.jsx` |
| Form system | `src/components/Form/index.js` |
| CRUD container | `src/containers/ListingPage/index.js` |
| Modal system | `src/components/Popup/Modal/index.js` |
| Table component | `src/components/Table/index.js` |
| Notifications hook | `src/hooks/useNotificationManager.js` |
| Push notifications | `src/hooks/usePushNotifications.js` |
| Firebase service | `src/services/firebaseService.js` |
| i18n translator | `src/lib/translator/index.js` |
| Error boundary | `src/components/ErrorBoundary/index.js` |
| State management utils | `src/lib/stateManagement/index.js` |
| Storage wrapper | `src/lib/storage/index.js` |
| Environment config | `src/config/app.js` |
| Design tokens | `src/containers/App/tokens.css` |
| Global styles | `src/containers/App/app.css` |

---

## 17. Pages & Feature Domains

| Page | Domain | Key Microservices |
|------|--------|-------------------|
| `pages/operations/` | Orders, Returns, Customers | order-service, customer-service |
| `pages/catalogue/` | Products, Categories, Brands | catalogue-service |
| `pages/delivery/` | Trips, Vehicles, Logistics | logistics-service |
| `pages/marketing/` | Coupons, Campaigns, Promos | promo-service, offer-service |
| `pages/analytics/` | Reports, Dashboards | report-service, analytics-service |
| `pages/hr/` | Employees, Attendance | account-service |
| `pages/settings/` | 39 sub-modules (themes, billing, permissions, etc.) | Multiple services |
| `pages/online-store/` | Website builder, CMS layouts | website-service |
| `pages/customer-support/` | Tickets, Calls | ticket-service |
| `pages/pricing/` | Plans, Billing | billing-service |
| `pages/home/` | Dashboard | Multiple services |
| `pages/sellers/` | Multi-seller management | account-service |
| `pages/stores/` | Store management | account-service |
| `pages/reviews/` | Product reviews | catalogue-service |
| `pages/bookings/` | TidyCal/OneHashCal bookings | External integrations |
| `pages/abandonedCart/` | Cart recovery | website-service |

---

*Generated for interview preparation reference.*
