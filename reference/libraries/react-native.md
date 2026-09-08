# react-native

Illustrated against React Native + Expo (Expo Router, urql, Zustand, Reanimated,
`react-native-keyboard-controller`). Swap the named libraries for this repo's own equivalents;
the seams they guard against — safe areas, keyboard overlap, stale cache on unfocused screens —
are the same regardless of which library sits on top of them.

## Rule 1: Drive the UI from API data, never from a hardcoded tier/plan model

Mobile apps cannot force an immediate update — a build in the wild keeps running old code
until the user updates it. Any conditional keyed to a fixed set of plan names, a hardcoded
tier-ordering array, a hardcoded price, or a string comparison like `tier === 'pro'` freezes
that shape into the binary. The day the backend adds a plan, renames one, or reprices one, the
installed base that hasn't updated renders wrong or crashes — and there is no way to force them
off that code path.

```
DON'T: type Tier = 'free' | 'plus' | 'pro'; if (tier === 'pro') { ... }
DO:    tier: string  // from API; sort by `sortOrder`, compare by `priceCents`, render `price` as given
```

## Rule 2: Keep onboarding route maps exhaustive over the state type

A state-machine onboarding flow that resolves a route via `routeMap[state]` and feeds it to
Expo Router's `<Redirect>` throws `TypeError: Cannot read property 'pathname' of undefined`
the moment a state exists in the type but not in the map — a cryptic crash with no useful
stack. Every state added to the state type must be added to the route map in the same change.
Type the map as `Record<RedirectableState, Href>`, not `Record<string, Href>`, so a missing
state is a compile error instead of a runtime crash.

## Rule 3: Give SafeAreaView the edges that match its header

`SafeAreaView` with no `edges` prop next to a Stack header double-pads the top — the header
already reserves the safe area, and the default view reserves it again, leaving a visible gap.
With `headerShown: true`, use `edges={['bottom']}`. With `headerShown: false`, use
`edges={['top', 'bottom']}` (or omit the prop). Never toggle `headerShown` between render
states on the same screen (loading vs. loaded) — dynamic toggling breaks Android's safe-area
calculation and the header renders under the status bar. Pick one header strategy per screen
and use it in every state, including loading and error.

## Rule 4: Use `react-native-keyboard-controller`, not `keyboard-aware-scroll-view`

`react-native-keyboard-aware-scroll-view` has known scroll-position bugs and is not actively
maintained; `react-native-keyboard-controller` is native, animates smoothly, and behaves the
same on iOS and Android. Wrap the root layout in `KeyboardProvider`. On screens with inputs,
use its `KeyboardAwareScrollView` with `bottomOffset` (~100) and
`keyboardShouldPersistTaps="handled"`, styled with `style`/`contentContainerStyle` — not a
NativeWind `className`, which the library does not read reliably for layout-affecting props.

## Rule 5: Don't stack SafeAreaView's bottom edge under KeyboardAwareScrollView

`SafeAreaView` reserves bottom padding for the nav bar; `KeyboardAwareScrollView` adds its own
`bottomOffset` when the keyboard opens. Include `'bottom'` in both and the keyboard-open state
gets padded twice — a visible dead zone above the input. Without a header, use
`edges={['top']}` only; with a Stack header, use `edges={['left', 'right']}` only. If bottom
inset is genuinely needed elsewhere on the screen, apply it conditionally on keyboard state
(`Keyboard.addListener` + `useSafeAreaInsets`), not unconditionally through `SafeAreaView`.

## Rule 6: Use the keyboard-controller's KeyboardAvoidingView inside modals

React Native's built-in `KeyboardAvoidingView` overshoots and animates jerkily when a modal
with text inputs opens the keyboard. Import it from `react-native-keyboard-controller` instead,
use `behavior="padding"` (the library normalizes the platform difference internally — no
`Platform.OS` branch needed), and set `pointerEvents="box-none"` on it so touches pass through
to content behind the avoiding view.

