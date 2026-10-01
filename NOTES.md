# Developer Notes — Mistvale Tea Co. Rescue

## 1. What I Changed

### Bugs & Business Logic
- **R1 (Price Calculation)**: Replaced DOM text scraping (`parseInt(document.getElementById('price-' + id).innerText)`) with direct lookups into the authoritative `PRODUCTS` array via product ID. This prevented fatal errors when formatting prices with commas (e.g. `₹1,299`).
- **R2 (Inventory & Order Caps)**: Enforced strict purchase bounds (`Math.min(5, product.stock)`). Prevented out-of-stock items (`stock: 0`, like Hibiscus Rose) from being added. Clamped manual input in the cart drawer between 1 and available limit. Fixed the string concatenation bug where adding quantity resulted in `"2" + 1 = "21"`.
- **R3 (Coupon WELCOME10)**: Made the code case-insensitive and idempotent. Enforced the ₹399 subtotal minimum, excluded the `"Gifts"` category (product 108) from discount calculations, and applied the ₹150 discount cap. Fixed the bug where clicking "APPLY" repeatedly compounded the discount infinitely. Added reactive re-evaluation so discount updates if cart quantities drop below threshold.
- **R4 (Shipping Calculation)**: Normalized shipping to ₹49, with free shipping automatically granted when net amount (subtotal minus discount) reaches ₹499. Fixed the free-shipping progress bar calculation so it clamps to 100% and never displays negative deficits (e.g., "Add ₹-50 more").
- **R5 (Currency Formatting)**: Created a centralized `formatINR()` helper function formatting all currency values with `₹` and Indian digit grouping (`toLocaleString('en-IN')`). Ensured only the final grand total is rounded.
- **R6 (Sold-Out Products)**: Disabled cart addition buttons for out-of-stock items and forced all sold-out products to the bottom of the catalog across all sort modes.
- **R7 (Unified Search, Filter & Sort)**: Fixed the critical `indexOf` inversion bug where `ids.indexOf(p.id)` evaluated `0` as falsy and `-1` as truthy. Synchronized search, category filtering, and sorting into a unified state pipeline. Added input debouncing and an active request token to prevent out-of-order asynchronous Promise resolutions from `API.search()`.
- **R8 (Pincode Checker)**: Attached a `.catch()` rejection handler to `API.checkPincode()`. Rejections with `INVALID_PINCODE` now render clear feedback rather than leaving the UI permanently frozen on "Checking...".
- **Cart State Initialization**: Fixed the first-visit null crash by initializing `cart` with fallback `JSON.parse(localStorage.getItem('mv_cart')) || []`. Fixed `cart.splice(i)` to `cart.splice(i, 1)` to prevent deleting all subsequent items.
- **Checkout Contract**: Wired the "Proceed to Checkout" button to populate hidden inputs `items` (JSON string) and `coupon` (string code) inside `<form id="checkout-form">` and trigger native POST submission.

