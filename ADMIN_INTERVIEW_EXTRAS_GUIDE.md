# Zopping Admin — Interview Extras (Topics Not in Other Guides)

This covers everything NOT already in ARCHITECTURE_GUIDE.md, ZS_COMPONENTS_GUIDE.md, or DATA_FLOW_GUIDE.md.

---

## 1. Performance Optimization Patterns

### Code Splitting & Lazy Loading
```javascript
// src/containers/LazyComponentLoader/index.jsx
// Every major page section is lazy loaded:
const Operations = LazyComponent(() =>
  import(/* webpackChunkName: "operations" */ '../../pages/operations')
);
const Catalogue = LazyComponent(() =>
  import(/* webpackChunkName: "catalogue", webpackPrefetch: true */ '../../pages/catalogue')
);

// LazyComponent wraps React.lazy + adds .preLoad() for pre-fetching
// Suspense fallback: <FallbackLoader /> (shows menu + spinner)
```

**Interview talking point:** "We use route-based code splitting. Each of the 21 feature domains is a separate webpack chunk. Critical paths use `webpackPrefetch: true` so the browser pre-fetches them during idle time."

### React.memo & useCallback (~12+ occurrences)
```javascript
// NavBar memoized to prevent re-renders on parent state changes:
// src/components/NavBar/index.js
const NavBar = React.memo((props) => { ... });

// useCallback in notification hooks to stabilize references:
// src/hooks/useNotificationManager.js
const addNotification = useCallback((notification) => { ... }, []);
const removeNotification = useCallback((id) => { ... }, []);
```

### shouldComponentUpdate (3 occurrences)
```javascript
// Used in PhoneNumberUI and RadialMapComponent
// Prevents expensive re-renders for phone number popup and Google Maps
shouldComponentUpdate(nextProps, nextState) {
  return nextProps.value !== this.props.value || nextState !== this.state;
}
```

### Debouncing (~14+ occurrences)
```javascript
// lodash.debounce used heavily in:
// - Layout resize handlers (400ms) — ImageSlideshow, Footer, ProductDetail, etc.
// - Search inputs — Searchable, SelectSearch form inputs
// - Website builder preview updates (300ms) — useLayoutUpdater hook
// - Form auto-save

// Example: src/components/Layout/ImageSlideshow/index.js
this.handleResize = debounce(this.handleResize.bind(this), 400);
window.addEventListener('resize', this.handleResize);
```

### Request Cancellation
```javascript
// Every API instance has built-in CancelToken:
// src/lib/api/index.js
constructor({ url }) {
  this.signal = axios.CancelToken.source();
}
cancel() {
  this.signal.cancel("Cancelled");
}
// Components call api.cancel() in componentWillUnmount to prevent memory leaks
```

**Interview talking point:** "We debounce resize handlers at 400ms, search inputs dynamically, and cancel in-flight API requests on unmount using Axios CancelTokens. NavBar is React.memo'd, and notification handlers use useCallback to prevent unnecessary child re-renders."

---

## 2. Security Patterns

### Authentication Security
```
- JWT stored in localStorage (not httpOnly cookies)
  - Trade-off: Vulnerable to XSS but simpler for SPA architecture
  - Mitigation: CSP headers, input sanitization, HTTPS enforcement
- HTTPS auto-redirect in index.html (except localhost/staging)
- Google reCAPTCHA on login (test key for staging, real key for production)
- Bearer token in Authorization header (not URL params)
- Session cleared on 401 response → redirect to login
```

### XSS Concerns
```
- dangerouslySetInnerHTML used in 5 files:
  1. PhoneNumberUI/PopUp — renders flag emoji HTML
  2. FlagEmoji component — renders emoji SVG
  3. RichTextLayout — renders user-authored rich text content ⚠️
  4. IndividualSupportComponent — renders support article HTML ⚠️
  5. marketing/Campaigns/Form/utils — renders campaign preview ⚠️

- NO DOMPurify or HTML sanitization library installed
  - This is a genuine vulnerability worth acknowledging in interviews
  - Mitigation: Content comes from admin-authored sources (not user-generated)
  - Ideal fix: Add DOMPurify before dangerouslySetInnerHTML
```

### CSRF Protection
```
- No explicit CSRF tokens found
- Relying on:
  1. JWT Bearer token in headers (not cookies, so CSRF less relevant)
  2. CORS configuration on backend
  3. Same-origin policy
```

