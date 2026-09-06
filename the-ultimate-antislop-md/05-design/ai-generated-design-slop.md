# AI-Generated Design & UI "Slop" Tells

By 2025, enough software had been scaffolded, prototyped, or fully built by AI coding assistants that a shared visual dialect emerged across totally unrelated products — different companies, different industries, different stated brand values, yet the same typeface, the same purple gradient, the same centered hero with a pill-shaped badge floating above the headline. None of these patterns are wrong in isolation. Inter is a fine typeface. A purple-to-blue gradient is not inherently ugly. Rounded corners are not a crime. What makes this "slop" is the *absence of a decision* — the pattern shows up not because it was chosen for this product, this audience, this brand, but because it is the highest-probability output of a model (or a human copying a model) reaching for "professional-looking UI" with no other constraint. The tell isn't the ingredient, it's the lack of a reason for the ingredient.

This section catalogs the specific, recognizable patterns that mark an interface as generated-not-designed, explains the mechanism that produces each one (why a model or a rushed developer converges on it), and gives a concrete, testable fix. The goal is not to ban gradients, rounded corners, or Inter forever — it's to make every visual choice traceable to an actual reason: a brand attribute, a content constraint, a usability requirement. An interface that happens to use a purple gradient because the brand's actual identity is purple is not slop. An interface that uses that exact gradient because it's what showed up when nobody specified anything is.

Two audiences read this part differently. A human designer or developer can use it as a self-audit before shipping. An AI coding assistant should read it as a set of default behaviors to actively override — because unless a prompt says otherwise, these are precisely the patterns a language model's own training distribution will pull it toward, and "the user didn't say not to" is not a justification for defaulting to the most generic possible answer.

## Typography Tells

Typography is often the fastest way to spot AI-generated or template-default UI, because type choices are cheap to leave at their defaults and expensive to get intentionally right. A model (or a developer moving fast) has to make an active decision to deviate from "whatever the framework or the model's own training distribution suggests first," and most don't.

### The Inter/Roboto/Arial Default

Inter, Roboto, and system-default Arial/Helvetica dominate AI-assisted and template-driven UI because they are the *statistically safest* choice: they're free, they're the default in Figma, Tailwind, shadcn/ui, and Google Fonts' most-copied snippets, and they were almost certainly the most frequently seen typeface in whatever training or reference corpus produced the pattern. That safety is exactly why they read as generic — when literally every SaaS landing page, dashboard, and marketing site uses the same face, the face stops communicating anything about the product wearing it.

This isn't an argument that Inter is a bad typeface — it's a well-drawn, highly legible UI face and there are legitimate products (developer tools, data-dense dashboards, anything prioritizing neutrality over personality) where it's the *right* choice, chosen on purpose. The tell is when it appears with zero evidence of a decision: no fallback stack tuned to the brand, no weight variation beyond regular/bold, no pairing with anything else, same as ten thousand other products built the same week.

```css
/* Generic: whatever the framework scaffolded, untouched */
body {
  font-family: Inter, sans-serif;
}

/* Intentional: a real decision, documented, with a considered fallback stack */
body {
  /* Chosen for warmth + a slightly humanist skew vs. the geometric-sans default;
     fallback stack picked to degrade gracefully on systems without Sentient. */
  font-family: "Sentient", "Iowan Old Style", Georgia, serif;
}
```

- **DO:** Pick a primary typeface as a deliberate response to the product's tone, then write down the one-sentence reason. If the honest reason is "it's fast and neutral and this is an internal tool," that's a valid reason — but it should be a reason, not a non-decision.
- **DON'T:** Ship Inter, Roboto, or Arial as the sole typeface just because it's what the scaffold, the AI tool, or the design system's default theme shipped with. Left unquestioned, the default becomes the product's entire typographic identity by accident, not by design.
- **DO:** Maintain a short list of 2-3 alternate typefaces you've actually evaluated against the brand (a grotesque, a humanist sans, maybe a serif for editorial contexts) so "not Inter" doesn't just become "the next most popular default" (Söhne, General Sans, Geist) chosen for the same lack of reason.
- **DON'T:** Treat "switch to a different popular font" as the fix on its own. Swapping Inter for Geist or Switzer with the same zero-reasoning process just replaces one generic default with a slightly less common one — the underlying problem (no decision was made) is unchanged.

### The "Trendy Pairing" Autopilot

A specific micro-trend compounds the typeface-default problem: a geometric or grotesque sans for body/UI text, paired with a display serif (often italicized) for one or two accent words or the hero headline. This pairing — think a clean grotesque next to something like Fraunces, Canela, or a similarly high-contrast editorial serif — became so common in 2023-2025 AI-assisted and template-driven design that it now reads as a genre marker rather than a design choice, the same way stock photography of diverse people laughing at salad once did.

The pairing isn't bad typography — sans/serif contrast is a legitimate, time-tested hierarchy technique. What makes it slop is that it gets applied as a reflex, disconnected from whether the brand has any actual editorial, artisanal, or "premium craft" positioning that a display serif is supposed to signal. A fintech dashboard, a dev tool, and a boutique candle brand should not reach for the same visual vocabulary to say completely different things about themselves.

```html
<!-- Generic: the "trendy pairing" applied with no connection to brand voice -->
<h1 class="sans">Build faster with <em class="serif">AICore</em></h1>

<!-- Intentional: serif accent only where the brand narrative earns it -->
<!-- (e.g., an actual editorial/craft brand, used consistently, not just once) -->
<h1 class="grotesque">Every batch, roasted by hand</h1>
<p class="serif-italic">— our founder, on why we still do it this way</p>
```

- **DO:** Reserve a display serif or script accent for products whose actual brand story involves craft, heritage, editorial voice, or human touch — and use it consistently across the product, not as a one-off flourish on the hero headline alone.
- **DON'T:** Apply a serif-italic accent word purely because it "looks premium." If the rest of the brand voice is technical, transactional, or utilitarian, a lone italic serif word reads as a costume, not a personality — pick a typographic hierarchy tool (weight, size, color) that's consistent with the rest of the product's voice instead.
- **DO:** When you do pair two typefaces, define the *rule* for when each is used (all body copy in the sans, only H1s and pull-quotes in the serif, never mixed within one line except the specific accent-word pattern) so the pairing reads as a system, not a decoration applied inconsistently page to page.
- **DON'T:** Let the pairing be the *only* differentiator between your product and every other AI-scaffolded landing page from the same season. If ten competitors are all doing "grotesque sans + italic serif accent," matching them isn't establishing a brand, it's blending into a trend cycle that will look dated within a year or two.

### The Serif Italic Accent Word

A narrower, more specific version of the pairing problem deserves its own callout because it's so recognizable on sight: a single word in a headline — usually a verb or an evocative noun — rendered in an italic serif while everything else on the page is a plain sans-serif. "Ship *beautifully*." "The future of *work*." It's become such a reliable AI/template tell that design communities can identify a generated landing page from a screenshot with no other context, purely from this one visual habit.

The mechanism is straightforward: it's an easy way to inject "editorial sophistication" into an otherwise flat page with a single CSS change, and it shows up constantly in the design references and Dribbble shots that both human copyists and AI training data draw from. Used once, thoughtfully, tied to actual meaning, it can work. Used as a reflex on every generated hero section, it becomes wallpaper.

```html
<!-- Generic: the reflexive italic-serif-accent-word pattern -->
<h1>Ship <em style="font-family: Georgia, serif;">beautifully</em>.</h1>

<!-- Intentional: emphasis achieved through the brand's own type system,
     and only where the word actually carries the sentence's meaning -->
<h1>Ship without the <strong class="brand-accent-weight">3am rollback</strong>.</h1>
```

- **DO:** If you use an italic serif accent, make it a rare, load-bearing device — one that appears a handful of times across the whole product where a word genuinely needs a different register, not a per-page reflex.
- **DON'T:** Apply the italic-serif treatment to whichever word feels "important" in every headline you write. If it happens on nearly every page, it isn't emphasis anymore — it's just how the font renders, and the reader stops perceiving it as meaningful.
- **DO:** Build emphasis primarily from your existing type scale (weight, size, color, tracking) so it stays consistent with the rest of the system, and treat a second typeface as a deliberate, occasional exception rather than the default emphasis mechanism.

### Flat Type Scale (No Real Hierarchy)

A subtler but more damaging tell than font choice is the absence of a real type scale. Generated and rushed UI frequently uses two or three font sizes for the entire interface — a big one for headings, a slightly smaller one for "body," and everything else defaults to whatever the browser or component library gives it. The result is a page where an H1, an H3, and a card title are barely distinguishable, or where a tiny caption and a paragraph both render at 14px because nobody defined intermediate steps.

This happens because a type scale is invisible infrastructure — it doesn't show up as a single obviously "generated" element the way a purple gradient does, so it gets skipped when someone is optimizing for a good-looking screenshot rather than a system. It's also a common failure mode of literal, unplanned prompting: asking for "a heading" and "some text" produces exactly two sizes, because that's exactly what was asked for.

```css
/* Generic: two sizes, no defined relationship between them */
h1 { font-size: 28px; font-weight: 700; }
p  { font-size: 16px; font-weight: 400; }
/* h2, h3, small print, captions, labels all inherit body size by accident */

/* Intentional: a defined scale with a clear ratio and named steps */
:root {
  --text-xs:   0.75rem;  /* 12px — captions, metadata */
  --text-sm:   0.875rem; /* 14px — secondary body, labels */
  --text-base: 1rem;     /* 16px — body */
  --text-lg:   1.25rem;  /* 20px — subheadings, lead paragraphs */
  --text-xl:   1.75rem;  /* 28px — section headings */
  --text-2xl:  2.5rem;   /* 40px — page/hero headings */
}
h1 { font-size: var(--text-2xl); font-weight: 700; line-height: 1.1; }
h2 { font-size: var(--text-xl);  font-weight: 600; line-height: 1.2; }
small, .caption { font-size: var(--text-xs); font-weight: 500; color: var(--text-muted); }
```

- **DO:** Define a named type scale (5-7 steps is usually enough) with a deliberate ratio between steps — a modular scale (1.125, 1.25, 1.333, etc.) or a hand-tuned set — before writing a single component, and reference the scale's tokens everywhere instead of one-off pixel values.
- **DON'T:** Let every heading level and body variant converge on the same two or three font sizes because nobody defined the steps in between. If an H2 and an H4 are only distinguishable by weight, the page has no real hierarchy, just varying degrees of bold.
- **DO:** Pair size steps with matching line-height and letter-spacing adjustments — larger text generally wants tighter line-height and sometimes tighter tracking, while small text needs more breathing room to stay legible. A scale that only changes `font-size` and leaves `line-height: 1.5` everywhere will still look flat even with more steps defined.
- **DON'T:** Treat "make the heading bigger" as equivalent to "create hierarchy." Weight, color/contrast, spacing above and below, and max-width for line length all contribute to perceived hierarchy — relying on size alone produces headlines that are technically larger but don't actually organize the page.
## Color Tells

