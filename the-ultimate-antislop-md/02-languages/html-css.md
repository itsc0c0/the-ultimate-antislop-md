# HTML/CSS

## Semantic HTML

- **DO:** Use `<em>`/`<strong>` for genuine semantic emphasis and importance, and reserve purely visual bold/italic styling for `<b>`/`<i>` (which do still have legitimate, narrower uses — a keyword the first time it's introduced, a ship's name) or plain CSS `font-weight`/`font-style` — a screen reader can announce `<strong>` with vocal emphasis, which a CSS-only bold styling on a `<span>` does not provide.
- **DO:** Reach for the element that actually describes the content's meaning before reaching for a generic `<div>`/`<span>` — `<nav>` for navigation, `<article>` for self-contained content, `<section>` for a thematic grouping, `<header>`/`<footer>` for introductory/closing content, `<button>` for anything clickable that performs an action, `<a>` for anything that navigates. Semantic elements carry meaning that assistive technology, search engines, and browsers' built-in behavior (keyboard focus, default styling, form submission) all rely on.
  ```html
  <!-- BAD — "div soup": no element conveys any structural meaning -->
  <div class="nav">
    <div class="nav-item" onclick="go('/home')">Home</div>
  </div>

  <!-- GOOD — semantic elements carry meaning and correct default behavior -->
  <nav>
    <a href="/home">Home</a>
  </nav>
  ```
- **DON'T:** Build a clickable "button" out of a `<div>` or `<span>` with a click handler. A real `<button>` (or `<a>` for navigation) is keyboard-focusable and activatable by default, announces its role to screen readers automatically, and participates in form submission where relevant — a `<div onclick>` gets none of that without manually re-implementing `tabindex`, `role="button"`, and keyboard event handling, and it's easy to miss a case.
- **DO:** Use heading elements (`<h1>`–`<h6>`) to reflect the actual document outline/hierarchy, with exactly one `<h1>` per page (or per major sectioning root) representing the page's primary topic, and no skipped heading levels (an `<h2>` followed directly by an `<h4>`) — screen reader users commonly navigate by jumping between headings, and a broken hierarchy makes that navigation misleading.
- **DO:** Use `<ul>`/`<ol>`/`<li>` for any actual list of items (navigation links, a list of features, search results), not a sequence of `<div>`s or `<br>`-separated lines — list semantics let assistive technology announce "list of 5 items" and let users navigate item-by-item.
- **DO:** Use `<table>` (with `<thead>`, `<tbody>`, `<th scope="col">`/`<th scope="row">`) specifically for tabular data — content that genuinely has rows and columns of related values — never for layout purposes, and never fake tabular data with nested `<div>`s styled to look like a grid when the content actually is a table.
- **DON'T:** Use a heading element purely for its visual size (choosing `<h3>` because it "looks right" at that font size) rather than its position in the actual document outline. Style headings with CSS to achieve the desired visual weight; choose the heading *level* based on structure alone.
- **DO:** Use `<button type="button">` for buttons that don't submit a form (to avoid the default `type="submit"` behavior firing unexpectedly inside a `<form>`) and reserve unmodified `<button>`/`<button type="submit">` for actual form-submitting actions.
- **DO:** Use `<main>` to wrap a page's primary, unique content (excluding repeated site-wide chrome like the nav and footer), with exactly one `<main>` per page, so assistive technology users can jump straight to it via a "skip to content" mechanism or landmark navigation.
- **DON'T:** Nest interactive elements inside one another (a `<button>` inside an `<a>`, or a `<a>` inside a `<button>`) — this produces invalid HTML with unpredictable, browser-dependent focus and activation behavior. Choose one interactive element per interactive region and style it to contain whatever visual content is needed.
- **DO:** Use `<figure>`/`<figcaption>` to associate an image, diagram, or code sample with its caption, so the relationship is structurally explicit rather than implied purely by visual proximity.
- **DO:** Use `<time datetime="2026-09-04">` to mark up a human-readable date/time with its machine-readable ISO value, so assistive technology, browsers, and scripts can interpret the date correctly regardless of how it's formatted for display.
  ```html
  <time datetime="2026-09-04">September 4th</time>
  ```
