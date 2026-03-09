# Accessibility (a11y)

Accessibility ensures the application is usable by everyone, including users with disabilities. The primary tools are semantic HTML, keyboard navigation, and ARIA attributes when HTML semantics are insufficient.

shadcn/ui components already handle most accessibility concerns. This pattern covers what you need to do in custom code.

---

## Semantic HTML

Use the correct HTML element for its purpose. Never use `div` or `span` for interactive elements.

| Instead of | Use |
|-----------|-----|
| `<div onClick={...}>` | `<button onClick={...}>` |
| `<div>` for page sections | `<main>`, `<section>`, `<nav>`, `<aside>`, `<header>`, `<footer>` |
| `<div>` for lists | `<ul>` / `<ol>` with `<li>` |
| `<span>` for links | `<a>` or Next.js `<Link>` |
| `<div>` for headings | `<h1>` through `<h6>` (in order, no skipping levels) |

---

## Keyboard Navigation

All interactive elements must be operable with keyboard alone.

**Rules:**
- `<button>`, `<a>`, `<input>`, `<select>`, `<textarea>` are keyboard-accessible by default — use them
- Custom interactive components must have `tabIndex={0}` and handle `onKeyDown` (Enter and Space for activation)
- Focus order must follow visual order — avoid `tabIndex` values greater than 0
- Dialogs must trap focus while open (shadcn/ui `Dialog` handles this)
- Dropdown menus must support arrow key navigation (shadcn/ui `DropdownMenu` handles this)

---

## Labels and Descriptions

Every form input must have an associated label.

```tsx
{/* Correct — label is linked to input */}
<Label htmlFor="animal-name">Name</Label>
<Input id="animal-name" data-testid="animal-name-input" />

{/* Also correct — label wraps input */}
<Label>
  Name
  <Input data-testid="animal-name-input" />
</Label>
```

**Rules:**
- Never use placeholder text as the only label — placeholders disappear when typing
- Error messages must be linked to their input via `aria-describedby`
- Required fields should use `aria-required="true"` or the `required` attribute

```tsx
<Input
  id="animal-name"
  aria-describedby="animal-name-error"
  aria-required="true"
  data-testid="animal-name-input"
/>
{error && (
  <p id="animal-name-error" role="alert">
    {error.message}
  </p>
)}
```

---

## Images

```tsx
{/* Informative image — describe what it shows */}
<img src={animalPhoto} alt="Brown cow in pasture" data-testid="animal-photo" />

{/* Decorative image — empty alt to hide from screen readers */}
<img src={decorativeIcon} alt="" aria-hidden="true" />

{/* Icon buttons — label the button, not the icon */}
<button aria-label="Delete animal" data-testid="delete-animal-button">
  <TrashIcon aria-hidden="true" />
</button>
```

---

## ARIA Attributes

Use ARIA only when semantic HTML is insufficient. Most cases are covered by the right HTML element.

| Scenario | ARIA attribute |
|----------|---------------|
| Icon-only buttons | `aria-label="Action description"` |
| Loading states | `aria-busy="true"` on the container |
| Live updates (toasts, alerts) | `role="alert"` or `aria-live="polite"` |
| Expandable sections | `aria-expanded="true/false"` |
| Current page in navigation | `aria-current="page"` |
| Disabled but visible elements | `aria-disabled="true"` |

**Never add ARIA that duplicates native HTML semantics:**
```tsx
{/* Wrong — button already has the button role */}
<button role="button">Click</button>

{/* Wrong — nav already has navigation role */}
<nav role="navigation">...</nav>
```

---

## What NOT to do

- Never remove focus outlines (`outline: none`) without providing a visible alternative
- Never use `tabIndex` greater than 0 — it breaks natural tab order
- Never use color alone to convey information (add icons or text)
- Never auto-play audio or video without user consent
- Never create custom interactive elements when a native HTML element exists