Color tells are often the single fastest way to identify AI-generated UI, because color is the visual layer most influenced by whatever aesthetic dominated the training data and default component themes of the 2023-2025 era: dark backgrounds, purple-to-blue gradients, and glowing accent colors that read as "futuristic AI product" regardless of what the product actually does.

### The "AI Purple" Gradient

If there is one single visual signature most associated with "this was probably built with AI assistance," it's a specific gradient family: violet or indigo blending into blue or magenta, usually diagonal, usually applied to a hero background, a CTA button, or as a text-fill effect on a headline. It became the default "tech/AI product" palette so quickly and so thoroughly — echoing the branding of several prominent AI companies and the countless AI-wrapper products that copied them — that by 2025 it functioned as an unintentional genre signifier: seeing it primes a viewer to think "this is probably a thin AI wrapper" before reading a single word of copy.

The mechanism is a feedback loop: early AI-product branding used purple/indigo gradients, template marketplaces and component libraries copied that palette because it "looked like AI," AI coding tools trained partly on that generation of templates reproduced it as a default, and now it's so overrepresented that it actively undermines credibility — it signals derivative effort rather than a real brand decision, even for products that aren't AI tools at all.

```css
/* Generic: the instantly-recognizable "AI purple" default */
.hero {
  background: linear-gradient(135deg, #6366f1 0%, #8b5cf6 50%, #d946ef 100%);
}
.cta-button {
  background: linear-gradient(90deg, #7c3aed, #6366f1);
}

/* Intentional: a palette derived from an actual brand color, used flat,
   with the gradient (if any) reserved for a single deliberate moment */
.hero {
  background: var(--brand-surface); /* a flat, brand-derived tone */
}
.cta-button {
  background: var(--brand-primary); /* one solid color, not a gradient */
}
```

- **DO:** Derive the palette from an actual brand attribute — an existing logo color, a physical product's material, a competitor-differentiation decision — and be able to state in one sentence why this hue and not another.
- **DON'T:** Default to violet/indigo/magenta gradients simply because they read as "modern tech product." Unless the brand has a specific, considered reason to sit in that hue family, this palette choice actively signals derivative, unconsidered work rather than the "cutting-edge" feeling it's aiming for.
- **DO:** If purple genuinely is the right brand color (it can be — plenty of real, well-considered brands use it), commit to a specific shade and use it consistently and flatly rather than as a shifting multi-stop gradient that changes hue every time it's applied.
- **DON'T:** Assume "not purple" alone is the fix. Swapping to a teal-to-blue or orange-to-pink gradient with the same undirected process just relocates the same problem to a different hue family — the issue is the absence of a brand-derived reason, not the specific color.

### Gradient Overuse

Beyond the specific purple family, generated UI shows a broader pattern of applying gradients reflexively to nearly every surface that could theoretically hold one: button backgrounds, card borders, section backgrounds, and — especially telling — text itself, via `background-clip: text` headline treatments. Each individual instance can look polished in isolation; the tell is the density. When gradients appear on the hero background, the CTA button, three feature icons, and the headline text all on one screen, none of them read as a considered accent anymore — they read as a texture applied uniformly because it was available.

Gradients are cheap to add and instantly make a flat design feel less static, which is exactly why they get overused: a single line of CSS turns a plain button into something that looks "designed" at a glance, with no need to think about spacing, type, or content quality. That shortcut is also why experienced designers use gradients sparingly — reserving them as one deliberate accent makes that accent mean something, while distributing them everywhere flattens the hierarchy they were supposed to create.

```css
/* Generic: gradient applied to nearly everything on the screen */
.hero { background: linear-gradient(135deg, #4f46e5, #9333ea); }
.card { border: 1px solid transparent; background-image: linear-gradient(#fff,#fff), linear-gradient(135deg,#4f46e5,#9333ea); background-origin: border-box; }
.btn { background: linear-gradient(90deg, #4f46e5, #9333ea); }
.headline { background: linear-gradient(90deg,#4f46e5,#9333ea); -webkit-background-clip: text; color: transparent; }

/* Intentional: one gradient moment, everything else flat and calm around it */
.hero { background: var(--surface-1); }
.card { border: 1px solid var(--border-subtle); background: var(--surface-1); }
.btn-primary { background: var(--brand-primary); }
.btn-primary--featured { background: linear-gradient(90deg, var(--brand-primary), var(--brand-accent)); } /* the one deliberate exception, used on a single hero CTA only */
```

- **DO:** Pick at most one place on a given screen where a gradient earns its keep — usually a single hero moment or a single primary CTA — and keep every other surface flat.
- **DON'T:** Apply gradients to buttons, cards, borders, backgrounds, and headline text all within the same view. Stacking gradients everywhere cancels out the very sense of emphasis a gradient is supposed to create, and it reads as decoration-by-default rather than an intentional highlight.
- **DO:** Test whether a flat color communicates the same hierarchy before reaching for a gradient. If a solid brand color does the job, the gradient wasn't necessary — it was just easier to reach for.
- **DON'T:** Use a gradient text-fill on a headline as a default "make it pop" move. Gradient text often reduces legibility (especially at small sizes or on busy backgrounds) and is one of the single most-flagged AI-generated-landing-page tells in circulation — treat it as a specific, rare device, not a default headline treatment.

### Default Dark Mode with All-Caps Labels and Low-Contrast Grey Body Text

A recurring aesthetic cluster shows up specifically in AI-generated dashboards and product UIs: a near-black background, small all-caps labels with wide letter-spacing (often on section headers, table columns, or metadata), and body copy set in a medium grey that looks sleek in a screenshot but fails basic contrast requirements against that dark background. This combination reads as "premium developer tool" at a glance — it's the aesthetic of a lot of well-known dev-tool dashboards — but it gets reproduced without the contrast discipline that made the originals actually legible.

The all-caps-label habit specifically comes from wanting a "technical, systematic" feel cheaply: uppercase with tracked-out letter-spacing looks like a design decision even when the underlying hierarchy is thin. The low-contrast grey body text happens because pure white body text on black looks harsh in isolation, so text gets dimmed down — but frequently past the point of WCAG AA compliance (4.5:1 for normal text), especially once it's dimmed *and* set in a light font-weight *and* rendered small.

```css
/* Generic: looks sleek in a screenshot, fails contrast in practice */
body { background: #0a0a0a; color: #6b7280; } /* ~3.5:1 on #0a0a0a — fails AA for normal text */
.section-label { text-transform: uppercase; letter-spacing: 0.1em; font-size: 11px; color: #52525b; }

/* Intentional: dark mode with a body color that actually clears AA contrast */
body { background: #0a0a0a; color: #d4d4d8; } /* ~13:1 — comfortably passes AA/AAA */
.text-secondary { color: #a1a1aa; } /* ~7.5:1 — still passes AA at normal text sizes */
.section-label { text-transform: uppercase; letter-spacing: 0.08em; font-size: 12px; font-weight: 600; color: #a1a1aa; }
```

- **DO:** Check every text/background pairing in dark mode against actual WCAG contrast ratios (4.5:1 for normal text, 3:1 for large text/18px+bold) using a contrast checker, not by eye — dimmed grey-on-black is deceptively easy to misjudge because it looks fine on a bright calibrated monitor and fails on dimmer or lower-contrast displays.
- **DON'T:** Dim body text to a medium grey on a near-black background purely for a "premium, muted" look without verifying contrast. This is one of the single most common accessibility failures in AI-generated dark-mode UI, and it disproportionately hurts users with low vision, older users, and anyone in a bright-light environment.
- **DO:** Reserve all-caps tracked-out labels for genuinely secondary metadata (timestamps, category tags, table headers) and keep them short — uppercase text is measurably slower to read at length because it removes the word-shape cues lowercase provides.
- **DON'T:** Apply uppercase styling reflexively to every label, button, and section header as a way to seem "systematic." Overused, it stops signaling hierarchy (since everything is doing it) and starts hurting readability across the whole interface.

### Diffuse Colored Glows as Fake Depth

Another dark-mode-adjacent tell: large, soft, colored `box-shadow` or radial-gradient "glows" placed behind cards, buttons, or hero graphics — a blurred purple or blue blob sitting behind an element to make it look like it's emitting light. This is frequently used as a stand-in for actual visual hierarchy: instead of establishing depth through a real elevation system (consistent shadow direction, size, and opacity tied to z-index), a glow gets dropped behind whatever element needs to "pop," one element at a time, with no consistent rule for which elements deserve one.

The appeal is that a glow is a fast way to make a flat, static composition feel more dynamic and "alive" without touching layout or content. The tell is when glows appear behind three or four unrelated elements on the same screen, each a slightly different color and blur radius, with no shared logic — at that point they've become decoration rather than a depth cue, and they often just muddy the background and hurt legibility of whatever's on top of them.

```css
/* Generic: glow applied per-element with no consistent rule */
.hero-card { box-shadow: 0 0 120px 40px rgba(139, 92, 246, 0.35); }
.cta-button { box-shadow: 0 0 60px 10px rgba(59, 130, 246, 0.4); }
.stat-badge { box-shadow: 0 0 80px 20px rgba(236, 72, 153, 0.3); }

/* Intentional: one defined elevation system, glow reserved for a single
   deliberate "spotlight" moment rather than applied per element */
:root {
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.08);
  --shadow-md: 0 4px 12px rgba(0,0,0,0.12);
  --shadow-lg: 0 12px 32px rgba(0,0,0,0.16);
}
.card { box-shadow: var(--shadow-sm); }
.card--elevated { box-shadow: var(--shadow-md); }
.hero-spotlight { box-shadow: 0 0 100px 30px var(--brand-glow); } /* the one intentional exception */
```

- **DO:** Build a small, named elevation scale (2-4 shadow levels tied to actual UI meaning — resting, hover, elevated/modal) and use it consistently instead of inventing a new shadow value per component.
- **DON'T:** Drop a large, soft colored glow behind every hero graphic, card, or button that "needs to pop." Repeated across a screen with no shared system, glows stop reading as depth and start reading as an unearned decorative habit — and they often reduce contrast for whatever content sits on top of them.
- **DO:** If a glow effect fits the brand (it can, especially for genuinely futuristic or gaming-adjacent products), treat it as a signature applied to one specific element type consistently — e.g., always behind the primary CTA — not as a generic "make it fancier" tool used differently on every screen.

### The Timid, Personality-Free Palette

The inverse failure mode of gradient/glow overuse is a palette so cautious it says nothing at all: near-white background, near-black text, one blue accent color for links and primary buttons, and nothing else. This is common in AI-generated UI because a narrow, safe palette is the least likely to look "wrong" to a model optimizing for inoffensive competence — but the same caution that avoids obvious mistakes also avoids any actual brand personality, and the result is indistinguishable from a hundred other tools using the identical default blue.

