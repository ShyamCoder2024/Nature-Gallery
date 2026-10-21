# 03 — Design System & Design Requirements (DRD)

The design goal: **quiet luxury**. Generous space, beautiful photography, refined type, and motion that feels physical. Guests should feel they are holding a premium printed menu that happens to be alive. Admins should feel calm and in control, like a well-made professional tool.

---

## 1. Design principles

| Principle | What it means in practice |
|-----------|---------------------------|
| **Food first** | Photos are the hero. Chrome (bars, borders, labels) is minimal and recedes. |
| **One thing at a time** | Each screen has one primary action. Secondary actions are quieter. |
| **Instant, then accurate** | Every tap responds immediately (optimistic UI); the network confirms in the background, and the UI is honest when it can't. |
| **Physical motion** | Things move like real objects: sheets slide from where they live, items fly to the cart, nothing teleports. Motion is short and never blocks. |
| **Calm states** | Loading uses skeletons that match the final layout; errors are friendly and tell you what to do; empty states invite action. |
| **Brand, not template** | Each business's colours, logo, and fonts make the guest app feel like *their* app, within guardrails that protect quality and accessibility. |
| **Thumb-reachable** | On phones, primary actions sit in the bottom third of the screen. |
| **Accessible by default** | Contrast, touch target size, screen reader labels, reduced motion and large text are built in, not added later. |

---

## 2. Design tokens

Tokens are the single source of every visual value. They live in `packages/design-tokens` (doc 05 §4) and compile to CSS custom properties. Tenant themes override only the **semantic** layer, never the primitive scales.

### 2.1 Token layers

1. **Primitive** — raw values: `gray-50 … gray-950`, `space-1 … space-16`, `radius-sm …`.
2. **Semantic** — meaning: `color-bg`, `color-surface`, `color-text`, `color-text-muted`, `color-accent`, `color-on-accent`, `color-border`, `color-success`, `color-warning`, `color-danger`, `color-veg`, `color-nonveg`.
3. **Component** — specific: `button-primary-bg`, `card-radius`, `sheet-shadow`.

### 2.2 Colour

**Neutral scale (warm gray)** — used by all themes; warm grays flatter food photos more than blue-grays.

| Token | Hex |
|-------|-----|
| gray-0 | `#FFFFFF` |
| gray-25 | `#FCFBF9` |
| gray-50 | `#F7F5F2` |
| gray-100 | `#EFECE7` |
| gray-200 | `#E2DDD5` |
| gray-300 | `#CFC8BD` |
| gray-400 | `#A9A196` |
| gray-500 | `#857D72` |
| gray-600 | `#655E55` |
| gray-700 | `#4A443D` |
| gray-800 | `#302C27` |
| gray-900 | `#1C1A17` |
| gray-950 | `#0F0E0C` |

**Semantic colours (Ivory theme, default)**

| Token | Value | Use |
|-------|-------|-----|
| color-bg | gray-25 | Page background |
| color-surface | gray-0 | Cards, sheets |
| color-surface-sunken | gray-50 | Inputs, grouped sections |
| color-text | gray-900 | Primary text |
| color-text-muted | gray-600 | Descriptions, meta |
| color-text-subtle | gray-500 | Placeholders, disabled |
| color-border | gray-200 | Hairlines (1 px) |
| color-accent | tenant (default `#1F4D3A` deep green) | Primary buttons, active tab |
| color-on-accent | computed (white or gray-950) | Text on accent |
| color-success | `#2F7D4F` | Confirmed, paid |
| color-warning | `#B7791F` | Waiting, amber timers |
| color-danger | `#B83A2E` | Errors, red timers |
| color-veg | `#2E8B57` | Veg mark (Indian standard green square + dot) |
| color-nonveg | `#A0522D` | Non-veg mark (brown square + triangle, per FSSAI 2022 symbol) |
| color-egg | `#D4A017` | Egg mark |

**Rules**
- Accent is used sparingly: primary button, active tab indicator, selected options, links. Never for large backgrounds on the guest app.
- Status colours always pair with an icon and text (never colour alone).
- Dark themes (Noir, KDS) use the same semantic names with dark values (see §9).

### 2.3 Spacing

