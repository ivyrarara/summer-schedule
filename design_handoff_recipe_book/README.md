# Handoff: Recipe Book (mobile web app)

## Overview
A mobile-web recipe book with two tabs (Cooking / Baking), a search, drag-to-reorder recipe cards, an "Add a recipe" flow that parses pasted text or a photo into a structured recipe via the Claude API, per-recipe ingredient scaling, inline-editable ingredients, and a "Modify" log that auto-applies ingredient swaps/quantity changes and can be reverted.

## About the Design Files
The file in this bundle (`Recipe Book.dc.html`) is a **design reference built in HTML** — a working prototype showing intended look, layout, and interaction, not production code to copy directly. The task is to **recreate this design in the target codebase's existing environment** (React Native, SwiftUI, Android/Compose, or web React/Vue — whatever the project already uses) following its established patterns, component libraries, and state-management approach. If no environment exists yet, choose the framework best suited to a mobile web/app recipe tool.

## Fidelity
**High-fidelity.** Colors, typography, spacing, and interactions in the HTML file are final — recreate them pixel-perfectly using the target codebase's own component/styling system (do not port inline styles verbatim; translate them to the codebase's conventions, e.g. Tailwind classes, styled-components, SwiftUI modifiers).

## Screens / Views

### 1. Book (list) — default screen
- Full-height single column, max-width 480px, centered, background solid burgundy `#550010`.
- Fixed header (does not scroll): oversized italic serif "My recipe book" title (Instrument Serif italic, ~92px, color `#C4342B`, opacity 0.4, sits as a decorative overlay bleeding left/up out of frame — intentionally clipped), recipe count top-right in "00N recipes" format (DM Sans, 11px, bold, letter-spacing 2px, 60% opacity cream).
- Two pill tab buttons "Cooking" / "Baking", equal width, gap 8px, no border. Active tab: solid pink `#F2A9C4` background, burgundy `#550010` text. Inactive: solid dark red `#7A0F22` background, pink text.
- Search input directly below tabs: full width minus 24px margin, pill shape, 8px vertical padding, background `rgba(0,0,0,0.28)`, inset shadow, placeholder color `#A8324B`. Filters recipes live by title or any ingredient name (substring match).
- Below the fixed header: scrollable recipe list, each row a solid card (`#3D000B` background, 18px radius, 14px padding, flex row): 58×58px initial-letter avatar (solid `#C4342B`, italic serif, 30px), title (Instrument Serif, 20px, cream) + meta line (time · N ingredients · tags, 12px, 60% cream), trailing arrow (pink, italic serif).
- Long-press (~420ms) + drag on a card reorders the list with FLIP animation on displaced siblings and a spring-back if the drag is released without a swap. Regular scroll works normally; only an actual long-press arms drag mode (see Interactions).
- A thin shadow fades in under the search bar only once the list has scrolled (opacity transitions 0→1 based on scroll position, not always visible).
- Fixed "+ New recipe" pill button bottom-center, small size, red `#C4342B` background, cream text, sits above a burgundy gradient scrim so cards don't visually collide with it.

