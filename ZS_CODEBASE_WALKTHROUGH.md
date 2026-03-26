# `@zopsmart/zs-components` — Complete Codebase Walkthrough

## 1. What Is This Project?

This is a **shared React component library** (`@zopsmart/zs-components` v2.3.3) used across multiple Zopsmart products — EazyUpdates, HiringMotion, MentorPro, TestPaper, Training, and Zopping. It exports **118+ components, hooks, and utilities** from a single entry point.

**Interview angle:** "I built and maintained an internal design system / component library consumed by 6+ product teams. It's a multi-brand, themeable library with 95+ components."

---

## 2. Project Structure (Top-Down)

```
zs-components/
├── src/
│   ├── index.js              ← Single barrel export (118 exports)
│   ├── components/           ← 95 component directories
│   ├── common/               ← 21 shared primitives (Button, Image, Grid, etc.)
│   ├── utils/                ← 9 utility files (debounce, GUID, ErrorBoundary)
│   └── css/                  ← 11 theme/token CSS files
├── dist/                     ← Build output (CJS + ESM)
├── .storybook/               ← Storybook 7 config
├── .github/workflows/        ← CI/CD (GitHub Actions → Google Artifact Registry)
├── example/                  ← Example consumer app
└── package.json              ← Microbundle build, peer dep on React 17
```

### Key directories:

| Directory | Purpose | Count |
|-----------|---------|-------|
| `src/components/` | Feature-rich UI components | 95 |
| `src/common/` | Reusable primitives shared across components | 21 |
| `src/utils/` | Helpers: debounce, GUID, ErrorBoundary, upload | 9 |
| `src/css/` | Design tokens + 8 brand themes | 11 files |

---

## 3. Architecture & Design Patterns

### A. Component Pattern: Functional Components + Hooks

Every component is a **modern functional component** (no class components except `ErrorBoundary`). Standard hooks used throughout:

- `useState` — local state management
- `useEffect` — side effects (event listeners, body scroll lock)
- `useRef` — DOM access (positioning, outside-click detection)
- `useCallback` — memoized event handlers

**Interview talking point:** "We deliberately avoided class components. The only class component is the ErrorBoundary — because React still requires class components for `componentDidCatch`."

### B. Styling: styled-components + CSS Custom Properties (Design Tokens)

This is the **most interview-worthy part** of the architecture:

```
┌─────────────────────────────────────────┐
│           CSS Variables (tokens.css)     │  ← Design tokens layer
│  --color-primary, --size-spacing-m, etc │
├─────────────────────────────────────────┤
│         Brand Theme Overrides           │  ← zopping.css, mentorPro.css, etc.
│  Override token values per brand        │
├─────────────────────────────────────────┤
│         styled-components               │  ← Component styling layer
│  Consume tokens via var(--token-name)   │
│  Dynamic styling via props              │
└─────────────────────────────────────────┘
```

**How theming works:**
1. `src/css/tokens.css` defines the base design tokens (colors, spacing, typography, shadows, border-radius)
2. Brand-specific CSS files (e.g., `src/css/zopping.css`) override those variables
3. styled-components reference tokens via `var(--color-primary)`, making components automatically adapt to whatever theme CSS is loaded

**Example from Button:**
```js
const getButtonBackgroundColor = (variant, color) => {
  switch (variant) {
    case 'textOnly': return 'transparent'
    case 'lowEmphasis': return `var(${COLOR_MAP[color]}-light)`
    default: return `var(${COLOR_MAP[color]})`
  }
}
```

**Interview talking point:** "We use a two-layer theming system. CSS custom properties act as design tokens, and styled-components consume them. This lets us support 8 different brand themes without any JS runtime cost for theme-switching — just swap the CSS file."

### C. Component Composition Patterns

**1. Variant-driven components** — Most components accept `variant`, `size`, `color` props:
```js
<Button variant="filled" size="m" color="primary" />
<Button variant="textOnly" size="s" color="error" />
```