This is distinct from genuine minimalism, which is a deliberate constraint applied with confidence (limited palette, but every color choice considered and the restraint itself communicating something). A timid palette isn't restrained on purpose — it's just unfinished, one step short of an actual color decision, usually because nobody asked "what should this brand's accent color be and why" and the placeholder blue from the component library's default theme never got replaced.

```css
/* Generic: the untouched component-library default accent */
:root {
  --primary: #3b82f6; /* Tailwind's default blue-500, unmodified */
  --text: #111827;
  --bg: #ffffff;
}

/* Intentional: a considered accent tied to brand, with a defined role
   for where it appears vs. where neutral tones carry the page */
:root {
  --brand-accent: #d4632f;  /* the brand's actual signature color */
  --text: #1a1a18;
  --bg: #faf9f6;
}
/* accent reserved for primary actions and key data points only —
   not applied to every link, icon, and hover state indiscriminately */
```

- **DO:** Choose an accent color deliberately and define explicit rules for where it's used (primary actions, key data highlights, focus states) versus where neutral tones carry the interface, so the accent stays meaningful rather than diluted across every element.
- **DON'T:** Leave the component library's default accent color (Tailwind's default blue, shadcn's default slate/zinc-plus-blue) unmodified and call the palette "clean" or "minimal." Unmodified defaults are not a design decision — they're the absence of one, and they make the product visually interchangeable with every other app built on the same starter.
- **DO:** Test the palette against the brand's actual differentiators. If the product's whole pitch is "the fun, human alternative to enterprise software," a palette of pure greyscale-plus-default-blue actively contradicts the positioning — the color system should reinforce the stated brand, not sit neutral to it.
## Layout & Component Tells

Layout and component patterns are where AI-generated UI is most immediately recognizable, because these patterns repeat at the structural level — the same page skeleton, the same card anatomy, the same decorative flourishes — regardless of what the product actually does. Most of these patterns trace back to the same root cause: a component or section pattern that looks acceptable in isolation gets applied reflexively, without asking whether it fits this specific content and this specific product.

### The Centered-Hero-With-Badge-Pill Template

The single most recognizable landing-page skeleton in AI-generated and template-driven marketing sites: a small rounded "pill" badge floating above the headline (often reading something like "✨ New" or "Introducing X" or a fake version number), a centered sans-serif headline (often two lines, often with the trendy-pairing accent word), a centered subheadline in muted grey, and one or two centered CTA buttons — all vertically stacked, all center-aligned, all within a max-width container on a plain or lightly-decorated background. It's become so ubiquitous that design communities can identify a landing page as "probably AI-generated" from a thumbnail alone, purely from this skeleton.

The pattern persists because it's genuinely a reasonable default — center-aligned hero content is a legitimate, long-standing pattern for a reason: it's easy to scan, works at any viewport width, and requires no real layout decisions. That's also exactly the problem: it requires no decisions, which is why it's the first thing any AI tool or fast-moving developer reaches for, and why it stops differentiating one product from the next the moment everyone uses it identically.

```html
<!-- Generic: the exact skeleton that appears on thousands of generated sites -->
<section class="hero centered">
  <span class="badge-pill">✨ Now in public beta</span>
  <h1>The <em>smartest</em> way to manage your team</h1>
  <p class="muted">Streamline workflows, boost productivity, and scale with confidence.</p>
  <div class="cta-row">
    <button class="btn-primary">Get Started</button>
    <button class="btn-secondary">Learn More</button>
  </div>
</section>

<!-- Intentional: asymmetric layout driven by actual content —
     a real product screenshot doing the persuasive work instead of a badge pill -->
<section class="hero split">
  <div class="hero-copy">
    <h1>Approve a purchase order in 20 seconds, not 3 emails</h1>
    <p>Built for finance teams tired of chasing Slack threads for sign-off.</p>
    <button class="btn-primary">See it on your workflow</button>
  </div>
  <div class="hero-media">
    <img src="/product-screenshot-approval-flow.png" alt="Purchase order approval screen showing a one-click approve button" />
  </div>
</section>
```

- **DO:** Choose a hero layout based on what the product actually needs to show. A product with a genuinely compelling visual (a real screenshot, a live demo, a before/after) usually benefits from an asymmetric split layout that gives that visual room, rather than burying it below a centered text block.
- **DON'T:** Default to centered-text-plus-badge-pill purely because it's the fastest thing to generate. If every competitor's landing page uses the identical skeleton, matching it makes a new product look like a clone rather than a contender — differentiation at the structural level matters as much as copy.
- **DO:** If a badge/pill above the headline is genuinely warranted (announcing a real launch, a real funding round, a real award), make its content specific and true, and consider whether it needs to exist at all if it's not carrying real information.
- **DON'T:** Add a badge pill with vague, low-information filler text ("✨ AI-Powered", "🚀 New") just because the slot "looks incomplete" without one. An empty slot is better than one filled with meaningless decoration — cut it rather than pad it.

### Identical Feature Cards with Centered Icons

A close relative of the hero template: a row or grid of 3-6 cards, each with an icon (often from the same icon set, often circled or boxed in a light tint of the accent color) centered above a bolded title, followed by 1-2 sentences of body copy that are suspiciously close to the same length across every card. The uniformity is the tell — real features rarely have equally important, equally explainable, equally-verbose descriptions, but a generated or templated card grid enforces that symmetry because visual balance was prioritized over honest content.

This happens because card grids are trivial to template — write one card, duplicate it, swap the icon/title/text — and a model or a developer optimizing for a clean-looking grid will naturally pad short descriptions and trim long ones to hit a matching line count, rather than let each feature's actual importance and complexity dictate its own space.

```html
<!-- Generic: three cards, forced to identical length and identical visual weight -->
<div class="feature-grid">
  <div class="feature-card">
    <div class="icon-circle"><svg><!-- icon --></svg></div>
    <h3>Fast Performance</h3>
    <p>Experience lightning-fast load times and smooth interactions across the app.</p>
  </div>
  <div class="feature-card">
    <div class="icon-circle"><svg><!-- icon --></svg></div>
    <h3>Secure by Design</h3>
    <p>Enterprise-grade security keeps your data safe and protected at all times.</p>
  </div>
  <div class="feature-card">
    <div class="icon-circle"><svg><!-- icon --></svg></div>
    <h3>Easy Integration</h3>
    <p>Connect with your favorite tools in just a few clicks, no code required.</p>
  </div>
</div>

<!-- Intentional: weight and length driven by actual importance/complexity,
     one feature gets more room because it needs it -->
<div class="feature-grid asymmetric">
  <div class="feature-card feature-card--primary">
    <h3>Reconciles bank feeds automatically</h3>
    <p>Matches every transaction against your ledger and flags the ~3% that need
       a human look, instead of making you review all of them. Most teams cut
       reconciliation time from a day to under 20 minutes.</p>
  </div>
  <div class="feature-card">
    <h3>SOC 2 Type II certified</h3>
  </div>
  <div class="feature-card">
    <h3>Exports to QuickBooks and Xero</h3>
  </div>
</div>
```

