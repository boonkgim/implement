# ux

This governs how a user-facing surface behaves and reads, on any platform — web, mobile, or
native. Each rule states the underlying invariant first; where an illustration names a specific
framework, component library, or CSS utility, read it as an example, never as the requirement
itself.

## Rule 1: Prefer built-in and library components over custom-built primitives

A hand-built button, modal, select, or dropdown reimplements accessibility semantics, focus
management, and years of edge-case handling that a maintained component library already solved.
The failure this prevents: a custom control missing keyboard support, screen-reader labeling, or
an edge case (long text, RTL, disabled state) the library handles for free.

DON'T: style a link element as a button by hand, losing its native navigation semantics.
DO: use the framework's "render as" pattern (e.g. `asChild`) to keep native behavior underneath
custom styling.

## Rule 2: Design for zero-instruction first use

If people need a walkthrough to find the primary action, the affordance is the bug, not the
missing help text. The failure this prevents: a first-run modal that blocks the actual screen
before anyone has seen it, or a tooltip carousel users click through without reading and forget
immediately after.

DO: guiding empty states, a one-time contextual hint on first use, placeholders that suggest
what to type. DON'T: a multi-step onboarding modal that must be dismissed before the app is
usable.

## Rule 3: Respect reserved system regions

The platform reserves parts of the screen for its own chrome — status bar, notch, home
indicator/gesture bar on mobile; address bar and toolbars that resize a mobile browser's visible
area. The failure this prevents: a button rendered under the home indicator that's unreachable,
or a "full height" page sized against the full viewport that gets its bottom controls cut off
when browser chrome is showing.

DO: inset edge content by the platform's safe-area value; size full-bleed layouts against the
dynamic/visible viewport, not the theoretical full one. DON'T: assume a fixed full-screen height
that includes chrome the user can currently see on top of your content.

## Rule 4: Every async operation shows a visible, content-matched loading state

Any operation with non-instant, uncertain latency needs feedback the moment it starts — a
skeleton shaped like the real content when the layout is known, a spinner when it isn't, and for
a per-item action inside a list, the spinner scoped to only the item acted on. The failure this
prevents: a user staring at a static screen unsure whether their tap registered, or tapping a
delete icon a second time because nothing visibly happened, or every row in a list showing a
spinner because one row's action fired.

## Rule 5: Never fail silently

Every request has three outcomes — success, error, and unexpected empty/null — and each needs a
visible response. The failure this prevents: an error swallowed by an empty catch block, or a
conditional that branches on success and error but has no branch for a null response, leaving
the screen blank with nothing telling the user anything went wrong.

## Rule 6: Never let the user get stuck

Initialization, auth checks, and splash/loading flows must always resolve to something the user
can act on, even when the underlying check fails unexpectedly. The failure this prevents: an
uncaught exception during startup leaves a loading spinner running forever with no retry and no
way forward.

DO: wrap initialization in error handling with an always-run cleanup step, so the flow leaves the
loading screen no matter what happens inside it.

## Rule 7: Disable in-flight controls to prevent duplicate submission

A control that triggers an async action must be disabled (and show progress) for the duration of
that action, and a submit control specifically must also stay disabled until its input is valid,
re-enabling only once the operation finishes either way. The failure this prevents: a double tap
on "submit" firing two requests and creating a duplicate record, or a form letting a user press
submit while a required field is empty.

## Rule 8: Fully support dynamic list items — focus and completeness

When a user adds a repeatable entry (another address, another line item), the new input must
receive focus and scroll into view immediately, and the surrounding form must not let a
half-filled entry pass through submission unnoticed. The failure this prevents: a user adding a
row and having to manually tap into it before typing, or submitting a form where one entry has a
blank required field that silently drops from the saved result.

## Rule 9: Keep the active input visible and reachable

An on-screen keyboard or IME covers roughly the bottom half of the screen; the field currently
being edited, and whatever submits it, must stay visible above that overlay, and moving to the
next field must not require hunting for it manually. The failure this prevents: a user typing
into the fourth field of a form who can't see what they're typing because the keyboard covers it,
or who has to dismiss the keyboard and tap each subsequent field by hand.

DO: scroll the focused field above the keyboard automatically; support sequential "next field"
advancement that ends in a submit action on the last field.

## Rule 10: Keep visually similar controls visually consistent

Controls that sit side by side and serve a similar purpose — a date field next to a text field —
must match on height, border, radius, padding, and background, checked against what already
exists rather than invented fresh per component. The failure this prevents: a row of inputs that
reads as sloppy because one control is two pixels taller than its neighbor.

## Rule 11: Align a CTA with its parent content block, not against it

