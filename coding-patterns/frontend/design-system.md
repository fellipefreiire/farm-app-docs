# Design System Pattern

The design system defines how visual consistency is achieved across the application. It covers color tokens, typography scale, dark mode, and component primitives. All styling uses Tailwind CSS v4 with CSS-first configuration — no `tailwind.config.ts`.

---

## File locations

```
frontend/
  src/
    app/
      globals.css                          <- all tokens, @theme inline, :root, .dark
    shared/
      components/
        ui/                                <- shadcn/ui primitives (Button, Dialog, Input, etc.)
        fields/                            <- form field components (InputField, SelectField, etc.)
```

---

## Tailwind v4 CSS-first configuration

Tailwind v4 replaces `tailwind.config.ts` with CSS-native `@theme` directives. All design tokens live in `globals.css`.

```css
/* globals.css */
@import "tailwindcss";

/* ── Light theme (default) ── */
:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);

  --card: oklch(1 0 0);
  --card-foreground: oklch(0.145 0 0);

  --popover: oklch(1 0 0);
  --popover-foreground: oklch(0.145 0 0);

  --primary: oklch(0.205 0 0);
  --primary-foreground: oklch(0.985 0 0);

  --secondary: oklch(0.97 0 0);
  --secondary-foreground: oklch(0.205 0 0);

  --muted: oklch(0.97 0 0);
  --muted-foreground: oklch(0.556 0 0);

  --accent: oklch(0.97 0 0);
  --accent-foreground: oklch(0.205 0 0);

  --destructive: oklch(0.577 0.245 27.325);
  --destructive-foreground: oklch(0.985 0 0);

  --border: oklch(0.922 0 0);
  --input: oklch(0.922 0 0);
  --ring: oklch(0.708 0 0);
  --radius: 0.625rem;

  --chart-1: oklch(0.646 0.222 41.116);
  --chart-2: oklch(0.6 0.118 184.714);
  --chart-3: oklch(0.398 0.07 227.392);
  --chart-4: oklch(0.828 0.189 84.429);
  --chart-5: oklch(0.769 0.188 70.08);

  --sidebar: oklch(0.985 0 0);
  --sidebar-foreground: oklch(0.145 0 0);
  --sidebar-primary: oklch(0.205 0 0);
  --sidebar-primary-foreground: oklch(0.985 0 0);
  --sidebar-accent: oklch(0.97 0 0);
  --sidebar-accent-foreground: oklch(0.205 0 0);
  --sidebar-border: oklch(0.922 0 0);
  --sidebar-ring: oklch(0.708 0 0);
}

/* ── Dark theme ── */
.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);

  --card: oklch(0.145 0 0);
  --card-foreground: oklch(0.985 0 0);

  --popover: oklch(0.145 0 0);
  --popover-foreground: oklch(0.985 0 0);

  --primary: oklch(0.985 0 0);
  --primary-foreground: oklch(0.205 0 0);

  --secondary: oklch(0.269 0 0);
  --secondary-foreground: oklch(0.985 0 0);

  --muted: oklch(0.269 0 0);
  --muted-foreground: oklch(0.708 0 0);

  --accent: oklch(0.269 0 0);
  --accent-foreground: oklch(0.985 0 0);

  --destructive: oklch(0.577 0.245 27.325);
  --destructive-foreground: oklch(0.985 0 0);

  --border: oklch(0.269 0 0);
  --input: oklch(0.269 0 0);
  --ring: oklch(0.439 0 0);

  --chart-1: oklch(0.488 0.243 264.376);
  --chart-2: oklch(0.696 0.17 162.48);
  --chart-3: oklch(0.769 0.188 70.08);
  --chart-4: oklch(0.627 0.265 303.9);
  --chart-5: oklch(0.645 0.246 16.439);

  --sidebar: oklch(0.205 0 0);
  --sidebar-foreground: oklch(0.985 0 0);
  --sidebar-primary: oklch(0.488 0.243 264.376);
  --sidebar-primary-foreground: oklch(0.985 0 0);
  --sidebar-accent: oklch(0.269 0 0);
  --sidebar-accent-foreground: oklch(0.985 0 0);
  --sidebar-border: oklch(0.269 0 0);
  --sidebar-ring: oklch(0.439 0 0);
}

/* ── Theme mapping (Tailwind v4) ── */
@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);

  --color-card: var(--card);
  --color-card-foreground: var(--card-foreground);

  --color-popover: var(--popover);
  --color-popover-foreground: var(--popover-foreground);

  --color-primary: var(--primary);
  --color-primary-foreground: var(--primary-foreground);

  --color-secondary: var(--secondary);
  --color-secondary-foreground: var(--secondary-foreground);

  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);

  --color-accent: var(--accent);
  --color-accent-foreground: var(--accent-foreground);

  --color-destructive: var(--destructive);
  --color-destructive-foreground: var(--destructive-foreground);

  --color-border: var(--border);
  --color-input: var(--input);
  --color-ring: var(--ring);

  --color-chart-1: var(--chart-1);
  --color-chart-2: var(--chart-2);
  --color-chart-3: var(--chart-3);
  --color-chart-4: var(--chart-4);
  --color-chart-5: var(--chart-5);

  --color-sidebar: var(--sidebar);
  --color-sidebar-foreground: var(--sidebar-foreground);
  --color-sidebar-primary: var(--sidebar-primary);
  --color-sidebar-primary-foreground: var(--sidebar-primary-foreground);
  --color-sidebar-accent: var(--sidebar-accent);
  --color-sidebar-accent-foreground: var(--sidebar-accent-foreground);
  --color-sidebar-border: var(--sidebar-border);
  --color-sidebar-ring: var(--sidebar-ring);

  --radius-sm: calc(var(--radius) - 0.125rem);
  --radius-md: calc(var(--radius));
  --radius-lg: calc(var(--radius) + 0.125rem);
  --radius-xl: calc(var(--radius) + 0.25rem);
}
```

