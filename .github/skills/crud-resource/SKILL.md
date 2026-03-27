---
name: crud-resource
description: Guide for scaffolding a complete full-stack CRUD resource — database schema, Zod schemas, API routes, list page, and create/edit form. Use this when asked to add a full new resource or entity to the application.
---

# Skill: crud-resource

> Use this skill to scaffold a complete CRUD resource end-to-end. This skill orchestrates the `add-drizzle-schema`, `create-api-endpoint`, `create-page`, `create-form`, and `create-ui-component` skills in the correct order.

---

## Phase Order (Always Follow This Sequence)

1. **Schema** — Add table to `db/schema.ts` → run `yarn db:push`
2. **Zod schemas** — Derive insert/update schemas in `lib/schemas/{resource}.schema.ts`
3. **API routes** — Collection + single-item route handlers
4. **UI components** — Card/row component for displaying the resource
5. **List page** — Server Component listing all records
6. **Form component** — `"use client"` form for create + edit
7. **Error/loading states** — `loading.tsx` + `error.tsx` + `not-found.tsx`

**Run `get_errors` after each phase before proceeding to the next.**

---

## Step-by-Step Example: `developers` Resource

### Phase 1 — Schema (`db/schema.ts`)

> See `add-drizzle-schema` skill for full column reference.

```ts
// db/schema.ts — add to existing file
export const developers = sqliteTable("developers", {
  id:            int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  projectId:     int("project_id", { mode: "number" }).notNull().references(() => projects.id),
  name:          text("name").notNull(),
  role:          text("role", { enum: ["frontend", "backend", "fullstack", "devops", "qa"] }).notNull(),
  skillset:      text("skillset").notNull(),    // JSON array: '["React","Node.js"]'
  capacityHours: real("capacity_hours").notNull().default(80),
  createdAt:     int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:     int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
export type Developer    = typeof developers.$inferSelect;
export type NewDeveloper = typeof developers.$inferInsert;
```

```bash
yarn db:push
```

---

### Phase 2 — Zod Schemas (`lib/schemas/developer.schema.ts`)

```ts
// lib/schemas/developer.schema.ts
import { createInsertSchema, createSelectSchema } from "drizzle-zod";
import { developers } from "@/db/schema";

export const developerSchema       = createSelectSchema(developers);
export const createDeveloperSchema = createInsertSchema(developers).omit({ id: true, createdAt: true, updatedAt: true });
export const updateDeveloperSchema = createDeveloperSchema.partial();

export type CreateDeveloperInput = typeof createDeveloperSchema._type;
export type UpdateDeveloperInput = typeof updateDeveloperSchema._type;
```

---

### Phase 3 — API Routes

> See `create-api-endpoint` skill for the full route handler template.

**Collection:** `app/api/developers/route.ts`
- `GET` — list all developers (optionally filtered by `?projectId=`)
- `POST` — create a developer

**Single item:** `app/api/developers/[id]/route.ts`
- `GET` — fetch one by id
- `PATCH` — partial update
- `DELETE` — remove

Example collection route with query filter:

```ts
// app/api/developers/route.ts
import { eq } from "drizzle-orm";
import { db } from "@/db";
import { developers } from "@/db/schema";
import { createDeveloperSchema } from "@/lib/schemas/developer.schema";
import { badRequest, created, internalError, ok } from "@/lib/api-response";

export async function GET(request: Request) {
  try {
    const { searchParams } = new URL(request.url);
    const projectId = searchParams.get("projectId");

    const rows = projectId
      ? await db.select().from(developers).where(eq(developers.projectId, Number(projectId)))
      : await db.select().from(developers);

    return ok(rows);
  } catch {
    return internalError();
  }
}

export async function POST(request: Request) {
  try {
    const body: unknown = await request.json();
    const parsed = createDeveloperSchema.safeParse(body);
    if (!parsed.success) return badRequest("Validation failed", parsed.error.flatten());
    const [row] = await db.insert(developers).values(parsed.data).returning();
    return created(row);
  } catch {
    return internalError();
  }
}
```

---

### Phase 4 — UI Component (`app/components/developers/developer-card.tsx`)

