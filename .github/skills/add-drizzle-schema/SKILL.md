---
name: add-drizzle-schema
description: Guide for adding new SQLite tables to db/schema.ts using Drizzle ORM. Use this when asked to add a new table, entity, or column to the database schema.
---

# Skill: add-drizzle-schema

> Use this skill whenever you need to add a new database table or modify an existing table in `db/schema.ts`.

---

## Project Context

| Concern    | Choice                                    |
|------------|-------------------------------------------|
| ORM        | Drizzle ORM (`drizzle-orm`)               |
| Database   | SQLite via `better-sqlite3` (`local.db`)  |
| Validation | Zod 4 derived via `drizzle-zod`           |
| Schema file | `db/schema.ts` — single source of truth  |

---

## Step-by-Step: Adding a New Table

### 1. Import the correct column builders at the top of `db/schema.ts`

```ts
import { int, sqliteTable, text, real } from "drizzle-orm/sqlite-core";
```

**Available column types for SQLite:**

| Column builder | SQLite type | Use for |
|---|---|---|
| `text(name)` | TEXT | strings, enums, JSON |
| `int(name, { mode: "number" })` | INTEGER | integers, IDs |
| `int(name, { mode: "timestamp" })` | INTEGER | dates stored as Unix ms |
| `real(name)` | REAL | floats (hours, percentages) |

### 2. Define the table

```ts
// db/schema.ts

export const developers = sqliteTable("developers", {
  id:           int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  name:         text("name").notNull(),
  role:         text("role", { enum: ["frontend", "backend", "fullstack", "devops"] }).notNull(),
  skillset:     text("skillset").notNull(),          // comma-separated or JSON string
  capacityHours: real("capacity_hours").notNull().default(80), // hours per sprint
  createdAt:    int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:    int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
```

### 3. Export inferred TypeScript types (always required)

```ts
export type Developer    = typeof developers.$inferSelect;
export type NewDeveloper = typeof developers.$inferInsert;
```

### 4. Add foreign key references when needed

```ts
import { int, sqliteTable, text } from "drizzle-orm/sqlite-core";
import { projects } from "./schema"; // same file — define the referenced table first

export const userStories = sqliteTable("user_stories", {
  id:          int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  projectId:   int("project_id", { mode: "number" }).notNull().references(() => projects.id),
  title:       text("title").notNull(),
  // ...
});
```

### 5. Sync the schema to the local database

After every schema change, run:

```bash
yarn db:push
```

> **Never** run `db:push` in production. It is a dev-only sync command that drops and recreates tables.

---

## Column Conventions

| Convention | Rule |
|---|---|
| Table name | `snake_case` plural (e.g. `user_stories`, `task_estimates`) |
| Column name | `snake_case` (e.g. `estimated_hours`, `project_id`) |
| Timestamps | Always include `createdAt` + `updatedAt` on every table |
| Enums | Use `text(col, { enum: [...] })` — SQLite has no native enum |
| IDs | Always `int("id", { mode: "number" }).primaryKey({ autoIncrement: true })` |
| Foreign keys | Use `.references(() => table.id)` — always `int`, mode `"number"` |
| Optional text | Omit `.notNull()` — Drizzle infers `string | null` |

---

## Domain Tables for This Project

These are the tables expected by the AI Task Estimator features. Define them all in `db/schema.ts`:

### Projects table
```ts
export const projects = sqliteTable("projects", {
  id:          int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  name:        text("name").notNull(),
  description: text("description"),
  status:      text("status", { enum: ["active", "archived"] }).notNull().default("active"),
  createdAt:   int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:   int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
export type Project    = typeof projects.$inferSelect;
export type NewProject = typeof projects.$inferInsert;
```

### Developers table
```ts
export const developers = sqliteTable("developers", {
  id:            int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  projectId:     int("project_id", { mode: "number" }).notNull().references(() => projects.id),
  name:          text("name").notNull(),
  role:          text("role", { enum: ["frontend", "backend", "fullstack", "devops", "qa"] }).notNull(),
  skillset:      text("skillset").notNull(),    // JSON array string: '["React","Node.js"]'
  capacityHours: real("capacity_hours").notNull().default(80),
  createdAt:     int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:     int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
export type Developer    = typeof developers.$inferSelect;
export type NewDeveloper = typeof developers.$inferInsert;
```