## Rule 7: In fixed-input chat UIs, condition bottom inset on keyboard state

A chat screen with a input pinned to the bottom should use `KeyboardAvoidingView`, not
`KeyboardAwareScrollView` — the scroll variant fights a fixed-position input. Apply
`insets.bottom` to the input container only when the keyboard is closed. With
`edgeToEdgeEnabled: true` on Android, the reported keyboard height already includes the nav
bar; adding `insets.bottom` on top of an open keyboard produces a visible white bar above the
input that has no content in it.

## Rule 8: Give FlatList screens the header-aware keyboard offset

A scrollable list with a text input needs `KeyboardAvoidingView` with
`keyboardVerticalOffset={90}` (Stack header ~44px + status bar ~44px) — without it, the
keyboard covers the input instead of pushing it into view. Use the Stack header on these
screens rather than a custom one; a custom header combined with `SafeAreaView
edges={['top']}` and `headerShown: false` breaks the offset math the same way Rule 3 breaks
padding. Pair `edges={['left', 'right']}` with the Stack header — never `['top']`. Apply
`paddingBottom` conditionally (`isKeyboardOpen ? 0 : insets.bottom`), and set
`keyboardShouldPersistTaps="handled"` on the list so a tap on a row doesn't first swallow-close
the keyboard.

## Rule 9: Dismiss the keyboard before opening any overlay