### How it works

1. **`:root`** defines CSS variables with light-mode OKLCH values
2. **`.dark`** overrides those same variables with dark-mode values
3. **`@theme inline`** maps each CSS variable to a Tailwind `--color-*` namespace, generating utility classes (`bg-primary`, `text-muted-foreground`, etc.)

The `inline` keyword is required because these reference `:root` variables — without it, Tailwind would embed the variable name instead of resolving the value, causing issues in nested contexts.

---

## Color tokens

### Semantic tokens (mandatory)

Every color used in the application must go through a semantic token. Never use raw Tailwind colors like `red-500`, `blue-600`, or `gray-200` directly in components.

| Token | Purpose | Usage example |
|-------|---------|---------------|
| `background` / `foreground` | Page background and default text | `bg-background text-foreground` |
| `card` / `card-foreground` | Card surfaces | `bg-card text-card-foreground` |
| `popover` / `popover-foreground` | Floating surfaces (dropdowns, tooltips) | `bg-popover text-popover-foreground` |
| `primary` / `primary-foreground` | Primary actions (buttons, links) | `bg-primary text-primary-foreground` |
| `secondary` / `secondary-foreground` | Secondary actions | `bg-secondary text-secondary-foreground` |
| `muted` / `muted-foreground` | Subdued backgrounds and text | `bg-muted text-muted-foreground` |
| `accent` / `accent-foreground` | Highlights, hover states | `bg-accent text-accent-foreground` |
| `destructive` / `destructive-foreground` | Danger actions (delete, errors) | `bg-destructive text-destructive-foreground` |
| `border` | Borders and dividers | `border-border` |
| `input` | Form input borders | `border-input` |
| `ring` | Focus rings | `ring-ring` |

### Adding project-specific tokens

When a project needs a color that doesn't fit the existing tokens (e.g., `warning`, `success`, `info`), add it following the same pattern:

```css
/* In :root */
--warning: oklch(0.84 0.16 84);
--warning-foreground: oklch(0.28 0.07 46);

/* In .dark */
--warning: oklch(0.41 0.11 46);
--warning-foreground: oklch(0.99 0.02 95);

/* In @theme inline */
--color-warning: var(--warning);
--color-warning-foreground: var(--warning-foreground);
```

This makes `bg-warning`, `text-warning-foreground`, etc. available as utility classes.

### Why OKLCH