### User Stories table
```ts
export const userStories = sqliteTable("user_stories", {
  id:                 int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  projectId:          int("project_id", { mode: "number" }).notNull().references(() => projects.id),
  title:              text("title").notNull(),
  description:        text("description").notNull(),
  acceptanceCriteria: text("acceptance_criteria"),
  priority:           text("priority", { enum: ["low", "medium", "high", "critical"] }).notNull().default("medium"),
  status:             text("status", { enum: ["pending", "analyzed", "planned"] }).notNull().default("pending"),
  createdAt:          int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:          int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
export type UserStory    = typeof userStories.$inferSelect;
export type NewUserStory = typeof userStories.$inferInsert;
```

### Tasks table (AI-generated sub-tasks)
```ts
export const tasks = sqliteTable("tasks", {
  id:              int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  userStoryId:     int("user_story_id", { mode: "number" }).notNull().references(() => userStories.id),
  developerId:     int("developer_id", { mode: "number" }).references(() => developers.id),
  title:           text("title").notNull(),
  description:     text("description"),
  layer:           text("layer", { enum: ["backend", "frontend", "database", "infrastructure", "testing", "other"] }).notNull().default("other"),
  estimatedHours:  real("estimated_hours").notNull(),
  status:          text("status", { enum: ["unassigned", "assigned", "in_progress", "done"] }).notNull().default("unassigned"),
  sprintNumber:    int("sprint_number", { mode: "number" }),
  isAiGenerated:   int("is_ai_generated", { mode: "number" }).notNull().default(1), // 1 = true, 0 = false (SQLite has no boolean)
  createdAt:       int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:       int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
export type Task    = typeof tasks.$inferSelect;
export type NewTask = typeof tasks.$inferInsert;
```

### Sprints table
```ts
export const sprints = sqliteTable("sprints", {
  id:          int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  projectId:   int("project_id", { mode: "number" }).notNull().references(() => projects.id),
  sprintNumber: int("sprint_number", { mode: "number" }).notNull(),
  startDate:   int("start_date", { mode: "timestamp" }),
  endDate:     int("end_date", { mode: "timestamp" }),
  totalHours:  real("total_hours").notNull().default(0),
  status:      text("status", { enum: ["draft", "active", "completed"] }).notNull().default("draft"),
  createdAt:   int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:   int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
export type Sprint    = typeof sprints.$inferSelect;
export type NewSprint = typeof sprints.$inferInsert;
```

### Analysis Runs table (audit log for AI calls)
```ts
export const analysisRuns = sqliteTable("analysis_runs", {
  id:          int("id", { mode: "number" }).primaryKey({ autoIncrement: true }),
  projectId:   int("project_id", { mode: "number" }).notNull().references(() => projects.id),
  status:      text("status", { enum: ["pending", "running", "completed", "failed"] }).notNull().default("pending"),
  prompt:      text("prompt"),       // stored for debugging
  rawResponse: text("raw_response"), // stored for debugging
  errorMessage: text("error_message"),
  createdAt:   int("created_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
  updatedAt:   int("updated_at", { mode: "timestamp" }).$defaultFn(() => new Date()).notNull(),
});
export type AnalysisRun    = typeof analysisRuns.$inferSelect;
export type NewAnalysisRun = typeof analysisRuns.$inferInsert;
```

---

## Checklist: Adding a New Table

- [ ] Import new column builders from `drizzle-orm/sqlite-core` if needed
- [ ] Define the table with `sqliteTable("snake_case_plural", { ... })`
- [ ] Include `id`, `createdAt`, `updatedAt` on every table
- [ ] Add `.references(() => table.id)` for foreign keys
- [ ] Export `type MyEntity = typeof myTable.$inferSelect`
- [ ] Export `type NewMyEntity = typeof myTable.$inferInsert`
- [ ] Run `yarn db:push` to sync the schema
- [ ] Derive Zod schemas in `lib/schemas/{entity}.schema.ts` (see `create-api-endpoint` skill)