**Interview talking point:** "We use JWT in localStorage — I'm aware this is vulnerable to XSS compared to httpOnly cookies, but it simplifies our SPA architecture. We have a few places using dangerouslySetInnerHTML for rich text content. If I were to improve security, I'd add DOMPurify sanitization and consider moving to httpOnly cookie-based auth."

---

## 3. Real-Time Features

### WebSocket Connection
```javascript
// src/registerSocket.js
export default function registerSocket() {
  const token = getAuthToken();
  const storeId = getDefaultStore();

  const socket = new WebSocket(
    `wss://events.zopsmart.com/ws/store?storeId=${storeId}&guid=${token}`
  );
  return socket;
}
// Used for: real-time order updates, store events
// NOT called on app boot — called when specific pages need it
```

### Firebase Push Notifications
```
Flow:
1. App boot → usePushNotifications hook initializes
2. Checks browser support (serviceWorker + Notification API + PushManager)
3. Requests permission → gets FCM token (cached 7 days)
4. Registers device: POST /account-service/device
5. Foreground: Firebase onMessage() → transforms → toast notification
6. Background: Service worker shows native browser notification
7. New order notification → plays audio alert

Retry: 3 attempts with exponential backoff
Timeout: 10 seconds for token fetch
Cache: 7-day TTL in localStorage
```

### Custom Event System (Pub/Sub)
```javascript
// 7 custom events used across the app:
// 1. "storeChanged"       — Menu → all store-dependent components re-fetch
// 2. "breadCrumb:updated" — BreadCrumbs component listens for manual updates
// 3. "message"            — App.js listens for postMessage from internal dashboard
// 4. "unhandledrejection" — AuthenticatedPage catches unhandled promise rejections
// 5. "popstate"           — WebsiteBuilder handles browser back/forward
// 6. "storage"            — Orders page listens for localStorage changes across tabs
// 7. "animationstart"     — Login form detects browser autofill via CSS animation trick
```

**Interview talking point:** "We use three real-time channels: WebSocket for live store events, Firebase Cloud Messaging for push notifications with 7-day token caching and exponential backoff retry, and a custom window event system for cross-component communication like store switching."

---

## 4. Advanced React Patterns Used

### Higher-Order Components (HOCs)
```javascript
// 1. addValidations — wraps ALL form inputs with error display
//    src/components/Form/Inputs/index.js
const Input = addValidations(RawInput);
// Adds: error styling, validation message, showErrors logic

// 2. withRouter — used on NavBar, Menu for location access
//    React Router v4 pattern
export default withRouter(NavBar);

// 3. makestoreDependentComponent — wraps components that need store data
//    src/containers/StoreSelector/index.js
// Adds: store loading, store filtering by page context
```

### Portal Pattern (4 usages)
```javascript
// Modals render outside the component tree:
// src/components/Popup/Modal/index.js
ReactDOM.createPortal(
  <div className="modal-backdrop">{children}</div>,
  document.getElementById('portal')  // <div id="portal"> in index.html
);

// Also used by: CustomizedPopup, StoreSelector, DeliveryArea ActionButtons
```

### Factory Pattern
```javascript
// Website builder layouts use factory functions:
// src/pages/settings/Themes/Layouts/Accordion.jsx
const EditLayoutWrp = () => {
  return {
    fields: (props) => <EditFaqLayout {...props} />,
  };
};
// The caller invokes: layout().fields(props)
// This decouples layout registration from rendering
```

### Imperative Handle (1 usage)
```javascript
// src/pages/operations/Orders/PackOrder/CreatePackage.js
const CreatePackage = forwardRef((props, ref) => {
  useImperativeHandle(ref, () => ({
    getPackageData: () => { ... }  // Parent can call child method directly
  }));
});
```

### Compound Component Pattern
```javascript
// Table uses composition:
<Table>
  <Header items={headers} />
  <Row><Cell>...</Cell><Cell>...</Cell></Row>
  <Row><Cell>...</Cell><Cell>...</Cell></Row>
</Table>
// Each sub-component is independently exported and composed by the consumer
```

### Context Providers (7 instances)
```javascript
// 1. Enterprise context — PublicPage wraps login/signup
//    src/containers/Context/index.js → Provider/Consumer for isEnterprise boolean

// 2. WastageContext — Returns page shares wastage reasons
//    src/pages/operations/Returns/index.js

// 3. TripIdContext — Trip details shares trip ID
//    src/pages/delivery/TripDetails/index.js