- **DO:** Use `<dialog>` for a native modal dialog where browser support allows, since it provides built-in focus trapping, `Escape`-to-close, and top-layer stacking behavior that a hand-rolled `<div>`-based modal has to reimplement manually and often gets subtly wrong.
- **DO:** Use `<details>`/`<summary>` for a native, keyboard-accessible disclosure widget (an expandable FAQ answer, a collapsible section) instead of a custom JavaScript-driven show/hide `<div>` when the interaction is a simple expand/collapse, since the native element handles keyboard interaction and accessible state announcement for free.
  ```html
  <details>
    <summary>What is your refund policy?</summary>
    <p>Refunds are processed within 5 business days.</p>
  </details>
  ```
- **DON'T:** Use a `<br>` sequence to fake paragraph or list spacing (`Line one<br><br>Line two`) instead of actual `<p>` elements or a real list — this produces no structural boundary an assistive technology or a stylesheet can reliably target.
- **DO:** Use `<address>` specifically for contact information for its nearest `<article>`/document (not for a postal address in unrelated content), matching its actual semantic meaning rather than reaching for it purely because it visually resembles what's needed.
- **DO:** Mark a form's required fields with the `required` HTML attribute (which also enables built-in browser validation and is announced by assistive technology) rather than only a visual asterisk that a screen reader user has no way to perceive as meaning "required."
- **DO:** Use `<select>` for a genuine fixed-choice dropdown, and reserve a custom-built dropdown widget for cases with requirements a native `<select>` can't meet (rich item content, multi-select with search) — a native `<select>` gets correct keyboard support, mobile-optimized native pickers, and form submission behavior for free.
- **DO:** Group related form controls with `<fieldset>` and a `<legend>` describing the group (a set of radio buttons for one question, a set of checkboxes for one category) so assistive technology announces the group's purpose before reading its individual controls.
  ```html
  <fieldset>
    <legend>Shipping method</legend>
    <label><input type="radio" name="shipping" value="standard"> Standard</label>
    <label><input type="radio" name="shipping" value="express"> Express</label>
  </fieldset>
  ```
- **DON'T:** Use a heading (`<h1>`-`<h6>`) or a bolded `<span>` where a real form `<label>` is what's actually needed — a visually label-like element that isn't a real `<label>` provides no programmatic association with its input field.
- **DO:** Use the correct `<input type>` for the data being collected (`email`, `tel`, `number`, `date`, `url`) rather than a generic `type="text"` for everything — the specific type triggers the right mobile keyboard layout, enables relevant built-in validation, and is announced with more specific semantics by assistive technology.
- **DO:** Use `<output>` to represent the result of a calculation tied to specific form inputs (a computed total in a checkout form), so its relationship to the inputs it depends on is structurally declared via its `for` attribute.
- **DO:** Structure a multi-column layout's markup in a logical reading order in the underlying HTML even when CSS visually rearranges the columns, since screen readers and keyboard tab order follow the DOM order, not the visual/CSS order — a `flex-direction`- or grid-`order`-reversed layout can create a confusing mismatch between what's seen and what's read/tabbed through if the underlying markup order isn't chosen deliberately.

## Accessibility Basics

- **DO:** Give every meaningful `<img>` a descriptive `alt` attribute that conveys the image's purpose or content in context, and give a purely decorative image an empty `alt=""` (not a missing `alt` attribute) so screen readers skip it silently instead of announcing the image's filename or "unlabeled image."
  ```html
  <!-- BAD — missing alt forces a screen reader to fall back to the filename -->
  <img src="quarterly-revenue-chart-2026.png">

  <!-- GOOD — describes what the image conveys, not just that it exists -->
  <img src="quarterly-revenue-chart-2026.png"
       alt="Quarterly revenue grew from $2M to $5M across 2025–2026">

  <!-- GOOD — purely decorative, explicitly empty so it's skipped -->
  <img src="divider-flourish.png" alt="">
  ```
