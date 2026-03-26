# @zopsmart/zs-components — Complete Connection Guide

## 1. What Is It?

`@zopsmart/zs-components` (v2.3.9) is Zopping's **shared component library** — a private npm package with **120+ React components** built with `styled-components`. It provides the building blocks for the **Website Builder** (the drag-and-drop CMS that lets store owners customize their storefront).

There's also a legacy sibling: `zopsmart-ui` (v0.1.0) — an older Git-based dependency with 47 components, mostly superseded by zs-components.

**Key Dependencies of the library:**
- React ^17.0.2 (peer dependency)
- styled-components ^5.3.9
- @tanstack/react-table ^8.11.3
- recharts ^2.6.2
- @react-google-maps/api 1.8.2
- react-dates 21.8.0
- quill ^1.3.7

---

## 2. How It's Connected — The 3-Step Bootstrap

The connection happens in `src/containers/App/css.js`:

```javascript
import "@zopsmart/zs-components/dist/index.css"; // Step 1: Library CSS + design tokens
import "./tokens.css";                            // Step 2: App-level token overrides
```

Then `app.css` adds app-specific variables on `:root`. This creates a **3-layer CSS cascade**:

```
Layer 1: @zopsmart/zs-components/dist/index.css
         └── Base design tokens on `html` selector
         └── Component-level styles (styled-components output)
         └── Example: --size-spacing-m, --size-font-m, --color-*

Layer 2: src/containers/App/tokens.css
         └── Overrides on `html` selector
         └── Example: --color-primary: #4ab819, --color-error: #fa1f4b

Layer 3: src/containers/App/app.css
         └── App-specific variables on :root
         └── Example: --primary-text-color, --border-color, --bg-color
```

**No `<ThemeProvider>` needed** — the entire design system flows through **CSS custom properties** (CSS variables). Components from zs-components automatically pick up overridden tokens.

---

## 3. The Two Worlds — Edit vs Display Components

The library exports components in **pairs** — one for the admin editing the layout, one for the customer viewing it:

| Edit Component (Admin) | Display Component (Storefront) | Purpose |
|------------------------|-------------------------------|---------|
| `EditFAQ` | (renders internally) | Accordion/FAQ sections |
| `EditCatalog` | `Catalog` | Product catalog grid |
| `EditFeatureGridSection` | `FeatureGridSection` | Feature comparison grids |
| `EditHighlightsSection` | `HighlightsSection` | Highlight cards |
| `EditBenefitsSection` | `Benefits` | Benefits listing |
| `EditTestimonialComponent` | `NonEditTestimonialComponent` | Customer testimonials |
| `EditCarouseComponent` | `NonEditCarouselComponent` | Image carousels |
| `EditImageSlider` | `ImageSlider` | Image slideshows |
| `EditImageWithTextWithButton` | (renders internally) | Image + text + CTA |
| `EditPricingWidget` | `PricingWidget` | Pricing tables |
| `EditRichTextFunction` | `RichText` | Rich text blocks |
| `EditImage` | `ImageBanner` / `ImageBannerWithButton` | Banner images |
| `EditLayout` / `EditIndexImageWithText` | (renders internally) | Content layouts |

---

## 4. All 120+ Components by Category

### Layout & Container Components (14)
Accordion, Banner, BannerWithButton, BannerWithMultipleButton, BGImageWithContent, BreadCrumbs, Card, CardSlider, Cards, ContentLayout, Drawer, FooterContainer, Header, MasonryLayout

### Form & Input Components (16)
Form, FormControl, GenericFormLayout, HMCustomForm, InputFields, SelectInputField, TextArea, Phone, DynamicInput, FileUpload, MonthPicker, Calendar, Checkbox, Radio, Switch

### Chart & Data Visualization (8)
AreaChart, BarGraph, LineGraph, DoughnutChart, PieChart, ProgressChart, CircularProgressBar, ProgressBar

### Product & Commerce (8)
Product, ProductCard, ProductCollection, CartItem, QuantityController, PriceFilter, PricingCard, PricingWidget

### Image & Media (15)
ImageBanner, ImageBannerWithButton, ImageBannerWithMultipleButton, ImageCollection, ImageGrid, ImageSlider, ImageSliderZopping, ImageWithTextUI, ImageWithTextWithButton, Imageslideshow, SlideShow, VideoCarousel, EditImage, EditImageCollection, EditImageSlider