**2. Compound components** — Table exports sub-components:
```js
import { Table, TableRow, TableCell, TableDataWrapper } from '@zopsmart/zs-components'
```

**3. Controlled components** — Form inputs follow React's controlled pattern:
```js
<InputFields value={val} onChange={setVal} isError={hasError} message="Required" />
```

**4. Render callback / custom rendering** — Table supports `customRowHandler` for custom row rendering.

### D. State Management

- **No Redux, no Context API** (by design for a library)
- All state is **local** via `useState`
- Data flows **top-down via props**, consumers manage app state
- Components expose callbacks: `onChange`, `onClick`, `onClose`

**Interview talking point:** "As a library, we intentionally avoid global state. Each component is self-contained. The consuming app owns the state and passes it via props + callbacks. This keeps the library framework-agnostic at the state layer."

### E. Side Effect Patterns

Common patterns across Modal, Drawer, Calendar, Tooltip:

```js
// Body scroll lock
useEffect(() => {
  document.body.style.overflow = 'hidden'
  return () => { document.body.style.overflow = 'auto' }
}, [])

// Outside-click detection
useEffect(() => {
  const handler = (e) => {
    if (ref.current && !ref.current.contains(e.target)) onClose()
  }
  document.addEventListener('click', handler)
  return () => document.removeEventListener('click', handler)
}, [])
```

---

## 4. Build System

### Microbundle-CRL (internally uses Rollup)

```
src/index.js  ──→  Microbundle  ──→  dist/index.js        (CJS, ~3 MB)
                                 ──→  dist/index.modern.js (ESM, ~3 MB)
                                 ──→  dist/index.css       (14.5 KB)
                                 ──→  *.map                (source maps)
```

**Dual output:**
- `main: dist/index.js` — CommonJS for older bundlers / Node
- `module: dist/index.modern.js` — ES Modules for tree-shaking

**Interview talking point:** "We use dual CJS/ESM output so consuming apps get tree-shaking benefits with modern bundlers while maintaining backwards compatibility. The `module` field in package.json tells bundlers like Webpack to prefer the ESM build."

### Why Microbundle?
- Zero-config bundler designed specifically for libraries
- Rollup under the hood (better tree-shaking than Webpack for libraries)
- Automatic source maps, CJS/ESM dual output

---

## 5. Design Token System

`tokens.css` defines a comprehensive token hierarchy:

| Category | Examples | Range |
|----------|----------|-------|
| **Spacing** | `--size-spacing-xxs` to `--size-spacing-h` | 0.25rem → 2.5rem |
| **Font sizes** | `--size-font-xs` to `--size-font-l` | 0.625rem → 1rem |
| **Headings** | `--size-heading-s` to `--size-heading-xxxl` | 1.25rem → 6rem |
| **Colors** | `--color-gray-50` to `--color-gray-900`, `--color-primary`, `--color-error` | Full palette |
| **Shadows** | `--shadow-xs` to `--shadow-xl` | 5 levels |
| **Border radius** | `--size-border-radius-tiny` to `--size-border-radius-h` | 1px → 16px |
| **Font weights** | `--font-weight-xxs` to `--font-weight-xl` | 100 → 700 |

**8 brand themes** override these tokens: default-light, default-dark, zopping, eazyupdates, hiringmotion, mentorPro, testpaper, training.

---

## 6. Component Categories

### Form Components
`InputFields`, `TextArea`, `Checkbox`, `Radio`, `Switch`, `SelectInputField`, `Phone`, `DynamicInput`, `MonthPicker`, `TimePicker`, `Calendar`, `FileUpload`, `Form`, `GenericForm`

- All support `label`, `mandatory`, `isError`, `message` pattern
- Controlled component pattern throughout

### Data Display
`Table`, `EnhanceTable` (TanStack React Table v8), `DataCard`, `DashboardCard`, `Timeline`, `Pagination`

- Table is **responsive**: renders as `<table>` on desktop, cards on mobile (768px breakpoint)
- EnhanceTable wraps `@tanstack/react-table` for advanced features