A button or CTA takes its horizontal alignment from the content block it belongs to — centered
copy gets a centered button, left-aligned copy gets a left-aligned button — and no single block
mixes alignments across its own elements. This is a parent-child relationship, distinct from
Rule 10's sibling-to-sibling consistency: even a block with no neighboring control to match still
owes its button the alignment of its own text. The failure this prevents: a hero section with
centered headline and body copy but a button pinned to the left edge, reading as two components
stitched from different designs rather than one intentional block.

DON'T: left-align a button beneath centered headline and body text.
DO: carry the block's alignment through every element inside it, button included.

## Rule 12: Refresh data without discarding view state or flicker

Forcing a refetch by discarding and recreating a component (changing its identity) throws away
in-flight state and can break when several instances share a cache. Use the data layer's own
refetch mechanism instead, and keep previously loaded content visible while refetching so the
screen doesn't flash back to a loading state on every background refresh. The failure this
prevents: a list that blanks to a spinner every time it silently re-syncs with the server.

## Rule 13: Trim before validating; time validation to user intent

Platforms inject trailing whitespace during autocomplete, so validation must check a trimmed copy
of the input without mutating the value actually stored. Validation itself should stay silent
while someone is still making their first pass through a field, appearing on blur or submit —
whichever comes first — and once an error is shown, clear live as they fix it. The failure this
prevents: a false "invalid" error from trailing autocomplete whitespace, or an error that flashes
red on every keystroke of a field the user hasn't finished typing yet.

## Rule 14: Match error surface to severity, and always leave a way forward

A field-level error renders inline next to the field it concerns, with identical styling across
every step of the same flow. A background failure that doesn't block input (a network hiccup) gets
a transient, non-blocking notice. Only an error that truly blocks progress and needs a decision
uses a modal that requires acknowledgment. Whichever surface, the message says what went wrong
and every failure state offers a retry. The failure this prevents: a background network error
interrupting the whole flow with a blocking dialog, or an error explaining the problem with no
button to try again — a dead end.

## Rule 15: Confirm irreversible actions before executing

Any action that destroys data or is expensive to undo — delete, remove member, leave — needs an
explicit confirmation naming what will be lost, before it runs. Actions that are free or cheap to
undo (removing a row from a form not yet saved, closing an auto-saved draft) don't need one; a
confirmation there just trains users to click through confirmations without reading them. The
failure this prevents: an accidental delete with no undo, or confirmation fatigue that makes the
real warnings ignorable.

## Rule 16: Size and space interactive targets for the input method

Interactive elements need a minimum hit area (roughly 44px/dp on touch) and enough separation
from their neighbors (roughly 8px) that an imprecise tap can't land on the wrong one; on a narrow
viewport, a primary action should expand to fill the available width rather than sit as a small
fixed-width target in open space. The failure this prevents: an edit icon and a delete icon close
enough together that a fat-finger tap deletes instead of edits.

## Rule 17: Never drop text below the readable minimum, or clip it under user scaling

Body, label, and feedback text has a platform-appropriate floor below which it's a strain to
read, and a user who increases their system font size must not have that text truncated by a
fixed-height container. The failure this prevents: a status message rendered small enough that it
reads as fine print rather than feedback, or a label that gets cut off mid-word once someone
turns their device font size up.

DO: size containers that hold text with a minimum height, not a fixed one.

## Rule 18: Give async and stateful controls stable, truthful feedback

A button's dimensions shouldn't shift when its label is replaced by a spinner — reserve the width
up front. An error being retried should stay on screen rather than being swapped for a different
message mid-attempt. A non-navigating success (copy, send invite) should confirm itself in place
— "Copied" — briefly, rather than doing nothing visible or relabeling to a vague progressive tense
that adds no information over the spinner already showing. The failure this prevents: a button
that visibly resizes every time it starts loading, or a user who copies a link and sees no
confirmation it worked.

## Rule 19: Design empty states, don't leave blank ones

Every list, table, or data view has a state for zero items, and that state explains what belongs
there and offers a concrete next action. The failure this prevents: a blank pane a user can't
distinguish from a loading bug or a broken query.

## Rule 20: Animate state changes with easing; degrade under reduced motion

A visible change in color, size, or position reads as intentional when it's brief and eased —
fast-out on opening, fast-in on closing — and as a bug when it snaps instantly or slides at a
constant linear rate. When the platform reports a reduced-motion preference, replace
translate/scale transitions with instant or opacity-only changes rather than removing feedback
entirely; functional indicators like loading spinners are the exception and may keep animating.
The failure this prevents: a sheet that appears with no transition at all, or a user with motion
sensitivity getting a parallax effect they explicitly asked their device to suppress.

## Rule 21: Reserve optimistic UI for instant, reversible actions

