# Zopping Admin — Complete Data Flow Guide (Top to Bottom)

## The Big Picture

```
index.js (Bugsnag + Router)
  └── App.js (Session init, GTM, Faro, Notifications)
        └── router.js (Permission filtering, lazy loading)
              ├── PublicPage → Login/Signup (no auth needed)
              └── PrivateRoute (auth + permission check)
                    └── AuthenticatedPage (NavBar + Menu + Content)
                          └── Page Component (e.g., Orders)
                                └── ListingPage Container (CRUD orchestrator)
                                      ├── Table (renders rows from API data)
                                      ├── Form (user input → API submission)
                                      ├── Filters (query params → API refetch)
                                      └── Pagination (page change → API refetch)
```

**Data lives in 3 places:**
1. **localStorage** — Session, auth token, permissions, selected store, language
2. **Component state** — Each page/component manages its own data (no Redux)
3. **API responses** — Fetched on mount, on action, or on store change

---

## Phase 1: App Boot (`src/index.js`)

```javascript
// 1. Bugsnag error tracking initialized
const bugsnagClient = bugsnag({ apiKey: '0aeff65b611808adeed5b85d78193511' });

// 2. App wrapped in BrowserRouter
ReactDOM.render(
  <Router basename={process.env.PUBLIC_URL}>  {/* basename = "/admin/" */}
    {IS_STAGING ? <App /> : <ErrorBoundary><App /></ErrorBoundary>}
  </Router>,
  document.getElementById('root')
);
```

**Data created:** Bugsnag client (global error handler), Router context (provides `history`, `location`, `match` to all children)

---

## Phase 2: App Component (`src/containers/App/App.js`)

This is the **orchestrator**. On mount, it does 5 things in parallel:

### 2a. CSS Loading (`css.js`)
```javascript
import "@zopsmart/zs-components/dist/index.css";  // Design tokens from library
import "./tokens.css";                              // App overrides
```

### 2b. Session Loading (synchronous, from localStorage)
```javascript
const session = getSession();
// Returns: { organization, user, billingSettings, isPendingActionCompleted }
// All read from localStorage (saved during last login)

const orgId = session?.organization?.id;
const userId = session?.user?.id;
const userIsOwner = session?.user?.isOwner;
```

### 2c. GTM + Google Analytics (useEffect, runs once)
```javascript
// Production: GTM-MZBV747, GA: G-F0HVPCNDPD, Google Ads: AW-10877183916
// Staging:    GTM-NQ48S5L, GA: G-NM3GFN7ZG1
TagManager.initialize({ gtmId });
// Dynamically injects <script> tags into <head>
```

### 2d. Fresh Permissions Fetch (useEffect, runs once)
```javascript
// Only if user was previously logged in (localStorage has permissions)
const meApi = new API({ url: "/account-service/me" });
const orgApi = new API({ url: `/account-service/organization/${orgId}` });

// Fetch fresh data from backend
const [userResponse, orgResponse] = await Promise.all([meApi.get(), orgApi.get()]);

// Save updated session to localStorage
saveSession({
  user: userResponse.data.user,
  organization: orgResponse.data.organization
});
// saveSession() also calls savePermissions(user.endpointPermissions)
// which formats and stores permissions in localStorage
```

### 2e. Notification Setup (hooks)
```javascript
// Toast notifications (in-app)
const { notifications, addNotification, removeNotification } = useNotificationManager();
// → notifications = [] (React state array)
// → addNotification(obj) appends to array with unique ID
// → removeNotification(id) filters from array

// Push notifications (Firebase)
const { requestPermission } = usePushNotifications(userId, (notification) => {
  addNotification(notification);  // Bridge: push notification → toast
  if (notification?.data?.type === "owner/placeOrder") {
    playNotificationSound();  // Audio alert for new orders
  }
});
// → Checks browser support
// → Gets FCM token (cached 7 days in localStorage)
// → Registers device with backend: POST /account-service/device
// → Sets up foreground message handler via Firebase onMessage()
```

### 2f. Faro Initialization (lazy loaded)
```javascript
const { initializeFaro } = await import('./initialize');
initializeFaro();
// → OpenTelemetry instrumentation for:
//   - Document load timing
//   - Fetch/XHR request tracing
//   - User interaction tracking
//   - Console capture
//   - Error capture
// → Sends to Grafana: https://grafana-agent.observability-prod.gcp.zopsmart.com/collect
```

