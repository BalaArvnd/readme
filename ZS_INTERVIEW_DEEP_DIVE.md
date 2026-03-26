# Interview Deep Dive — Advanced Topics

> Everything NOT covered in the previous guides. Performance, accessibility, complex components, Storybook, edge cases, and the hard questions interviewers ask SDE-2 candidates.

---

## TABLE OF CONTENTS

1. [Performance Optimization Patterns](#1-performance-optimization-patterns)
2. [Complex Component Deep Dives](#2-complex-component-deep-dives)
3. [Accessibility — What's Done & What's Missing](#3-accessibility)
4. [Error Handling & Validation Patterns](#4-error-handling--validation)
5. [Storybook Architecture & Story Patterns](#5-storybook-architecture)
6. [Third-Party Integrations](#6-third-party-integrations)
7. [Touch/Swipe & Animation Patterns](#7-touchswipe--animation-patterns)
8. [Prop Validation & Type Safety](#8-prop-validation--type-safety)
9. [Known Gaps & How You'd Fix Them](#9-known-gaps--how-youd-fix-them)
10. [Hard Interview Questions & Answers](#10-hard-interview-questions--answers)

---

## 1. PERFORMANCE OPTIMIZATION PATTERNS

### 1A. PureComponent (102 files!)

The **most widely used optimization**. All CMS edit containers and many shared components extend `React.PureComponent`:

```js
// Automatically does shallow comparison of props AND state
// Skips re-render if nothing changed
class EditCardsContainer extends React.PureComponent {
  // Only re-renders if props.data or state.data actually changed (shallow check)
}
```

**Where:** All edit containers (EditCardsContainer, EditHighlightsContainer, etc.), SlideCarousel controllers, Image, Bullets, RenderSliderChildren — **102 files total**.

**Interview talking point:** "We use PureComponent extensively in the CMS edit system because the containers manage complex state objects. PureComponent's shallow comparison prevents unnecessary re-renders when the parent re-renders but the data hasn't actually changed."

### 1B. React.memo (7 components)

Functional component equivalent of PureComponent:

```js
// TestimonialUI.js, CarouselUI.js, MapComponent
const TestimonialUI = React.memo(({ data, style }) => {
  // Only re-renders if data or style actually change
})
```

**Where:** TestimonialUI, TestimonialAdmin, CarouselUI, MapComponent (Google Maps), plus a few others.

### 1C. useCallback (11+ components)

Prevents child re-renders from handler recreation:

```js
// InfiniteScroll — memoizes the debounced scroll handler
const debounceHandleOnScroll = useCallback(debounce(handleOnScroll, 200), [])

// ImageSlider — memoizes resize handler
const resetCardSize = useCallback(() => {
  window.requestAnimationFrame(() => {
    // recalculate card dimensions
  })
}, [dependencies])
```

**Where:** InfiniteScroll, Pagination, ImageSlider (ContentSlider), VideoCarousel.

### 1D. useMemo (4+ components)

Caches expensive computed values:

```js
// Pagination — memoizes page options array
const pageOptions = useMemo(() => {
  return Array.from({ length: totalPages }, (_, i) => i + 1)
}, [totalPages])

// MonthPicker — memoizes month/year values
const currentMonth = useMemo(() => new Date().getMonth(), [])
const currentYear = useMemo(() => new Date().getFullYear(), [])
```

### 1E. Debouncing (2 implementations)

```js
// src/utils/debounce.js
export const debounce = (fn, delay) => {
  let timer
  return (...args) => {
    clearTimeout(timer)
    timer = setTimeout(() => fn(...args), delay)
  }
}
```

**Used in:**
- `InfiniteScroll` — debounces scroll event (200ms)
- `useWindowSize` — debounces resize event (300ms)

**Interview talking point:** "Scroll and resize are high-frequency events. Without debouncing, they'd fire 60+ times per second. We debounce to 200-300ms, reducing event processing by ~95% while keeping the UI responsive."

### 1F. requestAnimationFrame (6+ files)

Syncs DOM updates with browser paint cycle:

```js
// ImageSlider/ContentSlider
const resetCardSize = useCallback(() => {
  animationFrameRef.current = window.requestAnimationFrame(() => {
    // Recalculate card dimensions after browser paint
    if (cardRef.current) {
      setCardWidth(cardRef.current.offsetWidth)
    }
  })
}, [])

// Cleanup: cancelAnimationFrame on unmount
useEffect(() => {
  return () => cancelAnimationFrame(animationFrameRef.current)
}, [])
```

**Where:** ImageSlider (resize), FeatureNavigator (smooth scroll animation with easing function).

**FeatureNavigator smooth scroll:**
```js
// Custom easing function using requestAnimationFrame
const smoothScroll = (element, target, duration) => {
  const start = element.scrollLeft
  const change = target - start
  let startTime = null

  const animateScroll = (timestamp) => {
    if (!startTime) startTime = timestamp
    const elapsed = timestamp - startTime
    const progress = Math.min(elapsed / duration, 1)
    element.scrollLeft = start + change * easeInOutQuad(progress)
    if (elapsed < duration) requestAnimationFrame(animateScroll)
  }
  requestAnimationFrame(animateScroll)
}
```

### 1G. Table Page Caching

```js
// Table component caches fetched pages
const [fetchedData, setFetchedData] = useState({})

useEffect(() => {
  if (fetchedData[currentPage]) {
    // Use cached data — no network request
    setDisplayData(fetchedData[currentPage])
  } else {
    // Fetch from API, then cache
    fetch(`${api}?page=${page}&size=${size}`)
      .then(res => res.json())
      .then(data => {
        setFetchedData(prev => ({ ...prev, [currentPage]: data }))
        setDisplayData(data)
      })
  }
}, [currentPage])
```

### 1H. Ref-based State (no re-renders)

```js
// ImageSlider uses refs for animation state that shouldn't trigger re-renders
const actionRef = useRef(null)
const stopAnimationRef = useRef(false)
const timeoutRef = useRef(null)
const intervalRef = useRef(null)

// FeatureNavigator uses ref for active index tracking
const activeIndexRef = useRef(0)  // Changes don't trigger re-render
```

**Interview talking point:** "We use useRef for values that change frequently but shouldn't trigger re-renders — animation frame IDs, interval handles, scroll positions. This avoids the React reconciliation overhead for purely internal bookkeeping."

### Performance Summary Table:

| Technique | Count | Where Used |
|-----------|-------|------------|
| PureComponent | 102 files | All CMS edit containers, SlideCarousel internals |
| React.memo | 7 components | Testimonial, Carousel, Map |
| useCallback | 11+ components | InfiniteScroll, ImageSlider, VideoCarousel |
| useMemo | 4+ components | Pagination, MonthPicker |
| Debounce | 2 implementations | Scroll (200ms), Resize (300ms) |
| requestAnimationFrame | 6+ files | ImageSlider, FeatureNavigator |
| Page caching | 1 component | Table (server-side pagination) |
| Ref-based state | 5+ components | ImageSlider, FeatureNavigator |

---

## 2. COMPLEX COMPONENT DEEP DIVES

### 2A. SlideCarousel — The Most Complex Component

**Location:** `src/common/SlideCarousel/`

This is the most heavily tested component (8 test files). Here's why — it handles:

**Infinite Loop via Element Duplication:**
```
Original slides: [A, B, C, D]

Actual rendered:  [D', A, B, C, D, A']
                   ↑                ↑
              Cloned last      Cloned first

When user reaches cloned A' (end):
  → Instantly teleport to real A (no animation)
  → User perceives seamless infinite loop
```

**Touch/Swipe Implementation:**
```js
handleTouchStart(e) {
  startX = e.touches[0].clientX
  startY = e.touches[0].clientY
}

handleTouchMove(e) {
  // Track finger movement
}

handleTouchEnd(e) {
  endX = e.changedTouches[0].clientX
  const diff = startX - endX

  if (diff > threshold)      goToNext()     // Swipe left
  else if (diff < -threshold) goToPrevious() // Swipe right
}
```

**CSS Translation System:**
```js
// Position calculated as percentage
currentTranslationPercentage = -(activeIndex * 100 / totalSlides)

// Applied via inline style
transform: `translateX(${currentTranslationPercentage}%)`
transition: 'transform 0.3s ease-in-out'
```

**Controller System:**
```
SlideCarouselUI
  ├── RenderSliderChildren (PureComponent) — renders slides with duplication
  ├── CarouselControllers
  │   ├── LeftAngleBracket — previous button
  │   └── RightAngleBracket — next button
  ├── Bullets (PureComponent) — dot navigation
  ├── CustomControllerRender (PureComponent) — custom arrows
  └── CustomDotRender (PureComponent) — custom dots
```

### 2B. RichText Editor (Quill Integration)

**How Quill initializes:**
```js
// EditRichTextUI.js — componentDidMount
componentDidMount() {
  const quill = new Quill('#editor', {
    modules: {
      toolbar: [
        [{ size: ['12px', '16px', '24px', '32px'] }],
        ['bold', 'italic', 'underline'],
        [{ color: [] }],
        [{ align: [] }],
        [{ list: 'ordered' }, { list: 'bullet' }]
      ]
    },
    theme: 'snow'
  })

  // Real-time sync: Quill → React state → parent
  quill.on('text-change', () => {
    const html = document.querySelector('.ql-editor').innerHTML
    this.props.getInnerHTML(html)  // Propagate to container
  })
}
```

**Data flow:**
```
Quill editor (DOM-level editing)
  → 'text-change' event
    → Extract innerHTML from .ql-editor
      → getInnerHTML(html) callback
        → EditRichTextContainer.setState({ data: { html } })
          → componentDidUpdate → updateData(data) to parent
```

### 2C. PhoneNumber — Country Code + Emoji Flags

**Data flow:**
```
User types: "+91 9876543210"
  ↓
extractCountryCode("+91"):
  → Detects dial code: "91"
  → Looks up country: India
  → Gets emoji flag: 🇮🇳
  → Sets: { dialCode: "+91", countryCode: "IN", emoji: "🇮🇳" }
  ↓
isValidPhoneNumber("9876543210", "IN"):
  → Uses 'phone' npm library
  → Validates format for India
  → Returns: { isValid: true }
  ↓
getFormattedValue():
  → Returns: "+91 9876543210"
  → Callback to parent with formatted value
```

**Emoji rendering:** Uses `twemoji.parse()` to convert Unicode emoji to consistent cross-platform images via CDN.

### 2D. ImageSlider — Infinite Carousel with requestAnimationFrame

```
Slides: [1, 2, 3, 4]
Rendered: [4', 1, 2, 3, 4, 1']  (cloned endpoints)

User clicks "Next" on slide 4:
  1. CSS transition slides to cloned 1' (index 5)
  2. onTransitionEnd fires
  3. Instantly set transform to real slide 1 (index 1) — NO transition
  4. User sees seamless loop

Resize handling:
  window.resize → requestAnimationFrame → recalculate card widths
  Cleanup: cancelAnimationFrame on unmount
```

### 2E. Google Maps Integration

```
MapWrapper (Container — PureComponent)
  ├── State: markers[], infoWindows
  ├── Lifecycle: componentDidUpdate → updateData to parent
  └── Children:
      └── MapComponent (React.memo)
          ├── GoogleMap (from @react-google-maps/api)
          ├── DrawingManager (edit mode only)
          ├── Autocomplete (search location)
          └── Markers with InfoWindows
              ├── Click → toggle InfoWindow
              ├── Double-click → delete marker (edit mode)
              └── onBlur (InfoWindow) → save content changes
```

### 2F. FeatureNavigator — Smooth Scroll with Custom Easing

```js
// Custom easing: easeInOutQuad
const easeInOutQuad = (t) =>
  t < 0.5 ? 2 * t * t : -1 + (4 - 2 * t) * t

// Smooth scroll using requestAnimationFrame
const smoothScroll = (element, target, duration) => {
  const start = element.scrollLeft
  const change = target - start

  const animate = (timestamp) => {
    const elapsed = timestamp - startTime
    const progress = Math.min(elapsed / duration, 1)
    element.scrollLeft = start + change * easeInOutQuad(progress)
    if (elapsed < duration) requestAnimationFrame(animate)
  }
  requestAnimationFrame(animate)
}
```

### 2G. InfiniteScroll Hook

```js
// src/common/InfiniteScroll/index.js
const useInfiniteScroll = (fetchData, uniqueID) => {
  const [data, setData] = useState([])
  const [isLoading, setIsLoading] = useState(false)
  const [page, setPage] = useState(1)

  const handleOnScroll = () => {
    const { scrollTop, scrollHeight, clientHeight } = document.documentElement
    if (scrollTop + clientHeight >= scrollHeight) {
      // User reached bottom — load next page
      setPage(prev => prev + 1)
    }
  }

  // Debounced scroll listener
  const debouncedScroll = useCallback(debounce(handleOnScroll, 200), [])

  useEffect(() => {
    window.addEventListener('scroll', debouncedScroll)
    return () => window.removeEventListener('scroll', debouncedScroll)
  }, [])

  useEffect(() => {
    // Reset on uniqueID change (e.g., filter change)
    setData([])
    setPage(1)
  }, [uniqueID])

  useEffect(() => {
    setIsLoading(true)
    fetchData(page).then(newData => {
      setData(prev => [...prev, ...newData])
      setIsLoading(false)
    })
  }, [page])

  return [isLoading, data]
}
```

**Used by:** `InfiniteGrid` — combines InfiniteScroll + AutoGrid for paginated grid layouts.

---

## 3. ACCESSIBILITY

### What's Implemented:

| Feature | Status | Components |
|---------|--------|------------|
| `aria-label` | Partial | DynamicInputFields, ModalMolecule, Checkbox, CircularProgress |
| `aria-valuenow/min/max` | Done | CircularProgress (proper progressbar) |
| `role="progressbar"` | Done | CircularProgress |
| Semantic HTML | Good | `<button>`, `<input>`, `<label>` used correctly |
| Disabled states | Good | All form components: opacity 0.5, cursor not-allowed |
| Click-outside close | Good | Modal, Drawer, Calendar, Tabs, DropdownMenu |
| Body scroll lock | Good | Modal, Drawer |

### What's Missing (know these for interviews):

| Gap | Components Affected | How You'd Fix It |
|-----|-------------------|------------------|
| No `role="dialog"` + `aria-modal` | Modal, Drawer | Add to wrapper element |
| No focus trap | Modal, Drawer | Use `focus-trap-react` library or manual implementation |
| No focus restoration | Modal, Drawer | Save `document.activeElement` before open, restore on close |
| No `role="tablist/tab/tabpanel"` | Tabs | Add roles + `aria-selected` on active tab |
| No `aria-expanded` | Accordion, DropdownMenu | Add to trigger button |
| No `role="alert"` | Notification, StickyAlert | Add to container |
| No `aria-live` regions | Any dynamic content | Add `aria-live="polite"` for updates |
| No `aria-describedby` | Form error messages | Link error message to input via ID |
| No keyboard nav (arrow keys) | Tabs, DropdownMenu, Calendar | Add onKeyDown handlers |
| No Escape key to close | Modal, Drawer, DropdownMenu | Add keydown listener for Escape |
| No `<nav>` wrapper | BreadCrumbs | Wrap in `<nav aria-label="Breadcrumb">` |
| No i18n support | All text content | All strings are hardcoded English |

**Interview talking point (honest + solution-oriented):** "Accessibility is an area we've partially addressed. We have semantic HTML, aria-labels on form inputs, and proper ARIA on CircularProgress. But we're missing focus traps in modals, keyboard navigation in dropdowns and tabs, and aria-live regions for dynamic content. If I were to improve it, I'd start with Modal/Drawer focus traps using `focus-trap-react`, add `role='dialog'` + `aria-modal='true'`, and implement Escape key handlers across all overlay components."

---

## 4. ERROR HANDLING & VALIDATION

### Form Validation Pattern:

```js
// Pattern used across Form, DynamicInputFields, FileUpload
const handleChange = (event) => {
  const { name, value } = event.target

  switch (name) {
    case 'Name':
      if (!/^[a-z ,.'-]+$/i.test(value) && value !== '') {
        setError({ ...error, nameError: 'Invalid name' })
      } else {
        setError({ ...error, nameError: '' })
      }
      setFields({ ...fields, name: value })
      break

    case 'Email':
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value) && value !== '') {
        setError({ ...error, emailError: 'Invalid email' })
      } else {
        setError({ ...error, emailError: '' })
      }
      setFields({ ...fields, email: value })
      break

    case 'Phone':
      if (value.length !== 10) {
        setError({ ...error, phonenumberError: 'Must be 10 digits' })
      }
      break
  }
}
```

### DynamicInputFields — Smart Field Detection:

```js
// Auto-detects whether user is entering email or phone
const detectFieldType = (value) => {
  if (value.startsWith('+') || /^\d+$/.test(value)) {
    return 'phone'  // Render Phone component
  }
  return 'email'    // Render InputFields component
}
```

### FileUpload — Multi-Layer Validation:

```
Layer 1: Format check — fileFormats.includes(extension)
  → Invalid: setInvalidFileFormat(true), show "Invalid file format"

Layer 2: Size check — file.size < maxFileSizeInBytes
  → Too large: setSelectedFile({error: {fileSizeError: true}})

Layer 3: Upload check — xhr.onerror or non-200 status
  → Failed: setSelectedFile({error: {fileUploadError: true}})
  → Show retry button
```

### ErrorBoundary Pattern:

```js
class ErrorBoundary extends React.Component {
  state = { hasError: false }

  static getDerivedStateFromError(error) {
    return { hasError: true }
  }

  componentDidCatch(error, errorInfo) {
    // Can log to external service
    logErrorToMyService(error, errorInfo)
  }

  render() {
    if (this.state.hasError) return null  // Silent fail (no fallback UI)
    return this.props.children
  }
}
```

**Gap:** Returns `null` on error — should show fallback UI. Good interview point.

---

## 5. STORYBOOK ARCHITECTURE

### How Stories Are Structured:

Every component uses this pattern:

```jsx
// ComponentName/index.stories.jsx
import { ComponentName } from './index'
import { action } from '@storybook/addon-actions'

export default {
  title: 'Category/ComponentName',    // Sidebar hierarchy
  component: ComponentName,
  argTypes: {
    variant: { control: 'select', options: ['Line', 'Contained', 'Filled'] },
    color: { control: 'select', options: ['primary', 'secondary', 'error'] },
    size: { control: 'select', options: ['s', 'm', 'l'] },
    disabled: { control: 'boolean' },
  }
}

const Template = (args) => <ComponentName {...args} />

export const Default = Template.bind({})
Default.args = {
  text: 'Click me',
  variant: 'filled',
  color: 'primary',
  onClick: action('clicked')
}

export const Disabled = Template.bind({})
Disabled.args = {
  ...Default.args,
  disabled: true
}
```

### Theme Switching (Global Decorator):

```js
// .storybook/withTheme.decorator.js
const withTheme = (Story, context) => {
  const theme = context.globals.theme || 'default-light'

  useEffect(() => {
    document.documentElement.className = theme
  }, [theme])

  return <Story />
}

// .storybook/preview.js
export const decorators = [withTheme]

export const globalTypes = {
  theme: {
    toolbar: {
      items: [
        { value: 'default-light', title: 'Default Light' },
        { value: 'default-dark',  title: 'Default Dark' },
        { value: 'zopping',       title: 'Zopping' },
        { value: 'eazyupdates',   title: 'EazyUpdates' },
        // ... 4 more themes
      ]
    }
  }
}
```

### Interaction Testing (play function):

```jsx
// PersonCard/index.stories.jsx
import { within } from '@storybook/testing-library'
import { expect } from '@storybook/jest'

FullDetails.play = ({ canvasElement }) => {
  const canvas = within(canvasElement)
  expect(canvas.getByRole('link')).toHaveAttribute('href', 'https://zopsmart.com')
}
```

**This is a Storybook 7 feature** — tests run in the browser, visible in the Interactions panel.

### Webpack Alias (Important for how imports work in Storybook):

```js
// .storybook/webpack.config.js
module.exports = async ({ config }) => {
  config.resolve.alias = {
    ...config.resolve.alias,
    '@zopsmart/zs-components': path.resolve(__dirname, '../src')
  }
  config.resolve.fallback = {
    path: false,
    stream: false,
    fs: false,
    constants: require.resolve('constants-browserify')
  }
  return config
}
```

This means inside stories, `import { Button } from '@zopsmart/zs-components'` resolves to `src/` not `dist/` — so you get hot-reloading during development.

### Auto-Action Tracking:

```js
// preview.js
parameters: {
  actions: { argTypesRegex: '^on[A-Z].*' }
}
```

Any prop like `onClick`, `onChange`, `onClose` automatically shows in the Actions panel without manual `action()` imports.

---

## 6. THIRD-PARTY INTEGRATIONS

| Library | Component | How It's Used |
|---------|-----------|---------------|
| `@tanstack/react-table@8` | EnhanceTable | Headless table: sorting, filtering, pagination row models |
| `quill@1.3` | RichText | WYSIWYG editor, custom toolbar, text-change events for React sync |
| `recharts@2` | AreaChart, LineChart, BarGraph, PieChart, DoughnutChart | Wrapped with data transformation + custom tooltip/legend |
| `@react-google-maps/api` | Map, MapContactForm | DrawingManager, Autocomplete, Markers with InfoWindows |
| `react-google-recaptcha` | Recaptcha | Google reCAPTCHA v2 with theme/size props |
| `react-dates@21` | Calendar (partially) | Airbnb's date picker (legacy) |
| `moment@2` | Date components | Date manipulation (heavy — 67KB gzipped) |
| `phone@3` | PhoneNumber | International phone validation by country code |
| `twemoji@13` | PhoneNumber | Unicode emoji → cross-platform image rendering |
| `styled-breakpoints@4` | Grid components | Declarative responsive breakpoints in styled-components |
| `fast-equals@5` | Installed | Deep equality checks (available but minimally used) |

**Interview talking point:** "We wrap third-party libraries rather than exposing them directly. For example, EnhanceTable wraps TanStack React Table with auto-column generation and simplified props. AreaChart wraps Recharts with data transformation, custom tooltips, and our design token colors. This gives consumers a consistent API while we can swap underlying libraries without breaking changes."

---

## 7. TOUCH/SWIPE & ANIMATION PATTERNS

### SlideCarousel Touch Events:

```
touchstart → record startX, startY coordinates
touchmove  → (optional tracking)
touchend   → record endX, calculate diff

if (diff > 50px threshold)  → goToNext()
if (diff < -50px threshold) → goToPrevious()
```

### CSS Transition-based Animation:

```js
// SlideCarousel sliding animation
style={{
  transform: `translateX(${translationPercentage}%)`,
  transition: isAnimating ? 'transform 0.3s ease-in-out' : 'none'
}}
```

### Drawer Slide Animation:

```js
// Direction maps to initial transform
Left:   translateX(100%)   → translateX(0)   // Slide from right
Right:  translateX(-100%)  → translateX(0)   // Slide from left
Top:    translateY(100%)   → translateY(0)   // Slide from bottom
Bottom: translateY(-100%)  → translateY(0)   // Slide from top

// CSS transition handles the animation
transition: transform 0.3s ease-out
```

### FeatureNavigator Scroll Animation:

Uses `requestAnimationFrame` with `easeInOutQuad` easing function for smooth programmatic scrolling.

---

## 8. PROP VALIDATION & TYPE SAFETY

### PropTypes Patterns Used:

```js
// Basic types
Button.propTypes = {
  text: PropTypes.string,
  onClick: PropTypes.func,
  disabled: PropTypes.bool,
}

// Constrained values (enum)
Tabs.propTypes = {
  variants: PropTypes.oneOf(['Line', 'Contained', 'Filled', 'WithBorder']).isRequired,
  borderStyle: PropTypes.oneOf(['Bottom', 'Right', 'Top', 'Left']),
}

// Union types
Calendar.propTypes = {
  date: PropTypes.oneOfType([PropTypes.string, PropTypes.array]),
}

// Required props
Rating.propTypes = {
  value: PropTypes.number.isRequired,
  onChange: PropTypes.func.isRequired,
}

// Array/object (generic — not shape-validated)
Checkbox.propTypes = {
  options: PropTypes.array.isRequired,
}
```

### DefaultProps Pattern:

```js
Recaptcha.defaultProps = {
  theme: 'light',
  _reCaptchaKey: '6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI'  // Test key
}
```

### Gaps in Type Safety:

| Gap | Example | Better Approach |
|-----|---------|-----------------|
| Generic `PropTypes.array` | Checkbox options | `PropTypes.arrayOf(PropTypes.shape({ value: PropTypes.string, label: PropTypes.string }))` |
| No TypeScript | Entire project | Migrate to TS for compile-time safety + consumer autocomplete |
| Missing `isRequired` | Most Button props | Mark critical props as required |
| Hardcoded default data | Popover defaults to "Dilip walia" | Use null/empty string defaults |

---

## 9. KNOWN GAPS & HOW YOU'D FIX THEM

This is **the most important section for interviews**. Interviewers LOVE when you can critique your own codebase constructively.

### Gap 1: No TypeScript
**Problem:** Consumers get no autocomplete, no compile-time type checking.
**Fix:** Incremental migration — start by adding `.d.ts` declaration files for the most-used components. Then gradually convert `.jsx` → `.tsx`.

### Gap 2: 3MB Bundle Size
**Problem:** Importing one component loads the entire 3MB bundle.
**Fix:**
- Enable per-component imports: `import Button from '@zopsmart/zs-components/Button'`
- Or use a monorepo tool (Turborepo) to split into `@zopsmart/core`, `@zopsmart/charts`, `@zopsmart/forms`
- Add `sideEffects: false` to package.json for better tree-shaking

### Gap 3: moment.js (67KB gzipped)
**Problem:** Heavy dependency used only for date formatting.
**Fix:** Migrate to `dayjs` (2KB) — API-compatible drop-in replacement.

### Gap 4: No Focus Traps in Modals
**Problem:** Users can Tab out of Modal/Drawer into background content.
**Fix:** Add `focus-trap-react` wrapper or manual implementation cycling focus within the modal.

### Gap 5: Class Components in Edit System
**Problem:** Edit containers use class components with lifecycle methods — harder to compose, test, and maintain.
**Fix:** Migrate to hooks. Replace `componentDidMount/Update/WillUnmount` with `useEffect`. Replace `React.cloneElement` with Context or render props.

### Gap 6: Deep Cloning with JSON.parse(JSON.stringify())
**Problem:** Loses Date objects, undefined values, functions, circular references. Slow for large objects.
**Fix:** Use `structuredClone()` (native) or `immer` for immutable updates.

### Gap 7: dangerouslySetInnerHTML without Sanitization
**Problem:** Used in EditText, RichText, Tooltip — potential XSS vector.
**Fix:** Add `DOMPurify.sanitize()` before rendering HTML content.

### Gap 8: No Error Boundary Fallback UI
**Problem:** ErrorBoundary renders `null` — user sees blank screen.
**Fix:** Add a proper fallback UI: "Something went wrong. Please refresh."

### Gap 9: Low Test Coverage
**Problem:** 13 test files for 95+ components.
**Fix:** Prioritize testing complex interactive components. Use Storybook interaction tests (`play()`) for visual components. Add React Testing Library for unit tests.

### Gap 10: React 17 Peer Dependency
**Problem:** Can't use React 18 features (concurrent rendering, useTransition, Suspense for data fetching).
**Fix:** Update peer dep to `react@^18.0.0`, test all components, leverage automatic batching.

---

## 10. HARD INTERVIEW QUESTIONS & ANSWERS

### "Why not use Context API for theming instead of CSS variables?"

> "CSS variables cascade through the DOM automatically — no React re-renders needed. Context would force every styled-component to subscribe and re-render on theme change. With CSS variables, theme switching is literally changing one class on `<html>` — the browser handles the rest in a single layout/paint pass. Zero JavaScript cost."

### "How would you make this library tree-shakeable?"

> "The ESM output already enables tree-shaking. But the barrel export (`index.js` re-exporting everything) can defeat it if bundlers can't statically analyze the exports. I'd add `sideEffects: false` to package.json, ensure no top-level side effects in component files, and consider offering direct imports like `@zopsmart/zs-components/Button`."

### "How would you handle breaking changes when updating a component?"

> "Semantic versioning. Breaking changes get a major version bump. I'd use a codemod for automated migration where possible. For the transition period, I'd support both APIs with a deprecation warning in the console, then remove the old API in the next major version."

### "Why PureComponent everywhere in the edit system instead of React.memo?"

> "The edit system was built with class components because it predates widespread hooks adoption, and class lifecycle methods (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`) were needed for the three-point sync pattern with the parent. PureComponent was the natural optimization for class components. If rewriting today, I'd use functional components with `React.memo` and `useEffect`."

### "Your Table component fetches data inside itself. Is that a good pattern for a library?"

> "For a self-contained demo, it works. But for a production library, I'd prefer the headless approach — like what we did with EnhanceTable wrapping TanStack React Table. The library should handle rendering and UI logic; data fetching should be the consumer's responsibility. This follows the separation of concerns principle and makes the component testable without network mocking."

### "How do you handle version conflicts when multiple apps use different versions?"

> "Since we publish to a private registry (Google Artifact Registry), each consuming app pins its own version. We use semantic versioning and a CHANGELOG. For gradual rollout, we'd publish a beta channel (`2.4.0-beta.1`) and let teams opt in before promoting to stable."

### "What would you do differently if starting this library from scratch?"

> "Five things: (1) TypeScript from day one for consumer DX. (2) Monorepo structure — split into @zopsmart/core, @zopsmart/charts, @zopsmart/forms for independent versioning and smaller bundles. (3) Replace moment.js with dayjs. (4) Build with Vite instead of Microbundle for faster dev experience. (5) Full accessibility compliance from the start — focus traps, ARIA roles, keyboard navigation."

### "How would you measure and monitor component performance?"

> "Three levels: (1) Dev-time — React DevTools Profiler to identify unnecessary re-renders, `why-did-you-render` library for automated detection. (2) Build-time — Bundle analyzer to track size per component, Lighthouse CI in the pipeline. (3) Runtime — Web Vitals (CLS, LCP, FID) monitoring in consuming apps, with component-level attribution."

### "Explain the trade-off between styled-components and CSS Modules for a component library."

> "styled-components: Dynamic styling via props, guaranteed no class conflicts (generated unique names), co-located with components. Downside: runtime CSS-in-JS overhead (~12KB runtime), harder to extract critical CSS for SSR. CSS Modules: Zero runtime cost, native CSS, works great with SSR. Downside: Can't do dynamic prop-based styling without inline styles, class names need careful scoping for library consumers. We chose styled-components because dynamic prop-based styling (variant, color, size) is core to our component API, and the runtime cost is acceptable for our use case."

### "How do you ensure no regressions when changing a shared component used by 6 products?"

> "Three safety nets: (1) Storybook stories for every component — visual check. (2) Chromatic for automated visual regression testing — screenshot diffing on every PR. (3) Semantic versioning — breaking changes only in major versions. Consuming apps pin versions and upgrade deliberately. We also have Storybook interaction tests for critical user flows."