### 2. Recipe detail
- Full-screen overlay, burgundy `#550010` background, back link "← back to the book" (pink italic serif, top-left).
- Title row: large italic serif title (36px, pink) + a small link icon button to its right (🔗). Tapping the icon when no URL is set reveals an inline URL input (auto-focus, Enter/blur commits); once a URL exists the icon turns solid red and tapping opens the link in a new tab.
- Tag chips (pill, red border/text) plus a "scaled ×N" chip when a scale factor is active (tap to reset to ×1).
- **Ingredients** section header: italic serif, 23px, red `#C4342B`, 80% opacity text with a fully-opaque red underline positioned to strike through the text (not below it).
  - Each ingredient row: quantity/amount (tap to edit inline, commits on Enter/blur), ingredient name (tap to edit inline separately), an "×" remove button (dim cream, 22px). Row separator is a dotted red line (radial-gradient dot pattern, not a CSS dashed border).
  - Editing a quantity rescales the **entire recipe** proportionally (multiplies every ingredient's quantity and servings by the new/old ratio) — not just that one ingredient.
  - New-ingredient inline add row below the list (qty + name inputs, no explicit add button — Enter or blur on the name field commits it).
- **How to** section (same header treatment): numbered steps, italic serif step number (22px) tight-spaced (6px gap) to the step text (15px, cream).
- **Modify** section (same header treatment):
  - Logged modifications render as plain text (no card/box), tightly stacked (no vertical gap), format `M/D — description`, dimmed italic cream text, with an "×" (same style as ingredient remove) to revert — reverting restores the exact prior ingredient object and deletes the log entry.
  - A bordered input below the log ("Add your own tweak, e.g. swap butter → oil") — Enter or blur (not a button) parses the text and applies it live:
    - Pattern `"swap X → Y"` (or `->`): finds ingredient whose name contains X (word-stem matched, so "tomato" matches "tomatoes") and replaces its name with Y.
    - Free text mentioning an ingredient name plus a number+unit (e.g. "used 1.5 tbsp olive oil"): finds the best-matching ingredient by name overlap and updates its quantity/unit.
    - Every applied modification is timestamped (`M/D`) and stored with enough info (previous ingredient snapshot) to revert.

### 3. Add a recipe (overlay)
- Burgundy background, "← never mind" back link, italic serif "Add a recipe" title, one-line description.
- Small photo-drop button/box (dotted red border, click-to-upload or drag-and-drop) — resizes the image client-side (max 1000px, JPEG q0.7) before sending, then calls the Claude API with an image content block to parse the photo directly into a recipe (title, tab, tags, time, servings, ingredients, steps, 1-2 modification suggestions), and adds it to the book.
- "or" divider (solid thin lines either side, not dashed).
- Large textarea for pasted recipe text, dark solid fill, pink placeholder text.
- "Save this recipe" full-width pill button (red) — sends the pasted text to Claude with a JSON-schema prompt, parses the JSON response, and adds the recipe to the book (auto-selects the correct tab).
- Inline error message (pink italic) shown if parsing/JSON fails.

## Interactions & Behavior
- **Tab switching**: instant, filters the visible list by `tab` field.
- **Search**: live substring filter on title + every ingredient name; does not remove non-matches from cooking/baking count elsewhere.
- **Drag reorder**: pointer-down arms a timer (~420ms); if the pointer moves more than ~6px before the timer fires, treat as a scroll gesture instead (manually forwards the delta to the list's `scrollTop`, since `touch-action: none` is required on the row for drag to work reliably and that disables native scroll on that element only, not with two-step touch/hold). Firing the timer arms drag: row gets `position: relative`, lifted shadow, 1.03 scale, follows the pointer via translateY; crossing the vertical midpoint of a neighboring row swaps array order with a FLIP transform animation (~0.28s ease). Releasing without a swap snaps back to the original slot with the same FLIP technique.
- **Ingredient quantity edit**: click the amount → inline input (same font/size/position as the static label, so no layout shift) → Enter or blur commits → recomputes a scale `factor` for the whole recipe = newValue / originalValue, applied multiplicatively to every ingredient's displayed quantity and to servings.
- **AI parsing (text)**: `window.claude.complete(promptString)` with an explicit JSON schema instruction; strips markdown fences; `JSON.parse`s the substring between the first `{` and last `}`; on failure shows an inline error and leaves the textarea intact.
- **AI parsing (photo)**: same idea but sends `window.claude.complete({ messages: [{ role: 'user', content: [{type:'image', source:{type:'base64', media_type, data}}, {type:'text', text: prompt}] }] })` after client-side downscaling (canvas → JPEG base64).
- **Recipe link**: click icon → if no `link` set, reveal URL input; committing prefixes `https://` if missing; if `link` is set, click opens `window.open(link, '_blank')` instead of re-editing (need a separate small "edit" affordance if you want to change an existing link — current prototype doesn't have one beyond clearing state manually).

## State Management
Per-recipe fields: `id, tab, title, time, servings, factor, tags[], ings[{q, u, n}], steps[], mods[{text, date, ingIdx, prevIng}], link`.
Screen-level state: active `tab`, `selId` (open recipe), `screen` ('book' | 'add'), search `query`, drag state (`dragId`, `dragClientY`, `armId`), inline-edit state for quantity/name/link/new-ingredient/mod-input, parsing/error flags.

## Design Tokens
- Burgundy background: `#550010`
- Card/panel fill (darker): `#3D000B`
- Inactive tab / photo-box fill: `#7A0F22` / `#480009` (two slightly different dark reds used in different places — consolidate to one token when implementing)
- Primary red (headers, buttons, active accents): `#C4342B`
- Pink (secondary accent, links, placeholders on dark red): `#F2A9C4`
- Placeholder/dim text: `#A8324B` (on dark inputs), `rgba(252,237,230,0.5–0.65)` (dim cream on burgundy)
- Cream text: `#FCEDE6`
- Fonts: "Instrument Serif" (italic) for display/headers, "DM Sans" for body/UI text
- Radius: 999px (pills/tabs/search), 18px (recipe card), 14px (avatar), 0px (inline edit inputs — intentionally square)
- Card shadow while dragging: `0 12px 28px rgba(0,0,0,0.4)`

## Assets
No external images — avatars are solid-color initials, no photo attachments in this version (a "photo of the dish" slot was tried and explicitly removed by the user as unwanted).

## Files
- `Recipe Book.dc.html` — full source (template + logic class) for the entire app, single file.
