# Zopping Themes — Complete Codebase Deep-Dive (Interview Reference)

> **Project**: Multi-tenant, multi-theme e-commerce storefront
> **Stack**: Next.js 14.2.0 | React 18 | styled-components | TypeScript
> **Version**: 3.2.5 | **Themes**: 34+ | **Components**: 222+

---

## 1. What Is This Project?

A **multi-tenant, multi-theme e-commerce storefront** built with **Next.js 14.2.0 + React 18**. A single codebase serves **34+ distinct themes** (like Shopify themes but custom-built). Each merchant on the Zopping platform gets a theme, and this app dynamically renders the correct one based on the domain in the request.

**Key mental model**: Think of it as a **Theme Engine** — one Next.js app, many visual skins, shared business logic.

---

## 2. Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 14.2.0 (Pages Router, not App Router) |
| UI Library | React 18.3.1 |
| Language | JavaScript (JSX) with TypeScript config for type-checking |
| Styling | styled-components 5.3.11 (CSS-in-JS) |
| HTTP Client | Axios 1.13.2 |
| State Management | React Context API (no Redux) |
| Testing | Jest 29 + React Testing Library |
| Error Monitoring | Bugsnag |
| Observability | Grafana Faro + OpenTelemetry |
| Analytics | Google Analytics (react-ga) + GTM |
| i18n | Manual JSON-like language files (en, hi, ar, kn) with RTL support |
| Deployment | Docker + Kubernetes |
| CI/CD | GitHub Actions |
| Code Quality | ESLint + Prettier + Husky + lint-staged + commitlint |

---

## 3. Directory Structure & Architecture

```
zopping-themes/
├── pages/                  # Next.js file-based routing (40+ routes)
├── containers/             # Business logic components (62+ files)
├── providers/              # React Context providers (13 providers)
├── repositories/           # API abstraction layer (34 files)
├── themes/                 # 34 themes + 1 common shared library
│   ├── common/             # 100+ shared UI components
│   ├── default/            # Default theme
│   ├── fashion/            # Fashion theme
│   ├── beauty/             # Beauty theme (with color variants)
│   └── ... (30+ more)
├── zs-components/          # Additional shared component library (122+ components)
├── lib/                    # Core utilities (fetch wrapper, ErrorBoundary, tracking)
├── utils/                  # 76+ helper functions
├── languages/              # i18n files (en, hi, ar, kn)
├── styles/                 # Global CSS tokens
├── public/                 # Static assets
├── configs/                # App configuration
├── server.js               # Custom Node.js server (multi-tenant routing)
├── initialize.js           # Grafana Faro initialization
├── next.config.js          # Next.js config
├── Dockerfile              # Docker build
├── deployment.yaml         # Kubernetes config
└── package.json            # v3.2.5
```

---

## 4. Architecture — Layered Design

The app follows a **clean layered architecture** with clear separation of concerns:

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

## 5. Multi-Tenancy — How It Works

### Request Flow:
1. **`server.js`** — Custom Node.js HTTP server (not Express)
2. Extracts **domain** from the `Host` header of each request
3. Injects the domain into Next.js query params
4. **`DomainProvider`** stores the domain in React Context
5. **`CommonDataProvider`** fetches organization-specific config (theme name, navigation, layouts) from the Zopping API using that domain
6. **`ThemeNameProvider`** resolves which theme to render
7. **`themes/index.jsx`** — Central theme registry maps theme names to their component factories using `next/dynamic` (code-split per theme)

**Key point**: Only the selected theme's code is loaded (dynamic imports), so a user visiting a "fashion" store never downloads "pharma" theme code.

---

## 6. Theme System — Factory Pattern

### `themes/index.jsx` — Theme Registry
```javascript
COMPONENTS_PATHS = {
  'Default-Theme': { Header, Footer, ComponentFactory, SectionHeading },
  'Fashion':       { Header, Footer, ComponentFactory, SectionHeading },
  // ... 34+ themes
}
```

### Each Theme Has:
```
theme-name/
├── componentFactory.jsx    # Maps layout names → React components
├── theme.jsx               # Color palette & design tokens
└── components/             # Theme-specific overrides
    ├── Layout/Header.jsx
    ├── Footer.jsx
    ├── ProductCollection/
    ├── ImageCarousel/
    └── ... (40-80+ components)
```

### `componentFactory.jsx` — The Heart of Each Theme
Uses a **massive switch statement** to map layout names (from API) to components:
```javascript
switch(layout.name) {
  case 'ProductCollection': return <ProductCollection />
  case 'ImageSlideShow':    return <ImageSlideShow />
  case 'CartPage':          return <CartPage />
  case 'CheckoutPage':      return <CheckoutPage />
  // 50+ cases
}
```