> See `create-ui-component` skill for the full component template.

```tsx
// app/components/developers/developer-card.tsx
import type { Developer } from "@/db/schema";
import { Badge } from "@/app/components/ui/badge";

export function DeveloperCard({ developer }: { developer: Developer }) {
  const skills: string[] = (() => {
    try { return JSON.parse(developer.skillset); } catch { return []; }
  })();
  return (
    <div className="rounded-lg border bg-white p-4 shadow-sm">
      <div className="flex items-center justify-between">
        <h3 className="font-semibold text-gray-900">{developer.name}</h3>
        <Badge variant="info">{developer.role}</Badge>
      </div>
      <div className="mt-2 flex flex-wrap gap-1">
        {skills.map((s) => <Badge key={s}>{s}</Badge>)}
      </div>
      <p className="mt-2 text-xs text-gray-500">{developer.capacityHours}h / sprint</p>
    </div>
  );
}
```

---

### Phase 5 — List Page (`app/projects/[id]/developers/page.tsx`)

> See `create-page` skill for full page templates.

```tsx
// app/projects/[id]/developers/page.tsx
import { eq } from "drizzle-orm";
import { db } from "@/db";
import { developers } from "@/db/schema";
import { DeveloperCard } from "@/app/components/developers/developer-card";
import { Modal } from "@/app/components/ui/modal";
import { Button } from "@/app/components/ui/button";
import { DeveloperForm } from "@/app/components/developers/developer-form";

interface Props { params: Promise<{ id: string }> }

export default async function DevelopersPage({ params }: Props) {
  const { id } = await params;
  const rows = await db.select().from(developers).where(eq(developers.projectId, Number(id)));

  return (
    <div className="mx-auto max-w-5xl px-4 py-8">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Team Roster</h1>
        <Modal title="Add Developer" trigger={<Button>Add Developer</Button>}>
          <DeveloperForm projectId={Number(id)} />
        </Modal>
      </div>
      <div className="mt-6 grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {rows.map((dev) => <DeveloperCard key={dev.id} developer={dev} />)}
      </div>
    </div>
  );
}
```

---

### Phase 6 — Form Component

> See `create-form` skill for the full form template.

Create `app/components/developers/developer-form.tsx` with:
- Fields: `name`, `role` (select), `skillset` (text), `capacityHours` (number)
- POST to `/api/developers` on create
- PATCH to `/api/developers/:id` on edit
- Zod `safeParse` before submission
- `router.refresh()` on success

---

### Phase 7 — Loading + Error states

Add to the same route segment:
- `loading.tsx` — animated skeleton
- `error.tsx` — `"use client"` error boundary
- `not-found.tsx` — fallback when record is missing

---

## CRUD Checklist

### Backend

- [ ] Table added to `db/schema.ts` with `id`, `createdAt`, `updatedAt`
- [ ] `yarn db:push` executed
- [ ] `lib/schemas/{resource}.schema.ts` created with select, insert, and update schemas
- [ ] `app/api/{resource}/route.ts` — `GET` + `POST`
- [ ] `app/api/{resource}/[id]/route.ts` — `GET` + `PATCH` + `DELETE`
- [ ] All handlers: try/catch, `safeParse`, response helpers only

### Frontend

- [ ] Card/row component in `app/components/{resource}/`
- [ ] List page at `app/{segment}/page.tsx` — Server Component with direct `db.select()`
- [ ] Form component with `"use client"`, Zod validation, and `fetch` calls
- [ ] Edit flow: form accepts `initialValues` + `id`, uses `PATCH`
- [ ] Delete button with confirmation
- [ ] `loading.tsx` for list page
- [ ] `error.tsx` for dynamic segments
- [ ] `not-found.tsx` for `[id]` routes

---

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Table | `snake_case` plural | `user_stories` |
| Route | `kebab-case` plural | `/api/user-stories` |
| Schema file | `{resource}.schema.ts` | `user-story.schema.ts` |
| Component folder | `{domain}/` | `app/components/stories/` |
| Page path | `app/{segment}/` | `app/stories/` |
