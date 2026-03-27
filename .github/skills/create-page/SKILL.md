---
name: create-page
description: Guide for creating Next.js App Router pages and layouts. Use this when asked to add a new page, route segment, loading state, or error boundary.
---

# Skill: create-page

> Use this skill whenever you need to scaffold a new page route in the AI Task Estimator.

---

## Project Context

| Concern  | Choice |
|----------|--------|
| Routing  | Next.js 16 App Router (file-based) |
| Language | TypeScript 5 (strict) |
| Styling  | Tailwind CSS v4 |
| Data fetching | Server Components — direct `db` calls (no `fetch` in Server Components) |
| Client state | `useState` / `useEffect` in `"use client"` components only |

---

## Route Structure

```
app/
  page.tsx                       ← / (home / dashboard)
  layout.tsx                     ← root layout (shared shell)
  globals.css

  projects/
    page.tsx                     ← /projects (list)
    [id]/
      page.tsx                   ← /projects/:id (detail)
      loading.tsx                ← loading skeleton for /projects/:id
      error.tsx                  ← error boundary for /projects/:id
      stories/
        page.tsx                 ← /projects/:id/stories
      sprints/
        page.tsx                 ← /projects/:id/sprints

  developers/
    page.tsx                     ← /developers (list)
    [id]/
      page.tsx                   ← /developers/:id
```

---

## Step-by-Step: Scaffolding a Page

### 1. Server Component page with direct DB access (most common)

```tsx
// app/projects/page.tsx
// Server Component — no "use client" directive

import { db } from "@/db";
import { projects } from "@/db/schema";
import { ProjectCard } from "@/app/components/projects/project-card";
import { Button } from "@/app/components/ui/button";
import Link from "next/link";

export const metadata = {
  title: "Projects — AI Task Estimator",
};

export default async function ProjectsPage() {
  const allProjects = await db.select().from(projects).orderBy(projects.createdAt);

  return (
    <div className="mx-auto max-w-5xl px-4 py-8">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold text-gray-900">Projects</h1>
        <Link href="/projects/new">
          <Button>New Project</Button>
        </Link>
      </div>

      {allProjects.length === 0 ? (
        <p className="mt-8 text-center text-gray-500">No projects yet. Create your first one.</p>
      ) : (
        <div className="mt-6 grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
          {allProjects.map((project) => (
            <ProjectCard key={project.id} project={project} />
          ))}
        </div>
      )}
    </div>
  );
}
```

### 2. Dynamic route page with params

```tsx
// app/projects/[id]/page.tsx
// Server Component

import { notFound } from "next/navigation";
import { eq } from "drizzle-orm";
import { db } from "@/db";
import { projects, userStories, tasks } from "@/db/schema";

interface ProjectDetailPageProps {
  params: Promise<{ id: string }>;
}

export default async function ProjectDetailPage({ params }: ProjectDetailPageProps) {
  const { id } = await params;   // Always await params in Next.js 16

  const [project] = await db.select().from(projects).where(eq(projects.id, Number(id)));
  if (!project) notFound();       // Triggers the nearest not-found.tsx or Next's default 404

  const stories = await db.select().from(userStories).where(eq(userStories.projectId, project.id));

  return (
    <div className="mx-auto max-w-5xl px-4 py-8">
      <h1 className="text-2xl font-bold">{project.name}</h1>
      {/* ... */}
    </div>
  );
}
```

### 3. Loading state skeleton

```tsx
// app/projects/[id]/loading.tsx
// Server Component — shown automatically while the page is streaming

export default function ProjectDetailLoading() {
  return (
    <div className="mx-auto max-w-5xl animate-pulse px-4 py-8">
      <div className="h-8 w-48 rounded bg-gray-200" />
      <div className="mt-6 space-y-4">
        {[1, 2, 3].map((i) => (
          <div key={i} className="h-24 rounded-lg bg-gray-100" />
        ))}
      </div>
    </div>
  );
}
```

### 4. Error boundary

```tsx
// app/projects/[id]/error.tsx
"use client"; // Error boundaries MUST be Client Components

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function ProjectDetailError({ error, reset }: ErrorProps) {
  return (
    <div className="mx-auto max-w-md px-4 py-16 text-center">
      <h2 className="text-lg font-semibold text-gray-900">Something went wrong</h2>
      <p className="mt-2 text-sm text-gray-600">{error.message}</p>
      <button
        onClick={reset}
        className="mt-4 rounded bg-blue-600 px-4 py-2 text-sm text-white hover:bg-blue-700"
      >
        Try again
      </button>
    </div>
  );
}
```

### 5. Not found page

```tsx
// app/projects/[id]/not-found.tsx
// Server Component

import Link from "next/link";

export default function ProjectNotFound() {
  return (
    <div className="mx-auto max-w-md px-4 py-16 text-center">
      <h2 className="text-lg font-semibold text-gray-900">Project not found</h2>
      <p className="mt-2 text-sm text-gray-600">
        The project you are looking for does not exist or has been deleted.
      </p>
      <Link href="/projects" className="mt-4 inline-block text-sm text-blue-600 hover:underline">
        Back to Projects
      </Link>
    </div>
  );
}
```

---

## Layout Shell Pattern

Use a shared layout for all authenticated pages:

```tsx
// app/layout.tsx  (root layout — always exists)
import type { Metadata } from "next";
import "./globals.css";
import { Sidebar } from "@/app/components/layout/sidebar";
import { Header } from "@/app/components/layout/header";

export const metadata: Metadata = {
  title: "AI Task Estimator",
  description: "AI-powered sprint planning and task estimation",
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="bg-gray-50">
        <div className="flex h-screen overflow-hidden">
          <Sidebar />
          <div className="flex flex-1 flex-col overflow-hidden">
            <Header />
            <main className="flex-1 overflow-auto">{children}</main>
          </div>
        </div>
      </body>
    </html>
  );
}
```

---

## Data Fetching Rules

| Scenario | Approach |
|---|---|
| Fetch data in a page | `await db.select()...` directly in the Server Component |
| Fetch after user interaction | `fetch("/api/...")` inside a `"use client"` component |
| Mutate data | `fetch("POST /api/...")` inside a `"use client"` component |
| Never | `useEffect` + `fetch` in a Server Component |
| Never | Import `db` inside a Client Component |

---

## Page Conventions

| Convention | Rule |
|---|---|
| File | `page.tsx` — exactly this name |
| Export | `export default async function {Name}Page()` |
| Metadata | Export `metadata` object or `generateMetadata` for SEO |
| Params | Always `params: Promise<{ id: string }>` and `await params` |
| Loading | `loading.tsx` in the same segment for auto Suspense boundary |
| Errors | `error.tsx` must be `"use client"` |
| Not found | Call `notFound()` from `next/navigation` when row is missing |

---

## Checklist: New Page

- [ ] File created at `app/{segment}/page.tsx`
- [ ] `export default async function {Name}Page()` (async for Server Components)
- [ ] `params` typed as `Promise<{ id: string }>` and awaited
- [ ] `notFound()` called when fetched row is `undefined`
- [ ] `loading.tsx` added for any page with DB calls
- [ ] `error.tsx` added with `"use client"` for dynamic segments
- [ ] `export const metadata = { title: "..." }` set
- [ ] No `db` imports inside `"use client"` components
