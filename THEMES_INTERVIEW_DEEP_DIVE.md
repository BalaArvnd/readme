# Zopping Themes — Complete Data Flow & Architecture Deep Dive

> **For**: SDE-2 Interview Preparation
> **Project**: Multi-tenant, multi-theme e-commerce storefront
> **Stack**: Next.js 14.2.0 | React 18 | styled-components | Axios | React Context
> **Version**: 3.2.5 | **Themes**: 34+ | **Components**: 222+

---

## TABLE OF CONTENTS

1. [Project Overview](#1-project-overview)
2. [Request Lifecycle — Server to Screen](#2-request-lifecycle)
3. [Provider Tree — How State is Shared](#3-provider-tree)
4. [What Each Provider Does](#4-what-each-provider-does)
5. [How a Page Renders](#5-how-a-page-renders)
6. [Theme System Architecture](#6-theme-system-architecture)
7. [API Layer — Repository Pattern](#7-api-layer)
8. [Complete User Journeys](#8-complete-user-journeys)
9. [Cross-Cutting Concerns](#9-cross-cutting-concerns)
10. [Directory Structure](#10-directory-structure)
11. [Tech Stack](#11-tech-stack)
12. [Design Patterns](#12-design-patterns)
13. [Interview Q&A](#13-interview-qa)

---

## 1. PROJECT OVERVIEW

This is a **multi-tenant, multi-theme e-commerce storefront**. A single Next.js codebase serves **34+ distinct themes**. Each merchant on the Zopping platform gets a theme, and this app dynamically renders the correct one based on the domain in the request.

**Key mental model**: Think of it as a **Theme Engine** — one Next.js app, many visual skins, shared business logic. Similar to Shopify themes but fully custom-built.

### Architecture Layers

```
┌─────────────────────────────────────────────────┐
│                   PAGES LAYER                    │
│  (Next.js file-based routing, entry points)      │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│               CONTAINERS LAYER                   │
│  (Business logic, data orchestration, HOCs)      │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│               PROVIDERS LAYER                    │
│  (React Context - global state management)       │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│              REPOSITORIES LAYER                  │
│  (API abstraction - axios calls to backend)      │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│                THEMES LAYER                      │
│  (Component Factory + theme-specific UI)         │
└─────────────────────────────────────────────────┘
```

---

## 2. REQUEST LIFECYCLE

### Step 1 — server.js (Custom Node.js HTTP Server)

When a user types `https://fashionstore.com` in their browser:

```
Browser Request: GET https://fashionstore.com/
         ↓
server.js receives it via createServer()
         ↓
Extracts domain from req.headers.host → "fashionstore.com"
Extracts protocol from X-Forwarded-Proto → "https"
         ↓
Injects into query: query.domain = "https://fashionstore.com"
         ↓
Passes to Next.js: handle(req, res, parsedUrl)
```

**Why this matters**: The domain IS the tenant identifier. Every API call goes to `${domain}/api/...`. Different domains = different merchants = different themes, products, configs.

In **dev mode**, it reads `process.env.DOMAIN_URL` instead.

### Step 2 — _app.jsx → getInitialProps() (Server-Side)

Runs on the **server** before any HTML is sent:

```
getInitialProps(appContext)
         ↓
verifyPagePermission(ctx, Component)  → Can this page be accessed?
         ↓
configureAxios(domain, ctx)  → Sets axios baseURL to domain/api
         ↓
Reads cookies: AUTH_TOKEN_KEY, CHECKOUT_KEY
         ↓
Calls Component.getInitialProps() if page has one
         ↓
Returns: { pageProps, domain, authToken, checkoutAddress, dir, pathname }
```

### Step 3 — _app.jsx → componentDidMount() (Client-Side)

Once the browser receives HTML and React hydrates:

```
componentDidMount()
         ↓
Sets apiLoader: true (shows spinner)
         ↓
getDetailsClient() makes 6 PARALLEL API calls:
  ┌─ GET /organization      → org config, theme name, colors, features
  ├─ GET /nav               → header navigation links
  ├─ GET /footerLinks       → footer links
  ├─ GET /navigation        → full navigation tree
  ├─ GET /layout/home       → home page layout descriptor (CMS data)
  └─ GET /announcement      → banner announcements

  + GET /me                 → user profile (if authToken exists)
         ↓
All responses stored in state:
  organizationData, navigationData, footerData, homeData,
  accountData, dictionaryList, websiteTheme, style, dir
         ↓
Router event listeners attached:
  routeChangeStart → loading: true
  routeChangeComplete → loading: false, scroll to top
         ↓
Tawk.to chat widget injected (if configured)
         ↓
Faro observability initialized (Grafana + OpenTelemetry)
```

**Interview point**: "We make 6 parallel API calls on mount to avoid waterfall loading. Instead of sequential calls taking 6x latency, we get all data in ~1x latency."

---

## 3. PROVIDER TREE

The render method wraps the entire app in **13 nested Context providers**. The order matters because inner providers depend on outer ones:

```
<ThemeProvider theme={themeColors}>              ← styled-components tokens
  <ErrorBoundary>                                ← Bugsnag error catching
    <DomainProvider domain={domain}>             ← API base URL
      <AccountProvider accountData={...}>        ← Auth state
        <GTMWrapper>                             ← Google Tag Manager
          <CommonDataProvider                    ← Org config, nav, layouts
            org, nav, home, footer>
            <AddressProvider address={...}>      ← Delivery address, storeId
              <AppModeProvider>                  ← WEBSITE vs ENDLESS_AISLE
                <OrderTypeProvider>              ← DELIVERY vs PICKUP
                  <CartProvider>                 ← Cart items, totals
                    <LanguageProvider>            ← Translations, RTL
                      <WishlistProvider>         ← Saved items
                        <SearchProvider>         ← Search query
                          <DeliveryLocationProvider>
                            <LocationProvider>
                              <ThemeNameProvider>

                                <Header />
                                <Component />    ← The page
                                <Footer />

                              </ThemeNameProvider>
                            </LocationProvider>
                          </DeliveryLocationProvider>
                        </SearchProvider>
                      </WishlistProvider>
                    </LanguageProvider>
                  </CartProvider>
                </OrderTypeProvider>
              </AppModeProvider>
            </AddressProvider>
          </CommonDataProvider>
        </GTMWrapper>
      </AccountProvider>
    </DomainProvider>
  </ErrorBoundary>
</ThemeProvider>
```

### Why This Order?

```
CartProvider DEPENDS ON:
  ├── AccountContext (needs accessToken for API calls)
  ├── AddressContext (needs storeId to fetch cart)
  ├── CommonDataContext (needs organizationData for config)
  ├── OrderTypeContext (needs DELIVERY/PICKUP)
  └── AppModeContext (needs endless aisle flag)

AddressProvider DEPENDS ON:
  ├── AccountContext (clears address on logout)
  └── CommonDataContext (updates home on store change)

WishlistProvider DEPENDS ON:
  ├── AccountContext (needs login status)
  └── AddressContext (needs storeId)
```

---

## 4. WHAT EACH PROVIDER DOES

### AccountProvider — The Auth Brain

```
STATE:
  userData: { name, email, phone, accessToken }
  isLoggedin: boolean
  isVerified: boolean
  showLoginForm: boolean

METHODS:
  login(data)   → POST /login → sets token in localStorage + cookies + axios headers
  logout()      → clears token from everywhere, resets axios headers
  signup(data)  → POST /register → auto-login on success

STORAGE:
  localStorage: "userData" (JSON)
  Cookie: "auth_token" (30-day expiry via nookies)
  Axios: Authorization = "Bearer ${token}" on all requests
```

### CommonDataProvider — The Config Hub

```
STATE:
  organizationData: { name, config, theme, colors, features }
  navigationData: formatted nav links for header
  completeNavData: full nav tree (for category pages)
  homeLayoutData: array of layout descriptors for home page
  footerData, announcementData

METHODS:
  updateHomeResponse(storeId, orderType) → refetches /layout/home and /nav
    ↑ Called when user changes delivery address (different store = different products)
```

### CartProvider — The Most Complex (691 lines)

```
STATE:
  cart: { items: [...], youPay, savings, amounts, offers }
  loading, error, showCartSlider, isSideCartVisible

METHODS:
  updateProductQuantity(item, parentContainer)
    → POST /cart with merged items → updates cart state

  getProductCount(uniqueKey, productId, sellerId)
    → Returns quantity of specific item (used by Add to Cart buttons)

  clearCart() → DELETE /cart → clears localStorage → resets state

  updateCartUsingGetCart() → GET /cart (syncs with server)

SIDE EFFECTS:
  - On login: syncs guest cart (localStorage) with server cart
  - On logout: clears cart
  - On address change: re-posts cart to new store
  - On route change to /cart or /checkout: auto-syncs with server
  - Guest users: cart in localStorage
  - Logged-in users: cart on server only
```

### AddressProvider — Store Resolution

```
STATE:
  address: { lat, lng, city, storeId, etc. }
  hub: { storeId, hasPicking, pickingStoreId }

METHODS:
  validateAndUpdateAddress(address)
    → GET /serviceableArea?lat=...&lng=...
    → Returns which store serves this address

  updateAddress(data)
    → Sets address in state + cookies
    → Triggers CommonDataProvider.updateHomeResponse()
    → Triggers CartProvider to re-post cart to new store
```

### WishlistProvider

```
STATE: wishlist (array of product objects), loading, error

METHODS:
  wishlisted(productId, sellerId) → checks if product is saved
  addToWishlist(id, sellerId) → POST to API (requires login)
  removeFromWishlist(id, sellerId) → DELETE from API
  getWishlistCount() → returns count

SIDE EFFECTS:
  - Auto-fetches on login, clears on logout
  - Redirects to login if accessing without token
```

### LanguageProvider

```
STATE: language ({ id: "ar" }), dictionary (translations object)

METHODS:
  setLanguage(lang) → sets cookie → reloads page

DERIVED:
  langSettings.dir → "rtl" for Arabic, "ltr" for others
```

### Other Providers

| Provider | State | Purpose |
|----------|-------|---------|
| DomainProvider | defaultDomain, queryDomain | API base URL |
| ThemeNameProvider | currentTheme, websiteTheme | Which theme to render |
| OrderTypeProvider | orderTypeEA | DELIVERY vs PICKUP (endless aisle) |
| LocationProvider | currentLocation {lat, lng} | User's geo location |
| DeliveryLocationProvider | currentDeliveryLocation | Delivery coordinates |
| SearchProvider | searchString | Current search query |
| AppModeProvider | mode, isEndlessAisleActive | WEBSITE vs ENDLESS_AISLE |

---

## 5. HOW A PAGE RENDERS

### Product Detail Page Example

```
1. Next.js matches URL /product/blue-shirt → pages/product/[slug]/index.jsx

2. Component mounts, reads contexts:
   - CommonDataContext → organizationData
   - LocationContext → lat/lng
   - AddressContext → storeId, orderType
   - ThemeNameContext → websiteTheme

3. useEffect triggers on [storeId, router.asPath]:
   const res = await getProductsBySlug({}, {
     slug: "blue-shirt",
     storeId: "store-123",
     orderType: "DELIVERY",
     latitude: 12.97,
     longitude: 77.59
   })

4. Repository calls: GET /layout/product?url=blue-shirt&storeId=store-123

5. API returns layout descriptor:
   {
     layouts: [
       { name: "ProductDetail", value: { product, variants } },
       { name: "ProductCollection", value: { collection: { product: [...] } } }
     ]
   }

6. formatDataOfPage() transforms the data

7. Renders: <Main websiteTheme="Fashion" pageData={formattedLayouts} />
```

### Main Component → Layout Resolution

```
<Main pageData={layouts} websiteTheme="Fashion">
         ↓
<MainUI>
  For EACH layout in pageData:
    checkLayoutExist(layout) → Should this layout render?
         ↓
    <LayoutComponent layout={layout} websiteTheme="Fashion" />
         ↓
    themes/index.jsx looks up COMPONENTS_PATHS["Fashion"].ComponentFactory
         ↓
    Dynamically imports: themes/fashion/componentFactory.jsx
         ↓
    ComponentFactory:
      switch(layout.name) {
        case 'ProductDetail': return <ProductDetail data={layout.value} />
        case 'ProductCollection': return <ProductCollection {...} />
      }
</MainUI>
```

### Page Lifecycle Summary

| Page | Data Source | Fetch Trigger | Dependencies |
|------|------------|---------------|-------------|
| Home | /layout/home | componentDidMount (in _app) | storeId, orderType |
| Product | /layout/product | useEffect | storeId, router.asPath |
| Category | /layout/category | useEffect | storeId, router.asPath |
| Cart | CartProvider context | None (context-driven) | - |
| Checkout | CartProvider + /checkout API | componentDidMount | cart state |
| Search | /layout/search | useEffect | storeId, query.q, filter |
| Account | None | None | Auth required |

---

## 6. THEME SYSTEM ARCHITECTURE

### Theme Resolution Chain

```
1. API returns: organizationData.config.website.theme = "Fashion"
2. _app.jsx stores: websiteTheme = "Fashion"
3. ThemeNameProvider makes it available everywhere
4. _app.jsx renders:
   Header({ websiteTheme: "Fashion" })
   Component({ websiteTheme: "Fashion" })
   Footer({ websiteTheme: "Fashion" })
5. themes/index.jsx resolves:
   COMPONENTS_PATHS["Fashion"] = {
     Header: dynamic(() => import('./fashion/components/Layout/Header')),
     Footer: dynamic(() => import('./fashion/components/Footer')),
     ComponentFactory: dynamic(() => import('./fashion/componentFactory')),
   }
6. next/dynamic loads ONLY Fashion's code (code splitting)
   Other 33 themes are NOT downloaded
```

### Each Theme Structure

```
theme-name/
├── componentFactory.jsx    # Maps layout names → React components (50+ cases)
├── theme.jsx               # Color palette & design tokens (50-140 tokens)
└── components/             # Theme-specific overrides (40-80 components)
    ├── Layout/Header.jsx
    ├── Footer.jsx
    ├── ProductCollection/
    └── ...
```

### Theme Color Tokens

```javascript
// themes/fashion/theme.jsx
export const defaultTheme = {
  primaryColor: '#000000',
  secondaryColor: '#f9596c',
  buttonBackground: '#000000',
  buttonTextColor: '#ffffff',
  headerBackground: '#ffffff',
  footerBackground: '#000000',
  // ... 140+ tokens
}

// Multiple variants per theme:
export const purpleTheme = { ...defaultTheme, primaryColor: '#9087ca' }
export const greenTheme = { ...defaultTheme, primaryColor: '#54cc64' }
```

Tokens accessed in styled-components:

```javascript
const Button = styled.button`
  background: ${props => props.theme.buttonBackground};
  color: ${props => props.theme.buttonTextColor};
`;
```

### Component Inheritance (80/20 Rule)

```
themes/common/ProductCollection/  ← Used by 30+ themes (SHARED, 80%)
themes/fashion/components/Header/ ← Only fashion theme (OVERRIDE, 20%)
```

New themes only need: Header, Footer, theme.jsx, and any component overrides.

### ComponentFactory — CMS-Driven Rendering

```
API returns layout descriptor (array of blocks):
[
  { name: "ImageSlideShow", value: { images: [...] } },
  { name: "ProductCollection", value: { collection: { product: [...] } } },
  { name: "BannerWithButton", value: { image, text, buttonUrl } },
  { name: "Testimonial", value: { reviews: [...] } }
]
         ↓
ComponentFactory maps each block:
  switch(layout.name) {
    case 'ImageSlideShow':    return <ImageSlideShow images={layout.value.images} />
    case 'ProductCollection': return <ProductCollection {...layout.value.collection} />
    case 'BannerWithButton':  return <BannerWithButton data={layout.value} />
    // 50+ cases
  }
```

**Interview point**: "Merchants drag-and-drop layout blocks in the admin panel. The API sends us an array of layout descriptors, and our ComponentFactory renders the right component for each block. The frontend doesn't decide what goes on the page — the CMS does."

---

## 7. API LAYER

### Repository Pattern

34 repository files in `/repositories/`, each a thin wrapper over axios:

```javascript
// cartRepository.jsx
export const postCart = async (orderType, address, eaStoreId, items, { accessToken }) => {
  // Merge duplicate items by ID
  const res = {}
  items?.forEach(item => {
    if (res[item.id]) res[item.id].q += item.q
    else res[item.id] = item
  })

  const response = await axios.post('/cart', {
    cart: { items: Object.values(res) },
    orderType,
    storeId: eaStoreId || address.storeId,
  })

  const cart = response?.data?.data?.cart
  // Add uniqueKey to each item, extract bundle metadata
  if (!accessToken) setData('cart', JSON.stringify(cart))  // localStorage for guests
  return cart
}
```

### Key Repositories

| Repository | Key Endpoints |
|-----------|--------------|
| authRepository | POST /login, POST /register, POST /otp |
| cartRepository | POST /cart, GET /cart, DELETE /cart |
| productRepository | GET /product, GET /layout/product, GET /layout/category |
| orderRepository | POST /order, GET /order/:id, DELETE /order/:id |
| addressRepository | POST /address, PUT /address/:id, GET /serviceableArea |
| wishlistRepository | GET /wishlist, POST /wishlist, DELETE /wishlist |

### Fetch Wrapper (lib/fetch.jsx)

```javascript
const headersWithAuthToken = (authToken) => ({
  ...(authToken && { Authorization: `Bearer ${authToken}` }),
  Accept: 'application/json',
  'Accept-Language': langId,
})
```

### Data Transformation Layer (lib/formatDataOfPage/)

```javascript
function formatDataOfPage(layouts = []) {
  return layouts.map(layout => {
    if (layout.name === 'ProductCollection') {
      return {
        ...layout,
        value: formatHomePageProductCollection(layout.value),
        // Normalizes products, offers, categories, variants
      }
    }
    return layout  // Pass through unchanged
  })
}
```

---

## 8. COMPLETE USER JOURNEYS

### Journey 1: Add Item to Cart

```
Step 1: User clicks "Add to Cart" button
  └── Button is inside theme component, onClick from AddToCartContainer

Step 2: AddToCartContainer.handleAddToCart()
  └── Checks address present? → No → showAddressSelectorPopup()
  └── Increments local count
  └── Sets cartAction.current = 'POST'

Step 3: useEffect detects count change, debounces 200ms
  └── Calls handlePostCart(count)

Step 4: handlePostCart calls CartContext.updateProductQuantity()
  └── Builds item: { id, q, t: Date.now(), uniqueKey, sellerID }

Step 5: CartProvider calls cartRepository.postCart()
  └── Merges duplicate items by ID

Step 6: axios.post('/cart', payload)
  └── POST https://fashionstore.com/api/cart
  └── Body: { cart: { items }, orderType, storeId, addressId }
  └── Headers: { Authorization: "Bearer eyJhb..." }

Step 7: API returns updated cart
  └── { items, youPay: 599, savings: 100, amounts: { subTotal, discount } }

Step 8: cartRepository processes response
  └── Adds uniqueKey to each item
  └── Guest? → localStorage | Logged in? → server only

Step 9: CartProvider updates state → setCart(updatedCart)

Step 10: ALL CartContext consumers re-render
  └── Cart icon → updated count
  └── Add to Cart button → shows quantity
  └── Side cart → shows new item
```

### Journey 2: Complete Checkout

```
Step 1: User navigates to /checkout
  └── CheckoutContainer.componentDidMount()

Step 2: POST /checkout
  └── { storeId, orderType, cart, couponCodes: [], paymentMethod: 'COD' }
  └── Returns: { cart, slots, paymentMethods, deliveryEstimates }

Step 3: User fills form
  └── Name, Email, Phone → handleUserDetailsChange()
  └── Apply coupon → handleOnApplyPromoCodeClick() → re-calls checkout()
  └── Select slot → handleSlotSelection()
  └── Select payment → handlePaymentMethodChange()

Step 4: "Place Order" → handleOnPlaceOrderClick() [debounced 300ms]
  └── Validates ALL fields:
      ✓ Minimum order value
      ✓ Name, phone, email
      ✓ Address selected
      ✓ Delivery slot selected
      ✓ Custom mandatory fields
      ✓ Billing address (if required)

Step 5: placeOrder() → POST /order
  └── { name, email, phone, paymentMethod, storeId, type,
       cart: [{ id, q, sellerID }], addressId, preferredDate,
       preferredSlotId, couponCodes, useWallet, metaData }

Step 6: API returns order confirmation
  └── ONLINE payment? → redirect to payment gateway → /thankyou
  └── COD? → clearCart() → Router.push('/thankyou?orderId=REF-123456')
```

### Journey 3: Address Change Cascade

```
User changes delivery address
         ↓
AddressProvider.validateAndUpdateAddress(newAddress)
  → GET /serviceableArea → returns new storeId
         ↓
AddressProvider.updateAddress({ address, hub: { storeId: "new-store" } })
  → Stores in state + cookies
         ↓
CommonDataProvider.updateHomeResponse("new-store", orderType)
  → GET /layout/home?storeId=new-store → new products for this store
  → GET /nav?storeId=new-store → updated navigation
         ↓
CartProvider detects address change (useEffect)
  → POST /cart with existing items to new store
  → Store validates: some items may be unavailable
  → Shows "items changed" popup if cart differs
         ↓
Home page re-renders with new store's products
Cart re-renders with validated items
```

---

## 9. CROSS-CUTTING CONCERNS

### Authentication Flow

```
Login: POST /login → { customer: { accessToken } }
  ↓
Token stored in 3 places:
  1. axios.defaults.headers.Authorization = "Bearer ${token}"
  2. localStorage.setItem("userData", JSON.stringify(user))
  3. nookies.set("auth_token", token, { maxAge: 30 days })
  ↓
Every subsequent API call automatically includes Bearer token
  ↓
On logout: all 3 locations cleared
```

### Guest vs Logged-in Cart

```
GUEST USER:
  Cart stored in localStorage("cart")
  POST /cart sends address object (not addressId)
  No server-side persistence

LOGGED-IN USER:
  Cart stored on server only
  POST /cart sends addressId
  On login: guest cart merged with server cart
```

### Internationalization

```
LanguageProvider reads cookie "selectedLanguage"
  → Loads dictionary: languages/ar.js (46KB translations)
  → Sets direction: "ar" → dir="rtl" (entire app flips)
  → Components use: dictionary.ADD_TO_CART → "أضف إلى السلة"

Supported: English, Hindi, Arabic (RTL), Kannada
```

### Error Monitoring Stack

```
Layer 1: ErrorBoundary (React) → catches render errors → fallback UI
Layer 2: Bugsnag → reports errors with stack traces (prod/staging only)
Layer 3: Grafana Faro → OpenTelemetry instrumentation:
  - Document load performance
  - Every fetch/XHR traced
  - User interaction events
  - Core Web Vitals (LCP, FID, CLS)
  - Session tracking
```

### Performance Optimizations

| Technique | Implementation |
|-----------|---------------|
| Code Splitting | next/dynamic for all 34 themes → only active theme loaded |
| Lazy Loading | IntersectionObserver hook for images below fold |
| Debouncing | Add to cart: 200ms, Scroll: 250ms, Place order: 300ms |
| Parallel API Calls | 6 concurrent calls on app mount |
| Progressive Images | Blur-up loading pattern |
| Skeleton Loaders | Shown while dynamic imports load |
| Brotli Compression | brotli-webpack-plugin for smaller bundles |
| Bundle Analysis | @next/bundle-analyzer (ANALYZE=true) |

### Theme-Specific Logic in Containers

Many containers have theme-specific branches:

```
'Getz': Different slot format, zone type, hierarchical addresses
'Fashion'/'Pharma': No serviceable area check
'Trendsetter'/'Gifts': Billing address required
'WSI': Custom address metadata
'Quick'/'Bazar'/'DailyCart': Side category navigation
```

### Marketplace Support

When `organizationData.config.website.isMarketPlace = true`:
- Products have `seller: { name, id }` — multiple sellers per product
- Cart items include `sellerID`
- Product queries add `source: 'seller'`
- Wishlist operations include seller ID

---

## 10. DIRECTORY STRUCTURE

```
zopping-themes/
├── pages/                  # Next.js routing (40+ routes)
│   ├── _app.jsx            # App wrapper, provider nesting, boot
│   ├── _document.jsx       # HTML skeleton, preconnect hints
│   ├── index.jsx           # Home page
│   ├── product/[slug]/     # Product detail (dynamic route)
│   ├── category/[name]/    # Category listing
│   ├── cart/               # Cart page
│   ├── checkout/           # Checkout page
│   ├── account/            # Account pages (orders, profile)
│   ├── search/             # Search results
│   └── blog/               # Blog listing
│
├── containers/             # Business logic (62+ files)
│   ├── CheckoutContainer   # 2800 lines — checkout orchestration
│   ├── AddToCartContainer  # Add-to-cart + quantity management
│   ├── CartContainer       # Cart page logic
│   ├── AllProductsContainer # Product listing + infinite scroll
│   └── ProductDetailsContainer # PDP logic
│
├── providers/              # React Context (13 providers)
│   ├── CommonDataProvider  # Org config, navigation, layouts
│   ├── CartProvider        # Cart state (691 lines, most complex)
│   ├── AccountProvider     # Auth state
│   ├── AddressProvider     # Address + store resolution (341 lines)
│   ├── WishlistProvider    # Saved items
│   ├── LanguageProvider    # i18n + RTL
│   └── ...7 more
│
├── repositories/           # API abstraction (34 files)
│   ├── cartRepository      # Cart CRUD
│   ├── productRepository   # Product search/details
│   ├── authRepository      # Login/register/OTP
│   ├── orderRepository     # Order management
│   └── addressRepository   # Address CRUD + serviceability
│
├── themes/                 # 34 themes + common library
│   ├── index.jsx           # Central registry (maps theme → components)
│   ├── common/             # 100+ shared components
│   │   ├── ProductCollection/
│   │   ├── Banner/
│   │   ├── Carousel/
│   │   ├── CartPage/
│   │   ├── CheckoutPage/
│   │   ├── Main/           # Layout rendering engine
│   │   ├── atomicComponents/ # Button, Input, TextUI
│   │   └── ...100+ more
│   ├── default/            # Default theme
│   ├── fashion/            # Fashion theme (140+ tokens)
│   ├── beauty/             # Beauty theme (color variants)
│   └── ...31 more themes
│
├── zs-components/          # Extra shared library (122+ components)
├── lib/                    # Core utilities
│   ├── fetch.jsx           # Axios wrapper with auth
│   ├── index.jsx           # localStorage/cookie helpers
│   ├── ErrorBoundary.jsx   # Bugsnag error boundary
│   ├── formatDataOfPage/   # Layout data transformation
│   ├── tracking/           # Google Analytics
│   └── GtmWrapper.jsx      # GTM initialization
│
├── utils/                  # 76+ helpers
│   ├── constants.js        # Global constants
│   ├── formatPrice.js      # Price formatting
│   ├── apiUrlHoc.js        # withApiUrl HOC
│   ├── intersectionObserver.js # Lazy loading hook
│   ├── checkLayoutExist.js # Layout visibility logic
│   └── debounce.js         # Debounce utility
│
├── languages/              # i18n (en, hi, ar, kn)
├── server.js               # Custom Node.js server (multi-tenant)
├── initialize.js           # Grafana Faro setup
├── next.config.js          # Security headers, image domains, bundle config
├── Dockerfile              # Node Alpine, port 3000
└── deployment.yaml         # Kubernetes config
```

---

## 11. TECH STACK

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | Next.js (Pages Router) | 14.2.0 |
| UI Library | React | 18.3.1 |
| Language | JavaScript (JSX) + TypeScript config | TS 5.9.3 |
| Styling | styled-components | 5.3.11 |
| HTTP Client | Axios | 1.13.2 |
| State | React Context API | (no Redux) |
| Testing | Jest + React Testing Library | 29.6.2 |
| Errors | Bugsnag | 8.8.1 |
| Observability | Grafana Faro + OpenTelemetry | - |
| Analytics | react-ga + react-gtm-module | 3.0.0 |
| Cookies | nookies | 2.0.8 |
| Dates | date-fns | 2.8.1 |
| Carousel | react-slick | 0.29.0 |
| Recaptcha | react-google-recaptcha | 2.1.0 |
| Compression | brotli-webpack-plugin | - |
| Linting | ESLint + Prettier | 8.57.0 |
| Git Hooks | Husky + lint-staged + commitlint | 3.1.0 |
| Storybook | @storybook/react | 8.1.1 |
| Deployment | Docker + Kubernetes + GitHub Actions | - |

---

## 12. DESIGN PATTERNS

| Pattern | Where | Interview Explanation |
|---------|-------|----------------------|
| **Factory** | componentFactory.jsx | "Maps layout descriptors from API to React components. Makes the frontend CMS-driven." |
| **Provider/Context** | 13 providers | "Isolated state domains — cart, auth, address each have their own provider. Avoids Redux boilerplate." |
| **Repository** | 34 repository files | "Thin wrappers over axios. Containers never call HTTP directly. Single place to change endpoints." |
| **Container/Presenter** | containers/ → themes/ | "Containers own business logic, theme components own presentation. Swap themes without touching logic." |
| **Strategy** | Theme system | "Same interface (Header, Footer, ComponentFactory), different implementations per theme." |
| **HOC** | withApiUrl | "Injects domain/API URL into any component via higher-order component." |
| **Observer** | IntersectionObserver | "Lazy-loads images and triggers infinite scroll when elements enter viewport." |
| **Singleton** | GA/Bugsnag/Faro init | "Window flags prevent re-initialization on re-renders." |
| **Facade** | lib/fetch.jsx | "Wraps axios with auth headers, language headers, and response parsing." |
| **Atomic Design** | atomicComponents/ | "Button, Input, TextUI are base atoms composed into larger molecules/organisms." |

---

## 13. INTERVIEW Q&A

### Q: "Tell me about the architecture of a project you've worked on."
> "I worked on a multi-tenant e-commerce platform serving 34+ themes from a single Next.js codebase. We used a Factory Pattern for theme rendering, Repository Pattern for API abstraction, and React Context for state management. The architecture was layered: Pages → Containers (logic) → Providers (state) → Repositories (API) → Themes (UI). The key innovation was the CMS-driven layout system — the API returns an array of layout descriptors, and our ComponentFactory renders the correct component for each block, making the frontend fully configurable without code changes."

### Q: "How did you handle multi-tenancy?"
> "Our custom Node.js server extracts the domain from the Host header, injects it into Next.js query params, and a DomainProvider makes it available via React Context. The CommonDataProvider then fetches tenant-specific configuration (theme, navigation, layouts) from the API. Every API call goes to ${domain}/api/... so each merchant gets their own data."

### Q: "How did you optimize performance?"
> "Code splitting was critical with 34 themes. We used next/dynamic to lazy-load only the active theme — users download ~200KB of theme code instead of the full bundle. We also made 6 parallel API calls on app mount to avoid waterfall loading, used IntersectionObserver for lazy image loading, debounced rapid user actions (200-300ms), added Brotli compression, and used skeleton loaders during dynamic imports."

### Q: "Why Context over Redux?"
> "Our state domains are naturally isolated — cart, auth, wishlist, address — they rarely interact directly. Context gave us simpler code without Redux boilerplate (actions, reducers, action types). Each provider is self-contained with its own state and methods. The dependency chain between providers is managed through nesting order."

### Q: "How does the theming system work?"
> "Each theme implements a componentFactory that maps layout names to React components via a switch statement. The API returns a page layout descriptor (array of layout objects from the website builder), and the factory renders the right component for each block. Themes inherit ~80% from themes/common/ and only override Header, Footer, color tokens, and any components they want to customize."

### Q: "Describe a complex feature you built."
> "The checkout flow (CheckoutContainer at 2800 lines) handles multiple payment methods, address management with hierarchical addresses (Province→City→Zipcode), digital vs physical product separation, B2B GST verification, multiple order types (delivery/pickup/dine-in), coupon application with real-time recalculation, delivery slot selection, wallet/credit integration, OTP verification, and prescription uploads — all while remaining theme-agnostic through the container/presenter pattern."

### Q: "How do cascading state updates work?"
> "When a user changes their delivery address, it triggers a chain: AddressProvider validates the address against the serviceability API to get the new storeId → CommonDataProvider refetches the home layout and navigation for the new store → CartProvider re-posts the cart to the new store (some items may become unavailable). This cascade is managed through useEffect hooks that watch context values."

### Q: "How do you handle auth?"
> "We store the auth token in three places: axios default headers (for API calls), localStorage (for client-side persistence), and HTTP cookies via nookies (for server-side access during SSR). On login, all three are set. On logout, all three are cleared. Guest users store their cart in localStorage, and on login, the guest cart is merged with the server cart."

### Q: "How do you handle errors in production?"
> "Three layers: React ErrorBoundary catches render errors and shows a fallback UI. Bugsnag captures and reports all errors with stack traces and user context. Grafana Faro provides OpenTelemetry instrumentation for every fetch request, document load, user interaction, and Core Web Vitals — so we can trace performance issues end-to-end."

### Q: "How do you handle i18n?"
> "Manual language files for English, Hindi, Arabic, and Kannada. LanguageProvider stores the selection in a cookie and provides translations via Context. Arabic triggers RTL layout across the entire app using CSS direction property. The dictionary is a flat key-value object (~50KB per language) loaded on app boot."

### Q: "What would you improve?"
> "1) Migrate CheckoutContainer (2800 lines) to hooks and break it into smaller composable hooks. 2) Move from Pages Router to App Router for better streaming SSR. 3) Add server-side caching (Redis) for organization/navigation data since it rarely changes. 4) Replace manual language files with a proper i18n library like next-intl. 5) Add E2E tests with Playwright — currently only unit tests exist."

---

*Generated for interview preparation. Last updated: 2026-03-26.*