4 px base grid: `space-1 = 4`, `2 = 8`, `3 = 12`, `4 = 16`, `5 = 20`, `6 = 24`, `8 = 32`, `10 = 40`, `12 = 48`, `16 = 64`.

- Screen side padding: 20 px (phones), 24 px (tablets), 32 px (desktop admin).
- Card internal padding: 16 px.
- Vertical rhythm between sections: 32 px; between related items: 12 px.

### 2.4 Radius

`radius-xs 6` · `radius-sm 10` · `radius-md 14` (cards, inputs) · `radius-lg 20` (sheets, large images) · `radius-xl 28` (bottom sheet top corners) · `radius-full` (pills, avatars, steppers).

### 2.5 Elevation

Soft, warm, low shadows. Never harsh black.

| Token | Value |
|-------|-------|
| shadow-1 | `0 1px 2px rgba(28,26,23,0.06)` |
| shadow-2 | `0 4px 12px rgba(28,26,23,0.08)` |
| shadow-3 | `0 12px 32px rgba(28,26,23,0.12)` (sheets, cart bar) |
| shadow-focus | `0 0 0 3px color-mix(in srgb, var(--color-accent) 35%, transparent)` |

### 2.6 Breakpoints

`sm 360` (small phones) · `md 600` (large phones / small tablets) · `lg 900` (tablets landscape) · `xl 1200` (desktop) · `2xl 1600`.

---

## 3. Layout

### 3.1 Guest app
- Single column on phones, max content width 560 px centred on tablets/desktop with the cover image extending full width.
- Persistent elements: top header (collapses on scroll), category tab bar (sticky), floating cart bar (bottom, only when cart has items), bottom safe-area respected (`env(safe-area-inset-bottom)`).
- No bottom navigation bar on the menu (keeps focus on food). A compact menu button in the header opens: Your table, My orders, Call waiter, About, Language.

### 3.2 Admin app
- Desktop: left sidebar (collapsible to icons) + top bar (outlet switcher, search, notifications, profile) + content area max 1440 px.
- Tablet/phone: bottom tab bar for the 4 most used sections for that role (e.g. waiter: Floor, Orders, Requests, More).
- Dense tables use 44 px rows on desktop, become cards on phones.

### 3.3 KDS
- Landscape grid of ticket columns (3–5 depending on width), dark theme, 20 px base font, elapsed-time chips large.

---

## 4. Typography

### 4.1 Scale (guest app, rem with 16 px root, respects user font scaling)

| Token | Size / line height | Weight | Use |
|-------|-------------------|--------|-----|
| display | 32 / 38 | Display font 500 | Business name on cover |
| title-1 | 24 / 30 | Display font 500 | Category headers |
| title-2 | 20 / 26 | Text font 600 | Sheet titles, item name in sheet |
| headline | 17 / 24 | Text font 600 | Item name on card |
| body | 15 / 22 | Text font 400 | Descriptions |
| callout | 14 / 20 | Text font 500 | Prices, buttons secondary |
| caption | 12 / 16 | Text font 500 | Badges, meta |

Admin uses the same scale one step smaller for dense views. KDS uses a larger scale (item 22 px, spot 32 px).

### 4.2 Rules
- Prices use tabular numerals (`font-variant-numeric: tabular-nums`) so they align.
- Max line length 65 characters for descriptions.
- Never all-caps for more than 3 words; caps labels get +0.06em letter spacing.
- Indian Rupee sign rendered as `₹` with a thin space when needed: `₹280`.

### 4.3 Curated font pairings (tenant chooses one)

All fonts are self-hosted, subset to Latin, WOFF2, `font-display: swap`, with size-adjusted fallback to avoid layout shift. Each pairing ≤ 90 KB total.

**Hindi (D-11)**: none of the Latin display fonts include Devanagari, so every pairing has a fixed Hindi companion: **Mukta** (text, 400/600) and **Tiro Devanagari Hindi** (display, for the Classic and Elegant pairings) or **Mukta 600** (others). Devanagari fonts are loaded **only when the guest switches to Hindi** (≈ 60 KB extra), using `unicode-range` so mixed English/Hindi text picks the right font per character. Devanagari needs ~10 % more line height: Hindi mode adds `line-height` +2 px on body and title styles. Every guest screen must be designed and checked in both languages; Hindi strings are typically 20–30 % longer, so buttons and tabs must not assume English widths.

