---
name: create-ui-component
description: Guide for creating reusable React UI components with Tailwind CSS v4. Use this when asked to build a new shared component, card, badge, button, table, or any reusable UI element.
---

# Skill: create-ui-component

> Use this skill whenever you need to create a new reusable UI component for the AI Task Estimator.

---

## Project Context

| Concern   | Choice |
|-----------|--------|
| Framework | Next.js 16 App Router |
| Styling   | Tailwind CSS v4 |
| Language  | TypeScript 5 (strict) |
| Component location | `app/components/` |

---

## Component Architecture

```
app/
  components/
    ui/                   ← Generic primitives (Button, Badge, Card, Table, etc.)
    layout/               ← Layout pieces (Header, Sidebar, PageShell)
    projects/             ← Project-scoped components
    developers/           ← Developer-scoped components
    stories/              ← User story scoped components
    tasks/                ← Task-scoped components
    sprints/              ← Sprint board components
```

**Rule:** Generic primitives live in `app/components/ui/`. Domain components live in their own folder. Pages import components — they do not contain markup themselves.

---

## Server vs Client Components

| Component is... | Directive needed |
|---|---|
| Reading data, no interactivity | None (Server Component by default) |
| Has `onClick`, `onChange`, `useState`, `useEffect` | `"use client"` at top of file |
| Uses browser APIs (`window`, `localStorage`) | `"use client"` |

**Always default to Server Component** unless interactivity is required.

---

## Step-by-Step: Creating a UI Primitive

### Example: `Badge` component

```tsx
// app/components/ui/badge.tsx
// Server Component — no "use client" needed

import { type ReactNode } from "react";

type BadgeVariant = "default" | "success" | "warning" | "danger" | "info";

interface BadgeProps {
  variant?: BadgeVariant;
  children: ReactNode;
  className?: string;
}

const variantClasses: Record<BadgeVariant, string> = {
  default: "bg-gray-100 text-gray-800",
  success: "bg-green-100 text-green-800",
  warning: "bg-yellow-100 text-yellow-800",
  danger:  "bg-red-100 text-red-800",
  info:    "bg-blue-100 text-blue-800",
};

export function Badge({ variant = "default", children, className = "" }: BadgeProps) {
  return (
    <span
      className={`inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium ${variantClasses[variant]} ${className}`}
    >
      {children}
    </span>
  );
}
```

### Example: `Button` component (interactive)

```tsx
// app/components/ui/button.tsx
"use client";

import { type ButtonHTMLAttributes } from "react";

type ButtonVariant = "primary" | "secondary" | "danger" | "ghost";
type ButtonSize    = "sm" | "md" | "lg";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
  size?: ButtonSize;
  isLoading?: boolean;
}

const variantClasses: Record<ButtonVariant, string> = {
  primary:   "bg-blue-600 text-white hover:bg-blue-700 disabled:bg-blue-300",
  secondary: "bg-gray-200 text-gray-800 hover:bg-gray-300 disabled:bg-gray-100",
  danger:    "bg-red-600 text-white hover:bg-red-700 disabled:bg-red-300",
  ghost:     "bg-transparent text-gray-700 hover:bg-gray-100",
};

const sizeClasses: Record<ButtonSize, string> = {
  sm: "px-3 py-1.5 text-sm",
  md: "px-4 py-2 text-sm",
  lg: "px-6 py-3 text-base",
};

export function Button({
  variant = "primary",
  size = "md",
  isLoading = false,
  disabled,
  children,
  className = "",
  ...rest
}: ButtonProps) {
  return (
    <button
      disabled={disabled || isLoading}
      className={`inline-flex items-center justify-center rounded-md font-medium transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 disabled:cursor-not-allowed ${variantClasses[variant]} ${sizeClasses[size]} ${className}`}
      {...rest}
    >
      {isLoading ? (
        <span className="mr-2 inline-block h-4 w-4 animate-spin rounded-full border-2 border-current border-t-transparent" />
      ) : null}
      {children}
    </button>
  );
}
```

---

## Step-by-Step: Creating a Domain Component

### Example: `DeveloperCard` (Server Component)

```tsx
// app/components/developers/developer-card.tsx

import type { Developer } from "@/db/schema";
import { Badge } from "@/app/components/ui/badge";

interface DeveloperCardProps {
  developer: Developer;
}

const roleBadgeVariant = (role: Developer["role"]) => {
  const map = { frontend: "info", backend: "success", fullstack: "warning", devops: "danger", qa: "default" } as const;
  return map[role] ?? "default";
};

export function DeveloperCard({ developer }: DeveloperCardProps) {
  const skills: string[] = JSON.parse(developer.skillset ?? "[]");

  return (
    <div className="rounded-lg border border-gray-200 bg-white p-4 shadow-sm">
      <div className="flex items-center justify-between">
        <h3 className="text-sm font-semibold text-gray-900">{developer.name}</h3>
        <Badge variant={roleBadgeVariant(developer.role)}>{developer.role}</Badge>
      </div>
      <div className="mt-2 flex flex-wrap gap-1">
        {skills.map((skill) => (
          <Badge key={skill} variant="default">{skill}</Badge>
        ))}
      </div>
      <p className="mt-3 text-xs text-gray-500">
        Capacity: <span className="font-medium">{developer.capacityHours}h / sprint</span>
      </p>
    </div>
  );
}
```

---

## Component Conventions

| Rule | Detail |
|---|---|
| File name | `kebab-case.tsx` |
| Export | Named export only — never `export default` for components |
| Props interface | Always defined above the component, named `{Component}Props` |
| TypeScript | All props typed — never use `any` |
| Tailwind classes | Use string template literals or `cn()` helper for conditional classes |
| No inline `style={}` | Use Tailwind classes instead |
| No raw `<button>` in pages | Always use the shared `<Button>` primitive |
| No raw `<table>` in pages | Always use shared table primitives |

---

## `cn()` Utility for Conditional Classes

Create a small utility if conditional class merging is needed:

```ts
// lib/cn.ts
export function cn(...classes: (string | undefined | false | null)[]): string {
  return classes.filter(Boolean).join(" ");
}
```

Usage:
```tsx
import { cn } from "@/lib/cn";

<div className={cn("base-class", isActive && "active-class", className)} />
```

---

## Checklist: New UI Component

- [ ] Correct directory: `app/components/ui/` for primitives, `app/components/{domain}/` for domain components
- [ ] Server Component by default — add `"use client"` only if interactive
- [ ] Props interface named `{ComponentName}Props`
- [ ] All props fully typed — no `any`
- [ ] Named export (never `export default`)
- [ ] Uses `@/` path alias for all internal imports
- [ ] No inline styles — Tailwind only
- [ ] No raw `<button>` / `<input>` / `<table>` in domain components — import primitives from `app/components/ui/`