On iOS, the first tap on a non-`TextInput` element while the keyboard is focused can be
swallowed by the system's focus-resolution pass — even with `keyboardShouldPersistTaps=
"handled"` on the scroll container. A user taps "pick a date" and nothing opens; they tap
again and it works, and the bug report reads as flaky. Call `Keyboard.dismiss()` at the top of
every handler that opens a date picker, bottom sheet, modal, or action sheet from a screen that
has a `TextInput`.

## Rule 10: Use `StyledTextInput`, not raw `TextInput`

Raw `TextInput` from `react-native` does not vertically center text or placeholder on iOS —
single-line inputs render text pinned to the top with visible dead space below it.
`StyledTextInput` (`@/components/text-input`) fixes this and gives two consistent sizes
(`default` 56px, `sm` 48px for compact card contexts). Use it for every user-facing input;
reach for raw `TextInput` only inside a component that is itself building the styled
abstraction.

## Rule 11: Wrap multiline inputs for vertical centering, on both platforms

Multiline `TextInput` (chat composer, comment box) needs a wrapper `View` with
`justifyContent: 'center'` and `minHeight: 44` (the minimum touch target) to center on iOS, and
`textAlignVertical: 'center'` for the equivalent fix on Android — the two platforms need both,
not one or the other. Use `StyleSheet`, not NativeWind, for these — pixel-level padding tuning
(iOS typically needs a touch more vertical padding, gated on `Platform.OS`) is unreliable
through a utility-class layer.

## Rule 12: Add `triggerOnPressOut` to animated buttons on screens with text inputs

`Animated.createAnimatedComponent(Pressable)` from `react-native-reanimated` has a known bug
where `onPress` silently fails to fire while the keyboard is open. If `AnimatedButton` sits on
a screen with any `TextInput`, add `triggerOnPressOut` (it distinguishes a tap from a scroll
with a 10px movement threshold). Skipping this reads as an intermittent "button doesn't work"
report that only reproduces with the keyboard up.

## Rule 13: Mount data-fetching hooks after interactions settle, from a hookless wrapper

A screen that calls `useQuery`, `useMutation`, or a Zustand selector directly in its top-level
component competes with the navigation transition animation for the JS thread, and the
animation stutters or drops frames. Split the screen into a wrapper with no urql/Zustand
hooks — only `useState`, `useEffect`, `useRouter`, `useAuthStore`, `useSafeAreaInsets` — and a
content component that owns the data hooks and mounts only after
`InteractionManager.runAfterInteractions`. Show a skeleton in the wrapper until then. Do not
substitute `setTimeout(..., 0)` or `requestAnimationFrame` — both fire before the transition
actually finishes.

## Rule 14: Pause every urql query on every screen with `!isFocused`

An unfocused screen that keeps its query active still receives cache updates while another
screen is rendering, and React throws or warns about updating a component that isn't the one
currently animating in. `pause: !isFocused` (from `useIsFocused`) is required on every query on
every screen that has one — the destination screen's Rule 13 wrapper is not sufficient on its
own; the *source* screen must pause too, or it keeps eating cache writes triggered by the
destination. Compose it into existing conditions: `pause: !familyId || !isFocused`.

## Rule 15: Never pair `useFocusEffect` auto-refresh with `pause: !isFocused`

`useFocusEffect` refetching on focus return defeats the pause from Rule 14 — the refetch fires
a urql cache write that propagates to every other mounted screen holding the same query,
exactly the update-during-render failure `pause: !isFocused` exists to prevent. Use an explicit
refresh control (pull-to-refresh or a button) with its own `isRefreshing` state instead — urql's
`fetching` flag does not reliably flip on a cache hit, so a manual flag is the only way to show
refresh feedback correctly.

## Rule 16: Thread `isFocused` through every custom hook that queries

A custom hook wrapping `useQuery` needs its own `isFocused: boolean = true` parameter, ORed
into its pause condition (`|| !isFocused`), with every call site passing the screen's own
`useIsFocused()` result in. Skip this and a hook used on screen A silently receives cache
updates triggered by screen B's mutation, because the hook has no way to know which screen is
currently focused — the exact "Cannot update Screen A while rendering Screen B" failure Rule 14
guards against, just one layer further from the query call site.

## Rule 17: Defer cross-screen store writes off the navigation transition

A synchronous Zustand/store write inside a `useEffect` that other mounted screens read from can
race the navigation transition — the write and the transition's own state updates land in the
same tick and the result depends on ordering. Wrap the write in `setTimeout(() => ..., 0)` when
it's triggered by navigation (not by user interaction on a settled screen) so it lands after
the transition's own updates rather than during them.

## Rule 18: Keep native-module peer deps single-copy in a monorepo

A pnpm monorepo that resolves two copies of `react-native-reanimated`, `react-native-worklets`,
or another native module makes animations silently stop firing — no error, just dead
animations, because the native side is bound to one copy and JS calls into the other. Run
`npx expo-doctor` to catch duplicates, fix with `npx expo install --check`, then clear the
Metro cache and restart. Prevent recurrence by pinning shared packages' peer deps to ranges
(`">=3"`), not exact versions, so they resolve to whatever the app already has instead of
forcing a second copy.

## Rule 19: Point Metro at the app's own `node_modules` for context-dependent packages

In a pnpm monorepo, `metro.config.js` needs `resolver.extraNodeModules` mapping `react`,
`react-native`, the native-module packages from Rule 18, any singleton library (e.g. a toast
manager), and any context-dependent package (state/query libraries) to the app's own
`node_modules` — otherwise pnpm's strict linking hands different call sites different module
instances, and singleton state (a query cache, a toast queue) silently forks in two. Pair it
with `auto-install-peers=false` in `.npmrc`, or pnpm re-introduces the duplicate peer install
that `extraNodeModules` was added to route around.

## Not this

- A conditional that switches on a plan name, tier string, or hardcoded price instead of API
  data.
- `SafeAreaView` with no `edges` prop, or `headerShown` toggled between a screen's own render
  states.
- `KeyboardAvoidingView` or a scroll-view keyboard helper imported from `react-native` /
  `react-native-keyboard-aware-scroll-view` instead of `react-native-keyboard-controller`.
- A screen-level `useQuery`/`useMutation`/store selector with no `InteractionManager` wrapper
  and no `pause: !isFocused`.
- `useFocusEffect`-triggered refetch on a query that also sets `pause: !isFocused`.
- A raw `TextInput` for user-facing input instead of `StyledTextInput`.
