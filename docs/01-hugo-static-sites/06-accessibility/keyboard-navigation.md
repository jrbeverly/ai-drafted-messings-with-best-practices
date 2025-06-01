# Keyboard Navigation

Every interactive element must be fully operable using only a keyboard -- Tab, Enter, Space, Escape, and arrow keys.

## Why It Matters

- Users with motor disabilities, power users, and screen reader operators rely entirely on keyboard
- Native HTML elements (`<a>`, `<button>`, `<input>`) are keyboard-accessible by default; custom components must replicate this
- Keyboard patterns defined in Hugo templates and shared JS ensure consistent behavior across all pages

## Key Recommendations

**Use native elements first:** `<button>` for actions, `<a href>` for navigation. Custom elements must add `tabindex="0"`, `role`, and key handlers for Enter/Space.

**tabindex rules:** `0` = add to tab order, `-1` = programmatic focus only. Never use positive values (1, 2, 3) -- they create unpredictable order.

**Visible focus indicators are required:**

```css
:focus-visible {
  outline: 3px solid #1a73e8;
  outline-offset: 2px;
}
```

**Standard key expectations:** Tab/Shift+Tab for navigation, Enter for links/buttons, Space for buttons/checkboxes, Escape to close overlays, arrow keys within composite widgets.

**Roving tabindex for composite widgets** (tabs, menus, radio groups): only one item has `tabindex="0"`, arrow keys move focus, Tab exits the widget entirely.

**Modal dialogs must trap focus:** Tab cycles within the modal, Escape closes it, focus returns to the trigger on close.

**Prevent default scroll** when Space or arrow keys control a widget: `event.preventDefault()`.

**Hugo nav pattern:** use `<button>` for menu toggles with `aria-expanded`, move focus to first item on open, close on Escape and restore focus.

## Pitfalls to Avoid

- Positive `tabindex` values (1, 2, 3) creating unpredictable tab order
- Removing focus outlines with `outline: none` without a visible replacement
- Using `<div>` or `<span>` for interactive controls when `<button>` or `<a>` works
- Hover-only interactions without keyboard equivalents
- Keyboard traps where the user cannot Tab out (except intentional modal traps)
- `tabindex="0"` on non-interactive content like paragraphs
- Using `event.keyCode` instead of `event.key`
- Assuming Hugo theme components are keyboard-accessible by default