### UI Atoms (20)
Avatar, AvatarGroup, Button, Chip, Divider, Modal, ModalMolecule, Notification, Popover, Rating, Skeleton, Stepper, Sticky, Tabs, Timeline, TimelineItem, Tooltip, TypographyText, DropdownMenu, CarouselIndicator

### Editable/Admin Sections (13)
EditBenefitsSection, EditCarouseComponent, EditCatalog, EditFAQ, EditFeatureGridSection, EditHighlightsSection, EditImageWithTextWithButton, EditIndexImageWithText, EditLayout, EditPricingWidget, EditRichTextFunction, EditTestimonialAdmin, EditTestimonialComponent

### View/Non-Editable Components (3)
NonEditCarouselComponent, NonEditTestimonialAdmin, NonEditTestimonialComponent

### Data & Table Components (5)
Table, TableCell, TableDataWrapper, TableRow, EnhanceTable, PaginationTestPaper

### Content Sections (8)
Benefits, Catalog, FeatureGridSection, FeatureNavigator, HighlightsSection, QuestionAnswerList, RichText, UpdateCards, UpdateImageGallery

### Maps & Location (2)
GoogleMap, MapContactForm

### Authentication (1)
GoogleLogin

### Custom Hooks (2)
- `useAuth` — Authentication hook
- `useWindowSize` — Window resize tracking hook

---

## 5. The Architecture Pattern — How Layouts Work

There are **two systems** (old and new):

### Old System: The `fields()` Function Pattern

Every layout exports a **factory function** that returns `{ fields: (props) => JSX }`:

```javascript
// src/pages/settings/Themes/Layouts/Accordion.jsx
import { EditFAQ } from "@zopsmart/zs-components";
import { Input, Radio, Upload } from "../../../../components/Form";

const EditFaqLayout = (props) => {
  const { updateData, getState, updateState, registerValidation, edit, layoutBackground } = props;

  const style = {
    bgColor: layoutBackground?.backgroundColor || "",
    bgImage: layoutBackground?.backgroundImage || "",
  };

  return (
    <Fragment>
      {edit && (
        <div className="form-sections rich-text-layout">
          <Radio
            options={[
              { text: "BACKGROUND IMAGE", value: "IMAGE" },
              { text: "BACKGROUND COLOR", value: "COLOR" },
            ]}
            value={getState(["layoutBackground", "backgroundType"]) || ""}
            onChange={(value) => updateState(["layoutBackground", "backgroundType"], value)}
          />
          {/* Color picker or Image upload based on selection */}
        </div>
      )}
      <EditFAQ
        data={props?.data}
        edit={edit}
        updateData={(val) => updateData(val)}
        activeStyle={1}
        style={style}
        defaultStyle="#7e7efc"
      />
      <LayoutVisibility {...props} />
      <ContentAlignment {...props} />
    </Fragment>
  );
};

// The factory pattern - returns { fields: fn }
const EditLayoutWrp = () => {
  return {
    fields: (props) => <EditFaqLayout {...props} />,
  };
};
export default EditLayoutWrp;
```

The caller (`layout.js`) invokes `this.fields(...)` passing standardized props:

```javascript
this.fields({
  getState: this.getState.bind(this),
  updateState: this.updateState.bind(this),
  registerValidation: this.registerValidation.bind(this),
  edit: isEdit,
  layoutName: this.props.LayoutComponentData.name,
  data: this.state.values,
  updateData: this.handleUpdateData,
  uploadContent: this.uploadContent,
});
```

### New System: WebsiteBuilder with Direct Components

The newer WebsiteBuilder uses direct React components with `useLayoutUpdater` hook:

```javascript
// src/pages/settings/Themes/WebsiteBuilder/LayoutComponents/Images/ImageCardLayout.js
const ImageCardLayout = ({ layoutName, layoutDetails, layoutId, setLayoutData, iframeRef }) => {
  const { updateLayout } = useLayoutUpdater({ layoutId, layoutName, setLayoutData, iframeRef });

  // updateLayout({ data: updatedData }) → updates state + sends to iframe preview (300ms debounce)
};
```