### 2g. Render
```javascript
return (
  <NotificationContainer notifications={notifications} duration={20000} position="top-right" />
  <FaroErrorBoundary>
    <ErrorBoundary>
      <Router isLoggedIn={isLoggedIn()} />
    </ErrorBoundary>
  </FaroErrorBoundary>
);
```

**Data passed down:** `isLoggedIn()` boolean → Router

---

## Phase 3: Router (`src/containers/App/router.js`)

### 3a. Permission-Based Menu Building

The router defines ALL possible routes with their permission requirements:

```javascript
const requiredPermissions = [
  {
    slug: "orders",
    label: "Orders",
    href: "orders/orders",
    endpoints: ["order-service/order"],  // API endpoint user needs access to
    extensions: [],                       // Feature flags that must be enabled
    icon: OrdersIcon,
  },
  {
    slug: "catalogue",
    label: "Products",
    href: "catalogue/products",
    endpoints: ["catalogue-service/product"],
    extensions: [],
  },
  {
    slug: "abandonedCart",
    label: "Abandoned Carts",
    href: "abandoned-carts",
    endpoints: ["website-service/abandoned-user-cart"],
    extensions: ["AbandonedCart"],  // Only shown if this extension is enabled
  },
  // ... 50+ more route definitions
];
```

### 3b. Menu Filtering

```javascript
// For each route, check if user has access:
const menu = requiredPermissions.filter(item => {
  return hasAccess({
    endpoints: item.endpoints,   // Check localStorage["formattedPermissions"]
    extensions: item.extensions   // Check organization.extension[] array
  });
});
```

**`hasAccess()` does two checks:**
1. **Endpoint check:** Does `formattedPermissions["order-service/order"]` exist and have GET?
2. **Extension check:** Does `organization.extension[]` contain slug "AbandonedCart"?

### 3c. Route Rendering

```javascript
// Public routes (no auth needed)
<Route path="/login" render={() => <PublicPage><Login /></PublicPage>} />
<Route path="/signUp" render={() => <PublicPage><SignUp /></PublicPage>} />

// Private routes (auth + permissions required)
<PrivateRoute
  path="/orders"
  component={Operations}           // Lazy loaded
  isLoggedIn={isLoggedIn}
  menu={filteredMenu}              // Only items user can access
  requiredPermissions={requiredPermissions}
  isLazy={true}
/>
```

**Data passed down:**
- `isLoggedIn` → PrivateRoute
- `menu` (filtered array) → PrivateRoute → AuthenticatedPage → Menu + NavBar
- `requiredPermissions` → PrivateRoute (for page-level access check)

---

## Phase 4: PrivateRoute (`src/containers/PrivateRoute/index.jsx`)

This is the **gate keeper**. It decides: render the page, redirect to login, or show "Access Denied".

### 4a. Authentication Check
```javascript
// Three scenarios:
if (hasGuidParam) {
  // Internal dashboard redirect — fetch fresh user/org data via API
  const userData = await getUserDetails();          // GET /account-service/me
  const orgData = await getOrganizationDetails();   // GET /account-service/organization/{id}
  saveSession({ user: userData, organization: orgData });
}
else if (redirectedFromApp) {
  // App redirect with guid — same as above
}
else {
  // Normal flow — use existing localStorage session
}
```

### 4b. Authorization Check
```javascript
const redirectToLogin = !isLoggedIn || !hasPermissionsInfo;

if (redirectToLogin) {
  return <Redirect to="/login" />;  // Not authenticated
}

const pageAccess = hasAccess({
  endpoints: pagePermissions.endpoints,
  extensions: pagePermissions.extensions
});

if (!pageAccess) {
  return <MissingPage />;  // Authenticated but no permission (403)
}

// Access granted — render the page
if (isLazy) {
  return <LazyComponentLoader component={Component} menu={menu} {...props} />;
} else {
  return <Component menu={menu} {...props} />;
}
```

**Data passed to page component:**
- `menu` — filtered menu array
- `isLoggedIn` — boolean
- `rerender` — callback for forcing re-render
- `hasIndustry` — from organization data
- All React Router props (`match`, `location`, `history`)

---

## Phase 5: AuthenticatedPage (`src/containers/AuthenticatedPage/index.js`)

This is the **layout shell** — NavBar on top, Menu on left, content in center.

