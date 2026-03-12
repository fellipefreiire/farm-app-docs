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
| `primary` / `primary-foreground` | Primary actions, links, active sidebar sub-items | `bg-primary text-primary` |
| `secondary` / `secondary-foreground` | Secondary actions | `bg-secondary text-secondary-foreground` |
| `muted` / `muted-foreground` | Subdued backgrounds (table hover, placeholder containers), secondary text | `bg-muted text-muted-foreground` |
| `accent` / `accent-foreground` | Highlights | `bg-accent text-accent-foreground` |
| `destructive` / `destructive-foreground` | Danger actions (delete, errors) | `bg-destructive text-destructive` |
| `border` | Borders and dividers | `border-border` |
| `input` | Form input borders | `border-input` |
| `ring` | Focus rings | `ring-ring` |
| `overlay` | Modal/dialog backdrop | `bg-overlay/50` |
| `placeholder-icon` | Placeholder image icons in tables and headers | `text-placeholder-icon` |

### Current color values (light theme)

| Token | Value | Hex equivalent | Notes |
|-------|-------|----------------|-------|
| `primary` | `oklch(0.50 0.15 155)` | Vibrant emerald green | Agricultural identity |
| `muted` | `oklch(0.975 0.005 248.127)` | `#F4F7FA` | Light blue-gray, used for table hover (100% opacity) |
| `overlay` | `oklch(0.466 0.027 271.633)` | `rgb(84, 89, 105)` | Dialog backdrop |
| `placeholder-icon` | `oklch(0.844 0.026 255.600)` | `#C1CDDD` | Placeholder image icons |
| `sidebar-accent` | `oklch(0.975 0.005 248.127)` | `#F4F7FA` | Same as muted — sidebar hover matches table hover |

### Adding project-specific tokens

When a project needs a color that doesn't fit the existing tokens, add it following the same pattern:

```css
/* In :root */
--my-token: oklch(0.84 0.16 84);

/* In @theme inline */
--color-my-token: var(--my-token);
/* OR define directly: */
--color-my-token: oklch(0.84 0.16 84);
```

This makes `bg-my-token`, `text-my-token`, etc. available as utility classes.

### Why OKLCH

