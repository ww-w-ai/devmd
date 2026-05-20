---
devmd: design
version: "1.0"
project: documenso
updated: 2026-05-13

design_system:
  name: "Documenso UI"
  package: "@documenso/ui"
  base: "Radix UI + shadcn/ui + CVA"
  styling: "Tailwind CSS"
  animation: "Framer Motion"
  canvas: "Konva (PDF rendering)"
  figma: "https://www.figma.com/community/file/documenso"
  docs: "https://design.documenso.com"

theme:
  modes: [light, dark]
  default: system
  strategy: "CSS custom properties + Tailwind dark: variant"

colors:
  primary:
    DEFAULT: "hsl(142.1, 76.2%, 36.3%)"    # Brand green
    foreground: "hsl(355.7, 100%, 97.3%)"
  secondary:
    DEFAULT: "hsl(240, 4.8%, 95.9%)"
    foreground: "hsl(240, 5.9%, 10%)"
  destructive:
    DEFAULT: "hsl(0, 84.2%, 60.2%)"
    foreground: "hsl(0, 0%, 98%)"
  muted:
    DEFAULT: "hsl(240, 4.8%, 95.9%)"
    foreground: "hsl(240, 3.8%, 46.1%)"
  accent:
    DEFAULT: "hsl(240, 4.8%, 95.9%)"
    foreground: "hsl(240, 5.9%, 10%)"
  background: "hsl(0, 0%, 100%)"
  foreground: "hsl(240, 10%, 3.9%)"
  card:
    DEFAULT: "hsl(0, 0%, 100%)"
    foreground: "hsl(240, 10%, 3.9%)"
  popover:
    DEFAULT: "hsl(0, 0%, 100%)"
    foreground: "hsl(240, 10%, 3.9%)"
  border: "hsl(240, 5.9%, 90%)"
  input: "hsl(240, 5.9%, 90%)"
  ring: "hsl(142.1, 76.2%, 36.3%)"

  dark:
    background: "hsl(240, 10%, 3.9%)"
    foreground: "hsl(0, 0%, 98%)"
    primary:
      DEFAULT: "hsl(142.1, 70.6%, 45.3%)"
      foreground: "hsl(144.9, 80.4%, 10%)"
    card:
      DEFAULT: "hsl(240, 10%, 3.9%)"
      foreground: "hsl(0, 0%, 98%)"
    border: "hsl(240, 3.7%, 15.9%)"

typography:
  font_sans: "Inter, system-ui, sans-serif"
  font_signature: "Caveat, cursive"
  scale:
    xs: { size: "0.75rem", line_height: "1rem" }
    sm: { size: "0.875rem", line_height: "1.25rem" }
    base: { size: "1rem", line_height: "1.5rem" }
    lg: { size: "1.125rem", line_height: "1.75rem" }
    xl: { size: "1.25rem", line_height: "1.75rem" }
    2xl: { size: "1.5rem", line_height: "2rem" }
    3xl: { size: "1.875rem", line_height: "2.25rem" }
    4xl: { size: "2.25rem", line_height: "2.5rem" }

spacing:
  unit: "0.25rem (4px)"
  scale: [0, 1, 2, 3, 4, 5, 6, 8, 10, 12, 16, 20, 24, 32, 40, 48, 64]

border_radius:
  sm: "calc(var(--radius) - 4px)"
  md: "calc(var(--radius) - 2px)"
  lg: "var(--radius)"
  xl: "calc(var(--radius) + 4px)"
  full: "9999px"
  radius_variable: "0.5rem"
---

# DESIGN.md — Documenso

## Design System Overview

Documenso's UI is built on the **Radix UI + shadcn/ui** pattern with **CVA** (Class Variance Authority) for component variants. The `@documenso/ui` package contains 50+ primitives shared across the application.

### Component Architecture