### 5a. Store Loading (componentDidMount)
```javascript
if (isExtensionEnabled("MultiStoreSupport")) {
  // Fetch stores from API
  const storesApi = new API({ url: "/account-service/store" });
  const response = await storesApi.get();
  const stores = response.data.store;
  setStores(stores);  // Save to localStorage["stores"]

  // Set current store
  const savedStoreId = get("store");  // From localStorage
  if (savedStoreId) {
    this.setState({ storeId: savedStoreId, stores });
  } else {
    set("store", stores[0].id);  // Default to first store
    this.setState({ storeId: stores[0].id, stores });
  }
} else {
  // Single store — use organization's default
  const org = JSON.parse(get("organization"));
  set("store", org.defaultStore.id);
}
```

### 5b. Balance Fetch
```javascript
if (hasPermissions("billing-service", "balance", "GET")) {
  const balanceApi = new API({ url: "/billing-service/balance" });
  const response = await balanceApi.get();
  this.setState({ balance: response.data.balance });
}
```

### 5c. Layout Rendering
```javascript
render() {
  const menuProps = {
    items: this.props.menu,           // Filtered menu from router
    stores: this.state.stores,        // Store list (if multi-store)
    storeId: this.state.storeId,      // Currently selected store
    showLanguageDropDown: true,
  };

  return (
    <div className="app-layout">
      <NavBar
        menuProps={menuProps}
        balance={this.state.balance}
      />
      <Menu
        {...menuProps}
        onChange={(storeId) => this.updateStore(storeId)}  // Store change handler
      />
      <main>
        <ErrorBoundary>
          {this.props.children}  {/* ← The actual page content */}
        </ErrorBoundary>
      </main>
    </div>
  );
}
```

### 5d. Store Change Flow
```javascript
updateStore(storeId) {
  set("store", storeId);  // Persist to localStorage
  this.setState({ storeId });

  // Dispatch custom event — ALL listening components will re-fetch data
  window.dispatchEvent(new CustomEvent("storeChanged", { detail: { stores: storeId } }));
}
```

**Data passed to children:**
- NavBar gets: `menuProps`, `balance`
- Menu gets: `menuProps`, `onChange` callback
- Page content gets: whatever the Route passed (menu, match, location, history)

---

## Phase 6: NavBar & Menu Components

### NavBar (`src/components/NavBar/index.js`)
```
DATA IN:
  - menuProps.items (for mobile menu)
  - balance (account balance number)
  - getSession().organization.domain (for "View Store" link)

RENDERS:
  - Hamburger menu toggle (mobile)
  - "View Store" link → https://{org.domain}
  - Language selector dropdown
  - In-app notification bell
  - User actions dropdown (profile, change password, logout)
```

### Menu (`src/components/Menu/index.js`)
```
DATA IN:
  - items: Array of { href, label, slug, icon, children? }
  - stores: Array of store objects (if multi-store)
  - storeId: Currently selected store ID
  - onChange: Callback for store change
  - location.pathname: Current URL (via withRouter HOC)

PROCESSING:
  - Matches current URL against menu items → determines active item
  - Separates items into: withChildren (expandable), withoutChildren (flat), myAccount
  - Filters by isOwner (some items hidden for non-owners)

RENDERS:
  - Store selector dropdown (if multi-store)
  - Menu items with active highlighting
  - Expandable parent menus (Marketing, Operations, Settings, etc.)
```

---

## Phase 7: Page Components — The ListingPage Pattern

This is where **most of the app's data flow** happens. ~50+ pages use this pattern.

### 7a. Page Configuration (e.g., Orders)

```javascript
// src/pages/operations/Orders/Table/index.js
class OrdersContainer extends Component {
  render() {
    return (
      <ListingPage
        menu={this.props.menu}
        className="orders-page"

        // API Configuration
        api={{
          url: "/order-service/order",
          params: { storeId: get("store"), status: "PENDING" },
          headers: {},
          transform: (response) => response.data.order,  // Extract array from response
        }}

        // Table Configuration
        primaryKey="referenceNumber"
        tableProperties={{
          headers: ["Order #", "Customer", "Items", "Amount", "Status", "Actions"],
          row: OrderRow,  // Component that renders each row
        }}

        // Form Configuration
        form={{
          component: OrderForm,
          transformSubmit: (data) => ({
            ...data,
            customerId: data.customer?.id,  // Transform before API call
          }),
        }}

        // Filter Configuration
        filters={{
          component: OrderFilters,
          transformSubmit: (data) => ({
            ...data,
            slotStartTime: JSON.parse(data.slot)?.startTime,
          }),
        }}

        storeDependent={true}  // Re-fetch when store changes
        tableDynamic
      />
    );
  }
}
```

