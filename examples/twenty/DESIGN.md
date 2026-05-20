---
devmd: design
version: "1.0"
project: Twenty CRM

design_system:
  package: twenty-ui
  build: "Vite + SWC (ESM-first)"
  styling: Linaria (CSS-in-JS, zero-runtime)
  icons: "Tabler Icons (via registry)"
  colors: Radix UI Colors
  animation: Framer Motion
  responsive: react-responsive
  storybook: "v10.3"
  visual_regression: Chromatic

themes:
  - name: light
    file: light-theme.css
  - name: dark
    file: dark-theme.css
  default: light
  switching: "CSS class on <html> element"

tokens:
  colors:
    primary:
      light: "#1A1A1A"
      dark: "#F2F2F2"
    accent:
      light: "#4D77FF"
      dark: "#6B93FF"
    background:
      primary:
        light: "#FFFFFF"
        dark: "#141414"
      secondary:
        light: "#F9F9F9"
        dark: "#1C1C1C"
      tertiary:
        light: "#F0F0F0"
        dark: "#242424"
    text:
      primary:
        light: "#1A1A1A"
        dark: "#F2F2F2"
      secondary:
        light: "#666666"
        dark: "#999999"
      tertiary:
        light: "#999999"
        dark: "#666666"
    border:
      light: "#E0E0E0"
      dark: "#333333"
    error:
      light: "#E53E3E"
      dark: "#FC8181"
    warning:
      light: "#DD6B20"
      dark: "#F6AD55"
    success:
      light: "#38A169"
      dark: "#68D391"
    info:
      light: "#3182CE"
      dark: "#63B3ED"

  spacing:
    unit: 4px
    scale: [0, 1, 2, 3, 4, 5, 6, 8, 10, 12, 16, 20, 24, 32, 40, 48, 64]
    description: "spacing[n] = n * 4px. spacing[4] = 16px."

  typography:
    font_family: "Inter, -apple-system, BlinkMacSystemFont, sans-serif"
    font_mono: "'JetBrains Mono', 'Fira Code', monospace"
    scale:
      - name: xxs
        size: 10px
        line_height: 14px
        weight: 400
      - name: xs
        size: 12px
        line_height: 16px
        weight: 400
      - name: sm
        size: 13px
        line_height: 18px
        weight: 400
      - name: md
        size: 14px
        line_height: 20px
        weight: 400
      - name: lg
        size: 16px
        line_height: 24px
        weight: 500
      - name: xl
        size: 20px
        line_height: 28px
        weight: 600
      - name: xxl
        size: 24px
        line_height: 32px
        weight: 700

  border_radius:
    sm: 4px
    md: 8px
    lg: 12px
    xl: 16px
    full: 9999px

  shadows:
    sm: "0 1px 2px rgba(0, 0, 0, 0.05)"
    md: "0 4px 6px rgba(0, 0, 0, 0.07)"
    lg: "0 10px 15px rgba(0, 0, 0, 0.1)"
    xl: "0 20px 25px rgba(0, 0, 0, 0.15)"

  animation:
    duration:
      fast: 100ms
      normal: 200ms
      slow: 300ms
    easing:
      ease_in_out: "cubic-bezier(0.4, 0, 0.2, 1)"
      ease_out: "cubic-bezier(0, 0, 0.2, 1)"
      spring: "spring(1, 80, 10, 0)"

component_library:
  framework: Mantine 8
  custom_layer: twenty-ui
  submodules:
    - name: accessibility
      description: "Focus management, keyboard navigation, screen reader utilities"
    - name: components
      description: "Base components (Button, Input, Chip, Tag, Avatar)"
    - name: display
      description: "Data display (Table, Card, Badge, Tooltip)"
    - name: feedback
      description: "Snackbar, Toast, Dialog, Modal, ProgressBar"
    - name: input
      description: "TextInput, Select, DatePicker, Toggle, Checkbox, Radio, AutoComplete"
    - name: json-visualizer
      description: "JSON tree viewer/editor for raw data fields"
    - name: layout
      description: "PageContainer, SplitPane, Sidebar, Header, Section"
    - name: navigation
      description: "NavigationBar, Breadcrumb, Tab, CommandMenu"
    - name: testing
      description: "Test utilities and decorators for Storybook"
    - name: theme
      description: "ThemeProvider, useTheme, theme context"
    - name: theme-constants
      description: "Token values exported as JS constants"
    - name: utilities
      description: "Hooks, helpers, formatters"
    - name: assets
      description: "Illustrations, logos, brand assets"
