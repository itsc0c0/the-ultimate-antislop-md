# General Design Principles

This part is the foundational, timeless counterpart to the AI-slop-specific design guidance found elsewhere in this document. It does not describe what an AI assistant tends to get wrong; it describes what good visual and interaction design actually requires, independent of who or what produced it. Every rule here predates modern AI tooling and will outlive it — a modular type scale, a real color system, a disciplined spacing grid, fully-stated interactive components, purposeful motion, and accessible, honest content are not stylistic preferences, they are the baseline that separates deliberate design from generic output. Use this part as the standard every design decision — human-made or AI-generated — should be measured against.

## Typography

Typography carries more of a product's perceived quality than almost any other single design decision, because text is present on nearly every screen and read far more than it is looked at. A typographic system that is arbitrary — sizes picked by eye, line-heights left at browser defaults, line lengths that stretch edge to edge — reads as unfinished even when every other visual element is polished. The sub-sections below cover the mechanics of building a typographic system that holds together.

### Establishing a Real Type Scale

- **DO:** Define a modular type scale built from a base size and a fixed ratio (for example a base of 16px and a ratio of 1.25, producing steps like 16 → 20 → 25 → 31 → 39 → 49), and pull every font size in the product from that scale. A ratio-based scale guarantees that every size relates mathematically to every other size, which is what makes a page of mixed headings and body text feel like one coherent system instead of a pile of unrelated numbers.

- **DON'T:** Pick font sizes ad hoc per component — 14px here, 15px there, 22px for this one heading because it "looked right" on one screen. Ungoverned sizes accumulate silently until the type system has dozens of near-duplicate sizes that serve no distinct purpose and make the interface feel inconsistent even though no single instance looks wrong in isolation.

```css
/* BAD: arbitrary, ungoverned sizes scattered through the codebase */
.card-title { font-size: 17px; }
.section-heading { font-size: 23px; }
.page-title { font-size: 27px; }

/* GOOD: every size traces back to a defined 1.25 modular scale */
:root {
  --text-100: 0.8rem;   /* 12.8px — captions, meta */
  --text-200: 1rem;     /* 16px   — body */
  --text-300: 1.25rem;  /* 20px   — subheading */
  --text-400: 1.563rem; /* 25px   — heading */
  --text-500: 1.953rem; /* 31px   — page title */
  --text-600: 2.441rem; /* 39px   — display */
}
```

- **DO:** Choose a ratio that fits the content's density and purpose. Tighter ratios (1.125 minor second, 1.2 minor third) suit information-dense UI where many heading levels must coexist without huge jumps; looser ratios (1.333 perfect fourth, 1.5 perfect fifth) suit editorial and marketing contexts where a small number of dramatic size contrasts is the goal.

- **DON'T:** Apply a dramatic ratio like the golden ratio (1.618) to a dense application UI. In a data-heavy dashboard or admin panel, a golden-ratio scale produces headings that are wildly oversized relative to the surrounding content, wasting vertical space and breaking the visual density the interface needs.

- **DO:** Cap the scale at roughly six to nine steps and give each step a clear, named purpose (caption, body, body-large, heading-4 through heading-1, display). A bounded, purposeful scale is something a team can memorize and reuse; an unbounded one becomes a a la carte menu where every new screen invents "the size that felt right this time."

- **DON'T:** Add a new step to the scale just to make one screen's heading look "a little bigger." If an existing step is close but not exactly what a designer wants, that is a signal to use the existing step, not to fork the scale — a scale that grows without discipline stops being a scale.

- **DO:** Express font sizes in relative units (`rem`) rather than fixed pixels, so the entire scale resizes proportionally when a user changes their browser or OS default text size. This is both a typographic-consistency win and an accessibility requirement — see Accessibility Foundations below.

- **DON'T:** Hardcode `font-size` in `px` throughout a codebase. Pixel values ignore the user's base font-size preference entirely, which means a user who has set a larger default size for low vision gets no benefit from that setting inside your product.

- **DO:** Set a body text base size of at least 16px (1rem) for primary reading content on the web, and no smaller than roughly 11pt (about 14.7px) in native mobile apps for anything beyond incidental labels. Text below this threshold measurably slows reading speed and increases eye strain, especially for older users and anyone on a lower-resolution or lower-brightness display.

- **DON'T:** Shrink body copy to 12–13px to fit more content on screen. This trades a small, permanent layout win for a real, ongoing readability cost paid by every reader, and it is one of the most common ways a design that "looks clean" in a mockup becomes hard to actually use in daily practice.

- **DO:** Treat the type scale as a token set (`font-size-100` … `font-size-900`) documented once and consumed everywhere, in both the design tool and the codebase, so a scale change propagates instead of requiring a hunt-and-replace across every file.

- **DON'T:** Let a scale exist only informally, as "whatever the last few screens used." Without an explicit, named, referenceable scale, new contributors have no way to discover what sizes already exist and will default to typing in a number that feels right — which is exactly how scale drift begins.

### Line-Height and Leading Discipline

- **DO:** Set `line-height` as a unitless multiplier (e.g., `line-height: 1.5`) rather than a fixed length. A unitless value scales correctly with the element's own font size and inherits sanely into children with different sizes; a fixed-length value (`line-height: 24px`) applied high in the DOM produces cramped or excessively loose leading wherever a descendant's font size differs from the size the fixed value was tuned for.

```css
/* BAD: fixed leading breaks the moment a child's font-size differs */
body { line-height: 24px; }
h1 { font-size: 40px; }         /* inherits 24px leading — text overlaps */

/* GOOD: unitless leading scales correctly with any descendant's size */
body { line-height: 1.5; }
h1 { font-size: 2.5rem; }       /* gets 1.5 × its own size automatically */
```

- **DON'T:** Leave `line-height` at the browser or framework default and assume it is fine everywhere. Default leading is usually tuned as a single compromise value and is rarely correct simultaneously for both large display headings and small dense body text.

- **DO:** Use tighter leading for large headings (roughly 1.1–1.3) and looser leading for smaller body text (roughly 1.5–1.65). As text gets larger, the eye needs less vertical gap to distinguish one line from the next; as text gets smaller, more gap is needed to prevent lines from visually fusing together during continuous reading.

- **DON'T:** Apply one flat line-height value — commonly 1.5, tuned for body copy — to a 48px hero heading. The result is headline lines that look disconnected from each other, floating with awkward gaps that make a two- or three-line headline read as several unrelated fragments instead of one statement.

- **DO:** Increase leading as line length increases, and decrease it as line length shortens. A wide column of text needs more vertical breathing room between lines so the eye can find the start of the next line during the return sweep; a narrow column (like a sidebar or a caption) can use tighter leading without becoming hard to track.

- **DON'T:** Use identical tight leading (1.2 or below) for long-form paragraphs that span a wide measure. Tight leading on wide text causes line-to-line "bleed" where the reader's eye loses track of which line it just finished and drifts to the wrong line on the return sweep, forcing re-reading.

- **DO:** Treat leading as part of vertical rhythm — pair the type scale's leading values with the spacing scale so that a paragraph's line-height and a block's margin both resolve to multiples of the same base spacing unit. This keeps a consistent baseline grid across text and non-text elements alike.

- **DON'T:** Let paragraph leading and block spacing evolve independently with unrelated values (e.g., 1.47 line-height next to 13px margins). Numbers that don't share a common base unit make the vertical grid feel accidental rather than designed, even if no individual value looks wrong.

### Measure and Line Length

- **DO:** Constrain body text measure (the number of characters per line) to roughly 45–75 characters, with about 60–66 as a strong default for long-form reading. Lines in this range let the eye's return sweep reliably land on the start of the next line; both much shorter and much longer lines measurably slow reading and increase the error rate of that sweep.

```css
/* GOOD: constrains prose to a comfortable reading measure regardless of viewport width */
.article-body {
  max-width: 65ch;
  margin-inline: auto;
}
```

- **DON'T:** Let paragraphs stretch to the full width of a wide viewport with no `max-width` constraint. On an ultrawide monitor, an unconstrained paragraph can exceed 150–200 characters per line, which forces large, fatiguing eye movements and makes sustained reading noticeably harder even though nothing else about the typography changed.

- **DO:** Use the `ch` unit (or an equivalent rem-based approximation) to cap text container width, since `ch` is defined relative to the font's own character width and therefore keeps the measure roughly constant across different fonts and sizes.

- **DON'T:** Rely on percentage-based widths alone (`width: 70%`) to control measure in a responsive layout. A percentage that produces a good measure at one viewport width can produce an unreadably wide or narrow measure at another, since it has no relationship to the actual character width of the text it contains.

- **DO:** Shorten the measure for secondary or scannable content (captions, list items, card summaries) relative to primary long-form prose. Short, discrete chunks of text are read differently from continuous prose and tolerate — sometimes benefit from — a narrower column.

- **DON'T:** Apply the same wide, prose-tuned measure to UI microcopy, table cells, or form helper text. These are read in short bursts, not scanned line by line, so a `65ch` constraint here often just wastes horizontal space without any readability benefit.

### Font Pairing

- **DO:** Pair typefaces on purposeful contrast — a distinct structural difference the reader can immediately perceive, such as a humanist serif for long-form body copy against a geometric sans for UI chrome and headings, or a display face for large headlines against a neutral workhorse sans for everything functional. The contrast should be legible as intentional, not accidental.

- **DON'T:** Pair two typefaces that are similar-but-not-identical in style, weight, and proportions (two different geometric sans-serifs with slightly different x-heights, for instance). Near-identical pairings read as a mistake — as though the design system couldn't decide on one font — rather than as a deliberate choice, because the eye registers the difference as noise without understanding why it's there.

- **DO:** Limit an interface to two type families at most: one for display/headings (optional — can be the same family as body at a different weight) and one for body/UI text. A tight family budget keeps rendering performance in check, keeps visual harmony intact, and forces every typographic decision to happen within weight, size, and spacing rather than by reaching for a third font.

- **DON'T:** Let four or more typefaces accumulate across a single product because each new feature team or each new marketing page picked its own. This is one of the most common and most visible signs of a fragmented design system — every screen ends up feeling like it belongs to a different product.

- **DO:** Match structural traits — x-height, stroke contrast, overall proportions — when combining two families, even if their historical style differs (serif vs. sans). Fonts with wildly mismatched x-heights look uneven when set at the same nominal point size, because point size doesn't correspond to visually perceived size across different type designs.

- **DON'T:** Combine a high-contrast, thin-stroked display serif with a heavy, low-contrast geometric sans at equal visual weight without adjusting sizes or weights to compensate. Mismatched "typographic personality" — one face whispers, the other shouts — creates unintentional tension instead of a deliberate contrast.

- **DO:** Prefer a single variable font family offering a wide weight and optical-size range when the goal is simplicity and performance. A variable font can supply everything from a light caption weight to a heavy display weight from one file, which reduces network payload and guarantees stylistic harmony across every weight used.

- **DON'T:** Load five or six separate static font-weight files when a single variable font file would cover the same range with less payload and less risk of subtle metric mismatches between weights sourced from different files or foundries.

### Hierarchy Through Weight, Size, and Color

- **DO:** Build hierarchy from at least two of {size, weight, color/contrast} acting together, not from a single lever alone. Two elements that differ only in weight, with identical size and color, produce a hierarchy signal that is easy to miss at a glance and disappears entirely for anyone with low vision or a low-contrast display.

- **DON'T:** Rely on bold alone to indicate that one piece of text outranks another when both are otherwise the same size and color. This is a common, weak pattern — the bold text reads as "emphasized" more than as "more important," and the distinction can vanish completely in a quick scan or a screenshot compressed for messaging.

- **DO:** Reserve heavier weights (600–700 and above) for a genuinely small number of high-priority elements per screen — typically a page title, primary section headings, and perhaps one callout. Scarcity is what makes a heavy weight mean something; if half the screen is bold, none of it reads as more important than the rest.

- **DON'T:** Bold entire paragraphs, sentences, or long phrases for "emphasis." Long runs of bold text are harder to read than regular weight (the letterforms are heavier and closer together), and every additional bold run on a screen dilutes the signal value of every other bold element already there.

- **DO:** Use color deliberately to separate primary from secondary text — for example, a near-black for primary reading text and a mid-gray for metadata or secondary captions — while keeping every color choice within WCAG contrast minimums for its role (see Color, below).

- **DON'T:** Let a color-only hierarchy be the sole distinguishing signal if it wouldn't still read correctly in grayscale or for a color-blind reader. If removing color leaves two text roles visually identical, the hierarchy was never robust — pair the color difference with a size or weight difference too.

### Preventing Orphans and Widows

- **DO:** Prevent orphans (a single short word stranded alone on the last line of a paragraph or heading) and widows (a single line stranded alone at the top of a new column or page) in headlines, pull quotes, and any prominent short-form text. A dangling single word breaks the visual rhythm of an otherwise well-set block and reads as an unpolished, accidental line break.

```css
/* GOOD: modern browsers can balance heading lines and avoid orphans automatically */
h1, h2, h3 {
  text-wrap: balance;
}
p {
  text-wrap: pretty; /* avoids orphans in body copy where supported */
}
```

- **DON'T:** Manually insert a `<br>` tag at a specific point in a heading to "fix" its wrapping. A manual break only looks correct at the one viewport width it was tuned for — resize the window, translate the copy into a longer language, or change the font, and the manually placed break produces an even worse wrap than the one it was meant to fix.

- **DO:** Use `text-wrap: balance` for headings (which evens out line lengths across a multi-line heading) and `text-wrap: pretty` for body paragraphs (which avoids orphans with a small, browser-managed layout cost), falling back gracefully in browsers that don't yet support them since both are additive, non-breaking properties.

- **DON'T:** Leave long, unbalanced headline wraps unaddressed in high-visibility marketing or hero contexts where they are most noticeable — a three-word first line towering over a lonely one-word second line undermines an otherwise carefully designed page.

### Constraining Weights and Sizes for Consistency

- **DO:** Limit the number of actively used font weights in a system to three or four — commonly regular (400), medium (500), semibold (600), and bold (700) — and assign each a specific role (body text, emphasis/labels, subheadings, headings) rather than letting weight be chosen freely per instance.

- **DON'T:** Use every weight a variable font ships with (thin, extralight, light, regular, medium, semibold, bold, extrabold, black) just because they're technically available. A wide, ungoverned weight palette makes hierarchy ambiguous — readers can no longer tell whether a slightly heavier weight means "important" or was simply an arbitrary pick.

- **DO:** Reuse the same handful of sizes and weights everywhere a given role recurs — every card title uses the same size/weight pair, every form label uses the same pair, every error message uses the same pair. Repetition across contexts is what makes a system recognizable and predictable rather than novel every time.

- **DON'T:** Introduce a "just this once" 15px label size for a single component because the existing 14px/16px steps didn't feel quite right. One-off sizes are how a disciplined six-step scale quietly becomes a disorganized eighteen-step scale within a year, with no one able to explain what half the sizes are for.

### Responsive Typography

- **DO:** Scale type fluidly across viewport widths using `clamp()`, so headline and body sizes adjust smoothly between a minimum and maximum instead of jumping abruptly at fixed breakpoints.

```css
/* GOOD: fluid heading size that scales smoothly between mobile and desktop */
h1 {
  font-size: clamp(1.75rem, 4vw + 1rem, 3.5rem);
}
```

- **DON'T:** Ship a single fixed heading size that is either too large for small viewports (causing awkward wraps or horizontal overflow) or too small for large viewports (wasting the impact a display size is meant to create). A size tuned for one screen width is rarely correct at another without adjustment.

- **DO:** Re-check measure, line-height, and weight together at each breakpoint, not just font size in isolation. A heading that looks balanced at desktop width can become cramped or oddly wide once it reflows to two lines on mobile, even if the font-size itself scaled "correctly."

- **DON'T:** Assume a type scale designed and tested only at desktop width will hold up on mobile without dedicated review. Mobile viewports change both the measure and the number of lines a heading wraps to, which changes its perceived hierarchy relative to surrounding content.

### Tabular Figures for Numeric Data

- **DO:** Use tabular (fixed-width) numeral figures for any numbers presented in a column — prices, statistics, table cells, countdowns — via `font-variant-numeric: tabular-nums`, so digits align vertically and don't visually jitter as they change.

```css
/* GOOD: keeps digit columns aligned in tables and live-updating counters */
.price, .stat-value, table td.numeric {
  font-variant-numeric: tabular-nums;
}
```

- **DON'T:** Use a typeface's default proportional numerals in a data table or financial figure column. Proportional digits have varying widths (a "1" is narrower than an "8"), which makes a column of numbers ragged and harder to compare at a glance — exactly the opposite of what a table of numbers is for.

### Letter-Spacing (Tracking)

- **DO:** Apply slightly positive letter-spacing (tracking) to small all-caps labels and slightly negative tracking to very large display headlines. Small caps text loses legibility as letters crowd together at small sizes, so a touch of extra spacing restores it; very large headline sizes have proportionally larger natural gaps between letters, so tightening slightly keeps the headline feeling like one solid word-group rather than loosely scattered letters.

```css
/* GOOD: opens up small caps labels, tightens an oversized display headline */
.eyebrow-label { text-transform: uppercase; letter-spacing: 0.08em; font-size: 0.75rem; }
.hero-display { font-size: 5rem; letter-spacing: -0.02em; }
```

- **DON'T:** Apply the same letter-spacing value uniformly across every size in the type scale. A tracking value tuned for a 12px caption looks needlessly loose at 48px, and a tracking value tuned for a 48px display headline looks uncomfortably tight at 12px — tracking needs to be tuned per size, not applied as one global constant.

- **DON'T:** Add loose letter-spacing to body paragraph text "for style." Extra tracking in continuous reading text measurably slows reading speed by weakening the visual word-shapes readers rely on for fast recognition; reserve tracking adjustments for short labels, headlines, and all-caps text.

### Text Alignment and Justification

- **DO:** Left-align (or right-align, for RTL languages) body paragraphs and long-form content by default. Ragged-edge alignment gives natural, even word spacing and a consistent line-start position the eye can rely on for the return sweep, both of which support faster, more comfortable reading than the alternatives.

- **DON'T:** Center-align paragraphs of body text beyond a line or two. Centered paragraphs create an irregular left edge that the eye cannot use as a reliable anchor for the return sweep, forcing extra work to relocate the start of each new line — acceptable for a short pull-quote or a two-line caption, damaging for anything longer.

- **DO:** Reserve full justification (`text-align: justify`) for contexts with hyphenation enabled and a wide-enough measure to avoid noticeable spacing distortion — traditionally print, and only cautiously on the web.

- **DON'T:** Justify body text on the web without hyphenation. Justified text without hyphenation produces "rivers" of uneven word spacing, especially in narrow columns, since the browser can only stretch or compress space between whole words to fill the line — the result looks worse than the ragged edge it was meant to fix.

### Typographic Details and Punctuation

- **DO:** Use real typographic characters — curly quotation marks (" " ' '), an em dash (—) or spaced en dash (–) for parenthetical breaks, an ellipsis character (…) instead of three periods, and a true multiplication sign (×) rather than a lowercase "x" where multiplication is meant. These details are what separates typeset-feeling text from a plain-text dump, and most content and CMS pipelines can auto-substitute them.

```text
BAD:  "Don't" use "straight quotes" -- or three dots...
GOOD: "Don't" use "curly quotes" — or a real ellipsis…
```

- **DON'T:** Leave straight/typewriter quotes (`"`, `'`) and double-hyphen dashes (`--`) unconverted in rendered marketing or editorial copy. These are typewriter-era substitutes for characters that don't exist on that limited keyboard — modern typesetting has no such constraint, and leaving them in reads as a plain-text export that was never finished.

- **DO:** Use non-breaking spaces between a number and its unit, and before certain punctuation in languages that require it, so a value like "42 kg" or "$5" never breaks across two lines with the number stranded from its unit or currency symbol.

- **DON'T:** Let numeric values and their units be free to wrap independently. A price or measurement split across a line break ("42<br>kg") is confusing at a glance and looks like a layout bug even though it's purely a text-wrapping oversight.

### Font Loading and Perceived Performance

- **DO:** Set `font-display: swap` (or an equivalent strategy) on custom web font declarations so text renders immediately in a fallback font and swaps to the custom font once it loads, rather than leaving text invisible while the font downloads.

```css
/* GOOD: text is visible immediately in a fallback, swaps in once the custom font loads */
@font-face {
  font-family: "Brand Sans";
  src: url("/fonts/brand-sans.woff2") format("woff2");
  font-display: swap;
}
```

- **DON'T:** Leave web fonts at the browser default loading behavior, which in several browsers hides text entirely for up to a few seconds while the font downloads (a "flash of invisible text"). A slow or failed font load shouldn't be able to make a page's text completely unreadable in the meantime.

- **DO:** Choose a fallback font stack whose metrics (x-height, average character width) reasonably approximate the custom web font, so the swap from fallback to final font causes minimal layout shift.

- **DON'T:** Pair a custom font with a wildly different-proportioned fallback (a narrow fallback standing in for a wide display face). A large metric mismatch causes a visible, sometimes jarring reflow the moment the real font finishes loading — noticeable most on text-heavy pages with a slower connection.

- **DO:** Limit the number of font weights and styles actually loaded to what's genuinely used, since each additional weight/style is a separate file the browser must fetch, directly adding to page weight and font-swap delay.

- **DON'T:** Load an entire font family's full weight range (all nine weights, italics included) when the interface only actually uses three of them. Unused font weight files are pure wasted payload with zero design benefit.

### Readability: Long-Form vs. UI Text

- **DO:** Apply different typographic rules to long-form reading content (articles, documentation, blog posts) than to interface text (labels, buttons, table cells) — long-form content benefits from a wider, more book-like measure and more generous leading; interface text benefits from a tighter, more compact treatment suited to scanning rather than sustained reading.

- **DON'T:** Apply one uniform typographic treatment across both contexts. A measure and leading tuned for comfortable sustained reading wastes space and looks oddly loose in a dense settings table; a measure and leading tuned for a compact UI feels cramped and fatiguing across a multi-paragraph article.

- **DO:** Increase paragraph spacing (not just line-height) in long-form content to give the eye clear rest points between ideas, distinct from the tighter spacing appropriate between elements in a UI form or table.

- **DON'T:** Rely on line-height alone to separate paragraphs in long-form content, with no additional space between them. Paragraphs that are only as far apart as their own internal line-height are harder to visually parse as distinct units, especially in longer articles.

### Monospace and Code Typography

- **DO:** Use a genuine monospace typeface for code, identifiers, technical values, and any content where consistent character width matters (aligned CLI output, diffs, tabular technical data), and choose one with clearly disambiguated characters (a slashed zero, a distinct lowercase `l` versus uppercase `I` versus digit `1`).

```text
BAD:  using a proportional font for code — "l1I" and "0O" can be
      genuinely ambiguous to the reader
GOOD: a monospace font designed for code, with disambiguated glyphs —
      "l1I" and "0O" are visually distinct at a glance