| Pairing | Display | Text | Feel |
|---------|---------|------|------|
| **Classic** (default) | Fraunces | Inter | Warm editorial, upscale restaurant |
| **Modern** | Inter Tight | Inter | Clean, cafe, contemporary |
| **Elegant** | Cormorant Garamond | Manrope | Fine dining, hotels |
| **Friendly** | DM Serif Display | DM Sans | Casual dining, family |
| **Geometric** | Space Grotesk | Space Grotesk | Bars, young brands |

Admin app always uses **Inter** (consistency, legibility).

---

## 5. Iconography and imagery

- Icon set: **Lucide** (outline, 1.5 px stroke, 20/24 px), tree-shaken per icon. Custom icons for dietary marks (vector, follow FSSAI shapes), spice level (chilli), and service requests.
- Food photos: 4:3 for list cards, 1:1 for grid and thumbnails, 16:9 or 4:3 for item sheet hero. Corners `radius-md` in cards, `radius-lg` in sheet.
- Items without photos never show a broken image or grey box: the card switches to a text layout automatically.
- Placeholders: tiny blurred preview (ThumbHash, ~25 bytes) inlined in the menu data, faded to the real image in 200 ms.
- Cover image: gentle parallax on scroll (disabled with reduced motion).

---

## 6. Motion

Motion makes the app feel physical and helps the guest understand what happened. It must never delay an action.

### 6.1 Timing tokens

| Token | Duration | Use |
|-------|----------|-----|
| motion-instant | 80 ms | Press states, toggles |
| motion-fast | 160 ms | Fades, small moves, tab indicator |
| motion-base | 240 ms | Sheets, page transitions |
| motion-slow | 360 ms | Celebrations (order placed), large reveals |

### 6.2 Easing tokens

| Token | Curve | Use |
|-------|-------|-----|
| ease-out | `cubic-bezier(0.22, 1, 0.36, 1)` | Things entering |
| ease-in | `cubic-bezier(0.55, 0, 1, 0.45)` | Things leaving (use shorter duration) |
| ease-in-out | `cubic-bezier(0.65, 0, 0.35, 1)` | Moves on screen |
| spring-gentle | stiffness 260, damping 30 | Sheets, cart bar |
| spring-bouncy | stiffness 400, damping 22 | Count bump, add confirmation |

### 6.3 Signature interactions

| Interaction | Behaviour |
|-------------|-----------|
| **Press** | Buttons and cards scale to 0.97 with `motion-instant`; release springs back. Light haptic (`navigator.vibrate(8)`) on Android for add-to-cart and stepper, only if supported and the user hasn't disabled it. |
| **Add to cart** | A small copy of the item thumbnail (or a dot in accent colour) arcs from the button to the cart bar in 450 ms; the cart count bumps (spring-bouncy); the total counts up. |
| **Cart bar appear** | Slides up from below the screen with spring-gentle, shadow-3. |
| **Item sheet** | Opens from the tapped card's photo (shared element transition where supported via View Transitions API; otherwise slide-up). Follows the finger on drag; dismisses past 30 % height or with fast flick; background dims to 40 %. |
| **Category tabs** | Active indicator slides between tabs; tab bar auto-scrolls to keep the active tab visible. |
| **Stepper** | Number rolls vertically (old number exits up, new enters from below) in 160 ms. |
| **Remove line** | Line collapses height with ease-in 200 ms; Undo toast slides up. |
| **Order placed** | Button morphs into a progress ring while sending; on confirmation the ring completes into a check and the screen transitions to the status timeline. No confetti. |
| **Status timeline** | Each step fills with a short progress sweep when reached; the current step has a soft pulse (opacity 0.6 ↔ 1, 1.6 s). |
| **Skeletons** | Shimmer of 1.2 s, very low contrast; replaced with cross-fade. |
| **Page transitions** | Forward: new screen slides in 24 px from the right with fade; back: reverse. Use View Transitions API where available. |
| **Pull to refresh** | Not used on the menu (menu updates itself). Used on admin lists on mobile. |

