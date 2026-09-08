# landing-page

Assumes a marketing or conversion-focused page in a web app: hero, problem, solution, pricing,
FAQ, footer. Examples use React/Next.js/Tailwind conventions where a concrete example helps.
The rules hold for any stack that renders a page; swap the component names.

When an upstream brief or PRD defines the page's section list, that list is authoritative. A
rule below that would add a section (a trust wall, a comparison block, an extra FAQ) downgrades
to a flagged, not-silently-applied recommendation when the brief doesn't call for it: never
fail a check for the intentional absence of a section the brief deliberately omitted.

## Rule 1: Hero Headline Passes the 4Us Test

Before scoring anything else, proofread the headline: grammar, subject-verb agreement,
consistent capitalization. A grammatically broken headline invalidates every other check. Then
it must score on all four: **Useful** (states the outcome, not the category: "Learn 500
Spanish words in 30 days" beats "The best language app"), **Urgent**, **Unique**, and
**Ultra-specific** (a real number, not a vague claim). Under 10 words. For a problem-aware
audience, lead with the felt problem, not the solution — naming the answer before the reader
feels the question reads weaker. A question headline must not be one the reader can
comfortably answer "no" to ("Frustrated with X?" invites a bail-out).

## Rule 2: Hero Copy Cascade Steps Down Without Repeating Itself

The hero is a headline + subheadline, or a three-tier cascade: headline (hook) → subheadline
(a concrete, vivid scene) → elaboration (reframe + forward payoff into the CTA). Step the type
size down across tiers and group them tight so they read as one unit: without a visible size
step the tiers blur together. Each lower tier must add a mechanism, the audience, or a concrete
detail; if it shares more than half its key nouns/verbs with the headline (synonyms count),
it's redundant, not additive. The concrete middle tier stays outcome-agnostic — it must not
assert a result the product's own outcome, or an adjacent card, could contradict. The
CTA-adjacent phrase is forward-looking ("Find your next move"), never deficit-framed ("Find
what to fix") or hyped ("Unlock", "Supercharge").

## Rule 3: One Primary CTA Above the Fold

Exactly one primary CTA button visible without scrolling, in action-first outcome language
("Start Learning Free"), never "Submit" or "Click Here." It must be the highest-contrast
element on the page; a muted CTA is a bug, not a style choice. At most one secondary CTA, and
it's a text link, not a second button: multiple competing buttons above the fold split
attention and can cut conversion by as much as 266%.

## Rule 4: Hero Visual Shows the Product or the Outcome

A product screenshot, a short demo loop, a real user, or the aspirational outcome — never
abstract or generic stock imagery. A visitor who can't tell what the product looks like from
the hero keeps scrolling to find out whether it's worth their time; most won't.

## Rule 5: Social Proof Sits Near Every CTA, Not Just the Footer

A one-line proof element — customer count, star rating, named logos — belongs near the hero
CTA and near every repeated CTA down the page, not only in the footer where nobody scrolls to
verify a claim before they've already decided to leave. A 4.0-4.7 star rating is the
conversion-maximizing range; a perfect 5.0 reads as fake.

## Rule 6: Above-the-Fold Content Renders and Performs Independently of JS

The hero must render fully without waiting on client JS — server-render or statically generate
it — and stay usable if below-the-fold scripts fail entirely. Reserve exact dimensions for the
hero image/video so nothing shifts after paint. Budget the three metrics that move conversion:
LCP under 2.5s (preload the hero image, next-gen formats), INP under 200ms (defer non-critical
JS), CLS under 0.1 (explicit width/height on every image). Lazy-load below the fold and defer
third-party scripts until the main thread is idle — a visitor from an ad who hits blank white
space is gone before the JS finishes loading.

## Rule 7: Navigation Matches the Page's Traffic Type

A dedicated landing page built for paid or campaign traffic carries no site navigation: every
nav link is an escape route before the visitor reaches the CTA. The only exceptions: a logo
linking home, a single sign-in link. A homepage or organic-traffic page may keep navigation,
but flag a site with 3+ content categories for a mega menu or grouped dropdown — hub pages as
columns, spoke pages as rows; 6+ categories makes it a hard requirement. Never ship a flat
10+-link list on desktop, or any flat list on mobile. The footer is a separate, sitewide
link-equity channel: don't prune it for "cleanliness" on an SEO-driven site. 30-40 organized
links covering every hub and spoke is normal, and every hub page must reach its spokes in two
clicks.

## Rule 8: One Copywriting Framework, Applied End to End

Pick one framework — PAS, AIDA, StoryBrand, BAB, PASTOR, SCQA, QUEST, Hook-Story-Offer — and
run every section through its stages; a page that opens in PAS and drifts into AIDA by the
features section reads as disconnected because it is. Choose by drafting the hero copy in
several candidate frameworks and picking the winner, not by defaulting to whichever one you
always use. Match posture to audience awareness: problem-first frameworks win for a
problem-aware audience, benefit-led frameworks for one that already knows the solution
category. In a crowded category, add an explicit "old way vs. new way" contrast block naming
the category default and standing against it: every claim must be literally true; overclaiming
what competitors do or don't do is a false-exclusivity failure, not positioning.

## Rule 9: Problem Section Names Specific Pain, Never the Product

State the reader's frustration in "you" language, addressing the internal emotional pain, not
just the external symptom ("You study a word, feel confident, then forget it three days later"
beats "Learning vocabulary is hard"). 3-5 pain points as bullets or short cards. No product
mention — the reader has to feel understood before the solution shows up, or it reads as a
sales pitch instead of a rescue.

## Rule 10: Solution Section Makes the Brand the Guide, Not the Hero

The customer is the hero; the brand is the guide (StoryBrand). Bridge naturally from the
problem section, introduce the product by name with a one-line description, and explain the
mechanism: why this works when the obvious alternatives don't. Lead with empathy ("We know
the frustration of...") before authority — authority-first reads as the brand talking about
itself instead of the reader's problem.

## Rule 11: How It Works Caps at 3 Steps

3 steps, 5 at the absolute maximum, each one sentence plus an icon. Steps describe the user's
actions ("Practice for 5 minutes"), never the backend process ("Our AI analyzes your data").
This section exists to kill the "this seems complicated" objection, and a step written from the
system's point of view reintroduces the complexity it was supposed to remove.

## Rule 12: Every Feature Completes Feature → Advantage → Benefit

A feature listed without its advantage and benefit stops at "what it does" and never answers
"so what" — the reader has to do the persuasive work the copy owed them. Group by user goal,
not product architecture; 4-6 features is the sweet spot, more than 8 causes decision fatigue,
and the 2-3 most differentiated features lead the list.

## Rule 13: Testimonials Follow Problem → Result → Outcome, With a Name Attached

Minimum 3, each with a full name and photo (or video) and a specific number pulled out as a
featured stat — "Great product!" with no name and no number adds no persuasive weight and
should be cut, not kept as filler. Include a job title or context line. Anonymous testimonials
measurably reduce trust; don't ship one. Never invent a testimonial quote, customer name,
statistic, or case-study number to fill a content slot: use a clearly marked placeholder (e.g.
`[TESTIMONIAL: needs real customer quote]`) and flag it in the report. A fabricated number that
makes the layout look finished is worse than an honest gap.

## Rule 14: Trust Signals Sit Next to Every CTA

A guarantee badge, security seal, or privacy line belongs adjacent to every CTA, not only in
the footer: 61% of users abandon purchases when trust seals are missing, and that abandonment
happens at the CTA itself, where a footer badge is already scrolled out of view. Privacy
assurance ("We never share your email") belongs next to every email form.

## Rule 15: Pricing Uses One Recommended Tier, Anchored and Bounded

3 plans maximum, one marked "Recommended," each with a "Best for" line and its own CTA copy:
not identical text across tiers. The recommended tier's CTA uses the primary button style;
every other tier uses secondary or outline — identical styling erases the hierarchy the
recommendation depends on. Never list a feature that isn't deliverable at launch (pricing
tiers are a contractual promise, not a roadmap slide); flag "all," "unlimited," "everything,"
or "infinite" unless the offering is truly unbounded and sustainable: replace with a real cap
("Up to 10,000 API calls/month").

## Rule 16: FAQ Is the Safety Net — Objection Handling Is Not Optional Elsewhere

Identify the top 3-5 purchase objections for the category and resolve them in the page's main
flow, not only in the FAQ: an objection answered exclusively inside a collapsed accordion item
is functionally unanswered. When planning a page, produce an explicit objection-to-section map
(e.g., "Is it hard to use?" → How It Works; "Can I trust this company?" → Testimonials + Trust
Signals) as a required deliverable, mapping to sections that already exist rather than
inventing new ones. The FAQ still needs 5-10 questions covering pricing, cancellation,
commitment, and comparison, in 2-4 sentence answers, placed directly below pricing. Risk
reversal (money-back guarantee) must appear near both pricing and the final CTA.

## Rule 17: Final CTA Uses a Different Emotional Frame Than the Hero

If the hero CTA sells desire/aspiration, the final CTA sells relief from pain or urgency: the
same frame twice is one persuasive attempt repeated, not two. Include a micro-testimonial or
result stat and a risk-reversal reminder; keep the section simple — headline, one or two lines,
button, trust signal.

## Rule 18: Forms Start at 3 Fields With Anxiety-Reducing Micro-Copy

3 fields maximum to start (email-only on step one maximizes capture). Labels sit above the
input, never only as placeholder text that vanishes once the user starts typing: a user who
forgets what a field wants because the label disappeared abandons the form. Inline, real-time
validation, not submit-time errors. Multi-step forms show a progress bar. Micro-copy under the
submit button ("No credit card required," "Takes 30 seconds") reduces abandonment; only mark
truly optional fields "optional."

## Rule 19: Mobile Is the Primary Design, With a Sticky CTA

Design mobile layout first, desktop as the enhancement. Every tap target is at least 44x44px.
A sticky CTA that stays visible through scroll is the single highest-leverage mobile
conversion change — ship it. Stack multi-column sections to one column on mobile and remove
hover-only interactions; a hover-triggered CTA doesn't exist on a touchscreen, which means it
doesn't exist for most of the traffic.

## Rule 20: Headline Mirrors the Traffic Source

When the page receives paid traffic, the hero headline matches the ad's exact language — even
a faithful rephrase reads as a mismatch to a visitor primed by the ad, and a visitor who
doesn't see the message they clicked for bounces immediately. For organic traffic, match the
likely search intent instead.

## Rule 21: Visual Hierarchy Survives a Skim

Short paragraphs (2-3 lines max), bullets over walls of text, bold on key phrases, generous
whitespace, one typeface with a clear size hierarchy. Every section headline must communicate
its value on its own: if a reader who skips every body paragraph and reads only headlines
can't reconstruct the page's pitch, the hierarchy has failed.

## Rule 22: Urgency Must Be Real

A genuine urgency trigger (deadline, limited spots, early-bird pricing) is fine to state
directly. A fake countdown timer or a "only 3 left" claim against unlimited inventory is a
trust violation the moment a visitor notices, and it costs more trust than urgency's absence
would have. With no genuine trigger, use implied urgency ("every day you wait is a day of
progress lost") instead of manufacturing a fake one.

## Rule 23: CTA Repeats at Consistent Intervals, With One Label Per Destination

Repeat the primary CTA after the hero, after social proof, after pricing, and as the final
section: 3-4 placements on a long page; fewer than 2 under-serves it, more than 5 reads as
aggressive. Same button style and text throughout for one destination. Check the whole
codebase, not just the page under review: if `/pricing` is labeled "View Pricing" on one page
and "See Plans" on another, grep for CTA elements and their hrefs across pages and unify the
label.

## Rule 24: Contrast, Text Size, and Keyboard Access Are Non-Negotiable

Body text meets WCAG AA (4.5:1 contrast, 3:1 for large text), never smaller than 16px, with
1.5-1.75 line height. The CTA button is the highest-contrast element on the page. Every
interactive element is keyboard-reachable (Tab, Enter, Escape); every image has alt text
(`alt=""` for decorative ones); every form input has a real `<label>`, not a placeholder
standing in for one; color is never the sole indicator of state; semantic HTML (`<nav>`,
`<main>`, `<section>`) replaces a page of unstructured `<div>`s.

## Rule 25: SEO Foundations Are Present on Every Page

Unique `<title>` (50-60 chars) and `<meta description>` (150-160 chars) with the primary
keyword and a reason to click. Exactly one `<h1>` — the hero headline — with `<h2>` below it.
Clean canonical URLs with no query params. Schema markup for FAQ and Product/Organization where
applicable, plus Open Graph and Twitter Card tags: a page missing OG tags shares as a bare
link with no preview, which suppresses social click-through.

## Rule 26: Every Major Section Earns Its Persuasion, Not Just Its Layout

A section that only lists information without doing persuasive work is a section that could be
cut. Check each against LIFT — does it increase Value Proposition, Relevance, Clarity, or
Urgency, and decrease Anxiety or Distraction — and against 2-3 behavioral principles suited to
its type: named individuals and specific numbers in testimonials and pain points (not aggregate
stats), anchoring and a decoy tier in pricing, loss-aversion framing in CTA copy, a progress
indicator in multi-step forms. The hero must score on all six LIFT factors; other sections need
2-3, and none should actively work against one.

## Rule 27: The CTA's Promise Must Be Kept

Two failure modes, same root cause: the label lies about the destination, or the destination
isn't ready to receive the click. Match the label to what actually happens — a contact form
gets "Talk to Us," a booking link gets "Book a Call," checkout gets "Buy Now," a docs link
gets "Read the Docs"; a generic "Learn More" is acceptable only when the destination type
genuinely isn't known yet. And every href must resolve to content that matches the page's
promise: "coming soon," placeholder copy, or a booking calendar with no open slots is worse
than a 404, because it costs the trust of a visitor who was ready to convert. Both are trust
violations at the exact moment of highest intent; flag unready destinations before the CTA
ships pointing at them.

## Rule 28: Plain Register — Contractions, Bounded Antithesis, Short Headings

Use contractions by default ("don't," "you're"): a no-contractions rule produces
legal-notice register, the fastest tell that copy was machine-written; override only for an
explicit brand guideline, applied consistently. The "not X, but Y" negation pivot is allowed
once per page, reserved for the core positioning line: used more, it reads as performing
cleverness instead of stating a claim. Utility sections take their plain, universal names
("Frequently asked questions," "Trusted by") with no clever lede reframing them as a
value-claim heading. Section headings stay under 8 words. Em dashes are a similar tell: one
per page for genuine emphasis is fine, more than two is a style smell — replace with a comma,
colon, period, or hyphen depending on what the dash was doing.

## Rule 29: Copy Never Talks About the Page, the Site, or Its Own Restraint — or Repeats Itself

Four sub-patterns, all banned because real sites never mention their own furniture: **IA
narration** ("described briefly here rather than sold as its own product" — the visitor has no
concept of your page tree; delete it). **Deixis** ("above," "below," "this page": breaks the
moment the section reorders or the copy gets reused in a search snippet). **Policy
narration**: enact restraint, don't announce it; "not a lock-in trick" plants the accusation
it denies. **Internal vocabulary**: no tier/epic/gate labels, codenames, or roadmap-stage
names leaking from a planning doc into visitor-facing copy. A fifth, related tell crosses
pages rather than sitting within one: the same positioning *idea* can recur sitewide, but the
same distinctive sentence repeated verbatim across pages reads as templated, because it
usually is: copy generated per page from one prompt instead of written for each page's
audience moment. Two uses of a signature phrase is a deliberate echo; three or more is
recycling. Watch too for copy padded to match a neighboring card's character count: fix the
layout (align-to-top, min-height), never the copy.

## Rule 30: Grid Item Count Is a Multiple of the Column Count

A 3-column grid with 4 items leaves a lone item stranded in a half-empty row. Before shipping
any repeating grid — features, pricing cards, testimonials, logos — check `items % columns ==
0` at every breakpoint the grid uses; add or remove items to close the gap, and if a breakpoint
changes column count, the total needs to work for both.

## Rule 31: Templates Own Visual Context — Adjacent Sections Never Share a Background

A template that stacks 3+ section components vertically (Hero, FeatureGrid, CTA, FAQ) must
pass each child an explicit background/variant prop: never rely on a component's own default
being visually compatible with its neighbors. Two adjacent full-width sections with the same
background merge into one visual block and the page loses its scroll rhythm; alternate white,
muted, and accent treatments so the pattern reads clearly top to bottom. The one exception is a
deliberate visual group (pricing + FAQ sharing a background) — and even then a border or
spacing change must mark the seam.

```
DON'T: <Hero /><FeatureGrid /><CTA />        // each renders its own default background
DO:    <Hero background="white" /><FeatureGrid background="muted" /><CTA background="accent" />
```

## Rule 32: Content Rewrites Are Scoped and Link-Safe

Before touching existing page content, classify every section the task might touch as **MUST
CHANGE** (directly required: "rewrite hero headline" makes the hero must-change) or **SHOULD
CHANGE** (would improve consistency but isn't asked for). Modify only MUST CHANGE sections
without separate approval; note SHOULD CHANGE sections in the result and leave them alone. In
the same pass, extract every internal link in the file before editing, extract them again
after, and diff the two lists. Any link present before and missing after is a blocking finding,
not a note for later: either restore the link, or name the removal explicitly in the plan or
report as a deliberate, justified change. An unexplained removed link does not ship silently
just because it was flagged — it runs through the same fix-loop mechanism a failed critique
finding does (see `loop.md`): a fix pass, or the slice fails and gets reported. A section can
carry inbound anchor links from other pages that its own content gives no hint of; the
inventory catches that, a read-through of the diff does not.

## Rule 33: A New Page Isn't Done Until the Completeness Checklist Passes

Creating a page or route means all of the following are true before the task is marked
complete, not treated as follow-up polish:

| Check | Verify |
| --- | --- |
| Metadata (title, description) | Read the metadata export |
| OG/Twitter cards | `curl` the page, check `og:title` etc. |
| JSON-LD structured data | `curl` the page, check for `application/ld+json` |
| Breadcrumbs | Rendered breadcrumb nav or JSON-LD `BreadcrumbList` |
| Sitemap inclusion | Page path present in `sitemap.ts`/`sitemap.xml` |
| Inbound nav link | At least one existing page links to this one |

Missing metadata, cards, structured data, or sitemap inclusion is blocking. A missing inbound
link is a warning, not a blocker, but a page with zero inbound links is invisible to crawlers
and visitors alike and should be flagged rather than shipped silently.

## Not this

- A hero headline that restates the category ("The best way to...") instead of a specific outcome.
- Two competing buttons above the fold, or a CTA that isn't the highest-contrast element on the page.
- Social proof that only lives in the footer.
- A flat 10-link nav bar on a multi-category site instead of a mega menu.
- A pricing tier listing "unlimited" anything, or a feature that isn't actually shippable yet.
- An objection answered only inside a collapsed FAQ accordion.
- A content rewrite that drops an internal link without restoring it or naming the removal as a deliberate, justified change.
- A template stacking sections with no explicit background prop, so two neighbors collide visually.
- Copy that explains its own restraint ("not a lock-in trick") or points at itself ("as shown above").
- A fabricated testimonial, customer name, or statistic standing in for a real one instead of a marked placeholder.
- More than two em dashes, or a "not X, but Y" pivot repeated more than once on the same page.