```

- **DON'T:** Render code or technical identifiers in a proportional body font. Beyond looking visually inconsistent with the rest of a code-adjacent interface, a proportional font makes it genuinely harder to spot subtle but critical differences — a missing character, a similar-looking identifier — that a monospace font's fixed-width, disambiguated glyphs make much easier to catch.

- **DO:** Keep monospace font sizing and line-height tuned separately from the surrounding prose scale, since monospace typefaces typically have different proportions (often a larger apparent x-height at the same nominal size) than the paired body font.

- **DON'T:** Apply the exact same font-size token used for body prose directly to inline or block monospace text without checking how it actually renders. A monospace face at the "same" nominal size as the body font frequently reads as visually larger or smaller than intended.

### Print vs. Screen Typography

- **DO:** Adjust type choices for a print stylesheet or exported document context — increasing body size slightly if needed for print legibility, ensuring sufficient contrast survives non-color (black-and-white) printing, and avoiding UI-only conventions like link-colored text that carries no meaning once printed.

- **DON'T:** Ship a printed or exported (PDF) version of a page with unmodified screen typography and no dedicated print styles. Content sized and colored for a backlit screen frequently prints poorly — links that were only distinguished by color lose that distinction entirely in black-and-white output, for instance.

- **DO:** Hide purely interactive, screen-only chrome (navigation, buttons, hover-only elements) in print output via a print stylesheet, so a printed page shows only the actual content, not UI controls that have no function on paper.

- **DON'T:** Let an unstyled print output include navigation bars, buttons, and interactive controls that serve no purpose once printed, wasting paper and cluttering the printed page with irrelevant UI chrome.

### Text Truncation and Overflow Handling

- **DO:** Truncate overflowing text deliberately and predictably (an ellipsis at a sensible boundary, a defined max line count with a "show more" control) and make the full text available on demand — via a tooltip, an expand action, or a detail view — whenever it might genuinely matter to the user.

```css
/* GOOD: predictable single-line truncation, with the full value available via title/tooltip */
.truncate {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
```

- **DON'T:** Let text overflow its container unpredictably — wrapping into neighboring elements, getting silently clipped with no ellipsis or indication that content is missing, or breaking a layout's alignment. Silent, unindicated truncation is worse than visible truncation, because the user doesn't even know there's more to see.

- **DO:** Choose a truncation strategy proportional to how often the full content actually matters — a data table cell might reasonably truncate with a hover tooltip for the full value, while a critical warning message should probably never be truncated at all.

- **DON'T:** Apply the same aggressive truncation rule uniformly regardless of content importance, truncating a critical warning or error message down to an ellipsis just as readily as a decorative subtitle. Truncation rules should reflect how essential the truncated content is to the user's task.

### Emphasis Conventions: Italics, Bold, and Underline

- **DO:** Reserve italics for their conventional uses — titles of works, foreign-language terms, light editorial emphasis on a single word or short phrase — and use them sparingly, since italic letterforms are measurably harder to read at length than upright text.

- **DON'T:** Italicize entire sentences or paragraphs for emphasis. Extended italic text slows reading and, combined with its conventional association with minor/incidental content, can undercut rather than reinforce the importance the emphasis was meant to convey.

- **DO:** Reserve underline styling specifically for hyperlinks in body text (matching a long-standing, near-universal web convention), and use a different treatment (bold, color, background highlight) for non-link emphasis.

- **DON'T:** Underline non-link text for emphasis, especially inline within a paragraph that also contains real links. Underlined non-link text creates a false affordance exactly as described under Components — a user will reasonably expect underlined text to be clickable, and be confused or frustrated when it isn't.

### Text Case Conventions

- **DO:** Choose one consistent capitalization convention per text role — sentence case for body copy and most UI labels, a deliberate title case only where a system-wide style guide calls for it (some brands use title case for buttons or headings specifically) — and apply that convention identically across the whole product.

```text
BAD (inconsistent case across equivalent elements):
  Button: "Save Changes"      Button: "delete account"      Button: "SUBMIT"

GOOD (one consistent convention applied everywhere):
  Button: "Save changes"      Button: "Delete account"      Button: "Submit"
```

- **DON'T:** Mix capitalization styles across equivalent UI elements — one button in title case, another in sentence case, a third in all caps — with no consistent rule governing the choice. Inconsistent casing is a small detail in isolation but, like inconsistent spacing or color, accumulates into a product that reads as uncoordinated.

- **DO:** Reserve all-caps styling for short, clearly-bounded labels (a small eyebrow label, a badge) where the reduced legibility of all-caps at length isn't a concern, and pair it with slightly increased letter-spacing (see Letter-Spacing, above) to compensate.

- **DON'T:** Set long runs of text in all caps. All-caps text is measurably harder to read at length than mixed-case text, since it removes the varied word-shape silhouettes that make mixed-case text fast to recognize — reserve it for short labels, never for sentences or paragraphs.

### Font Fallback Stacks and System Fonts

- **DO:** Define a sensible fallback font stack for every custom typeface, ending in a generic family (`sans-serif`, `serif`, `monospace`), so text remains readable and reasonably well-proportioned even if the custom font fails to load for any reason.

```css
/* GOOD: a sensible fallback chain, not just the one custom font */
body {
  font-family: "Brand Sans", -apple-system, "Segoe UI", Roboto, Arial, sans-serif;
}
```

- **DON'T:** Specify only a single custom font with no fallback stack. If that font fails to load — a network issue, a blocked CDN, a typo in the font name — text can fall back to the browser's default serif font or, worse, render with missing glyphs, with no graceful degradation designed in.

- **DO:** Consider using the operating system's native font stack (`-apple-system`, `Segoe UI`, Roboto, etc.) deliberately for interfaces that want to feel maximally native and performant, trading a distinctive brand typeface for zero font-loading cost and automatic OS-level consistency.

- **DON'T:** Assume a custom web font is always the right choice without weighing its real cost — additional network requests, potential flash-of-unstyled-text handling, and font-licensing overhead — against the brand differentiation it provides. For some products, particularly performance-sensitive ones, the system font stack is a legitimate, deliberate choice rather than a compromise.

### Text Selection Styling

- **DO:** Style the browser's native text-selection highlight (`::selection`) to use brand-appropriate colors that still maintain clear contrast between selected text and its highlight background, rather than leaving it at the often visually jarring or brand-mismatched browser default.

```css
/* GOOD: a branded selection color that still keeps text legible */
::selection {
  background-color: var(--color-primary-200);
  color: var(--color-text-primary);
}
```

- **DON'T:** Override text-selection styling in a way that reduces contrast between the selected text and its highlight to the point of being hard to read. A selection highlight exists to give clear feedback about exactly what's selected — a low-contrast override defeats that purpose for a purely cosmetic gain.

### Multilingual Font Support

- **DO:** Verify that the chosen typefaces actually support every script the product needs to render (Latin, Cyrillic, Greek, CJK, Arabic, Hebrew, Devanagari, and others as relevant), and provide an appropriate fallback font for any script the primary typeface doesn't cover, rather than letting unsupported characters render as missing-glyph boxes.

- **DON'T:** Assume a Latin-designed display or brand typeface automatically covers every script a global product needs. A typeface with no CJK or Arabic glyphs, used without a proper fallback, renders unsupported text as literal empty boxes ("tofu") — a jarring, obviously broken experience for users in that locale.

- **DO:** Adjust line-height and vertical spacing per script where needed, since CJK characters and some other scripts have different vertical metrics than Latin text and can need more generous line-height to avoid feeling cramped, even at a font-size that looks correct for Latin script.

- **DON'T:** Apply Latin-tuned line-height and letter-spacing values unchanged to CJK, Arabic, or other non-Latin scripts. Typographic rhythm tuned for one script's proportions frequently doesn't transfer directly to a script with meaningfully different character shapes, density, or vertical metrics.

## Color

### Neutral Palette Undertone Consistency

- **DO:** Choose a single, consistent undertone (warm/beige-leaning, cool/blue-leaning, or a true neutral) for the entire gray scale, and keep every step of that scale on the same undertone, since mixing undertones within one supposedly neutral scale produces steps that subtly clash against each other.

```text
BAD:  gray-200 leans slightly blue, gray-400 leans slightly warm/beige
      → the two grays look like they don't belong to the same palette

GOOD: every gray step shares the same subtle blue (or warm, or true-neutral)
      undertone, so the whole scale reads as one coherent, deliberate ramp
```

- **DON'T:** Let neutral/gray tokens get added to the scale independently over time by different contributors, each nudging the hue slightly differently. Undertone drift within a supposedly neutral scale is subtle enough to be hard to spot in isolation but becomes visible the moment two steps sit next to each other, which happens constantly in real layouts (borders next to backgrounds, text next to surfaces).

- **DO:** Match the neutral scale's undertone deliberately to the brand's primary color family — a cool-leaning brand palette generally pairs better with cool-leaning neutrals, and a warm-leaning brand palette with warm-leaning neutrals — so the whole system feels unified rather than having a brand palette and a neutral palette that were designed independently of each other.

- **DON'T:** Default to a generic, off-the-shelf gray scale with no consideration of how its undertone relates to the brand's actual color palette. A mismatched undertone between brand colors and neutrals is a common, subtle reason a color system can look technically correct (good contrast, consistent steps) while still feeling slightly "off" as a whole.

### Surface Elevation Colors in Light Mode

- **DO:** Assign distinct, deliberately-chosen surface colors to each elevation level even in light mode (not just relying on shadows) — a base background, a slightly different card/panel background, a distinct modal/overlay background — so elevation is legible even in contexts (print, a screenshot, a low-shadow-visibility environment) where shadow alone might not read clearly.

- **DON'T:** Rely purely on `box-shadow` to communicate elevation in light mode with every surface using the identical background color. In many real viewing conditions (a low-quality screenshot, a printed page, certain color-vision differences), a subtle shadow can be far less perceptible than a genuine background-color shift, making elevation ambiguous.

- **DO:** Keep the lightness steps between adjacent elevation surfaces in light mode subtle but perceptible — small enough not to look like separate, disconnected color blocks, large enough to be genuinely distinguishable on close inspection.

- **DON'T:** Make elevation-surface color differences either imperceptibly small (providing no real signal) or jarringly large (making adjacent surfaces look like unrelated, disconnected sections rather than a layered, coherent whole).

Color is the design decision most likely to be treated as purely aesthetic and least likely to actually be purely aesthetic. A color system determines whether users with low vision or color blindness can use the product at all, whether error and success states are distinguishable under bad lighting or a low-quality screen, and whether a dark theme is a genuinely designed experience or a broken inversion. Building color as a governed system — scales, semantic roles, and verified contrast — is what turns "picking colors" into a discipline with testable outcomes.

### Building a Semantic Color System

- **DO:** Build color as full scales per role — typically primary, secondary, accent, neutral/gray, success, warning, danger/error, and info — with each scale spanning a consistent range of steps (commonly 9–11 steps from a very light tint to a very dark shade, e.g., 50 through 900). A scale, not a single hex value, is what lets one role support a background tint, a border, body text, and a hover state without anyone picking new one-off colors for each.

```css
/* GOOD: a governed scale per role, not one hex value per role */
:root {
  --color-primary-50:  #eef4ff;
  --color-primary-100: #dbe6fe;
  --color-primary-500: #4f6df5;  /* base brand color */
  --color-primary-600: #3c56d6;  /* hover/active */
  --color-primary-900: #1e2a63;  /* high-contrast text on light bg */

  --color-danger-50:  #fef2f2;
  --color-danger-500: #dc2626;
  --color-danger-700: #991b1b;
}
```

- **DON'T:** Hardcode individual hex or RGB values directly inside component styles. A value like `color: #4f6df5` scattered across dozens of files has no name, no documented role, and no way to be updated consistently — a rebrand or a contrast fix becomes a find-and-replace exercise instead of a single token change.

- **DO:** Generate each scale with a consistent, systematic method for stepping lightness (or perceptual lightness, when using a perceptually uniform space like OKLCH) rather than eyeballing each step. A systematic scale keeps the visual "distance" between adjacent steps roughly even, so `primary-400` to `primary-500` feels like the same size jump as `primary-500` to `primary-600` everywhere in the palette.

- **DON'T:** Mix colors picked from different tools, different color pickers, or different unrelated inspirations each time a new UI need arises ("we needed a slightly different blue for this one chart, so I picked one that looked nice"). Ungoverned color accretion is the color equivalent of an ungoverned type scale — every new addition looks fine alone and the whole palette becomes incoherent together.

- **DO:** Separate raw palette tokens (hue-named, e.g. `blue-500`, `red-500`) from semantic alias tokens (role-named, e.g. `color-primary`, `color-danger`) and have components reference only the semantic layer. This two-tier structure is what makes a rebrand — swapping the brand hue from blue to purple — a one-line change at the alias layer instead of a search-and-replace across every component.

```css
/* Tier 1: raw palette (rarely referenced directly by components) */
--blue-500: #4f6df5;
--red-500:  #dc2626;

/* Tier 2: semantic aliases (what components actually use) */
--color-primary:  var(--blue-500);
--color-danger:   var(--red-500);
```

- **DON'T:** Name a token after its raw appearance and then reuse that name for an unrelated semantic role — for instance calling the brand action color `color-blue` and then using `color-blue` for both the primary button and, later, an unrelated info banner, so that a future rebrand to a green primary color leaves a token literally named "blue" holding a green value everywhere.

- **DO:** Define a neutral/gray scale with enough steps (typically 9–11) to cover every surface need — page background, card background, border, disabled state, placeholder text, secondary text, primary text — without reaching for arbitrary opacity hacks on a single gray.

- **DON'T:** Simulate the neutral scale by applying different opacity values to one black or white color throughout the interface. Opacity-based grays composite differently depending on what's behind them, which means the "same" gray can look different across different backgrounds — a real, fixed neutral scale avoids that inconsistency entirely.

### WCAG Contrast Ratios

- **DO:** Ensure body text maintains a contrast ratio of at least 4.5:1 against its background, per WCAG 2.1 Level AA, for every text/background pairing shipped in the product. This is the threshold below which a meaningful percentage of users with low vision or moderate visual impairment cannot comfortably read the text at all.

- **DO:** Ensure large text (roughly 24px/18pt regular weight or 19px/14pt bold and above) and non-text UI components — icons, input borders, focus indicators, chart elements that carry meaning — meet a lower but still mandatory minimum of 3:1 contrast against their adjacent background.

```text
Example pairing check:
  #767676 text on #FFFFFF background → 4.54:1  → passes AA for body text
  #999999 text on #FFFFFF background → 2.85:1  → FAILS AA for body text
  #999999 text on #FFFFFF background → passes only for large text / UI components (≥3:1)
```

- **DON'T:** Ship light-gray-on-white or low-contrast color pairings because they look "softer" or more "modern." A stylistic preference for low contrast has a direct, measurable accessibility cost — text that reads as "elegant" on a calibrated design monitor in good lighting can be functionally unreadable on a dim phone screen outdoors.

- **DO:** Check every new color pairing against an actual contrast tool — the WebAIM Contrast Checker, browser DevTools' built-in contrast inspector, a plugin like Stark, or an automated audit like axe — before it ships, and treat a failing ratio as a defect, not a style note to revisit later.

- **DON'T:** Rely on "it looks readable to me" as the check. Human perception of contrast is unreliable and highly dependent on monitor calibration, ambient lighting, and the checker's own visual acuity — designers who stare at bright, well-calibrated monitors all day consistently underestimate how much contrast a pairing needs for the general population.

- **DO:** Verify contrast for every state a text or component can be in, not only its default appearance — hover, focus, disabled, placeholder, and any dynamically-applied overlay or scrim. A pairing that passes in its resting state can silently fail once a semi-transparent hover overlay or a disabled-state opacity reduction is applied on top of it.

- **DON'T:** Let disabled or placeholder text fall to an arbitrarily low contrast just because "it's disabled, so it doesn't matter." WCAG does relax strict text-contrast requirements for genuinely inert disabled controls, but placeholder text that conveys real instructional content (a format hint, an example value) still needs to be legible enough to actually read — invisible placeholder text defeats its own purpose.

- **DO:** Test contrast against both the light-theme and dark-theme background explicitly, treating them as two separate checks rather than one. A specific color's contrast ratio changes completely depending on what it sits against, so a value validated only in light mode carries no guarantee about its dark-mode performance.

- **DON'T:** Assume a single semantic token (e.g., `color-danger`) "just works" across both themes without measuring each theme's actual rendered contrast. It is common for a saturated red tuned to pass AA on white to fail badly against a dark navy or near-black background, or vice versa — each theme needs its own contrast-checked value behind the same semantic name.

### Designing for Color Blindness

- **DO:** Pair every color-coded meaning with at least one non-color cue — an icon, a text label, a pattern, a shape, or a position — so that the meaning survives even for a reader who cannot distinguish the colors involved. Roughly 1 in 12 men and 1 in 200 women have some form of color vision deficiency, most commonly red-green; color alone reliably fails to communicate to that population.

```text
BAD:  a status dot that is only red or only green, with no other differentiator
GOOD: a status dot that is red with an "x" icon and the word "Failed",
      or green with a checkmark icon and the word "Passed"
```

- **DON'T:** Use only red and green to distinguish opposite states — error vs. success, off vs. on, decrease vs. increase — with no accompanying icon, label, or shape difference. This is the single most common color-blindness failure in interface design, and it is also one of the cheapest to fix by adding an icon or label that was probably needed for clarity anyway.

- **DO:** Test finished palettes with a color-blindness simulator (Sim Daltonism, Coblis, or a browser DevTools vision-deficiency emulation mode) covering at minimum protanopia, deuteranopia, and (less commonly but still worth checking) tritanopia, as part of routine design review, not as a one-time audit.

- **DON'T:** Rely on a color legend as the sole indicator of meaning in a chart or diagram ("red = at risk, yellow = warning, green = on track" printed once in a corner). A reader who cannot distinguish red from green has to constantly cross-reference the legend for every single data point, which defeats the purpose of a chart being scannable at a glance.

- **DO:** Choose a categorical palette for charts and data visualization that remains distinguishable under common color-vision deficiencies — blue/orange pairings, for instance, tend to survive red-green deficiencies far better than red/green pairings do — and vary lightness or pattern in addition to hue once a chart needs more than four or five categories.

- **DON'T:** Rely purely on hue to differentiate more than a handful of categories in a chart. Beyond about four to six hues, even people with typical color vision struggle to reliably map color back to category from memory — combine hue with position, direct labeling, or texture/pattern once the category count grows.

### Light/Dark Theme Parity

- **DO:** Build a genuinely distinct, separately contrast-checked palette for dark mode, treating it as its own design deliverable rather than a mechanical transformation of the light palette. A well-executed dark theme requires its own decisions about which surface sits "above" which, how saturated colors should be desaturated to sit comfortably on a dark background, and where elevation is communicated (since shadows read poorly on dark backgrounds).

```css
/* GOOD: two deliberately designed palettes, not a mechanical inversion of one */
:root {
  --surface-bg: #ffffff;
  --surface-elevated: #f7f8fa;
  --text-primary: #1a1a1e;
  --color-danger: #b3261e;   /* tuned to pass AA on white */
}
:root[data-theme="dark"] {
  --surface-bg: #121214;
  --surface-elevated: #1e1e22;
  --text-primary: #f2f2f4;
  --color-danger: #ff6b64;   /* re-tuned, not inverted, to pass AA on dark */
}
```

- **DON'T:** Literally invert lightness values (`100% - L`) or apply a CSS `filter: invert()` across the whole page to produce a "dark mode." Naive inversion turns saturated mid-tone brand colors into garish, oversaturated, or muddy results, and it inverts photographic imagery and icons into unusable negatives — it is a shortcut that reliably produces a worse dark theme than a deliberately designed one.

- **DO:** Reduce saturation and carefully calibrate perceived lightness for surface and accent colors in dark mode. A saturated color that reads as vivid and pleasant on white can appear to visually vibrate, glow, or bloom against a dark background at the same saturation — dark-mode accent colors typically need somewhat lower saturation and higher lightness than their light-mode counterparts to feel equivalent.

- **DON'T:** Pair a pure black (`#000000`) background with pure white (`#FFFFFF`) text as "true dark mode." Maximum contrast at this extreme causes visible halation and eye strain for many readers, especially on OLED screens at night — prefer a near-black surface (commonly in the `#111`–`#1c1c1e` range) paired with an off-white/near-white text color instead.

- **DO:** Re-verify contrast for every semantic color independently in dark mode, since a color that passes on a white background carries no guarantee about its performance on a dark surface — success, warning, and danger colors in particular often need distinct dark-mode values to remain both legible and to preserve the same relative "loudness" the light theme intended.

- **DON'T:** Assume the same red used for a light-mode error state will automatically pass contrast requirements once the background flips to a dark navy or near-black — verify it, and adjust the token's dark-mode value if it doesn't, rather than shipping an unchecked assumption.

- **DO:** Preserve the same relative visual hierarchy and semantic meaning across both themes — whatever reads as "the primary action" in light mode should still read as clearly primary in dark mode, even if the exact hex values differ. Parity is about equivalent experience, not identical numbers.

- **DON'T:** Let a dark theme become a second-class citizen where hierarchy, spacing, or component states subtly differ from the light theme because it received less design attention. Users who prefer or need dark mode (light sensitivity, low-light environments, battery savings on OLED) deserve the same quality of experience as users on the default theme.

### The 60-30-10 Color Balance Principle

- **DO:** Apply the 60-30-10 rule as a loose compositional guide: roughly 60% of a screen's visible color should come from a dominant neutral (backgrounds, large surfaces), roughly 30% from a secondary supporting color, and roughly 10% from a high-impact accent reserved for the most important elements. This ratio is what keeps an accent color feeling special rather than ambient.

```text
BAD:  brand-blue used as page background, card background, AND button color
      → nothing stands out as "the action to take"

GOOD: 60% neutral background/surfaces, 30% secondary/supporting color
      (borders, secondary buttons, muted sections), 10% saturated accent
      reserved for the primary CTA and key highlights
```

- **DON'T:** Use the brand's most saturated accent color as the background of every panel, card, or section "for brand consistency." An accent color that appears everywhere stops functioning as a signal — its entire job is to draw the eye by being rare, and it cannot do that job while covering 60% of the screen.

- **DO:** Reserve the single most saturated, highest-contrast color in the palette for the one primary action per screen — the button or link the design most wants the user to notice and choose.

- **DON'T:** Give four or more competing saturated colors equal visual weight on one screen. When everything is loud, nothing reads as primary, and the user is left to guess which of several equally emphasized elements they're actually supposed to act on.

### Color Naming, Documentation, and Governance

- **DO:** Document what each semantic color token means and when to use it — not just its value — so a token like `color-warning` clearly states it is for "recoverable issues that need attention but don't block the user," distinct from `color-danger` for "destructive or blocking failures." Ambiguous role boundaries lead different teams to apply the same token inconsistently.

- **DON'T:** Leave semantic token boundaries to tribal knowledge or guesswork. If nobody can articulate the difference between `color-info` and `color-accent` without opening the source file, the distinction isn't actually serving its purpose and will be applied inconsistently across the product.

- **DO:** Maintain a single, versioned source of truth for the color palette (a design-tokens file, a Style Dictionary config, or an equivalent) that both design tools and code consume, so a color update happens once and propagates everywhere rather than being manually re-entered in two or more disconnected places.

- **DON'T:** Let a Figma color style library and a codebase's actual CSS/token values drift apart over time, with each edited independently by whoever touched it last. Divergence here is one of the most common causes of "the design and the shipped product don't quite match" bug reports.

- **DO:** Require a lightweight review step before a new color is added to the system — even a new shade within an existing scale — so scale integrity and contrast compliance are checked before, not after, the color starts appearing in shipped screens.

- **DON'T:** Let any contributor introduce a new one-off color value directly in component code with no review. Uncontrolled palette growth is exactly how a system that started with eight clean, purposeful colors ends up with forty near-duplicates that nobody can tell apart or justify keeping.

### Gradients, Overlays, and Effects

- **DO:** Use gradients and overlays purposefully — to establish depth, to ensure text remains legible over a photographic background, or as a deliberate brand signature — and keep the gradient's start/end contrast checked against whatever content sits on top of it at every point along the gradient, not just at its lightest or darkest end.

- **DON'T:** Layer a decorative gradient behind body text without checking contrast across the gradient's full range. Text that passes contrast where the gradient is darkest can fail where it's lightest, producing a section of a paragraph that becomes unreadable exactly where the background happens to be brightest.

- **DO:** Keep a single, documented gradient direction, stop count, and hue relationship (e.g., "always a 135° gradient between two adjacent steps of the primary scale") if gradients are part of the visual language, so every gradient in the product reads as the same system rather than a new invention each time.

- **DON'T:** Let each designer or screen invent its own gradient angle, color pairing, and opacity. Inconsistent gradients are one of the fastest ways to make an interface feel like it was assembled from unrelated design files rather than built as one product.

### Color Psychology and Cultural Context

- **DO:** Research the cultural associations of a color choice for any product with a genuinely international audience, since color meaning is not universal — red signals danger or stop in much of the West but signals luck and celebration in Chinese culture; white signals purity in Western weddings but is associated with mourning in parts of East Asia.

- **DON'T:** Assume a color's connotation in the design team's own cultural context applies globally. A status or warning color scheme designed with only one culture's associations in mind can send an unintended, even confusing signal to users elsewhere.

- **DO:** Treat color psychology as a soft, supporting signal — calming blues for finance/trust contexts, energetic warm tones for urgency or excitement — while still prioritizing the functional requirements (contrast, semantic consistency, accessibility) covered elsewhere in this section, since a color's felt "personality" matters less than whether it reliably functions.

- **DON'T:** Choose a color primarily for its psychological association while ignoring whether it meets contrast requirements or fits coherently into the existing semantic scale. A color that "feels trustworthy" but fails accessibility standards, or clashes with the established palette, isn't a net improvement.

### Color in Data Visualization

- **DO:** Choose the palette type to match the data's structure — a sequential palette (one hue, increasing lightness/saturation) for ordered data with a clear low-to-high range, a diverging palette (two hues meeting at a neutral midpoint) for data with a meaningful zero or midpoint, and a categorical palette (distinct hues, no implied order) for unordered categories.

```text
Sequential (ordered, one direction):  light blue → dark blue
Diverging (meaningful midpoint):      red ← white → blue
Categorical (unordered groups):       blue, orange, green, purple, ...
```

- **DON'T:** Use a categorical (unordered, high-contrast hue) palette for genuinely ordered/sequential data, or a sequential single-hue palette for unrelated categories. Mismatching palette type to data structure actively misleads a reader about whether a relationship or order exists in the data.

- **DO:** Cap categorical chart palettes at roughly 6-8 clearly distinguishable colors; beyond that, most readers can no longer reliably map color back to category from a legend, regardless of how distinct the colors are in isolation.

- **DON'T:** Assign a unique color to every one of fifteen-plus categories in a single chart and expect a reader to track them all through the legend. Consider grouping less significant categories into an "other" bucket, or splitting the data into multiple smaller charts, once the category count exceeds what a palette can reasonably differentiate.

- **DO:** Keep color mappings consistent for the same category or value across every chart within one report or dashboard — if "Region: West" is blue in one chart, it should be blue in every other chart in the same context, not reassigned a new color each time.

- **DON'T:** Let a charting library or ad hoc color assignment reassign colors to categories independently per chart. Inconsistent category-to-color mapping across a multi-chart dashboard forces the reader to re-learn the legend for every single chart instead of building one mental map that holds throughout.

### Testing Contrast Programmatically

- **DO:** Add automated contrast checks to the design-token pipeline or CI process, so any change to a color token that would drop a documented text/background pairing below its required WCAG ratio is caught automatically before merge, rather than relying purely on manual spot checks.

```text
GOOD CI check: a script verifies every declared semantic text/background
pairing (e.g., color-text-primary on color-surface-bg) still meets
its required contrast ratio whenever either token's value changes.
```

- **DON'T:** Rely entirely on a one-time manual contrast audit performed when colors were first chosen, with no ongoing check preventing a later, unrelated token update from silently breaking a previously-passing pairing.

- **DO:** Document, alongside each semantic color token, which specific pairings it's been validated against (e.g., "`color-danger-500` is contrast-checked against `color-surface-bg` and `color-surface-elevated`") so anyone reusing the token in a new context knows whether it's already been validated there or needs a fresh check.

- **DON'T:** Assume a token that passes contrast in its originally-designed context automatically passes in every other context it might get reused in. A token's contrast validity is a property of a specific pairing, not an intrinsic, context-free property of the color itself.

### Brand Color vs. Functional/System Color

- **DO:** Distinguish clearly between brand colors (used for identity — logo, marketing, primary visual signature) and functional/system colors (used for meaning — success, warning, danger, informational states), keeping them as separate token families even when a brand color and a functional color happen to be visually similar.

- **DON'T:** Reuse the exact brand accent color to also mean "success" or "primary action" with no distinction, such that a rebrand which changes the brand color accidentally also changes what "success" looks like everywhere, or such that the brand color's presence gets misread as a specific semantic status by users who've learned to associate it with meaning elsewhere in the product.

- **DO:** Allow functional colors (danger, warning, success) to remain stable and conventionally recognizable (red-family for danger, green-family for success, in cultures where that convention is well established) even across a brand refresh, since users' learned associations with these colors are a genuine usability asset worth preserving independently of brand identity changes.

- **DON'T:** Let a brand refresh casually reassign the hue used for critical functional meanings (making "danger" a shade of purple because it now matches the new brand palette, for instance) without deliberately considering the usability cost of breaking a well-established, widely-learned color convention.

### Supporting Forced-Colors and High-Contrast Modes

- **DO:** Test the interface under the operating system's forced-colors/high-contrast mode (Windows High Contrast Mode and the `forced-colors` CSS media feature being the most common), where the OS overrides most author-defined colors with a small, user-chosen palette, and ensure critical information (borders, focus indicators, icons) still survives when background colors and box-shadows are stripped away.

```css
/* GOOD: ensures a focus ring and border remain visible under forced-colors mode */
@media (forced-colors: active) {
  .btn { border: 1px solid ButtonText; }
  :focus-visible { outline: 2px solid Highlight; }
}
```

- **DON'T:** Rely purely on background-color or box-shadow alone to convey a component's boundary or state, with no border or outline as a fallback. Forced-colors mode frequently removes background-color and box-shadow styling entirely, which can make a component that depended solely on those properties become invisible or lose its perceivable boundary under that mode.

- **DO:** Use real, distinguishable system color keywords (`ButtonText`, `Highlight`, `Canvas`) or ensure sufficient structural (non-color) cues remain under forced-colors mode, so the interface stays usable for the users — often people with specific visual impairments — who rely on this OS-level override.

- **DON'T:** Assume high-contrast/forced-colors mode is a rare edge case not worth testing. It's a real, actively-used operating system accessibility feature for a meaningful population, and an interface that silently breaks under it is failing exactly the users who turned it on because they need it.

### Opacity and Transparency as a Design Tool

- **DO:** Use opacity and transparency deliberately for specific, well-understood purposes — a disabled-state reduction, a subtle scrim behind a modal, a layered glass/blur effect — always re-checking contrast on the resulting composited color, not the opaque value alone.

- **DON'T:** Use partial opacity as a substitute for a genuinely designed muted color token (see Design Tokens Over One-Off Magic Values, under Components). An opacity-based "gray" composites differently depending on what's visually behind it, producing inconsistent actual color depending on context, whereas a fixed neutral-scale token does not.

- **DO:** Verify that any text or icon rendered at reduced opacity still meets its required contrast ratio against whatever is actually behind it once composited, since a value that looks fine against a design tool's default white artboard can fail against a real, non-white production background.

- **DON'T:** Apply a reduced-opacity treatment to informational text (not just decorative or genuinely inert content) without checking the resulting contrast. It's easy to accidentally push meaningful text below the WCAG minimum by lowering its opacity for a "softer" look.

### Multi-State Status Color Systems

- **DO:** Design a coherent, ordered color progression for systems with more than the basic success/warning/danger triad — a multi-stage pipeline, a kanban board, a priority scale — where each state's color is distinguishable, consistently applied everywhere that state appears, and (per the color-blindness guidance above) paired with a text label or icon.

```text
GOOD: a 5-stage pipeline with a consistent color per stage, reused
identically across every board, list, and report in the product —
Backlog (gray) → In Progress (blue) → In Review (amber) → Blocked (red) → Done (green)
```

- **DON'T:** Let different views or teams within the same product invent their own color mapping for the same underlying set of states. If "In Review" is amber on one team's board and purple on another's, the color loses its value as a fast, at-a-glance status signal the moment a user moves between contexts.

- **DO:** Keep a status color's meaning stable over the life of the product — once "blocked" is established as red, avoid quietly repurposing red for a different, unrelated status later, since users build real muscle memory around established status colors.

- **DON'T:** Reassign an established status color to a new, different meaning without a strong reason and clear communication. Repurposing a well-learned color signal is one of the more disorienting changes a product can make, since it invalidates users' existing pattern recognition rather than simply adding something new to learn.

## Layout & Spacing

### Full-Bleed vs. Contained Sections

- **DO:** Alternate deliberately between full-bleed (edge-to-edge) and contained/max-width sections on longer marketing or content pages to create visual rhythm and clearly demarcate distinct sections, using full-bleed treatment purposefully (a hero image, a background-color band) rather than as a default for every section.

- **DON'T:** Make every section on a page full-bleed by default with no variation, or leave text and interactive content itself running full-bleed with no contained max-width applied on top. Even within a full-bleed background band, the actual readable content typically still needs its own contained measure (see Measure and Line Length, under Typography).

### Grid Edge Cases: Incomplete Final Rows

- **DO:** Decide deliberately how a card or item grid should handle a final row that doesn't fully fill all available columns — left-align the remaining items (the common, usually correct default) rather than letting them stretch to fill unused space or center awkwardly with asymmetric gaps.

```css
/* GOOD: an incomplete final row stays left-aligned, not stretched or centered */
.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  justify-content: start;
}
```

- **DON'T:** Let a grid layout's default behavior stretch the last row's items to fill the full row width when the count doesn't divide evenly, producing inconsistent item widths between a full row and a partial one. An item in an incomplete final row stretched wider than its siblings above is a subtle but noticeable inconsistency once a user compares row to row.

### Content-First Space Budgeting

- **DO:** Give the majority of a screen's visual real estate to actual content the user came for, keeping navigational and structural chrome (headers, sidebars, toolbars) as compact as its function genuinely requires.

- **DON'T:** Let persistent chrome — an oversized header, a wide always-expanded sidebar, redundant toolbars — consume a disproportionate share of the viewport relative to the actual content or task the user is there for. Every pixel spent on chrome is a pixel not spent on the content the interface exists to serve.

- **DO:** Re-evaluate chrome sizing specifically at smaller viewports, where the cost of fixed-size chrome is proportionally much higher — a sidebar that's a reasonable proportion of a 1440px-wide desktop screen can consume nearly all of a 375px-wide phone screen if not deliberately adapted.

- **DON'T:** Let chrome elements retain their desktop proportions unmodified on mobile viewports. What's a minor, proportionate space cost on a wide screen can become the dominant use of screen space on a narrow one if the same fixed sizing is carried over unchanged.

Layout and spacing are what make an interface feel calm, intentional, and trustworthy — or cramped, arbitrary, and cheap — independent of any individual color or font choice. A screen with perfect typography and color but inconsistent, ungoverned spacing still reads as unfinished, because misaligned edges and unpredictable gaps are exactly the kind of small inconsistency human perception is extremely good at detecting, even when the viewer can't articulate what's wrong.

### A Consistent Spacing Scale

- **DO:** Adopt a base-unit spacing scale — most commonly a 4px or 8px base — and restrict every margin, padding, and gap value in the system to multiples of that base unit. A governed scale (4, 8, 12, 16, 24, 32, 48, 64…) guarantees that any two spacing values in the interface relate to each other in a predictable, learnable way.

```css
/* GOOD: an 8pt-based spacing scale expressed as tokens */
:root {
  --space-1: 4px;    /* half-step, for tight inline gaps */
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;
  --space-12: 48px;
  --space-16: 64px;
}
.card { padding: var(--space-4); }
.card + .card { margin-top: var(--space-6); }
```

- **DON'T:** Use arbitrary spacing values — `13px`, `22px`, `7px` — scattered through component styles because each looked "about right" in isolation. Off-scale values accumulate invisibly and produce a layout where nothing quite lines up with anything else, even though no single value looks obviously wrong on its own.

- **DO:** Express spacing as named, referenceable tokens (`space-1` through `space-16`, or similarly named) consumed by every component, exactly as with color and type, so a global rhythm adjustment is a token change rather than a search across the whole codebase.

- **DON'T:** Hardcode raw spacing numbers per component with no shared reference. Hardcoded values make it impossible to answer "what is our standard gap between a label and its input?" with anything more precise than "whatever the last person who touched that component typed in."

- **DO:** Pair the spacing scale with the type scale's line-height so vertical rhythm stays consistent — a paragraph's line-height and the margin below it should both resolve to multiples of the same base unit, so text blocks and non-text elements share one implicit grid.

- **DON'T:** Let spacing and typography evolve on separate, disconnected number systems. When a 1.47 line-height sits next to a 13px margin, the vertical rhythm feels accidental even if each number individually seems reasonable.

- **DO:** Use a slightly denser sub-scale (a 4px increment) for tight, related groupings — icon-to-label gaps, a form field's internal padding — and the coarser scale (8px multiples and up) for structural spacing between distinct sections, components, or content blocks.

- **DON'T:** Apply the same large spacing value uniformly regardless of how closely related two elements are. Equal spacing between closely related items and unrelated items erases the proximity cue spacing is supposed to provide (see Gestalt principles, below).

### Alignment Discipline

- **DO:** Align elements to a shared grid or a consistent set of edges so that related content lines up visually — card edges align with the edges of the section above them, form labels align with each other, icon and text baselines align. Aligned edges are one of the strongest, cheapest signals of a deliberately designed layout; the eye registers alignment even when it isn't consciously looking for it.

- **DON'T:** Nudge one element a few pixels off its shared alignment "because it looked better" in one specific spot. A single broken alignment is often more noticeable than a whole layout that is uniformly slightly off, because it breaks the implicit grid the eye had already started relying on.

- **DO:** Use a consistent left edge (in LTR layouts) for related content blocks — text, icons, form controls — within a section, so a user's eye can track straight down the page without re-locating the start of each new element.

- **DON'T:** Center-align long-form body text, lists, or forms. Centered text and controls lack a consistent starting edge, which both slows scanning and looks noticeably less structured than left-aligned content — reserve centering for short, self-contained elements like a hero headline, a single button, or a small icon-and-caption pairing.

- **DO:** Align icons to the optical center of the text they accompany, adjusting for the icon's own internal visual weight, rather than trusting the icon's bounding box for vertical centering. Many icons (arrows, play buttons, triangular shapes) are not visually centered within their own square bounding box, so a mathematically centered bounding box can still look visibly off-center next to text.

- **DON'T:** Trust automatic bounding-box centering as a substitute for a visual check. Render the icon next to its label at actual size and adjust by eye (or use an icon set that has already corrected for optical centering) rather than assuming the math is sufficient.

- **DO:** Keep a consistent right edge for right-aligned content (numeric table columns, action button groups) just as rigorously as the left edge is kept consistent for reading content — a ragged right edge on a column of right-aligned numbers is just as visible a flaw as ragged left edges on body text.

- **DON'T:** Let padding or icon-width differences between rows silently break a table's right-alignment. A single row with a slightly wider action icon can shift its numeric column out of alignment with every row above and below it — verify alignment matters as much as any padding value used to achieve it.

### Whitespace as an Active Design Tool

- **DO:** Treat whitespace as an active tool for grouping, separating, and drawing attention — not as empty leftover space to be minimized. The gap around an element communicates its relationship to everything nearby just as much as any visible border or background color does.

- **DON'T:** Treat whitespace as wasted real estate to be filled with more content, more decoration, or a tighter layout "to fit more on screen." Removing whitespace to cram in more content usually increases cognitive load and error rate faster than it increases useful information density.

- **DO:** Increase the whitespace surrounding a primary call-to-action relative to its neighbors, since isolation is one of the strongest ways to draw the eye — an element surrounded by generous empty space reads as more important even before its color or size is considered.

- **DON'T:** Crowd a primary action edge-to-edge against secondary controls with no differentiating space. Equal spacing around competing actions removes one of the cheapest, most effective hierarchy signals available.

- **DO:** Use two distinct scales of whitespace deliberately: macro whitespace between major structural sections (page margins, the gap between a header and the content below it) and micro whitespace between closely related elements within one component (a label and its input, an icon and its text). Different relationship strengths deserve visibly different amounts of space.

- **DON'T:** Apply one flat spacing value everywhere regardless of the strength of the relationship between the elements it separates. A single undifferentiated gap value makes every relationship in the layout read as equally strong, which is rarely true and defeats the purpose of using spacing as a hierarchy signal at all.

- **DO:** Let generous whitespace do the work of separating sections instead of defaulting to a visible divider line for every boundary. A well-judged gap frequently communicates separation as clearly as a rule line, with a lighter visual footprint.

- **DON'T:** Reach for a border or divider line as the default solution for every section boundary. Overuse of visible dividers adds visual clutter and can make a screen feel more fragmented and busier than the same content separated purely by whitespace.

### Responsive and Mobile-First Layout

- **DO:** Design and build for the smallest supported viewport first, then progressively add layout complexity, columns, and secondary content as viewport width increases. Starting from the most constrained case forces an honest prioritization of what content and actions actually matter most, which then carries through cleanly to larger screens.

- **DON'T:** Design for desktop first and treat the mobile layout as an afterthought "shrink to fit" pass. Desktop-first design routinely produces a mobile experience with buried primary actions, dense multi-column content forced into a single narrow column with no re-prioritization, and touch targets sized for a mouse cursor.

- **DO:** Use relative and fluid sizing — percentages, `fr` units in CSS Grid, `clamp()` for spacing and type — so layouts adapt continuously across the full range of viewport widths rather than only at a handful of fixed breakpoints.

```css
/* GOOD: fluid spacing that scales continuously with viewport width */
.section {
  padding-inline: clamp(1rem, 5vw, 4rem);
}
```

- **DON'T:** Hardcode fixed pixel widths for layout containers that only look correct at one specific viewport width and either overflow or leave awkward empty space at every other width.

- **DO:** Re-prioritize content, not just resize it, at each breakpoint — collapse secondary navigation into a menu, stack multi-column comparisons vertically, and hide or defer non-essential content behind progressive disclosure on small screens.

- **DON'T:** Simply scale every element down proportionally on a small screen while keeping the same information density. Content that fits comfortably at desktop width frequently needs genuine re-architecture, not just uniform shrinking, to remain usable on a phone.

- **DO:** Test layouts at real device widths and with a real on-screen keyboard visible for mobile forms, since a form that looks fine in a static mockup can have its primary action pushed off-screen once the keyboard occupies the bottom half of the viewport.

- **DON'T:** Only test responsive layouts by resizing a desktop browser window. A resized browser doesn't reproduce mobile-specific realities like the on-screen keyboard's viewport impact, safe-area insets around notches/home indicators, or actual touch-target behavior.

### Grid Systems and Breakpoints

- **DO:** Use a consistent column grid — commonly 4 columns on mobile, 8 on tablet, and 12 on desktop, each with a defined gutter width — as the structural basis for every layout, so components and content align to the same underlying grid across the whole product.

```css
/* GOOD: a defined 12-column grid with consistent gutters, reused everywhere */
.grid-12 {
  display: grid;
  grid-template-columns: repeat(12, 1fr);
  gap: var(--space-6);
}
.content-span-8 { grid-column: span 8; }
```

- **DON'T:** Invent a new, one-off column arrangement per page because a particular layout "needed" a slightly different split. Ad hoc grids make it impossible for components to be reused reliably across pages, since each page's grid math is different.

- **DO:** Choose breakpoints based on where your own content and components actually start to break — where a nav collapses awkwardly, where a card grid gets too cramped — rather than blindly adopting a generic device-width list from a framework's defaults.

- **DON'T:** Copy a standard breakpoint list without validating it against your own layouts. Generic breakpoints are a reasonable starting point, but shipping them unchecked risks a layout that breaks between two defined breakpoints, in the range nobody actually tested.

- **DO:** Document and reuse a small, fixed set of standard breakpoints (commonly 3–5) across the entire codebase, referenced by name (`sm`, `md`, `lg`, `xl`) rather than by raw pixel value at each call site.

- **DON'T:** Sprinkle one-off `@media` queries with unique, uncoordinated pixel values throughout the codebase. Each additional unique breakpoint value is one more place layouts can silently diverge from each other at slightly different widths, producing inconsistent behavior that's hard to reason about or fix globally.

### Visual Hierarchy Through Size, Contrast, Position, and Whitespace

- **DO:** Establish one clear focal entry point per screen by combining size, contrast, position, and whitespace together, so a user's eye has an obvious, unambiguous first place to land.

- **DON'T:** Give several elements equal visual weight and expect the user to intuit which one matters most. When five things compete for attention with no distinguishing hierarchy, the effective result is that nothing successfully draws attention — the user has to read everything before finding what to do.

- **DO:** Place the primary action or key content where established scanning patterns naturally land — the top-left-to-bottom-right diagonal of an F-pattern for text-heavy content, or the four corners of a Z-pattern for simpler, more visual layouts — rather than fighting against how people actually scan a page.

- **DON'T:** Bury the primary action below secondary or tertiary actions in visual weight (smaller, lower-contrast, further down) just because of an arbitrary content order. Visual hierarchy should reflect actual task priority, not incidental document order.

- **DO:** Use size differences deliberately and with enough contrast to be unambiguous — a heading meant to outrank another should be noticeably larger, not marginally larger by a few pixels that could be attributed to rendering variance.

- **DON'T:** Rely on a single hierarchy signal in isolation when a decision genuinely matters. Combine at least two of size, weight, color/contrast, position, and whitespace for any hierarchy relationship that the user actually needs to correctly perceive (as opposed to a purely decorative distinction).

### Gestalt Principles Applied to Interface Layout

- **DO:** Apply the principle of proximity by placing related elements closer together than unrelated ones — a label directly above or beside its input, with more space before the next unrelated field, so grouping is perceived instantly without needing a visible border around each group.

```text
BAD (equal spacing everywhere — no perceived grouping):
Name  [______]
Email [______]
Phone [______]

GOOD (proximity signals which label belongs to which field):
Name
[______]

Email
[______]
```

- **DON'T:** Space a label and its own input the same distance apart as that input is from the next, unrelated field. Equal spacing destroys the proximity cue entirely — the user has to read every label carefully to figure out which field it actually belongs to, instead of perceiving the grouping instantly.

- **DO:** Apply the principle of similarity by styling elements that share the same function identically across the whole product — every primary button uses the same shape, color, and weight; every card uses the same corner radius and shadow. Similarity is what lets a user recognize "this is a button" or "this is a card" without reading it, purely from its visual treatment.

- **DON'T:** Let two controls that perform the same kind of action look meaningfully different from each other for no functional reason (one primary button rounded, another primary button square, elsewhere in the same flow). Inconsistent styling for equivalent functions breaks the pattern-recognition shortcut similarity is supposed to provide.

- **DO:** Apply the principle of continuity by aligning related elements along a shared implied line or axis, so the eye flows smoothly from one to the next — a vertical list of icons and labels that share a left edge, a horizontal row of steps in a progress indicator that share a baseline.

- **DON'T:** Scatter functionally related elements off any shared axis, forcing the eye to jump unpredictably between them instead of following one continuous line. Broken continuity is a common, subtle cause of a layout feeling "busy" even when individual elements are each well-designed.

- **DO:** Apply the principle of closure where appropriate — in logo marks, icons, or loading indicators — allowing the eye to perceive a complete shape from an incomplete outline, which can produce a more elegant, memorable mark than fully rendering every edge.

- **DON'T:** Rely on closure so subtly that the intended shape becomes illegible, especially at small sizes. Closure is a powerful technique in a large logo mark; the same technique applied to a 16px icon can simply read as a broken or malformed shape rather than an intentional, implied one.

- **DO:** Apply the figure/ground principle by ensuring sufficient contrast between foreground content ("figure") and its background ("ground"), so text and interactive elements are unambiguously perceived as sitting on top of, not blended into, whatever is behind them.

- **DON'T:** Let a decorative background pattern, texture, or photograph compete directly with foreground text or controls for attention. When figure and ground have similar visual weight, both become harder to parse — apply an overlay, scrim, or sufficient contrast adjustment to clearly separate the two.

- **DO:** Apply the principle of common region by grouping related content inside a shared visual container — a card, a bordered panel, a shaded section — when proximity and similarity alone aren't a strong enough grouping signal, particularly for dense dashboards with many small, related data points.

- **DON'T:** Default to wrapping every group of content in a bordered card regardless of whether the grouping is actually ambiguous. Overusing common region containers when proximity alone would have communicated the grouping clearly adds visual noise and nested-box fatigue to a layout.

### Elevation, Depth, and Layering

- **DO:** Use a small, defined set of elevation levels (e.g., flat, raised, overlay, modal) each with a consistent shadow, border, or background-lightness treatment, so a user learns to read "how far above the base surface" a given element sits purely from its consistent visual treatment.

```css
/* GOOD: a governed elevation scale reused everywhere depth is needed */
:root {
  --elevation-1: 0 1px 2px rgba(0,0,0,0.06);
  --elevation-2: 0 4px 8px rgba(0,0,0,0.10);
  --elevation-3: 0 12px 24px rgba(0,0,0,0.16);
}
.card    { box-shadow: var(--elevation-1); }
.popover { box-shadow: var(--elevation-2); }
.modal   { box-shadow: var(--elevation-3); }
```

- **DON'T:** Let each component invent its own one-off shadow value (blur radius, spread, opacity all picked independently). Ungoverned shadows make elevation an unreliable signal — a card with a heavier shadow than the modal above it inverts the depth cue the shadow was supposed to communicate.

- **DO:** Reduce reliance on drop shadows for perceived elevation in dark mode, since shadows are far less visible against a dark background — use a lighter surface color, a subtle border, or a combination of both to communicate elevation instead.

- **DON'T:** Reuse the exact same shadow values from the light theme unmodified in dark mode and expect them to communicate elevation. A shadow tuned to be barely-there against white becomes nearly invisible against a near-black surface, silently losing the depth cue it was meant to provide.

### Z-Index and Stacking Context Management

- **DO:** Define a small, governed set of z-index tiers as tokens (e.g., base, dropdown, sticky-header, overlay, modal, toast/notification) and require every stacking element to use one of them, so the relative stacking order of any two UI layers is always predictable.

```css
/* GOOD: a governed z-index scale, not arbitrary numbers per component */
:root {
  --z-dropdown: 100;
  --z-sticky-header: 200;
  --z-overlay: 300;
  --z-modal: 400;
  --z-toast: 500;
}
```

- **DON'T:** Let individual components set arbitrary z-index values (`z-index: 9999`, `z-index: 99999999`) whenever something visually appears behind something else it shouldn't. Escalating ad hoc z-index values is a common "fix" that treats the symptom, not the actual stacking-context problem, and it makes the next similar conflict even harder to resolve predictably.

- **DO:** Understand and account for how new stacking contexts are created (by `transform`, `opacity` less than 1, `filter`, and other properties, not just `position` and `z-index`), since a component with an unexpectedly low effective stacking position is often the result of an ancestor accidentally creating a new stacking context, not a wrong z-index value on the component itself.

- **DON'T:** Debug a stacking-order problem purely by escalating the z-index number on the element that appears "behind" without checking whether an ancestor element has created an isolating stacking context. Increasing a nested element's z-index does nothing if its stacking context ancestor is already positioned below the competing element.

### Safe Areas and Device Insets

- **DO:** Account for device safe areas — the notch, dynamic island, home indicator, and rounded corners on modern phones — using the platform's safe-area inset values, so critical content and interactive controls are never rendered underneath or dangerously close to these physical obstructions.

```css
/* GOOD: respects the device's safe area insets */
.bottom-action-bar {
  padding-bottom: max(16px, env(safe-area-inset-bottom));
}
```

- **DON'T:** Position a primary action or important content flush against the very edge of the screen with no safe-area accommodation. On modern phones this risks the control being partially obscured by a home indicator, rounded corner, or notch, or simply landing in a zone that's uncomfortable or impossible to reliably tap.

- **DO:** Test layouts on actual devices (or accurate simulators) with notches, dynamic islands, and varying aspect ratios, not only on a generic rectangular viewport in a browser resize test.

- **DON'T:** Assume a layout validated only in a standard browser device-emulation mode fully represents real device constraints. Emulators frequently don't reproduce every physical safe-area nuance a real device enforces.

### Dense Data Layouts: Tables and Grids

- **DO:** Right-align numeric columns and left-align text columns in a data table by default, use tabular figures (see Typography) for numeric alignment, and keep row height and cell padding consistent throughout, so a dense table remains scannable rather than becoming a wall of ambiguous numbers.

- **DON'T:** Left-align numeric columns in a data table. Left-aligned numbers of varying digit-length are far harder to visually compare at a glance than right-aligned ones, where the ones-place digits all line up in a single scannable column.

- **DO:** Use zebra striping, subtle row dividers, or adequate row height sparingly and consistently to help the eye track across a wide table row without losing its place, particularly for tables wider than a single screen's comfortable reading width.

- **DON'T:** Rely on extremely tight row spacing with no visual differentiation between adjacent rows in a dense table. Rows that are too close together and visually undifferentiated make it easy to lose track of which row the eye was following, especially when scanning horizontally across many columns.

- **DO:** Provide sticky headers (and, for very wide tables, sticky key columns) so column labels and identifying information stay visible as the user scrolls a long or wide table.

- **DON'T:** Let column headers scroll out of view on a long table with no sticky behavior, forcing the user to scroll back to the top repeatedly just to remember what a given column means.

### Container Queries and Component-Level Responsiveness

- **DO:** Use container queries, where supported, to let a component adapt its own internal layout based on the space actually available to it, rather than only the overall viewport width — a card component should be able to reflow correctly whether it's placed in a wide single-column layout or a narrow sidebar.

```css
/* GOOD: the card adapts to its own container's width, not just the viewport's */
.card-container { container-type: inline-size; }
@container (min-width: 400px) {
  .card { grid-template-columns: 120px 1fr; }
}
```

- **DON'T:** Rely purely on viewport-level media queries for a component meant to be reused in varying-width contexts (a sidebar, a modal, a full-width section). A component that only responds to overall viewport width will misrender whenever it's placed somewhere narrower or wider than the viewport itself would suggest.

- **DO:** Design reusable components to degrade gracefully at a range of container widths, not just the specific width they were originally designed and tested at, since a genuinely reusable component will end up placed in contexts its original designer didn't anticipate.

- **DON'T:** Test a shared component only at the one container width it happens to be used at today. A component with untested behavior at other widths is a latent bug waiting for the first time someone reuses it in a narrower or wider context.

### Print Stylesheets and Page Breaks

- **DO:** Provide a dedicated print stylesheet for any content users are likely to print or export as a static document (invoices, reports, printable schedules), controlling page breaks explicitly so content doesn't split awkwardly across pages.

```css
/* GOOD: prevents an awkward mid-element page break when printed */
@media print {
  .invoice-line-item, .card, table tr { break-inside: avoid; }
  h2 { break-after: avoid; }
}
```

- **DON'T:** Leave printable content with no page-break control, letting a table row, a card, or a heading split arbitrarily across two printed pages wherever the content happens to overflow. An uncontrolled page break in the middle of a logically single unit (a table row, a labeled figure) makes printed output genuinely hard to read.

### Page Margins and Content Extremes

- **DO:** Give a page's outer margins enough breathing room to keep content from feeling like it's pressed against the edge of the viewport, scaling that margin down sensibly (never to zero, on any but the smallest components) as viewport width decreases.

- **DON'T:** Let primary content run edge-to-edge on a desktop-width viewport with no outer margin at all. Content with no framing margin reads as unfinished and makes the page's edges feel arbitrary rather than intentional, especially on wide displays.

- **DO:** Keep consistent outer margins across every page of the same type, so the overall page "frame" feels stable as a user navigates between them, even as the content inside that frame changes.

- **DON'T:** Let outer margin values vary inconsistently from page to page within the same product for no functional reason. Inconsistent framing is a subtle but real contributor to a product feeling assembled from disconnected pieces rather than built as one system.

### Density Settings: Comfortable vs. Compact Views

- **DO:** Offer a density toggle (comfortable/compact, or similar) for data-heavy interfaces — tables, lists, dashboards — where different users have genuinely different needs (a casual user wanting more breathing room, a power user wanting to see more rows at once), implemented by switching spacing-scale values, not by rebuilding the layout.

- **DON'T:** Force one fixed density on every user of a data-heavy interface when the underlying use cases genuinely diverge — a support agent triaging hundreds of tickets a day has very different density needs from an occasional user checking a single record.

- **DO:** Keep a density setting's effect systematic — driven by swapping a set of spacing tokens, not by manually adjusting each component's padding independently — so every component responds to the density setting consistently.

- **DON'T:** Implement "compact mode" as a one-off, manually-tuned override applied inconsistently to only some components. A partial, inconsistent compact mode where some elements shrink and others don't produces a visually uneven, unfinished-feeling result.

### Scroll Behavior and Anchor Positioning

- **DO:** Use smooth scrolling deliberately for short, purposeful jumps within a page (an in-page anchor link, "back to top") while keeping it brief enough not to feel sluggish, and always allow it to be interrupted by further user scroll input.

- **DON'T:** Apply smooth-scroll globally to every scroll interaction, including the user's own manual scrolling via mouse wheel or trackpad. Intercepting and smoothing native scroll input makes an interface feel laggy and unresponsive to a user accustomed to their scroll input translating directly and immediately into movement.

- **DO:** Offset scroll targets for in-page anchor links so the target content doesn't land hidden underneath a sticky header, accounting for the header's height in the scroll calculation.

- **DON'T:** Let an anchor-link jump land content directly underneath a sticky header with no offset, hiding exactly the content the link was supposed to reveal — a small, easily-overlooked detail that undermines an otherwise well-built in-page navigation feature.

### Master-Detail and Split-View Layouts

- **DO:** Use a master-detail (list-and-detail) split layout on wide viewports for content where users frequently move between browsing a list and viewing one item's details, letting them do both without a full page navigation each time, and collapse gracefully to a single-column, drill-down pattern on narrow viewports.

- **DON'T:** Force a full-page navigation and back-button round trip for every single item view in a list a user is actively browsing through sequentially. On sufficiently wide viewports, a split view removes this repeated navigation cost entirely; forcing it anyway on wide viewports is a missed opportunity, not a neutral choice.

- **DO:** Keep the master list's scroll position and selection state intact when a user views different items' details in sequence, so browsing through a list feels continuous rather than resetting with every selection.

- **DON'T:** Reset the list panel's scroll position back to the top every time a different item is selected in a master-detail view. Losing scroll position mid-browse forces the user to re-scroll to find where they were, repeatedly, which quickly becomes a tedious, avoidable friction point.

### Border Radius Systems

- **DO:** Define a small, governed scale of corner-radius values (e.g., none, small, medium, large, full/pill) as tokens, applied consistently by component type — buttons and inputs sharing one radius, cards another, avatars using the full/circular value — so corner treatment reads as a deliberate system rather than a per-component guess.

```css
/* GOOD: a governed radius scale applied consistently by component role */
:root {
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 16px;
  --radius-full: 9999px;
}
.button, .input { border-radius: var(--radius-sm); }
.card { border-radius: var(--radius-md); }
.avatar, .badge-pill { border-radius: var(--radius-full); }
```

- **DON'T:** Let each component use its own hand-picked corner radius with no shared scale — one card at 6px, another at 10px, a button at 5px. Inconsistent, off-scale corner radii are a subtle but consistently noticeable sign of an ungoverned visual system, similar in effect to inconsistent spacing or shadow values.

- **DO:** Scale corner radius proportionally to component size where it matters visually — a small chip and a large modal both using the exact same fixed-pixel radius can look disproportionate, since the same absolute radius reads as much "rounder" relative to a small shape than a large one.

- **DON'T:** Apply one single fixed radius value uniformly to components spanning a wide range of sizes with no adjustment. What looks like a subtle, appropriate rounding on a large card can look like an almost-pill shape on a small chip using the identical pixel value.

### Consistent Border and Divider Treatment

- **DO:** Define a small set of border and divider tokens (width, color, style) reused consistently for the same structural purpose everywhere — every card outline, every table row divider, every input border pulling from the same defined set.

- **DON'T:** Let border width and color vary arbitrarily between similar elements (a 1px border here, a 1.5px border there, in two slightly different grays). Inconsistent border treatment is easy to overlook individually but adds up to a layout that feels subtly unrigorous on close inspection.

- **DO:** Choose between a border and a background-color shift (or both) deliberately for separating adjacent regions, based on how strong a separation is actually needed, rather than defaulting to a visible border for every boundary regardless of context.

- **DON'T:** Add a visible border around every single container by default "for clarity." As with whitespace, overusing visible borders where a subtler cue (spacing, background shift) would do the job adds unnecessary visual noise and makes a layout feel more fragmented than it needs to.

### Sticky and Persistent Elements Beyond Headers

- **DO:** Use sticky positioning deliberately for elements that genuinely benefit from staying reachable during scroll — a persistent form-submission action bar, a sticky filter toolbar above a long list — and keep the stuck element's total footprint small enough that it doesn't meaningfully eat into the content area on smaller viewports.

- **DON'T:** Make too many elements sticky simultaneously (a sticky header, a sticky sub-nav, and a sticky filter bar all stacking at once), collectively consuming a large portion of a small viewport's vertical space for the entire duration of scrolling. Stacked sticky elements are a common, easy-to-overlook way to accidentally starve a mobile viewport of usable content space.

- **DO:** Ensure a sticky footer action bar never permanently obscures content the user needs to see or interact with beneath it, particularly the very last item in a scrollable list, which can otherwise end up permanently hidden behind a fixed-position bar.

- **DON'T:** Let a sticky bottom action bar overlap the final items in a list with no compensating bottom padding on the scrollable content. This is a common, easily-tested oversight where the last one or two items become effectively unreachable, hidden permanently behind the sticky element.

### Content Reflow During Dynamic Height Changes

- **DO:** Animate height changes when content expands or collapses in place (an accordion opening, a "show more" reveal) so the surrounding layout shifts smoothly rather than snapping instantly, and account for the resulting reflow so content below doesn't jump unpredictably out from under the user's cursor or reading position.

- **DON'T:** Let an in-place content expansion instantly change a container's height with no transition, abruptly shoving every element below it down (or up, on collapse) with no visual continuity. An instant, large layout jump — especially one triggered by the user's own click, right as they're still looking at the area — is disorienting and can cause a mis-click on whatever now occupies the space.

- **DO:** Reserve space proactively for content that's about to load or expand where the eventual size is predictable, minimizing the layout shift even before any transition animation begins.

- **DON'T:** Let asynchronously-loaded content (an ad, an embed, an image, a dynamically-fetched section) insert itself into the layout with zero reserved space, pushing everything below it down unpredictably at an unpredictable moment mid-read. This is a common, well-documented source of accidental mis-clicks and reading disruption, distinct from — but related to — the image-dimension guidance under Iconography & Imagery.

## Components & Interaction States

### Undo Patterns for Low-Stakes Reversible Actions

- **DO:** Prefer an "undo" affordance (a toast with an Undo action, a brief grace period before an action is finalized) over an upfront blocking confirmation dialog for actions that are low-stakes and easily reversible, letting the user act quickly and recover just as quickly if the action turns out to be a mistake.

```text
GOOD: clicking "Archive" immediately archives the item and shows a toast —
      "Archived. [Undo]" — for a few seconds, instead of interrupting the
      user with an "Are you sure?" dialog before every archive action.
```

- **DON'T:** Interrupt the user with a blocking confirmation dialog for every low-stakes, easily-reversible action. Confirmation dialogs have a real cost — they slow down every single instance of an action to guard against a mistake that, for a genuinely reversible action, an undo pattern handles more gracefully and with far less friction for the common case where no mistake was made.

- **DO:** Give an undo affordance a reasonable, visible time window and make it easy to trigger (a clearly-labeled button, not a tiny link), since an undo option that's too brief or too hard to notice provides little real protection against mistakes.

- **DON'T:** Make an undo window so short or so visually subtle that it's rarely actually usable in practice. An undo pattern that's technically present but practically unusable offers false reassurance without the real safety net a genuinely usable undo provides.

### Optimistic UI Updates

- **DO:** Update the interface immediately to reflect the expected outcome of a user's action (checking a checkbox, liking a post) before the server has actually confirmed it, then quietly reconcile if the server response differs — this makes routine interactions feel instant rather than waiting on network round-trip time for every small action.

- **DON'T:** Apply optimistic updates to actions where a likely or costly failure would leave the user confused or would need to be silently and confusingly reverted. Optimistic UI is a good fit for low-risk, high-frequency, usually-successful actions; it's a poor fit for anything where failure is common enough or consequential enough that a jarring revert would do more harm than the responsiveness gain is worth.

- **DO:** Handle the failure case of an optimistic update gracefully and visibly — revert the UI change and clearly explain what happened — rather than leaving the interface in an inconsistent state that no longer matches the actual server-side truth.

- **DON'T:** Let an optimistic update's failure path silently leave the UI showing a state that doesn't match reality, with no correction and no explanation. A user who believes their action succeeded, when it silently didn't, may act on that false premise in ways that compound the original problem.

### Inline Editing Patterns

- **DO:** Make an inline-editable field's editability clear before the user clicks into it — a subtle hover affordance (an edit icon, a border appearing on hover) — and provide clear, unambiguous save and cancel actions once editing begins, rather than leaving the user unsure whether a change has been committed.

- **DON'T:** Let a click-to-edit field look completely indistinguishable from static, non-editable text until the user happens to click it. With no discoverable affordance, an inline-editable field is functionally hidden from users who don't already know it's there — the same false-negative affordance problem covered generally under Affordance, above, applied to this specific pattern.

- **DO:** Auto-save or clearly commit an inline edit on blur (clicking away) or on a specific save action, and give unambiguous confirmation that the change was saved, especially since inline editing typically has no separate, obvious "submit" step the way a full form does.

- **DON'T:** Leave it ambiguous whether clicking away from an inline-edited field saves the change or discards it. This ambiguity is worse for inline editing than for a full form, because there's often no visible submit button whose presence would otherwise make the save action obvious.

A component is not "done" when its default appearance looks right — it is done when every state it can legitimately be in has been designed, is visually distinct from its neighboring states, and behaves predictably. Most of what makes a shipped product feel unfinished or buggy is not a missing feature; it's an unstated state — a button with no visible pressed feedback, a form field with no error styling, a list with no defined empty state — encountered live in production for the first time.

### Full Interactive State Coverage

- **DO:** Define, before a component ships, its appearance in every state it can realistically enter: default, hover, focus, active/pressed, disabled, loading, error, success, and empty (where applicable). Treat this as a checklist attached to the component spec, not an implicit expectation left to whoever happens to notice a missing state later.

```text
Button component — required states:
  default | hover | focus-visible | active/pressed | disabled | loading
Input component — required states:
  default | focus | filled | disabled | readonly | error | success | loading
List/table component — required states:
  populated | loading (skeleton) | empty (zero data) | empty (no results) | error
```

- **DON'T:** Ship a component with only a default and hover state designed, leaving focus, disabled, and loading to fall back on inconsistent browser defaults or whatever the underlying framework happens to render. Missing states are exactly the gaps where inconsistency and accessibility failures accumulate.

- **DO:** Make every state visually distinguishable from its neighbors — hover should look different from focus, focus should look different from active, active should look different from disabled. A user (or a developer debugging a screenshot) should be able to tell which state a component is in without needing to interact with it live.

```css
/* GOOD: every state is visually distinct, not just "default plus a filter" */
.btn { background: var(--color-primary-500); }
.btn:hover { background: var(--color-primary-600); }
.btn:focus-visible { outline: 2px solid var(--color-primary-700); outline-offset: 2px; }
.btn:active { background: var(--color-primary-700); transform: translateY(1px); }
.btn:disabled { background: var(--color-neutral-200); color: var(--color-neutral-400); cursor: not-allowed; }
```

- **DON'T:** Use identical or near-identical styling for hover and focus states. A mouse user gets both hover and focus feedback redundantly when they click, so an under-designed team can miss that focus alone — the only cue a keyboard user actually gets — is weak or missing entirely.

- **DO:** Design a genuine loading state for any component that fetches or submits data — a spinner, a skeleton placeholder, or a progress indicator — rather than letting the component sit in a blank or stale state during the wait.

- **DON'T:** Leave a component showing stale or blank content with no visual indication that a request is in flight. A user who can't tell whether their click registered will often click again, potentially triggering duplicate submissions.

- **DO:** Design the transition between states, not just each state's static appearance — how a button moves from default to pressed, how a field transitions from valid to showing an error. An abrupt, undesigned jump between two well-designed static states can still feel broken.

- **DON'T:** Assume that because each individual state "looks fine" in a static mockup, the live transitions between them will automatically feel polished. Transitions need their own explicit design decisions (see Motion & Animation, below).

### Design Tokens Over One-Off Magic Values

- **DO:** Reference the same token for a given state's meaning across every component that has that state — every "error" border uses `color-danger-500`, every "disabled" text uses `color-neutral-400`, every focus ring uses the same `focus-ring` token. Consistency of meaning across components is what makes state changes instantly recognizable regardless of which component the user is looking at.

- **DON'T:** Let each component author hand-pick a new color, shadow, or size for a state because the exact system token "didn't feel quite right" for that one case. A red border here, a slightly different red border there, both meaning "error," is a subtle but real consistency failure that accumulates across a large component library.

- **DO:** Extend tokens to cover non-color state properties too — border widths, opacity levels for disabled states, transition durations, focus ring offsets — not just color. A disabled state governed by a token (`opacity: var(--disabled-opacity)`) stays consistent even as the base color palette changes.

- **DON'T:** Hardcode a magic opacity value (`opacity: 0.45`) directly in one component's disabled state while another component uses `0.5` and a third uses `0.6`. These tiny, inconsistent choices are individually invisible but collectively make the disabled state read as unreliable across the product.

### Affordance: Looking Interactive vs. Looking Static

- **DO:** Style genuinely clickable and interactive elements so they visually announce themselves as interactive — a distinct color or underline for links, a filled or bordered shape with a hover/pressed response for buttons, a pointer cursor on hover. Affordance is the visual promise that "you can act on this," and breaking that promise in either direction erodes trust in the whole interface.

- **DON'T:** Style a real, functioning link or button to look like plain static text with no visual distinction. Users scan interfaces expecting visual cues for interactivity; an interactive element with none of those cues is effectively undiscoverable to anyone who doesn't happen to hover over exactly the right pixels.

- **DO:** Keep static, non-interactive text and decorative elements visually inert — no hover state, no pointer cursor, no button-like shape — so users don't waste effort clicking things that do nothing.

- **DON'T:** Apply button-like styling (rounded background, shadow, hover color shift) to a purely decorative or non-interactive label. False affordance is arguably worse than under-styled real affordance, because it actively trains users to distrust visual cues throughout the rest of the interface once they discover one is a lie.

- **DO:** Keep affordance conventions consistent across the product — if underlined blue text means "link" in one place, it should mean "link" everywhere, not sometimes be a link and sometimes be static emphasized text.

- **DON'T:** Reuse the visual language of an interactive element (the exact button shape, the exact link color) for a non-interactive purpose elsewhere in the same product, purely because it "matched the palette." Overloading one visual signal with two different meanings actively confuses users who have learned the first meaning.

### Form Design

- **DO:** Keep field labels always visible — positioned above or beside the input — for the entire time the user is interacting with the form, never relying on placeholder text as the label's sole substitute.

```html
<!-- BAD: label disappears the moment the user starts typing -->
<input type="email" placeholder="Email address">

<!-- GOOD: a persistent, associated label; placeholder reserved for a format example -->
<label for="email">Email address</label>
<input id="email" type="email" placeholder="name@example.com">
```

- **DON'T:** Use placeholder text as the only label for a field. A placeholder disappears the instant the user types a character, so by the time they need to review what they entered — or a validation error appears — the context of what the field was even for is gone; this pattern also fails basic accessibility because many placeholder implementations don't reliably serve as an accessible name.

- **DO:** Validate fields inline, as close to real time as reasonably possible (typically on blur, or debounced while typing for fields like password strength), showing feedback directly next to the field it concerns.

- **DON'T:** Defer all validation to a full-form submit attempt, then dump every error at once at the top of the page with no per-field indication. This forces the user to hunt through the entire form to find which fields the errors refer to, turning a quick fix into a frustrating search.

- **DO:** Write field-specific, actionable validation messages that state the actual rule that failed and how to satisfy it — "Enter a valid email address, like name@example.com" or "Password must be at least 8 characters and include a number."

- **DON'T:** Show a generic, non-specific message like "Invalid input" or "Error" with no indication of which rule was violated or how to fix it. A message that doesn't tell the user what to do next forces them to guess-and-check, which is exactly what inline, specific validation exists to prevent.

- **DO:** Group related fields visually and structurally — an address block, a payment details section, a name/contact section — using spacing, subheadings, or containers so the form's structure communicates its logical organization at a glance.

- **DON'T:** Present a long, flat, ungrouped list of fields with no visual structure. An unstructured form reads as a wall of inputs and forces the user to read every label carefully just to understand what section of information they're currently filling in.

- **DO:** Mark required and optional fields clearly and consistently — commonly marking the minority case (if most fields are required, mark the few optional ones "(optional)"; if most are optional, mark the required ones) using text, not color alone.

- **DON'T:** Indicate a required field using only a red asterisk with no text equivalent and no `required`/`aria-required` attribute. A color-only or purely visual required-indicator fails for screen reader users and for anyone who can't reliably distinguish the asterisk's color from surrounding text.

- **DO:** Preserve user input across a failed submission or a page reload wherever technically possible, especially for long forms — nothing damages trust in a form faster than losing everything the user typed because of one field's error.

- **DON'T:** Clear the entire form back to empty state after a failed validation or a server error. Forcing a user to re-enter twenty fields because one was wrong is a severe, easily avoidable usability failure.

- **DO:** Keep the submit action clearly labeled with the specific outcome it produces ("Create account", "Save changes") and disable or show a loading state on it during submission to prevent duplicate submits, re-enabling it promptly if submission fails.

- **DON'T:** Leave a submit button clickable multiple times in rapid succession while a request is in flight with no loading feedback. This is one of the most common causes of duplicate orders, duplicate accounts, and duplicate database rows in production systems.

### Touch Target Sizing

- **DO:** Size every interactive touch target at a minimum of roughly 44×44pt on iOS or 48×48dp on Android (both translate to roughly 44–48 CSS px on the web), including the invisible tappable padding around a visually smaller icon or piece of text, not just its rendered visual size.

```css
/* GOOD: a visually 20px icon with an invisible tap target meeting the minimum */
.icon-button {
  width: 44px;
  height: 44px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}
.icon-button svg { width: 20px; height: 20px; }
```

- **DON'T:** Ship a 20–24px icon button on a touch interface with no additional padding, using the icon's visible bounds as the sole tap target. Undersized touch targets measurably increase mis-tap rate, which is especially costly for destructive actions (delete, remove) placed near safe ones.

- **DO:** Expand a visually compact control's tap area using padding, a larger invisible hit-box, or (on the web) a pseudo-element sized to the minimum, so the visual design can stay compact while the actual interactive area still meets the accessible minimum.

- **DON'T:** Rely purely on the rendered visual size of a small icon as its tap target because "shrinking the hit area to match" was simpler to implement. The visual and interactive sizes of a control are two different design decisions and don't need to match.

- **DO:** Maintain adequate spacing (commonly at least 8px) between adjacent touch targets, particularly when at least one of them triggers a destructive or hard-to-reverse action, to reduce the chance of an accidental mis-tap selecting the wrong one.

- **DON'T:** Place two small tappable controls directly adjacent with zero gap between them, especially a safe action next to a destructive one (e.g., "Archive" directly touching "Delete forever"). The cost of a mis-tap here is asymmetric and deserves defensive spacing.

- **DO:** Increase touch target size further for primary, frequently-used actions and for any control likely to be used one-handed or while the user is in motion (mobile navigation, a call-to-action reachable by a thumb). The comfortable target size scales up, not down, with how often and how casually a control gets tapped.

- **DON'T:** Apply only the bare legal/guideline minimum uniformly regardless of context. A primary "next" button on a checkout flow benefits from being noticeably larger than the 44px floor, both for ergonomics and for the confidence-inspiring visual weight a larger primary action conveys.

### Loading States and Skeleton Screens

- **DO:** Use skeleton screens (low-fidelity placeholders shaped like the content that's about to load) for predictable, structured content — lists, cards, article layouts — so the user perceives progress and understands roughly what's coming, rather than staring at a blank screen or a generic spinner.

- **DON'T:** Show a full-screen blocking spinner for content that could instead be progressively revealed with skeleton placeholders. A spinner communicates only "something is happening," while a skeleton communicates "this specific layout is about to appear," which reduces perceived wait time even when actual load time is identical.

- **DO:** Match a skeleton's shape and layout closely to the actual content it's standing in for, so the transition from skeleton to real content feels like a smooth reveal rather than a jarring layout shift.

- **DON'T:** Use a generic, undifferentiated skeleton shape (a few gray bars) for structurally different content types across the product. A skeleton that doesn't resemble what's actually loading provides little of the perceived-progress benefit and can make the eventual content swap feel like an unrelated layout jump.

- **DO:** Reserve simple spinners for short, unstructured, or unpredictable-duration waits (a button's own submit action, a small inline fetch) where a full skeleton layout would be disproportionate.

- **DON'T:** Build an elaborate skeleton screen for an operation that reliably completes in under roughly 300–400ms. Below that threshold, a loading indicator of any kind can introduce more perceived flicker and distraction than it removes — very fast operations are often better served by no loading indicator at all.

### Tooltips, Popovers, and Contextual Overlays

- **DO:** Reserve tooltips for supplementary, non-essential information — clarifying an icon's meaning, providing a keyboard shortcut hint — and ensure the same information is discoverable another way for users who can't easily trigger a hover state (touch devices, keyboard-only navigation).

- **DON'T:** Put essential information or the only path to a needed action inside a hover-only tooltip. Touch devices have no true hover state and keyboard navigation doesn't naturally trigger CSS `:hover`, so hover-only content is effectively hidden from a meaningful share of users.

- **DO:** Trigger tooltips on both hover and keyboard focus, and dismiss them on blur or Escape, so keyboard users get the same contextual help mouse users do.

- **DON'T:** Wire a tooltip exclusively to a mouseover/mouseout event pair with no focus/blur equivalent. This silently excludes keyboard-only users from information sighted mouse users take for granted.

- **DO:** Keep popovers and dropdown menus positioned so they never get clipped by a viewport edge or a scrolling container — reposition (flip above instead of below, or shift horizontally) when the default position would overflow.

- **DON'T:** Let a popover render partially off-screen or behind another element with a higher z-index with no collision detection. An overlay a user can't fully see or reach is a functional failure, not just a cosmetic one.

### Notifications and Toasts

- **DO:** Keep transient notifications (toasts) brief, dismissible, and non-blocking, auto-dismissing after a reasonable duration for purely informational messages while requiring explicit dismissal for anything the user must actively acknowledge (an error requiring action, for instance).

- **DON'T:** Auto-dismiss a toast carrying an error the user needs to read and act on before it disappears. A five-second auto-dismiss is appropriate for "Changes saved"; it is not appropriate for "Payment failed — update your card."

- **DO:** Stack multiple simultaneous notifications predictably (commonly newest on top or newest at the bottom, consistently) rather than letting them overlap or replace each other unpredictably.

- **DON'T:** Let a new toast silently replace or obscure a previous one the user hadn't finished reading. If notifications can arrive in quick succession, design an explicit stacking or queuing behavior rather than leaving it to whatever the DOM happens to do.

- **DO:** Position notifications where they don't obscure the content or control the user is most likely to need immediately after triggering them (e.g., not directly on top of the form the user just submitted).

- **DON'T:** Place a confirmation toast directly over the primary action button that triggered it, forcing the user to wait for it to disappear before they can interact with that area again.

### Disabled State Design and Its Alternatives

- **DO:** Consider whether a control should be disabled at all versus left enabled with inline validation explaining why an action can't proceed yet. A disabled button with no explanation is a dead end; an enabled button that, when clicked, explains exactly what's missing is often more usable.

- **DON'T:** Disable a submit button with no accompanying explanation of what the user still needs to do to enable it. A silently disabled control is one of the most common sources of "I don't know why I can't click this" support requests.

- **DO:** When a control genuinely should be disabled, style it with a clear, consistent disabled treatment (reduced opacity, neutral coloring, `cursor: not-allowed`) that's still legible enough to read what the control would say if it were enabled.

- **DON'T:** Style a disabled control so faintly it becomes essentially invisible or unreadable. A disabled state should communicate "not available right now," not "this doesn't exist" — the user should still be able to see what the option is, even if they can't currently use it.

- **DO:** Pair a disabled control with a tooltip, helper text, or inline message explaining the condition that would enable it, whenever that condition isn't otherwise obvious from context ("Select at least one item to enable this action").

- **DON'T:** Leave the reason for a disabled state entirely implicit, expecting users to correctly infer business logic that isn't visible anywhere on screen.

### Empty States as a Designed Component

- **DO:** Treat "zero data" as a first-class state for any list, table, dashboard, or collection component, with its own designed layout — not the same container simply rendering nothing.

- **DON'T:** Let an empty list or table silently collapse to a blank area with no heading, no explanation, and no next step. A blank rectangle where content should be reads as broken, not as "correctly showing that there's nothing here yet."

- **DO:** Distinguish between a true zero-state (the user has never created anything here) and a filtered-to-zero state (a search or filter combination matched nothing), since the correct next action differs — "create your first X" versus "adjust your filters" — and showing the wrong one misleads the user about what to do next.

- **DON'T:** Reuse identical empty-state copy and imagery for both situations. A new user staring at "No results found" when they haven't created anything yet may reasonably conclude the product is broken rather than simply empty.

### Component Composition and Props API Discipline

- **DO:** Design a component's variant and prop surface (size, tone/intent, state) as a small, closed set of explicit options, so every valid combination has been considered and looks correct together.

- **DON'T:** Expose free-form style overrides (arbitrary class injection, inline style props) as the primary way to customize a shared component. Unconstrained overrides let every consumer silently create a new one-off variant that the component's original design never accounted for, which is how design systems fragment from the inside.

- **DO:** Validate that every combination of variant props a component exposes actually renders correctly — a "large" + "destructive" + "loading" button, for instance — rather than only testing the common combinations designers happened to think of first.

- **DON'T:** Ship a component with variant props that were only ever tested in isolation from each other. Prop combinations that were never checked together are a common source of visual bugs that only surface once a real consumer happens to combine them.

### Modal and Dialog Design

- **DO:** Reserve modal dialogs for tasks that genuinely require the user's full, blocking attention before they can continue — a critical confirmation, a focused short task — since a modal by design interrupts and blocks everything else on screen.

- **DON'T:** Default to a modal for content that doesn't actually need to block the rest of the interface (supplementary details, a non-critical settings panel). Overusing modals for low-stakes content trains users to reflexively dismiss them without reading, which defeats their purpose for the times they're genuinely needed.

- **DO:** Give every modal a clear, specific title describing its purpose, a visible and reachable close mechanism (an explicit close button, Escape key, and typically an outside-click), and focus trapped within it while open, with focus returned to a sensible location on close (see Keyboard Operability under Accessibility Foundations).

- **DON'T:** Ship a modal with no visible close affordance, relying only on an implicit or undiscoverable dismissal method. A user who can't find how to close a modal is, for that moment, stuck — a serious usability failure regardless of how well-designed the modal's content is.

- **DO:** Size a modal appropriately to its content and keep its own internal content scrollable if it could exceed the viewport height, rather than letting the modal itself overflow the screen.

- **DON'T:** Let a modal's content overflow the viewport with no internal scroll handling, pushing its own footer action buttons off-screen and unreachable, especially on smaller viewports or with dynamic content that can grow unpredictably.

### Pagination and Infinite Scroll

- **DO:** Choose between pagination and infinite scroll based on the actual task — pagination suits tasks where a user needs to return to a specific position or reference "page 3 of the results," while infinite scroll suits casual, exploratory browsing where there's no need to bookmark or return to an exact position.

- **DON'T:** Default to infinite scroll for content users frequently need to reference precisely or return to later (search results someone might want to revisit, an admin table someone needs to cite a specific row from). Infinite scroll makes "go back to exactly where I was" unreliable, and it also tends to make reaching a page's footer or a summary/total count difficult or impossible.

- **DO:** Show a clear loading indicator and handle the end-of-results state explicitly in an infinite scroll pattern, so a user can tell the difference between "still loading more" and "you've reached the end."

- **DON'T:** Let an infinite scroll list simply stop growing with no signal, leaving the user unsure whether more content exists, is still loading, or whether they've genuinely reached the end.

- **DO:** Preserve scroll position and loaded state when a user navigates away from a paginated or infinitely-scrolled list and returns (e.g., via browser back navigation), so they don't lose their place and have to re-scroll or re-paginate from the beginning.

- **DON'T:** Reset a list back to its first page or top of scroll every time a user navigates back to it after viewing a detail page. Losing scroll position on return is a small but frequently-felt frustration in any list-heavy browsing flow.

### Drag-and-Drop Interaction Design

- **DO:** Provide clear, continuous visual feedback throughout a drag operation — a visible "ghost" of the dragged item, a highlighted valid drop target, and a clear indicator of where the item will land if dropped — so the user always understands the current state of the interaction.

- **DON'T:** Implement drag-and-drop with no visual feedback beyond the browser's default drag ghost, and no indication of valid vs. invalid drop targets. Ambiguous drag feedback leaves users unsure whether their drop will succeed until after they've already released it.

- **DO:** Provide a non-drag alternative for any action achievable via drag-and-drop (a context menu "Move to…" option, up/down reorder buttons, cut/paste), since drag-and-drop is difficult or impossible for some motor-impaired users and for keyboard-only users entirely.

- **DON'T:** Make drag-and-drop the sole method for accomplishing an important task (reordering a critical list, moving an item between categories) with no keyboard- or click-based alternative. A mouse-only or touch-only interaction pattern with no equivalent excludes users who can't perform that specific gesture.

### Onboarding Tours and Coach Marks

- **DO:** Keep product tours and coach marks (contextual pointers highlighting a specific UI element) short, dismissible at any point, and focused on the small number of things a new user genuinely needs to know before their first meaningful action — not an exhaustive tour of every feature.

- **DON'T:** Build a lengthy, multi-step, un-skippable tour covering every feature in the product before letting a new user do anything. Long mandatory tours are frequently abandoned partway through or clicked through without reading, defeating their own purpose while adding friction for every single new user, including ones who didn't need the guidance.

- **DO:** Trigger contextual coach marks at the moment a feature becomes relevant to what the user is actually doing, rather than front-loading all guidance into a single onboarding sequence the user has to remember later.

- **DON'T:** Explain every feature up front during initial onboarding, expecting the user to remember guidance about a feature they won't actually use until much later. Just-in-time contextual guidance, offered when a feature first becomes relevant, is retained far better than guidance delivered before the user has any context for why it matters.

### Table and List Row Actions

- **DO:** Keep primary, frequently-needed row actions visible by default in a table or list, reserving hover-reveal or overflow-menu treatment for secondary, less-frequent actions — and ensure hover-revealed actions are still reachable via keyboard focus and on touch devices where hover doesn't exist.

- **DON'T:** Hide all row actions behind a hover-only reveal with no keyboard or touch equivalent. As with tooltips, hover-only interaction silently excludes touch and keyboard-only users from actions that a mouse user can see plainly.

- **DO:** Group related row actions into a clearly labeled overflow menu ("⋯") once there are more than roughly two or three, rather than crowding a row with many individually-visible icon buttons that compete for limited horizontal space.

- **DON'T:** Cram five or six separate icon-only action buttons into a single table row. Beyond being visually cluttered, this makes each individual action's already-ambiguous icon even harder to parse quickly, and increases the risk of touch mis-taps given how little space each one gets.

### Segmented Controls, Tabs, and Selection Patterns

- **DO:** Use a segmented control or tab set for a small number (roughly two to five) of mutually exclusive, closely-related views or filters, with the currently selected option always clearly and unambiguously indicated.

- **DON'T:** Use a segmented control or tab pattern for more options than can be comfortably scanned at a glance, or for options that aren't actually mutually exclusive. Beyond a handful of options, a dropdown, a sidebar list, or a different pattern entirely usually serves the user better than an overcrowded row of tabs.

- **DO:** Keep tab and segmented-control labels short, parallel in phrasing (all nouns, or all consistent in grammatical form), and stable — reordering or renaming tabs frequently undermines the spatial memory users build for where a given view lives.

- **DON'T:** Mix inconsistent label styles within one tab set ("Overview," "Settings," "Manage Users" — inconsistent noun vs. verb phrasing) or reorder tabs between sessions. Both undermine the quick, at-a-glance scannability tabs are meant to provide.

### Autocomplete and Combobox Design

- **DO:** Show autocomplete suggestions promptly as the user types, ranked by relevance, with the matched portion of each suggestion visually highlighted, and make the suggestion list fully operable via keyboard (arrow keys to navigate, Enter to select, Escape to dismiss).

- **DON'T:** Build an autocomplete/combobox pattern that only responds to mouse clicks on its suggestion list, with no keyboard navigation support. This is one of the more commonly under-implemented accessible interaction patterns, and the WAI-ARIA combobox authoring pattern exists specifically to provide a well-tested reference for getting it right.

- **DO:** Debounce autocomplete requests to a live backend so a fast typist doesn't trigger a flood of redundant network requests, and clearly indicate a loading state while suggestions are being fetched.

- **DON'T:** Fire a new network request on every single keystroke with no debouncing, and leave the suggestion list in a stale or blank state with no loading indicator while a request is in flight. Both degrade the perceived responsiveness of what should feel like an instantaneous interaction.

### Steppers and Multi-Step Wizards

- **DO:** Show a clear progress indicator in any multi-step flow (a numbered stepper, a progress bar) communicating both the current step and the total number of steps, so the user always knows how much of the process remains.

- **DON'T:** Drop a user into a multi-step flow with no indication of how many steps exist or how far along they are. An open-ended flow with no visible progress feels longer and more daunting than the same flow with clear progress shown, even when the actual number of steps is identical.

- **DO:** Allow users to navigate back to a previous step in a wizard to review or correct earlier input, preserving what they'd already entered on every step, both the one they're returning to and the ones ahead they'd already completed.

- **DON'T:** Make a wizard's steps strictly forward-only with no way to revisit and correct an earlier step, or clear a step's previously-entered data if the user does navigate back to it. Being unable to fix an earlier mistake without restarting the entire flow is a significant, avoidable frustration.

- **DO:** Validate each step's input before allowing progression to the next step (or at minimum, validate at the point the user tries to move forward), so errors are caught close to where they were made rather than surfacing all at once at final submission.

- **DON'T:** Defer all validation across every step to the final submission at the end of a long wizard. Discovering on the last step that something entered on step one was invalid forces the user all the way back through the flow to fix it, which is a severe, avoidable usability failure in exactly the kind of multi-step flow most likely to have real time investment behind it.

### Choosing the Right Input Type: Switches, Checkboxes, and Radio Buttons

- **DO:** Use a toggle switch for a setting that takes effect immediately with no separate save/submit step (an on/off preference), a checkbox for a selection that's part of a form to be submitted (including a single standalone yes/no choice within that form), and radio buttons for choosing exactly one option from a small visible set.

```text
GOOD mapping:
  Toggle switch  → "Enable dark mode" (applies immediately)
  Checkbox       → "I agree to the terms" (part of a form, submitted together)
  Radio buttons  → "Shipping speed: Standard / Express / Overnight" (one of few, visible options)
```

- **DON'T:** Use a toggle switch for a choice that's actually part of a larger form requiring an explicit submit action, or use checkboxes for a set of options where only one selection should ever be valid. Mismatching input type to its actual behavior (immediate vs. deferred effect, single vs. multiple selection) creates real ambiguity about what a control will actually do when interacted with.

- **DO:** Use a dropdown/select for a larger set of mutually exclusive options (roughly seven or more) where radio buttons would take up too much space, but prefer visible radio buttons over a hidden dropdown for a small set where showing all options at once aids quick comparison.

- **DON'T:** Default to a dropdown for every selection regardless of how few options it contains. Hiding two or three options behind a dropdown's extra click, when they could have been shown directly as radio buttons, adds friction with no corresponding benefit.

### Confirmation Patterns for Bulk and Irreversible Actions

- **DO:** Combine the bulk-action guidance (under Layout & Spacing / Filtering above) with the destructive-confirmation guidance (under Content Design & Microcopy) explicitly for bulk destructive actions — state the exact count and nature of what will be affected ("Delete 47 selected items? This can't be undone.") rather than a generic confirmation that doesn't reflect the bulk scope.

- **DON'T:** Show the same lightweight, generic confirmation for a bulk destructive action affecting many items as for a single-item one. The consequence scales with the count, and the confirmation's specificity and friction should scale with it too — silently applying a single-item confirmation pattern to a 500-item bulk delete under-communicates real risk.

### Search Input Component Design

- **DO:** Give a search input a clear, recognizable search icon, a visible clear/reset button once text has been entered, and immediate, obvious feedback once a query is submitted or results update, so the control's purpose and current state are always unambiguous.

- **DON'T:** Ship a search input with no clear/reset affordance, forcing a user to manually select and delete their entire query to start a new search. A one-click clear control is a small addition that removes a small but frequent friction point from one of the most-used controls in many products.

- **DO:** Decide deliberately between search-as-you-type (live, debounced results) and search-on-submit (results only after Enter or a button press) based on result cost and volatility — live search suits fast, cheap, locally-filterable data; submit-based search suits expensive, paginated, or server-heavy queries.

- **DON'T:** Default to live search-as-you-type for every search context regardless of backend cost or result stability. Live search against an expensive or slow backend query, fired on every keystroke, can create a laggy, request-flooded experience that a simple submit-based search would have avoided entirely.

### Rich Text Editors and Formatting Toolbars

- **DO:** Keep a rich text editor's toolbar limited to the formatting options genuinely relevant to the content being authored, and clearly indicate the currently active formatting state (bold/italic/list active, etc.) on the toolbar itself as the cursor moves through differently-formatted text.

- **DON'T:** Expose a full desktop-word-processor-scale formatting toolbar for a context that only ever needs basic formatting (a comment box, a short description field). An oversized toolbar for a simple field adds visual weight and choice-overload with no corresponding benefit for what the field is actually used for.

### Accordion and Expandable Section Design

- **DO:** Indicate an accordion or expandable section's current state unambiguously with a rotating chevron, a plus/minus swap, or an equivalent visual indicator, keeping that indicator's state perfectly synchronized with whether the section is actually expanded or collapsed.

- **DON'T:** Rely on the presence or absence of the revealed content alone as the only signal of an accordion's state, with no dedicated visual indicator on the trigger itself. Without a clear indicator on the header/trigger, a user scanning a list of collapsed accordion headers has no way to tell which ones (if any) are expandable at all.

- **DO:** Decide deliberately whether multiple accordion sections can be open simultaneously or whether opening one should close the others, based on whether the sections' content is meant to be compared side by side (favoring multiple-open) or read one at a time (favoring single-open, "exclusive" accordion behavior).

- **DON'T:** Leave accordion open/close behavior unspecified and inconsistent between similar accordion instances within the same product — one place allowing multiple sections open, a visually identical pattern elsewhere forcing exclusivity with no clear reason for the difference.

## Motion & Animation

### Physics-Based vs. Duration-Based Animation

- **DO:** Consider spring/physics-based animation (defined by stiffness, damping, and mass rather than a fixed duration and easing curve) for interactions that respond to continuous user input — a drag that needs to settle naturally, a dismissible sheet that should feel like it has real momentum — since spring animations naturally handle interruption and varying start velocities in a way fixed-duration easing curves don't.

```text
GOOD use of a spring: a bottom sheet the user drags partway down and
     releases — a spring animation naturally continues with the release
     velocity and settles into its resting position, whereas a fixed
     duration/easing curve would ignore how fast the user was dragging.
```

- **DON'T:** Force every animation into a fixed-duration, fixed-easing-curve model when the interaction it's responding to genuinely has variable input (drag velocity, gesture momentum). A duration-based animation that ignores the user's actual input velocity can feel disconnected from the gesture that triggered it.

- **DO:** Reserve spring-based animation for genuinely gesture-driven or physically-motivated interactions, and keep straightforward, duration/easing-based animation for simple, discrete state changes (a button's hover color, a modal's fade-in) where a spring's added complexity provides no real benefit.

- **DON'T:** Apply springy, bouncy physics-based animation indiscriminately to every UI transition, including simple discrete ones with no real physical or gestural basis. An exaggerated spring bounce on a routine state change (a checkbox toggling, a menu opening) can read as gimmicky rather than natural, precisely because there's no underlying physical interaction the bounce is actually modeling.

### Shared-Element and View Transitions

- **DO:** Use a shared-element transition (an item smoothly morphing from its position and size in a list into its expanded detail view) when navigating between two views that share a clear visual and conceptual relationship, since this class of transition is one of the most effective ways to preserve spatial context across a navigation change.

```css
/* GOOD: opts a shared element into a smooth cross-view transition
   (View Transitions API, where supported) */
.list-item-thumbnail { view-transition-name: item-thumb; }
/* the browser automatically animates between the list and detail states
   sharing this transition name */
```

- **DON'T:** Default to a hard cut or a generic fade between two views that share an obvious visual element (a thumbnail becoming a hero image, a card becoming a full page), when a shared-element transition would reinforce the connection between them at relatively low implementation cost using modern browser APIs.

- **DO:** Keep shared-element transitions reasonably quick (in the same 250–500ms range as other larger UI transitions) and gate them behind a `prefers-reduced-motion` check like any other non-essential motion.

- **DON'T:** Let a shared-element transition run long enough to feel like it's making the user wait for navigation to complete. Even a well-crafted transition overstays its welcome if its duration isn't kept in check relative to how frequently the navigation it accompanies is triggered.

Motion is the design layer most often treated as pure decoration, and that's exactly where it goes wrong. Good interface motion has a job: it explains what just changed, where something went, or what state the system is now in. When motion has no job — when it exists only because "animation feels premium" — it slows the interface down, distracts from the actual task, and, for a meaningful subset of users, can cause real physical discomfort.

### Purposeful Motion

- **DO:** Use motion to explain a state change, a spatial relationship, or a cause-and-effect link between two things on screen — a panel that slides in from the edge it will dock against, a card that expands in place into its detail view, a deleted item that visibly collapses out of a list rather than instantly vanishing.

```text
GOOD use of motion: a settings panel slides in from the right edge of the screen
  → tells the user where it came from and where it will go when dismissed

BAD use of motion: a settings panel fades in from the center with a bounce
  → decorative, communicates nothing about the panel's relationship to the screen
```

- **DON'T:** Add animation purely for perceived "delight" or polish with no explanatory function — a gratuitous bounce, an unnecessary parallax effect, a spin that serves no informational purpose. Motion without a job still costs attention and time; it just doesn't pay that cost back with any clarity gained.

- **DO:** Use motion to preserve context across a transition, so the user's mental model of "where did that go" or "where did this come from" stays intact — an element that visually morphs from its collapsed state into its expanded state, rather than one element disappearing while an unrelated-looking one appears in its place.

- **DON'T:** Hard-cut between two layouts that are conceptually connected (a list item and its detail view, a thumbnail and its full-size image) when a transition would help the user track the relationship. An abrupt cut here forces the user to re-orient and re-find their place, costing more actual time than a well-designed 200–300ms transition would have.

- **DO:** Use motion to draw attention to a change the user needs to notice but might otherwise miss — a newly added item briefly highlighting, a value that just updated animating from its old value to its new one — when that change happens outside the user's current focus.

- **DON'T:** Animate every single UI update indiscriminately, including changes squarely inside the user's current focus that need no extra attention-drawing. Motion applied to something the user is already looking directly at, doing exactly what they expect, adds only friction.

### Duration and Easing

- **DO:** Keep most UI micro-interaction durations short — roughly 100–250ms for small state changes (hover, toggle, button press) and roughly 250–500ms for larger transitions (a panel entering, a page-level transition). Durations in this range read as responsive; the interaction registers as immediate cause-and-effect rather than a separate, noticeable event.

- **DON'T:** Use durations well outside this range without a specific reason — either so fast the change is barely perceptible as motion at all, or so slow (800ms+) that a simple UI transition starts to feel sluggish and makes the interface feel like it's fighting the user's pace rather than keeping up with it.

- **DO:** Use eased, non-linear timing functions for essentially all interface motion — commonly an ease-out curve (fast start, gentle settle) for elements entering the screen, and an ease-in curve (gentle start, fast exit) for elements leaving.

```css
/* GOOD: natural, purpose-matched easing for enter vs. exit */
.panel-enter {
  animation: slide-in 240ms cubic-bezier(0.16, 1, 0.3, 1) forwards; /* ease-out */
}
.panel-exit {
  animation: slide-out 180ms cubic-bezier(0.4, 0, 1, 1) forwards;  /* ease-in */
}
```

- **DON'T:** Use linear easing (`transition-timing-function: linear`) for UI motion. Linear motion has constant velocity from start to finish, which reads as mechanical and unnatural — almost nothing in the physical world humans have intuitions about actually moves at a perfectly constant speed, so linear motion subtly signals "this is a machine doing something," not "this is responding naturally to you."

- **DO:** Match the easing curve's character to what the motion represents — a snappier, slightly overshooting curve for a playful confirmation (a like button, a small success check), a calmer, more restrained curve for structural UI changes (a modal opening, a page navigating).

- **DON'T:** Apply a single easing curve uniformly to every kind of motion in the product regardless of what it represents. Different motion types (a destructive delete vs. a delightful confirmation vs. a structural page transition) deserve some deliberate differentiation, within a shared overall motion language.

- **DO:** Treat duration and easing values as design tokens, exactly like color and spacing (`--duration-fast: 150ms`, `--ease-standard: cubic-bezier(0.4, 0, 0.2, 1)`), so motion stays consistent across the product instead of each component defining its own slightly different timing.

- **DON'T:** Let each component author hand-pick a new duration and curve because "this one felt a bit better." Inconsistent motion timing across otherwise-similar interactions is a subtle but real inconsistency, the motion equivalent of ungoverned spacing values.

### Respecting prefers-reduced-motion

- **DO:** Wrap non-essential motion — parallax effects, large sliding/scaling transitions, autoplaying decorative animation — in a check against the operating system's reduced-motion preference, and provide a reduced or static alternative that still communicates the same state change through a non-motion cue (an instant cross-fade, or no transition at all).

```css
/* GOOD: respects the OS-level accessibility preference */
.panel {
  transition: transform 240ms cubic-bezier(0.16, 1, 0.3, 1);
}

@media (prefers-reduced-motion: reduce) {
  .panel {
    transition: opacity 120ms linear;
    transform: none;
  }
}
```

- **DON'T:** Ignore the `prefers-reduced-motion` operating system setting. For users with vestibular disorders, certain kinds of motion-triggered migraines, or motion sensitivity, large-scale animated transitions can cause genuine physical discomfort — nausea, dizziness, disorientation — not merely a stylistic annoyance, so this preference is a functional accessibility requirement, not an optional nicety.

- **DO:** Keep small, purely functional feedback motion (a subtle color change on press, a short opacity fade) even in the reduced-motion path, since the goal is removing large or vestibular-triggering motion, not removing all feedback entirely — a completely static, feedback-free interface is its own usability regression.

- **DON'T:** Interpret "reduced motion" as "zero transitions of any kind" and strip out even essential state-change feedback. Reduced-motion users still benefit from knowing that a click registered; they specifically need to avoid large spatial movement, parallax, and vestibular-triggering effects.

### Avoiding Motion That Blocks the User

- **DO:** Let users interrupt, skip, or work through an animation rather than forcing them to wait for it to finish before they can proceed — a transition should be able to be interrupted by a new user action and gracefully redirect, not queue up and force the user to wait.

- **DON'T:** Force users to sit through a multi-second, unskippable animated sequence (a splash screen, an elaborate page transition) before they can interact with the interface. Every second of forced, non-interruptible animation is a second directly subtracted from the user's ability to accomplish their actual task.

- **DO:** Keep an element interactive as soon as it is functionally ready, even if its own entrance animation hasn't fully completed — don't gate input on an animation's `animationend` event unless there's a specific reason interaction genuinely can't happen yet.

- **DON'T:** Disable a button or control for the full duration of its own click-triggered animation when the underlying action it represents has already completed. If the save actually finished in 50ms but the success animation takes 400ms, the button shouldn't stay disabled for the full 400ms "for consistency" — that's motion dictating pacing to the user instead of the other way around.

- **DO:** Design animated transitions to be genuinely interruptible — if a user clicks to open a panel and then immediately clicks to close it, the interface should smoothly reverse or blend into the new target state, not finish playing the first animation before starting the second.

- **DON'T:** Queue conflicting animations so that a rapid double-action (open then immediately close) results in the interface finishing the first animation before it even begins reacting to the second. This makes the interface feel like it's lagging behind the user's actual input.

### Micro-Interactions and Feedback

- **DO:** Give every direct manipulation — a drag, a toggle, a swipe, a press-and-hold — continuous, real-time visual feedback that tracks the input itself, not just a discrete before/after state change once the gesture completes.

- **DON'T:** Wait until a drag or swipe gesture fully completes to show any visual response. Feedback that only appears after the gesture is done feels disconnected from the action itself — direct manipulation should feel like the interface is responding to the user's hand in real time, not just acknowledging the result afterward.

- **DO:** Use small, quick micro-interactions (a checkbox's check-mark drawing in, a like button's brief scale-and-color pulse) to confirm that a lightweight action registered, scaled appropriately to how minor the action is.

- **DON'T:** Give a trivial, frequent action (checking a checkbox, toggling a filter) an elaborate, lengthy animation. A heavy animation on a high-frequency interaction becomes actively annoying the tenth time a user triggers it in one session, even if it looked charming the first time.

### Animation Performance

- **DO:** Animate GPU-accelerated CSS properties — `transform` and `opacity` — wherever possible, since browsers can composite these without triggering a full layout/paint recalculation, keeping animations smooth even on lower-powered devices.

```css
/* GOOD: animates transform/opacity — cheap, GPU-composited */
.card { transition: transform 200ms ease-out, opacity 200ms ease-out; }
.card:hover { transform: translateY(-4px); opacity: 0.95; }

/* AVOID: animating layout-triggering properties causes jank */
.card-bad { transition: top 200ms ease-out, width 200ms ease-out; }
```

- **DON'T:** Animate layout-triggering properties (`width`, `height`, `top`, `left`, `margin`) for anything performance-sensitive or frequently triggered. These properties force the browser to recompute layout on every animation frame, which is a common, avoidable cause of visibly janky, stuttering motion, especially on lower-end mobile hardware.

- **DO:** Test animation performance on genuinely lower-powered devices, not only on a high-end development machine. An animation that looks buttery smooth on a developer's current-generation laptop or phone can drop frames badly on the mid-range or older hardware a meaningful share of real users have.

- **DON'T:** Assume an animation's performance profile is fine because it looks smooth in the browser's design/dev environment. Development machines are frequently far more powerful than the median real-world device, and dropped frames are one of the fastest ways an animation goes from "delightful" to "the app feels laggy."

### Page and Screen Transitions

- **DO:** Use a consistent, shared transition pattern for navigating between equivalent screen types (e.g., every "detail view" opens with the same slide/fade pattern), so the motion language itself becomes a learnable, predictable part of how the product's navigation feels.

- **DON'T:** Give each screen-to-screen transition a bespoke, one-off animation unrelated to how any other transition in the product behaves. Inconsistent transition styles make navigation feel unpredictable in a subtle way, even if each individual transition looks fine on its own.

- **DO:** Use directional transitions that reflect the actual navigational relationship — forward navigation sliding content in from one consistent direction, backward navigation reversing it — so the motion itself reinforces the user's sense of moving forward or back through a hierarchy.

- **DON'T:** Use the same generic fade or cross-dissolve for both forward and backward navigation with no directional distinction. A transition with no directional meaning misses a free opportunity to reinforce the user's spatial understanding of where they are in the navigation flow.

- **DO:** Keep page-level transition duration proportional to how much visual change is occurring — a full-screen navigation change can reasonably take a bit longer (around 300–400ms) than a small in-place component update (100–200ms).

- **DON'T:** Apply an identical, flat transition duration regardless of whether the change is a small in-place update or a full-screen navigation. A duration tuned for a small change feels abrupt at full-screen scale; a duration tuned for full-screen change feels sluggish applied to something small.

### Sound and Haptic Feedback

- **DO:** Use haptic feedback (on platforms/devices that support it) to reinforce a physical, tactile sense of an interaction's outcome — a light tap for a successful toggle, a stronger pulse for an error or a significant confirmation — matched proportionally to the significance of the action.

- **DON'T:** Apply the same intensity of haptic feedback to every interaction regardless of significance, or trigger haptics so frequently (on every minor scroll tick, for instance) that it becomes a constant, fatiguing background buzz rather than a meaningful signal.

- **DO:** Treat sound effects as optional, user-controllable, and off by default in most professional/productivity contexts, reserving audible feedback for contexts where it adds genuine value (a messaging app's new-message chime) and always respecting a system-level or in-app mute setting.

- **DON'T:** Play audio feedback for routine UI interactions with no way to disable it. Unexpected or unwanted sound is one of the fastest ways to make a product feel intrusive, particularly in shared or quiet physical environments where audible feedback is actively unwelcome.

### Choreography and Staggered Animation

- **DO:** Stagger the entrance of a group of related elements (a list of cards appearing in sequence, a set of chart bars growing one after another) with a short, consistent delay between each, when doing so helps the eye track the group forming rather than register a single jarring simultaneous change.

```css
/* GOOD: a small, consistent stagger helps the eye track a list appearing */
.list-item { animation: fade-up 200ms ease-out both; }
.list-item:nth-child(1) { animation-delay: 0ms; }
.list-item:nth-child(2) { animation-delay: 40ms; }
.list-item:nth-child(3) { animation-delay: 80ms; }
```

- **DON'T:** Stagger a large number of elements with a long per-item delay, making the user wait an unnecessarily long time for the full group to finish appearing. A stagger should read as a quick, cohesive ripple, not a slow, item-by-item reveal that delays the interface becoming fully usable.

- **DO:** Keep staggered animations subtle and brief enough that they don't become a repeated annoyance for a group of elements the user will see render frequently (a frequently-reloaded list, for instance) — reserve more noticeable staggering for genuinely first-time or infrequent reveals.

- **DON'T:** Apply an eye-catching staggered entrance animation to content that reloads or re-renders frequently during normal use. What reads as a delightful flourish the first time becomes a tedious, attention-grabbing distraction by the tenth time a frequent user sees it.

### Continuity Between Loading and Loaded States

- **DO:** Design the transition from a skeleton or loading placeholder into the final loaded content as a smooth, continuous handoff — a brief cross-fade, or content that grows into place from the skeleton's shape — rather than an abrupt, jarring swap.

- **DON'T:** Let loaded content simply replace a skeleton placeholder instantly with no transition, especially when the loaded content's actual dimensions differ noticeably from the skeleton's estimated shape, causing a visible layout jump on top of the abrupt content swap.

- **DO:** Size skeleton placeholders as closely as practical to the loaded content's actual eventual dimensions, so the loading-to-loaded transition doesn't itself introduce a layout shift, independent of whether an animated transition is used.

- **DON'T:** Use a generically-sized skeleton that differs substantially from the real content's eventual size. Beyond undermining the "smooth handoff" goal, a size mismatch here is a direct, avoidable contributor to cumulative layout shift.

### Parallax and Scroll-Triggered Effects

- **DO:** Use scroll-triggered animation and parallax sparingly and only where it genuinely reinforces a narrative or spatial idea (a marketing page building a sense of depth or progression), and always gate it behind a `prefers-reduced-motion` check.

- **DON'T:** Apply parallax scrolling broadly across content-heavy or task-focused interfaces. Parallax effects are among the motion patterns most likely to trigger discomfort for vestibular-sensitive users, and in a task-focused context they add visual noise without aiding comprehension of the actual content.

- **DO:** Keep scroll-triggered reveal animations subtle and quick, so they don't force the user to wait for content to "catch up" with their scroll position — the content should feel like it's already there, gently emphasized, not like it's racing to appear.

- **DON'T:** Delay content's full visibility behind a slow scroll-triggered reveal animation that lags noticeably behind the user's actual scroll position. A user scrolling purposefully to find something shouldn't have to pause and wait for a decorative reveal animation to finish catching up.

### Loading Progress Indicators: Determinate vs. Indeterminate

- **DO:** Use a determinate progress indicator (an actual percentage or fraction) whenever real progress is knowable — a file upload, a multi-step process with known step count — since a determinate indicator lets the user gauge how much longer to wait.

```text
GOOD (determinate): a progress bar at 62%, with a numeric percentage, for a
     file upload where the actual bytes-transferred fraction is known.
GOOD (indeterminate): a looping spinner or pulse for a server request whose
     duration genuinely can't be predicted in advance.
```

- **DON'T:** Use an indeterminate spinner for a process whose actual progress is knowable and could be shown as a real percentage. An indeterminate spinner conveys only "something is happening," which is strictly less useful than a real progress indication whenever the latter is available.

- **DO:** Use an indeterminate indicator (a spinner, a looping pulse) honestly, only when actual progress genuinely can't be estimated, rather than fabricating a progress bar that doesn't correspond to real progress.

- **DON'T:** Fake a determinate progress bar with an animation that isn't tied to real progress (a bar that always animates to 90% quickly then stalls, unrelated to the actual operation's state). A fabricated progress indicator that doesn't correspond to reality erodes trust once users notice the mismatch — an honest indeterminate spinner is preferable to a dishonest determinate one.

### Cursor and Hover Feedback on Non-Touch Devices

- **DO:** Use cursor changes (`pointer` for clickable, `text` for editable text, `not-allowed` for disabled, `grab`/`grabbing` for draggable) purposefully on pointer-based devices to reinforce what an element under the cursor actually does, supplementing — not replacing — the other affordance cues covered under Components.

- **DON'T:** Leave the default cursor unchanged over custom interactive elements (a styled `<div>`-based button, a custom slider handle) on desktop, where the default arrow cursor gives no indication that the element beneath it is actually interactive.

- **DO:** Keep hover-triggered visual feedback (a color shift, a subtle elevation change) quick and subtle for standard interactive elements, reserving more elaborate hover effects for contexts where a richer preview genuinely helps (a media thumbnail previewing on hover, for instance).

- **DON'T:** Apply an elaborate, slow hover animation to every single interactive element on a dense screen. On a page with many interactive elements in close proximity, an exaggerated hover effect on each one becomes visually distracting as the cursor naturally passes over several of them while navigating toward an intended target.

### Animating Numbers and Counters

- **DO:** Animate a numeric value's transition from its old value to its new one (a counter ticking up, a total recalculating) when the change happens live in front of the user and the specific delta is meaningful to notice, keeping the animation brief (a few hundred milliseconds) regardless of how large the numeric jump is.

- **DON'T:** Animate a number's transition when it's simply being displayed for the first time on page load, with no prior value to transition from. Animating a counter "counting up from zero" purely for visual flair on initial load, with no real prior state, is decoration rather than the meaningful before/after signal number-transition animation is otherwise good for.

- **DO:** Keep animated number transitions duration-capped regardless of the size of the numeric change, so a jump from 10 to 10,000 doesn't animate for an absurdly long time just because the raw numeric difference is large.

- **DON'T:** Let a number-counting animation's duration scale linearly and unboundedly with the size of the value change. Cap the duration and let the animation's easing (not raw elapsed time proportional to magnitude) communicate the size of the change.

### Exit Animations for Removed Items

- **DO:** Animate an item's removal from a list (a card sliding out and the gap closing smoothly, a fade-and-collapse) rather than having it disappear instantly with the rest of the list snapping into its new layout with no transition.

- **DON'T:** Let a deleted item vanish instantly, causing every item below it to jump immediately into the newly-vacated space with no transition. An abrupt list-reflow snap is jarring, and if the deletion happened near the user's current scroll position or cursor, it can also cause a moment of disorientation about what just happened.

- **DO:** Keep exit animations brief and let the surrounding layout's collapse animate in tandem with the removed item's own fade/slide, so the two motions read as one coherent event rather than two disconnected ones.

- **DON'T:** Animate only the removed item's own disappearance while leaving the surrounding list's reflow to snap instantly once the item is gone, or vice versa. Half-animated transitions — one part smooth, one part abrupt — often look more broken than no animation at all, because the inconsistency itself draws attention.

## Accessibility Foundations

### Alternative Input Methods

- **DO:** Build interactions so they work correctly through the full range of standard input events (click, keyboard activation) rather than depending on assumptions specific to a mouse or touchscreen, since a well-built interaction that correctly responds to a semantic `click` event and keyboard activation generally also works correctly for switch control, voice control, and other assistive input methods that translate into those same standard events.

- **DON'T:** Build custom interactions around low-level, device-specific events (raw `mousedown`/`mouseup` pairs, touch-specific gesture events) with no accommodation for how alternative input methods actually interact with a page. Assistive technologies like switch control and voice control largely work by triggering standard activation events on focusable, semantically correct elements — exactly the same foundation that keyboard accessibility (covered above) depends on, which is one more reason semantic, keyboard-operable markup pays off broadly.

- **DO:** Keep interactive elements a reasonable, unambiguous size and give them clear, distinguishable accessible names (see Form Labels and Icons Paired with Labels, above), since voice-control users often activate a control by speaking its visible name, and switch-control users scan through focusable elements sequentially, both of which benefit from the same clarity that helps every other user.

- **DON'T:** Assume "keyboard accessible" and "fully accessible to alternative input methods" are automatically the same thing without any further consideration. They overlap heavily and the same underlying practices (semantic markup, clear accessible names, standard event handling) tend to serve both, but it's worth explicitly considering voice and switch control specifically rather than assuming keyboard testing alone has covered every input modality.

### Avoiding Unexpected Context Changes

- **DO:** Trigger context changes (a navigation, a form submission, opening a new window) only from an explicit user action — activating a button or a link — never automatically as a side effect of merely receiving focus or of a value changing, per WCAG's on-focus and on-input success criteria.

- **DON'T:** Navigate away, submit a form, or open a new window automatically the moment a field receives focus or a dropdown's selection changes, with no separate confirming action. An unexpected context change triggered by simply tabbing through a form is disorienting for any user and can be genuinely difficult to recover from for a screen reader or keyboard-only user who wasn't expecting the page to change out from under them.

- **DO:** Warn the user before a context change that they didn't directly and explicitly request but that the interface needs to make (a session timeout redirect, for instance), giving them the chance to prepare for or prevent it where feasible.

- **DON'T:** Silently redirect, log a user out, or otherwise change context with no warning when the trigger wasn't the user's own direct, explicit action. An unannounced context change breaks the user's mental model of what caused it and can cause real confusion about what just happened and why.

Accessibility is not a separate feature bolted onto a finished design — it is a property of the design itself, in the same way that a building either has a usable entrance for someone in a wheelchair or it doesn't. A product that fails WCAG 2.1 AA basics isn't a product with a missing "accessibility feature"; it's a product a meaningful number of real users cannot use to complete real tasks. This section covers the foundational, high-leverage practices — the ones that prevent the large majority of accessibility failures when applied consistently from the start.

### Semantic Structure

- **DO:** Build interfaces from real semantic HTML elements — `<button>` for actions, `<a>` for navigation, `<nav>`, `<header>`, `<main>`, `<footer>` for page regions, `<ul>`/`<ol>`/`<li>` for lists — so structure and interactivity are programmatically discoverable by assistive technology without any extra work.

```html
<!-- BAD: a div with a click handler has no semantic meaning to a screen reader -->
<div class="button" onclick="submit()">Submit</div>

<!-- GOOD: a real button is keyboard-operable and announced correctly by default -->
<button type="submit">Submit</button>
```

- **DON'T:** Build interactive controls out of bare `<div>` or `<span>` elements with a click handler and no semantic role. A screen reader has no way to know a plain `<div>` is meant to be a button — it announces nothing, and it isn't part of the keyboard tab order unless extensive ARIA and manual keyboard handling is added to reconstruct behavior that a native `<button>` already provides for free.

- **DO:** Maintain a single, logical heading hierarchy per page (`h1` → `h2` → `h3`, without skipping levels) that reflects actual document structure, since screen reader users frequently navigate a page by jumping between headings and rely on the hierarchy to understand how content is organized.

- **DON'T:** Choose a heading level purely because of its default visual size ("this text needs to be big, so I'll make it an `h2`" regardless of where it actually sits in the document outline). Using heading levels for styling rather than structure breaks heading-based navigation for screen reader users, who land on a heading level that misrepresents the page's actual organization.

- **DO:** Use landmark regions (`<nav>`, `<main>`, `<aside>`, `<header>`, `<footer>`, or their ARIA `role` equivalents) so screen reader users can jump directly to a page's major regions instead of tabbing or reading through everything linearly from the top.

- **DON'T:** Wrap an entire page in generic `<div>` containers with no landmark structure at all. Without landmarks, a screen reader user's only way to reach the main content is to tab or read through every single element that precedes it, every single time they load the page.

### Alt Text for Images

- **DO:** Write alt text that describes the image's purpose and content in the context it appears — what information or function the image conveys to a sighted user, phrased concisely enough to be quickly understood when read aloud.

```html
<!-- BAD: meaningless or missing alt text -->
<img src="chart-q3.png" alt="image">
<img src="chart-q3.png">

<!-- GOOD: alt text conveys the actual information the image provides -->
<img src="chart-q3.png" alt="Q3 revenue grew 18% year-over-year, reaching $4.2M">
```

- **DON'T:** Leave `alt="image"`, `alt="photo"`, a filename, or an entirely empty/missing `alt` attribute on an image that conveys real content or meaning. Generic or missing alt text gives a screen reader user no usable information about what they're missing, which is functionally equivalent to the image not existing for them at all.

- **DO:** Use an explicitly empty `alt=""` for purely decorative images — background flourishes, spacer graphics, icons that are already redundant with adjacent visible text — so screen readers correctly skip over them instead of announcing meaningless filler.

- **DON'T:** Force screen reader users to sit through an announcement like "decorative swirl graphic" or "background pattern image" for every purely ornamental image on a page. An empty `alt=""` (not a missing attribute — an explicitly empty one) is the correct, deliberate way to mark an image as decorative.

- **DO:** Provide a text alternative for complex images that convey substantial information — charts, infographics, diagrams — either as detailed alt text, adjacent visible text, or a linked long description, since a single short `alt` attribute often can't capture everything a complex chart communicates.

- **DON'T:** Rely on a single generic `alt` attribute as the sole accessible description of a genuinely complex data visualization. If the chart is meaningful enough to include, its data should also be available in an accessible form — a data table, a detailed caption, or a long-description link — not compressed into one short alt string that can't represent it.

### Keyboard Operability

- **DO:** Ensure every interactive element on a page is reachable and fully operable using the keyboard alone — Tab/Shift+Tab to move between controls, Enter or Space to activate, Escape to dismiss overlays, and arrow keys where a native pattern expects them (menus, tabs, radio groups).

- **DON'T:** Ship a custom control — a dropdown, a date picker, a drag-and-drop reorder list — that only responds to mouse events. Any interactive element that a keyboard-only user cannot reach or operate is, for that user, functionally absent from the interface, regardless of how it looks.

- **DO:** Maintain a logical, predictable tab order that follows the visual and reading order of the page, so pressing Tab repeatedly moves focus in the sequence a sighted user would naturally scan.

- **DON'T:** Manipulate `tabindex` values (especially positive ones) in a way that creates an unpredictable, scrambled tab sequence jumping around the page out of visual order. A tab order that doesn't match visual order is disorienting and error-prone for keyboard users trying to track where focus currently is.

- **DO:** Ensure focus never gets permanently trapped somewhere the user can't escape from with the keyboard, except where a trap is intentional and correct — a modal dialog should trap focus within itself while open, but must release it back to a sensible location (typically the element that opened it) when closed.

```javascript
// GOOD: a modal traps focus while open and restores it on close
function openModal(modalEl, triggerEl) {
  const focusable = modalEl.querySelectorAll('button, [href], input, [tabindex]:not([tabindex="-1"])');
  focusable[0]?.focus();
  modalEl.dataset.returnFocusTo = triggerEl.id;
  // Tab/Shift+Tab handlers cycle within `focusable`; Escape calls closeModal()
}
function closeModal(modalEl) {
  document.getElementById(modalEl.dataset.returnFocusTo)?.focus();
}
```

- **DON'T:** Build a modal, dropdown, or overlay with no focus management at all — leaving focus wherever it happened to be when the overlay opened (often still on a now-hidden trigger button, or worse, lost entirely to the document body), which strands keyboard users with no clear sense of where they are.

- **DO:** Make Escape a universal, reliable way to dismiss any transient overlay — modal, dropdown, tooltip, popover — returning focus to a sensible location afterward.

- **DON'T:** Implement dismissal only via a mouse-clickable close button or an outside-click handler, with no keyboard equivalent. A keyboard user with no way to close a modal is stuck inside it.

### Visible Focus Indicators

- **DO:** Keep a clearly visible focus indicator (an outline, a ring, a background change) on every focusable element, distinct enough from the default state to be unambiguous at a glance, so keyboard users always know exactly which element currently has focus.

```css
/* GOOD: a strong, visible focus ring for keyboard users only, via :focus-visible */
:focus-visible {
  outline: 2px solid var(--color-primary-600);
  outline-offset: 2px;
}
/* Mouse clicks don't trigger :focus-visible in most browsers' default heuristic,
   so this avoids showing the ring on every casual mouse click */
```

- **DON'T:** Apply `outline: none` (or an equivalent that removes the browser's default focus styling) globally, without providing an equally visible custom replacement. Removing focus styles with nothing to replace them is one of the single most common, most damaging accessibility regressions in web design — it makes an otherwise keyboard-operable interface effectively unusable for anyone navigating by keyboard, because they lose all ability to see where they are.

- **DO:** Use the `:focus-visible` pseudo-class (rather than plain `:focus`) to show a strong focus ring specifically for keyboard and other non-pointer interactions, while avoiding an unwanted ring flash on every incidental mouse click — this gives both mouse and keyboard users the experience each of them actually wants.

- **DON'T:** Suppress focus styles because a designer finds the default ring visually unappealing, without designing a deliberate, equally-visible custom alternative first. "It doesn't match the aesthetic" is a design problem to solve with a better-looking focus style, not a reason to remove the accessibility signal entirely.

- **DO:** Ensure the focus indicator itself meets a minimum 3:1 contrast ratio against both the element it outlines and the surrounding background, per WCAG's non-text contrast requirement, so the ring itself is actually perceivable.

- **DON'T:** Use a focus ring color so close to the background or the element's own color that it's barely distinguishable. A focus indicator that technically exists in the DOM but isn't visually perceptible provides no real benefit to the user it's meant to serve.

### ARIA Used Correctly, Not as a Crutch

- **DO:** Reach for ARIA attributes only when a native HTML element can't already express the needed semantics — a custom combobox, a live region announcing dynamic updates, a custom-styled tab panel set. Native HTML elements come with correct semantics, keyboard behavior, and assistive-technology support built in, for free; ARIA has to reconstruct all of that manually and correctly, which is easy to get subtly wrong.

- **DON'T:** Layer ARIA roles and attributes onto elements that already have correct native semantics — adding `role="button"` to an actual `<button>`, or `aria-label` that duplicates visible text unnecessarily. Redundant or conflicting ARIA can override or confuse the correct native behavior rather than reinforcing it — the well-known guiding principle here is that no ARIA is better than bad ARIA.

- **DO:** Keep every ARIA state attribute (`aria-expanded`, `aria-selected`, `aria-checked`, `aria-current`, `aria-disabled`) accurately synchronized with the component's actual current state at every point, updating it programmatically alongside every state change.

```html
<!-- GOOD: aria-expanded is kept in sync with actual open/closed state -->
<button aria-expanded="false" aria-controls="menu-1" onclick="toggleMenu(this)">
  Options
</button>
<ul id="menu-1" hidden>...</ul>
```

- **DON'T:** Set an ARIA state attribute once at initial render and never update it as the component's actual state changes. A menu button stuck announcing `aria-expanded="false"` after the menu has visibly opened actively misinforms screen reader users about the interface's real state — worse than providing no state information at all.

- **DO:** Test any custom ARIA pattern against the WAI-ARIA Authoring Practices reference patterns (for comboboxes, tabs, menus, dialogs, etc.) rather than inventing a bespoke ARIA structure, since these documented patterns encode the specific role/state/keyboard combinations that actually work reliably across real screen readers.

- **DON'T:** Invent a custom combination of ARIA roles and properties without validating it against real assistive technology. ARIA support varies meaningfully across screen reader and browser combinations, and a pattern that seems correct by reading the spec can still fail in practice — testing against an established reference pattern avoids reinventing already-solved, already-tested problems.

### Form Labels

- **DO:** Associate every form input with an explicit, programmatically-linked label — `<label for="id">` matched to the input's `id`, or `aria-label`/`aria-labelledby` for cases where a visible label genuinely can't be shown (a search icon-only field, for instance).

```html
<!-- GOOD: explicit programmatic association between label and input -->
<label for="search-query">Search products</label>
<input id="search-query" type="search">
```

- **DON'T:** Rely on visual proximity alone — placing text next to an input with no `for`/`id` relationship — as the input's accessible name. Visual proximity communicates the association to a sighted user glancing at the layout, but a screen reader has no way to infer that relationship without an explicit programmatic link.

- **DO:** Mark required fields in a way that's conveyed to assistive technology (the `required` attribute, or `aria-required="true"`), not solely through a visual marker like a red asterisk.

- **DON'T:** Mark a field as required using only a color or symbol with no text or attribute equivalent. A required-field marker that's purely visual is invisible to a screen reader user, who may submit the form only to be told — often ambiguously — that something is missing.

- **DO:** Associate helper text and validation error messages with their field via `aria-describedby`, so a screen reader announces the relevant instructions or error alongside the field itself, not as disconnected text elsewhere on the page.

```html
<!-- GOOD: error message is programmatically tied to its field -->
<label for="pwd">Password</label>
<input id="pwd" type="password" aria-describedby="pwd-error" aria-invalid="true">
<span id="pwd-error">Password must be at least 8 characters</span>
```

- **DON'T:** Render an error message visually next to a field with no `aria-describedby` link and no `aria-invalid` state. A sighted user sees the red text right next to the field; a screen reader user navigating that same field may hear nothing about the error at all unless the association is made explicit.

### Sufficient Contrast and Resizable Text

- **DO:** Ensure every text/background pairing meets WCAG AA contrast minimums (4.5:1 body, 3:1 large text) as described under Color, above, and additionally verify the interface remains fully usable when the user zooms browser text up to 200% or increases their OS-level text size setting.

- **DON'T:** Build layouts with fixed-height containers and `overflow: hidden` around text content. When a user increases their text size, fixed-height clipping truncates or hides content that was fully visible at the default size — a layout should accommodate larger text gracefully, typically by growing the container or allowing internal scroll, not by silently cutting content off.

- **DO:** Use relative units (`rem`, `em`) for font sizes throughout the interface so the entire type scale responds correctly when a user adjusts their browser or OS default font size preference.

- **DON'T:** Hardcode font sizes in fixed pixel values that ignore the user's font-size preference entirely. A user who has deliberately increased their default text size for low vision gets no benefit from that setting inside a product built entirely on fixed-pixel type.

- **DO:** Verify that reflow — not just scaling — works correctly at 400% zoom (WCAG 2.1's 1.4.10 Reflow success criterion), meaning content restructures into a single column without requiring horizontal scrolling for anything other than genuinely two-dimensional content like data tables, maps, or images.

- **DON'T:** Assume a responsive layout that works well at typical mobile viewport widths automatically also works correctly at 400% desktop zoom. These are related but distinct scenarios, and a layout can pass one while failing the other — test both explicitly.

### Designing for Assistive Technology from the Start

- **DO:** Test with an actual screen reader (VoiceOver on macOS/iOS, NVDA or JAWS on Windows) and with keyboard-only navigation during the design and development process itself, not only as a pre-launch audit step performed by a separate team.

- **DON'T:** Treat accessibility exclusively as a final QA checklist item, run once shortly before launch, disconnected from the actual design and build process. Retrofitting accessibility into a finished design is dramatically more expensive and less effective than designing with it in mind from the first wireframe — many accessibility failures are structural decisions (a `<div>` used instead of a `<button>`, a missing landmark structure) that are cheap to get right the first time and expensive to unwind later.

- **DO:** Include specific, concrete accessibility acceptance criteria in a component's design spec and development handoff from the very start — expected focus behavior, required ARIA attributes if any, keyboard interaction map, and minimum contrast values — treated with the same rigor as functional requirements.

- **DON'T:** Leave "make it accessible" as a vague, unowned follow-up task with no specific criteria attached. A requirement with no concrete, testable definition of done tends to get silently deprioritized against requirements that do have one.

- **DO:** Involve people who actually use assistive technology day-to-day in usability testing when possible, since lived experience surfaces friction points that a sighted, non-AT-using tester following a checklist will reliably miss, however thorough the checklist is.

- **DON'T:** Rely solely on automated accessibility scanners (axe, Lighthouse, WAVE) as the complete accessibility validation process. Automated tools reliably catch a meaningful subset of issues — missing alt text, insufficient contrast, missing form labels — but structural and interaction-level problems (a confusing tab order, a focus trap that doesn't release correctly, a screen reader announcement that's technically present but doesn't make sense read aloud) generally require actual manual or assisted-technology testing to catch.

### Motion, Vestibular Safety, and Flashing Content

- **DO:** Avoid content that flashes more than three times per second, and keep any large-area flashing effect well below established safety thresholds (WCAG 2.3.1). Rapidly flashing content is a known seizure trigger for people with photosensitive epilepsy, and this is one of the few accessibility rules with a genuine, immediate physical-safety dimension rather than a purely usability one.

- **DON'T:** Ship auto-playing strobing effects, rapid flash transitions, or high-contrast rapid-flicker loading animations without checking them against flash-safety thresholds. This is a rare mistake but a serious one, and it is entirely avoidable with a basic check before shipping any rapidly repeating visual effect.

- **DO:** Provide a way to pause, stop, or hide any content that moves, blinks, scrolls, or auto-updates for more than five seconds (WCAG 2.2.2) — auto-advancing carousels, auto-playing background video, live-updating tickers — since continuously moving content is both distracting and can be genuinely disorienting for some users.

- **DON'T:** Ship an auto-advancing carousel or auto-playing looping video with no visible pause control. Users who need more time to read a slide, or who find continuous motion in their peripheral vision distracting or disorienting, have no way to stop it.

### Language and Internationalization Basics

- **DO:** Set the correct `lang` attribute on the HTML document (and on any inline content in a different language) so screen readers use the correct pronunciation rules and translation tools correctly identify the content's language.

- **DON'T:** Omit the `lang` attribute, or leave it mismatched with the page's actual content language. A screen reader given the wrong language hint will mispronounce content in ways that range from mildly confusing to completely unintelligible.

- **DO:** Design layouts that accommodate right-to-left (RTL) languages by using logical CSS properties (`margin-inline-start` rather than `margin-left`, `padding-inline-end` rather than `padding-right`) so the layout mirrors correctly when the document direction flips.

- **DON'T:** Hardcode physical-direction properties (`left`/`right`) throughout a codebase that is expected to support RTL languages. Physical properties require manually overriding every single directional value for RTL, which is exactly the kind of exhaustive, error-prone rework that logical properties exist to eliminate.

### Cognitive Accessibility

- **DO:** Write in plain, direct language, break complex processes into clear discrete steps, and keep interaction patterns consistent throughout the product, since cognitive accessibility — supporting users with learning disabilities, memory or attention differences, or simply anyone under stress or time pressure — benefits enormously from reduced complexity and predictability.

- **DON'T:** Assume cognitive accessibility is a niche concern affecting only a small, separate population. Clear language, consistent patterns, and reduced unnecessary complexity measurably help a very wide range of users, including fully able-bodied users who are simply tired, distracted, or unfamiliar with the domain.

- **DO:** Avoid unnecessary time limits on tasks (form completion, reading content, session timeouts) wherever the task doesn't genuinely require one, and when a time limit is unavoidable, warn the user before it expires and offer a way to extend it.

- **DON'T:** Impose an aggressive, unexplained session or form timeout with no warning and no way to extend it. Losing a partially completed task to a silent timeout is frustrating for any user and can be a genuine barrier for users who need more time to read, process, or complete input.

- **DO:** Keep interaction patterns predictable and consistent across the product — the same gesture, the same control, should do the same thing everywhere it appears — so a user can build and rely on a stable mental model instead of re-learning behavior on each new screen.

- **DON'T:** Let the same visual control behave differently in different contexts with no clear signal of the difference (a "swipe to delete" pattern in one list, "swipe to archive" in a visually identical list elsewhere). Inconsistent behavior behind consistent-looking controls is disorienting and erodes the trust a consistent design system is supposed to build.

### Captions, Transcripts, and Media Accessibility

- **DO:** Provide accurate closed captions for all video content with spoken dialogue or important audio information, and a transcript for audio-only content (podcasts, voice recordings), so the content is accessible to users who are deaf or hard of hearing, and usable in sound-off environments by everyone else.

- **DON'T:** Ship video content with no captions, or with auto-generated captions left uncorrected and riddled with errors. Inaccurate captions can be actively misleading, not merely incomplete, particularly for technical or domain-specific terminology an automated system is likely to mis-transcribe.

- **DO:** Provide audio descriptions (a narrated description of important visual information) for video content where meaning depends on visual information not conveyed through dialogue alone, so blind or low-vision users don't miss critical context.

- **DON'T:** Assume captions alone fully cover accessibility for video content that relies heavily on visual information the dialogue doesn't describe (an on-screen demonstration, a chart shown without being verbally explained). Captions address audio access; visual-only information needs its own accessible equivalent.

### Accessibility Testing Tools and Workflow

- **DO:** Integrate automated accessibility scanning (axe-core, Lighthouse, or equivalent) into the CI/CD pipeline so basic, catchable issues — missing alt text, insufficient contrast, missing form labels, invalid ARIA usage — are flagged automatically before code merges, not discovered later in a manual audit.

```text
GOOD workflow:
  PR opened → automated a11y scan runs in CI → contrast/label/ARIA
  issues block merge automatically, alongside unit test failures
```

- **DON'T:** Treat accessibility scanning as a manual, occasional, pre-launch-only activity performed by a separate team long after the code has already shipped internally. By the time a manual audit catches an issue that automated CI could have caught on the original pull request, it's typically far more expensive to fix and more likely to have already reached production.

- **DO:** Supplement automated scanning with periodic manual testing — actual keyboard-only navigation, actual screen reader testing — since automated tools reliably catch only a subset (commonly estimated around 30-50%) of real accessibility issues; the rest require a human evaluating actual usability with assistive technology.

- **DON'T:** Treat a clean automated scan result as proof the product is fully accessible. A passing axe or Lighthouse score confirms the absence of a specific, mechanically-detectable set of issues — it says nothing about whether the actual experience of using the product with a screen reader or keyboard alone is coherent and usable.

### Accessible Data Visualization

- **DO:** Provide an accessible alternative to any meaningful chart or graph — a data table, a text summary of the key takeaway, or both — so the information a chart conveys is available to users who can't perceive the visualization itself (screen reader users, users with certain visual impairments) rather than being locked exclusively inside a canvas or image rendering.

- **DON'T:** Render a chart's data exclusively as an unlabeled `<canvas>` or a flat image with no accessible data equivalent. A chart that exists only as pixels conveys nothing at all to a screen reader and nothing precise to a low-vision user relying on magnification, regardless of how well-designed the visual chart itself is.

- **DO:** Ensure interactive chart elements (a bar that reveals a tooltip on hover, a legend that toggles a series) are keyboard-operable and expose their information through an accessible name or live region, not solely through mouse hover.

- **DON'T:** Build chart interactivity exclusively around mouse hover events, silently excluding keyboard-only and touch users from any information or interaction gated behind that hover.

### Skip Links and Bypass Blocks

- **DO:** Provide a visually-hidden-until-focused "skip to main content" link as the first focusable element on any page with a substantial, repeated navigation structure, so keyboard users can bypass repeated navigation and jump straight to the page's unique content.

```html
<!-- GOOD: hidden until focused, then clearly visible and functional -->
<a class="skip-link" href="#main-content">Skip to main content</a>
...
<main id="main-content">...</main>
```

- **DON'T:** Force keyboard users to tab through an entire repeated navigation structure — every single time, on every single page — before reaching a page's actual unique content. Without a skip link, this repetitive tabbing is a real, cumulative burden for keyboard-only users navigating a multi-page product.

### Pointer Target Spacing (WCAG 2.5.5 / 2.5.8)

- **DO:** Meet the WCAG target-size guidance (a minimum of 24×24 CSS pixels at Level AA, with 44×44 recommended at the stricter AAA level) for any pointer-operable control that isn't part of a sentence of inline text, in addition to the platform-level 44–48px conventions covered under Components — these two requirements reinforce each other and should both be satisfied.

- **DON'T:** Treat platform touch-target conventions (44pt/48dp) and the WCAG success criterion as two unrelated, optional guidelines only one of which needs to be satisfied. They overlap substantially but aren't identical in scope, and a genuinely accessible interface should satisfy the WCAG minimum as a floor even where a platform convention might technically allow smaller.

- **DO:** Provide adequate, uncrowded spacing between small adjacent targets as an acceptable alternative where a target genuinely can't be enlarged (per WCAG's spacing exception), ensuring there's no overlap in the effective clickable/tappable area between adjacent controls.

- **DON'T:** Pack small pointer targets tightly together with no spacing and no way to reliably distinguish which one a slightly imprecise click or tap was actually intended for.

### Error Identification and Suggestion (WCAG 3.3.1 / 3.3.3)

- **DO:** Programmatically identify which specific field an error relates to (via `aria-invalid` and `aria-describedby`, as covered under Form Labels), and, wherever the correction is knowable, suggest it — matching the practical Helpful Error Messages guidance under Content Design & Microcopy, but as a formal, testable WCAG success criterion, not merely a best practice.

- **DON'T:** Announce that a form submission failed with no indication of which field(s) are actually invalid, or provide error text visually adjacent to a field with no corresponding programmatic association a screen reader can detect (see Form Labels, above, for the concrete implementation).

### Accessible Names vs. Visible Labels (Label in Name)

- **DO:** Ensure a control's programmatic accessible name contains the same visible text a sighted user sees as its label — if a button visually reads "Search," its accessible name (via `aria-label` or otherwise) should also contain the word "Search," not an unrelated string.

```html
<!-- BAD: visible label and accessible name don't match -->
<button aria-label="Submit form">Search</button>

<!-- GOOD: accessible name matches (or contains) the visible label text -->
<button aria-label="Search products">Search</button>
```

- **DON'T:** Give a control an `aria-label` that doesn't match or contain its visible text label. This specifically breaks speech-input users (Dragon NaturallySpeaking and similar tools), who operate controls by speaking their visible label out loud — if the accessible name doesn't match what's visibly printed on the control, a voice command referencing the visible text will fail to find it.

- **DO:** Prefer letting a control's accessible name be derived naturally from its own visible text content whenever possible, reaching for a supplementary `aria-label` only to add necessary extra context (not to replace the visible text entirely).

- **DON'T:** Override a control's accessible name entirely with an `aria-label` that replaces rather than supplements its visible text, when the visible text alone would already have provided a perfectly adequate accessible name on its own.

### Text Spacing and Adjustable Text (WCAG 1.4.12)

- **DO:** Ensure content and layout remain fully functional and undamaged when a user overrides text spacing — line height, paragraph spacing, letter spacing, and word spacing — up to the levels defined in WCAG's text-spacing success criterion, without content being clipped, overlapping, or truncated.

- **DON'T:** Build layouts with fixed-height text containers that assume a specific line-height and break — clipping or overlapping text — the moment a user's assistive technology or browser extension increases spacing beyond the assumed default.

- **DO:** Test critical flows with a text-spacing override applied (several browser extensions and some assistive technologies offer this) as part of the same testing pass used for zoom/reflow testing, since both stress similar layout assumptions.

- **DON'T:** Treat text-spacing adjustment as too rare an edge case to bother testing. It's a real, standards-defined accessibility need for users with certain reading or attention-related disabilities, and it's typically inexpensive to support correctly if flexible layout principles (see Layout & Spacing) were followed in the first place.

### Orientation and Reflow

- **DO:** Support both portrait and landscape orientation for any content that isn't inherently and legitimately restricted to one (a bank check deposit camera view being a defensible exception; a general content or form page is not), letting the operating system's orientation setting govern rather than locking it in code.

- **DON'T:** Lock a general-purpose screen to a single orientation with no functional reason tied to the content itself. Orientation lock imposed for aesthetic or convenience reasons alone can create a genuine barrier for a user who has their device mounted or fixed in a specific orientation for a physical or motor-related reason.

### Live Regions for Dynamic Content Announcements

- **DO:** Mark a region that updates dynamically without a full page navigation — a live search result count, a form submission status, a chat message arriving — with an appropriate `aria-live` region (`polite` for most updates, `assertive` only for genuinely urgent, immediate information) so screen reader users are informed of the change without needing to manually re-discover it.

```html
<!-- GOOD: screen readers announce the updated count without a page reload -->
<div aria-live="polite">
  <p id="result-count">42 results found</p>
</div>
```

- **DON'T:** Let content update dynamically on screen with no live region at all. A sighted user visually notices a result count changing or a new chat message appearing; a screen reader user gets no equivalent signal unless the update happens inside a properly marked live region, silently missing information a sighted user receives automatically.

- **DO:** Use `aria-live="polite"` for the large majority of dynamic updates (it waits for the screen reader to finish its current announcement before speaking the update), reserving `aria-live="assertive"` narrowly for information that genuinely needs to interrupt immediately (a critical, time-sensitive error).

- **DON'T:** Default every live region to `assertive`. Overusing assertive announcements interrupts the screen reader user's current task repeatedly and unpredictably, which is disruptive in a way that's easy for a sighted developer to overlook, since it has no equivalent visual effect for them to notice while testing.

### Accessible Data Tables

- **DO:** Use real `<table>` markup with `<th>` header cells (and a `scope="col"`/`scope="row"` attribute, or `headers`/`id` associations for complex tables) for genuinely tabular data, so a screen reader can announce each cell's corresponding row and column header as the user navigates through it.

```html
<!-- GOOD: proper table semantics with scoped headers -->
<table>
  <thead>
    <tr><th scope="col">Product</th><th scope="col">Q3 Revenue</th></tr>
  </thead>
  <tbody>
    <tr><th scope="row">Widget A</th><td>$42,000</td></tr>
  </tbody>
</table>
```

- **DON'T:** Build a visually table-like grid out of generic `<div>` elements with CSS Grid or Flexbox for genuinely tabular data, with no semantic table structure underneath. A screen reader user navigating a div-based pseudo-table gets none of the row/column header announcements that make a real data table navigable — they hear a disconnected sequence of values with no context for what each one means.

- **DO:** Give a data table a caption or an accessible name summarizing what it contains, especially when a page has multiple tables, so a screen reader user can distinguish between them without reading each one's full content first.

- **DON'T:** Leave multiple tables on a page with no distinguishing caption or accessible name, forcing a screen reader user to read into each one's content just to figure out which table is actually relevant to what they're looking for.

## Content Design & Microcopy

### Time Zone Display and Ambiguity

- **DO:** Always make the time zone a displayed time refers to unambiguous — showing the user's local time zone explicitly (or converting to it automatically with a clear indicator), especially for any time that involves coordination across people who might be in different zones (a meeting, a deadline, an event).

```text
BAD:  "Meeting at 3:00 PM"  (whose 3:00 PM? the server's? the organizer's?)
GOOD: "Meeting at 3:00 PM EDT (your local time)" or an auto-converted
      "Meeting at 12:00 PM your time (3:00 PM organizer's time, EDT)"
```

- **DON'T:** Display a bare time with no time zone context in any situation where the viewer might reasonably be in a different zone than whoever set the time. Ambiguous time zone display is a well-known, high-consequence source of real-world mistakes — missed meetings, missed deadlines — precisely because the error is invisible until it's too late to correct.

- **DO:** Convert and display times in the viewer's own local time zone by default wherever the underlying data supports it, since most users expect a time shown to them to already be in their own zone unless a specific reason (a global event with one canonical time) calls for showing a fixed reference zone instead.

- **DON'T:** Force every user to mentally convert a displayed time from an unfamiliar reference zone (a server's zone, the product's headquarters zone) into their own. Automatic, clearly-indicated local-time conversion removes a genuinely error-prone mental task from the user entirely.

The words in an interface are not an afterthought layered on top of finished visuals — they are load-bearing. A beautifully designed error state with a confusing message still fails the user at the exact moment they need help most; a generic "Submit" button on a form with several possible outcomes leaves the user guessing what will actually happen when they click it. Content design applies the same rigor to words that visual design applies to layout: clarity, consistency, and purpose over decoration.

### Clear, Concise UI Copy

- **DO:** Write UI copy in plain, direct language that a first-time user — with no special training on the product — can understand without needing to look anything up. Prefer the word a general audience actually uses over the internal or technical term the team uses among itself.

```text
BAD:  "Instantiate a new workspace entity"
GOOD: "Create a new workspace"
```

- **DON'T:** Use internal system, database, or engineering terminology in user-facing copy — "Entity," "Object," "Record," "Payload" — when a plain, task-oriented word conveys the same thing more clearly to the person actually using the product.

- **DO:** Keep copy as short as it can be while remaining fully clear — every extra word a user has to read is a small tax on their attention, and UI copy is read under time pressure far more often than it's read carefully.

- **DON'T:** Pad microcopy with throat-clearing filler phrases — "Please note that...", "In order to...", "It is important to understand that..." — that delay the actual point without adding meaning. Cut straight to the information or instruction.

- **DO:** Write in second person ("your," "you") when addressing the user directly, and use active voice and concrete verbs rather than passive constructions, since active voice generally reads faster and states more clearly who is doing what.

```text
BAD:  "An error was encountered while your file was being processed"
GOOD: "We couldn't process your file"
```

- **DON'T:** Default to passive voice or vague third-person phrasing that obscures who or what is responsible for an action or an error. Passive constructions frequently hide the exact information — what happened, and to what — that the user actually needs.

- **DO:** Keep terminology consistent across the entire product — if a feature is called "Workspaces" in the navigation, don't refer to the same thing as "Projects" or "Spaces" in a different screen's copy.

- **DON'T:** Let different teams or screens independently invent their own names for the same concept. Inconsistent terminology forces users to constantly re-verify whether two differently-named things are actually the same thing or genuinely different — a small but real ongoing cognitive tax.

### Actionable Button and CTA Labels

- **DO:** Label buttons and calls-to-action with the specific action or outcome they produce — "Save changes," "Delete project," "Send invite," "Download report" — so the user knows exactly what will happen before they click, without needing surrounding context to disambiguate.

```text
BAD:  [ OK ]  [ Submit ]  [ Yes ]
GOOD: [ Delete project ]  [ Send invite ]  [ Keep editing ]
```

- **DON'T:** Default every confirmation button to a generic "Submit," "OK," or "Yes" when a specific verb-object label would be clearer and would help prevent accidental mis-clicks, especially in a dialog where the consequences of the two options genuinely differ.

- **DO:** Make destructive or irreversible actions unambiguous in their label — "Delete forever," "Permanently remove," "Cancel subscription" — so their weight is clear from the label alone, without requiring the user to have carefully read the surrounding paragraph.

- **DON'T:** Label a destructive, hard-to-reverse action with the same generic phrasing used for safe, reversible ones. If "Remove" is used both for "temporarily hide from this view" and "permanently delete," users will eventually apply the wrong mental model to one of them, with real consequences for the destructive case.

- **DO:** Match a confirmation dialog's button labels to the specific question being asked, rather than defaulting to generic Yes/No or OK/Cancel — a dialog asking "Discard unsaved changes?" reads far more clearly with buttons labeled "Discard changes" and "Keep editing" than with generic "Yes" and "No."

- **DON'T:** Use a generic "Yes"/"No" or "OK"/"Cancel" pair on a dialog where the question itself could be misread or where "Yes" and "No" don't map obviously onto which button does what. A user skimming quickly is far more likely to correctly interpret an explicit action label than to correctly map an abstract "Yes" back to the specific question asked.

- **DO:** Keep button label length appropriately short (typically one to three words) while still being specific — "Save draft" rather than either the too-generic "Save" or the overly verbose "Save this as a draft version."

- **DON'T:** Write button labels so long they wrap awkwardly, get truncated, or visually overwhelm the button's role as a small, scannable action trigger.

### Helpful Error Messages

- **DO:** State plainly what happened and how to fix it, using the user's own language rather than internal system terminology or a raw error code, in every error message the user might encounter.

```text
BAD:  "Error 4032: Validation failed"
GOOD: "That email address doesn't look valid. Check for typos and try again."
```

- **DON'T:** Surface raw exceptions, stack traces, HTTP status codes, or internal error identifiers directly to end users with no plain-language explanation. A user confronted with "Error 500" or an unhandled exception dump has no actionable next step and correctly perceives the product as broken.

- **DO:** Pair every error message with a concrete next step or recovery action wherever one exists — a link to retry, a suggestion of what to check, a way to contact support with the error already logged — so the user isn't left at a dead end.

- **DON'T:** End an error message with no path forward, leaving the user to guess whether to retry, wait, reload, or give up entirely. An error with no recovery path is worse than no error message at all, because it confirms something went wrong while offering nothing to do about it.

- **DO:** Write error messages that are specific to the actual failure — distinguish "This email is already registered" from "That password is too short" from "We couldn't reach the server, try again" — rather than collapsing every possible failure into one generic message.

- **DON'T:** Use one catch-all error message ("Something went wrong") for every possible failure mode when the underlying cause is actually known and could be communicated specifically. A single undifferentiated error message forces the user to guess at a cause the system already knows.

- **DO:** Keep error message tone calm, neutral, and non-blaming, even when the error is caused by user input — "That doesn't look like a valid phone number" rather than anything that implies the user did something wrong in a judgmental way.

- **DON'T:** Write error copy that blames or lectures the user ("You entered an invalid value," "You must fill out this field correctly"). Neutral, helpful phrasing gets the same information across without adding an unnecessary emotional cost to an already-frustrating moment.

### Meaningful Empty States

- **DO:** Use an empty state to explain what belongs in this space and provide a clear, specific call-to-action to add the first item — "You haven't created any projects yet. [Create your first project]" — rather than leaving a bare, unexplained void.

- **DON'T:** Show a plain "No data" label or an entirely blank area with no explanation and no next step. A first-time user encountering true emptiness with zero guidance often can't tell whether the product is broken, still loading, or genuinely just waiting for them to create something.

- **DO:** Differentiate a true zero-state ("you haven't created anything here yet") from a filtered-to-zero state ("your search or filters matched nothing"), with distinct copy and a distinct next action for each — a zero-state points toward creation, a no-results state points toward adjusting the search or filter.

```text
Zero-state:      "No projects yet. Create your first one to get started."
No-results state: "No projects match 'quarterly report'. Try a different search or clear your filters."
```

- **DON'T:** Reuse identical empty-state copy for both situations. Telling a new user with genuinely nothing created yet that their "search returned no results," or telling a user who over-filtered that they need to "create their first item," misdirects both of them toward the wrong next action.

- **DO:** Use empty states as an opportunity to briefly explain the value or purpose of the feature, especially for a first-time user who hasn't yet seen it populated with real content — a short line of context plus the call-to-action does more work than the call-to-action alone.

- **DON'T:** Treat every empty state identically regardless of context — a first-run empty state (before the user has ever used the feature) and a later empty state (after the user deleted everything) can reasonably use different tones and different levels of onboarding-style explanation.

### Voice and Tone Consistency

- **DO:** Define a consistent voice for the product — the underlying personality that doesn't change (e.g., direct and reassuring, or precise and technical) — while allowing tone to flex appropriately by context (a celebratory tone for a success state, a calmer and more careful tone for an error or destructive-action warning).

- **DON'T:** Let copy swing unpredictably between a casual, jokey tone in one part of the product and a stiff, formal tone in another, with no deliberate reason for the difference. Inconsistent voice makes a product feel like it was written by several different people who never talked to each other — which, in a large team without a documented voice guide, it usually was.

- **DO:** Match tone to the emotional stakes of the moment — playful, light copy is appropriate for a minor confirmation ("Nice, you're all set!") but inappropriate for a serious error, a data-loss warning, or a billing failure, where a calmer, more direct tone serves the user better.

- **DON'T:** Apply a uniformly casual, jokey tone across every single message regardless of context, including moments of genuine user frustration or risk (a failed payment, a permanent deletion warning). Humor in the wrong moment reads as tone-deaf rather than charming.

### Confirmation and Destructive-Action Dialogs

- **DO:** State the specific consequence of a destructive action plainly in the confirmation dialog's body text — what exactly will be deleted, whether it's reversible, and what (if anything) else depends on it — rather than a generic "Are you sure?"

```text
BAD:  "Are you sure you want to do this?"
GOOD: "Delete 'Q3 Planning'? This will permanently remove the document and
       its 14 comments. This can't be undone."
```

- **DON'T:** Use a generic, context-free confirmation prompt ("Are you sure?") for every destructive action across the product. A generic prompt gives the user no new information to base their decision on — they're being asked to confirm a decision they may not fully remember making, without being reminded what it actually does.

- **DO:** Require a higher-friction confirmation step (typing the item's name, a secondary "type DELETE to confirm" pattern) for especially high-consequence, irreversible actions — deleting an entire account, a production database, or a large body of content — proportional to how catastrophic and irreversible the action actually is.

- **DON'T:** Apply the exact same one-click confirmation pattern to both a low-stakes reversible action and a catastrophic irreversible one. Friction should scale with consequence; treating "archive this note" and "permanently delete this organization and all its data" identically under-protects the second one.

### Writing for Scannability

- **DO:** Structure longer in-product content (help text, onboarding copy, settings descriptions) with clear headings, short paragraphs, and bulleted lists where appropriate, since most users scan interface copy rather than reading it linearly start to finish.

- **DON'T:** Write long, dense paragraphs of unstructured prose for content the user is likely to scan rather than read closely — settings descriptions, feature explanations, help documentation embedded in the product. Unstructured prose forces a scanning user to read every word to find the one relevant piece of information.

- **DO:** Front-load the most important word or piece of information at the start of a sentence, label, or list item, since scanning users disproportionately read beginnings and skip or skim the rest.

- **DON'T:** Bury the key word or action at the end of a long label or sentence, forcing a scanning reader to read the whole thing to extract the part that actually matters to their decision.

### Pluralization and Localization-Ready Copy

- **DO:** Build UI copy with proper pluralization handling from the start — "1 item" vs. "2 items," "No comments" vs. "1 comment" vs. "5 comments" — using a real pluralization/ICU message format rather than string concatenation, since different languages have different (and sometimes more complex, e.g. zero/one/few/many/other) pluralization rules than English's simple singular/plural split.

```text
BAD (string concatenation, breaks pluralization and translation):
  count + " item(s)"

GOOD (ICU MessageFormat, handles pluralization correctly per locale):
  {count, plural, =0 {No items} one {# item} other {# items}}
```

- **DON'T:** Hardcode an "(s)" suffix pattern ("1 item(s)") as a shortcut for pluralization. This reads awkwardly even in English and breaks entirely once the copy is translated into a language with different or more granular plural rules.

- **DO:** Avoid concatenating sentence fragments around a variable when writing copy meant to be localized — write full, translatable sentences with placeholders, rather than assembling a sentence from separately-translated pieces, since word order and grammar vary significantly across languages.

- **DON'T:** Build a sentence by gluing together separately-stored string fragments around a dynamic value ("You have " + count + " new messages"). This pattern frequently breaks once translated into a language with different word order, grammatical gender agreement, or case requirements that the English fragment-based structure never had to account for.

### Content Governance and Style Guides

- **DO:** Maintain a written content style guide covering terminology, tone, capitalization rules, punctuation conventions, and formatting for numbers/dates/currency, so every contributor writing UI copy — designer, engineer, or product manager — produces text that reads as one voice.

- **DON'T:** Leave content decisions entirely to whoever happens to be implementing a given screen, with no shared reference. Without a style guide, "Cancel" vs. "cancel," "e-mail" vs. "email," and "sign in" vs. "log in" will all coexist inconsistently across the same product, each individually minor but collectively signaling a lack of care.

- **DO:** Review UI copy with the same rigor applied to visual design — a copy review pass before a feature ships, not copy treated as a placeholder that "someone will clean up later."

- **DON'T:** Ship UI copy as a rough placeholder ("Lorem ipsum," "TODO: better error message," a raw technical description) with the intention of refining it after launch. Placeholder copy reaches real users far more often than teams expect, and "we'll fix the wording later" is one of the most common paths by which permanently bad copy ships.

### Numbers, Dates, Units, and Currency Formatting

- **DO:** Format dates, times, numbers, and currency according to the user's actual locale — using a proper internationalization library (`Intl.DateTimeFormat`, `Intl.NumberFormat`, or an equivalent) — rather than a single hardcoded format assumed to be universal.

```text
BAD:  a single hardcoded format like "MM/DD/YYYY" shown to every user
GOOD: locale-aware formatting — "04/09/2026" (en-US) vs. "09/04/2026" (en-GB)
      vs. "2026年9月4日" (ja-JP), derived from the user's actual locale
```

- **DON'T:** Hardcode a single date, number, or currency format (commonly a US-centric one) and ship it globally. A date written "04/09" is genuinely ambiguous between April 9th and September 4th depending on the reader's locale convention, and that ambiguity has real consequences wherever the date matters (a deadline, a billing date, an appointment).

- **DO:** Display relative time ("2 hours ago," "yesterday") for recent events where relative framing is more useful, but always make the exact absolute timestamp available (commonly via a hover tooltip or an expandable detail) for cases where precision matters.

- **DON'T:** Show only a relative timestamp with no way to see the exact date and time. Relative time is convenient for a quick scan but becomes ambiguous or useless once precision matters — confirming exactly when an event happened, for a support ticket or an audit trail, for instance.

- **DO:** Show currency with its correct symbol and formatting convention for the relevant locale, and be explicit about which currency is being displayed (an ISO code or symbol) whenever a product handles more than one currency, to avoid ambiguity between similarly-symbolized currencies.

- **DON'T:** Show a bare currency symbol with no currency code in a multi-currency context, where "$" alone is ambiguous between US, Canadian, Australian, and several other dollar currencies. Ambiguous currency display is a real, costly mistake in any product handling international payments.

### Notification and Email Copy

- **DO:** Write push notification and email subject copy to be immediately clear about what the message contains and why the user is receiving it, since notification copy is read under even more time pressure and with even less surrounding context than in-app copy.

- **DON'T:** Write vague, clickbait-style notification copy ("You won't believe what happened!") purely to maximize open rate. Vague notification copy trains users to distrust or ignore future notifications and, for anything transactional or important, actively works against the goal of the notification being trusted and acted on.

- **DO:** Make transactional email and notification copy scannable and specific — the key information (an order status, a specific action needed, a specific date) visible without requiring the recipient to read a full paragraph to extract it.

- **DON'T:** Bury the actual actionable information inside a long paragraph of marketing-style prose in what is fundamentally a transactional message (a receipt, a password reset, a shipping update). Transactional messages are read quickly and functionally — treat their copy accordingly, distinct from marketing copy's different goals and different acceptable pacing.

### Legal, Compliance, and Consent Copy

- **DO:** Write consent and permission requests (cookie banners, data-sharing opt-ins, terms acceptance) in plain language that clearly states what's being asked and what the consequence of each choice is, even where legal review requires certain specific mandated phrasing elsewhere in the same flow.

- **DON'T:** Bury the actual choice being made inside dense legal boilerplate with no plain-language summary. A consent request a user can't actually understand isn't meaningfully informed consent, regardless of whether it satisfies a legal minimum — and it also just produces frustrated, confused users.

- **DO:** Make declining or opting out of a non-essential request exactly as easy and visually equal as accepting it — the same visual prominence, the same number of steps — rather than making the "yes" path frictionless and the "no" path deliberately harder to find or complete.

- **DON'T:** Design a consent flow with a large, high-contrast "Accept" button and a small, low-contrast, hard-to-find "Decline" or "Manage preferences" option. This is a well-documented dark pattern — making a technically-available choice practically difficult to exercise — and it undermines the legitimacy of consent obtained through it (see the AI-slop-specific dark-patterns guidance elsewhere in this document for more).

### Search Result and No-Match Copy

- **DO:** Write a specific, helpful message for a search that returns no results, ideally including the actual search term back to the user and a suggestion of what to try next (broaden the query, check spelling, browse categories instead).

- **DON'T:** Show a bare, generic "No results" message with no reference to what was searched for and no suggested next step. A specific no-match message that echoes the query back confirms the search was actually understood correctly, which matters when the real cause might be a typo the user can't otherwise identify.

- **DO:** Distinguish between "no results because nothing matches" and "no results because of a temporary error" — these have different causes and different appropriate user responses, and conflating them into one generic message misleads the user about what actually happened.

- **DON'T:** Show the same "no results found" copy for both a genuinely empty result set and a failed search request (a timeout, a backend error). A user told "no results" when the real problem was a failed request will reasonably conclude their search term simply doesn't exist, rather than retrying a request that might succeed the second time.

### Placeholder and Example Text

- **DO:** Use placeholder text specifically to show a format example (e.g., "MM/DD/YYYY" or "name@example.com") rather than as a substitute for the field's actual label, which — per Form Design, above — must remain visible on its own.

- **DON'T:** Write placeholder text that duplicates the label with no additional information ("Email" as both the label and the placeholder). A placeholder that adds nothing beyond what the label already says is a wasted opportunity to show a genuinely useful format example instead.

- **DO:** Keep placeholder text visually distinct (typically lighter/muted) from actual user-entered input, so a user can never mistake placeholder text for a value they've already typed.

- **DON'T:** Style placeholder text so close in contrast and weight to real input text that a user might glance at a field and mistakenly believe it's already filled in. This is both a usability problem and, at low enough contrast, an accessibility one.

### Confirmation and Success Messaging

- **DO:** Confirm that a significant action succeeded with clear, specific feedback ("Changes saved," "Invitation sent to jane@example.com") rather than leaving the user to infer success from the absence of an error.

- **DON'T:** Provide no positive confirmation after a significant action completes, leaving the user uncertain whether it actually worked. Silence is not a confirmation — a user who isn't sure an action succeeded will often retry it unnecessarily or abandon the flow out of uncertainty.

- **DO:** Match the confirmation's prominence to the action's significance — a subtle inline confirmation for a minor autosave, a more visible toast or banner for a significant, deliberate action (submitting an application, completing a purchase).

- **DON'T:** Give a trivial, frequent confirmation (an autosave, a minor toggle) the same prominent, attention-grabbing treatment as a major, infrequent one. Over-signaling minor successes trains users to tune out confirmation messages altogether, including the ones that matter.

### Instructional Copy and Help Text

- **DO:** Place helper text close to the field or control it explains, phrased as guidance for what to do, not just a restatement of the label — "Use at least 8 characters, including a number" rather than a redundant repeat of "Password."

- **DON'T:** Write helper text that just restates the label in slightly different words, adding no new, actually useful information. Helper text exists to answer a question the label alone doesn't — a format, a constraint, a reason — and should be cut if it doesn't do that.

- **DO:** Keep help text visible by default for fields with non-obvious requirements, rather than hiding it behind a hover or click interaction that a user might not discover before making an avoidable mistake.

- **DON'T:** Bury essential formatting or validation requirements behind a hover-only tooltip icon next to a field, especially when that requirement will otherwise only be discovered through a failed submission. Showing the constraint up front prevents the error rather than merely explaining it after the fact.

### Character Limits and Input Constraints

- **DO:** Show a live, visible character (or item) count as the user approaches a genuine limit, rather than only rejecting the input silently or after the fact once the limit is exceeded.

- **DON'T:** Let a user type well past an input's actual limit with no visible counter, only to have the excess silently truncated or the whole submission rejected after the fact. Silent truncation in particular risks losing content the user assumed was saved in full.

- **DO:** State the limit itself clearly in the interface ("500 characters remaining" or "120/500") rather than leaving the user to discover the ceiling only by hitting it.

- **DON'T:** Leave an input's maximum length completely undisclosed until the user happens to bump into it. An undisclosed constraint the user only discovers by failing against it is a preventable, unnecessary source of frustration.

### Abbreviations, Acronyms, and Jargon

- **DO:** Spell out an acronym or abbreviation on its first use in a given context, or provide an accessible expansion (an `<abbr title="...">` element, a tooltip, a glossary link), especially for domain-specific or internal terminology a first-time user won't already know.

- **DON'T:** Use unexplained acronyms and internal jargon throughout the interface, assuming every user shares the same background knowledge as the team that built the feature. What's an obvious, everyday abbreviation to the internal team is frequently opaque to a new or less technical user encountering it for the first time.

- **DO:** Prefer a plain, spelled-out term over an abbreviation in primary interface copy wherever space allows, reserving abbreviations for genuinely space-constrained contexts (a narrow table column, a small badge) where the full term doesn't fit.

- **DON'T:** Abbreviate a term purely out of habit or brevity preference when the full word would have fit comfortably. Unnecessary abbreviation adds a small comprehension tax for no real space benefit.

### Large Number Abbreviation and Display

- **DO:** Abbreviate large numbers consistently (1.2K, 3.4M, 2.1B) in space-constrained contexts (a compact stat tile, a chart axis label) while making the exact, unabbreviated value available on demand — a hover tooltip, an expandable detail — wherever precision might matter.

- **DON'T:** Abbreviate a number in a context where precision is actually important (an exact balance, a precise count the user needs to verify) with no way to see the full, unrounded value. Silent rounding in a context that calls for precision can lead a user to make a decision based on an imprecise figure they didn't realize was rounded.

- **DO:** Keep abbreviation formatting locale-aware and consistent with the surrounding number formatting conventions (see Numbers, Dates, Units, and Currency Formatting, above), since large-number abbreviation conventions can also vary by locale.

- **DON'T:** Hardcode a single abbreviation format (commonly a US/English convention) globally with no consideration for locale differences, compounding the same kind of formatting assumption problem covered under general number and date formatting.

### Permission and Access-Denied Messaging

- **DO:** Explain plainly why a user can't access something they've hit a permission boundary on, and — where appropriate — what they could do about it (request access, contact an admin, upgrade a plan), rather than a bare, unexplained denial.

- **DON'T:** Show a generic "Access denied" or "403 Forbidden" message with no explanation and no suggested next step. A permission error is exactly the kind of moment a user needs the most context, since the underlying reason (wrong role, expired trial, wrong workspace) isn't something they can typically infer on their own.

- **DO:** Distinguish, in the message itself, between "this doesn't exist" and "this exists but you don't have access to it" wherever that distinction doesn't itself leak sensitive information the system needs to protect — the two situations call for different user reactions and different copy.

- **DON'T:** Collapse every access-related failure into one generic, indistinguishable message when the underlying causes and appropriate user responses genuinely differ. (Note: in some security-sensitive contexts, deliberately not distinguishing "doesn't exist" from "exists but forbidden" is itself the correct choice to avoid leaking information — apply this guidance only where that specific security tradeoff doesn't apply.)

## Iconography & Imagery

### Icon Color Usage

- **DO:** Default to single-color (monochrome, currentColor-driven) icons that inherit their color from surrounding text or a semantic token, so an icon automatically adapts correctly across themes, states (hover, disabled), and semantic contexts (danger, success) without needing a separate colored asset for each.

```css
/* GOOD: a monochrome icon inherits color from context automatically */
.icon { fill: currentColor; }
.btn-danger .icon { color: var(--color-danger-600); }
```

- **DON'T:** Bake fixed colors directly into icon assets (a red trash icon exported as a red SVG) that then can't adapt to context, theme, or state without manually swapping the entire asset. A monochrome, `currentColor`-driven icon is dramatically easier to theme, recolor for different states, and keep consistent than a fixed-color asset.

- **DO:** Reserve genuinely multi-color icons (duotone, brand marks, illustrative icons) for specific, deliberate contexts — a brand logo, a distinctive empty-state illustration — where the multi-color treatment is itself part of the intended visual identity, not applied inconsistently to ordinary functional UI icons.

- **DON'T:** Mix single-color functional icons and multi-color decorative icons within the same dense UI context (a toolbar, a nav bar) with no consistent rule for when each applies. Functional icons benefit from the simplicity and adaptability of a single-color treatment; introducing multi-color icons into that same dense context adds visual noise without a corresponding functional benefit.

### Filled vs. Outline Icon States

- **DO:** Use a filled version of an icon to represent an active/selected/favorited state and an outline version for the inactive/default state (a filled heart for "favorited," an outline heart for "not favorited"), keeping this convention consistent everywhere the same kind of toggle appears.

```text
GOOD: ♡ (outline) = not favorited → tap → ♥ (filled) = favorited,
      using this exact filled/outline convention everywhere favoriting appears
```

- **DON'T:** Use an unrelated pair of icons, or inconsistent fill treatment, to represent a toggle's two states across different parts of the product. If "favorited" is a filled heart in one place and a star icon entirely in another, the visual language stops being learnable as one consistent system.

- **DO:** Pair the filled/outline state change with an additional cue (a color change, a brief animation) when the toggle represents a meaningful action, so the state change is doubly reinforced rather than relying on the filled/outline distinction alone, which can be subtle at small icon sizes.

- **DON'T:** Rely on the filled/outline distinction alone at a very small icon size where the difference may be hard to perceive at a glance. At small sizes, supplement the fill-state change with a color shift or a brief confirming animation so the state change reads clearly even when the fill difference itself is subtle.

### Custom Icons vs. Platform-Native Iconography

- **DO:** Weigh platform-native icon sets (SF Symbols on iOS, Material Symbols on Android) against a fully custom icon system deliberately — native icons offer instant familiarity and automatic platform-convention alignment (weight matching, dynamic type scaling), while a custom set offers tighter brand alignment at the cost of extra design and maintenance effort.

- **DON'T:** Mix native-platform icons and custom-brand icons inconsistently within the same interface with no clear rule for which is used where. A jarring mismatch between an OS-native icon's style and a custom-drawn icon's style right next to each other undermines the visual coherence either approach would have delivered on its own.

- **DO:** Where a custom icon set is used on a native mobile platform, match its underlying grid, optical sizing, and stroke conventions closely enough to the platform's own icon set that it still feels at home within standard platform chrome (navigation bars, tab bars) even though it's not literally the system set.

- **DON'T:** Drop a custom icon set with a visibly different grid, corner treatment, or stroke weight directly into platform-standard UI chrome with no accommodation. An icon set designed with no reference to the platform's own visual conventions can look conspicuously foreign inside otherwise-native navigation elements.

Icons and images carry meaning fast — often faster than text — which is exactly why inconsistency or ambiguity in this layer is so costly. An icon system that mixes styles looks unfinished in a way many users can sense without being able to name; an icon used with no label leaves users guessing at an action they can't safely undo if they guess wrong.

### Consistent Icon Style

- **DO:** Use icons from a single family or a single internally-designed set with consistent stroke weight, corner radius, corner treatment, and overall visual grid across every icon in the product, so any icon — regardless of who added it or when — looks like it belongs to the same family as every other.

```text
BAD:  a mix of thin 1px outline icons, thick 2.5px outline icons, and
      filled/solid icons scattered across the same toolbar

GOOD: every icon drawn on the same grid, at the same stroke weight,
      with the same corner radius and terminal style
```

- **DON'T:** Mix filled and outlined icon styles, or pull icons from multiple different icon libraries, within the same interface. Even when each individual icon is well-designed, mixing visual styles reads as an unplanned patchwork rather than a considered system — this is one of the fastest ways to make a polished layout suddenly look assembled rather than designed.

- **DO:** Design or select icons on a shared pixel grid (commonly 24×24) with consistent optical sizing, so icons of genuinely different shapes (a square, a circle, an arrow) still read as visually the same size next to each other, even though their literal bounding boxes may differ slightly to compensate for optical weight.

- **DON'T:** Scale one icon set's icons arbitrarily to "look about the same size" as icons from a different set, by eye, on a per-instance basis. Ad hoc scaling produces subtly inconsistent icon sizes throughout the interface that are individually hard to notice but collectively make the icon layer feel unrigorous.

- **DO:** Maintain a single source icon library (an SVG sprite sheet, an icon component library, or a shared Figma icon set) that both design and engineering pull from, so a new icon added anywhere in the product is automatically consistent with every existing one.

- **DON'T:** Let individual contributors source new icons ad hoc from different free icon websites whenever a screen needs "an icon for X." Icons sourced independently, one at a time, from different providers essentially never share a consistent style, weight, or grid with each other.

### Icons Paired with Labels

- **DO:** Pair icon-only controls with a visible text label, or at minimum a clear, accessible name (`aria-label`) and a tooltip, whenever the action the icon represents isn't close to universally understood at a glance.

```html
<!-- BAD: an ambiguous icon with no label and no accessible name -->
<button><svg>...</svg></button>

<!-- GOOD: an accessible name for screen readers, plus a visible label or tooltip -->
<button aria-label="Archive conversation">
  <svg aria-hidden="true">...</svg>
  <span class="visually-hidden">Archive conversation</span>
</button>
```

- **DON'T:** Ship icon-only buttons for ambiguous or non-obvious actions with no accompanying label and no accessible name. A user (sighted or not) who can't confidently interpret an icon either avoids the action entirely or clicks it hesitantly, unsure of the consequence — both are usability failures for something as simple as a button.

- **DO:** Reserve fully unlabeled icon-only controls for a small set of genuinely universal, widely-recognized icons — a close "×," a search magnifying glass, a hamburger menu, a back arrow — where near-universal recognition makes an additional label largely redundant for most users, while still keeping an accessible name present for assistive technology.

- **DON'T:** Assume every icon in a design library is universally understood just because it feels self-evident to the designer who chose it. Icon metaphors that feel obvious to someone deeply familiar with a product's domain (a specific chart-type icon, a niche status symbol) are frequently opaque to first-time or occasional users.

- **DO:** Use a tooltip as a supplementary confirmation for an icon-only control aimed at experienced/frequent users (a compact toolbar, a power-user shortcut bar), while still keeping a real accessible name present regardless of whether the tooltip is visually shown.

- **DON'T:** Treat a hover-triggered tooltip as sufficient labeling on its own for an icon-only control. As covered under Components, tooltips are not reliably accessible to touch and keyboard-only users, so an icon relying purely on a hover tooltip for its meaning has effectively no label for a meaningful share of users.

### Icon Accessibility

- **DO:** Mark purely decorative icons — ones that are redundant with adjacent visible text, or add no independent meaning — as `aria-hidden="true"` (for inline SVG) or with empty alt text, so assistive technology skips over them instead of announcing a redundant or meaningless icon name.

- **DON'T:** Let a decorative icon's underlying SVG title, filename, or default icon-font character get announced by a screen reader as extra, meaningless noise alongside text that already conveys the same information.

- **DO:** Ensure any icon conveying real information on its own (a status indicator, a severity marker) has a proper accessible name conveying that meaning, not just a purely visual treatment.

- **DON'T:** Rely on an icon alone, with no accessible name, to convey status or meaning that isn't otherwise present as text nearby. A colored dot or icon with no text equivalent conveys nothing to a screen reader user and, per the Color section above, often conveys ambiguous information to sighted users too.

### SVG vs. Icon Fonts

- **DO:** Prefer inline SVG (or SVG sprite systems) over icon fonts for new icon systems. SVG icons render crisply at any size, support per-icon multi-color treatments, fail gracefully (a missing SVG shows nothing, not a stray character), and integrate cleanly with CSS and accessibility attributes.

- **DON'T:** Introduce a new icon-font-based system today without a specific reason. Icon fonts render as literal font glyphs, which means a failed font load can display broken "tofu" boxes or unrelated characters instead of icons, and icon fonts historically caused real accessibility problems when a screen reader announced their underlying Unicode character as if it were text.

- **DO:** Optimize SVG icon assets (strip unnecessary metadata, simplify paths, use a build-time optimizer) to keep the icon system's total payload small, especially when using an inline sprite referenced across many components.

- **DON'T:** Ship unoptimized, editor-exported SVG files directly to production, complete with unused metadata, redundant groups, and excessive path precision. This bloats payload for no visual benefit and is easy to fix with a standard SVG optimization step in the build pipeline.

### Purposeful Imagery Over Generic Stock

- **DO:** Choose imagery that communicates something specific and true about the actual product, content, or people involved — a real product screenshot, a genuine data visualization, an illustration built specifically for the brand — so the image adds real information rather than merely filling space.

- **DON'T:** Default to generic stock photography — smiling people in blazers shaking hands, isolated laptops on a clean desk, abstract "teamwork" imagery — that says nothing specific about the actual product or its value. Generic stock imagery is instantly recognizable as filler and actively signals low effort rather than the polish it's meant to convey.

- **DO:** Use real product screenshots, real data, or illustrations custom-built for the brand's visual language when explaining what a product does or why it matters — showing the actual thing is almost always more convincing and more memorable than a generic metaphor for it.

- **DON'T:** Substitute an abstract or metaphorical image for content that could be shown directly and concretely. A screenshot of the actual feature being described communicates more, and more credibly, than a stock photo standing in for the idea of the feature.

- **DO:** Ensure decorative imagery supports and doesn't compete with the content or message it accompanies — sufficient contrast, sufficient simplicity, and a clear visual hierarchy that keeps the image from pulling attention away from the actual point of the page.

- **DON'T:** Let a busy, high-detail decorative image compete directly with foreground text and controls for the user's attention. If an image is meant to be a backdrop, it needs to visually behave like one — appropriately muted, blurred, or overlaid — not compete on equal visual terms with the content in front of it.

### Consistent Aspect Ratios and Treatment

- **DO:** Standardize aspect ratios by image role across the system — commonly 16:9 for hero/banner imagery, 1:1 for avatars and profile images, 4:3 or 3:2 for content thumbnails — and apply that standard consistently everywhere that role appears.

```css
/* GOOD: a consistent aspect ratio enforced per image role */
.avatar { aspect-ratio: 1 / 1; object-fit: cover; border-radius: 50%; }
.card-thumbnail { aspect-ratio: 4 / 3; object-fit: cover; }
.hero-banner { aspect-ratio: 16 / 9; object-fit: cover; }
```

- **DON'T:** Let every content card in the same grid use whatever aspect ratio its source image happened to be uploaded at. A grid of images with inconsistent, uncropped aspect ratios produces a ragged, unplanned-looking layout even if every individual image is high quality.

- **DO:** Apply a consistent visual treatment — corner radius, border, overlay gradient, filter/duotone effect — across every image within a given category, so a user can recognize "this is a profile photo" or "this is an article thumbnail" purely from its consistent framing.

- **DON'T:** Mix sharply-cornered and heavily-rounded image treatments, or apply an overlay/filter to some images in a category but not others, within the same list or grid. Inconsistent treatment within one content type is a small detail that nonetheless reads as carelessness once a user sees enough instances of it side by side.

### Responsive Images and Formats

- **DO:** Serve appropriately sized and modern-format images (`srcset`/`sizes` for responsive resolution switching, formats like WebP or AVIF with a fallback) so users on smaller viewports or slower connections aren't forced to download a full desktop-resolution image unnecessarily.

```html
<!-- GOOD: serves an appropriately sized, modern-format image per viewport -->
<img
  src="hero-800.jpg"
  srcset="hero-400.webp 400w, hero-800.webp 800w, hero-1600.webp 1600w"
  sizes="(max-width: 600px) 100vw, 50vw"
  alt="Product dashboard showing quarterly revenue trends">
```

- **DON'T:** Serve one large, fixed-resolution image to every viewport size and device, letting the browser scale it down with CSS. This wastes bandwidth on smaller screens and slower connections, directly hurting both load performance and the user's actual experience, especially on mobile networks.

- **DO:** Set explicit `width`/`height` attributes (or an equivalent `aspect-ratio` reservation) on images so the browser can reserve the correct space before the image loads, preventing content from visibly shifting once it arrives.

- **DON'T:** Omit image dimensions and let layout shift occur as images load in. Unreserved image space is one of the most common causes of cumulative layout shift — a page visibly jumping as content loads is jarring and can cause a misplaced click right as a shift happens.

### Illustration Style and Brand Consistency

- **DO:** Define a consistent illustration style (line weight, color palette, level of realism, character proportions) when custom illustration is part of the visual language, and apply it uniformly across every illustrated asset — empty states, onboarding, marketing pages — so illustrations read as one coherent set rather than a collection of individually commissioned pieces.

- **DON'T:** Mix illustration styles from different sources (a flat-vector empty-state illustration next to a hand-drawn onboarding graphic next to a 3D-rendered marketing hero) within one product experience. Inconsistent illustration style is exactly as damaging to perceived quality as inconsistent icon style, for the same reason — it signals the absence of a considered system.

- **DO:** Keep a favicon and app icon design simple, high-contrast, and legible at very small sizes (down to 16×16px for a browser tab), since these are viewed at the smallest size of any brand asset the product produces.

- **DON'T:** Use a detailed, low-contrast, or text-heavy logo mark as a favicon or app icon without simplifying it first. A logo that reads perfectly at business-card size frequently collapses into an illegible smudge at 16×16px — a favicon typically needs its own simplified, purpose-built version.

### Icon and Image Sourcing, Licensing, and Governance

- **DO:** Maintain a clear record of each icon and image asset's license and source, especially for any third-party or stock content, so the product's legal usage rights are actually known and auditable rather than assumed.

- **DON'T:** Pull icons or images from an unclear or unlicensed source "because it showed up in an image search" without verifying usage rights. Unlicensed asset usage is a real legal and business risk, not just a design-process nicety, and it's far cheaper to verify licensing up front than to replace an asset already shipped to production.

- **DO:** Centralize approved icon and image sources (a licensed stock library, an internal asset library, a specific icon set) so every contributor pulls from the same vetted, licensed pool rather than sourcing independently per project.

- **DON'T:** Let each team or contributor independently source imagery and icons from whatever free or convenient source they find. Beyond the licensing risk, decentralized sourcing is also how visual inconsistency (see Consistent Icon Style, above) creeps in in the first place.

### Photography and Art Direction

- **DO:** Establish a consistent art direction for photography used across the product — a consistent color grade/treatment, a consistent subject framing style, a consistent level of staging vs. candid realism — so photography reads as belonging to one considered visual system.

- **DON'T:** Mix photography with wildly different color treatments, lighting styles, and subject framing across the same product or campaign. Inconsistent photographic treatment is as visible a system failure as inconsistent icon style, and it's one of the most common tells of imagery sourced ad hoc from disparate stock libraries.

- **DO:** Choose imagery that authentically represents the actual range of people, contexts, and use cases the product serves, rather than a narrow, homogenous default that fails to reflect the real audience.

- **DON'T:** Default unthinkingly to the narrowest, most generic representation available in a stock library. Imagery that doesn't reflect the actual diversity of a product's real user base is both a missed opportunity for genuine connection and, in some contexts, actively alienating to users who never see themselves represented.

### Status Badges and Notification Iconography

- **DO:** Design a consistent, small set of badge and notification indicator styles (a count badge, a dot indicator, a status pill) reused identically everywhere a similar kind of status needs to be shown, so users learn to read them at a glance across the whole product.

- **DON'T:** Let different features invent their own visually distinct badge styles for conceptually equivalent information (an unread count shown as a red circle in one place and an orange square elsewhere). Inconsistent badge treatment forces the user to re-learn what a visually different but functionally identical indicator means each time.

- **DO:** Cap numeric badge counts at a sensible display maximum (commonly showing "99+" beyond a threshold) rather than letting an unbounded number distort the badge's shape and legibility.

- **DON'T:** Let a count badge grow to accommodate arbitrarily large numbers with no cap, producing a badge shape that no longer resembles the compact, consistent indicator it's supposed to be for every other, smaller count.

### Avatar and Profile Image Design

- **DO:** Define a consistent avatar treatment (shape — circle or rounded square, size steps, a defined fallback for users without a photo) reused identically everywhere a person or account is represented across the product.

- **DON'T:** Let avatar shape or size vary inconsistently across different parts of the product (circular avatars in one list, square ones in another). Inconsistent avatar treatment undermines the at-a-glance recognizability that a consistent visual identity element is meant to provide.

- **DO:** Design a clear, consistent fallback for users without a profile photo — initials on a deterministic background color, or a generic silhouette — so every user is represented by something, never a broken image icon.

- **DON'T:** Let a missing avatar image render as a broken-image icon or an empty blank space. A designed, deliberate fallback (initials, a generated color, a default icon) should always be in place before a broken-image state is ever visible to a user.

### Chart and Data Icon Consistency

- **DO:** Use a consistent, small set of icons to represent common chart types, data states, and trend indicators (an up/down trend arrow, a specific icon per chart type) reused identically across every dashboard and report in the product.

- **DON'T:** Let different dashboards or reporting surfaces use different iconography for the same underlying concept (one dashboard's "increase" arrow pointing a different direction or using a different color convention than another's). Inconsistent data iconography undermines the quick, pattern-based reading that dashboards are meant to support.

### Locale and Region Iconography

- **DO:** Use a country flag icon specifically to represent a country or region, never as a stand-in for a language — a language can be spoken in multiple countries (or multiple languages spoken in one country), so a flag-as-language-selector is both imprecise and can be unintentionally exclusionary to speakers of that language outside the flag's specific country.

- **DON'T:** Build a language picker using only flag icons with no accompanying text label naming the language. Beyond the language-vs-country mismatch, many flags are visually similar enough at small icon sizes to be genuinely hard to distinguish at a glance.

- **DO:** Pair any locale- or region-specific icon with a clear text label naming the actual language or region, so the control remains unambiguous regardless of how familiar a given user is with a specific flag's design.

- **DON'T:** Assume every user can instantly and correctly identify every relevant country's flag from memory. A text label removes this assumption entirely and is a small addition relative to the ambiguity it prevents.

### Icon Sizing Consistency Across Contexts

- **DO:** Define a small set of standard icon sizes tied to their usage context (e.g., 16px for inline-with-text, 20px for standard UI controls, 24px for primary navigation and toolbars) and apply them consistently, so the same icon at the same conceptual "weight" always renders at the same size wherever that context recurs.

- **DON'T:** Let icon size vary arbitrarily between visually similar contexts — one screen's toolbar icons at 18px, another visually equivalent toolbar's icons at 22px, with no functional reason for the difference. Inconsistent icon sizing across equivalent contexts is a subtle but real contributor to a product feeling assembled rather than designed.

- **DO:** Scale icon stroke weight appropriately alongside size changes — a larger icon rendered at a proportionally thin stroke can look weak or incomplete, while a small icon at a heavy stroke can look cramped or illegible — using a font/icon system that handles optical adjustment automatically where possible.

- **DON'T:** Naively scale one fixed-stroke-weight icon asset up or down without checking how the stroke weight reads at the new size. A stroke that reads as a clean, balanced line at 24px can look either too thin (lost at small sizes) or too thick (clunky at large sizes) once scaled without adjustment.

## Design Systems & Tokens

### Naming Conventions for Tokens and Components

- **DO:** Adopt one consistent, documented naming convention for tokens (a predictable pattern like `category-role-scale`, e.g. `color-text-secondary`, `space-inset-md`) and for components (a predictable pattern for base components vs. variants vs. composed patterns), applied uniformly so a name's structure alone tells you roughly what category of thing it refers to.

```text
GOOD naming pattern (predictable, learnable):
  color-{role}-{scale}     → color-danger-500, color-text-primary
  space-{purpose}-{size}   → space-inset-md, space-stack-lg
  Component-Variant        → ButtonPrimary, ButtonSecondary, CardElevated
```

- **DON'T:** Let token and component names accumulate ad hoc, with inconsistent word order, inconsistent abbreviation, and no predictable pattern (`primaryColor` next to `color-2` next to `brandBlueMain`). Inconsistent naming makes the system's contents hard to search, hard to predict, and hard to learn — a new contributor can't guess a token's name from its purpose if there's no consistent pattern to guess from.

- **DO:** Keep naming consistent in grammatical structure across similar categories — if spacing tokens go from general to specific (`space-inset-md`), color tokens should follow the same general-to-specific order (`color-text-secondary`), not a mixed or reversed order in a different category.

- **DON'T:** Use a different word-ordering convention for different token categories with no consistent underlying logic. Inconsistent internal grammar across the token system's own naming makes it harder to build the kind of predictable, guessable intuition a well-named system is supposed to provide.

### Breakpoints as Design Tokens

- **DO:** Treat the standard set of responsive breakpoints (see Grid Systems and Breakpoints, under Layout & Spacing) as design tokens themselves, defined once and referenced by name (`bp-sm`, `bp-md`, `bp-lg`) everywhere a responsive rule is needed, in both design tooling and code.

```css
/* GOOD: breakpoints defined once, referenced by name everywhere */
:root {
  --bp-sm: 480px;
  --bp-md: 768px;
  --bp-lg: 1024px;
}
```

- **DON'T:** Let individual components hardcode their own raw pixel breakpoint values independently. Just as with color, spacing, and type, ungoverned breakpoint values drift over time — one component responding at 767px, a visually adjacent one at 770px — producing layouts that shift at slightly different, uncoordinated points as the viewport resizes.

- **DO:** Keep breakpoint tokens synchronized between whatever design tool is used for mockups and the actual CSS/code values, exactly as with every other token category, so a designer's frame width assumptions genuinely match what the shipped responsive behavior will do.

- **DON'T:** Let breakpoint values live only in code with no corresponding reference in the design tool, forcing designers to either guess or manually ask engineering every time they need to know where a layout will actually shift.

### Component Testing and Visual Regression

- **DO:** Maintain automated visual regression tests (screenshot comparisons across component states and viewports) for core shared components, so an unintended visual change introduced anywhere in the system is caught automatically before it reaches every consumer at once.

- **DON'T:** Rely solely on manual eyeballing to catch unintended visual regressions in shared, widely-used components. A shared component's blast radius is large — a single unnoticed visual regression there silently propagates to every single consumer of that component across the entire product.

- **DO:** Maintain a living component "playground" or catalog (a Storybook instance or equivalent) showing every component in every documented variant and state, usable both as documentation and as the environment automated visual tests run against.

- **DON'T:** Let a component's documented variants exist only as static descriptions with no live, interactive example actually demonstrating them. A living catalog that's out of sync with the real component (or that doesn't exist at all) leaves both consumers and automated tests with no reliable reference for what "correct" actually looks like.

A design system is only as valuable as its ability to be a single, trusted source of truth. The moment a component starts getting duplicated with slight variations, or a value gets hardcoded instead of referencing a token, the system begins fragmenting — and fragmentation compounds, because every inconsistency makes the "correct" system version feel more optional to whoever encounters it next.

### A Single Source of Truth for Tokens

- **DO:** Centralize every spacing, color, typography, elevation, and motion value as a named, documented token, consumed identically by the design tool and the codebase, so a value exists in exactly one authoritative place regardless of how many surfaces reference it.

```json
// GOOD: tokens defined once, consumed everywhere via a build pipeline
{
  "color": { "primary": { "500": { "value": "#4f6df5" } } },
  "space": { "4": { "value": "16px" } },
  "fontSize": { "300": { "value": "1.25rem" } }
}
```

- **DON'T:** Let a Figma color/type/spacing style library and the codebase's actual CSS or theme values exist as two separately maintained sources that drift apart over time as each gets updated independently. Divergence between design and code tokens is one of the most common root causes of "the shipped product doesn't quite match the design" reports.

- **DO:** Pipe tokens from one canonical definition (a `tokens.json`, a Style Dictionary configuration, or an equivalent build step) into every consuming platform — web CSS variables, iOS/Android theme resources, and the design tool itself via a plugin — so a single edit propagates everywhere automatically.

- **DON'T:** Manually re-enter or re-derive the same design values separately for each platform or surface. Manual re-entry guarantees eventual drift, since there is no mechanism forcing the copies to stay in sync as either one changes.

- **DO:** Version the token set explicitly, with a changelog describing what changed and why, so downstream consumers can understand the impact of a token update before adopting it, especially for a breaking change (a renamed token, a removed color).

- **DON'T:** Silently change a shared token's value with no changelog or versioning, especially in a way that could break visual assumptions baked into existing components (e.g., silently changing what `space-4` resolves to). Silent breaking changes to shared infrastructure erode trust in the whole system.

### Component Variant Discipline

- **DO:** Model each shared component's variations as an explicit, constrained, documented set of props (size: sm/md/lg; tone: default/primary/danger; state: default/loading/disabled) rather than allowing arbitrary, uncatalogued customization.

```jsx
// GOOD: a constrained, documented variant surface
<Button size="md" tone="danger" loading={isSubmitting}>
  Delete project
</Button>

// BAD: unconstrained style overrides that create an uncatalogued one-off variant
<Button style={{ background: '#ff4433', padding: '11px 19px', fontSize: 15 }}>
  Delete project
</Button>
```

- **DON'T:** Let every screen or team spin up its own slightly-different, hand-tweaked version of a shared component ("Button" with custom padding here, a one-off border-radius there) instead of using or extending the system's actual variant set. Each of these individually-reasonable-seeming tweaks is how a system with one well-defined "Button" ends up with a dozen visually inconsistent near-duplicates.

- **DO:** Require a lightweight design and engineering review before a new variant is added to a shared component, so the system's growth stays intentional and the new variant is checked against every existing state and size combination before merging.

- **DON'T:** Let variant count grow unbounded with no periodic pruning of variants that turned out to be rarely or never used. An unbounded, unpruned variant surface eventually becomes as confusing to navigate as having no system at all — more options isn't automatically more useful if most of them are near-redundant or unmaintained.

- **DO:** Keep variant naming consistent and predictable across every component in the system (`size` always means the same scale — sm/md/lg — regardless of which component it's applied to), so a developer who has learned one component's API can predict another's.

- **DON'T:** Let different components use inconsistent naming for conceptually identical props — one component's `size` prop taking `small`/`large`, another's taking `sm`/`lg`, a third's taking a raw pixel number. Inconsistent naming conventions across an otherwise well-designed system quietly undermine its learnability.

### Documenting Components

- **DO:** Document each component's intended purpose, the situations where it should (and shouldn't) be used relative to similar alternatives, its full prop/variant API, its accessibility behavior, and concrete do/don't usage examples, published somewhere the whole team can reference.

- **DON'T:** Ship a component library with only source code (or only Figma files) and no usage guidance. Code or design files alone tell a consumer *how* a component is built, but not *when* or *why* to reach for it instead of a similar-looking alternative — that gap is reliably filled with guesswork, which produces misuse.

- **DO:** Include accessibility notes directly in each component's documentation — expected keyboard behavior, required ARIA usage if any, known limitations — so accessibility is part of how the component is understood and used correctly from day one, not a separate concern layered on afterward.

- **DON'T:** Assume a component's name alone communicates its correct usage. "Modal" and "Drawer" and "Popover" can look superficially similar but have different appropriate use cases (a modal for a task that must be completed or dismissed before continuing, a drawer for supplementary content that doesn't block the main flow) — without documentation, these distinctions get flattened to "whichever one I've used before."

- **DO:** Keep documentation up to date as part of the same change that updates the component itself, treating stale documentation as a defect with the same seriousness as a stale test.

- **DON'T:** Let documentation drift out of sync with the component's actual current behavior over successive updates. Documentation that describes an old version of a component's API is worse than no documentation, because it actively misleads a consumer who trusts it.

### Avoiding Snowflake Components

- **DO:** Default to extending or composing an existing system component before creating a new one — check whether 90% of what's needed already exists as a variant, a composition of existing primitives, or a small prop addition to something already in the system.

- **DON'T:** Build a bespoke, one-off component for a single screen's specific need when it substantially overlaps an existing pattern in the system. "Just this once" custom components are how systems fragment — each individually seems justified by a real, specific need, and the aggregate cost only becomes visible once there are dozens of them.

- **DO:** Periodically audit the component library for near-duplicate components that emerged independently, and consolidate them into one shared, sufficiently flexible component, retiring the redundant ones with a clear migration path.

- **DON'T:** Let near-duplicate components persist indefinitely once discovered, each maintained separately, each drifting slightly further from the other over time. Discovering duplication and not consolidating it defeats the point of having found it.

- **DO:** Treat a genuinely novel one-off component as a deliberate decision made with awareness of its cost — a documented exception, ideally with a plan to either promote it into the system if the need turns out to be recurring, or to keep it clearly scoped to its one specific context.

- **DON'T:** Let a one-off component quietly get copy-pasted into a second, then a third location once other teams notice it exists, without ever formally promoting it into the shared system with proper documentation and variant discipline. An unofficial component that spreads through copy-paste inherits none of the review, testing, or consistency benefits an official system component gets.

### Design-to-Development Handoff Fidelity

- **DO:** Hand off finished designs with explicit token references (not just visual values) for every color, spacing, and type decision, so an engineer implementing the design can map it directly onto existing system tokens rather than reverse-engineering approximate values from a static image.

- **DON'T:** Hand off a design as a flat image or a file with only visual (non-token) values, forcing engineering to guess which existing token a given pixel value was supposed to map to, or to introduce a new hardcoded value when no exact token match is obvious.

- **DO:** Flag explicitly, at handoff time, any place where a design intentionally deviates from the existing system (a new one-off color, a custom spacing value) so it can be reviewed and either justified as a real exception or corrected to use an existing token before implementation.

- **DON'T:** Let an unintentional deviation from the system slip through handoff unnoticed, simply because the designer used a slightly-off value in their design tool and nobody caught the mismatch before it was built. Small, unflagged deviations accumulate the same way ungoverned type or spacing values do.

### Theming and White-Labeling

- **DO:** Architect the token system so an entire visual theme (a different brand's colors, a light/dark mode, a high-contrast mode) can be swapped by changing only the semantic-token layer, leaving every component's code completely unchanged.

- **DON'T:** Bake brand-specific or theme-specific values directly into component logic or styles, requiring code changes (not just token changes) to support a new theme or a white-labeled brand variant. If supporting a second brand or theme requires touching component source code, the token architecture isn't actually doing its job.

- **DO:** Test every supported theme against the same component library in the same automated or manual review pass, since a component that looks correct in the default theme can still break in a secondary theme if that theme wasn't part of the same testing loop.

- **DON'T:** Treat secondary themes (dark mode, a white-labeled brand, a high-contrast mode) as lower-priority and only spot-checked occasionally. Every supported theme is a real, live surface real users will see — an under-tested secondary theme accumulates its own separate backlog of visual bugs that the primary theme never has.

### Governance and Contribution Model

- **DO:** Define a clear ownership and contribution model for the design system — who can approve a new component or token, how a proposed addition gets reviewed, and how disagreements between a consuming team's need and the system's existing patterns get resolved.

- **DON'T:** Leave the design system ownerless, with no clear process for how changes get proposed, reviewed, or approved. An unowned system tends to accumulate whatever any individual contributor pushed in most recently, with no consistent gatekeeping for quality or consistency.

- **DO:** Make it easy and low-friction for consuming teams to propose a genuinely missing pattern, so the system evolves to meet real needs rather than teams routing around it with unofficial one-off components because the contribution process felt too slow or too opaque.

- **DON'T:** Make the design system's contribution process so slow or bureaucratic that teams default to building their own local, unofficial version of something rather than going through it. A contribution process that's harder than building a workaround guarantees the workaround wins, and the system fragments as a direct result.

### Deprecation and Migration Strategy

- **DO:** Deprecate an outdated component or token with a clear replacement path, a reasonable transition window, and tooling (lint rules, codemods) that helps consuming teams migrate, rather than removing it abruptly.

- **DON'T:** Delete or silently change a widely-used component or token with no deprecation period and no migration guidance. An abrupt breaking change to shared infrastructure used across many teams creates disproportionate downstream cost and erodes trust in the system's stability.

- **DO:** Track which components and tokens are actively used across the product (via static analysis or usage telemetry where available) before making a deprecation decision, so the decision is based on real usage data rather than assumption.

- **DON'T:** Assume a component is safe to remove or a token is safe to change simply because it looks unused or outdated to the system's maintainers. Without actual usage data, a "surely nobody uses this anymore" assumption is a common way to break a screen nobody on the systems team happened to be thinking about.

### Linting and Automated Consistency Enforcement

- **DO:** Use automated linting (a stylelint rule set, a custom ESLint rule, a design-token linter) to catch hardcoded colors, spacing values, or font sizes that bypass the token system, flagging them at code-review time rather than relying purely on manual review to catch drift.

```text
GOOD lint rule: flag any raw hex color or raw px spacing value in a
component's styles that doesn't reference a design token, and fail
the build (or warn clearly) until it's replaced or explicitly allowed.
```

- **DON'T:** Rely solely on manual code review to catch every hardcoded, off-token value introduced across a large, fast-moving codebase. Manual review is valuable but inconsistent at catching this specific, high-volume class of drift — an automated lint rule catches it reliably, every time, for free, on every single change.

- **DO:** Track adoption metrics for the design system where feasible (the percentage of components using current tokens vs. legacy hardcoded values, the number of teams actively consuming the shared library) to understand where investment in migration or documentation is actually needed.

- **DON'T:** Assume a design system is succeeding simply because it exists and has documentation, with no actual measurement of how consistently it's being adopted across the product. A system with excellent documentation that most teams have quietly stopped using isn't actually solving the consistency problem it was built for.

### Cross-Platform Token Parity

- **DO:** Keep design tokens conceptually aligned across every platform a product ships on (web, iOS, Android, and any other surface), even where the concrete unit or format necessarily differs (`rem` on web, points on iOS, `dp` on Android), so the same named token means the same design decision everywhere.

- **DON'T:** Let a token's actual value drift independently across platforms with no shared source or reconciliation process, so that "primary color" or "spacing-4" quietly means something visually different on iOS than it does on web. Cross-platform drift undermines the very reason a shared token system exists — a consistent experience across surfaces.

- **DO:** Account for genuine, legitimate platform differences (native iOS and Android each having their own well-established interaction conventions) deliberately, as documented exceptions, rather than pretending a single token set can or should produce byte-identical experiences everywhere.

- **DON'T:** Force pixel-for-pixel identical implementation across platforms with fundamentally different native conventions, at the cost of feeling foreign to users of the platform-specific norms they already know. Consistency in underlying values and intent matters more than superficial identical rendering across platforms that legitimately behave differently.

### Onboarding New Contributors to the System

- **DO:** Provide a clear, low-friction starting point for a new contributor to the design system — a "getting started" guide, example implementations, a way to ask questions — so adopting the system correctly is easier than reinventing a piece of it from scratch.

- **DON'T:** Leave new contributors to reverse-engineer the system's conventions purely by reading existing component source code with no guide, no examples, and no clear point of contact. A high-friction onboarding experience for the system itself is a direct cause of teams giving up and building their own local, inconsistent alternative.

### Versioning the Design System

- **DO:** Version the design system's component library and token package using semantic versioning (major.minor.patch), treating any visual or behavioral change that could break a consumer's existing usage as a major version bump, communicated clearly in advance.

- **DON'T:** Ship breaking visual or behavioral changes to widely-consumed components under a patch or minor version bump. A consuming team that reasonably trusts semantic versioning to protect them from unannounced breaking changes will be blindsided by a "patch" that actually changes visible behavior.

- **DO:** Maintain a changelog documenting what changed in each release, with enough detail that a consuming team can assess whether and how the change affects their own usage before upgrading.

- **DON'T:** Release updates to shared design system packages with no changelog, forcing consuming teams to diff the source themselves to understand what changed, or to upgrade blind and discover the impact only after something breaks in production.

### Design Debt and Consistency Audits

- **DO:** Periodically audit shipped product surfaces against the current design system — spot-checking real screens, not just the system's own documentation — to catch drift that accumulates gradually as individual, reasonable-seeming exceptions pile up over time.

- **DON'T:** Assume the product stays consistent with the design system indefinitely just because the system itself is well-documented and well-maintained. Documentation quality doesn't automatically translate into consistent adoption — drift happens in the actual shipped screens, which is where it needs to be checked.

- **DO:** Track design debt (hardcoded values, deprecated component usage, off-system one-offs) with the same visibility and prioritization discipline given to engineering technical debt, rather than letting it be an invisible, never-prioritized category of work.

- **DON'T:** Let design inconsistency accumulate indefinitely with no tracking or remediation plan simply because, unlike a functional bug, it rarely blocks a release on its own. Unaddressed design debt compounds the same way technical debt does, and eventually becomes expensive enough that fixing it requires a dedicated, disruptive cleanup effort instead of steady, incremental correction.

## Information Architecture & Navigation

Information architecture is the part of design users never consciously notice when it's done well, and never stop noticing when it's done badly. A user who can predict where something will be, or who always knows how to get back to where they were, is a user who trusts the product enough to stop thinking about the interface and focus on their actual task.

### Predictable Navigation Patterns

- **DO:** Keep the primary navigation's structure, position, and behavior identical across every screen in the product, so a user who has learned where the main navigation lives and how it behaves never has to relearn it on a different section of the product.

```text
GOOD: the same left sidebar, in the same position, with the same items,
      present and behaving identically whether the user is on the
      dashboard, settings, or a detail page

BAD:  the sidebar collapses to icons-only on some pages but not others,
      or is replaced by a completely different top-nav layout on one
      specific section "because that section felt different"
```

- **DON'T:** Relocate, restyle, or restructure the main navigation differently across different sections of the same product without a strong, deliberate reason. Every inconsistency in navigation forces the user to re-orient, which is a small cost paid every single time it happens across the product's lifetime.

- **DO:** Use conventional, already-learned navigation patterns — a persistent top nav, a left sidebar for a content-heavy app, a bottom tab bar for a mobile app with a handful of top-level destinations — unless there's a specific, well-justified reason the product's actual usage pattern calls for something different.

- **DON'T:** Invent a novel navigation paradigm purely for the sake of looking distinctive. Users bring years of accumulated pattern-recognition from every other product they've used; a navigation pattern that deliberately breaks those learned expectations imposes a real, ongoing learnability cost in exchange for a novelty that rarely pays for itself.

- **DO:** Keep navigation item labels and their destinations stable over time — renaming, reordering, or relocating a primary navigation item should be a deliberate, infrequent decision, not something that shifts with every release.

- **DON'T:** Reorder or relabel primary navigation items frequently based on short-term priorities. Users build a spatial memory of where things are; navigation that keeps moving forces them to keep re-searching for items they previously knew exactly where to find.

### Breadcrumbs and Wayfinding

- **DO:** Provide breadcrumbs, a persistent section indicator, or an equivalent "you are here" signal for any hierarchy three or more levels deep, so a user can see their current location within the broader structure at a glance and navigate to any ancestor level directly.

```text
GOOD: Home  >  Settings  >  Team  >  Permissions
      (each segment is clickable and jumps directly to that level)
```

- **DON'T:** Strand a user on a deeply nested page with no indication of where it sits in the broader hierarchy and no quick way to jump back up multiple levels at once, other than repeatedly pressing a browser "back" button that may not even map cleanly onto the logical hierarchy.

- **DO:** Make every breadcrumb segment except the current page a working, clickable link that navigates directly to that level, so breadcrumbs function as real navigation, not just a decorative location label.

- **DON'T:** Render a breadcrumb trail where only the final (current) segment is meaningful and every earlier segment is inert, non-clickable text. A breadcrumb that can't actually be used to navigate provides only half its value.

- **DO:** Keep the current page clearly, visually distinguished within the breadcrumb trail (often unstyled/non-link text, since it's not a valid destination to click on itself) so it's unambiguous which segment represents "here."

- **DON'T:** Style the current page's breadcrumb segment identically to the clickable ancestor segments, inviting a click that either does nothing or triggers a confusing self-navigation.

### Progressive Disclosure

- **DO:** Surface the most commonly needed options and information by default, and deliberately tuck less common, advanced, or rarely-needed options behind a clearly labeled "More," "Advanced," or expandable section, so a new or casual user isn't confronted with the full complexity of the system on first contact.

```text
GOOD (progressive disclosure):
  [ Basic filters: Status, Date range ]
  [ ▸ Advanced filters (12 more options) ]

BAD (everything exposed at once):
  20 filter fields shown flat, all at equal visual weight,
  regardless of how rarely most of them are actually used
```

- **DON'T:** Present every possible option, setting, or filter flat on one screen at equal visual weight regardless of how frequently each is actually used. Overwhelming a first-time or casual user with the full surface area of a power-user feature set is one of the most reliable ways to make a capable product feel intimidating or confusing.

- **DO:** Reveal complexity gradually, in response to the user demonstrating a need for it — an "Advanced" section that expands on click, a setting that only appears once a related prerequisite is configured, a power-user shortcut that's discoverable but not forced on everyone.

- **DON'T:** Force every user, regardless of experience level or actual need, to parse an expert-level control panel just to complete a basic, common task. If 90% of users only ever need 3 of 20 available options, those 3 should be what's immediately visible.

- **DO:** Make progressively-disclosed content easy to find once a user does need it — a clearly labeled expand control, a logical location, a search function that surfaces buried options — so progressive disclosure doesn't become "advanced features that are effectively unreachable."

- **DON'T:** Bury genuinely necessary functionality so deeply behind progressive disclosure that experienced or power users can't efficiently reach it. Progressive disclosure should reduce initial complexity without permanently punishing the users who do need the deeper functionality regularly.

### Consistent Primary Action Placement

- **DO:** Place the primary action for a given screen type (Save, Continue, Publish, Submit) in the same relative position across every equivalent screen and dialog in the product, so a user's muscle memory for "where the main button is" transfers correctly from one screen to the next.

- **DON'T:** Swap a primary action's position (left vs. right, top vs. bottom) between similar screens or dialogs without a systemic reason. Inconsistent primary-action placement increases misclicks, particularly dangerous when a destructive action occupies the position a different dialog uses for the safe, primary one.

- **DO:** Keep destructive and primary/safe actions visually and spatially distinct — different color, different visual weight, and enough separating space — so a user under time pressure or acting from habit doesn't accidentally trigger the wrong one.

- **DON'T:** Place a destructive action ("Delete") directly adjacent to a primary action ("Save") with identical visual styling and no separating space or visual distinction. This is a well-known, entirely avoidable source of costly accidental clicks, especially for actions a user performs frequently enough to act on partly from muscle memory.

### Search as a Navigation Pattern

- **DO:** Treat in-product search as a first-class navigation method, not a fallback, for any product with enough content or functionality that browsing a hierarchy alone becomes inefficient — with fast, forgiving matching (typo tolerance, partial matches) and results that clearly indicate where each result lives within the broader structure.

- **DON'T:** Bury or under-invest in search for a content- or feature-rich product, forcing users to navigate multiple levels of hierarchy to find something they could have found instantly by name. When browsing a hierarchy takes meaningfully longer than typing a query would, search needs to be a prominent, well-supported option, not an afterthought.

- **DO:** Show enough context in each search result (its location in the hierarchy, a snippet of matching content, its type) that a user can judge relevance and pick the right result without needing to open several before finding the one they meant.

- **DON'T:** Return a flat, context-free list of matching titles with no indication of where each result lives or what kind of content it is. Context-free results force the user to open multiple candidates just to identify the one they were actually looking for.

### Mobile Navigation Patterns

- **DO:** Choose a mobile navigation pattern appropriate to the number of top-level destinations — a bottom tab bar for roughly 3–5 frequently used top-level sections, a hamburger/drawer menu for a longer or less frequently accessed list — based on actual usage frequency, not on which pattern looks more current.

- **DON'T:** Default to a hamburger menu for a small number of frequently-used top-level destinations that would be more discoverable and faster to reach as persistent, always-visible tab bar items. Hiding frequently used navigation behind an extra tap measurably reduces how often users engage with what's hidden there.

- **DO:** Keep the most important, most frequently used action within comfortable one-handed thumb reach on mobile — generally the lower half of the screen — since that's where a phone is most naturally and stably operated.

- **DON'T:** Place a primary, frequently-tapped mobile action at the very top of a tall screen, where it requires either a stretch or a hand repositioning to reach one-handed. Ergonomics on mobile is a navigation and layout concern, not purely a visual-design one.

### Onboarding and First-Run Navigation

- **DO:** Design a first-run experience that orients a new user toward their first meaningful action quickly, showing only what's needed to get started rather than the full breadth of the product's navigation and options all at once.

- **DON'T:** Drop a brand-new user directly into the full, unmodified interface with every navigation option, setting, and feature visible and no guidance on where to start. A first-run experience with no onboarding path leaves new users to guess at a mental model of the product's structure through trial and error alone.

- **DO:** Make onboarding skippable and revisitable for users who want to explore on their own or who want to review guidance again later, rather than forcing a linear, un-skippable tour on every user regardless of their prior familiarity.

- **DON'T:** Force every user, including returning ones or those already familiar with similar products, through a mandatory, un-skippable onboarding sequence with no way to exit early. Onboarding that can't be skipped punishes exactly the users who need it least.

### URL Structure and Page Titles as Information Architecture

- **DO:** Design URL structure to reflect the actual information hierarchy in a predictable, human-readable way (`/projects/acme/settings/permissions`), so URLs themselves function as a form of wayfinding and are shareable, bookmarkable, and guessable.

- **DON'T:** Use opaque, non-hierarchical URLs (`/view?id=48213&t=2`) for content that has a real, meaningful place in the product's structure. Opaque URLs can't be reasoned about, shared meaningfully, or manually navigated by editing the address bar — a real usability loss for power users and a missed information-architecture signal for everyone else.

- **DO:** Set descriptive, accurate page titles that reflect both the current page's specific content and its place in the product (e.g., "Permissions — Acme Team Settings"), since the browser tab title is itself a navigation and orientation aid, especially with many tabs open.

- **DON'T:** Leave every page in the product with an identical or generic browser tab title ("Dashboard," "App," the product's name repeated on every page with no page-specific context). A user with several tabs open has no way to distinguish them at a glance if every title looks the same.

### Dashboard and Home Screen Design

- **DO:** Design a product's home screen or dashboard around the specific tasks and information the majority of users need most often, rather than treating it as a catch-all summary of every feature the product offers.

- **DON'T:** Turn a dashboard into an unprioritized grid of every available widget, metric, and shortcut the product could theoretically show. A dashboard that tries to represent everything equally ends up helping a user find nothing quickly, since nothing has been prioritized over anything else.

- **DO:** Allow reasonable personalization or configurability of a dashboard's content for products where different user roles genuinely need to see different information, so a dashboard can stay relevant across a diverse user base without becoming generic for everyone.

- **DON'T:** Ship a rigid, one-size-fits-all dashboard for a product whose user base has meaningfully different roles and needs (e.g., an admin vs. a contributor), forcing every role to scroll past irrelevant sections to find what actually matters to them.

- **DO:** Surface the most time-sensitive or action-required information prominently on a dashboard (things awaiting the user's response, upcoming deadlines, anomalies needing attention), since a dashboard's highest-value job is often directing attention to what needs it most right now.

- **DON'T:** Give routine, non-actionable status information the same visual prominence as genuinely time-sensitive items needing a response. If everything on the dashboard looks equally urgent, the user has no way to triage what to look at first.

### Filtering, Sorting, and Bulk Actions

- **DO:** Make active filters and sort order clearly visible at all times (not just at the moment they're set), so a user scanning a filtered or sorted list always understands why they're seeing what they're seeing.

- **DON'T:** Apply a filter or sort silently, with no persistent visible indicator once the selection menu closes. A user who forgets a filter is active can easily misinterpret an incomplete, filtered view as the full data set — a mistake with real consequences in contexts like inventory, financial data, or search results.

- **DO:** Provide a clear, one-action way to reset all active filters back to a default/unfiltered state, prominently placed wherever filters are shown.

- **DON'T:** Force a user to manually undo several individually-applied filters one at a time to get back to an unfiltered view. A single, obvious "clear all filters" action removes an unnecessary, repetitive chore.

- **DO:** Support bulk actions (select multiple, then act once) for any list where users commonly need to perform the same action on several items, with a clear count of how many items are currently selected and an easy way to select/deselect all.

- **DON'T:** Force a user to repeat a multi-step action individually for each of many items when a bulk operation would be the obvious, expected solution. Forcing tedious repetition for a common batch task is a significant, avoidable efficiency cost for frequent users.

### Multi-Tenancy and Context Switching

- **DO:** Make the current context (organization, workspace, account, environment) persistently and clearly visible wherever a user could plausibly have more than one, and make switching between contexts fast, discoverable, and low-risk.

- **DON'T:** Leave the active organization, workspace, or environment ambiguous or only discoverable by digging into a settings page. A user who can't easily tell which context they're currently acting in risks making a change in the wrong one — creating a resource in the wrong workspace, or worse, in a production environment when they intended a test one.

- **DO:** Visually differentiate high-consequence contexts (a production environment vs. a staging/sandbox one) clearly and persistently, using a distinct, hard-to-miss visual treatment, since the cost of an accidental action taken in the wrong context can be severe.

- **DON'T:** Style a production and a non-production context identically, relying only on a small text label to distinguish them. A visually identical environment indicator is far too easy to overlook in the middle of a routine, fast-paced task, precisely when the consequences of a mistake are highest.

### Global vs. Contextual Navigation

- **DO:** Clearly separate global, product-wide navigation (present and identical everywhere) from contextual, section-specific navigation (relevant only within a particular area), so users can distinguish "where can I go from anywhere" from "what's available to me right here."

- **DON'T:** Blend global and contextual navigation items together into one undifferentiated list with no visual or structural distinction. When a user can't tell which nav items are always available and which are specific to their current section, the navigation structure itself stops communicating the product's actual information hierarchy.

- **DO:** Keep the number of top-level global navigation destinations small (rule-of-thumb: comfortably scannable at a glance, commonly under seven to nine items) so the global nav remains quickly scannable rather than becoming its own long list a user has to search through.

- **DON'T:** Let the global navigation grow unboundedly as new features are added, each new feature getting its own top-level global nav entry indefinitely. An ever-growing flat list of top-level items eventually defeats the purpose of having a small, memorable global navigation at all — new destinations belong nested under an existing section, or behind progressive disclosure, once the top level starts to feel crowded.

### Settings and Preferences Organization

- **DO:** Group settings by the user's mental model of what they're configuring (account, notifications, privacy, billing) rather than by internal implementation boundaries, and order groups by how frequently users actually need to reach each one.

- **DON'T:** Organize a settings area around internal system or database structure rather than how a user actually thinks about their own preferences. A settings page organized around implementation details forces users to guess which internal category their desired change might fall under.

- **DO:** Make a specific, commonly-sought setting findable via in-product search where the settings area is large, rather than requiring the user to know or guess which category it's filed under.

- **DON'T:** Force users to manually browse a large, deeply-nested settings hierarchy with no search capability to find one specific, commonly-needed toggle. A large settings area with no search is a common, avoidable source of support requests along the lines of "where do I turn off X."

### Error and 404 Page Navigation

- **DO:** Give a 404 or error page a clear explanation of what happened and functional navigation options to recover — a link back to the homepage, a search box, links to popular or likely-intended destinations — so a broken link doesn't strand the user in a dead end.

- **DON'T:** Show a bare, generic "404 Not Found" or a raw server error page with no navigation options and no way to recover other than the browser's back button. An error page with no path forward is exactly the same class of dead-end failure as a form error message with no path forward, covered under Content Design & Microcopy — the fix (give the user somewhere to go next) is the same principle applied to a different context.

### Deep Linking and Shareable State

- **DO:** Reflect meaningful application state (the active filters, the selected tab, the current search query, the item currently open in a detail view) in the URL, so a user can bookmark, share, or reload that exact state and land back exactly where they were.

- **DON'T:** Keep significant navigational or filter state purely in client-side memory with no URL reflection. A user who shares a link to what they're looking at, or simply refreshes the page, loses all of that state and lands back at a generic default view — a real, frequently-felt cost in any collaborative or reference-heavy product.

- **DO:** Keep deep-linked URLs stable over time wherever reasonably possible, since a bookmarked or shared link that stops working after a refactor breaks trust in linking as a reliable way to reference something in the product.

- **DON'T:** Restructure URL patterns casually during a refactor with no redirect strategy for existing links. Old links breaking silently — a bookmark, a link shared in a support ticket or a chat months ago — is a real cost even though it's easy to overlook during the refactor itself.

### Tabs vs. Separate Pages for Content Organization

- **DO:** Use in-page tabs for closely related views of the same underlying entity where a user is likely to compare or quickly switch between them (an item's "Details" and "Activity" tabs), and use fully separate pages for content that's more independent or that benefits from its own dedicated URL, title, and deep-linkability.

- **DON'T:** Default to tabs purely as a way to avoid building separate pages, even for content that's substantial enough, independent enough, or important enough to deserve its own addressable URL and page title. Content that users would reasonably want to bookmark, share, or find via search benefits from being a real page, not a tab state hidden behind client-side JavaScript.

- **DO:** Reflect the active tab in the URL (as a query parameter or path segment) so a specific tab remains deep-linkable and survives a page reload, applying the same deep-linking principle to tab state as to any other meaningful application state.

- **DON'T:** Implement tabs with purely client-side state that resets to the first tab on every page reload or direct navigation. A user who reloads the page or follows a link expecting to land on a specific tab, only to be dropped back at the default, experiences this as the product forgetting where they were.

### Contextual Help and In-App Support Placement

- **DO:** Place help and support access (documentation links, a chat widget, a contact option) consistently in the same location across the product, and where feasible make it contextual — surfacing help content relevant to the specific screen the user is currently on rather than always linking to a generic help homepage.

- **DON'T:** Scatter help and support entry points inconsistently across different screens, or always route every "Help" click to the same generic top-level help homepage regardless of what the user was actually doing when they clicked it. A user who clicks for help while stuck on a specific task benefits far more from being taken directly to relevant guidance than from being handed a full documentation site to search through themselves.

- **DO:** Keep a persistent support/help affordance from becoming visually intrusive or covering important content, particularly on smaller viewports where a fixed-position chat widget can obscure primary actions or content.

- **DON'T:** Let a persistent help or chat widget overlap critical interface elements (a form's submit button, a primary navigation item) with no way to dismiss or reposition it. A support tool that gets in the way of the task the user is actually trying to complete works against its own purpose.

### Notification Centers and Activity Feeds as Navigation

- **DO:** Treat an in-product notification center or activity feed as a navigation surface in its own right — each entry should link directly to the relevant object or location, not merely describe an event with no way to act on or investigate it further.

- **DON'T:** Show notification entries as inert, non-clickable text summaries with no path to the actual content or object they reference. A notification that can't be acted on or navigated from forces the user to manually go find whatever it was describing, defeating much of the point of having been notified in the first place.

- **DO:** Clearly distinguish read from unread notifications, provide an easy way to mark items as read (individually and in bulk), and keep the unread count accurate and in sync with what the user has actually seen.

- **DON'T:** Let a notification's read/unread state desync from what the user has actually viewed — a persistent unread badge that doesn't clear after the user has already seen and acted on the underlying item is a small but recurring source of confusion and distrust in the indicator.

### Command Palettes and Keyboard-Driven Navigation

- **DO:** Provide a searchable command palette (typically triggered with a shortcut like Cmd/Ctrl+K) for products with enough features, pages, or actions that clicking through a menu hierarchy becomes slow for a frequent, keyboard-comfortable user, letting them jump directly to any page or trigger any action by typing its name.

- **DON'T:** Force power users to navigate a deep menu hierarchy by mouse for every action, with no faster keyboard-driven alternative, in a sufficiently complex product. A command palette is one of the highest-leverage additions for frequent users' efficiency once a product has grown past a small, simple set of destinations.

- **DO:** Keep a command palette's results fast, fuzzy-matched, and ranked sensibly (recent or frequent actions weighted higher), and make its existence discoverable — a visible search bar that expands into the palette, a hint in a menu — rather than a shortcut only findable by insiders.

- **DON'T:** Ship a command palette as an undocumented, undiscoverable power-user secret with no visible entry point in the interface. A feature that only existing keyboard-shortcut experts can find fails to deliver value to everyone else who would benefit from discovering it.

### Related Content and Cross-Linking

- **DO:** Surface genuinely relevant related content or cross-links ("see also," related items, next steps) at natural points in a user's journey, based on real relationships in the content or data, so users can discover connected information without having to already know it exists and search for it directly.

- **DON'T:** Populate a "related content" section with weak, generic, or purely popularity-based suggestions that have no real relationship to what the user is currently looking at. Irrelevant related-content suggestions train users to ignore that section entirely, which wastes the space for the times a genuinely relevant connection could have been surfaced.

- **DO:** Place cross-links at the point in the content or flow where they're actually useful — inline where a concept is first mentioned, or at the natural end of a task — rather than dumping every possibly-related link into one undifferentiated list.

- **DON'T:** Front-load every conceivable related link into a single generic block disconnected from where each one would actually be relevant in the user's current context. Contextual placement, matched to where a specific piece of related information would actually help, does far more for discoverability than an exhaustive but undifferentiated list.

## Quick Checklist
- A real type scale (modular ratio) is defined and used consistently — not arbitrary one-off font sizes.
- Body text line-height and measure (line length) are set for readability, not left at defaults.
- Body text meets at least 4.5:1 contrast; large text/UI components meet at least 3:1 — checked, not assumed.
- Color is never the only signal for meaning (status, error, required field) — paired with an icon, label, or pattern.
- Light and dark themes are each designed with real hierarchy, not one inverted onto the other.
- Spacing follows a consistent scale (e.g., 4pt/8pt grid) rather than arbitrary pixel values per component.
- Alignment is deliberate — edges line up; nothing floats at a random offset from its neighbors.
- Every interactive element has defined hover/focus/active/disabled/loading/error/empty states.
- Focus indicators are visible and never removed without a compliant custom replacement.
- Touch targets meet the ~44×44pt / 48×48dp minimum.
- Form fields have persistent visible labels, inline validation, and specific, actionable error messages.
- Motion has a functional purpose (state change, spatial relationship) and respects `prefers-reduced-motion`.
- Icons are drawn from one consistent style/weight/grid, and unclear icon-only actions carry a text label.
- Imagery is purposeful and specific to the context, not generic filler stock photography.
- Design tokens (spacing, color, type) are the single source of truth — no hardcoded one-off values bypassing them.
- Components have documented purpose, variants, and states rather than being one-off "snowflake" builds.
- Navigation is predictable and consistently placed; deep hierarchies have real wayfinding (breadcrumbs, clear back paths).
- Progressive disclosure is used for advanced/rare options instead of surfacing every option at once.
- Related-content or cross-link sections contain genuinely relevant items, placed where they're contextually useful — not a generic dump.
- Empty states, loading states, and error states are designed, not left as a blank screen or a raw stack trace.