Tailwind v4 and shadcn/ui use OKLCH as the default color format. OKLCH provides perceptually uniform color manipulation — lightness changes look consistent across hues. Use OKLCH for all new tokens. Tools like [oklch.com](https://oklch.com) help pick values.

---

## Typography scale

All typography uses Tailwind utility classes. No custom typography components or CSS — just a defined scale for consistency.

### Heading scale

| Level | Tailwind classes | When to use |
|-------|-----------------|-------------|
| Page title | `text-3xl font-bold tracking-tight` | One per page. Main page heading (`<h1>`). |
| Section title | `text-2xl font-semibold tracking-tight` | Major section within a page (`<h2>`). |
| Subsection title | `text-xl font-semibold` | Subsection or card group title (`<h3>`). |
| Card title | `text-lg font-medium` | Card headers, dialog titles (`<h4>`). |
| Label title | `text-base font-medium` | Form section labels, small group titles. |

### Body text scale

| Level | Tailwind classes | When to use |
|-------|-----------------|-------------|
| Body default | `text-sm` | Default body text, table cells, form inputs. |
| Body small | `text-xs text-muted-foreground` | Captions, help text, metadata, timestamps. |
| Body large | `text-base` | Emphasized paragraphs, hero descriptions. |

### Rules

- **One `<h1>` per page** — always uses the "Page title" scale
- **Heading hierarchy must not skip levels** — no `<h1>` followed by `<h3>`
- **Body text defaults to `text-sm`** — this is the base size for all content
- **`text-muted-foreground` for secondary text** — never use `text-gray-*` or `opacity-*`
- **`font-bold` only for page titles** — sections use `font-semibold`, cards use `font-medium`
- **Never use arbitrary font sizes** (`text-[17px]`) — stick to the scale

### Example

```tsx
<main>
  <h1 className="text-3xl font-bold tracking-tight">Animals</h1>

  <section>
    <h2 className="text-2xl font-semibold tracking-tight">Active</h2>

    <div className="rounded-lg border bg-card p-4">
      <h3 className="text-lg font-medium">Holstein #4521</h3>
      <p className="text-sm">Weight: 450kg</p>
      <p className="text-xs text-muted-foreground">Last updated: 2 hours ago</p>
    </div>
  </section>
</main>
```

---

## Dark mode

Dark mode is always supported via `next-themes`, even if the project doesn't expose a theme toggle to the user. This ensures the design system is ready for dark mode from day one.

### Setup

```tsx
// src/app/layout.tsx
import { ThemeProvider } from 'next-themes'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <body>
        <ThemeProvider
          attribute="class"
          defaultTheme="light"
          enableSystem
          disableTransitionOnChange
        >
          {children}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

### How it works

- `next-themes` adds/removes the `dark` class on `<html>`
- `.dark` selector in `globals.css` overrides the `:root` variables
- All components automatically adapt — no per-component dark mode logic
- `defaultTheme="light"` makes light the default. Change to `"system"` to follow OS preference.

### Rules

- Never use `dark:` variant in component classes — all dark mode is handled via CSS variable overrides
- Every new color token must have both `:root` and `.dark` values
- Test dark mode when adding new tokens — colors that look good in light may not work in dark

---

## Layout components

Layout components (e.g., `PageHeader`, `PageContent`, `Sidebar`, `Container`) are project-specific — they depend on the application's navigation structure, which varies per project.

When a project defines layout components, document them in `docs/coding-patterns/frontend/layout.md` covering:

- Component names and file locations
- Where each component is used (which routes/pages)
- Props and variants
- Responsive behavior
- How they compose together (nesting rules)

Until that document exists, pages compose layout directly using semantic HTML and Tailwind utilities:

```tsx
<main className="mx-auto max-w-7xl px-4 py-8">
  <header className="mb-8">
    <h1 className="text-3xl font-bold tracking-tight">Page Title</h1>
  </header>
  <section>
    {/* content */}
  </section>
</main>
```

---

## Styleguide

By default, the boilerplate does **not** include a formal styleguide tool. The design system is documented in this file and enforced through coding patterns.

### Default approach (no extra tooling)

- Color tokens, typography scale, and component patterns are defined here
- Claude reads this file before creating any component
- Consistency is maintained through coding pattern compliance checks

### Optional: Storybook

For projects that need a visual component catalog (large teams, designer handoff, complex UI):

1. Install Storybook: `pnpm dlx storybook@latest init`
2. Write stories for `shared/components/ui/` primitives
3. Domain components do NOT need stories — they change too frequently
4. Add `pnpm storybook` to the Commands section in `CLAUDE.md`

### Optional: `/styleguide` page

For projects that want a lightweight visual reference without extra dependencies:

1. Create `src/app/(private)/styleguide/page.tsx`
2. Render all color tokens, typography scale, and shared UI components
3. Gate behind auth or feature flag so it doesn't ship to production

Choose the approach that fits the project's needs and document the decision in `docs/architecture.md` under the Decision Log.

---

## Shared UI primitives (shadcn/ui)

shadcn/ui components live in `src/shared/components/ui/`. They are the building blocks for all domain components.

### Rules

- Install components via CLI: `pnpm dlx shadcn@latest add <component>`
- Never modify shadcn/ui source files directly — if customization is needed, wrap in a new component
- Exception: updating generated code to match project tokens (e.g., replacing a hardcoded color with a token) is allowed
- All primitives use the semantic color tokens — they automatically support dark mode
- Domain components never redefine primitives — they import from `@/shared/components/ui/`

---

## Rules summary

- **Tailwind only** — no CSS Modules, no styled-components, no inline styles (except dynamic values)
- **Tailwind v4 CSS-first** — all tokens in `globals.css` via `@theme inline`, no `tailwind.config.ts`
- **OKLCH color format** — for all new tokens
- **Semantic tokens always** — never use raw colors (`red-500`, `blue-600`). Use `primary`, `destructive`, `warning`, etc.
- **Dark mode always ready** — every token has `:root` and `.dark` values, `next-themes` is always configured
- **Typography scale is fixed** — follow the heading/body scale defined above. No arbitrary sizes.
- **One `<h1>` per page** — heading hierarchy must be sequential
- **`text-sm` is the default body size** — not `text-base`
- **Layout is project-specific** — document in `layout.md` when defined
- **No formal styleguide by default** — coding patterns are the source of truth. Add Storybook or `/styleguide` page if the project needs it.
- **shadcn/ui primitives are immutable** — wrap, don't modify

---

## Anti-patterns

```tsx
// WRONG: using raw Tailwind colors
<div className="bg-red-500 text-white">Error</div>
// CORRECT: use semantic tokens
<div className="bg-destructive text-destructive-foreground">Error</div>

// WRONG: using dark: variant in components
<div className="bg-white dark:bg-gray-900">Content</div>
// CORRECT: use token that auto-switches
<div className="bg-background">Content</div>

// WRONG: arbitrary font size
<h2 className="text-[22px] font-bold">Title</h2>
// CORRECT: follow the typography scale
<h2 className="text-2xl font-semibold tracking-tight">Title</h2>

// WRONG: inconsistent heading weight
<h1 className="text-3xl font-semibold">Page</h1>  // should be font-bold
<h2 className="text-2xl font-bold">Section</h2>    // should be font-semibold

// WRONG: using opacity for muted text
<p className="text-sm opacity-50">Help text</p>
// CORRECT: use muted-foreground token
<p className="text-xs text-muted-foreground">Help text</p>

// WRONG: hardcoding color values in CSS
.custom-badge { background-color: #ef4444; }
// CORRECT: reference the token
.custom-badge { background-color: var(--destructive); }

// WRONG: skipping heading levels
<h1>Page</h1>
<h3>Subsection</h3>  // skipped h2

// WRONG: multiple h1 on the same page
<h1>Title A</h1>
<h1>Title B</h1>  // only one h1 per page

// WRONG: modifying shadcn/ui source
// src/shared/components/ui/button.tsx — editing directly
// CORRECT: wrap if customization is needed
// src/shared/components/custom-button.tsx — wraps Button with extra logic

// WRONG: defining colors in tailwind.config.ts
module.exports = { theme: { colors: { primary: '#000' } } }
// CORRECT: define in globals.css via :root + @theme inline
```
