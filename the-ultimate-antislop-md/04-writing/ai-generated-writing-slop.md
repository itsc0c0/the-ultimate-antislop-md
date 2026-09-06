# AI-Generated Writing & Content Slop

Writing has its own version of the genericness problem described in Part 1: text that is fluent, grammatically correct, and superficially well-organized, but that says less than it appears to, because it defaults to the most statistically common phrasing, structure, and rhetorical moves rather than being shaped by anything specific to what it's actually about. This section catalogs the recognizable tells of that failure mode and what to do instead.

None of these patterns are wrong in isolation — every phrase and structure named here appears in good writing sometimes. The tell is *reflexive, unexamined use*: reaching for "in today's fast-paced world" because it's the default way to open, not because this particular piece of writing needed a scene-setting opener; using a three-item list because three feels complete, not because there happen to be exactly three things worth saying.

## Generic Openings & Throat-Clearing

- **DON'T:** Open with a broad, contextless scene-setting sentence that could preface almost any piece of writing on almost any topic — "In today's fast-paced digital world," "In an era of rapid technological change," "Communication is the cornerstone of any successful relationship." These sentences delay the actual content and signal, from the first line, that what follows will be generic.
- **DO:** Open with the most specific, concrete, or important thing the piece has to say — a real claim, a real detail, a real question — and let context emerge from that rather than being established before it.
```text
BAD:  "In today's fast-paced business environment, effective
      communication has never been more important. Companies
      that fail to communicate well with their teams risk..."

GOOD: "Our last three product launches slipped because engineering
      and marketing found out about the ship date on different
      days. Here's what we're changing."
```
- **DON'T:** Restate the prompt or question back at the reader before answering it ("Great question! Let's explore what makes a good password manager."). If the piece is answering a question, answer it — the question doesn't need to be repeated first.
- **DON'T:** Open a piece of technical writing (a README, a doc, an announcement) with an unearned claim of importance or excitement ("We're thrilled to announce...", "This is a game-changing update...") before the reader has any information to evaluate whether that's true. Let the substance earn the reaction; don't instruct the reader to have it.
- **DON'T:** Begin an explanation by defining a term everyone in the actual audience already knows, as a stalling move before getting to the substance ("Before we dive in, let's define what an API is..." in a piece written for developers).

## Overused Vocabulary & Phrase Tells

Certain words and phrases have become recognizable markers of generic AI-adjacent writing, less because any one of them is bad and more because of how disproportionately often they appear relative to how people actually talk and write. Treat frequent, reflexive use of any of the following as a signal to find a more specific, more ordinary word instead.

- **DON'T:** Reach for "delve into," "navigate the landscape of," "unpack," "explore the nuances of," or "dive deep into" as a default way to say "discuss" or "look at." These phrases have become so common in generic AI writing that their presence is now itself a tell, independent of whether the sentence is otherwise fine.
- **DON'T:** Default to "boast," "showcase," "testament to," "underscore," "tapestry," "realm," or "elevate" where a plainer, more specific word would say the same thing more directly. "This library boasts an extensive feature set" says less than naming the three features that actually matter.
- **DON'T:** Overuse "leverage," "utilize," "facilitate," "robust," "seamless," "holistic," "cutting-edge," "game-changer," "unlock," or "unleash" as filler intensifiers that inflate a claim without adding information. "Utilize" almost always just means "use"; "leverage" almost always just means "use" too.
- **DON'T:** Reach for "it's important to note that," "it's worth mentioning," or "needless to say" as a throat-clearing lead-in to a point, rather than just making the point. If it's worth saying, say it; the phrase announcing that it's worth saying adds words without adding content.
- **DON'T:** Use "not just X, but Y" or "not only X, but also Y" as a reflexive sentence template applied to nearly every claim in a piece, regardless of whether the contrast is actually doing rhetorical work. Overused, the construction stops signaling emphasis and starts signaling padding.
```text
BAD (used constantly throughout a piece):
  "This isn't just a bug fix — it's a fundamental rethinking of
  how the system handles errors."
  "This isn't just faster — it's a completely new way of thinking
  about performance."
  "This isn't just a feature — it's a philosophy."

GOOD: Use the construction where the contrast is real and rare
      enough to still carry weight, and otherwise just state the
      claim directly: "This changes how the system handles
      errors" / "This is significantly faster" / "This adds
      undo support."
```
- **DON'T:** Reach for "in the ever-evolving world/landscape of X" or "in today's digital age" as a default framing device. These phrases add a vague sense of context without adding any actual context.
- **DO:** When an intensifier or a strong claim is used, back it with a specific reason or example in the same breath, rather than letting the strong word do the work alone. "This is a game-changer" is empty; "this cuts our deploy time from 40 minutes to 90 seconds" is not.
- **DON'T:** Default to "foster," "cultivate," or "champion" as verbs applied to abstract nouns ("foster collaboration," "cultivate innovation," "champion diversity") when a more concrete, specific verb and object would say what's actually meant. What, specifically, is being done?