// 4. PaymentModeContext — Trips page shares payment modes
//    src/pages/delivery/Trips/index.js

// 5. VehicleContext — Vehicle planner shares vehicle data
//    src/pages/delivery/VehiclePlanner/Context/index.js

// 6. CommConfigContext — Orders table shares communication config
//    src/pages/operations/Orders/Table/index.js

// 7. Notification context (implicit) — via useNotificationManager hook
```

---

## 5. Drag-and-Drop Features

### Trip Planner (react-beautiful-dnd)
```
Location: src/pages/delivery/TripPlanner/
Purpose: Drag orders between delivery trips to optimize routes

Architecture:
- DragDropContext wraps the entire planner
- Droppable zones: Each trip container, unmapped orders, not-ready orders
- Draggable items: Individual order cards
- onDragEnd: Reorders within trip or moves between trips
- Visual: Shows trip route, order count, estimated time

Files:
- TripPlanner/index.js — DragDropContext wrapper
- TripContainer/index.js — Droppable trip zone
- TripContainer/OrderContainer/index.js — Draggable order card
- TripSequence/TripSequenceOrders/index.js — Nested DnD for order sequence
- UnmappedOrders/index.js — Droppable zone for unassigned orders
```

### Sortable Tree (react-sortable-tree)
```
Location: src/components/Tree/index.js
Purpose: Hierarchical category/menu management
Usage: Category tree in catalogue, potentially menu ordering
```

**Interview talking point:** "The Trip Planner uses react-beautiful-dnd with nested droppable zones — you can drag orders between trips or reorder within a trip. We also use react-sortable-tree for hierarchical category management."

---

## 6. File Upload & Media

### Upload Component
```
Location: src/components/Form/Inputs/Upload/
- Handles image and file uploads
- Shows upload progress
- Preview of uploaded images
- Integrates with media-service API
```

### Rich Text Editors
```
Three editors available:
1. React Quill (primary) — used in most content editing
   Location: src/components/Form/Inputs/RichTextEditorQuill/
2. Draft.js — available but less used
3. Jodit React — alternative editor

Used in: Campaign content, product descriptions, website builder text blocks
```

### PDF Generation
```javascript
// src/pages/operations/Orders/Details/OrderInvoice/index.js
import jsPDF from 'jspdf';
import html2canvas from 'html2canvas';
// Converts order invoice HTML to canvas, then to PDF for download
```

---

## 7. CI/CD Pipeline

### Two Deployment Pipelines

```
Pipeline 1: Development (Asia)
  Trigger: Push to `development` branch
  ┌─────────────────────────────────────────────┐
  │ Build Job (Ubuntu)                           │
  │ ├── Checkout code                            │
  │ ├── Detect modified files                    │
  │ ├── Setup Node 16.x + SSH                    │
  │ ├── npm install (skip Puppeteer)             │
  │ ├── npm run build                            │
  │ ├── Docker build (Dockerfile.new)            │
  │ └── Push to asia-south1 Artifact Registry    │
  └──────────────────┬──────────────────────────┘
                     ▼
  ┌─────────────────────────────────────────────┐
  │ Deploy Job (GKE)                             │
  │ ├── Auth to GCP                              │
  │ ├── kubectl set image (update deployment)    │
  │ ├── Generate ConfigMap from .stage.env       │
  │ └── Apply to zopping cluster (asia-south1)   │
  └─────────────────────────────────────────────┘

Pipeline 2: Production (US)
  Trigger: Push to `main` branch / tags
  ┌─────────────────────────────────────────────┐
  │ Build → Stage Deploy → Prod Deploy           │
  │ (uses reusable workflows from zs-workflows)  │
  │                                              │
  │ Registry: us-central1 Artifact Registry      │
  │ Cluster: zopsmart-products-cluster           │
  │ Stage namespace: zopping-stage               │
  │ Prod namespace: zopping                      │
  └─────────────────────────────────────────────┘
```

### Docker Setup
```dockerfile
# Both Dockerfiles:
# Base: static-server:1.1.0 (lightweight static file server)
# Just copies built React app into /static/admin/
ADD build /static/admin
ADD build /static/
# No Node.js runtime needed — just static files served by nginx/similar
```

### Release Management
```
- Semantic Release (.releaserc.json)
- Conventional commit enforcement (ci/validateMRTitle.js)
  Pattern: feat|fix|chore|docs|refactor|perf|test(scope)?: description