**Interview framing**: This is a **Factory Pattern** — the API returns a page layout descriptor (array of layout objects), and the component factory renders the correct component for each layout block. This makes the storefront **CMS-driven** — merchants can rearrange their storefront via a website builder, and the theme just renders whatever layout the API returns.

### `theme.jsx` — Design Tokens
```javascript
export const beautyTheme = {
  primaryColor: '#000000',
  secondaryColor: '#fb586c',
  mainBackground: '#f2f2f2',
  buttonBackground: '#fb586c',
  // 50+ tokens
}
```

### Inheritance Model:
- **~80% shared**: `themes/common/` has 100+ reusable components
- **~20% custom**: Each theme overrides only what differs (Header, Footer, color palette, specific UI treatments)

---

## 7. State Management — Context API (No Redux)

**13 specialized providers**, each managing a distinct domain:

| Provider | Purpose |
|----------|---------|
| `CommonDataProvider` | Navigation, layouts, org config |
| `CartProvider` | Cart state, add/remove/update items (29KB — most complex) |
| `AccountProvider` | Auth state, login/logout, tokens |
| `AddressProvider` | Address CRUD, delivery addresses |
| `WishlistProvider` | Saved items |
| `ThemeNameProvider` | Current theme |
| `LanguageProvider` | Active language + RTL direction |
| `OrderTypeProvider` | DELIVERY vs PICKUP |
| `LocationProvider` | Geo-location tracking |
| `DeliveryLocationProvider` | Delivery area specifics |
| `SearchProvider` | Search query state |
| `DomainProvider` | Current domain/tenant |
| `AppModeProvider` | App mode (standard, endless-aisle) |

### Pattern Used:
```javascript
const [state, setState] = useState(initialState);
// Exposes methods like addToCart(), login(), updateAddress()
// Children consume via useContext(CartContext)
```

**Why Context over Redux**: "State domains are naturally isolated — cart state doesn't depend on wishlist state. Each provider is self-contained, which avoids Redux boilerplate."

---

## 8. API Layer — Repository Pattern

**34 repository files** in `/repositories/`:

| Repository | Responsibility |
|-----------|---------------|
| `authRepository.js` | Login, register, OTP, password reset |
| `cartRepository.jsx` | Cart CRUD operations |
| `productRepository.js` | Product search, filtering, details |
| `orderRepository.js` | Order placement & history |
| `addressRepository.js` | Address CRUD |
| `paymentObjectRepository.js` | Payment processing |
| `wishlistRepository.js` | Wishlist operations |
| `navigationDataRepository.js` | Navigation menus |
| `blogsRepository.js` | Blog content |
| `offerRepository.js` | Offers/discounts |
| ... | 24 more |

### HTTP Client Setup:
```javascript
// lib/fetch.jsx — Axios wrapper
// Bearer token auth, language headers, response parsing
axios.defaults.headers.common.Authorization = `Bearer ${token}`
```

**Interview framing**: "We use the Repository Pattern to abstract API calls. Containers never call axios directly — they go through repositories. This gives us a single place to change endpoint URLs, add headers, or swap HTTP clients."

---

## 9. Routing — Next.js Pages Router

```
pages/
├── index.jsx                          # Home page
├── _app.jsx                           # App wrapper (providers nested here)
├── _document.jsx                      # HTML document wrapper
├── cart/index.jsx                     # Cart page
├── checkout/index.jsx                 # Checkout page
├── product/[slug]/index.jsx           # Product detail page (dynamic)
├── product/[slug]/[sellerId].jsx      # Product by seller
├── category/[categoryName]/index.jsx  # Category listing
├── seller/[sellerSlug]/index.jsx      # Seller page
├── blog/index.jsx                     # Blog listing
├── account/                           # Account pages
├── pages/[page]/index.jsx             # CMS pages (dynamic)
└── ... (40+ routes total)
```

---

## 10. Container Pattern — Separation of Concerns

**62+ containers** in `/containers/`:

```
Container (logic) → passes data as props → Theme Component (presentation)
```

Key containers:
- `CheckoutContainer.jsx` — **89KB**, the most complex — handles entire checkout flow
- `AddToCartContainer.jsx` — Add-to-cart logic with variant selection
- `CartContainer.jsx` — Cart display and operations
- `ProductDetailsContainer/` — Product detail page logic
- `AllProductsContainer.jsx` — Product listing with filters
- `AddNewAddressContainer.jsx` — Address form logic (26KB)

**Interview framing**: "Containers own the business logic (API calls, state transformations, validation), while theme components are pure presentation. We can swap themes without touching business logic."

---

## 11. Shared Component Libraries