### 7b. ListingPage Initialization (`src/containers/ListingPage/index.js`)

```javascript
// STATE STRUCTURE:
{
  data: {
    items: null,        // Array of table rows (from API)
    paging: null,       // { count: 100, limit: 10, offset: 0 }
    filters: {},        // Currently applied filters
    viewItems: null,    // Transformed items for display
  },
  apiParams: {},        // Current query parameters
  loaders: { data: false },
  form: {
    shown: false,       // Is add/edit modal open?
    rowIndex: -1,       // Which row being edited (-1 = new)
    data: null,         // Initial form values
  }
}

// ON MOUNT:
componentDidMount() {
  this.api = new API({ url: this.props.api.url });

  // Build initial params
  const apiParams = {
    ...this.props.api.params,
    storeId: get("store"),
  };
  this.setState({ apiParams });

  // Fetch data
  this.fetchTableData(apiParams);

  // Listen for store changes
  if (this.props.storeDependent) {
    window.addEventListener("storeChanged", this.handleStoreChange);
  }
}
```

### 7c. Data Fetching Flow

```
fetchTableData(params)
    │
    ├── Show loading spinner
    │   setState({ loaders: { data: true } })
    │
    ├── API Call
    │   this.api.get(params, headers)
    │   → Axios GET to https://staging.zopping.com/api/order-service/order?storeId=1&status=PENDING&page=1
    │
    ├── Inside API class (src/lib/api/index.js):
    │   ├── ServiceHostMapping: "order-service" → GO_HOST
    │   ├── Add Authorization: "Bearer {localStorage.token}"
    │   ├── Add Accept-Language: "en" (from localStorage)
    │   ├── Add X-API-VERSION: 2 (for specific services)
    │   ├── Handle storeId=-1 (ALL STORES) conversion
    │   └── Return response.data on success
    │
    ├── Transform Response
    │   props.api.transform(response)
    │   → response.data.order → [order1, order2, ..., order10]
    │
    ├── Update State
    │   setState({
    │     data: {
    │       items: [order1, order2, ...],
    │       paging: { count: 100, limit: 10, offset: 0 }
    │     }
    │   })
    │
    └── Hide loading spinner
        setState({ loaders: { data: false } })
```

### 7d. Table Rendering

```javascript
// ListingPage render():
<Table tableDynamic={true}>
  <Header items={["Order #", "Customer", "Items", "Amount", "Status"]} />
  {this.state.data.items.map((row, index) => (
    <OrderRow
      key={row.referenceNumber}
      {...row}                          // SPREAD: all API fields become props
      onAction={this.performAction}     // Callback to trigger CRUD actions
      apiParams={this.state.apiParams}  // For context-aware rendering
      index={index}
    />
  ))}
</Table>
<Pagination
  count={paging.count}
  limit={paging.limit}
  offset={paging.offset}
  onChange={(page) => this.fetchTableData({ page })}
/>
```

### 7e. Row Component Receives Data

```javascript
// OrderRow receives ALL fields from API response as individual props:
class OrderRow extends Component {
  render() {
    const {
      referenceNumber,    // "ORD-001" (from API)
      customer,           // { name, email, phones } (from API)
      items,              // [{ sku, quantity }] (from API)
      status,             // "PENDING" (from API)
      invoiceAmount,      // 150 (from API)
      onAction,           // callback (from ListingPage)
    } = this.props;

    return (
      <Row>
        <Cell><Link to={`/orders/details/${referenceNumber}`}>{referenceNumber}</Link></Cell>
        <Cell>{customer.name}</Cell>
        <Cell>{invoiceAmount}</Cell>
        <Cell>
          <DropDown>
            <DropDownItem onClick={() => onAction("UPDATE", { referenceNumber }, { status: "COMPLETED" })}>
              Mark Complete
            </DropDownItem>
          </DropDown>
        </Cell>
      </Row>
    );
  }
}
```

---

## Phase 8: Form Data Flow (User Input → API)

### 8a. Opening a Form

