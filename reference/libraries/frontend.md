# frontend

Illustrated against a component-based web stack — React with Tailwind/CSS custom
properties. The mechanics (box model, tokens, render correctness) generalize to any
framework with a CSS cascade and a component tree; swap the class syntax, keep the rule.

## Rule 1: Account for borders in the box model

Every variant of a component (button, chip, badge, input) renders at the same size
regardless of which variant is shown. Give the base component a transparent border at
the variant's border width, so fill and outline share one padding scale — outline then
only overrides `border-color`. Compensating an outline's border with shaved padding
desyncs the instant a size modifier (`sm`/`lg`) overrides the padding independently,
leaving fill and outline buttons at different heights at every size but the default.

```
DON'T: .btn-outline { border: 2px solid; padding: 6px 12px; }   /* fill has 0 border, same padding → 4px taller */
DO:    .btn { border: 2px solid transparent; padding: 8px 14px; }
       .btn-outline { border-color: var(--accent); }
```

## Rule 2: Sizing tokens, not magic pixels

Spacing, radius, font size, and breakpoints come from a shared scale, not one-off
hardcoded values scattered per component. Never set the root font size in `px`
(`html { font-size: 16px }`) — it overrides the user's browser font-size preference;
use `%` on the root or leave it at the browser default so every `rem` value keeps
scaling with it. Express the type scale in `rem` (root-relative, doesn't compound);
reserve `em` for sizing that should track the element's own font size (icons, internal
padding) and `px` for hairlines. Adjusting the root at breakpoints for responsive type
scales every `rem` value including spacing — fine only when that split from `px` chrome
is deliberate, not an accident of which unit a component happened to use.

```
DON'T: html { font-size: 16px; }         .card { padding: 14px; font-size: 13px; }
DO:    html { font-size: 100%; }         .card { padding: var(--space-3); font-size: var(--text-sm); }
```

## Rule 3: Responsive by default

Layouts work from the smallest target width up — no fixed widths that overflow on
mobile, no horizontal scroll on the body. Use fluid primitives (flex/grid, `minmax`,
`clamp()`) over fixed column counts, and verify the wrapped/stacked state, not just the
desktop screenshot. A layout that was only ever viewed at one breakpoint is unverified,
not responsive.

## Rule 4: Use the semantic element for the job

`<button>` for actions, `<a>` for navigation, landmark elements (`nav`/`main`/`header`/
`footer`) for page structure, `<ul>`/`<ol>` for lists, one `<h1>` per page in logical
order. A click handler on a `<div>` or `<span>` where a `<button>` was meant loses
keyboard operability, focus handling, and the accessibility tree entry for free — all of
which now have to be hand-rebuilt with `tabindex` and `aria-*`, badly.

## Rule 5: Keyboard and assistive-tech access is not optional

Every interactive element is keyboard-operable with a visible focus state — never strip
`:focus`/`:focus-visible` without shipping a replacement ring. Images carry meaningful
`alt` (or `alt=""` if decorative); icon-only buttons carry `aria-label`; form inputs have
an associated `<label>`; color is never the only signal; text meets WCAG AA contrast.
`aria-*` fills gaps native semantics can't reach — it is not a substitute for using the
right element (Rule 4 first).

## Rule 6: Reserve space before content arrives

Media (`<img>`, `<video>`, embeds) declares `width`/`height` or `aspect-ratio` so the
browser reserves its box before the asset loads; skeletons match the final element's
dimensions, not a placeholder guess. Content injected above what's already rendered
shifts everything below it — the classic case is a banner or ad slot that pops in and
shoves the button the user was about to tap. Load web fonts with `font-display: swap`
plus a metric-matched fallback to cap the reflow when the real face arrives.

```
DON'T: <img src="hero.jpg" />                       /* no space reserved, layout jumps on load */
DO:    <img src="hero.jpg" width="800" height="450" style="aspect-ratio: 16/9" />
```

## Rule 7: Extract on the third duplicate, not the first

One component owns one UI concern. The third copy-pasted instance of the same markup
becomes a shared component with a variant API (props, `cva`, data attributes) — not a
fourth near-identical class string. When a UI pattern is fixed on one page, check the
other pages rendering the same inline markup; a fix that lands on one instance and
leaves three siblings unfixed is a bug the diff doesn't show.

## Rule 8: The template owns the seam between stacked components