### Charts (Recharts-based)
`AreaChart`, `LineChart`, `BarGraph`, `PieChart`, `DoughnutChart`, `ProgressChart`

### Media
`ImageSlider`, `ImageSlideShow`, `ImageGallery`, `ImageCollection`, `ImageGrid`, `VideoCarousel`, `SimpleVideoPlayer`, `Carousel`

### Overlays
`Modal`, `Drawer`, `Tooltip`, `Popover`, `Notification`, `StickyAlert`

### Layout
`Grid` (AutoGrid, CssGrid, InfiniteGrid, JsGrid), `MasonryLayout`, `Spacer`, `Divider`, `BGImageWithContent`, `ContentLayout`

### Business / Domain
`Product`, `ProductCard`, `CartItem`, `Coupon`, `PricingCard`, `PricingWidget`, `Catalog`, `QuantityController`, `Rating`

### Admin / CMS Edit Components
~20 "edit" variants (e.g., `EditBanner`, `EditCatalog`, `EditHighlightsSection`) — these are admin UI components that let CMS users visually edit sections.

---

## 7. Shared Utilities & Hooks

| File | Purpose |
|------|---------|
| `ErrorBoundary.js` | Class-based error boundary wrapping children |
| `debounce.js` | Debounce utility for performance |
| `guid.js` | GUID generation |
| `phoneUtil.js` | Phone validation helpers |
| `styleVariables.js` | Static responsive padding/width constants |
| `Upload.js` | File upload helpers |
| `useWindowSize` | Custom hook — returns debounced window dimensions |

---

## 8. Testing & Documentation

### Storybook 7
- **113 story files** — every component has a `.stories.jsx`
- Webpack5 builder
- Addons: a11y (accessibility auditing), interactions, essentials, links
- **Theme selector** in Storybook toolbar (switch between all 8 themes)
- `withTheme` decorator wraps every story

### Testing
- **Jest** via react-scripts (jsdom environment)
- **13 test files** (focused on SlideCarousel, Link, ImageCollection)
- **Storybook test runner** (`test-storybook`)
- **Chromatic** for visual regression testing

**Interview talking point (honest):** "Test coverage is focused on the most complex interactive components like Carousel. We rely heavily on Storybook stories + Chromatic visual regression for most components, which catches visual regressions that unit tests miss."

---

## 9. CI/CD & Publishing

```
GitHub Release created
        │
        ▼
GitHub Actions Workflow
        │
        ├── Checkout code
        ├── Authenticate to GCP
        ├── npm install
        ├── npm run build
        └── npm publish → Google Artifact Registry
                          (us-central1-npm.pkg.dev/zs-products/npm-packages/)
```

- Scoped package: `@zopsmart/zs-components`
- Published to **private Google Artifact Registry** (not public npm)
- Only `dist/` is included in the published package (`"files": ["dist"]`)

---

## 10. Responsive Design

- **Breakpoint**: 768px (`BREAKPOINTS.LARGE_MOBILE`)
- Media queries in styled-components
- Some components switch layout entirely (Table: table → cards on mobile)
- `useWindowSize` hook available for JS-level responsive logic
- `styled-breakpoints` library available for declarative breakpoints

---

## 11. Key Interview Questions You Should Be Ready For

### "How did you handle theming across multiple brands?"
> "Two-layer system: CSS custom properties as design tokens, styled-components consuming them. Brand themes override token values. Zero JS runtime cost for theme-switching — just load a different CSS file. 8 themes supported."

### "How is the library bundled and consumed?"
> "Microbundle (Rollup-based) outputs both CJS and ESM. ESM enables tree-shaking so consumers only bundle what they import. Source maps included for debugging. Published to private Google Artifact Registry via GitHub Actions."

### "How do you ensure consistency across 95+ components?"
> "Shared design tokens enforce visual consistency. Common primitives in `src/common/` (Button, Image, Grid, Text) are reused. Standard prop patterns across form components (label, isError, message, mandatory). Storybook serves as living documentation."