- **DO:** Let each feature's actual weight determine its space and treatment — a genuinely differentiating feature deserves more room, a real screenshot, or a supporting stat, while a table-stakes feature can be a single line in a list. Forcing every feature into the same card size flattens what should be a hierarchy.
- **DON'T:** Pad or trim feature descriptions to match a uniform character count across a grid. Matching lengths is a symptom of prioritizing visual symmetry over honest, specific content — write what's actually true about each feature, even if that makes the cards visually uneven, and adjust the layout to handle unevenness gracefully instead.
- **DO:** Vary the icon/visual treatment (or drop icons entirely for features that don't have a good visual metaphor) rather than forcing every single feature into an icon-above-title-above-text template regardless of fit.
- **DON'T:** Use the exact same icon set, circle-background treatment, and spacing for every card without checking whether some features would communicate better as a short list, a comparison table, or an inline callout instead of a card.

### Colored Top/Left Border as a Generic "Designed" Signal

A specific micro-pattern: cards, alerts, or list items get a 2-4px colored border on just one edge (usually top or left), often color-coded by category or status. Used with an actual, consistent color-coding system (green border = success state, red = error, etc.) this is a legitimate and useful convention. The tell is when it's applied purely as visual seasoning — every card in a grid gets a different colored top border with no meaning attached to the color choice, purely because a plain card "looked unfinished" without one.

This is a fast, low-effort way to make a flat card feel more "designed" — one line of CSS adds visual interest without requiring any actual layout or content decisions — which is exactly why it shows up so often in generated UI as a default flourish rather than a deliberate system.

```css
/* Generic: colored top borders with no attached meaning, just visual variety */
.card:nth-child(1) { border-top: 3px solid #6366f1; }
.card:nth-child(2) { border-top: 3px solid #ec4899; }
.card:nth-child(3) { border-top: 3px solid #22c55e; }

/* Intentional: the border color is a status/category system, not decoration */
.alert--success { border-left: 3px solid var(--color-success); }
.alert--warning { border-left: 3px solid var(--color-warning); }
.alert--error   { border-left: 3px solid var(--color-error); }
/* a plain feature card, meanwhile, stays plain — no border color needed to "finish" it */
.card { border: 1px solid var(--border-subtle); }
```

- **DO:** Reserve colored accent borders for cases where the color genuinely encodes information — status, category, severity — and make sure the same color always means the same thing across the product.
- **DON'T:** Add a colored top or left border to a card just because a plain card feels visually unfinished. If the border color doesn't map to anything (it's just "card 1 is indigo, card 2 is pink, card 3 is green" for variety), it's adding visual noise without adding information — leave the card plain, or fix "unfinished" through better spacing, type, or content instead.
- **DO:** If a design genuinely wants color-differentiated cards for a reason (comparing plan tiers, for instance), make the mapping explicit and legible — labeled, not just color-coded — since color alone isn't accessible to colorblind users or reliable as the sole differentiator.
### Reflexive Numbered "1, 2, 3" Step Sequences

Any process description — even ones with no inherent order, or with steps that happen in parallel, or where "step 2" depends on a choice rather than a linear progression — gets forced into a numbered "How it works: 1 → 2 → 3" layout with a circle-numbered icon above each step. This is one of the most common structural defaults in generated marketing and onboarding content because numbered steps *look* authoritative and easy to scan, regardless of whether the underlying process is actually sequential.

The tell shows up in two ways: first, when steps that don't have a real dependency order get artificially numbered (e.g., "1. Choose your plan, 2. Invite your team, 3. Customize your workspace" — these can usually happen in any order); second, when a process is forced down to exactly three steps because three is the visually "balanced" number for a row of icons, collapsing what's actually a five- or six-step process and hiding real complexity the user needs to know about.

```html
<!-- Generic: three steps forced onto a genuinely non-sequential set of actions -->
<div class="steps">
  <div class="step"><span class="step-number">1</span><h3>Sign up</h3></div>
  <div class="step"><span class="step-number">2</span><h3>Customize</h3></div>
  <div class="step"><span class="step-number">3</span><h3>Launch</h3></div>
</div>

<!-- Intentional: numbering only where order is real, and steps not
     artificially compressed to fit a tidy count -->
<ol class="setup-steps">
  <li>Connect your calendar (Google or Outlook) — required before anything else works.</li>
  <li>Set your available hours.</li>
  <li>Share your booking link, or embed it on your site.</li>
</ol>
<!-- vs. a set of independent options presented without false sequence -->
<div class="options-grid">
  <div class="option"><h3>Invite your team</h3></div>
  <div class="option"><h3>Import existing data</h3></div>
  <div class="option"><h3>Connect an integration</h3></div>
</div>
```

- **DO:** Use numbered steps only when the underlying process genuinely has a required order — where step 2 can't happen before step 1. If actions can happen in any order or in parallel, present them as an unordered list, a grid of options, or tabs instead.
- **DON'T:** Force a process into exactly three numbered steps purely because three fits a tidy visual row. If the real process has five steps, show five — compressing it to look cleaner hides information the user will need and sets up a support burden when they discover the missing steps later.
- **DO:** When steps are genuinely sequential, make the dependency visible in the copy itself ("first," "once you've done X," "finally") so the numbering is reinforced by language, not just by a circled digit.

### Fabricated Stat/Metric Banner Rows

A row of big bold numbers with small labels underneath — "10,000+ Users," "99.9% Uptime," "4.9/5 Rating," "50M+ Requests Processed" — dropped into a landing page as a trust-building section, frequently for products with no real usage data to report yet (a pre-launch product, an internal tool, a personal project). The visual pattern (big number, small caption, often in a horizontal row of 3-4) reads as credible at a glance specifically because real companies use this exact pattern with real numbers — which is what makes a fabricated version actively deceptive rather than merely lazy.

This is worth treating as more serious than a purely aesthetic tell: an invented stat isn't just generic-looking, it's a false claim, and shipping one (even as a "placeholder to replace later") risks it surviving into production, where it becomes actively misleading to users and potentially a legal/trust liability, not just an aesthetic misstep.

```html
<!-- Generic: numbers invented to fill a "social proof" section template -->
<div class="stats-row">
  <div class="stat"><span class="stat-number">10,000+</span><span class="stat-label">Happy Users</span></div>
  <div class="stat"><span class="stat-number">99.9%</span><span class="stat-label">Uptime</span></div>
  <div class="stat"><span class="stat-number">4.9/5</span><span class="stat-label">Average Rating</span></div>
</div>

<!-- Intentional: either real numbers, or the section is omitted entirely
     until there's something true to say -->
<div class="stats-row">
  <div class="stat"><span class="stat-number">312</span><span class="stat-label">Teams on the waitlist</span></div>
</div>
<!-- or, more honestly, for a pre-launch product: no stats section at all,
     replaced with a specific, verifiable claim about the product itself -->
<p class="credibility-line">Built by two engineers who spent six years running
   payments infrastructure at [real, named previous company].</p>
```

- **DO:** Only display a metric that is currently true and verifiable. If real usage numbers don't exist yet, either omit the section or use a genuinely available number (waitlist size, GitHub stars, a beta cohort size) even if it's modest.
- **DON'T:** Fill a stats banner with invented placeholder numbers "to be replaced later" — placeholder numbers routinely survive into shipped product because nobody remembers to circle back, and a fabricated stat is a false claim to users, not just an aesthetic shortcut.
- **DO:** For a genuinely early-stage product, replace generic social-proof numbers with specific, verifiable credibility signals instead — a named technical detail, a real customer logo (with permission), a specific and true claim about the team or the technology.

### Emoji-as-Icon-System

Sidebar navigation items, feature lists, and section headers styled with an emoji standing in for a real icon — 📊 for "Analytics," 🔒 for "Security," 🚀 for "Launch." This shows up constantly in AI-generated UI because emoji are trivial to produce (a single Unicode character, no asset pipeline, no icon library dependency) and they read as "friendly" at a glance — but they're a real icon system's placeholder, not a substitute for one, and they carry real functional and visual costs.

Functionally, emoji render inconsistently across operating systems and browsers (a 🚀 looks meaningfully different on Windows, macOS, and various Android skins), they don't inherit the interface's color system (they're full-color glyphs that can't be recolored to match a hover or active state), they don't scale or align with text baselines the way a well-drawn icon font or SVG set does, and they're a poor fit for interfaces that want to project any seriousness or maturity — emoji read as casual by default, which is a mismatch for most B2B, financial, health, or infrastructure products.

```html
<!-- Generic: emoji doing the job of an icon system -->
<nav>
  <a href="/analytics">📊 Analytics</a>
  <a href="/settings">⚙️ Settings</a>
  <a href="/security">🔒 Security</a>
</nav>

<!-- Intentional: a real icon set that inherits currentColor and matches the type scale -->
<nav>
  <a href="/analytics"><svg class="icon" aria-hidden="true"><use href="#icon-bar-chart"/></svg> Analytics</a>
  <a href="/settings"><svg class="icon" aria-hidden="true"><use href="#icon-gear"/></svg> Settings</a>
  <a href="/security"><svg class="icon" aria-hidden="true"><use href="#icon-lock"/></svg> Security</a>
</nav>
<style>
.icon { width: 16px; height: 16px; color: currentColor; vertical-align: -3px; }
a:hover .icon { color: var(--brand-accent); }
</style>
```

- **DO:** Use a real icon library (Lucide, Phosphor, Heroicons, a custom SVG set, an icon font) for any persistent UI chrome — navigation, buttons, status indicators — so icons inherit color, scale consistently, and render identically across platforms.
- **DON'T:** Use emoji as a stand-in for a navigation or feature icon system. Beyond the inconsistent cross-platform rendering, emoji can't be recolored for hover/active/disabled states, don't align cleanly to a text baseline, and impose a casual tone that's frequently wrong for the product.
- **DO:** Reserve emoji for genuinely casual, conversational contexts where they're used deliberately — a changelog entry, a Slack-style chat message, a playful empty state — not as infrastructure.
- **DON'T:** Reach for emoji because installing/importing an icon library feels like extra setup. The setup cost is trivial (most icon sets are a single package or even copy-pasted SVGs) compared to the inconsistency and tonal mismatch of shipping emoji as permanent UI chrome.

### Glassmorphism as a Default Treatment

Frosted-glass panels — semi-transparent backgrounds with a `backdrop-filter: blur()`, a subtle light border, sitting over a colorful or gradient background so the blur has something to show through — became a dominant AI-generated UI aesthetic because it's visually striking with very little effort (a few CSS properties turn a plain card into something that looks technically sophisticated) and it echoes real design-system precedent (frosted materials are a legitimate, well-executed pattern in iOS/macOS system UI). The problem is that glassmorphism only actually works when there's something worth blurring behind it and a coherent reason for the depth metaphor — applied to a plain white or single-color background, the blur does nothing but add rendering cost, and applied to text-heavy content, it frequently hurts legibility.

It's also become such a default "make it look modern" move that it now shows up on products with no design language that calls for translucency or layered depth at all — dashboards, forms, content-heavy pages — where a frosted panel is pure decoration disconnected from any actual visual system.

```css
/* Generic: glass treatment applied regardless of what's behind it or why */
.card {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.2);
}
/* ...sitting on a plain solid-color page background where there's nothing to blur */

/* Intentional: reserved for a real layered context, with a legible background
   underneath the blur and sufficient contrast for the content on top */
.command-palette-overlay {
  background: rgba(20, 20, 24, 0.75);
  backdrop-filter: blur(16px) saturate(140%);
  border: 1px solid rgba(255, 255, 255, 0.08);
}
/* used specifically for a floating overlay above real page content —
   a modal, a command palette, a floating toolbar — where translucency
   communicates "this sits above the page," not on a flat static card */
```

- **DO:** Reserve glass/blur treatments for elements that are genuinely floating above other content — modals, command palettes, floating toolbars, notification overlays — where the translucency communicates a real z-axis relationship.
- **DON'T:** Apply `backdrop-filter: blur()` to cards sitting on a flat, single-color background just because it "looks modern." With nothing behind it to blur, the effect does nothing visually while still costing render performance, and it signals a decorative habit rather than a design decision.
- **DO:** Verify text contrast on glass surfaces specifically, since the effective background color shifts depending on whatever content happens to be behind the blur — a translucent panel that looks fine over one background can fail contrast over a busier one.
- **DON'T:** Reach for glassmorphism as the default "premium" treatment for every card and panel in a product that has no other translucency or layering in its visual language. A single glass element in an otherwise flat, opaque design language looks like a leftover template component, not a considered accent.

### Unmodified Component-Library Defaults

Perhaps the most systemic tell of all: shipping a UI built on shadcn/ui, Material UI, Chakra, or a similar component library with zero customization — default border-radius, default spacing scale, default neutral palette (shadcn's default slate/zinc grey with the default blue/black accent), default shadow tokens, all untouched. Because these libraries are the fastest path to a working, accessible, reasonably attractive UI, an enormous fraction of AI-assisted and rapidly-shipped products use one of a small handful of them — and when none of them customize the theme, the result is that a huge swath of unrelated products are now visually identical down to the pixel: same button padding, same card corner radius, same dropdown shadow, same input focus ring.

This is a genuinely different failure mode from the others in this section, because the underlying components are usually well-built, accessible, and a legitimate foundation to build on — the problem isn't using shadcn/ui, it's stopping at the default theme rather than treating the library as a starting point for a system that should still express the specific product's identity.

```css
/* Generic: shadcn/ui's default theme tokens, never touched */
:root {
  --radius: 0.5rem;
  --primary: 222.2 47.4% 11.2%;
  --background: 0 0% 100%;
  /* ...every other token left at its scaffolded default */
}

/* Intentional: the same component architecture, tokens overridden
   to express an actual brand */
:root {
  --radius: 0.25rem; /* sharper corners, matching a more technical/precise brand feel */
  --primary: 21 90% 48%;      /* the brand's actual accent hue */
  --background: 40 20% 97%;   /* warm off-white instead of stark white */
  --font-sans: "Söhne", ui-sans-serif, system-ui;
}
```

- **DO:** Treat a component library's default theme as a starting scaffold, not a finished visual identity — override the color tokens, radius scale, spacing scale, shadow tokens, and typography before shipping, even if the component *structure* stays exactly as the library provides it.
- **DON'T:** Ship a product with a popular component library's default theme completely untouched. It's the single fastest way to make a new product visually indistinguishable from thousands of others built on the same starter — a specific, well-known giveaway to anyone who's used the library themselves.
- **DO:** Change at minimum the border-radius scale, the accent/primary color, and the typeface — these three token changes alone are usually enough to break the "obviously default shadcn" recognition pattern while keeping all the library's accessibility and interaction-pattern benefits.
- **DON'T:** Confuse "using a component library" with "the product has no design system." A well-customized theme layer on top of a solid component library is a legitimate, efficient way to build a real design system — the failure is skipping the customization step, not using the library itself.

### Uniform Rounded Corners Everywhere

Every element on the page — buttons, cards, inputs, avatars, badges, modals, images, even large section containers — uses the exact same border-radius value, frequently a large one (12-24px), applied with no variation regardless of element size or role. This produces a soft, bubbly uniformity that reads as inoffensive but also flat and undifferentiated, because border-radius is one of the few visual properties that scales meaning with an element's size and purpose: a small radius on a dense data table row communicates something different than a large radius on a hero card, and using identical values everywhere erases that distinction.

The habit comes from setting one `--radius` token (often a component library default) and applying it globally via a CSS reset or utility class, which is efficient but skips the step of actually deciding whether a 40px avatar, a 400px hero card, and an 8px badge should share a radius at all — proportionally, they usually shouldn't.

```css
/* Generic: one radius value, applied identically at every scale */
.button, .card, .input, .modal, .avatar, .badge {
  border-radius: 16px;
}

/* Intentional: radius scales with element size/role, small elements getting
   proportionally smaller values so corners don't dominate small components */
.badge   { border-radius: 4px; }   /* small, dense — subtle rounding */
.button  { border-radius: 8px; }
.input   { border-radius: 6px; }
.card    { border-radius: 12px; }  /* larger surface can carry more curve */
.avatar  { border-radius: 50%; }   /* circular is its own intentional choice, not the default radius */
.modal   { border-radius: 16px; }
```

- **DO:** Scale border-radius roughly with element size and treat it as a deliberate part of the type/spacing system — small, dense elements (badges, chips, small buttons) generally read better with a smaller radius; large surfaces (cards, modals, hero containers) can carry a larger one without looking disproportionate.
- **DON'T:** Apply one global border-radius value to every element regardless of size via a blanket CSS reset. At small sizes an oversized radius can approach or exceed half the element's height, producing an odd pill/lozenge shape that likely wasn't the intent — check radius at actual rendered size, not just as an abstract token value.
- **DO:** Decide on a radius *personality* for the brand (sharp and precise vs. soft and friendly) and apply it consistently as a scale, so the choice reads as systematic rather than arbitrary per-component guessing.

### Low-Opacity Drop Shadows on Everything

A close cousin of the uniform-radius problem: every card, button, input, and container gets the same faint drop shadow (commonly something like `box-shadow: 0 2px 4px rgba(0,0,0,0.1)`), applied as a blanket default rather than as a deliberate elevation cue. Used correctly, shadows communicate z-axis relationships — this element floats above that one. Applied to literally everything at the same intensity, they stop communicating elevation at all (if everything is elevated by the same amount, nothing is) and just add a faint, slightly muddy halo around every rectangle on the page.

This is another case of a component-library or CSS-reset default surviving unexamined into production: most starter kits ship a single default card/button shadow, and because it looks "fine" (mildly polished, not obviously wrong) in isolation, nobody revisits it to ask whether it should vary by context.

```css
/* Generic: identical faint shadow on every surface, communicating nothing */
.card, .button, .input, .dropdown, .badge, .avatar {
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Intentional: shadow presence and intensity map to actual elevation */
.input, .badge { box-shadow: none; border: 1px solid var(--border); } /* flush with the page */
.card { box-shadow: var(--shadow-sm); } /* resting elevation */
.dropdown, .popover { box-shadow: var(--shadow-md); } /* floats above content */
.modal { box-shadow: var(--shadow-lg); } /* highest in the stack */
```

- **DO:** Assign shadow intensity based on actual z-axis relationship — flush elements (inputs, inline badges) get none or just a border; floating elements (dropdowns, popovers, modals) get shadows that scale with how far "above" the page they sit.
- **DON'T:** Apply the same low-opacity shadow to every component as a blanket default. When flush elements like inputs and badges get the same shadow as floating elements like modals, the shadow stops encoding any real spatial information and just becomes visual noise.
- **DO:** Consider whether a border or a background-color shift communicates a component's boundary better than a shadow at all — plenty of well-designed flat/flush interfaces use shadow only for genuine overlays and rely on borders and spacing everywhere else.

### The Three-Boxes-With-Icons Feature Grid Cliché

A specialization of the identical-feature-cards problem specific enough to call out on its own: almost every generated landing page settles on exactly three feature callouts, each with a centered icon, a short bolded title, and one line of description, arranged in a single row. Three is neither a meaningful nor a researched number for most products' actual feature set — it's the number that fits comfortably in a row at typical desktop widths without wrapping, and it's become such a strong convention that it gets used regardless of whether the product has three standout differentiators, one, or fifteen.

The deeper problem with this pattern isn't the count, it's what the count forces: a product is compressed to exactly three bullet-point-level claims with no room for depth, evidence, or differentiation, when the honest story might be "there's one thing we do dramatically better than anyone else, and everything else is table stakes" — a message that a uniform three-box grid actively obscures.

```html
<!-- Generic: three generic-feeling boxes because three is the "right" number visually -->
<div class="three-box-grid">
  <div><svg/><h3>Fast</h3><p>Blazing fast performance.</p></div>
  <div><svg/><h3>Secure</h3><p>Bank-level security.</p></div>
  <div><svg/><h3>Simple</h3><p>Easy to use interface.</p></div>
</div>

<!-- Intentional: one differentiator gets real space and evidence,
     the rest are compressed to a scannable list -->
<section class="primary-differentiator">
  <h2>The only tool that reconciles across four payment processors at once</h2>
  <p>Most tools force you to export CSVs from Stripe, PayPal, and Square separately.
     We pull all three live and match them against your bank feed automatically.</p>
</section>
<ul class="table-stakes-list">
  <li>SOC 2 Type II</li>
  <li>SSO / SAML</li>
  <li>Audit log export</li>
  <li>Role-based permissions</li>
</ul>
```

- **DO:** Let the number of feature callouts reflect the product's actual differentiators, not a default row-of-three template. One genuinely strong differentiator with real supporting evidence outperforms three generic, evidence-free claims.
- **DON'T:** Force the feature set into exactly three icon-boxes because that's the visually "complete-looking" number. If there's really only one standout feature, give it a full section with real detail; if there are eight legitimately important ones, use a list, a table, or a tabbed layout instead of cramming or arbitrarily cutting to three.
- **DO:** Distinguish between genuine differentiators (worth a full callout with evidence) and table-stakes features (fine as a compact checklist) rather than giving every feature — important or not — identical visual weight.

### Fake Social Proof Blocks

Testimonial quotes attributed to generic names ("Sarah J., Product Manager") paired with stock or AI-generated avatar images, five-star rating displays with no link to a real review source, and "as featured in" logo rows for publications the product was never actually covered by. This is the most ethically serious tell in this entire list — unlike a generic gradient or an overused icon, fabricated social proof is a direct, checkable falsehood presented to build trust the product hasn't earned, and it's one that AI tools are particularly prone to generating as "placeholder" content that then ships unedited.

The mechanism is straightforward and specifically dangerous: asked to build a landing page for a product with no customers yet, an AI assistant (or a developer under time pressure) fills the "social proof" section with plausible-sounding invented quotes and generic placeholder avatars because the section template expects content there — and because it looks so similar to real testimonial sections, it's easy for both the builder and a casual reviewer to forget it isn't real before it ships.

```html
<!-- Generic: fabricated testimonial with a stock/generated avatar and no way to verify it -->
<div class="testimonial">
  <img src="/avatars/generic-woman-1.jpg" alt="" />
  <blockquote>"This tool completely transformed how our team works. Highly recommend!"</blockquote>
  <cite>— Sarah J., Product Manager</cite>
</div>

<!-- Intentional: real, attributable, or explicitly framed as early/aspirational -->
<div class="testimonial">
  <img src="/avatars/real-customer-headshot.jpg" alt="" />
  <blockquote>"We cut our reconciliation time from a day to twenty minutes."</blockquote>
  <cite>— <a href="https://linkedin.com/in/realperson">Actual Name, Actual Title, Actual Company</a></cite>
</div>
<!-- or, honestly, for a pre-launch product: no testimonial section at all -->
```

- **DO:** Only publish testimonials, ratings, and "as seen in" logos that are real and, ideally, verifiable (linked to a real person's profile, a real review platform, a real published article). If none exist yet, omit the section rather than inventing content to fill it.
- **DON'T:** Ever ship a fabricated testimonial, an invented star rating, or a stock/AI-generated avatar attached to a fake name, even "temporarily" or "as a placeholder." Placeholder fabricated content routinely ships to production because it looks finished — this is a trust and, in many jurisdictions, a legal/advertising-standards problem, not merely a design one.
- **DO:** For an early-stage product with no customers yet, use honest alternatives that build real trust — a clear statement of the team's background, a live/interactive demo, a specific and verifiable technical claim, or an invitation to a waitlist with a real, current count.
- **DON'T:** Treat "the design needs a social proof section to feel complete" as a justification for filling it with anything. A landing page missing a testimonial section is far less damaging than one caught displaying a fabricated one — cut the section instead.

### Decorative Blob/Wave SVG Background Filler

Abstract, softly-colored blob shapes or wavy divider lines used as background decoration — usually semi-transparent, usually gradient-filled, usually placed behind a hero section or between page sections to "add visual interest" to what would otherwise be a plain background. These shapes proliferated because free blob/wave generator tools made them trivial to produce with zero design skill required, and they became a fast way to make a flat page feel less static without touching layout, type, or content.

The tell is that these shapes almost never relate to the product, the brand, or the content around them — they're generic filler, interchangeable between a fintech app and a meditation app and a project management tool, which is precisely what makes them read as decoration-for-its-own-sake rather than a considered part of the visual system.

```html
<!-- Generic: an abstract blob dropped behind the hero purely to fill visual space -->
<section class="hero">
  <svg class="decorative-blob" viewBox="0 0 400 400">
    <path d="M400,200Q400,300,300,350..." fill="url(#gradient)" opacity="0.3"/>
  </svg>
  <h1>Welcome to Product</h1>
</section>

<!-- Intentional: either no decorative background at all (letting type and
     spacing carry the section), or a shape system tied to the actual brand -->
<section class="hero hero--minimal">
  <h1>Welcome to Product</h1>
</section>
<!-- or, if abstract shapes are used, they're derived from something real —
     a literal representation of the product's data, a brand-mark motif
     repeated consistently, not a generic downloaded blob asset -->
```

- **DO:** Let whitespace, type, and content carry a section's visual interest before reaching for decorative background shapes. A well-set headline with generous spacing rarely needs a blob behind it to avoid looking "empty."
- **DON'T:** Add generic downloaded/generated blob or wave shapes purely to fill perceived empty space. If the shape has no connection to the brand's actual visual language and would look equally at home on a completely unrelated product, it's filler, not design.
- **DO:** If abstract background shapes fit the brand, derive them from something specific — an actual brand motif, a literal visualization of the product's own data or process — used consistently across the product, rather than a one-off decorative asset unique to the hero section.

### Meaningless Scroll-Triggered Fade/Slide-In Animations

Every section of a page fades in and slides up slightly as it enters the viewport on scroll — a pattern so common in template and AI-generated marketing sites that it's essentially assumed as a checkbox feature ("does the site have scroll animations?") rather than a considered choice. Applied uniformly to every single section, paragraph, and card with the same easing and duration, the effect stops drawing attention to anything specific — when everything animates identically, the animation carries no information about what's actually important, and on longer pages it actively slows down the experience of scrolling to find something, since content is invisible or displaced until the animation completes.

Motion is a legitimate and valuable tool when it's used to communicate something — that an action succeeded, that this element is now interactive, that this content relates causally to what just happened. Blanket scroll-triggered fade-ins communicate none of that; they're applied as decoration because a scroll library was easy to install, not because a specific section benefited from a specific animation.

```javascript
// Generic: every section, every card, every paragraph gets identical fade+slide
document.querySelectorAll('section, .card, p').forEach(el => {
  observer.observe(el); // same fade-up-on-scroll, applied indiscriminately
});
```

```javascript
// Intentional: motion reserved for moments that need it — a state change,
// a genuinely new arrival (a live-updating list), not scroll position alone
function onOrderStatusChange(newStatus) {
  statusBadge.animate(
    [{ opacity: 0, transform: 'scale(0.9)' }, { opacity: 1, transform: 'scale(1)' }],
    { duration: 200, easing: 'ease-out' }
  ); // draws the eye specifically to the thing that just changed
}
```

- **DO:** Reserve motion for moments that communicate real state — something appeared, something changed, something succeeded or failed, this element is now focused/active. Motion tied to a real event carries information; motion tied only to scroll position doesn't.
- **DON'T:** Apply the same fade-up-on-scroll animation to every section, card, and paragraph on a page by default. Applied everywhere uniformly, it adds load time and perceived sluggishness without helping the user understand or navigate the content any better than if it had simply been visible.
- **DO:** Respect `prefers-reduced-motion` for any animation that isn't purely decorative-and-optional, and consider whether the animation would still be worth including if it couldn't be seen at all by a user with vestibular sensitivity — if the answer is no, it likely wasn't communicating anything essential.
- **DON'T:** Treat "does it have scroll animations" as a completeness checkbox for a landing page. A page with zero motion but strong type, spacing, and content beats a page with animation on every element but nothing underneath worth looking at once it settles.
## Functional/UX Gaps

Visual tells are the fastest way to spot AI-generated UI at a glance, but functional gaps are the more damaging tell, because they surface exactly when a user tries to actually use the product rather than just look at it. These gaps happen because generating a UI that *looks* complete (the happy-path screen, populated with plausible content, in its default state) is a much smaller task than building a UI that *behaves* completely (every input validated, every async state handled, every interactive affordance actually wired up) — and the gap between the two is invisible in a static screenshot or a quick demo click-through.

### Missing Form Validation and Inline Error Messages

Forms that accept any input and either silently fail, show a generic unhelpful browser alert, or — worse — submit invalid data without any feedback at all. This is one of the most common gaps in rapidly generated or prototyped UI because a form's happy path (fill in valid data, submit, see success) is trivial to build and demo, while the actual majority of form-handling logic — what happens with an invalid email, a password that doesn't meet requirements, a required field left blank, a network failure mid-submit — requires deliberately building and testing failure paths that a quick demo never exercises.

The absence of validation isn't just an aesthetic gap; it's usually the single most user-visible sign that a product wasn't built with real usage in mind, because every real user eventually mistypes something, and the product's response to that moment says more about its quality than its visual polish does.

```html
<!-- Generic: no client-side validation, no inline feedback, silent failure -->
<form>
  <input type="email" name="email" />
  <input type="password" name="password" />
  <button type="submit">Sign up</button>
</form>

<!-- Intentional: inline, specific, actionable validation feedback -->
<form novalidate>
  <label for="email">Email</label>
  <input type="email" id="email" name="email" required aria-describedby="email-error" />
  <p id="email-error" class="field-error" role="alert" hidden>
    Enter a valid email address, like name@example.com.
  </p>

  <label for="password">Password</label>
  <input type="password" id="password" name="password" required minlength="8"
         aria-describedby="password-hint" />
  <p id="password-hint" class="field-hint">At least 8 characters.</p>

  <button type="submit">Sign up</button>
</form>
```

- **DO:** Validate every form both client-side (for immediate, helpful feedback) and server-side (because client-side validation can always be bypassed) and surface errors inline, next to the specific field they concern, with specific and actionable messages ("Enter a valid email address" beats "Invalid input").
- **DON'T:** Ship a form that relies solely on the browser's default validation UI (which is inconsistent across browsers, easy to miss, and not stylable to match the product) or, worse, no validation at all. A form with no failure-path handling is a strong signal the product was never tested against real, imperfect human input.
- **DO:** Validate on blur or on submit rather than on every keystroke for most fields — validating too aggressively (marking a field invalid while the user is still typing their first character) is its own UX failure and almost as recognizable a "wasn't actually tested" tell as no validation at all.
- **DON'T:** Use a generic top-of-page banner ("There was an error") as the only error feedback for a multi-field form. The user has to hunt for which field is wrong; tie errors to their specific field so the fix is immediately obvious.

### No Visible Required-Field Indicators

Forms where some fields are required and others aren't, with no visual distinction between them until the user submits and discovers which ones were mandatory — usually via a jarring wall of errors that could have been avoided by simply marking required fields up front. This is a small omission with an outsized effect on perceived quality, because it forces the user to learn the form's rules through trial and error instead of being told the rules before they start.

It happens for the same underlying reason as missing validation: the happy path (a user who happens to fill in every field) never surfaces the problem, so it's invisible unless someone deliberately tests submitting a partially-filled form.

```html
<!-- Generic: no indication of which fields are actually required -->
<label for="name">Name</label>
<input id="name" name="name" />
<label for="company">Company</label>
<input id="company" name="company" />

<!-- Intentional: required fields marked clearly and accessibly up front -->
<label for="name">Name <span class="required" aria-hidden="true">*</span></label>
<input id="name" name="name" required aria-required="true" />
<label for="company">Company <span class="optional">(optional)</span></label>
<input id="company" name="company" />
<p class="form-note">Fields marked <span aria-hidden="true">*</span> are required.</p>
```

- **DO:** Mark required fields visibly (a `*` with a legend explaining it, or the word "required" in the label) before the user starts filling out the form, not only after a failed submission.
- **DON'T:** Leave the required/optional status of fields ambiguous until submit-time error messages reveal it. This turns form-filling into a guessing game and produces avoidable frustration on a task that should be straightforward.
- **DO:** Consider marking *optional* fields instead of required ones when most fields in a given form are mandatory — whichever is the minority case is usually clearer to label explicitly, and reduces visual clutter from asterisks on nearly every field.

### Missing Loading, Empty, and Error States for Async Content

A screen that only has one visual design — the "happy path, data loaded successfully" state — with no defined treatment for what the user sees while data is loading, what they see if there's no data yet (a new account, a search with no results, a list nobody has populated), or what they see if the request fails (network error, server error, permission denied). Generated and quickly-prototyped UI overwhelmingly shows only this one state because it's the only state a static mockup or a happy-path demo naturally produces — building the other three requires deliberately imagining and designing for failure and absence, not just success.

This is a functional gap with an immediate visual symptom: in production, its absence shows up as a blank white screen during loading, a confusing empty table with no explanation during a true empty state, or a raw stack trace/console error with no user-facing message at all during a failure — each of which reads unmistakably as "this was never actually tested against anything but the ideal case."

```jsx
/* Generic: only the happy path is designed; everything else falls through to nothing */
function OrderList({ orders }) {
  return (
    <ul>
      {orders.map(o => <li key={o.id}>{o.name}</li>)}
    </ul>
  ); // renders an empty <ul> with nothing in it if orders is [], undefined, or still loading
}

/* Intentional: all four states are explicitly designed for */
function OrderList({ orders, isLoading, error }) {
  if (isLoading) return <SkeletonList rows={5} />;
  if (error) return <ErrorState message="Couldn't load your orders. Try again." onRetry={refetch} />;
  if (orders.length === 0) return <EmptyState
    title="No orders yet"
    body="Orders you place will show up here."
    action={<Button href="/shop">Browse products</Button>}
  />;
  return <ul>{orders.map(o => <li key={o.id}>{o.name}</li>)}</ul>;
}
```

- **DO:** Design and implement all four states for any view that depends on async data — loading, populated, empty, and error — before considering the feature complete. Treat "what does this look like with zero items" and "what does this look like if the request fails" as first-class design questions, not afterthoughts.
- **DON'T:** Ship a view that only handles the successfully-loaded, non-empty case. In production this reliably surfaces as blank screens, confusing empty tables, or raw error output — each an unmistakable sign the feature was demoed once against a seeded happy-path dataset and never tested against reality.
- **DO:** Make empty states genuinely useful, not just an absence of content — explain why it's empty and offer the next action (a CTA to create the first item, a suggestion to adjust a filter), rather than a bare "No results" with nothing else on the screen.
- **DON'T:** Show a spinner with no context for loads that might take more than a second or two, and never show a raw technical error message (a stack trace, a raw HTTP status code) to an end user — translate errors into plain language with a clear next step (retry, contact support, go back).

### No Real Accessibility Support

Interfaces with no visible focus states on interactive elements (or focus outlines suppressed via `outline: none` with nothing put in its place), color-only status indicators with no text or icon backup, insufficient color contrast, and layouts that are impossible or painful to navigate via keyboard alone. This is one of the most common and most consequential gaps in AI-generated and rapidly-built UI, because accessibility failures are invisible to anyone testing only with a mouse and only visually — they surface specifically for keyboard users, screen-reader users, and low-vision users, a population that a sighted mouse-driven demo simply never represents.

The mechanism is straightforward: a model or developer optimizing for "this looks right" checks the thing by looking at it, which by definition validates only the visual/mouse-driven experience. Accessibility requires actively testing a different interaction mode (tab through the whole page; turn on a screen reader; check contrast with a tool, not with eyes) — a step that gets skipped under time pressure or when the reviewer never learned to check for it.

```css
/* Generic: focus outline suppressed with nothing to replace it */
button:focus { outline: none; }

/* Intentional: a clear, high-contrast focus indicator that survives customization */
button:focus-visible {
  outline: 2px solid var(--focus-ring);
  outline-offset: 2px;
}
```

```html
<!-- Generic: status communicated by color alone -->
<span class="status" style="color: red;">●</span> Failed

<!-- Intentional: status communicated redundantly — color, icon, and text together -->
<span class="status status--error">
  <svg aria-hidden="true"><!-- x-circle icon --></svg> Failed
</span>
```

- **DO:** Keep a visible, high-contrast focus indicator on every interactive element, and if the default browser outline doesn't match the visual design, replace it with an equally or more visible custom one via `:focus-visible` — never remove it with nothing in its place.
- **DON'T:** Suppress focus outlines with `outline: none` for aesthetic reasons without a replacement. This makes the entire interface unusable for anyone navigating by keyboard, which includes not just screen-reader users but power users, users with motor impairments, and anyone with a broken trackpad.
- **DO:** Pair every color-coded status, error, or category indicator with a redundant non-color cue — an icon, a text label, a pattern — so the information isn't lost for colorblind users (roughly 1 in 12 men) or anyone viewing the screen in poor lighting.
- **DON'T:** Rely on color alone to communicate meaning (red text for errors, green for success, with no icon or text label). If desaturating the screen to greyscale makes the interface's meaning disappear, the color is carrying information it shouldn't be carrying alone.
- **DO:** Actually tab through every interactive flow using only the keyboard, and check contrast ratios with a real tool rather than eyeballing them, before calling any UI work finished — both take only a few minutes and catch a large fraction of accessibility gaps.
- **DON'T:** Treat accessibility as a separate pass to "come back to later." Retrofitting keyboard navigation, focus order, and ARIA semantics onto a UI built without them in mind is dramatically more expensive than building them in from the start, and "later" frequently never arrives.

### Interactive-Looking Elements That Aren't Wired Up (and the Reverse)

Elements styled with clear affordance cues — a hover state, a pointer cursor, a button-shaped container — that do nothing when clicked, alongside the inverse problem: elements that are functionally clickable but give no visual indication that they are (a `<div>` with an `onClick` handler and no hover state, focus state, cursor change, or any visual difference from a static element). Both failure modes come from the same root cause: visual styling and interaction wiring are two separate steps, and generated or rushed UI frequently completes one without the other — a mockup or a component gets full visual treatment before the click handler exists, or a click handler gets added to an element that was never restyled to look interactive.

The "looks clickable but isn't" version is especially damaging to trust, because a user who clicks something that visually promises an action and gets nothing back doesn't just lose a moment — they start to doubt every other interactive-looking element on the page, since they no longer know which affordances are real.

```html
<!-- Generic: fully button-styled, but no handler wired up yet (a leftover from scaffolding) -->
<div class="button-style">Export CSV</div>

<!-- Generic (the reverse problem): functional but gives no visual affordance -->
<div onclick="deleteItem(id)">Delete</div>

<!-- Intentional: a real, semantic, fully-wired interactive element -->
<button type="button" class="btn-secondary" onclick="exportCsv()">
  Export CSV
</button>
```

- **DO:** Use real interactive elements (`<button>`, `<a>`, native form controls) for anything clickable, so hover, focus, active, and disabled states, keyboard operability, and screen-reader semantics all come for free rather than needing to be manually reconstructed on a generic `<div>`.
- **DON'T:** Style a non-interactive element (a `<div>`, a `<span>`) to visually look like a button or link and leave it non-functional, even temporarily during development — if it ships in that state (which happens more often than intended), it directly damages user trust the moment someone clicks it and nothing happens.
- **DO:** Give every genuinely interactive element a real hover and focus state that visually differs from its resting state, so users can tell at a glance — and via keyboard navigation — what on the page actually responds to input.
- **DON'T:** Attach click handlers to elements with no visual affordance change (no cursor change, no hover feedback) purely because it was faster than restyling. If it's clickable, it needs to look clickable; if it's not meant to be clickable, it shouldn't be wired up as if it were.
## How to Actually Fix It

Cataloging individual tells is necessary but not sufficient — a list of "don'ts" can be memorized and still produce generic output, because the underlying failure isn't any single pattern, it's a process that never asked for anything more specific than "make it look good." This section is about the process fix: how to prompt, brief, and review design work (whether the executor is a human or an AI) so that specific, intentional decisions get made instead of defaults filling every gap.

### Give Explicit Negative Constraints, Not Blind Trust

The single highest-leverage fix, and the one most specific to working with AI design/coding tools: state what to avoid, not just what to achieve. "Make a landing page for this product" reliably produces some version of the centered-hero-badge-pill template with an AI-purple gradient, because that's the highest-probability output for the prompt as given — there's nothing in it steering away from the statistically dominant pattern. "Make a landing page for this product; do not use a centered hero, do not use a purple/indigo gradient, do not use a badge pill above the headline, and do not default to Inter" produces something meaningfully different, because the negative space has been defined.

This isn't unique to AI tools — a human designer given zero constraints will also often reach for the most familiar, safest pattern under time pressure — but it's especially important for AI-generated work because a model has no independent taste to fall back on when a prompt is silent; silence gets filled with the training distribution's mode, which is, definitionally, the least distinctive possible answer.

- **DO:** Write explicit "don't do X" constraints into any design brief or prompt, especially naming the specific tells most likely to appear (no purple gradient, no badge pill, no emoji icons, no glassmorphism unless X) rather than relying on a general "make it good" or "make it modern" instruction.
- **DON'T:** Assume that describing the desired outcome ("modern," "clean," "professional," "premium") is enough to avoid generic output. These adjectives are exactly the words most associated with the generic default patterns in this document — they don't discriminate between a considered design and a templated one.
- **DO:** Maintain a running, product-specific "banned patterns" list as you notice AI tools or team members reaching for the same defaults repeatedly, and feed it back into future briefs — the list compounds in usefulness over time.
- **DON'T:** Treat negative constraints as a one-time setup step. Revisit and update them as new default patterns emerge (today's fix, once it becomes common enough itself, risks becoming tomorrow's new cliché) — the goal is intentionality, not a permanently fixed rulebook.

### Provide Reference Designs With Reasoning, Not Just Links

Dropping a link or screenshot of an admired design ("make it look like this") without explaining *why* it works transmits only the surface pattern, not the underlying principle — and a model or a developer working from surface pattern alone will often copy the most visible, easiest-to-imitate elements (the color palette, a specific layout trick) while missing the actual reason the reference succeeds (a genuinely well-thought-out information hierarchy, a considered relationship between content density and whitespace, a typographic choice that reinforces the brand's voice). The result is frequently a superficial visual echo of the reference with none of its actual functional or brand logic intact.

Articulating the "why" also protects against blindly reproducing a reference that's wrong for this specific product — a reference chosen because it's currently fashionable, rather than because its underlying reasoning actually applies to this brand's content, audience, or constraints.

- **DO:** When sharing a reference design, explain specifically what about it is worth emulating — "the way this pricing page uses a single strong color only on the recommended plan, so the eye goes there immediately" — rather than just "make it like this."
- **DON'T:** Share a reference with no annotation and expect the underlying design logic to transfer. Without an explanation of *why* a reference works, only its surface-level, most-copyable traits (a color, a layout shape) tend to survive into the new work — exactly the mechanism that produces derivative, trend-chasing results.
- **DO:** Choose references based on shared constraints or goals with the current project (similar content density, similar user goals, similar brand tone) rather than references chosen purely because they're currently well-regarded or trending — a beautiful reference solving a different problem can actively mislead.
- **DON'T:** Treat a single reference as sufficient. Multiple references, each contributing a specific, named principle (this one for hierarchy, this one for color restraint, this one for how it handles empty states) triangulate toward an original synthesis rather than a single imitation.

### Separate the Work Into Isolated Passes

Asking for "a complete, polished UI" in one shot compounds the risk of generic defaults, because every dimension — typography, color, layout, motion, content — gets decided simultaneously with no dedicated attention to any single one, and the fastest, most default-shaped answer for each dimension is what fills the gap when nothing is explicitly directing it otherwise. Splitting the work into deliberate, sequential passes — first get the layout and content hierarchy right with no color or type polish at all, then do a typography pass, then a color pass, then (if warranted) a motion pass — forces an actual decision at each layer instead of letting all of them default at once.

This mirrors how experienced designers already tend to work (grayscale wireframes before color, content structure before visual polish) and it has a specific additional benefit when directing an AI tool: each pass is a smaller, more reviewable unit of change, making it much easier to catch a generic pattern (a purple gradient introduced in the color pass, say) before it's tangled up with a dozen other simultaneous decisions.

```text
Generic single-shot prompt:
"Build a landing page for [product]." → everything defaults at once.

Passed approach:
1. "Lay out the page structure and content hierarchy in plain black-on-white,
    no styling decisions yet — just what content exists and in what order/emphasis."
2. "Now do a typography pass — propose a type scale and a typeface choice,
    with reasoning, before touching color."
3. "Now do a color pass — propose a palette derived from [specific brand input],
    apply it to the typographic layout from step 2."
4. "Now, only if it adds real value, propose specific, targeted motion —
    not a blanket scroll-animation treatment."
```

- **DO:** Break design requests into sequential passes — structure/hierarchy, then typography, then color, then (optionally) motion — reviewing and approving each before moving to the next, rather than requesting a fully polished result in a single step.
- **DON'T:** Ask for "a complete, beautiful UI" in one prompt or one sitting and expect every dimension to be handled with equal intentionality. Simultaneous decisions compound default-reaching behavior across every layer at once, and it's much harder to isolate and fix a bad decision buried inside an otherwise-finished-looking result.
- **DO:** Review each pass specifically against this document's tell list before moving to the next — check the layout pass for the centered-hero/three-box patterns, check the color pass for AI purple and gradient overuse, check the type pass for the Inter-default and trendy-pairing patterns.
- **DON'T:** Skip straight to visual polish before the content and hierarchy are settled. Applying beautiful typography and color to the wrong structure just produces a more convincing-looking version of the wrong answer, and structural problems are far more expensive to fix once buried under finished visual polish.

### Assign a Specific Point of View Instead of "Make It Look Good"

A generic instruction to "make it look good," "make it modern," or "make it professional" gives no discriminating signal — every one of the generic patterns in this document would technically satisfy that instruction, which is exactly why they're the default output. Assigning a specific point of view — a persona, a set of real constraints, a stated aesthetic lineage — gives the work something to differentiate against. "Design this as if for a technical, no-nonsense audience that distrusts marketing gloss, drawing more from developer-tool aesthetics than consumer-SaaS aesthetics" produces meaningfully different decisions at every layer than "make it look professional," because it rules things out as well as suggesting things in.

A point of view doesn't have to be elaborate to work — even a short, specific constraint set (three adjectives that are true of the brand and three that are explicitly *not* true of it) is enough to break the pull toward the generic default, because it gives every decision something concrete to be checked against.

```text
Generic brief: "Make it look modern and professional."

Point-of-view brief: "This is for compliance officers at mid-size banks —
an audience that is skeptical of anything that looks like a consumer startup.
It should feel more like a well-built internal tool than a marketing site:
dense information, minimal decoration, muted palette, no illustrations,
no marketing-speak headlines. Closer to a Bloomberg terminal than a Notion page."
```

- **DO:** Give any design brief a specific point of view — a named audience, an explicit aesthetic lineage ("closer to X than Y"), or a short list of adjectives the brand is and explicitly is not — so decisions have something concrete to be tested against at every layer.
- **DON'T:** Rely on generic descriptors like "modern," "clean," "professional," or "delightful" as the entire creative brief. These words are compatible with nearly every pattern in this document, generic and intentional alike, and provide no actual discriminating signal.
- **DO:** Make the point of view genuinely specific to this product and audience, not a borrowed persona from an unrelated, currently-fashionable brand. A persona chosen because it's true of this product will produce original decisions; a persona borrowed because it's currently admired elsewhere just relocates the imitation problem.
- **DON'T:** Confuse a strong point of view with a rigid style mandate that suppresses all creative judgment. The point of view should function as a filter that rules out mismatched choices, not a spec that dictates every pixel — there's still room for taste and iteration within it.

### Request Multiple Divergent Options Before Converging

Accepting the first generated result treats it as if it were already the best possible answer, when it's really just the single highest-probability answer — which, per every mechanism described above, tends to be the most generic one. Deliberately requesting two or three genuinely divergent directions (not three minor variations on the same layout, but different structural approaches, different typographic personalities, different color strategies) before picking one forces a comparison that a single output never invites, and comparison is what reveals whether the first instinct was actually a good fit or just the easiest one to produce.

This is a standard practice in professional design work for a reason — divergence-then-convergence surfaces options nobody would have thought to ask for directly, and it makes it much harder for a single generic default to slip through unexamined, since it's sitting next to visibly different alternatives that expose what it's lacking.

- **DO:** Ask for multiple genuinely different directions — not variations on a theme — before committing to one, especially for anything customer-facing or brand-defining (a landing page, a core product screen, a marketing identity).
- **DON'T:** Accept the first generated option as final, especially from an AI tool, on the assumption that "it looks fine" is the same as "it's the best available answer." The first output is the statistically likeliest one, which is a different property than being the best fit for this specific product.
- **DO:** Make the divergent options structurally different, not just palette-swapped versions of the same layout — different hero structures, different information hierarchies, different typographic personalities — so the comparison is actually informative rather than three shades of the same generic answer.
- **DON'T:** Ask for more options than can be meaningfully evaluated. Two or three well-differentiated directions beat ten superficially different ones; past a certain point, more options just adds decision fatigue without adding real signal.

### Audit a Finished UI Against a Concrete Checklist Before Calling It Done

Even with good intentions at every step above, generic patterns can still slip through individual decisions made under time pressure — which is why a final, explicit audit pass matters as a distinct step, not just a hoped-for byproduct of good process. A checklist works specifically because it names the exact patterns to look for (this document's Quick Checklist below is built for this purpose) rather than relying on a vague "does this look custom?" gut check, which is exactly the kind of unstructured judgment that let the pattern in in the first place.

The audit should happen as a deliberate, separate pass — after the work otherwise feels finished — specifically because it's hard to see your own defaults from inside the process that produced them; a checklist run with fresh eyes (or, ideally, by someone other than whoever built the UI) catches what momentum and familiarity blind the builder to.

- **DO:** Run a dedicated audit pass against a concrete, specific checklist (not a vague "does this feel generic?" gut check) before considering any UI work finished — treat it as a required step, the same as a code review, not an optional nicety.
- **DON'T:** Rely on "it looks fine to me" as the final quality gate. The person who built the UI is the least likely to notice its own defaults, precisely because those defaults felt natural and unremarkable while being chosen — a specific checklist catches what familiarity hides.
- **DO:** Have someone other than the original builder run the audit when possible — a second set of eyes, unfamiliar with the specific decisions made, is more likely to immediately spot "oh, this is the purple gradient thing" than the person who chose it under deadline pressure.
- **DON'T:** Treat the audit as complete after checking visual tells alone. The functional/UX gaps in this document (validation, empty/error states, accessibility, dead click targets) are just as important to the audit as the visual ones, and they're less visible in a quick look, so they need deliberate, explicit checking, not incidental notice.

### Use a "Filter, Not a Style Guide" Mindset

A style guide prescribes a specific aesthetic — this typeface, this palette, this exact spacing scale — and is the right tool once a brand identity actually exists and needs to be applied consistently. But this document, and the checklist that follows it, is deliberately not a style guide: it does not say "use this font" or "use this color" — it says "notice when a choice was made by default rather than on purpose," which is a different and more general kind of tool. A style guide answers "what should this look like"; a slop filter answers "did anyone actually decide this, or did it just happen."

The distinction matters because a rulebook that dictated one specific aesthetic (say, "never use gradients, always use Inter, always use flat colors") would just replace one set of unconsidered defaults with a different, equally unconsidered set — "sharp, flat, minimalist" is exactly as capable of being applied thoughtlessly as "soft, gradient-heavy, glassmorphic" is. The fix isn't a different default aesthetic; it's the discipline of checking whether *any* aesthetic choice, in either direction, was actually decided on purpose for this specific product.

- **DO:** Use this document's checklist as a filter that flags default, unconsidered patterns — regardless of which specific aesthetic direction a product ultimately takes — rather than as a prescription for one "correct" visual style.
- **DON'T:** Mistake "avoid the AI purple gradient" for "gradients are bad" or "avoid Inter" for "Inter is bad." The rule isn't about banning specific ingredients forever; it's about requiring a reason for whichever ingredient gets used, purple gradient and flat-grey-minimalism alike.
- **DO:** Pair this filter with an actual style guide once a product's brand identity is established — the filter catches default-driven slop during creation and audit; the style guide then keeps a *chosen* aesthetic consistent going forward. The two tools serve different stages and neither replaces the other.
- **DON'T:** Apply this document's checklist as if failing any single item is automatically disqualifying. A product with a genuine, well-reasoned brand fit for a purple gradient, or Inter, or a centered hero, passes the filter — the filter tests for the presence of a reason, not for the absence of any particular pattern on the list.
## Quick Checklist
An "AI Slop Detector" pass — run through this before calling any UI work done. Each item should have a real, stateable reason behind it; if the honest answer to any item is "no reason, it just ended up that way," that's the one to fix first.

**Typography**
- [ ] Primary typeface was chosen for a stated reason, not left at the Inter/Roboto/Arial default.
- [ ] If a second (serif/display) typeface is used, its role in the brand's actual voice is clear, not just "looks premium."
- [ ] No italic-serif accent word dropped into an otherwise plain sans-serif headline with no real emphasis purpose.
- [ ] A real type scale exists (5+ defined steps) — headings, body, captions, and labels are visibly distinct, not just varying shades of the same size.
- [ ] Line-height and letter-spacing are tuned per size step, not a flat `1.5` applied everywhere regardless of text size.
- [ ] Fallback font stacks are considered, not left as a bare single font name.

**Color**
- [ ] The palette wasn't defaulted to the violet/indigo/magenta "AI gradient" family without a brand-specific reason.
- [ ] Gradients appear in at most one deliberate spot per screen, not stacked across background, button, border, and text simultaneously.
- [ ] No gradient text-fill on headlines used as a reflexive "make it pop" move.
- [ ] Dark-mode body text passes 4.5:1 contrast against its background, checked with a real tool, not by eye.
- [ ] All-caps tracked labels are reserved for short metadata, not applied to every label and header by reflex.
- [ ] Colored glow/box-shadow effects follow one consistent system, not a different color and blur per element with no shared rule.
- [ ] The accent color is a deliberate brand choice, not the component library's unmodified default blue.
- [ ] Color alone is never the only carrier of meaning (status, category, error) anywhere in the UI.

**Layout & Components**
- [ ] The hero isn't a default centered-text-plus-badge-pill template chosen purely because it's fast to produce.
- [ ] Any badge/pill above a headline contains real, specific, true information — not vague filler like "✨ AI-Powered."
- [ ] Feature cards vary in length and visual weight according to actual importance, not forced to a uniform line count.
- [ ] Colored top/left borders on cards map to a real status or category system, not applied for arbitrary visual variety.
- [ ] Numbered "1, 2, 3" steps are used only where the process is genuinely sequential, not applied reflexively to independent actions.
- [ ] A numbered sequence isn't artificially compressed to exactly three steps when the real process has more.
- [ ] Every stat/metric displayed (user counts, ratings, uptime %) is real and currently true — none are placeholder numbers "to replace later."
- [ ] Navigation and persistent UI icons come from a real icon system (SVG/icon font), not emoji standing in as icons.
- [ ] Glassmorphism/backdrop-blur is only used where something meaningful sits behind it and a real z-axis relationship exists.
- [ ] The component library's default theme (radius, palette, shadows, type) has been intentionally overridden, not shipped as-is.
- [ ] Border-radius scales sensibly with element size, not one flat radius value applied to every component regardless of scale.
- [ ] Drop shadows are applied by actual elevation logic (flush vs. floating), not the same faint shadow on every surface.
- [ ] The feature grid isn't forced into exactly three icon-boxes purely because three fits a tidy row.
- [ ] Every testimonial, rating, and "as featured in" logo is real and attributable — none are fabricated placeholder social proof.
- [ ] Decorative background shapes (blobs/waves), if present, connect to the actual brand — not a generic downloaded filler asset.
- [ ] Scroll-triggered animations, if used, are applied selectively where motion adds real meaning, not uniformly to every section by default.
- [ ] `prefers-reduced-motion` is respected for any non-essential animation.

**Functional / UX**
- [ ] Every form field is validated (client- and server-side) with specific, actionable inline error messages next to the relevant field.
- [ ] Required fields are visibly marked before submission, not discovered only after a failed submit.
- [ ] Every view with async data has explicit loading, empty, and error states designed and implemented — not just the happy path.
- [ ] Empty states explain why the view is empty and offer a clear next action, not a bare "No results."
- [ ] User-facing error messages are in plain language with a next step — no raw stack traces or bare HTTP codes shown to end users.
- [ ] Every interactive element has a visible focus state (`:focus-visible`, not suppressed with `outline: none` and nothing replacing it).
- [ ] The entire primary user flow is operable by keyboard alone — tabbed through and verified, not just clicked through with a mouse.
- [ ] Elements styled to look clickable are actually wired up to real functionality — nothing that looks like a button does nothing.
- [ ] Elements that are functionally interactive have a real hover/focus affordance — nothing clickable looks static.
- [ ] Semantic, native interactive elements (`<button>`, `<a>`, form controls) are used instead of styled `<div>`s with manual click handlers.

**Process**
- [ ] The brief or prompt included explicit negative constraints (what to avoid), not only a description of the desired outcome.
- [ ] Any reference design shared came with an explanation of *why* it works, not just a link or screenshot.
- [ ] The work went through separate passes (structure, typography, color, motion) rather than being requested fully-polished in one shot.
- [ ] The brief gave a specific point of view or persona, not a generic "make it look good/modern/professional" instruction.
- [ ] Multiple genuinely divergent directions were considered before converging on one, rather than accepting the first output.
- [ ] A dedicated audit pass against this checklist happened after the work felt "finished," ideally by someone other than the builder.
- [ ] Every pattern used here has a stateable reason tied to this specific product/brand — this list is a filter for unconsidered defaults, not a mandate for one fixed aesthetic.