Toggles, likes, and other one-tap state flips should update immediately and reconcile with the
server silently in the background, reverting with a visible error only on failure. Never apply
this pattern to a destructive action, a multi-step commitment, or a payment — there, the user
needs to see the operation actually happen, and an optimistic success that later fails is worse
than a brief wait. The failure this prevents: a "payment successful" state shown before the
charge has actually cleared.

## Rule 22: Show loading and error states in place, not as a full-screen swap

Replacing an entire screen — header, navigation, chrome and all — with a loading view for what is
really a content-area fetch disorients the user and risks losing layout handling (like Rule 3)
that only the swapped-out state had. Keep the surrounding layout mounted and swap only the
content region. Exception: a genuine first-load or auth splash where no layout exists yet to
preserve. The failure this prevents: navigation controls disappearing and reappearing every time
a screen's content refetches.

## Rule 23: Give every interactive element distinct hover, focus, and active affordance

On any input method that supports it, a clickable element needs a visibly different look for
hoverable, keyboard-focused, and currently-pressed, plus a pointer cue where the platform has
one. Removing the default focus indicator without supplying a replacement is the single most
common accessibility regression — it leaves keyboard-only users with no visible indication of
where they are on the page. The failure this prevents: a keyboard user tabbing through a form
with no way to tell which field is active.

These states dim, lift, or scale the control's declared variant — they never swap it for a
different one. An outline button stays an outline button on hover, a ghost button stays ghost;
only the affordance changes, not the identity. The failure this prevents: an outline button that
flips to a filled background on hover, momentarily reading as a different, higher-emphasis action
than the one the user is actually about to trigger.

## Rule 24: Meet minimum text and UI contrast in every theme

Text and meaningful UI elements (icons, borders, form controls) need to clear a WCAG-AA-equivalent
contrast ratio against their background, checked independently in every theme the product ships —
a pair that passes in light mode commonly fails in dark mode. The failure this prevents:
placeholder or disabled-state text that's legible on the designer's monitor but unreadable on an
average phone outdoors.

## Rule 25: Click-activated disclosures follow the full open/close contract

A dropdown, menu, or panel opened by a click is a toggle, not a one-way door: clicking its trigger
again closes it, any directional indicator reflects open/closed state, selecting an item inside
closes the panel, and clicking outside or pressing Escape both dismiss it. The failure this
prevents: a menu that opens on click but does nothing on a second click, or stays open after the
user has already navigated away from the page it was on.

## Rule 26: Keep structural layout internally consistent across repeated elements

When the same container or pattern repeats — cards in a grid, sections down a page — every
instance must resolve consistently regardless of how much content it holds: footers align to the
same edge across cards of different heights, a grid doesn't leave a half-empty final row,
decorative elements that intentionally extend past their container aren't clipped by it, and
adjacent full-bleed sections alternate treatment so their boundary reads clearly. The failure
this prevents: a "view details" button floating mid-card on a short card while its neighbor's
sits at the bottom, or a "most popular" badge silently clipped by its own card.

## Rule 27: Render meaningful glyphs as real icons, not text stand-ins

Arrows, chevrons, checkmarks, and similar affordances belong in an icon library (SVG or vector),
never as Unicode characters — text glyphs render inconsistently across fonts and platforms, can't
be weight- or size-matched to the rest of an icon set, and baseline-misalign next to adjacent
text. Favor an icon over a text label only where the meaning is genuinely universal (play, pause,
close); fall back to a label when the action is ambiguous or app-specific. The failure this
prevents: a "Back" control whose arrow renders as a mismatched, misaligned character on one
platform's default font.

The same holds for emoji standing in for an icon or a list bullet — an emoji is just another
inconsistently-rendered text glyph, subject to the same font and platform drift. Use the icon
library for the icon, and a plain character or the list's native marker for the bullet.

## Rule 28: Label and order content by audience recognition, not internal jargon

Navigation labels, menu items, and grouped listings should use terms and ordering the audience
already recognizes, not internal codenames, team jargon, or acronyms only insiders understand;
where order isn't fixed by convention (alphabetical, chronological), lead with the most
recognized items so they anchor orientation. The failure this prevents: a menu that leads with an
internal product codename above the well-known option users are actually looking for, making them
second-guess whether they're even in the right menu.

## Not this

- A hand-rolled button/modal/select where the component library already has one.
- A tutorial modal standing in for a first screen that should explain itself.
- An API call with no loading indicator, no error branch, or neither.
- A delete, remove, or leave action with no confirmation — or a confirmation on an
  auto-saved, reversible one.
- A disabled button dimmed with opacity instead of a lighter, still-opaque color.
- A focus outline removed with nothing put in its place.
- Text doing real work at a size below the platform's readable floor, or clipped by a
  fixed-height container under larger user font settings.
- A repeated card/section pattern where instances don't line up with each other.