When a page or template composes multiple components stacked vertically (Hero,
FeatureGrid, CTA, FAQ), the template owns the visual context between them —
background, spacing, border — passed down explicitly as props (`className`/`variant`/
`background`), not left to each component's own defaults happening to be visually
compatible with whatever renders next to it. Two adjacent sections both defaulting to a
white background read as one section with a stray gap in the middle; two components both
defaulting to their own border produce a doubled seam. Component defaults stay neutral so
they compose safely anywhere — the template is what claims ownership of the seam.

```
DON'T: <Hero /><FeatureGrid /><CTA />                     /* each picks its own bg/border, seams are luck */
DO:    <Hero background="surface" />
       <FeatureGrid background="surface-alt" className="border-t" />
       <CTA background="surface" />
```

## Rule 9: List keys and effects must be honest

List items get a stable, unique `key` — never the array index once items can reorder,
insert, or delete; an index key silently reattaches the wrong item's state to the wrong
row. No state duplicates a value derivable during render. `useEffect` (or its framework
equivalent) synchronizes with an external system, not a place to compute a value that
belongs inline; its dependency array is complete, and anything it subscribes to is torn
down in the cleanup.

## Rule 10: Don't pay for renders and bytes you don't need

New object/array/function literals as props in a hot render path cause children to
re-render for no reason; memoize where it measurably helps, not everywhere by reflex.
Long lists need virtualization — a 500-row unvirtualized list is a scroll-jank bug
waiting for real data. Below-the-fold or rarely-used heavy components are code-split,
not bundled into the initial load. A child never re-fetches data its parent already
holds.

## Rule 11: Every async view has three states, not one

Loading, error, and empty are handled explicitly alongside success — a view coded only
for the happy path renders `undefined`, `NaN`, or a blank fragment the moment data is
still in flight or the request failed. Guard against races: ignore a stale response that
resolves after a newer request superseded it, and abort in-flight requests on unmount or
key change. Optimistic UI ships only with a rollback path for when the request fails.

## Rule 12: Forms confirm every input and prevent double-submit

Controlled inputs always pair with their change handler — a controlled input missing
`onChange` is silently read-only. Validation shows inline near the field, on blur or
submit, not as a red flash on every keystroke. The submit button disables (or shows a
spinner) for the duration of the request, and user input survives a failed submit rather
than being cleared. `type`, `inputmode`, and `autocomplete` are set so the right keyboard
and autofill show up on a real device.

## Rule 13: Images are sized, modern, and lazy — card media is never flatter than 3:2

Use responsive images (`srcset`/`sizes` or the framework's `<Image>`), serve modern
formats (AVIF/WebP), and lazy-load anything below the fold. Icons are inline SVG sized in
`em` with `currentColor`, not oversized raster files. Card and thumbnail media defaults
to a **3:2** aspect ratio and never goes flatter — 3:2, 4:3, 1:1, and portrait all read
fine; 16:9 or letterbox bands look starved and break a grid where other cards are taller.
Lock it with `aspect-ratio` on the container, not a fixed pixel height, so it reserves
space per Rule 6 and every card in the grid matches.

```
DON'T: .card-img { height: 160px; object-fit: cover; }         /* ratio depends on card width, drifts */
DO:    .card-img { aspect-ratio: 3/2; object-fit: cover; }
```

## Rule 14: No secrets in the bundle, no unsanitized HTML

Server secrets and private keys never reach a client bundle — a value under a
`NEXT_PUBLIC_`-style public env prefix is public, full stop; treat it as shipped source.
`dangerouslySetInnerHTML` (or `v-html`/raw `innerHTML`) on anything not fully
first-party and static is an XSS hole; if unavoidable, sanitize the string immediately
before it's rendered, not somewhere upstream that a future edit can skip.

## Rule 15: Color comes from semantic tokens

Colors — including dark mode — reference semantic tokens (`--surface`, `--text`,
`--border`), never a hex value repeated across components. A hardcoded `#1a1a1a` next to
`--surface: #1a1a1a` in the token file is the same color today and two different colors
the moment the palette changes, because only one of them updates. If both light and dark
are supported, both are designed on purpose — a blind CSS invert is not a dark theme.
Respect `prefers-color-scheme` and `prefers-reduced-motion` rather than forcing a mode.

## Not this

- Shaving an outline variant's padding to fake-match a filled variant's height.
- A hardcoded `px` root font size that ignores the user's browser preference.
- A clickable `<div>` standing in for a `<button>`.
- Card thumbnails cropped to 16:9 or any ratio flatter than 3:2.
- List items keyed by array index once the list can reorder.
- An async view with no error or empty state, only the success render.