- **DO:** Associate every form input with a visible `<label>`, either by wrapping the input in the label or by pairing `<label for="field-id">` with a matching `id` on the input — a placeholder is not a substitute for a label (it disappears once text is entered, and isn't reliably announced as a label by all assistive technology).
  ```html
  <!-- BAD — placeholder is not a label, and disappears on input -->
  <input type="email" placeholder="Email address">

  <!-- GOOD — a real, persistent, programmatically associated label -->
  <label for="email">Email address</label>
  <input type="email" id="email" name="email">
  ```
- **DO:** Use landmark roles/elements (`<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`, or explicit `role="banner"`/`role="navigation"` where a native element can't be used) so assistive technology users can jump directly between a page's major regions instead of reading through everything linearly.
- **DO:** Ensure every interactive element is reachable and operable via keyboard alone — verify `Tab` order is logical, focus is visible (don't remove the default focus outline with `outline: none` without providing an equally visible custom focus style), and custom interactive widgets (a custom dropdown, a modal) trap and restore focus correctly.
  ```css
  /* BAD — removes focus visibility with nothing to replace it */
  button:focus { outline: none; }

  /* GOOD — a clearly visible custom focus style */
  button:focus-visible { outline: 2px solid #2563eb; outline-offset: 2px; }
  ```
- **DO:** Maintain sufficient color contrast between text and its background — WCAG AA calls for at least 4.5:1 for normal-size text and 3:1 for large text — and never rely on color alone to convey meaning (an error state indicated only by red text, with no icon or text label, is invisible to colorblind users).
- **DO:** Use `aria-label`, `aria-labelledby`, or `aria-describedby` to give an accessible name/description to an interactive element whose visible content alone doesn't convey its purpose (an icon-only button, for instance), but prefer visible text content over an ARIA-only label whenever the design allows it — visible text helps sighted users too, and an invisible-only label is easy to let drift out of sync with the actual behavior.
  ```html
  <!-- An icon-only button needs an accessible name since it has no text -->
  <button aria-label="Close dialog">
    <svg aria-hidden="true">...</svg>
  </button>
  ```
- **DON'T:** Add ARIA attributes reflexively to elements that already have correct native semantics — a `<button role="button">` is redundant, and a `<nav role="navigation">` adds nothing a native `<nav>` doesn't already provide. Reserve ARIA for genuinely filling a semantic gap native HTML can't express, following the general principle "no ARIA is better than bad ARIA."
- **DO:** Test with an actual screen reader (VoiceOver, NVDA, JAWS) and keyboard-only navigation, not just automated accessibility linting — automated tools (axe, Lighthouse) catch a meaningful subset of issues (missing alt text, low contrast, missing labels) but cannot verify that the experience genuinely makes sense navigated non-visually.
- **DO:** Ensure touch targets (buttons, links, form controls) are large enough to comfortably tap — commonly recommended around 44×44 CSS pixels minimum — with adequate spacing between adjacent targets, so the interface is usable on a touchscreen without frequent mis-taps.
- **DO:** Set `lang` on the `<html>` element (`<html lang="en">`) so screen readers use correct pronunciation rules, and update it for any embedded content in a different language with its own `lang` attribute on the containing element.
- **DO:** Announce dynamic content changes that occur without a page reload (a form validation error appearing, a live search result count updating, a toast notification) using an `aria-live` region (`aria-live="polite"` for non-urgent updates, `"assertive"` for urgent ones) so screen reader users are informed of the change without needing to re-scan the page.
  ```html
  <div aria-live="polite" role="status">
    3 results found
  </div>
  ```
- **DO:** Manage focus explicitly after a significant UI change that removes or replaces the currently focused element (closing a modal, navigating to a new view in a single-page app) — move focus to a sensible next target (the element that opened the modal, the page's main heading) rather than letting focus silently reset to the document body, which disorients keyboard and screen reader users.
- **DO:** Provide visible text alternatives or labels for form validation errors tied to the specific field they concern (`aria-describedby` pointing at an inline error message next to the field), rather than a single generic error banner at the top of the form that doesn't indicate which field actually failed.
  ```html
  <label for="password">Password</label>
  <input type="password" id="password" aria-describedby="password-error" aria-invalid="true">
  <span id="password-error">Password must be at least 8 characters</span>
  ```
- **DON'T:** Use `tabindex` values greater than `0` to manually control tab order. A positive `tabindex` creates a separate, hard-to-maintain tab sequence that overrides the natural DOM order and almost always produces a confusing, inconsistent experience once the page has more than a couple of such overrides — use `tabindex="0"` (join the natural order) or `tabindex="-1"` (programmatically focusable, not in the tab order) instead, and fix tab order by reordering the actual markup when possible.
- **DO:** Provide accessible names for icon fonts and SVG icons used as meaningful (non-decorative) content — either visually hidden text (a `.sr-only`-style class) alongside the icon, or an `aria-label` on the icon's container, since an icon alone conveys no information to a screen reader without an icon font's ligature/Unicode fallback text being meaningful (which it usually isn't).

## CSS Architecture

- **DON'T:** Reach for `!important` to win a specificity fight. It overrides the normal cascade, makes the actual winning rule harder to find (since a later, more specific rule can no longer simply override it without its own `!important`), and tends to spread — one `!important` used to patch a specificity problem often triggers another one nearby to patch the patch.
  ```css
  /* BAD — escalates a specificity problem instead of fixing it */
  .card .title { color: red !important; }

  /* GOOD — fix the actual specificity/source-order issue instead */
  .card-title { color: red; }
  ```
- **DO:** Keep selector specificity as flat and low as reasonably possible — prefer single class selectors over deeply nested descendant selectors (`.card .header .title` → `.card-title`), since low, consistent specificity means later rules can override earlier ones predictably without an arms race.
- **DO:** Adopt a consistent naming methodology (BEM's `block__element--modifier`, or a utility-first system's atomic class names) across the project, rather than mixing several naming conventions — a consistent methodology makes a class name's scope and purpose predictable without reading the CSS that defines it.
  ```css
  /* BEM: block, element, modifier — scope is legible from the name alone */
  .card { }
  .card__title { }
  .card--featured { }
  ```
- **DO:** Choose deliberately between a utility-first approach (composing pre-defined single-purpose classes like `flex`, `p-4`, `text-lg`) and a component-based approach (semantic classes like `.card`, `.button--primary` bundling multiple properties), based on team size, design-system maturity, and how much visual variation the project needs — and apply the chosen approach consistently rather than mixing both styles arbitrarily within the same codebase.
- **DON'T:** Use ID selectors (`#header { }`) for styling. IDs carry very high specificity that's hard to override later without another ID selector or `!important`, and an ID is meant to be a unique per-page identifier for scripting/anchor purposes, not a general styling hook.
- **DO:** Scope component styles to avoid leaking into unrelated parts of the page — via CSS Modules, a naming convention like BEM, Shadow DOM encapsulation, or CSS-in-JS scoping — rather than writing broad, unscoped selectors (`.title { }`) that can unintentionally style every element matching that class anywhere on the page.
- **DO:** Use CSS custom properties (`--color-primary: #2563eb;`) for design tokens (colors, spacing scale, typography values) referenced from multiple places, so a single source-of-truth change (updating the token) propagates everywhere instead of requiring a find-and-replace across many files.
  ```css
  :root {
    --color-primary: #2563eb;
    --space-md: 1rem;
  }
  .button { background: var(--color-primary); padding: var(--space-md); }
  ```
- **DON'T:** Let specificity grow unbounded across a file by nesting selectors more deeply "just to be safe" that a style applies. Each added level of nesting/qualification raises the bar for any future rule that needs to override it; prefer a single well-named class over an increasingly specific selector chain.
- **DO:** Organize stylesheets predictably (by component, by page, or by a documented architecture like ITCSS) so a given style's source is easy to locate, rather than one large, undifferentiated stylesheet where rules for unrelated components are interleaved in the order they happened to be added.
- **DO:** Remove unused CSS periodically (via tooling like PurgeCSS or manual audit) — dead selectors accumulate silently in long-lived projects and make the actual live styling harder to reason about, since a reader can't tell at a glance whether a given rule still applies to anything.
- **DO:** Rely on the natural cascade and source order for the common case (later rules winning over earlier ones of equal specificity) rather than fighting it with specificity escalation — placing override rules later in the file, or later in the import order, is usually simpler and more maintainable than making an earlier rule artificially more specific just so it can be overridden more easily elsewhere.
- **DO:** Use the `:where()` pseudo-class (which contributes zero specificity) when writing default/reset-level styles meant to be trivially overridable by any component-level rule, so foundational styles never accidentally out-specify the component styles meant to customize them.
  ```css
  /* :where() keeps this reset at zero specificity, so any
     component-level selector can override it without a fight */
  :where(ul, ol) { margin: 0; padding: 0; list-style: none; }
  ```
- **DON'T:** Write deeply nested Sass/Less/CSS-nesting selectors that mirror the HTML's DOM structure one-to-one (`.page .content .card .header .title { }`). This couples the stylesheet tightly to a specific markup structure, produces high-specificity selectors that are hard to override, and breaks the moment the markup is refactored even slightly.
- **DO:** Use logical properties (`margin-inline-start`, `padding-block-end`) instead of physical ones (`margin-left`, `padding-bottom`) when building components that need to support right-to-left languages, so a single set of styles adapts automatically to text direction instead of requiring a separate RTL override stylesheet.
- **DO:** Establish and document a small set of reusable spacing/sizing scale values (via custom properties or a utility framework's built-in scale) rather than letting arbitrary pixel/rem values (`13px`, `17px`, `22px`) proliferate across the codebase — a constrained scale keeps visual rhythm consistent and makes design changes a matter of adjusting the scale, not hunting down every one-off value.
- **DO:** Use CSS layers (`@layer`) to explicitly order groups of rules (resets, base styles, components, utilities, overrides) by intent rather than relying purely on source order and specificity to control which group wins — layers let a lower-specificity rule in a later layer still lose to an earlier layer deliberately, decoupling "when it was written" from "how much it should override."
  ```css
  @layer reset, base, components, utilities;
  @layer components { .card { padding: 1rem; } }
  @layer utilities { .p-0 { padding: 0; } } /* wins over .card's padding
                                                 regardless of source order */
  ```
- **DON'T:** Write duplicate vendor-prefixed rules by hand for properties that no longer need them in any currently supported browser — check actual current browser support (via a tool like caniuse.com or Autoprefixer's browserslist config) before adding prefixes, since unnecessary prefixes are dead weight and a sign the stylesheet hasn't been revisited in a while.
- **DO:** Use a `postcss`/build-step-based autoprefixer keyed to the project's actual declared browser support target, rather than hand-maintaining vendor prefixes, so prefix coverage stays correct as browser support requirements evolve.
- **DO:** Prefer CSS Grid for two-dimensional layouts (an actual grid of rows and columns — a page shell, a card grid, a dashboard) and Flexbox for one-dimensional distribution (a row of nav items, a vertically stacked form) — reaching for the tool that matches the layout's actual dimensionality avoids the awkward workarounds either one needs when used for the other's job.
  ```css
  /* Grid: a genuine two-dimensional page shell */
  .page { display: grid; grid-template-columns: 240px 1fr; grid-template-rows: auto 1fr auto; }

  /* Flexbox: one-dimensional distribution within a single row */
  .toolbar { display: flex; justify-content: space-between; align-items: center; }
  ```
- **DO:** Respect `prefers-reduced-motion` by wrapping non-essential animations/transitions in a media query that disables or substantially reduces them for users who've requested it at the OS level — motion sickness and vestibular disorders make unconditional animation a genuine accessibility barrier, not just an aesthetic preference.
  ```css
  @media (prefers-reduced-motion: reduce) {
    * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
  }
  ```
- **DO:** Implement dark mode via the `prefers-color-scheme` media query combined with CSS custom properties for color tokens (redefining the token values inside the media query, rather than duplicating every color rule), and provide an explicit user override (a toggle backed by a `data-theme` attribute) when the design calls for one, following the same "system default plus explicit override" pattern most platforms use.
- **DO:** Provide a print stylesheet (`@media print`) for content genuinely likely to be printed (invoices, articles, receipts) that hides non-printable chrome (navigation, ads, interactive controls) and adjusts colors/layout for a printed page, rather than printing the same interactive-screen layout verbatim.
- **DO:** Set essential `<meta>` tags on every page — a `charset` declaration, `viewport`, a descriptive `<title>`, and a `description` meta tag — as baseline hygiene, plus Open Graph tags (`og:title`, `og:image`, `og:description`) when the page is meant to be shared on social platforms, so link previews render meaningfully instead of falling back to generic or blank content.

## Responsive Design Fundamentals

- **DO:** Use relative units (`rem` for typography and spacing that should scale with the user's font-size preference, `%`/`fr`/`vw`/`vh` for layout proportions) instead of hardcoded pixel values for anything that should adapt to viewport size or user preferences, and reserve `px` for things that genuinely shouldn't scale (hairline borders, for instance).
  ```css
  /* BAD — fixed pixel widths break on smaller viewports and ignore
     the user's font-size preference for text sizing */
  .container { width: 960px; }
  h1 { font-size: 32px; }

  /* GOOD — scales with viewport and respects user font-size settings */
  .container { max-width: 60rem; width: 100%; }
  h1 { font-size: 2rem; }
  ```
- **DO:** Design mobile-first — write base styles for the smallest viewport, then layer on `min-width` media queries to progressively enhance the layout for larger screens — rather than designing for desktop first and using `max-width` queries to squeeze the layout down, which tends to produce more overrides and a worse small-screen experience as an afterthought.
  ```css
  /* Mobile-first: base styles apply to the smallest screens by default */
  .grid { display: grid; grid-template-columns: 1fr; }

  @media (min-width: 768px) {
    .grid { grid-template-columns: repeat(2, 1fr); }
  }
  @media (min-width: 1024px) {
    .grid { grid-template-columns: repeat(3, 1fr); }
  }
  ```
- **DO:** Use modern layout tools (CSS Grid for two-dimensional layouts, Flexbox for one-dimensional distribution) instead of older float-and-clearfix or absolute-positioning hacks — they handle responsive reflow, alignment, and gap spacing natively without the fragile workarounds float-based layout required.
- **DO:** Set the viewport meta tag (`<meta name="viewport" content="width=device-width, initial-scale=1">`) on every page, since without it mobile browsers render the page at a desktop-width virtual viewport and then scale it down, defeating any responsive CSS.
- **DON'T:** Design layouts assuming a fixed set of device widths (checking only "mobile" at 375px and "desktop" at 1440px). Real viewports span a continuous range; test — and design fluidly for — the space between and beyond the specific breakpoints chosen, not just the exact breakpoint values.
- **DO:** Use `max-width` (not a fixed `width`) on images and media (`img { max-width: 100%; height: auto; }`) so they scale down within their container on narrow viewports instead of overflowing it or forcing horizontal scroll.
- **DO:** Use `clamp()` for fluidly scaling typography and spacing between a minimum and maximum bound (`font-size: clamp(1.25rem, 2vw + 1rem, 2rem);`) when a value should scale smoothly with viewport width rather than jumping abruptly at each breakpoint.
- **DON'T:** Disable pinch-to-zoom (`user-scalable=no` or `maximum-scale=1` in the viewport meta tag) — this removes an accessibility feature low-vision users rely on, for negligible design benefit, and is explicitly flagged by accessibility guidelines as harmful.
- **DO:** Test responsive layouts at the actual content's realistic length (long names, long navigation labels, empty states, overflowing tables) at each breakpoint, not just with short placeholder text — a layout that only works with "Lorem ipsum"-length content commonly breaks on real-world data.
- **DO:** Use container queries (`@container`) where component-level responsiveness (a card that reflows based on the width of its own container, not the viewport) is genuinely needed, since a component reused in both a narrow sidebar and a wide main area can't be made responsive correctly with viewport-based media queries alone.
- **DO:** Use `srcset`/`sizes` on `<img>` (or `<picture>` with multiple `<source>` elements) to serve appropriately sized images per viewport/device-pixel-ratio, instead of shipping one large image and scaling it down with CSS — the latter still downloads the full-size file on every device, wasting bandwidth on mobile connections.
  ```html
  <img src="photo-800.jpg"
       srcset="photo-400.jpg 400w, photo-800.jpg 800w, photo-1600.jpg 1600w"
       sizes="(max-width: 600px) 100vw, 50vw"
       alt="Product photo">
  ```
- **DO:** Set explicit `width`/`height` attributes (or an `aspect-ratio` in CSS) on images and embedded media so the browser can reserve the correct space before the media finishes loading, preventing the layout-shifting "jump" as content loads in — this directly affects the Cumulative Layout Shift metric and the perceived stability of the page.
- **DON'T:** Rely purely on JavaScript-driven breakpoint detection (measuring `window.innerWidth` and conditionally rendering different markup) for layout changes that CSS media/container queries handle natively and more reliably — reserve JS-driven responsive logic for behavior that genuinely can't be expressed in CSS (loading different data, not just different layout).
- **DO:** Test text reflow and zoom behavior up to 200% browser zoom (a WCAG success criterion) to confirm content remains readable and doesn't get clipped or overlap, not just that the layout "looks fine" at 100% zoom on a standard viewport.

## Common AI-Assistant Mistakes

- **DON'T:** Default to "div soup" — wrapping every piece of content in generic `<div>`s and `<span>`s with class names doing all the semantic work — instead of the semantic element that actually fits (`<nav>`, `<button>`, `<article>`, `<ul>`). Generated markup should reach for the correct element first and style it, rather than reaching for a `<div>` and trying to make it "act like" the correct element with ARIA and JavaScript.
- **DON'T:** Generate inline `style="..."` attributes scattered across markup as the default styling approach. Inline styles can't be reused, can't be overridden by normal cascade rules without `!important`, and separate presentation from structure far less cleanly than an external or component-scoped stylesheet — reserve inline styles for genuinely dynamic, computed-at-runtime values (e.g., a JS-computed position), not static design decisions.
  ```html
  <!-- AVOID as a default habit — unreusable, hard to override, no cascade -->
  <div style="color: #2563eb; padding: 16px; border-radius: 8px;">...</div>

  <!-- PREFER — reusable, overridable, and separates structure from style -->
  <div class="card">...</div>
  ```
- **DON'T:** Generate fixed-pixel, non-responsive layouts (`width: 1200px` on a top-level container with no fluid fallback) by default. Verify the layout adapts sensibly from mobile through desktop widths, using relative units and media/container queries, rather than assuming a single target viewport size.
- **DON'T:** Omit `alt` attributes, form labels, and landmark structure from generated markup. Accessibility is one of the most commonly skipped concerns in AI-generated HTML — treat labeling every image and form input, and using semantic landmark elements, as a default requirement of "correct" markup, not an optional add-on pass.
- **DON'T:** Reach for `!important` to resolve a CSS specificity conflict in generated code instead of fixing the actual selector/specificity issue. This is a common shortcut that papers over a structural problem the generated CSS itself created (overly specific selectors, ID selectors used for styling, poor source ordering).
- **DON'T:** Generate a clickable `<div>` with an `onclick` handler in place of a `<button>` or `<a>`, even for a quick prototype — the missing keyboard accessibility and semantic role are easy to overlook in a demo and easy to forget to fix before the code ships.
- **DON'T:** Mix multiple CSS naming/architecture conventions arbitrarily within the same generated stylesheet (some BEM-style classes, some utility classes, some ID selectors, some inline styles) without checking what convention the existing project already uses — match the established pattern instead of introducing a new one per generation.
- **DON'T:** Generate color combinations without checking contrast ratios, especially for text on colored/branded backgrounds and for placeholder/disabled-state text, which frequently ends up under the WCAG AA contrast minimum in generated designs. Check contrast values, don't just eyeball that colors "look readable."
- **DON'T:** Generate a custom modal, dropdown, tooltip, or tab widget entirely from scratch with unmanaged focus and no keyboard support when a native element (`<dialog>`, `<details>`) or an established accessible pattern (the WAI-ARIA Authoring Practices patterns for these exact widgets) already covers the interaction correctly — reinventing these from scratch is where most custom-widget accessibility bugs come from.
- **DON'T:** Ship large, unsized images or unbounded media without `width`/`height`/`aspect-ratio`, causing layout shift as the page loads — set explicit dimensions or aspect ratios by default for any generated media element.
- **DON'T:** Generate a form with no client-side or server-side validation feedback tied to specific fields, or with validation errors that appear only as a color change with no text — pair every validation state with a visible, field-associated text message.
- **DON'T:** Assume a generated page's layout is complete after checking it at exactly one viewport width. Verify the generated CSS holds up across a realistic range from small mobile widths through large desktop widths, not just the one size it happened to be checked at during generation.

## Quick Checklist
- Use the semantically correct element (`<nav>`, `<button>`, `<article>`, `<ul>`) instead of a generic `<div>`/`<span>` with a class doing the semantic work.
- Never build a clickable control out of a `<div onclick>`; use `<button>` or `<a>`.
- Keep exactly one `<h1>` per page and never skip heading levels.
- Wrap tabular data in a real `<table>`; never fake a grid with styled `<div>`s or use `<table>` for layout.
- Give every meaningful image a descriptive `alt`; give decorative images `alt=""`.
- Associate every form input with a real, persistent `<label>` — a placeholder is not a label.
- Use landmark elements (`<header>`, `<nav>`, `<main>`, `<footer>`) so assistive tech can jump between regions.
- Verify full keyboard operability and visible focus states; never remove `outline` without an equally visible replacement.
- Meet WCAG AA contrast minimums (4.5:1 normal text, 3:1 large text); never convey meaning by color alone.
- Use ARIA only to fill a genuine semantic gap — don't add it redundantly to elements with correct native semantics.
- Test with an actual screen reader and keyboard-only navigation, not just automated linting.
- Never use `!important` to resolve a specificity fight — fix the underlying selector/specificity issue.
- Keep selector specificity flat; avoid ID selectors and deep descendant chains for styling.
- Adopt one consistent naming methodology (BEM, utility-first, etc.) and apply it consistently.
- Scope component styles so they can't leak into unrelated parts of the page.
- Use CSS custom properties for design tokens (color, spacing, type scale) instead of repeating literal values.
- Use relative units (`rem`, `%`, `fr`) for anything that should scale; reserve `px` for things that genuinely shouldn't.
- Design mobile-first with `min-width` media queries, not desktop-first squeezed down with `max-width` queries.
- Set the viewport meta tag on every page; never disable pinch-to-zoom.
- Use CSS Grid/Flexbox for layout instead of float/clearfix or absolute-positioning hacks.
- Make images/media fluid (`max-width: 100%; height: auto`) instead of fixed-pixel-width.
- Test responsive layouts with realistic (long, overflowing) content, not just short placeholder text.
- Never default to inline `style=""` attributes for static design decisions.
- Never omit accessibility basics (alt text, labels, landmarks) from generated markup as an afterthought.
- Match the project's existing CSS architecture/naming convention instead of introducing a new one per generation.
- Check color-contrast ratios explicitly rather than eyeballing readability.
- Use `<time>`, `<details>`/`<summary>`, and `<dialog>` where they fit instead of custom-built equivalents.
- Mark required form fields with the `required` attribute, not just a visual asterisk.
- Announce dynamic content changes with `aria-live`; manage focus explicitly after modal/view changes.
- Tie validation error messages to their specific field via `aria-describedby`, not a single generic banner.
- Never use `tabindex` values greater than `0`; fix tab order by reordering markup instead.
- Use `:where()` for low-specificity reset/default styles so component rules can override them cleanly.
- Use logical properties (`margin-inline-start`) over physical ones when RTL support matters.
- Use `srcset`/`sizes` or `<picture>` to serve appropriately sized images instead of one oversized file scaled by CSS.
- Set explicit `width`/`height`/`aspect-ratio` on media to prevent layout shift while loading.
- Prefer native elements and established ARIA patterns over hand-rolled modals/dropdowns/tabs with unmanaged focus.
- Test text reflow and readability at up to 200% browser zoom, not just default zoom.