### `themes/common/` — 100+ shared theme components:
- **Layout**: Accordion, Banner, Carousel, Grid, ImageSlider, Marquee, Pagination
- **Commerce**: ProductCollection, OfferLabel, PricingWidget, Rating, Reviews
- **Forms**: GenericForm, CustomFields, RadioButton, Recaptcha
- **Content**: Blogs, RichText, Testimonial, BrandCollection, CategoryCollection
- **Utility**: ClientOnlyPortal, DynamicImport, ErrorBoundary, Loader, Toast

### `zs-components/` — 122+ reusable UI components:
- Separate component library with Storybook stories for documentation
- Higher-level composed components (ImageBanner, Catalog, Highlights, etc.)

### Atomic Components (`themes/common/atomicComponents/`):
- Button, Input, TextUI — base building blocks (Atomic Design)

---

## 12. Styling Architecture

**styled-components** (CSS-in-JS) throughout:

```javascript
import styled from 'styled-components';

export const ProductCard = styled.div`
  background: ${props => props.theme.mainBackground};
  color: ${props => props.theme.primaryColor};
`;
```

- **Theme tokens** injected via styled-components `ThemeProvider`
- **Responsive**: Uses `styled-breakpoints` library for media queries
- **Global CSS**: `/styles/tokens.css` for CSS custom properties
- **RTL support**: Dynamic `direction: rtl/ltr` based on language

---

## 13. Performance Optimization

| Technique | Implementation |
|-----------|---------------|
| **Code Splitting** | `next/dynamic` for all theme components — only active theme loaded |
| **Lazy Loading** | Custom `useIntersectionObserver` hook for images and components |
| **Bundle Analysis** | `@next/bundle-analyzer` (enabled via `ANALYZE=true`) |
| **Compression** | Brotli via `brotli-webpack-plugin` |
| **Skeleton Loaders** | `LayoutSkelatonLoader` component while dynamic imports load |
| **Progressive Images** | `ProgressiveImage` component for blur-up loading |
| **Debouncing** | `/utils/debounce.js` for search and scroll handlers |

**Interview talking point**: "With 34 themes, bundle size was critical. We use Next.js dynamic imports so each theme is a separate chunk. A user visiting a 'fashion' store loads only that theme's code instead of the full bundle of all themes combined."

---

## 14. Authentication Flow

**`AccountProvider.jsx`** handles all auth:

1. **Login** → POST to `/login` → receives `accessToken`
2. **Token Storage** → Dual storage: `localStorage` + `cookies` (via nookies)
3. **Session Persistence** → 30-day cookie expiry, checked on app mount
4. **Axios Interceptor** → `Authorization: Bearer ${token}` added to all requests
5. **Logout** → Clears localStorage + cookies, resets axios headers

Supports: Email/password, OTP, Google reCAPTCHA, GST verification (B2B)

---

## 15. Internationalization (i18n)

```
languages/
├── en.js    # English (54KB)
├── hi.js    # Hindi (93KB)
├── ar.js    # Arabic (46KB) — RTL
└── kn.js    # Kannada
```

- Language stored in `selectedLanguage` cookie
- `LanguageProvider` exposes translations via Context
- **RTL support**: When Arabic is selected, entire layout flips to right-to-left
- Server-side language detection via `getLangInServerSide(ctx)`

---

## 16. Error Handling & Monitoring

| Tool | Purpose |
|------|---------|
| **Bugsnag** | Error tracking (production/staging only) |
| **ErrorBoundary** | React error boundary wrapping Bugsnag |
| **Grafana Faro** | Performance monitoring + distributed tracing |
| **OpenTelemetry** | Document load, fetch, user interaction instrumentation |
| **Google Analytics** | Page views + events |
| **GTM** | Tag management for marketing |

---

## 17. DevOps & Deployment

```dockerfile
# Dockerfile — Node.js Alpine
FROM node:lts-alpine
COPY . .
RUN npm install && npm run build
EXPOSE 3000
CMD ["npm", "start"]  # Uses PM2 in production
```

- **Kubernetes**: `deployment.yaml` for container orchestration
- **CI/CD**: GitHub Actions (`ci.yaml`, `themes.yaml`)
- **Pre-commit**: Husky + lint-staged + commitlint (conventional commits)

---

## 18. Key Design Patterns