```javascript
// User clicks "Add" or "Edit" button in table row
onAction("ADD")  or  onAction("EDIT", rowData)
    ↓
ListingPage.performAction()
    ↓
// ADD: setState({ form: { shown: true, rowIndex: -1, data: null } })
// EDIT: setState({ form: { shown: true, rowIndex: 3, data: items[3] } })
    ↓
// Modal opens with Form component
<Modal>
  <FormComponent
    value={this.state.form.data}        // null for new, row data for edit
    onSubmit={this.createResource}       // or this.modifyResource
    method={form.rowIndex === -1 ? "add" : "edit"}
  />
</Modal>
```

### 8b. BaseForm Initialization (`src/components/Form/index.js`)

```javascript
class BaseForm extends React.Component {
  constructor(props) {
    this.state = {
      values: cloneMutables(props.value) || {},  // Deep clone of initial data
      touched: {},         // Which fields user has interacted with
      blurred: {},         // Which fields have lost focus
      validations: {},     // Validation errors per field
      submitting: false,
      pressedSubmitWithCurrentData: false,
    };
  }
}
```

### 8c. Input Binding — `generateStateMappers()`

This is the **core pattern** that connects every form input to state:

```javascript
// In a form component:
<Input
  label="Stock Buffer"
  name="stockBuffer"
  type="number"
  {...this.generateStateMappers({
    stateKeys: ["stockBuffer"],       // Path in state.values
    loseEmphasisOnFill: true,
  })}
/>

// generateStateMappers() returns:
{
  value: this.getState(["stockBuffer"]),      // Read: state.values.stockBuffer
  onChange: (newValue) => {
    this.updateState(["stockBuffer"], newValue);  // Write: state.values.stockBuffer = newValue
    this.setState({ pressedSubmitWithCurrentData: false });
  },
  onBlur: () => {
    this.updateState(["stockBuffer"], value.trim());  // Auto-trim strings
    updateStateRecursively(["blurred", "stockBuffer"], true);
  },
  onValidation: (error) => {
    this.registerValidation(["stockBuffer"], error);  // Store HTML5 validation result
  },
  showErrors: /* depends on validation scenario (ONCHANGE, ONBLUR, ONSUBMIT) */
}
```

### 8d. Nested State Example

For deeply nested data:
```javascript
<Input
  {...this.generateStateMappers({
    stateKeys: ["variants", 0, "price"],  // state.values.variants[0].price
  })}
/>

// getState(["variants", 0, "price"]):
//   → state.values → .variants → [0] → .price → 29.99

// updateState(["variants", 0, "price"], 39.99):
//   → Traverses path, creates missing objects/arrays
//   → Sets state.values.variants[0].price = 39.99
//   → Uses updateStateRecursively() from lib/stateManagement
```

### 8e. Input Component Internals

```javascript
// src/components/Form/Inputs/Input.js
class Input extends React.Component {
  handleChange(e) {
    let value = this.props.type === "number" ? Number(e.target.value) : e.target.value;
    this.props.onChange(value);          // → BaseForm.updateState()
    this.runValidation(e.target);       // → HTML5 checkValidity()
  }

  runValidation(input) {
    const validity = input.validity;    // HTML5 ValidityState
    this.props.onValidation({
      valid: validity.valid,
      valueMissing: validity.valueMissing,
      typeMismatch: validity.typeMismatch,
      patternMismatch: validity.patternMismatch,
      tooShort: validity.tooShort,
      tooLong: validity.tooLong,
    });
    // → BaseForm.registerValidation(stateKeys, error)
    // → Stored in state.validations.stockBuffer = { valid: true, ... }
  }
}
```

### 8f. Validation Wrapper HOC

```javascript
// src/components/Form/Inputs/index.js — addValidations HOC
// Wraps EVERY input component:
const Input = addValidations(RawInput);

// addValidations adds:
// - Error message display below input
// - Red border styling when invalid (class "input-error")
// - showErrors logic based on validation scenario:
//   ALWAYS    → always show
//   ONCHANGE  → show after user types (touched[field] === true)
//   ONBLUR    → show after focus leaves (blurred[field] === true)
//   ONSUBMIT  → show only after submit button pressed
```

### 8g. Form Submission Chain