- Auto-generates CHANGELOG.md
- Sends email notification via SendGrid on release
- Current version: 5.0.1
```

**Interview talking point:** "We have a multi-region deployment strategy — development goes to Asia (asia-south1) via direct GitHub Actions, while production uses reusable workflows deploying to US (us-central1). Both use GKE with dynamic ConfigMap generation from env files. Docker images are just static file servers — no Node.js runtime in production."

---

## 8. Testing Strategy

### Current State (Be Honest)
```
Framework: CodeceptJS 2.6.5 with Puppeteer
Type: End-to-end (E2E) tests ONLY — no unit tests, no integration tests

Test suites:
- Operations: dashboard, orders, customers, picking, rack management
- Settings: basic info, notifications, social, themes, extensions
- Customer Support: dashboard
- HR: attendance, designation, employees
- Marketing, Catalogue, Logistics: test files present

Config: Headless Chrome, localhost:3000, 10s timeout
```

**Interview talking point (be honest):** "We have E2E tests using CodeceptJS with Puppeteer covering the main user flows. We don't have unit or integration tests — that's a known gap. If I were to improve this, I'd add React Testing Library for component tests and Jest for utility function tests, focusing on the most critical paths first: auth flow, order management, and the form system."

---

## 9. Google Maps Integration

```javascript
// Two map libraries available:
// 1. react-google-maps — used in delivery area config, store location
// 2. @react-google-maps/api — in zs-components for customer-facing maps

// Map key management:
// - Default key in config: GOOGLE_MAP_DEFAULT_KEY
// - Per-org key fetched from: GET /website-service/config
// - Stored in localStorage for reuse

// Used in:
// - Delivery area configuration (RadialMapComponent with shouldComponentUpdate)
// - Store location setup
// - Trip planner route visualization (MapWithPath component)
// - Customer address selection (StandaloneSearchBox, react-places-autocomplete)
// - Website builder map layouts (GoogleMap, MapContactForm from zs-components)

// Custom maps service: https://maps.zopping.com (ZOPSMART_MAPS_URL)
```

---

## 10. Internationalization (i18n) Details

```javascript
// src/lib/translator/index.js
// Uses MessageFormat library for complex translations

// 4 languages: English (en), Hindi (hi), Arabic (ar), Kannada (kn)

// Usage pattern across the app:
import { getMessage } from '../lib/translator';
getMessage("order.status.pending")           // → "Pending"
getMessage("items.count", { count: 5 })      // → "5 items" (pluralization)

// Language datasets are:
// 1. Bundled in src/lib/translator/dataset/ (en.js, hi.js, ar.js, kn.js)
// 2. Can be fetched from API via: npm run update-languages
//    (scripts/getLanguages.js downloads from internal API)

// Language selection: LanguageSelector component in NavBar
// Storage: localStorage["language"]
// On change: Calls setLocale() → saves to localStorage → page reload
// API integration: Accept-Language header sent with every request
```

---

## 11. Multi-Tenancy Architecture

```
Three levels of isolation:

1. ORGANIZATION LEVEL (tenant)
   - Each org has: id, name, domain, extensions[], industry, isEnterprise
   - Extensions act as feature flags per org
   - Separate billing, plans, and settings
   - Organization ID injected into relevant API calls

2. STORE LEVEL (within an org)
   - One org can have multiple stores (if MultiStoreSupport extension enabled)
   - Store selector in sidebar for switching
   - storeId sent with every API call
   - Store change triggers re-fetch via CustomEvent("storeChanged")
   - Store-level features: delivery hub, picking, click-collect

3. USER LEVEL (within an org)
   - Endpoint-level permissions per user
   - isOwner flag for full access
   - Menu filtering based on permissions
   - Some features owner-only (delete account, billing)
```

**Interview talking point:** "The app supports three levels of multi-tenancy: organization (tenant isolation with feature flags via extensions), store (multiple physical stores per org with independent data), and user (endpoint-level RBAC). The store switching is particularly interesting — it uses a CustomEvent broadcast pattern so every data-dependent component can independently re-fetch without a centralized state manager."

---

## 12. Payment Integration

```javascript
// src/lib/payment/ — 7 files for payment handling

// index.js — utility functions:
getPaymentLogo(type)    // Maps: VISA → visa.svg, MASTERCARD → mastercard.svg, AMEX → amex.svg
formatCardNumber(number) // Returns: "•••• •••• •••• 1234" (masks all but last 4)