## Structural Tells

- **DON'T:** Structure every piece of writing — regardless of subject — as a numbered or bulleted list, especially a suspiciously round one (top 5, top 10). Not every topic decomposes naturally into a clean enumerated list; forcing one onto a subject that's actually more continuous or interconnected loses the connections between points to preserve a tidy structure.
- **DON'T:** Default to exactly three examples, three reasons, or three steps as a rhetorical reflex ("There are three key benefits..."), independent of how many things are actually true or relevant. The rule of three is a real rhetorical device with real force when the content actually has three natural, roughly-equal-weight parts — used as a template regardless of the actual content, it becomes a tell rather than a technique.
- **DON'T:** Give every section of a long piece uniform length and identical internal structure (same number of paragraphs, same rhythm of short-sentence-then-explanation) regardless of whether each section's content actually calls for that much space. Real writing has uneven pacing because real subjects have uneven weight — a point that needs one sentence and a point that needs five paragraphs shouldn't be forced into matching shapes.
- **DON'T:** Use a rigid, repeated paragraph template across a whole piece — topic sentence, three supporting sentences, transition sentence — applied mechanically to every paragraph regardless of what that specific paragraph needs to do.
- **DO:** Let structure follow from the content's actual shape. A comparison might genuinely want a table; a process genuinely wants numbered steps; an argument genuinely wants prose that can hold a chain of reasoning together in a way a bullet list would fragment.
- **DO:** Vary sentence length deliberately — a long, complex sentence followed by a short one is a real technique for controlling pace and emphasis; an unbroken run of similarly-structured, similarly-length sentences reads as monotonous regardless of whether each sentence, individually, is correct.
- **DON'T:** Let uneven editing quality slip through a finished piece — a document arguing against sloppiness is judged, fairly, by whether it practices what it argues for. Proofread structure and formatting with the same rigor as content.

## Hedging, Qualifiers & False Balance