### "What's your testing strategy?"
> "Unit tests for complex interactive logic (Carousel). Storybook stories for every component with interaction testing addon. Chromatic for visual regression testing. a11y addon for accessibility audits."

### "Why styled-components over CSS Modules or Tailwind?"
> "styled-components let us co-locate styles with components, support dynamic prop-based styling, and avoid class name collisions — all critical for a library consumed by multiple apps. Combined with CSS variables, we get theming without runtime overhead."

### "How do you handle accessibility?"
> "Semantic HTML elements, aria-labels on form inputs, disabled state handling, keyboard support. Storybook a11y addon runs automated accessibility audits on every component."

### "Walk me through a component's architecture" (pick Button)
> "Button directory has `index.jsx` (logic + render) and `style.jsx` (styled-components). Supports variants (filled, textOnly, lowEmphasis, bordered), sizes (s/m/l), colors mapped to design tokens, optional left/right icons, disabled state. Uses prop-driven styling via helper functions that resolve CSS variable names."

---

## 12. Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                  Consumer App (React 17)                 │
│  import { Button, Table } from '@zopsmart/zs-components'│
│  import '@zopsmart/zs-components/dist/index.css'        │
│  import 'path/to/zopping-theme.css'                     │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│              src/index.js (Barrel Export)                │
│         118 named exports — components, hooks, utils     │
├─────────────┬──────────────┬──────────────┬─────────────┤
│ components/ │   common/    │    utils/    │    css/      │
│ 95 feature  │ 21 shared    │ ErrorBndry   │ tokens.css   │
│ components  │ primitives   │ debounce     │ 8 themes     │
│             │ (Button,     │ GUID         │              │
│  Each has:  │  Image,      │ phoneUtil    │              │
│  index.jsx  │  Grid, etc)  │ upload       │              │
│  style.jsx  │              │              │              │
│  stories.jsx│              │              │              │
└─────────────┴──────────────┴──────────────┴─────────────┘
         │                         │
         │    styled-components    │
         │    consume CSS vars     │
         ▼                         ▼
   var(--color-primary)    var(--size-spacing-m)
```

---

## 13. Key Dependencies & Why

| Dependency | Purpose | Why chosen |
|------------|---------|------------|
| `styled-components@5` | Component styling | Co-located styles, dynamic props, no class collisions |
| `@tanstack/react-table@8` | Advanced data tables | Headless, flexible, performant |
| `recharts@2` | Charts/Graphs | React-native charting, composable |
| `quill@1.3` | Rich text editing | Feature-rich WYSIWYG editor |
| `react-dates@21` | Date picking | Airbnb's battle-tested date picker |
| `moment@2` | Date manipulation | Used by react-dates (legacy dependency) |
| `react-google-recaptcha` | Bot protection | Google reCAPTCHA integration |
| `@react-google-maps/api` | Google Maps | React wrapper for Maps API |
| `phone@3` | Phone validation | International phone number parsing |
| `twemoji@13` | Cross-platform emoji | Consistent emoji rendering |
| `fast-equals@5` | Deep comparison | Performant object equality checks |

---

## 14. Areas for Improvement (Good to mention in interviews)

1. **TypeScript migration** — Currently pure JS; TS would add type safety and better DX for consumers
2. **Test coverage** — Only 13 test files for 95+ components; increase unit + integration tests
3. **Bundle size** — 3MB output could be optimized with code-splitting or per-component imports
4. **React version** — Peer dep on React 17; upgrading to React 18 would enable concurrent features
5. **moment.js** — Heavy dependency; migrate to date-fns or dayjs
6. **Accessibility** — Add ARIA live regions, focus management, keyboard navigation testing
7. **Documentation** — Add JSDoc/TSDoc for better IDE support
8. **Monorepo** — Consider splitting into packages (core, charts, forms) for independent versioning

These show you think critically about the codebase rather than just maintaining it.