---

## 6. Complete Data Flow — From Layout Selection to Live Website

```
┌─────────────────────────────────────────────────────────────┐
│  1. LAYOUT REGISTRY                                          │
│  WebsiteBuilder/Utility/constants.js                         │
│  AVAILABLE_LAYOUT = { Images: [...], Banners: [...], ... }   │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  2. USER SELECTS LAYOUT                                      │
│  AddNewLayout/index.js → AddNewLayoutPopup                   │
│  User picks "Accordion" from categorized grid                │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  3. TEMPLATE INITIALIZED                                     │
│  PageLayoutLeftMenu.handleAddNewSection("Accordion")         │
│  → Finds template from preFilledLayoutData.js                │
│  → Creates { name: "Accordion", data: {...}, id: 999 }      │
│  → Appends to layoutData state                               │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  4. LAYOUT EDITOR RENDERS                                    │
│  renderLayoutComponent("Accordion")                          │
│  → switch/case maps name to <AccordionLayout />              │
│  → Inside: imports EditFAQ from @zopsmart/zs-components      │
│  → Renders admin form UI + zs-component with edit=true       │
│                                                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │  <EditFAQ                                         │       │
│  │    data={props.data}         // FAQ items array   │       │
│  │    edit={true}               // Enable editing    │       │
│  │    updateData={callback}     // Save changes      │       │
│  │    activeStyle={1}           // Visual variant    │       │
│  │    style={{ bgColor, bgImage }}  // Background   │       │
│  │  />                                               │       │
│  └──────────────────────────────────────────────────┘       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  5. LIVE PREVIEW (iframe)                                    │
│  updateLayout() → useLayoutUpdater hook                      │
│  → Debounced postMessage to iframe (300ms)                   │
│  → Iframe re-renders with updated data                       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  6. SAVE TO DRAFT                                            │
│  PUT /website-service/draft/{page}                           │
│  → Payload: { layouts: [...all layout data...] }             │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  7. PUBLISH TO LIVE                                          │
│  PUT /website-service/layout/{page}                          │
│  → Storefront now shows updated layouts                      │
│  → Display components (non-edit) render for customers        │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. The Wrapper Layer — src/components/Layout/

The admin app wraps zs-components in thin adapter components:

```
src/components/Layout/
├── AccordionComponent/    → wraps EditFAQ
├── BannerPagenics/        → wraps ImageBanner, ImageBannerWithButton, ImageBannerWithMultipleButton
├── Benefits/              → wraps Benefits
├── Cards/                 → wraps UpdateCards
├── CarouselPagenics/      → wraps NonEditCarouselComponent
├── Catalog/               → wraps Catalog
├── Content/               → wraps EditLayout, EditIndexImageWithText
├── FeatureGrid/           → wraps FeatureGridSection
├── FeatureNavigator/      → wraps FeatureNavigator
├── GenericForm/           → wraps GenericFormLayout
├── Highlights/            → wraps HighlightsSection
├── ImageCarousel/         → wraps ImageSlider
├── ImageGrid/             → wraps ImageGrid
├── ImageSlideshowPagenics → wraps Imageslideshow
├── Maps/                  → wraps GoogleMap, MapContactForm
├── PricingWidget/         → wraps PricingWidget
├── RichTextPagenics/      → wraps RichText
└── TestimonialPagenics/   → wraps NonEditTestimonialComponent
```

**What wrappers add:**
- Preview images for layout thumbnails (when `props.image === true`)
- Style mapping (converting `layoutBackground` → `{ bgColor, bgImage }`)
- Variant selection via `activeStyle` prop
- `edit={false}` for display-only rendering

**Example wrapper:**
```javascript
// src/components/Layout/AccordionComponent/index.js
import { EditFAQ } from "@zopsmart/zs-components";