```
@documenso/ui/
├── primitives/          # 50+ base components
│   ├── alert.tsx
│   ├── avatar.tsx
│   ├── badge.tsx
│   ├── button.tsx
│   ├── card.tsx
│   ├── checkbox.tsx
│   ├── command.tsx
│   ├── data-table.tsx
│   ├── dialog.tsx
│   ├── dropdown-menu.tsx
│   ├── form.tsx
│   ├── input.tsx
│   ├── label.tsx
│   ├── popover.tsx
│   ├── select.tsx
│   ├── separator.tsx
│   ├── sheet.tsx
│   ├── skeleton.tsx
│   ├── switch.tsx
│   ├── table.tsx
│   ├── tabs.tsx
│   ├── textarea.tsx
│   ├── toast.tsx
│   ├── tooltip.tsx
│   └── ... (50+ total)
├── lib/
│   └── utils.ts         # cn() helper (clsx + tailwind-merge)
└── document-signing/    # Domain-specific signing components
    ├── signature-pad.tsx
    ├── field-renderer.tsx
    ├── pdf-viewer.tsx
    └── signing-card.tsx
```

### CVA Pattern

All components use CVA for variant management:

```typescript
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  },
);
```

## Theme System

### CSS Custom Properties

Themes are implemented via CSS custom properties on `:root` and `.dark`:

```css
:root {
  --background: 0 0% 100%;
  --foreground: 240 10% 3.9%;
  --primary: 142.1 76.2% 36.3%;
  --primary-foreground: 355.7 100% 97.3%;
  --secondary: 240 4.8% 95.9%;
  --secondary-foreground: 240 5.9% 10%;
  --destructive: 0 84.2% 60.2%;
  --destructive-foreground: 0 0% 98%;
  --muted: 240 4.8% 95.9%;
  --muted-foreground: 240 3.8% 46.1%;
  --accent: 240 4.8% 95.9%;
  --accent-foreground: 240 5.9% 10%;
  --border: 240 5.9% 90%;
  --input: 240 5.9% 90%;
  --ring: 142.1 76.2% 36.3%;
  --radius: 0.5rem;
}

.dark {
  --background: 240 10% 3.9%;
  --foreground: 0 0% 98%;
  --primary: 142.1 70.6% 45.3%;
  --primary-foreground: 144.9 80.4% 10%;
  /* ... */
}
```

### Tailwind Configuration

```typescript
// packages/tailwind-config/tailwind.config.ts
export default {
  darkMode: ['class'],
  content: [
    './app/**/*.{ts,tsx}',
    '../../packages/ui/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        // ... all semantic colors
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
      fontFamily: {
        sans: ['Inter', ...fontFamily.sans],
        signature: ['Caveat', 'cursive'],
      },
    },
  },
  plugins: [tailwindcssAnimate],
};
```

## Animation

### Framer Motion

Page transitions and component animations use Framer Motion:

```typescript
// Common animation patterns
const fadeIn = {
  initial: { opacity: 0, y: 10 },
  animate: { opacity: 1, y: 0 },
  transition: { duration: 0.3 },
};

const stagger = {
  animate: { transition: { staggerChildren: 0.05 } },
};
```

### Tailwind Animate

Utility animations via `tailwindcss-animate` plugin:

- `animate-in` / `animate-out` for enter/exit
- `fade-in` / `fade-out`
- `slide-in-from-top` / `slide-in-from-bottom`
- Used primarily in dialogs, popovers, and toasts

## PDF Canvas (Konva)

Document viewing and field placement use **Konva** (HTML5 Canvas library):

- PDF pages rendered as images on Konva Stage
- Fields are draggable/resizable Konva shapes
- Signature pad uses custom Konva drawing layer
- Zoom and pan supported

See @UI.md#signing-flow for the signing UI flow.

## Iconography

- **Lucide React** — Primary icon set (consistent with shadcn/ui ecosystem)
- Custom icons for brand-specific elements (logo, document states)

## Figma

- Community file available at Figma Community
- Design documentation at `design.documenso.com`
- Tokens sync between Figma and code via CSS custom properties

## Cross-References

- Component usage in routes: @UI.md
- Brand colors and voice: @BRAND.md
- Tailwind config package: @ARCHITECTURE.md#monorepo