Tailwind v4 and shadcn/ui use OKLCH as the default color format. OKLCH provides perceptually uniform color manipulation — lightness changes look consistent across hues. Use OKLCH for all new tokens. Tools like [oklch.com](https://oklch.com) help pick values.

---

## Global base styles

### Container

Content max-width is **1280px** visible content + padding = **1360px** container:

```css
@utility container {
  max-width: 1360px;
  margin-inline: auto;
}
```

Pages use `className="container p-10"` which gives 40px padding each side (80px total), resulting in 1280px visible content.

### Cursor pointer

All clickable elements have cursor pointer by default:

```css
@layer base {
  button,
  [role='button'],
  a,
  [type='submit'],
  [type='reset'],
  [type='button'] {
    cursor: pointer;
  }
}
```

---

## Typography scale

All typography uses Tailwind utility classes. No custom typography components or CSS — just a defined scale for consistency.

### Heading scale

| Level | Tailwind classes | When to use |
|-------|-----------------|-------------|
| Page title | `text-2xl font-bold` | One per page. Main page heading (`<h1>`). |
| Detail page title | `text-[28px] font-bold leading-none` | Detail page entity name. |
| Section title | `text-xl font-bold` | Major section within a page (`<h2>`). |
| Submodule label | `text-[13px] font-medium uppercase text-muted-foreground` | Submodule type indicator (e.g., "VARIEDADE"). |

### Body text scale

| Level | Tailwind classes | When to use |
|-------|-----------------|-------------|
| Body default | `text-sm` | Default body text, table cells, form inputs. |
| Body small | `text-xs text-muted-foreground` | Captions, help text, metadata, timestamps. |
| Body large | `text-base` | Emphasized paragraphs, hero descriptions. |
| Field label | `text-[14px] text-muted-foreground` | Inline field labels in submodule detail pages. |
| Field value | `text-[14px]` | Field values (use `text-primary` for links). |

### Rules

- **One `<h1>` per page** — always uses the page title or detail page title scale
- **Heading hierarchy must not skip levels** — no `<h1>` followed by `<h3>`
- **Body text defaults to `text-sm`** — this is the base size for all content
- **`text-muted-foreground` for secondary text** — never use `text-gray-*` or `opacity-*`
- **`font-bold` for page/section titles and action buttons**

---

## Dark mode

Dark mode is always supported via `next-themes`, even if the project doesn't expose a theme toggle to the user.

### Rules

- Never use `dark:` variant in component classes — all dark mode is handled via CSS variable overrides
- Every new color token must have both `:root` and `.dark` values
- Test dark mode when adding new tokens

---

## Component-specific patterns

### Button

Default size uses compact padding: `px-2 py-1 has-[>svg]:px-2`. Action buttons in detail pages use `font-bold`.

### Dialog/Modal

- Padding: `p-4` (16px)
- Overlay: `bg-overlay/50`
- Confirmation button gap: `gap-2` (8px)
- Destructive confirmation buttons use `variant="destructive"`

### Popover (actions menu)

- Trigger button: `size-7` (28×28)
- Content padding: `p-0.5` (2px)
- Destructive options (e.g., delete) use `text-destructive`
- All destructive actions must open a confirmation dialog

### Table

- Row hover: `bg-muted` (100% opacity) — applied only on body rows, **not** on header rows
- Body rows use `group/row` class for descendant hover effects
- Image column placeholder: `size-10` (40×40) container, `rounded-[6px]`, `bg-muted`, icon `size-4` (16×16) with `text-placeholder-icon`
- On row hover, placeholder container gets `border-placeholder-icon`
- Actions column uses `meta: { isLast: true }` with `stopPropagation` to prevent row click

### Sidebar

- Active sub-item: `font-bold text-primary` (text + icon), **no background change**
- Active parent item: **no background change** when child is selected
- Hover background: `bg-sidebar-accent` (same as muted / table hover)

### Placeholder images

Used in tables and detail page headers for entities without actual images:

| Context | Container size | Border radius | Icon size | Icon color |
|---------|---------------|---------------|-----------|------------|
| Table row | `size-10` (40×40) | `rounded-[6px]` (6px) | `size-4` (16×16) | `text-placeholder-icon` |
| Detail header | `size-14` (56×56) | `rounded-md` | 16px | `text-placeholder-icon` |

Container uses `bg-muted`. In tables, adds `border border-transparent group-hover/row:border-placeholder-icon` for hover effect.

---

## Layout components

### DetailLayout

12-column grid with 9/3 split for main content and sidebar:

```tsx
<DetailLayout
  header={<DetailHeader ... />}
  main={/* 9 columns */}
  sidebar={/* 3 columns */}
/>
```

### DetailHeader

Shared header for detail pages with breadcrumbs, icon placeholder, title, badge, and actions:

```tsx
<DetailHeader
  breadcrumbs={[{ href: '/crop-types', label: 'Tipos de Cultura' }]}
  backHref={`/crop-types/${id}`}
  backLabel={cropType.name}
  title="Variedades"
  icon={Grape}           // optional — omit to hide placeholder image
  className="pb-2"       // optional — override default pb-6
  actions={<Button>...</Button>}
/>
```

- `icon` is optional — when omitted, the 56×56 placeholder container is not rendered
- `className` allows overriding the default `pb-6` padding

### DetailSection

Section container with title, optional link, and optional action button:

```tsx
<DetailSection
  title="Variedades"
  linkHref={`/crop-types/${id}/varieties`}
  linkLabel="Examinar mais itens"
  actions={<Button size="icon" className="size-7">...</Button>}
>
  {/* content */}
</DetailSection>
```

### EmptyState

- Listing pages: default `py-30` (120px)
- Detail page sections: `className="py-15"` (60px)

---

## Submodule detail pages

Sub-entities in a 1-N relationship (e.g., varieties under crop-types) use a different layout than top-level detail pages:

- **No `DetailLayout`** — no sidebar column, no grid
- **No placeholder image** in header
- **No breadcrumbs** — replaced by a type label with icon
- **Custom header** with: icon (`Rows4`, 12×12) + entity type label (uppercase, 13px, `text-muted-foreground font-medium`) + title below (8px gap)
- **Separator** below header: 8px gap above (`pb-2`), 16px gap below (`mb-4`)
- **Inline fields** after separator: label (`text-[14px] text-muted-foreground`) + value 8px below (`gap-2`), links use `text-primary`
- **Multiple fields** in a row separated by `<Separator orientation="vertical" />` with `mx-5` (20px gap)
- **32px gap** (`mb-8`) between fields row and next section (e.g., auditoria)
- Actions buttons aligned to bottom (`items-end`) of header

```tsx
<header className="flex shrink-0 items-end justify-between pb-2">
  <div className="flex flex-col gap-2">
    <div className="flex items-center gap-2">
      <Rows4 size={12} className="text-muted-foreground" />
      <span className="text-[13px] font-medium uppercase text-muted-foreground">
        Variedade
      </span>
    </div>
    <h1 className="text-[28px] font-bold leading-none">{name}</h1>
  </div>
  <div className="flex items-end gap-2">{actions}</div>
</header>
<Separator className="mb-4" />
<div className="mb-8 flex items-start">
  {/* fields with vertical separators between them */}
</div>
```

---

## i18n

The frontend is responsible for all user-facing text. All UI text is in **Portuguese (pt-BR)**:

- Button labels: "Criar variedade", "Editar Tipo de Cultura", "Excluir Colheita", "Voltar", "Salvar alterações"
- Confirmation messages: "Tem certeza que deseja excluir esta variedade? Esta ação não pode ser desfeita."
- Toast messages: "Criando variedade...", "Excluindo colheita...", "Salvando..."
- Empty states: "Nenhuma variedade cadastrada", "Nenhum registro de auditoria"
- Date formatting: `pt-BR` locale with `dateStyle: 'long'`

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
- **One `<h1>` per page** — heading hierarchy must be sequential
- **`text-sm` is the default body size** — not `text-base`
- **Container is 1360px** (1280px content + 80px padding)
- **All clickable elements have cursor pointer** via base layer
- **Table hover is 100% opacity `bg-muted`** — only on body rows, not headers
- **Sidebar active items have no background** — only text/icon color changes
- **Placeholder images use `text-placeholder-icon`** — never `text-muted-foreground`
- **All UI text in Portuguese (pt-BR)** — backend never returns translated strings
- **Destructive actions always require confirmation** — red text + confirmation dialog
- **shadcn/ui primitives are immutable** — wrap, don't modify
- **Never use `!important`** — no `!h-4`, `!p-0`, `!text-sm` or any Tailwind `!` prefix

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

// WRONG: using opacity for muted text
<p className="text-sm opacity-50">Help text</p>
// CORRECT: use muted-foreground token
<p className="text-xs text-muted-foreground">Help text</p>

// WRONG: table hover on header rows
<TableRow className="hover:bg-muted">  // hover is only on body rows
// CORRECT: hover applied via TableBody [&_tr:hover]:bg-muted

// WRONG: using text-muted-foreground for placeholder icons
<Wheat className="text-muted-foreground" />
// CORRECT: use dedicated token
<Wheat className="text-placeholder-icon" />

// WRONG: sidebar active item with background
data-[active=true]:bg-sidebar-accent  // no bg on active
// CORRECT: only text color
data-[active=true]:text-primary data-[active=true]:font-bold

// WRONG: English text in UI
<Button>Delete</Button>
// CORRECT: Portuguese
<Button>Excluir Variedade</Button>

// WRONG: destructive action without confirmation
onClick={() => deleteEntity(id)}
// CORRECT: open confirmation dialog first
onClick={() => setDeleteDialogOpen(id)}
```