| Pattern | Where Used | Why |
|---------|-----------|-----|
| **Factory Pattern** | `componentFactory.jsx` in every theme | Maps API layout descriptors → React components |
| **Provider Pattern** | 13 Context providers | Isolated state domains without Redux boilerplate |
| **Repository Pattern** | 34 repository files | Abstracts API calls, single point for HTTP changes |
| **Container/Presenter** | Containers → Theme Components | Separates business logic from UI |
| **HOC Pattern** | `withApiUrl()` in utils | Injects domain/API URL into components |
| **Strategy Pattern** | Theme system | Same interface, different visual implementations |
| **Singleton** | GA/Bugsnag/Faro init | One-time initialization with guard flags |
| **Atomic Design** | `atomicComponents/` | Button, Input, TextUI as base building blocks |
| **Dynamic Import** | `next/dynamic` everywhere | Code splitting per theme |
| **Observer Pattern** | IntersectionObserver | Lazy loading images and infinite scroll |

---

## 19. Data Flow — End-to-End Example

**"User adds a product to cart":**

```
1. User clicks "Add to Cart" button
   └── Theme Component (e.g., fashion/ProductCard.jsx)

2. Event handler calls container method
   └── AddToCartContainer.jsx → handleAddToCart()

3. Container calls provider method
   └── CartProvider.updateProductQuantity()

4. Provider calls repository
   └── cartRepository.postCart(payload)

5. Repository makes API call
   └── axios.post('/cart', data) with Bearer token

6. Response updates provider state
   └── CartProvider setState({ cartItems: [...] })

7. All consumers re-render
   └── CartIcon shows updated count
   └── CartPage shows new item
```

---

## 20. Notable Business Features

- **Multi-variant products**: Size/color selection with variant-specific pricing
- **Digital products**: Separate cart sections for physical vs digital
- **Dynamic filters**: API-driven product filters
- **Order tracking**: Google Maps integration for delivery tracking
- **B2B support**: GST verification, payment credit terms
- **Website builder integration**: CMS-driven layouts — merchants rearrange pages via API
- **Multiple order types**: Delivery, Pickup, Dine-in
- **Family management**: Add family members (Getz theme for HK/PH)
- **Hierarchical addresses**: Province → City → Zipcode (WSI/Getz themes)
- **Offer badges**: Configurable offer labels on product cards
- **VIP Membership**: Loyalty program support
- **Wallet**: User wallet with credit system

---

## 21. Interview Q&A Ready Answers

### Q: "Tell me about the architecture of a project you've worked on."
> "I worked on a multi-tenant e-commerce platform serving 34+ themes from a single Next.js codebase. We used a Factory Pattern for theme rendering, Repository Pattern for API abstraction, and React Context for state management. The architecture was layered: Pages → Containers (logic) → Providers (state) → Repositories (API) → Themes (UI)."

### Q: "How did you handle multi-tenancy?"
> "Our custom Node.js server extracts the domain from the Host header, injects it into query params, and a DomainProvider makes it available via React Context. The CommonDataProvider then fetches tenant-specific configuration (theme, navigation, layouts) from the API."

### Q: "How did you optimize performance?"
> "Code splitting was critical with 34 themes. We used next/dynamic to lazy-load only the active theme. We also used IntersectionObserver for lazy image loading, Brotli compression, skeleton loaders, and bundle analysis to keep chunk sizes small."

### Q: "Why Context over Redux?"
> "Our state domains are naturally isolated — cart state doesn't depend on wishlist state. Each provider is self-contained, which avoids Redux boilerplate and makes the code more maintainable across 34+ themes."

### Q: "How does the theming system work?"
> "Each theme implements a componentFactory that maps layout names to React components. The API returns a page layout descriptor (array of layout objects), and the factory renders the right component for each block. Themes inherit ~80% from a shared common library and only override what differs — mainly Header, Footer, and color palette."

### Q: "How do you handle errors in production?"
> "We use Bugsnag for error tracking with React ErrorBoundary integration, Grafana Faro for performance monitoring with OpenTelemetry instrumentation (document load, fetch, user interaction), and Google Analytics + GTM for business metrics."

### Q: "How do you ensure code quality?"
> "We have ESLint + Prettier for consistent formatting, Husky pre-commit hooks running lint-staged, commitlint for conventional commit messages, Jest + React Testing Library for unit tests, and Storybook for component documentation."

### Q: "Describe a complex feature you built."
> "The checkout flow (CheckoutContainer at 89KB) handles multiple payment methods, address management with hierarchical addresses (Province→City→Zipcode), digital vs physical product separation, B2B GST verification, multiple order types (delivery/pickup/dine-in), offer application, and wallet integration — all while remaining theme-agnostic through the container/presenter pattern."

### Q: "How do you handle internationalization?"
> "We have language files for English, Hindi, Arabic, and Kannada. The LanguageProvider stores the selection in a cookie for persistence. Arabic triggers RTL layout direction across the entire app. Server-side language detection works via getLangInServerSide() for SEO."

---

*Generated on 2026-03-26 for interview preparation.*