```
User clicks "Submit" button
    │
    ├── _submitHandler(e)
    │   ├── e.preventDefault()
    │   ├── this.beforeSubmit()  // Optional hook (e.g., scroll to first error)
    │   ├── setState({ pressedSubmitWithCurrentData: true })  // Show ALL errors
    │   │
    │   ├── this.isFormValid()
    │   │   └── Recursively checks state.validations
    │   │       └── Every field must have { valid: true }
    │   │       └── Returns: true or false
    │   │
    │   ├── IF INVALID: stops here, errors now visible
    │   │
    │   └── IF VALID:
    │       └── this.props.onSubmit(cloneMutables(this.state.values))
    │           // Deep clone prevents mutation of form state
    │
    ├── ListingPage receives form data
    │   ├── createResource(formData) — for new items
    │   └── modifyResource(formData) — for edits
    │
    ├── Transform
    │   const params = this._transformSubmit(formData);
    │   // Runs props.form.transformSubmit hook:
    │   // - Type conversion: String → Number
    │   // - Extract IDs from objects: customer → customerId
    │   // - Parse JSON strings: slot → slotStartTime, slotEndTime
    │   // - Remove empty/null fields
    │
    ├── API Call
    │   // CREATE: api.post(params)  → POST /order-service/order
    │   // EDIT:   api.put(params)   → PUT /order-service/order/{id}
    │
    ├── Success Response
    │   ├── _transformResponse(response)  // Extract new/updated item
    │   ├── Update table state:
    │   │   // CREATE: items.unshift(newItem), paging.count++
    │   │   // EDIT:   items.splice(index, 1, updatedItem)
    │   ├── _hideForm()  // Close modal
    │   └── afterSubmit() hook if defined
    │
    └── Error Response
        ├── throwError(error)
        ├── Show error dialog with error.message
        └── Form stays open for user to fix
```

---

## Phase 9: Filter Data Flow

```
User opens filter panel
    │
    ├── Filter component (extends BaseForm) renders inputs
    │   └── Same generateStateMappers() pattern as regular forms
    │
    ├── User fills filters and clicks "Apply"
    │   └── form.onSubmit(filterValues)
    │
    ├── ListingPage.applyFilters(filterValues)
    │   ├── transformSubmit hook processes filters:
    │   │   // e.g., customer object → customerId
    │   │   // e.g., date range → startDate, endDate
    │   │
    │   ├── Merge with existing apiParams:
    │   │   params = { ...apiParams, ...transformedFilters, page: 1 }
    │   │
    │   └── this.fetchTableData(params)
    │       └── Same fetch flow as initial load
    │
    └── Table re-renders with filtered data
```

---

## Phase 10: Store Change — Cross-Component Communication

This is the **only global event** in the system:

```
User selects different store in Menu dropdown
    │
    ├── Menu.onChange(newStoreId)
    │   └── AuthenticatedPage.updateStore(newStoreId)
    │       ├── set("store", newStoreId)  // localStorage
    │       ├── setState({ storeId: newStoreId })
    │       └── window.dispatchEvent(new CustomEvent("storeChanged"))
    │
    ├── EVERY storeDependent component listens:
    │   window.addEventListener("storeChanged", handler)
    │
    ├── ListingPage.handleStoreChange()
    │   ├── Read new storeId from localStorage
    │   ├── Update apiParams with new storeId
    │   └── fetchTableData(newParams)  // Re-fetch all data
    │
    ├── StoreSelector.handleStoreChange()
    │   └── Update selected store display
    │
    └── Any custom components listening
        └── Re-fetch their own data
```

---

## Phase 11: Notification Data Flow

```
PUSH NOTIFICATION ARRIVES (from Firebase Cloud Messaging)
    │
    ├── Service Worker receives (background)
    │   └── Shows native browser notification
    │
    ├── Firebase onMessage() fires (foreground)
    │   └── FirebaseService.setupForegroundHandler(callback)
    │       └── Transforms payload:
    │           { notification: { title, body }, data: { id, type } }
    │           → { id: "order-123", title, body, type, timestamp }
    │
    ├── usePushNotifications callback fires
    │   └── Calls addNotification(transformedNotification)
    │
    ├── useNotificationManager.addNotification()
    │   └── Appends to notifications[] state array
    │
    ├── NotificationContainer re-renders
    │   └── Shows toast in top-right corner
    │       - Auto-dismisses after 20 seconds
    │       - Progress bar shows time remaining
    │       - Types: info, success, warning, error (color-coded)
    │
    └── If type === "owner/placeOrder":
        └── playNotificationSound() via AudioContext
```

---

## Phase 12: localStorage — The Central Data Store

Since there's no Redux, **localStorage is the shared state**:

| Key | Written By | Read By | Content |
|-----|-----------|---------|---------|
| `token` | Login API response | API class (every request), isLoggedIn() | JWT string |
| `user` | saveSession() | getSession(), NavBar, Menu | `{ id, name, isOwner, verified }` |
| `organization` | saveSession() | getSession(), Router, AuthenticatedPage | `{ id, name, domain, extension[], industry, isEnterprise }` |
| `permissions` | savePermissions() | getPermissions() | `{ "order-service/order": { allowedMethods: ["GET","POST"] } }` |
| `formattedPermissions` | savePermissions() | hasPermissions(), hasAccess() | `{ "order-service/order": ["GET","POST"] }` |
| `store` | AuthenticatedPage, Menu | API class, ListingPage, StoreSelector | Store ID (number) |
| `stores` | AuthenticatedPage | StoreSelector, Menu | `[{ id, name, hasDeliveryHub }, ...]` |
| `billingSettings` | saveSession() | Pricing pages | Plan info |
| `language` | LanguageSelector | translator, API headers | "en", "hi", "ar", "kn" |
| `fcmToken` | FirebaseService | Push notification hook | FCM token string |
| `fcm_token_timestamp` | FirebaseService | Token cache validation | Timestamp (7-day TTL) |

---

## Phase 13: API Class Internals (`src/lib/api/index.js`)

Every API call goes through this class:

```
new API({ url: "/order-service/order" })
    │
    ├── Constructor:
    │   ├── ServiceHostMapping["order-service"] = GO_HOST
    │   │   // GO_HOST = window.location.origin + "/api"
    │   │   // OR "https://staging.zopping.com/api" (on localhost)
    │   ├── this.url = "https://staging.zopping.com/api/order-service/order"
    │   └── this.signal = axios.CancelToken.source()  // For request cancellation
    │
    ├── .get(params, headers):
    │   └── call(this.url, "GET", params, this.signal, headers)
    │
    └── call() function:
        ├── 1. Organization ID injection (for internal dashboard)
        ├── 2. GUID parameter handling
        ├── 3. Build query string (GET) or JSON body (POST/PUT)
        ├── 4. Set headers:
        │   ├── Content-Type: "application/json"
        │   ├── Authorization: "Bearer {token}"
        │   ├── Accept-Language: "en"
        │   ├── X-API-VERSION: 2 (for specific services)
        │   └── Custom headers from caller
        ├── 5. Handle storeId=-1 (ALL STORES):
        │   └── If not owner: convert to comma-separated store IDs
        ├── 6. axios request with validateStatus
        ├── 7. On success: return response.data
        └── 8. On error: return Promise.reject({
                code: error.response?.status,
                message: error.response?.data?.message,
                url: api,
                method: method
              })
```

---

## Phase 14: Error Handling Chain

```
Error occurs anywhere in the app
    │
    ├── Component-level try/catch
    │   └── ListingPage.throwError() → shows error dialog
    │
    ├── React Error Boundary (src/components/ErrorBoundary/index.js)
    │   ├── Catches render errors
    │   ├── Filters out: 401, 403, cancelled requests
    │   ├── Chunk loading errors: auto-retry 3 times, then hard reload
    │   ├── Reports to Bugsnag with metadata:
    │   │   { organization, user, store, gitHash }
    │   └── Reports to Faro analytics
    │
    ├── Faro Error Boundary (wraps everything)
    │   └── Sends to Grafana observability
    │
    └── Bugsnag (top-level, in index.js)
        └── Catches uncaught exceptions globally
```

---

## Summary: The 5 Data Flow Patterns

### Pattern 1: Top-Down Props
```
Router → PrivateRoute → AuthenticatedPage → Page Component
  (menu)    (menu)         (menu → NavBar, Menu)    (menu)
```

### Pattern 2: localStorage as Shared State
```
Login saves token → API reads token for every request
saveSession() writes → getSession() reads anywhere
set("store") writes → get("store") reads in API calls
```

### Pattern 3: ListingPage CRUD Cycle
```
Mount → fetchTableData() → API.get() → transform → setState → render Table
Action → Form opens → user edits → submit → API.post/put → update state → re-render
```

### Pattern 4: CustomEvent for Store Changes
```
Menu.onChange → window.dispatchEvent("storeChanged") → all listeners re-fetch
```

### Pattern 5: Firebase Push → Toast Notification
```
Firebase onMessage → usePushNotifications callback → addNotification → NotificationContainer renders
```

---

*Generated for interview preparation reference.*