### 6.4 Motion rules
- Respect `prefers-reduced-motion`: replace movement with fades of ≤ 120 ms; disable parallax, flying thumbnails, and pulses.
- Animate only `transform` and `opacity` (GPU-friendly). Never animate `height` on long lists — use FLIP technique.
- Interaction must remain possible during any animation (no blocking).
- Maximum 3 simultaneous animated elements on screen.
- Implementation: CSS transitions for simple states; the **Motion** library (`motion/react`, loaded via `LazyMotion` with `domAnimation` features only) for gestures, springs, and shared layouts. Budget: ≤ 18 KB gzip for animation code on the guest app (doc 05 §12).

---

## 7. Components

Each component is documented in Storybook with all states. States required for every interactive component: default, hover (pointer devices), pressed, focus-visible, disabled, loading, error.

### 7.1 Guest components

| Component | Notes |
|-----------|-------|
| **CoverHeader** | Cover image, logo, business name, spot label chip ("Table 12"), open/closed status, info button. Collapses into a compact bar with logo + name on scroll. |
| **CategoryTabs** | Horizontal scroll, sticky, active indicator, fade edges showing more tabs. |
| **ItemCard** (list / grid / text variants) | Photo, name, dietary mark, badges, 2-line description, price, quick-add "+" or "Customise" pill; sold-out state; in-cart quantity badge. |
| **DietaryMark** | Square + symbol per FSSAI; accessible label ("Vegetarian"). |
| **Badge** | Bestseller, Chef's pick, New, Spicy (chilli count). Max 2 per card. |
| **ItemSheet** | Hero photo, info, VariantSelector, ModifierGroup list, InstructionsField, QuantityStepper, AddButton with live price. |
| **VariantSelector** | Segmented control for ≤ 3 options, radio list for more. |
| **ModifierGroup** | Header with rule hint ("Choose up to 2", "Required"); options with price delta; error highlight when unmet. |
| **QuantityStepper** | 44 × 44 px buttons, rolling number, long-press to repeat (P1). |
| **CartBar** | Count, total, "View cart"; disabled shimmer while syncing; offline indicator dot. |
| **CartLine** | Item summary, edit affordance, stepper, line total, swipe-left to remove (with button alternative). |
| **BillSummary** | Rows for subtotal, discounts, taxes (expandable to show each tax), service charge (with "Remove" link), round off, total. |
| **PrimaryButton** | Full width on mobile, 52 px height, accent background, loading spinner state, success morph. |
| **StatusTimeline** | Vertical steps with icons, times, current pulse. |
| **OTPInput** | 6 boxes, paste support, auto-advance, auto-submit on 6th digit, `autocomplete="one-time-code"`, resend countdown. |
| **PhoneInput** | Country code selector, formatting as you type, numeric keyboard (`inputmode="tel"`). |
| **Toast** | Bottom, above cart bar; types: info, success, warning, error; optional action ("Undo", "Retry"). Auto-dismiss 4 s (errors stay until dismissed). |
| **ConnectionBanner** | Thin bar under the header: "You're offline — your cart is saved" / "Back online". |
| **ServiceRequestSheet** | Grid of large request buttons with icons. |
| **EmptyState** | Illustration (simple line art), message, action. |
| **Skeletons** | For cover, tabs, cards, sheet. |

### 7.2 Admin components

| Component | Notes |
|-----------|-------|
| **AppShell** | Sidebar, top bar, outlet switcher, command palette (Cmd/Ctrl + K) for search and actions. |
| **InsightCard** | Icon by type, headline sentence, one-line explanation, action button(s), "Why am I seeing this?" disclosure, dismiss/snooze. |
| **MetricTile** | Label, value, delta vs comparison (arrow + %, coloured), sparkline. |
| **Charts** | Line/area (revenue over time), bar (items, categories), heatmap (peak hours), donut only for ≤ 5 slices (payment split). One accent colour + grays; tooltips with exact values. |
| **DataTable** | Sorting, filters, column visibility, sticky header, virtualised rows for > 200 rows, bulk selection, CSV export. |
| **OrderCard** | Board card with spot, order no, timer chip, items, guest badge, actions. |
| **SpotTile** | Floor view tile coloured by state, session total, elapsed time. |
| **ImageUploader** | Drop zone, camera capture on mobile, cropper, upload progress, processing state, retry. |
| **Toggle / AvailabilitySwitch** | Large on mobile; optimistic with rollback on failure. |
| **FormField** | Label, input, helper text, error text, character counter. Inline validation on blur; never on first keystroke. |
| **ConfirmDialog** | For destructive actions; requires typing the name for very destructive actions (delete outlet, void bill > threshold). |
| **PinPad** | Staff quick-switch and KDS pairing. |
| **PreviewPhone** | Live guest app preview in a phone frame. |