- **DON'T:** Hedge a claim that the evidence actually supports plainly — "it could potentially be argued that this might, in some cases, be somewhat faster" when the honest claim is just "this is faster." Excessive hedging isn't epistemic humility; past a certain point it's a way of avoiding commitment to a position, which makes writing less useful even when it's technically more cautious.
- **DO:** Calibrate hedging to actual uncertainty. State what's confidently known plainly. Flag what's genuinely uncertain clearly, with the specific reason for the uncertainty, rather than applying a uniform layer of vague qualification to everything regardless of confidence level.
- **DON'T:** Manufacture "on the other hand" balance for questions that don't actually have two comparably strong sides, purely for the appearance of fairness. If the evidence clearly favors one answer, present it as favoring one answer; false balance misinforms just as surely as false certainty does.
- **DON'T:** Bury a direct answer inside several sentences of qualification and context such that a reader has to extract the actual answer themselves. Lead with the answer, then add the caveats and context that make it precise — not the reverse.
```text
BAD:  "There are many factors to consider when it comes to
      choosing a database, and different use cases may call for
      different solutions, and it's worth noting that there isn't
      necessarily a single right answer, but generally speaking,
      for your described use case, Postgres would likely be a
      reasonable choice to consider."

GOOD: "Use Postgres. Your workload is mostly relational with a few
      JSON fields — Postgres handles both natively, and you
      already have ops experience with it. The one case that
      would change this: if you need multi-region active-active
      writes, look at something built for that instead."
```
- **DON'T:** Use "some might argue" or "it could be said" as a way to introduce a viewpoint without actually attributing it to anyone, as a hedge against being wrong. Either the viewpoint is worth engaging with directly (in which case, engage with it), or it isn't (in which case, don't gesture at it vaguely).

## Filler Transitions & Padding

- **DON'T:** Default to "moreover," "furthermore," "additionally," and "in conclusion" as the primary way to connect ideas across a whole piece, especially when they appear at a similar rate throughout. These words aren't wrong, but overreliance on them as connective tissue is a specific, recognizable rhythm that substitutes for actually showing how ideas relate to each other.
- **DON'T:** Restate, at the start of a paragraph or section, what the previous paragraph or section just said, before adding anything new ("As mentioned above, X is important. Now let's look at Y."). If the connection between two ideas is genuine, it can usually be shown by how the new material is framed, without a separate sentence whose only job is to announce that a transition is happening.
- **DON'T:** Write a "roadmap" sentence or paragraph that describes what the piece is about to cover, in a piece short enough that the reader will reach the actual content within a few more sentences anyway ("In this article, we will explore X, then discuss Y, before finally covering Z."). This is sometimes genuinely useful in a long technical document with real navigational value; it is padding in a 400-word blog post.
- **DO:** Cut any sentence that could be deleted without losing information — a strong editing test is to remove a sentence and check whether anything specific was actually lost, or whether the piece reads identically minus a few words of connective filler.
- **DON'T:** Repeat the same point in slightly different phrasing across consecutive sentences or paragraphs as a way of appearing to elaborate, without actually adding a new angle, example, or piece of evidence each time.

## Empty & Generic Conclusions

- **DON'T:** Close a piece by restating the introduction in slightly different words ("In summary, as we've seen, X is important for reasons Y and Z...") without adding anything the reader doesn't already have from having just read the piece. A conclusion that could be swapped onto a different article about a different topic with minor edits is a generic conclusion.
- **DON'T:** End with a vague call to action or forward-looking statement that commits to nothing specific ("The future of X looks bright," "Only time will tell," "As technology continues to evolve, one thing is certain..."). These sentences are true of almost any topic and therefore convey almost nothing about this one.
- **DO:** End a piece by doing something the introduction and body haven't already done — state a concrete recommendation, name a specific next step, pose a genuinely open question the piece has earned the right to ask, or simply stop when the last substantive point has been made. Not every piece needs a separate "conclusion" section; sometimes the strongest ending is the last concrete point, with no summary wrapper around it.
- **DON'T:** Close a technical explanation with an unearned inspirational flourish disconnected from the technical content ("And that's the power of well-designed APIs — they don't just connect systems, they connect people.").

## Formatting Tics

- **DON'T:** Overuse the em dash as a default punctuation choice in nearly every paragraph, especially in place of a period, comma, or colon that would read more naturally. A single em dash used well is a legitimate, useful piece of punctuation for a genuine interruption or aside; a piece where every third sentence contains one reads as a stylistic tic rather than a deliberate choice.
- **DON'T:** Bold or italicize words or short phrases throughout a piece as a way of simulating emphasis or importance, applied so frequently that the formatting stops meaningfully distinguishing anything (if everything is emphasized, nothing is). Reserve emphasis formatting for the genuinely rare word or phrase that needs it.
- **DON'T:** Use Title Case For Every Heading And Subheading in running prose or informal writing, by default, regardless of the publication's actual house style. Match the capitalization convention the context actually calls for (sentence case is often more appropriate and more common for informal or technical writing) rather than defaulting to title case as if it were universal.
- **DON'T:** Sprinkle emoji into headings, bullet points, or section labels as a default decoration (✅, 🚀, 💡, 🎯) unless the context (a casual chat message, a specific brand voice that has established this) actually calls for it. In most professional and technical writing, emoji used this way reads as an unearned attempt at friendliness or energy rather than as genuine tone.
- **DON'T:** Overuse exclamation points as a substitute for actually interesting content ("Get started today!", "It's that easy!"). An exclamation point asserts enthusiasm; it doesn't create it for the reader.
- **DO:** Let formatting serve the content's actual structure — headings that mark real sections, bold that marks the rare genuinely critical phrase, lists where the content is actually a list — rather than applying formatting devices uniformly as decoration.

## Tone & Voice Problems

- **DON'T:** Write in a flattened, corporate-neutral register regardless of context — a tone that avoids any specific personality, opinion, or texture in favor of being inoffensive to everyone. Writing that could have been produced by swapping the subject and reads identically is writing that isn't actually saying anything distinctive about its subject.
- **DON'T:** Perform enthusiasm the writer/piece doesn't actually have grounds for ("We're so excited to share...", "You're going to love this...") as a default affective register for announcements, regardless of whether the actual content justifies that level of feeling.
- **DON'T:** Write with reflexive, unearned confidence about contested or uncertain claims, using the same flat declarative tone for well-established facts and for genuinely disputed or speculative ones — this erases a distinction the reader needs in order to calibrate how much to trust each claim.
- **DO:** Let the confidence level of the language track the actual confidence level of the claim — hedged language for genuinely uncertain claims, plain declarative language for well-established ones, and a clear signal (not just tone, but explicit words) when moving from one register to the other.
- **DON'T:** Address the reader with generic, unearned intimacy ("As you know...", "We've all been there...") that assumes a shared context or experience the writer has no actual basis for assuming.

## Lack of Specificity & Concrete Detail

- **DON'T:** Make a claim that would remain equally true if every specific noun in it were swapped for a different one ("This tool helps teams work more efficiently and achieve better results.") — true of almost any tool, which means it conveys almost nothing about this one.
- **DO:** Replace generic claims with specific, checkable ones wherever possible: numbers, named examples, concrete before/after comparisons, specific scenarios. "Cuts onboarding time from two weeks to three days" beats "significantly improves onboarding efficiency."
- **DON'T:** Use a vague intensifier ("very," "significantly," "substantially," "a lot") in place of an actual quantity when the actual quantity is known or knowable. If a change made something 40% faster, say 40% faster — "significantly faster" throws away information the writer actually had.
- **DON'T:** Illustrate a point with a hypothetical, generic example ("Imagine a user named Alex who wants to buy a product...") when a real example, real data, or a more specific scenario is available and would be more persuasive and more informative.
- **DO:** Prefer one well-chosen, specific example over three generic ones — specificity does more persuasive and explanatory work than repetition of the same level of abstraction.

## SEO & Marketing Slop

- **DON'T:** Stuff a target keyword or phrase into a piece of writing at an unnatural frequency, in a way that damages readability, purely to satisfy a search-optimization heuristic. Content written primarily to be found rather than to be read tends to be recognizably worse at being read.
- **DON'T:** Write a headline that overpromises relative to the actual content ("This One Trick Will Change How You Code Forever" for an article about a minor linter configuration tweak). A reader who feels misled by the headline discounts everything that follows, including the genuinely useful parts.
- **DON'T:** Pad an article's length specifically to hit a target word count, by restating points, adding generic filler sections ("What is X?" sections that define terms the target audience already knows), or expanding a genuinely short answer into an artificially long one. Length that doesn't carry proportional information is a cost to the reader, not a benefit.
- **DON'T:** Front-load a piece with a long list of "benefits" or "features" stated abstractly before any concrete content, as a marketing reflex, when the reader came for the concrete content itself.
- **DO:** Write for the reader who is actually going to read the piece, optimizing for them getting genuine value quickly — search and marketing performance follow from that far more reliably than they follow from optimizing directly for search/marketing heuristics at the expense of the reader.

## How to Actually Catch This in Your Own Writing

- **DO:** Read a draft aloud, or have it read aloud. Phrases that feel fine on the page ("navigate the ever-evolving landscape") often sound obviously wrong, stilted, or hollow when heard, because spoken language rarely uses them.
- **DO:** Delete the first paragraph and the last paragraph of a draft as a test, and check whether the piece is actually worse off. Generic openings and generic conclusions are disproportionately likely to be the parts that can be cut with no real loss — if that's true, cut them for real.
- **DO:** Search a draft for the specific overused words and phrases listed in this section (delve, boast, testament, leverage, utilize, robust, seamless, unlock, "not just X but Y," "it's important to note") and treat each hit as a prompt to consider a more specific alternative — not an automatic rule to delete every instance, since any of these words can be the right word sometimes, but a deliberate check rather than a blind spot.
- **DO:** Check whether any claim in the piece would survive having its specific nouns replaced with different ones. A claim that stays equally true about a different product, company, or topic isn't actually saying anything about this one — replace it with something that wouldn't survive that swap.
- **DO:** Compare a draft's paragraph lengths and internal structure across sections. Uniform rhythm throughout a long piece is a signal worth checking — real subjects rarely have perfectly even weight distributed across every section.
- **DO:** For anything published under a specific voice or brand, check the draft against real examples of that voice's actual past writing, not just against general "good writing" heuristics — genericness often specifically means "sounds like nothing in particular" rather than "sounds like anyone else in particular," and the fix is restoring the specific voice, not applying a different generic one.

## Quick Checklist
- No contextless scene-setting opener ("in today's fast-paced world...").
- Lead with the most important/specific thing, not a warm-up.
- Cut delve, boast, testament, tapestry, realm, foster, elevate, unlock, unleash, leverage, utilize, robust, seamless, holistic, cutting-edge, game-changer wherever a plainer word works.
- Cut "it's important to note," "needless to say," and similar throat-clearing lead-ins.
- Use "not just X, but Y" sparingly — only where the contrast is real, not as a sentence template.
- Don't force every topic into a numbered/bulleted list structure.
- Don't default to exactly three examples/reasons/steps regardless of what's actually true.
- Vary section length and paragraph rhythm to match actual content weight.
- Calibrate hedging to actual uncertainty — plain language for confident claims, specific caveats for uncertain ones.
- Don't manufacture false "on the other hand" balance for settled questions.
- Lead with the direct answer, then add caveats — don't bury the answer in qualification.
- Cut filler transitions (moreover, furthermore, in conclusion) used as a rhythm rather than a real connector.
- Don't restate the previous paragraph before adding a new one.
- Don't summarize-restate the intro as the conclusion — end on something new or just stop.
- No vague, noncommittal closing lines ("only time will tell," "the future looks bright").
- Use em dashes deliberately, not as reflexive default punctuation in most sentences.
- Use bold/italic sparingly enough that emphasis still means something.
- Match heading capitalization to actual house style instead of defaulting to Title Case.
- Skip decorative emoji in headings/bullets unless the context genuinely calls for it.
- Skip exclamation points as a substitute for actually interesting content.
- Write in a specific, real voice rather than a flattened corporate-neutral register.
- Don't perform enthusiasm the content doesn't earn.
- Replace vague intensifiers ("very," "significantly") with real numbers when the number is known.
- Prefer one specific, real example over several generic ones.
- Don't keyword-stuff or pad length to hit a target word count.
- Don't write a headline that overpromises relative to the actual content.
- Read drafts aloud and test-delete the first/last paragraph before calling a piece done.