export default function AccordionComponent(props) {
  const { data, name } = props;

  // If thumbnail mode, show static preview image
  if (props.image) {
    return <img src={AccordianImg} alt="" style={{ maxWidth: "100%", maxHeight: "100%" }} />;
  }

  // Otherwise render the actual zs-component
  const style = {
    bgColor: data.layoutBackground?.backgroundColor || "",
    bgImage: data.layoutBackground?.backgroundImage || "",
  };
  return (
    <EditFAQ {...props} style={style} edit={false} activeStyle={activeStyleNumber} defaultStyle="#7e7efc" />
  );
}
```

---

## 8. Standard Props Contract — What Every zs-component Receives

```javascript
{
  // Data
  data: Object,              // The layout's content data (FAQ items, images, text, etc.)

  // Mode
  edit: Boolean,             // true = show editor UI, false = display-only preview

  // Style
  style: {
    bgColor: String,         // Background color hex
    bgImage: String,         // Background image URL
  },
  activeStyle: String|Number, // Visual variant (1, 2, "Normal text", "With subtitle")
  defaultStyle: String,       // Default accent color

  // State management (old system)
  getState: Function,         // getState(["path", "to", "value"]) → reads nested state
  updateState: Function,      // updateState(["path", "to", "value"], newValue)
  registerValidation: Function, // registerValidation(["path"], errorObject)

  // Callbacks
  updateData: Function,       // Called when component internally updates its data
  uploadContent: Function,    // File upload handler

  // Meta
  layoutName: String,         // e.g., "Accordion", "BannerPagenics"
  page: String,               // e.g., "home", "about"
}
```

---

## 9. Token Customization — CSS Variables

### Tokens from zs-components (Layer 1)
The library's `dist/index.css` declares base tokens on `html`:
- Sizing: `--size-spacing-xxs` through `--size-spacing-xxxl`
- Fonts: `--size-font-xs` through `--size-font-xxxl`
- Headings: `--size-heading-xs` through `--size-heading-xxxl`
- Font weights: `--size-font-weight-xxs` through `--size-font-weight-bold`
- Borders: `--size-border-width-s`, `--size-border-radius-m`
- Colors: `--color-primary`, `--color-secondary`, `--color-success`, `--color-error`, `--color-warning`, `--color-info`
- Component-specific: Slider arrows, product card dimensions, slideshow defaults, data card colors

### App-level Overrides (Layer 2) — tokens.css
```css
html {
  --color-primary: #4ab819;
  --color-skeleton: #e8e8e8;
  --bg-gradient: linear-gradient(to right, #f0f0f0 8%, #e0e0e0 18%, #f0f0f0 33%);
  --content-description-font-size: 1rem;
  --content-title-font-size: 1.5rem;
  --size-l: 1.5rem;
  --size-m: 2.5rem;
  --size-spacing-s: 0.25rem;
  --size-font-m: 0.875rem;
  --color-black: #000000;
  --color-white: #ffffff;
  --size-border-width-s: 0.063rem;
  --color-border-input: #00000040;
  --color-gray-300: #00000040;
  --size-spacing-xs: 0.5rem;
  --color-error: #fa1f4b;
}
```

### App-specific Variables (Layer 3) — app.css
```css
:root {
  --border-color: #dadee0;
  --primary-text-color: #2b3238;
  --secondary-text-color: #80959d;
  --primary-color: #4ab819;
  --secondary-action-color: #f3f5f6;
  --separator-color: #eaefef;
  --bg-color: #fbfcfc;
  --disabled-color: #f4f7f9;
  --black-transparent: #00000099;
}
```

---

## 10. How to Use zs-components — Quick Start

### Step 1: Import CSS (once, at app level)
```javascript
import "@zopsmart/zs-components/dist/index.css";
```

### Step 2: Override tokens if needed (once, at app level)
```css
html {
  --color-primary: #your-brand-color;
}
```

### Step 3: Import and use components
```javascript
// Display component (for storefront/preview)
import { Catalog, Benefits, ImageSlider } from "@zopsmart/zs-components";

<Catalog data={catalogData} />
<Benefits data={benefitsData} />
<ImageSlider data={sliderData} />

// Edit component (for admin panel)
import { EditFAQ, EditCatalog } from "@zopsmart/zs-components";

<EditFAQ data={faqData} edit={true} updateData={handleUpdate} activeStyle={1} />
<EditCatalog data={catalogData} edit={true} updateData={handleUpdate} />

// UI atoms
import { Button, Avatar, Modal, Tabs, Rating } from "@zopsmart/zs-components";

// Charts
import { BarGraph, PieChart, LineGraph } from "@zopsmart/zs-components";

// Tables
import { Table, EnhanceTable } from "@zopsmart/zs-components";

// Maps
import { GoogleMap, MapContactForm } from "@zopsmart/zs-components";

// Hooks
import { useAuth, useWindowSize } from "@zopsmart/zs-components";
```

---

## 11. zopsmart-ui (Legacy Package) — For Reference

`zopsmart-ui` (v0.1.0) is the older Git-based library. Key differences:

| Aspect | zopsmart-ui | @zopsmart/zs-components |
|--------|-------------|------------------------|
| Version | 0.1.0 | 2.3.9 |
| Source | Git SSH dependency | npm registry |
| React | ^16.3.2 | ^17.0.2 |
| Styling | Plain CSS | styled-components |
| Build | Gulp + Babel | microbundle |
| Components | 47 | 120+ |
| Status | Legacy, mostly superseded | Actively maintained |

**zopsmart-ui exports** (47 components): Banner, BannerWithText, BannerCarousel, Image, ImageWithText, ImageCard, ImageCardCarousel, ContentCard, Form, VideoCard, FooterSimple, FooterCentre, TabsCard, Button, Arrow, Logo, Menu, MenuBar, NavigationBlock, etc.

**Usage in project** — only 2-3 files still import from zopsmart-ui:
- `src/pages/settings/Themes/Layouts/BannerWithText.jsx`
- `src/pages/settings/Themes/Layouts/BannerWithMultipleButtons.jsx`

---

## 12. Key File Paths

| File | Purpose |
|------|---------|
| `src/containers/App/css.js` | Bootstrap — imports zs-components CSS + token overrides |
| `src/containers/App/tokens.css` | App-level CSS variable overrides |
| `src/containers/App/app.css` | Global app styles + utility classes |
| `src/components/Layout/` | 18 wrapper components around zs-components |
| `src/pages/settings/Themes/Layouts/` | Layout editor components (old system, `fields()` pattern) |
| `src/pages/settings/Themes/WebsiteBuilder/` | New website builder system |
| `src/pages/settings/Themes/WebsiteBuilder/Utility/constants.js` | Layout registry (AVAILABLE_LAYOUT) |
| `src/pages/settings/Themes/WebsiteBuilder/Utility/preFilledLayoutData.js` | Default template data for each layout |
| `src/pages/settings/Themes/WebsiteBuilder/LayoutComponents/index.js` | renderLayoutComponent() — maps names to components |
| `src/pages/settings/Themes/WebsiteBuilder/AddNewLayout/index.js` | Layout selection popup UI |
| `src/pages/settings/Themes/WebsiteBuilder/Sidebar/PageLayoutLeftMenu.js` | Layout list, add/edit/delete orchestration |
| `src/pages/settings/Themes/layout.js` | Old system — invokes `fields()` with props |
| `node_modules/@zopsmart/zs-components/dist/index.css` | Library CSS + design tokens |
| `node_modules/@zopsmart/zs-components/dist/index.js` | Library CJS entry |
| `node_modules/@zopsmart/zs-components/dist/index.modern.js` | Library ESM entry |

---

## 13. Interview Talking Points

1. **"We have a shared component library (zs-components) with 120+ components that serves both the admin panel and the customer-facing storefront. Components export Edit and Display variants — the admin sees editable forms, customers see the rendered output."**

2. **"The design system is CSS-variable-based, not JS-based (no ThemeProvider). This means any app can override tokens by just setting CSS custom properties on the html element — zero JS configuration needed."**

3. **"The Website Builder follows a registry pattern — layouts are registered in a constants file, each with a factory function that returns `{ fields: (props) => JSX }`. This decouples layout selection from layout rendering."**

4. **"Live preview works via iframe postMessage — every edit triggers a debounced message (300ms) to the preview iframe, giving instant visual feedback without a full page reload."**

5. **"The save flow is two-stage: Save to Draft (PUT /website-service/draft) for preview, then Publish (PUT /website-service/layout) to go live. This gives store owners a safe editing workflow."**

6. **"We migrated from zopsmart-ui (plain CSS, Gulp, 47 components) to @zopsmart/zs-components (styled-components, microbundle, 120+ components) — the old package is still referenced in 2-3 legacy layout files."**

---

*Generated for interview preparation reference.*
