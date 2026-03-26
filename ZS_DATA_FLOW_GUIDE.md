# Complete Data Flow & Working Model — `@zopsmart/zs-components`

> Interview-ready guide for SDE-2. Covers how every piece connects, from consumer app to individual pixel.

---

## TABLE OF CONTENTS

1. [The Big Picture — How a Consumer App Uses This Library](#1-the-big-picture)
2. [Entry Point — The Barrel Export](#2-entry-point--the-barrel-export)
3. [The Theming Pipeline — How CSS Tokens Flow](#3-the-theming-pipeline)
4. [The Styling Pipeline — How styled-components Consume Tokens](#4-the-styling-pipeline)
5. [Component Data Flow Patterns (5 Categories)](#5-component-data-flow-patterns)
6. [Deep Dive: Every Major Component's Data Flow](#6-deep-dive-every-major-components-data-flow)
7. [The CMS Edit Pattern — Dual-Mode Architecture](#7-the-cms-edit-pattern)
8. [Hooks & Utilities — Shared Logic](#8-hooks--utilities)
9. [Build Pipeline — Source to dist/](#9-build-pipeline)
10. [Complete Flow Diagram — Request to Render](#10-complete-flow-diagram)

---

## 1. THE BIG PICTURE

Here's what happens when a Zopsmart product (e.g., Zopping admin panel) uses this library:

```
┌─────────────────────────────────────────────────────────────┐
│                    CONSUMER APP (e.g., Zopping)              │
│                                                              │
│  // Step 1: Import the component                             │
│  import { Button, Table, Modal } from '@zopsmart/zs-components'│
│                                                              │
│  // Step 2: Import base CSS tokens                           │
│  import '@zopsmart/zs-components/dist/index.css'             │
│                                                              │
│  // Step 3: Import brand theme (overrides token values)      │
│  import '@zopsmart/zs-components/src/css/zopping.css'        │
│                                                              │
│  // Step 4: Set theme class on <html>                        │
│  document.documentElement.className = 'zopping'              │
│                                                              │
│  // Step 5: Use components — pass data DOWN, receive         │
│  //         callbacks UP                                     │
│  <Button variant="filled" color="primary" onClick={handler}  │
│          text="Save" size="m" />                             │
└─────────────────────────────────────────────────────────────┘
```

**The fundamental contract:**
- **Data flows DOWN** via props (parent → component)
- **Events flow UP** via callbacks (component → parent)
- **Styling flows THROUGH** CSS variables (tokens → theme override → styled-component)
- **The library owns ZERO global state** — the consumer app owns all state

---

## 2. ENTRY POINT — THE BARREL EXPORT

**File:** `src/index.js` (126 lines)

This is a **barrel export** — a single file that re-exports everything from 95+ component directories. When a consumer writes:

```js
import { Button, Table, Modal } from '@zopsmart/zs-components'
```

Webpack/Rollup resolves this to `dist/index.modern.js` (ESM) which traces back to `src/index.js`.

**What gets exported:**

```
src/index.js
  ├── 106 Components (Button, Table, Modal, InputFields, etc.)
  ├── 2 Hooks (useWindowSize, useAuth)
  ├── CSS side-effects via: export * from './css/index'
  └── That's it. No providers, no context, no store.
```

**The CSS side-effect import** (`src/css/index.js`) is critical:

```js
// src/css/index.js
import './tokens.css'                              // Base design tokens
import '../components/Circular_Progress/Circular_Progress.css'  // Component CSS
import '../components/Modal/custom.css'            // Component CSS
import '../components/InputFields/custom.css'      // Component CSS
// ... 35 more component CSS imports
```

This means: **when you import ANY component, ALL CSS gets loaded**. The tokens and component-specific CSS are bundled into `dist/index.css`.

---

## 3. THE THEMING PIPELINE

This is the most architecturally interesting part. Here's exactly how a color goes from a CSS file to a rendered pixel:

### Layer 1: Base Tokens (`src/css/tokens.css`)

Defines ALL design variables on `:root` (the `<html>` element):

```css
:root {
  /* Atomic color palette */
  --color-purple-500: #414eff;
  --color-red-500: #ef4444;
  --color-gray-50: #fafafa;
  --color-gray-900: #212121;

  /* Semantic tokens (reference atomic tokens) */
  --color-primary: var(--color-purple-500);
  --color-error: var(--color-red-500);
  --color-text-primary: var(--color-gray-900);

  /* Spacing scale */
  --size-spacing-xs: 0.5rem;
  --size-spacing-m: 1rem;
  --size-spacing-xl: 1.5rem;

  /* Typography scale */
  --size-font-s: 0.75rem;
  --size-font-m: 0.875rem;
  --size-font-l: 1rem;

  /* Shadows */
  --shadow-xs: 0px 1px 2px rgba(34,34,34,0.06);
  --shadow-xl: 0px 16px 32px rgba(34,34,34,0.12);

  /* Border radius */
  --size-border-radius-s: 4px;
  --size-border-radius-m: 8px;
}
```

### Layer 2: Theme Overrides (e.g., `src/css/zopping.css`)

Each brand theme targets a CSS class on `<html>` and overrides specific tokens:

```css
/* zopping.css */
html.zopping {
  --color-primary: #bf1139;           /* Red instead of purple */
  --color-primary-light: #fce4ec;
  --color-primary-dark: #880e2b;
  --color-secondary: #1a237e;
  /* Gray scale stays the same — inherited from tokens.css */
}

/* default-dark.css — inverts the ENTIRE palette */
html.default-dark {
  --color-gray-50: #1a1a1d;          /* Was light, now dark */
  --color-gray-900: #fcfcfc;         /* Was dark, now light */
  --color-white: #000;               /* Full inversion */
  --color-black: #fff;
}
```

### Layer 3: Component Custom Tokens (e.g., `components/Avatar/custom.css`)

Some components extend the token system with component-specific variables:

```css
/* Avatar/custom.css */
html {
  --size-avatar-m: 2.5rem;
  --size-avatar-l: 3rem;
  --size-avatar-huge: 5rem;
}

/* DashboardCard/dashboard_card.css */
html {
  --color-dashboardCard-border: var(--color-gray-200);
  --dashboardCard-min-height: 120px;
}
```

### The Complete Cascade:

```
tokens.css (:root)          →  --color-primary: #414eff (purple)
    ↓
zopping.css (html.zopping)  →  --color-primary: #bf1139 (red)    ← OVERRIDES
    ↓
Avatar/custom.css (html)    →  --size-avatar-m: 2.5rem           ← EXTENDS
    ↓
Component styled-component  →  background: var(--color-primary)  ← CONSUMES
    ↓
Browser renders             →  background: #bf1139               ← RESOLVED
```

### How Theme Switching Works (Storybook):

```
Storybook toolbar → User selects "Zopping"
    ↓
withTheme.decorator.js:
    useEffect(() => {
      document.documentElement.className = theme || 'default-light'
    }, [theme])
    ↓
<html class="zopping">  ← CSS class changes
    ↓
CSS cascade recalculates:
    html.zopping { --color-primary: #bf1139 } now applies
    ↓
All styled-components using var(--color-primary) instantly update
    ↓
Zero JavaScript re-renders needed. Pure CSS cascade.
```

**Interview talking point:** "Theme switching has zero JS runtime cost. We swap a CSS class on `<html>`, and the browser's CSS cascade handles the rest. No React re-renders, no context updates, no prop drilling."

---

## 4. THE STYLING PIPELINE — How styled-components Consume Tokens

Every component has two files:
- `index.jsx` — Logic, state, render
- `style.jsx` — styled-components definitions

Here's how props flow into styles using **Button** as the canonical example:

### Step 1: Parent passes props

```jsx
<Button variant="lowEmphasis" color="primary" size="m" text="Save" onClick={fn} />
```

### Step 2: Component passes props to styled-component

```jsx
// Button/index.jsx
const Button = ({ text, icon, onClick, disabled, type, ...rest }) => {
  return (
    <Wrapper               // ← styled-component
      onClick={onClick}
      disabled={disabled}
      type={type}
      {...rest}            // ← variant, color, size forwarded here
    >
      {icon && <StyledImage src={icon} />}
      {text}
    </Wrapper>
  )
}
```

### Step 3: styled-component resolves props to CSS variables

```jsx
// Button/style.jsx
const COLOR_MAP = {
  primary: '--color-primary',
  secondary: '--color-secondary',
  error: '--color-error',
  warning: '--color-warning',
  success: '--color-success',
}

const getButtonBackgroundColor = (variant, color) => {
  switch (variant) {
    case 'textOnly':      return 'transparent'
    case 'lowEmphasis':   return `var(${COLOR_MAP[color]}-light)`  // var(--color-primary-light)
    case 'bordered':      return 'transparent'
    default:              return `var(${COLOR_MAP[color]})`        // var(--color-primary)
  }
}

export const Wrapper = styled.button`
  background-color: ${({ color, variant }) => getButtonBackgroundColor(variant, color)};
  color: ${({ color, variant }) => getButtonFontColor(variant, color)};
  font-size: var(--size-font-${({ size }) => SIZE_OPTIONS[size]});
  padding: ${({ size }) => `var(--size-spacing-${SPACING_OPTIONS[size]})`};
  border-radius: var(--button-radius);
  font-weight: var(--font-weight-m);
  cursor: ${({ disabled }) => disabled ? 'not-allowed' : 'pointer'};
  opacity: ${({ disabled }) => disabled ? 0.5 : 1};
  flex-direction: ${({ iconPosition }) => iconPosition === 'end' ? 'row-reverse' : 'row'};
`
```

### Step 4: Browser resolves CSS variables

```
styled-component output:
  background-color: var(--color-primary-light)

CSS cascade resolves (with zopping theme):
  --color-primary-light: #fce4ec

Final rendered CSS:
  background-color: #fce4ec
```

**This pattern is consistent across ALL 95+ components.**

---

## 5. COMPONENT DATA FLOW PATTERNS

Every component in this library follows one of these 5 patterns:

### Pattern A: Stateless / Presentational (Button, Chip, Divider, Avatar, Badge)

```
Props IN → Render → Done. No internal state.

Parent                          Component
  │                               │
  ├─ text="Save"    ──────────→   │ Render <button>Save</button>
  ├─ variant="filled" ────────→   │ Apply filled styling
  ├─ onClick={fn}    ─────────→   │ Attach to DOM onClick
  │                               │
  │  (User clicks)                │
  │  ←─── onClick() ─────────────┤  Event bubbles up
```

**Components:** Button, Chip, Divider, Avatar, Badge, BreadCrumbs, Stepper, ProgressBar, CircularProgressBar, Skeleton

---

### Pattern B: Controlled Inputs (InputFields, Checkbox, Radio, TextArea, Switch)

```
Parent OWNS the value. Component just renders + reports changes.

Parent                          Component
  │                               │
  ├─ value={state}  ───────────→  │ Render input with value
  ├─ onChange={fn}  ───────────→  │ Attach to input onChange
  ├─ isError={bool} ───────────→  │ Show red border + error message
  ├─ message="Required" ──────→  │ Show helper text
  │                               │
  │  (User types "hello")        │
  │  ←─── onChange(event) ────────┤  Parent updates state
  │                               │
  ├─ value="hello"  ───────────→  │ Re-render with new value
```

**Components:** InputFields, TextArea, Checkbox, Radio, Switch, SelectInputField, Phone, DynamicInput

**Key detail for Checkbox (multi-select):**
```
Parent: selectedOption = ['opt1', 'opt3']
  ↓
Checkbox renders options, checks: selectedOption.includes(option.value)
  ↓
User clicks opt2 → handleOnClick(event) → parent adds 'opt2'
  ↓
Parent: selectedOption = ['opt1', 'opt3', 'opt2'] → re-render
```

**Key detail for Radio (single-select):**
```
Parent: selectedOption = 'opt1'
  ↓
Radio renders options, checks: selectedOption === option.value
  ↓
User clicks opt2 → onChange('opt2') → parent replaces value
  ↓
Parent: selectedOption = 'opt2' → re-render
```

---

### Pattern C: Self-Managed + Callback (Accordion, Tabs, Calendar, Tooltip, Rating)

```
Component manages UI state internally, notifies parent on meaningful changes.

Parent                          Component
  │                               │
  ├─ options=[...]  ───────────→  │ Render tabs/accordion/calendar
  ├─ onClick={fn}   ───────────→  │ Store callback
  │                               │
  │                               │ (internal state: activeTab=0)
  │  (User clicks tab 2)         │
  │                               │ setActiveTab(2)  ← internal
  │  ←─── onClick(2) ────────────┤  Notify parent
  │                               │
  │  (Parent can use index 2     │
  │   to show tab 2 content)     │
```

**Components:** Accordion, Tabs, Calendar, Tooltip, Rating, MonthPicker, TimePicker, Filter

**Key detail for Calendar (range picker):**
```
multipicker=true

Click 1: selectedDates = [Mar 5]
Click 2: selectedDates = [Mar 5, Mar 6, Mar 7, Mar 8, Mar 9, Mar 10]
         ↑ Auto-fills every day between click 1 and click 2
Click 3: Starts new range — selectedDates = [Mar 15]

useEffect([selectedDates]) → onDateClick(convertedDates)  // callback to parent
```

**Key detail for Tooltip (position calculation):**
```
User hovers → setIsOpen(true) →
  useEffect calculates position:
    childRect = childRef.getBoundingClientRect()
    tooltipRect = tooltipRef dimensions

    if placement='top':
      top = child.offsetTop - tooltip.height - 9px
      left = child.offsetLeft + (child.width - tooltip.width) / 2

    if placement='bottom':
      top = child.offsetTop + child.height + 9px
      left = same centering logic

  → setPosition({top, left})
  → Tooltip renders at absolute position
```

---

### Pattern D: Self-Managed + Side Effects (Modal, Drawer, FileUpload, Table)

```
Component manages complex internal state + DOM side effects.

Parent                          Component
  │                               │
  ├─ show={true}   ────────────→  │ Render overlay
  ├─ onClose={fn}  ────────────→  │ Store callback
  │                               │
  │                               │ useEffect: body.overflow='hidden'
  │                               │ useEffect: addEventListener('click')
  │                               │
  │  (User clicks outside)       │
  │                               │ if (!ref.contains(target))
  │  ←─── onClose() ─────────────┤  Notify parent to set show=false
  │                               │
  │                               │ cleanup: body.overflow='auto'
  │                               │ cleanup: removeEventListener
```

**Components:** Modal, Drawer, FileUpload, Table (with API mode), EnhanceTable

**Key detail for FileUpload (XHR data flow):**
```
1. User selects file (input onChange OR drag-drop)
     ↓
2. Validate format: fileFormats.includes(extension)
   Validate size: file.size < maxFileSizeInBytes
     ↓
3. Create FormData, append file with formDataKey
     ↓
4. Create XMLHttpRequest
   Set custom headers
   xhr.open('POST', apiUrl)
     ↓
5. xhr.upload.onprogress → setUploadProgress((loaded*100)/total)
   ProgressBar component re-renders: 0% → 25% → 50% → 75% → 100%
     ↓
6. xhr.onload (200-299) → setSuccessMessage(true)
                         → callBackFunction(JSON.parse(response))
                         → Parent receives file URL
     ↓
7. xhr.onerror → setSelectedFile({error: {fileUploadError: true}})
               → Show retry button
     ↓
8. Parent can reset via isFileRemoved prop → clears all upload state
```

**Key detail for Table (dual-mode data flow):**

```
MODE 1: Client-Side Pagination (no api prop)
─────────────────────────────────────────────
data = [{name:'A', age:25}, {name:'B', age:30}, ...]
countPerPage = 10

Render: data.slice(currentPage * 10, (currentPage+1) * 10)
  ↓
User clicks "Next" → setCurrentPage(1)
  ↓
Re-slice: data.slice(10, 20) → show rows 11-20


MODE 2: Server-Side Pagination (with api prop)
─────────────────────────────────────────────
api = "https://api.example.com/users"

useEffect([currentPage]):
  fetch(`${api}?page=${currentPage * countPerPage}&size=${countPerPage}`)
    ↓
  Response: { data: [...], totalPassengers: 500 }
    ↓
  setDisplayData(response.data)
  setTotalCount(response.totalPassengers)
    ↓
  Cache: fetchedData[currentPage] = response.data  // avoid re-fetch
    ↓
User clicks "Next" → setCurrentPage(1) → fetch page 2
  ↓
If already cached → use fetchedData[1] instead of fetching
```

**Key detail for Table (sorting):**
```
User clicks column header "Age"
  ↓
sortData('age'):
  if sortConfig.key === 'age' && sortConfig.direction === 'asc':
    direction = 'desc'
  else:
    direction = 'asc'

  sorted = [...displayData].sort((a, b) => {
    if (a['age'] < b['age']) return direction === 'asc' ? -1 : 1
    if (a['age'] > b['age']) return direction === 'asc' ? 1 : -1
    return 0
  })

  setDisplayData(sorted)
  setSortConfig({ key: 'age', direction })
```

**Key detail for Table (responsive):**
```
window.innerWidth <= 768?
  YES → Render <CardContainer> with card layout per row
  NO  → Render <table> with <thead>/<tbody>/<tr>/<td>
```

---

### Pattern E: Container-UI-Toggle (CMS Edit Components)

```
A dual-mode architecture where the same data renders as either
read-only display OR editable admin interface.

This is covered in detail in Section 7 below.
```

---

## 6. DEEP DIVE: EVERY MAJOR COMPONENT'S DATA FLOW

### Button
```
Props: text, icon, onClick, disabled, type, size, variant, color, iconPosition
State: NONE
Flow:  Props → styled Wrapper → render
       onClick prop attached directly to DOM
```

### InputFields
```
Props: value, onChange, label, placeholder, isError, message, mandatory,
       disabled, variant (OUTLINED/STANDARD), backGround (STANDARD/FILLED),
       leftIcon, rightIcon, onClick, onClear, borderRadius
State: NONE (fully controlled)
Flow:  value → <input> → user types → onChange(event) → parent updates value
       isError → red border + Caption shows message
       variant → isBorder/isBottomBorder flags → styled InputSection
       leftIcon click → onClick(), rightIcon click → onClear() ?? onClick()
```

### Checkbox
```
Props: options=[{value,label,disabled}], selectedOption=[], handleOnClick,
       label, size, mandatory, disableGroup
State: NONE (fully controlled)
Flow:  options.map → render each checkbox
       checked = selectedOption.includes(option.value)
       click → handleOnClick(event) → parent toggles value in array
```

### Radio
```
Props: options=[{value,label,disabled}], selectedOption='', onChange,
       label, size, row, mandatory, disableGroup
State: NONE (fully controlled)
Flow:  options.map → render each radio
       checked = selectedOption === option.value
       click → onChange(option.value) → parent sets single value
       Hidden <input type="radio"> for native behavior
       Visible <RadioIndicator> for custom styling
```

### Modal
```
Props: children, onClose, closeButton
State: NONE (parent controls visibility by mounting/unmounting)
Flow:  Mount → useEffect: body.overflow='hidden'
       Render: ModalBackground → ModalWrapper → {children}
       Click close button → onClose()
       Click background (e.target === e.currentTarget) → onClose()
       Unmount → cleanup: body.overflow='auto'
```

### Drawer
```
Props: show, direction (Left/Right/Top/Bottom), children, onClose
State: NONE
Flow:  show=true → render SideDrawer with CSS transform animation
       direction → maps to translateX/translateY for slide-in effect
       Click outside drawerRef → onClose()
       Desktop: 20-35% width | Mobile: 40-60% width
```

### Accordion
```
Props: title, children, upIcon, downIcon, openByDefault
State: isOpen (boolean)
Flow:  openByDefault → useEffect → setIsOpen(openByDefault)
       Click header → setIsOpen(!isOpen)
       isOpen=true → render AccordionContent with {children}
       isOpen=false → hide content
       Custom icons: upIcon/downIcon or default ▲/▼
```

### Tabs
```
Props: options=['Tab1','Tab2'], variants (Line/Contained/Filled/WithBorder),
       onClick, borderStyle, children
State: activeTab (number), isEditing (boolean), editedOptions (array)
Flow:  Click tab → setActiveTab(index) → onClick(index) to parent
       Double-click tab → setIsEditing(true) → show input for rename
       "Remove" button → splice tab from editedOptions
       "+ Add New Tab" → push 'New Tab' to editedOptions, auto-edit
       Click outside → setIsEditing(false)
       Styled: active tab gets highlight (Line/Contained/Filled style)
```

### Rating
```
Props: value (1-5), onChange, size
State: NONE (fully controlled)
Flow:  Render 5 stars: [0,1,2,3,4].map
       Star filled if index < value, empty otherwise
       Click star at index → onChange(index + 1)  // 1-indexed
```

### Tooltip
```
Props: text, placement (top/bottom), arrowPosition, children,
       primaryButton, secondaryButton
State: isOpen (boolean), position ({top,left}), arrowCalibri
Flow:  onMouseEnter → setIsOpen(true)
       useEffect([isOpen]) → calculatePosition using getBoundingClientRect()
       Render tooltip at absolute {top, left} position
       onMouseLeave → setIsOpen(false)
       Text rendered via dangerouslySetInnerHTML (supports HTML)
```

### Calendar
```
Props: date, multipicker, variant (Clocked), footer,
       onDateClick, onTimeChange, onClickCancel, onClickContinue, onClose, time
State: selectedDate, selectedDates[], selectedTime, selectedYear,
       showDatePicker, showYearPicker
Flow:  Single mode: click date → toggle in selectedDates
       Range mode: click 1 = start, click 2 = end, auto-fill between
       useEffect([selectedDates]) → onDateClick(dates) to parent
       Month nav: prev/next arrows → shift selectedDate by 1 month
       Year picker: click header → show year grid → click year → update
       Time picker: variant='Clocked' → render TimePicker → onTimeChange
       Click outside calendarRef → onClose()
```

### FileUpload
```
Props: text, label, apiUrl, maxFileSizeInBytes, fileFormats,
       formDataKey, callBackFunction, headers, isFileRemoved, mandatory
State: selectedFile, uploadProgress (0-100), isUploading, successMessage,
       invalidFileFormat, xhrRequest
Flow:  Select file (click or drag-drop)
       → validate format → validate size
       → create FormData + XMLHttpRequest
       → xhr.upload.onprogress → setUploadProgress(%)
       → xhr.onload → callBackFunction(response) to parent
       → xhr.onerror → show error + retry button
       Parent resets via isFileRemoved prop change
```

### Table
```
Props: data[], rowOrder[], countPerPage, api, totalDataCount,
       handleApiCall, customRowHandler
State: currentPage, displayData, sortConfig, isLoading, tableHeaders,
       fetchedData (cache)
Flow:  Client mode: slice data by page → render
       Server mode: fetch from api per page → cache → render
       Sort: click header → sort displayData
       Responsive: <=768px → card layout, >768px → table layout
       Custom render: customRowHandler(rowData) for each row
```

### EnhanceTable (TanStack React Table v8)
```
Props: data[], columns[], isPagination, border
State: sorting, filtering, tableColumns (auto-generated if not provided)
Flow:  Auto-generate columns from data[0] keys if columns not provided
       useReactTable({data, columns, getCoreRowModel, getPaginationRowModel,
                      getSortedRowModel, getFilteredRowModel})
       Header click → getToggleSortingHandler() → null→asc→desc→null
       Pagination: <<, <, >, >> buttons → table.setPageIndex()
       Cell render: flexRender(cell.column.columnDef.cell, cell.getContext())
```

### Stepper
```
Props: type (horizontal/vertical), stepperInfo=[{title, status, desc}]
State: NONE
Flow:  Map stepperInfo → render each step with icon based on status:
         'inactive' → gray circle
         'active'   → blue dot
         'completed' → checkmark
       Horizontal: steps in a row, ::after pseudo-element for connector line
       Vertical: steps stacked, border-left for connector line
       Line color: completed → var(--color-primary), else → var(--color-gray-400)
```

### Form (example composite component)
```
Props: NONE (self-contained demo)
State: fields{name,email,phone,password,radioValue,textArea,dateLocalTime,
              fileUrl,rating}, error{...Errors}, selectedRadio, rating
Flow:  Each child component gets a slice of state:
         <InputFields value={fields.name} onChange={handleChange} isError={error.nameError} />
         <Radio selectedOption={selectedRadio} onChange={changeRadio} />
         <Rating value={rating} onChange={handleRating} />
         <FileUpload callBackFunction={fileUploadResult} />

       handleChange(event):
         switch(event.target.name):
           'Name' → validate regex → setFields({name: value}) + setError
           'Email' → validate regex → setFields({email: value}) + setError

       Submit → validate all → if errors, show messages → if valid, reset all
```

---

## 7. THE CMS EDIT PATTERN — Dual-Mode Architecture

This is a **unique architectural pattern** in this codebase. ~20 components have both a VIEW mode and an EDIT mode, used by the Zopsmart CMS to let admins visually edit page sections.

### Architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    EditComponentFactory                       │
│  (Routes layout.id → correct Edit component)                 │
│                                                              │
│  case 2: → <EditImageWithText updateData={onSubmit} />       │
│  case 5: → <EditCards updateData={onSubmit} />               │
│  case 7: → <EditHighlights updateData={onSubmit} />          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    EditViewToggle                             │
│  (The MUX — switches between View and Edit mode)             │
│                                                              │
│  if (props.edit === true)                                    │
│    → Render EditComponent + pass updateData callback         │
│  else                                                        │
│    → Render ViewComponent (read-only)                        │
└──────────┬──────────────────────────┬───────────────────────┘
           │                          │
     edit=false                  edit=true
           │                          │
           ▼                          ▼
┌──────────────────┐    ┌──────────────────────────────┐
│  ViewComponent   │    │  EditContainer (Class Comp)   │
│  (e.g. CardsUI)  │    │                               │
│                  │    │  constructor: state = {data}   │
│  - Stateless     │    │  componentDidMount: updateData │
│  - Props only    │    │  componentDidUpdate: updateData│
│  - Read-only     │    │  componentWillUnmount: update  │
│  - No callbacks  │    │                               │
│                  │    │  Handlers: saveTitle,          │
│                  │    │    saveDescription,            │
│                  │    │    imageUpload, addNew,        │
│                  │    │    removeBlock                 │
│                  │    │                               │
│                  │    │  render: React.cloneElement(   │
│                  │    │    child, {data, handlers...}) │
│                  │    └───────────────┬───────────────┘
│                  │                    │
│                  │                    ▼
│                  │    ┌──────────────────────────────┐
│                  │    │  EditUI Component             │
│                  │    │                               │
│                  │    │  Uses contentEditable spans   │
│                  │    │  for inline text editing      │
│                  │    │                               │
│                  │    │  Calls handler functions      │
│                  │    │  injected from Container      │
│                  │    └──────────────────────────────┘
└──────────────────┘
```

### Data Flow for an Edit (e.g., user edits a Card title):

```
Step 1: User types in contentEditable span
          ↓
Step 2: EditText.onBlur fires
          ↓ save(newText, cardId)
Step 3: EditCardsUI.saveNewState(newText, 'title', cardId)
          ↓ (handler injected by Container via cloneElement)
Step 4: EditCardsContainer.saveNewState():
          - temp = JSON.parse(JSON.stringify(state.data))  // DEEP CLONE
          - temp.data.map(item => item.id === cardId ? {...item, title: newText} : item)
          - setState({data: temp})
          ↓
Step 5: componentDidUpdate() detects state.data changed
          ↓ this.props.updateData(this.state.data)
Step 6: EditViewToggle receives updated data
          ↓ this.props.updateData(data)
Step 7: EditComponentFactory.onSubmit(data, index)
          ↓
Step 8: Parent persists data to backend API
```

### The EditText Primitive (contentEditable):

```jsx
// How inline text editing works
<span
  contentEditable={true}
  ref={this.ref}
  onInput={() => {
    const text = this.ref.current.innerText.trim()
    if (this.props.onChange) this.props.onChange(text)
  }}
  onBlur={() => {
    this.props.save(this.ref.current.innerText, this.props.id)
  }}
  onKeyDown={(e) => {
    // Block Ctrl+B/I/U (no formatting)
    // Block Enter in singleLine mode
    // Strip HTML from paste
  }}
  dangerouslySetInnerHTML={{ __html: this.props.value }}
/>
```

### View vs Edit Comparison:

| Aspect | View Mode | Edit Mode |
|--------|-----------|-----------|
| Component | `{Name}UI.js` | `Edit{Name}Container.js` + `Edit{Name}UI.js` |
| Component Type | Functional | Class (PureComponent) |
| State | None | Full state in Container |
| Data Source | props.data (read-only) | props.data → state.data (mutable) |
| Text Rendering | `dangerouslySetInnerHTML` | `contentEditable` spans |
| Callbacks | None | Handlers via `React.cloneElement()` |
| Immutability | N/A | `JSON.parse(JSON.stringify())` before setState |

**Interview talking point:** "The CMS edit system uses a Container-UI-Toggle pattern. EditViewToggle acts as a mux — one prop (`edit`) switches the entire component tree between read-only display and admin editing. The Container uses class component lifecycle hooks to keep parent data in sync on mount, update, and unmount."

---

## 8. HOOKS & UTILITIES

### useWindowSize Hook
```
Location: src/components/WindowSizeHook/index.js

State: { width: window.innerWidth, height: window.innerHeight }
Effect: window.addEventListener('resize', debounced(handleResize, 300ms))
Cleanup: removeEventListener on unmount
Returns: { width, height }

Usage:
  const { width } = useWindowSize()
  if (width <= 768) renderMobileLayout()
```

### useAuth Hook
```
Location: src/components/LoginPackage/index.js

Params: tokenStorage, baseUrl, loginEndPoint, refreshEndPoint,
        providers[{authClientID, onSuccess, onError, name}],
        roles, theme, tenantID, storage, createOrgCallback

Returns: { login, logout, refreshApi, guestLogin }

Flow:
  login() → Render Google sign-in button → OAuth flow
  loginApi() → POST to loginEndPoint → store tokens in localStorage
  refreshApi() → POST to refreshEndPoint → update tokens
  Auto-refresh: setInterval(refreshApi, 270000)  // every 4.5 minutes

  SSR-safe: if (typeof window === 'undefined') return {}
```

### ErrorBoundary
```
Location: src/utils/ErrorBoundary.js

Type: Class component (required for componentDidCatch)
State: { hasError: false }

Flow:
  Child throws → getDerivedStateFromError() → { hasError: true }
  componentDidCatch(error, errorInfo) → can log to service
  Render: hasError ? null : this.props.children

Note: Currently renders null on error (no fallback UI)
```

### Utility Functions:
```
debounce.js    → debounce(fn, delay) — delays execution until pause
guid.js        → guid() — generates unique identifier for array items
phoneUtil.js   → addNewError/deleteError — phone validation helpers
Upload.js      → file upload utility functions
configs.js     → BREAKPOINTS = { SMALL_MOBILE:'380px', LARGE_MOBILE:'414px', TABLET:'834px' }
styleVariables.js → Static responsive padding/margin/maxWidth constants
```

---

## 9. BUILD PIPELINE

```
SOURCE                      BUILD                         OUTPUT
──────                      ─────                         ──────
src/index.js ─────→   Microbundle-CRL ─────→   dist/index.js        (CJS)
  ├─ components/       (Rollup under           dist/index.modern.js  (ESM)
  ├─ common/            the hood)              dist/index.css        (CSS)
  ├─ utils/                                    dist/index.js.map     (sourcemap)
  └─ css/                                      dist/index.modern.js.map

Build command: microbundle-crl --no-compress --format modern,cjs

What Microbundle does:
  1. Resolves all imports starting from src/index.js
  2. Transpiles JSX → React.createElement via Babel
  3. Bundles all JS into single files (CJS + ESM)
  4. Extracts all CSS into dist/index.css
  5. Generates source maps
  6. Marks peer dependencies (react) as external

What gets published (npm):
  Only dist/ folder (via "files": ["dist"] in package.json)

Consumer's bundler then:
  1. Reads "module": "dist/index.modern.js" → imports ESM build
  2. Tree-shakes unused exports (ESM only)
  3. Resolves CSS from dist/index.css
```

---

## 10. COMPLETE FLOW DIAGRAM — Request to Render

Here's the ENTIRE journey from "user clicks a button in the consumer app" to "pixel changes on screen":

```
CONSUMER APP (e.g., Zopping Admin Panel)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  import { Button } from '@zopsmart/zs-components'
    │  import '@zopsmart/zs-components/dist/index.css'
    │  import '@zopsmart/zs-components/src/css/zopping.css'
    │
    │  <html class="zopping">  ← Theme set here
    │
    │  const [loading, setLoading] = useState(false)
    │
    │  <Button
    │    text="Save Product"
    │    variant="filled"
    │    color="primary"
    │    size="m"
    │    disabled={loading}
    │    onClick={() => saveProduct()}
    │  />
    │
    ▼
LIBRARY: src/index.js
━━━━━━━━━━━━━━━━━━━━━
    │
    │  export { Button } from './components/Button'
    │
    ▼
COMPONENT: src/components/Button/index.jsx
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  Destructure: text="Save Product", onClick={fn}
    │  Forward rest: variant="filled", color="primary", size="m"
    │
    │  return (
    │    <Wrapper {...rest} onClick={onClick} disabled={disabled}>
    │      {text}
    │    </Wrapper>
    │  )
    │
    ▼
STYLED-COMPONENT: src/components/Button/style.jsx
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    │
    │  Wrapper receives props: variant="filled", color="primary", size="m"
    │
    │  Resolves:
    │    background-color: var(--color-primary)        ← from getButtonBackgroundColor("filled","primary")
    │    color: var(--color-white)                     ← from getButtonFontColor("filled","primary")
    │    font-size: var(--size-font-m)                 ← from SIZE_OPTIONS["m"]
    │    padding: var(--size-spacing-s) var(--size-spacing-l)
    │    cursor: pointer                               ← disabled=false
    │
    ▼
CSS CASCADE
━━━━━━━━━━━
    │
    │  tokens.css:   --color-primary: var(--color-purple-500) → #414eff
    │  zopping.css:  html.zopping { --color-primary: #bf1139 }  ← OVERRIDES!
    │
    │  Resolved: --color-primary = #bf1139
    │
    ▼
BROWSER RENDER
━━━━━━━━━━━━━━
    │
    │  <button style="
    │    background-color: #bf1139;   ← Zopping red
    │    color: #ffffff;
    │    font-size: 0.875rem;
    │    padding: 0.75rem 1.25rem;
    │    cursor: pointer;
    │  ">
    │    Save Product
    │  </button>
    │
    ▼
USER INTERACTION
━━━━━━━━━━━━━━━━
    │
    │  User clicks button
    │    ↓
    │  DOM onClick fires
    │    ↓
    │  Button's <Wrapper onClick={onClick}> catches it
    │    ↓
    │  Calls parent's onClick={() => saveProduct()}
    │    ↓
    │  Parent executes saveProduct(), maybe sets loading=true
    │    ↓
    │  Parent re-renders: <Button disabled={true} ...>
    │    ↓
    │  Button re-renders: cursor=not-allowed, opacity=0.5
    │
    ▼
DONE. Full cycle: ~16ms (one React render frame)
```

---

## 11. RESPONSIVE BREAKPOINT SYSTEM

Two systems work in parallel:

### JS-Side Breakpoints (src/utils/configs.js)
```js
BREAKPOINTS = {
  SMALL_MOBILE: '380px',
  LARGE_MOBILE: '414px',
  TABLET: '834px'
}

// Used in: Table (desktop vs card layout), styled-components media queries
// Example: @media (max-width: ${BREAKPOINTS.LARGE_MOBILE}) { ... }
```

### CSS-Side Breakpoints (lib/Media.js via styled-breakpoints)
```js
breakpoints = {
  mobile: '1px',     mobileM: '400px',   mobileL: '600px',
  tablet: '768px',   desktop: '1024px',   desktopL: '1280px',
  hd: '1366px',      uhd: '1440px',      UhdL: '1600px'
}

// Provides SCREEN_SIZE.From.Desktop, SCREEN_SIZE.Below.Tablet, etc.
// Used in: Grid components, complex responsive layouts
```

---

## 12. KEY INTERVIEW TALKING POINTS SUMMARY

### On Architecture:
> "It's a headless-ish component library — we own the UI rendering but the consumer owns all state. Zero global state, no Context, no Redux. Pure props-down, callbacks-up. This makes it framework-agnostic at the state layer."

### On Theming:
> "Three-tier CSS variable system: base tokens → brand theme overrides → component extensions. Theme switching is pure CSS class swap — zero JS re-renders. We support 8 brands including a full dark mode that inverts the entire color palette."

### On the CMS Pattern:
> "We have a dual-mode rendering system for CMS pages. EditViewToggle is a mux that routes between read-only ViewComponents and editable EditContainers. The edit system uses contentEditable spans for inline WYSIWYG editing, class component lifecycle hooks for parent sync, and deep cloning for immutable state updates."

### On Build:
> "Microbundle outputs CJS + ESM. The ESM build enables tree-shaking. All CSS is extracted to a single file. Published to private Google Artifact Registry via GitHub Actions on release."

### On Trade-offs:
> "The 3MB bundle could be split. moment.js is heavy — I'd migrate to dayjs. TypeScript would add type safety for consumers. Class components in the edit system could migrate to hooks. Test coverage is low — we rely on Storybook + Chromatic for visual regression."
