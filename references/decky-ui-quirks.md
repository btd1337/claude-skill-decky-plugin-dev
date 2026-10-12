# @decky/ui component quirks

Things in `@decky/ui` that don't behave the way you'd naively expect from their props/CSS, and
have cost real debugging time.

## Focus ring invisible inside a custom modal

`DialogButton`'s focus ring does not reliably render when used inside a custom/non-standard
modal (a plain `<div>` overlay you built yourself, rather than Steam's own dialog system). If
gamepad focus looks "invisible" on a button inside a custom modal - the button still technically
works via click/activate, it's just not visibly focused during D-pad/analog navigation - swap
that button to `Focusable` (with `onActivate`) instead. This has concretely fixed the exact
symptom before.

## Focus ring shape follows the styled element, not a wrapper

Whether the native gamepad focus ring follows your intended shape (e.g. a pill/rounded-corner
badge) depends on *which* element is actually `Focusable`/interactive. If you wrap a
separately-styled inner `<div>` inside a `Focusable`/button-like outer element, the focus ring
can end up rectangular regardless of the inner element's `border-radius`. Style the actual
focusable/interactive element directly (background, padding, border-radius) rather than nesting
a styled div inside it.

## `Dropdown.selectedOption` must exactly match an option's `data`

`Dropdown`'s `selectedOption` prop is matched by strict equality against each option's `data`
value. If you derive a "default" value independently from your options list (e.g. a hardcoded
default color hex that isn't *exactly* one of the hex strings in your preset list), the dropdown
renders with nothing visibly selected - even though your component state technically holds a
valid, non-empty value.

Fix: make the default/fallback value *literally* one of the listed option values - e.g. choose
your default color to equal one of the named presets, rather than inventing a separate "close
enough" value that happens to render similarly but isn't string-identical.

## Flex `gap` does not affect a `Field`-style component's own margin

`TextField`, `Dropdown`, and other `@decky/ui` `Field`-style form components carry their own
built-in vertical margin, independent of any `gap` set on a parent flex container. Setting
`gap` controls spacing between true flex siblings generally, but has **no effect** on a `Field`
component's intrinsic top/bottom margin - so reducing a parent's `gap` to visually tighten the
space between e.g. a `TextField` and a `Dropdown` stacked under it can do *nothing visible at
all*, which looks exactly like "my CSS change didn't apply."

To actually reduce that gap, apply a *negative* `marginTop` to a wrapper `<div>` around the
second component, pulling it toward the first. There's no documented exact default margin
value - tune the negative margin empirically against what you see rendered.

## Reliably adding space between two rows: border+padding, not margin alone

Conversely, when you need to reliably **add** visible space between two rows that keep looking
"glued together" despite `marginBottom`/`marginTop` on your own wrapper `<div>`, don't rely on
margin alone - margins on elements near a `PanelSectionRow` boundary can collapse or get
swallowed with no visible effect, the same silent-failure pattern as above but in the opposite
direction.

Use `borderTop` + `paddingTop` together - padding physically cannot collapse away the way margin
can, so it reliably reserves space. If you don't want a visible dividing line, the same
`marginTop` + `paddingTop` combo *without* `borderTop` still reliably reserves the space, because
the non-zero `paddingTop` forces box height regardless.

When verifying a spacing fix is "really" visible, prefer meaningfully-sized values (10px+) for
your first test. Very small deltas (4-6px) placed next to a component's own already-substantial
intrinsic padding can be functionally invisible even when technically applied correctly - which
makes it hard to distinguish "the fix didn't work" from "the fix worked but is too subtle to
see." Confirm visibility with a larger value first, then dial it back down if needed.

## `ButtonItem` does not accept a `style` prop

`ButtonItem` will produce a TypeScript warning (`Property 'style' does not exist on type ...
ButtonItemProps`) if you pass `style` directly. Wrap it in a plain styled `<div>` if you need to
control its outer spacing/sizing.

## A `transform` on an ancestor changes what `position: fixed` is relative to

Per the CSS spec, a `transform` on an ancestor element makes that ancestor the containing block
for any `position: fixed` descendant. If a full-screen modal overlay is rendered as a *child* of
some wrapper that has (or conditionally gets, e.g. for centering) a `transform` applied, the
modal's `position: fixed` gets scoped to that transformed ancestor's box instead of the actual
viewport - visually shrinking the "full screen" overlay down to the small wrapper's size instead
of covering the screen.

Fix: render the modal as a *sibling* outside the transformed wrapper, not as its descendant.

## Module-scanning fragility across SteamUI updates

Valve ships frequent SteamUI beta/stable updates that change internal webpack bundle shapes.
Plugins using `@decky/ui`'s module-scanning utilities (`findModuleChild`, `findModuleExport`,
`findModuleByExport`) to locate internal Steam React components are fragile to this: a changed
minifier/build can make some module export properties become arrays containing JS `Symbol`s,
and naive `.toString()`/`.join()` calls during the scan throw
`TypeError: Cannot convert a Symbol value to a string`.

- This is a known, recurring bug class - `@decky/ui` itself has shipped patch releases fixing
  exactly this kind of breakage.
- If such a scan runs at **module scope** (outside the plugin's main `definePlugin` entry), an
  uncaught exception there aborts the *entire* plugin import - nothing in the plugin loads at
  all, not just the feature that used the scan.
- Mitigation: keep `@decky/ui` reasonably current, and wrap these scans in a try/catch helper
  that returns `undefined` on failure instead of throwing, then guard any code that dereferences
  the result so it skips/disables just that one feature instead of crashing the whole plugin.
- `findModuleChild` is deprecated in favor of `findModuleExport`/`findModuleByExport`.