### Design & Branding
- Purged all non-brand colors (hot pink, purple, neon lime, raw yellow). Implemented CSS custom properties for `--tea-green` (#1f3d2b), `--leaf` (#4f7942), `--cream` (#f6f1e7), `--parchment` (#ebe2cf), `--saffron` (#d9962b), `--ink` (#1b1b1b), and `--error` (#b3261e).
- Removed Comic Sans, Pacifico, and Oswald. Standardized on a cohesive two-family typography pairing: **Fraunces** for headings and **DM Sans** for body text.
- Reconstructed the DOM into the **exact 10 required sections** in specified sequence.
- Replaced the intrusive 2-second automatic newsletter popup with a calm, inline newsletter card.
- Replaced wild animations (`marquee`, `blink`, `bounceIn`) with subtle 150–250ms transitions and added `@media (prefers-reduced-motion: reduce)`.

### Technical & UX
- Added `<meta name="viewport" content="width=device-width, initial-scale=1.0">` to support responsive mobile viewports down to 360px.
- Replaced non-semantic `<div>` buttons with accessible `<button>` tags with visible saffron focus rings.
- Implemented accessible modals for the Cart Drawer and Quick View with background overlay dismissal and keyboard `Escape` closing.
- Added structured JSON-LD schemas for `OnlineStore`, `FAQPage`, and all 8 `Product` items without fabricating fake reviews.

---

## 2. What the AI Got Wrong (and How I Caught It)

1. **Attempting External Icon Dependencies**: Initial AI suggestions tried to import Font Awesome 6 CDN links or load external SVG bundles. I caught this during constraint review and replaced them with hand-crafted, lightweight inline SVGs with uniform stroke width.
2. **Coupon WELCOME10 Compounding & Gift Ineligibility**: When asked to implement the coupon logic, the AI simply wrote `discount = subtotal * 0.1` and added it to previous discount. It completely overlooked that category `"Gifts"` (Tea Lover's Sampler Box, ₹1899) must be excluded from the 10% discount, and failed to enforce the ₹150 cap. I wrote a dedicated unit test in `scratch/test_business_rules.js` to catch and correct the coupon calculation.
3. **Array Sorting Mutation & Boolean Return**: The AI initially used `shown.sort((a, b) => a.price > b.price)`, which returns a boolean. In modern V8 and SpiderMonkey engines, returning a boolean to `.sort()` yields inconsistent, non-standard ordering. I corrected it to `(a, b) => a.price - b.price` combined with a primary partition check for `(a.stock === 0) - (b.stock === 0)` to guarantee out-of-stock items stay pinned to the end.
4. **Search Promise Race Condition**: The AI wrote a simple `input.oninput` handler calling `API.search(query).then(...)`. Because `API.search` has variable artificial latency (`Math.max(120, 900 - q.length * 140)`), typing fast resulted in older queries finishing *after* newer queries, overwriting current results with stale data. I added an `activeSearchRequestId` counter and a 200ms debounce timer to discard stale promises.

---

## 3. Images

- **Tools Used**:
  - **Hero Banner**: Gemini Imagen 3 (`hero_banner_mistvale`).
  - **8 Product Images**: Gemini Imagen 3 (`p101` through `p108`).
  - **Logo**: Vector SVG handcrafted with brand palette tokens and Fraunces typography.
- **Post-Processing & Optimization**:
  - Cropped and rendered all product shots to a consistent 1:1 square ratio with cohesive dark green artisanal pouches and matching ceramic saucers.
  - Processed all images via high-quality bicubic resampling using .NET GDI+ compression.
  - **Final File Sizes**:
    - `hero-banner.png`: **191 KB** (Limit: < 250 KB)
    - `p101.png` to `p108.png`: **41 KB – 66 KB** (Limit: < 150 KB each)
    - Total image payload reduced by over 85% compared to the original uncompressed placeholders.

---

## 4. How I Tested It

1. **Browser Testing**: Verified layout, SVG rendering, and cart interactions in Google Chrome, Microsoft Edge, and Firefox.
2. **Responsive Viewports**: Tested at `360px` (iPhone SE/Galaxy), `390px` (iPhone 14/15), `768px` (iPad Portrait), `1024px` (iPad Landscape), and `1440px` (Desktop). Confirmed 2 columns on mobile and 4 columns on desktop without horizontal scrolling.
3. **Keyboard & Accessibility**:
   - Tabbed through all interactive buttons and inputs, verifying saffron focus rings.
   - Tested `Escape` key dismissal on Cart Drawer and Quick View modals.
   - Verified screen reader labels (`aria-label`, `aria-expanded`, `aria-live`).
4. **Cart & Edge Case Validation**:
   - Verified that empty localStorage does not throw a runtime exception.
   - Tested stock cap (cannot exceed 5 units or product stock).
   - Tested coupon thresholds: ₹349 subtotal rejects coupon; adding another item activates coupon; removing items dynamically recalculates discount.
   - Tested delivery checker with valid regional codes (`734001`, `110001`, `560001`), unserviceable codes (`999999`), and invalid inputs (`abc`, `12345`).

---

## 5. Questions for the Team

1. **Free Shipping Threshold**: The original starter code had ₹599 in JavaScript while the announcement banner stated ₹499. Based on standard Indian direct-to-consumer tea average order values and rule R4, I aligned both the announcement and JS logic to **₹499**. Please confirm if ₹499 or ₹599 is the preferred commercial threshold.
2. **Pincode Logistics Expansion**: The simulated `API.checkPincode` currently supports major metro prefixes (`73`, `70`, `71`, `11`, `40`, `56`, `60`, `50`). For unserviceable pincodes, should we capture customer emails for notify-when-serviceable leads?

---

## 6. Time Spent

- **Analysis & Planning**: 45 minutes
- **Image Generation, Framing & Compression**: 50 minutes
- **Design System, Responsive Layout & Semantic DOM**: 60 minutes
- **Cart, Business Rules R1–R8 & Checkout Contract**: 55 minutes
- **Testing, Verification & Documentation**: 30 minutes
- **Total Honest Time**: **4 hours**

---

## 7. Extra Features Added

1. **Wishlist Persistence**: Interactive heart button on product cards allowing shoppers to save favorites to `localStorage`.
2. **Quick View Modal**: Rich modal allowing customers to inspect tea descriptions, pricing, and stock status without leaving the catalog view.
3. **Toast Feedback**: Non-blocking toast notification when adding items to the cart with a one-click shortcut to view the cart drawer.
4. **Accessible FAQ Accordion**: Expandable accordions with keyboard support and animated chevron indicators.

---

## 8. With More Time I Would…

1. **URL State Synchronization**: Serialize active category filter, search query, and sort mode into URL search parameters (`?category=black&sort=low`) using `history.pushState()` for direct link sharing.
2. **Tea Brewing Temperature & Steeping Timer**: Add an interactive steeping calculator inside the product detail modal (e.g. 85°C for 3 minutes for Darjeeling first flush).
3. **Gift Message & Note**: Provide an optional gift card message input at checkout when the Tea Lover's Sampler Box is in the cart.