### 7.3 KDS components

**Ticket** (spot, order no, timer, lines, notes in highlighted box, bump button), **StationFilter**, **ConnectionLight**, **AllDayCounts**.

---

## 8. Content and voice

- **Tone**: warm, brief, confident. Like a good host, not a robot.
- Use the guest's name when known ("Welcome back, Riya").
- Buttons say what happens: "Add · ₹280", "Place order", "Request bill" — never "Submit" or "OK".
- Errors say what happened and what to do, never blame: "We couldn't reach the kitchen. Your order is saved and we'll keep trying." (Full copy catalogue in doc 05 §13.7.)
- Numbers: Indian digit grouping for INR (`₹1,25,000`), configurable by locale.
- Times: relative for recent ("2 min ago"), absolute for history ("12 Mar, 8:42 PM").

---

## 9. Tenant theming

### 9.1 What tenants can change
Logo (square + wide), cover image, accent colour, theme preset, font pairing, menu layout per category, welcome text.

### 9.2 What tenants cannot change
Spacing, radius, motion, component structure, error/empty states, accessibility behaviour. This protects quality across every business.

### 9.3 Theme presets

| Preset | bg | surface | text | Default accent | Notes |
|--------|----|---------|------|----------------|-------|
| **Ivory** | `#FCFBF9` | `#FFFFFF` | `#1C1A17` | `#1F4D3A` | Default, warm light |
| **Noir** | `#0F0E0C` | `#1C1A17` | `#F7F5F2` | `#C9A96E` (gold) | Dark, bars and fine dining; photos pop |
| **Garden** | `#F6F8F4` | `#FFFFFF` | `#1B231C` | `#3F6B45` | Fresh cafes |
| **Coastal** | `#F5F8FA` | `#FFFFFF` | `#16202A` | `#1E5A7A` | Seafood, resorts |

### 9.4 Accent safety algorithm
1. Tenant picks accent `A`.
2. Compute contrast of white and gray-950 against `A`; choose the higher as `color-on-accent`. If both < 4.5:1, darken/lighten `A` in OKLCH lightness until one passes, and show the owner "We adjusted your colour slightly so text stays readable".
3. Check `A` against `color-bg` for non-text use (tab indicator) ≥ 3:1; adjust similarly.
4. Generate `accent-soft` (10 % tint) for selected states.

### 9.5 Delivery
Theme compiles to a small CSS block of custom properties embedded in the menu snapshot HTML (no extra request). Fonts are preloaded by pairing.

---

## 10. Accessibility requirements

- Contrast: text ≥ 4.5:1, large text and UI parts ≥ 3:1 (WCAG 2.2 AA).
- Touch targets ≥ 44 × 44 px with ≥ 8 px spacing.
- Every interactive element reachable by keyboard (admin) and screen reader (both apps), with visible focus ring (`shadow-focus`).
- Dietary marks, badges, and status always have text alternatives.
- Sheets and dialogs trap focus, restore it on close, and close on Escape / Android back.
- Supports system font scaling up to 200 % without loss of function (layout reflows; prices never truncated).
- Live regions announce cart changes ("Iced Latte added. Cart: 2 items, ₹560") and order status changes.
- Forms: labels always visible (no placeholder-only labels), errors linked via `aria-describedby`.
- No time limits for guests except OTP expiry, which is announced.

---

## 11. Design deliverables checklist (before Phase 1 build)

- [ ] Figma (or Penpot) library with tokens, typography, components and all states.
- [ ] Guest app high-fidelity screens for every screen in doc 04 §3, in Ivory and Noir, at 360 px and 414 px widths, in English and Hindi.
- [ ] Admin key screens: dashboard, menu studio (list + item editor), orders board, floor view, bill & settle, settings.
- [ ] KDS screen in landscape tablet.
- [ ] QR card print templates (A6 tent, 8 × 8 sticker, A4 poster, room card).
- [ ] Prototype of signature motions (add to cart, sheet, order placed) to validate feel on a real mid-range Android.