// Payment processing handled server-side
// Frontend only handles:
// - Displaying payment method cards with brand logos
// - Masking card numbers for display
// - Wallet dialog (WalletDialog component)
// - Store credit management (credit-service)
```

---

## 13. PWA Capabilities

```
// public/manifest.json
{
  "name": "Zopping Dashboard",
  "short_name": "Zopping",
  "start_url": "./login",
  "display": "standalone"
}

// Service workers:
// 1. firebase-messaging-sw.js — Push notification handling in background
// 2. service-worker.js — CRA default (currently disabled in registerServiceWorker.js)

// PWA features active:
// ✅ Web app manifest (installable on mobile)
// ✅ Push notifications via FCM
// ✅ HTTPS enforcement
// ❌ Offline support (service worker disabled)
// ❌ Background sync
```

---

## 14. Browser Compatibility

```
// babel.config.js
presets: ['@babel/preset-env', '@babel/preset-react']
plugins: [
  '@babel/plugin-proposal-nullish-coalescing-operator',  // ??
  '@babel/plugin-proposal-optional-chaining',             // ?.
  'babel-plugin-transform-object-assign'                  // Object.assign polyfill
]

// IE9 polyfills imported in index.js
// Polyfills in src/lib/polyfills/:
// - Array.from, Object.assign, Promise, etc.

// Target: Modern evergreen browsers + IE9+ (legacy support)
// Node: Pinned to 16.14.2 (.nvmrc)
```

---

## 15. Storybook (Component Documentation)

```
// .storybook/config.js
// Runs on port 9009
// Loads stories from src/stories/

// Commands:
npm run storybook        // Development server
npm run build-storybook  // Static build

// Used for: Documenting and developing shared components in isolation
// Version: Storybook 5.3.19
```

---

## 16. Key Design Decisions to Discuss in Interviews

### Why These Choices Were Made (and Trade-offs)

| Decision | Why | Trade-off |
|----------|-----|-----------|
| No Redux | Admin pages are independent CRUD views | No centralized debugging, relies on localStorage |
| localStorage for state | Simple, persistent, works across tabs | XSS vulnerability, no reactivity (need CustomEvents) |
| Class components for forms | BaseForm pattern predates hooks | Can't use hooks inside, harder to compose |
| CSS variables over CSS-in-JS | Works with zs-components, no runtime cost | No dynamic theming per component, no type safety |
| CodeceptJS E2E only | Tests real user flows end-to-end | Slow, flaky, no unit test safety net |
| React 16 (not 18) | Stability, large codebase migration risk | No concurrent features, no automatic batching |
| CRA (not Vite/Next) | Stable, well-known, team familiarity | Slower builds, no SSR, harder to customize webpack |
| Axios 0.19 | Works, no breaking changes encountered | Missing security patches, no AbortController |
| Custom form system | 23 input types with unified validation | Learning curve, class-based, not ecosystem standard |
| Factory pattern for layouts | Decouples registration from rendering | Indirect, harder to trace data flow |

### What You'd Improve (Shows Growth Mindset)

1. **Add DOMPurify** for all dangerouslySetInnerHTML usage
2. **Migrate to React 18** for automatic batching and concurrent features
3. **Add React Testing Library** for component tests on critical paths
4. **Replace localStorage auth** with httpOnly cookie approach
5. **Upgrade Axios** to latest (security patches + AbortController)
6. **Migrate BaseForm to React Hook Form** — modern, smaller, hook-based
7. **Add TypeScript** to more files (currently mixed JS/TS)
8. **Replace CRA with Vite** for faster builds
9. **Add Zustand or Jotai** for shared state instead of localStorage + CustomEvents
10. **Add error monitoring for API failures** beyond just Bugsnag (structured logging)

---

## Quick Cheat Sheet — Numbers to Remember

| Metric | Value |
|--------|-------|
| React version | 16.12.0 |
| Feature domains | 21 pages |
| Shared components | 70+ |
| Backend microservices | 26 |
| Form input types | 23 |
| Languages supported | 4 (en, hi, ar, kn) |
| Settings sub-modules | 39 |
| Layout components (zs) | 120+ |
| Custom events | 7 |
| Context providers | 7 |
| Debounce usages | 14+ |
| E2E test suites | 9 |
| Docker regions | 2 (Asia, US) |
| Current version | 5.0.1 |
| Node version | 16.14.2 |

---

*Generated for interview preparation reference.*