---

# Twenty CRM Design System

## twenty-ui Package

The design system lives in `packages/twenty-ui/`. ESM-first build using Vite + SWC. All components use Linaria for zero-runtime CSS-in-JS.

### Architecture

```
twenty-ui/
├── src/
│   ├── accessibility/     # Focus trap, keyboard nav, aria utilities
│   ├── assets/            # SVG illustrations, logos
│   ├── components/        # Atomic components
│   │   ├── Button/
│   │   │   ├── Button.tsx
│   │   │   ├── Button.styles.ts    # Linaria styled components
│   │   │   ├── Button.stories.tsx  # Storybook stories
│   │   │   └── Button.spec.tsx     # Unit tests
│   │   ├── Input/
│   │   ├── Chip/
│   │   ├── Tag/
│   │   ├── Avatar/
│   │   └── ...
│   ├── display/           # Data display components
│   ├── feedback/          # User feedback components
│   ├── input/             # Form input components
│   ├── json-visualizer/   # JSON tree viewer
│   ├── layout/            # Layout components
│   ├── navigation/        # Navigation components
│   ├── testing/           # Storybook decorators, test utils
│   ├── theme/             # ThemeProvider, context
│   ├── theme-constants/   # Token values as JS
│   └── utilities/         # Shared hooks and helpers
├── .storybook/            # Storybook v10.3 config
├── light-theme.css        # CSS custom properties (light)
├── dark-theme.css         # CSS custom properties (dark)
└── package.json
```

### Linaria Styling Pattern

```typescript
// Button.styles.ts
import { styled } from '@linaria/react';
import { THEME_CONSTANTS } from '../../theme-constants';

export const StyledButton = styled.button<{ variant: 'primary' | 'secondary' }>`
  display: inline-flex;
  align-items: center;
  gap: ${THEME_CONSTANTS.spacing[2]}px;
  padding: ${THEME_CONSTANTS.spacing[2]}px ${THEME_CONSTANTS.spacing[4]}px;
  border-radius: ${THEME_CONSTANTS.borderRadius.md}px;
  font-size: ${THEME_CONSTANTS.typography.sm.size};
  font-weight: 500;
  cursor: pointer;
  transition: background-color ${THEME_CONSTANTS.animation.duration.fast};

  background-color: ${(props) =>
    props.variant === 'primary'
      ? 'var(--color-accent)'
      : 'var(--color-background-secondary)'};
  color: ${(props) =>
    props.variant === 'primary'
      ? 'var(--color-text-inverted)'
      : 'var(--color-text-primary)'};

  &:hover {
    opacity: 0.9;
  }

  &:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
`;
```

### Icon System

Icons from Tabler Icons, loaded via a registry pattern for tree-shaking.

```typescript
import { IconUser, IconBuilding, IconCurrencyDollar } from '@tabler/icons-react';

// Icon registry for dynamic icon rendering (e.g., from metadata)
const ICON_REGISTRY = {
  IconUser,
  IconBuilding,
  IconCurrencyDollar,
  // ... 200+ icons registered
};

// Usage with metadata-driven icon name
<DynamicIcon name={fieldMetadata.icon} size={16} />
```

### Theme Switching

Dual theme via CSS custom properties. Theme class applied to `<html>`.

```css
/* light-theme.css */
:root {
  --color-primary: #1A1A1A;
  --color-accent: #4D77FF;
  --color-background-primary: #FFFFFF;
  --color-background-secondary: #F9F9F9;
  --color-text-primary: #1A1A1A;
  --color-text-secondary: #666666;
  --color-border: #E0E0E0;
}

/* dark-theme.css */
:root.dark {
  --color-primary: #F2F2F2;
  --color-accent: #6B93FF;
  --color-background-primary: #141414;
  --color-background-secondary: #1C1C1C;
  --color-text-primary: #F2F2F2;
  --color-text-secondary: #999999;
  --color-border: #333333;
}
```

### Storybook & Chromatic

- Storybook v10.3 for component documentation and development
- Every component has stories for all variants and states
- Decorators for light/dark theme, responsive breakpoints, RTL
- Chromatic runs on every PR for visual regression detection. See @TESTING.md#visual-regression

## Cross-References

- @UI.md — How design system components compose into pages
- @BRAND.md — Voice and messaging that inform UI copy
- @TESTING.md — Visual regression testing with Chromatic
- @CLAUDE.md — Coding conventions for styling (Linaria, no hardcoded colors)
