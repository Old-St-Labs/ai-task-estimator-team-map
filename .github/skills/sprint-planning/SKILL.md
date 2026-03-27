---
name: sprint-planning
description: Guide for implementing the sprint planning engine — packing estimated tasks into 2-week sprints while respecting developer capacity. Use this when asked to implement or extend sprint generation, sprint distribution, or capacity planning.
---

# Skill: sprint-planning

> Use this skill when implementing the "Generate Sprints" feature — the algorithm that takes AI-estimated and assigned tasks and organizes them into 2-week sprint cycles.

---

## Feature Overview (from PRD)

1. User clicks **"Generate Sprints"** after reviewing AI-generated tasks
2. The backend algorithm:
   - Groups tasks by developer
   - Respects per-developer capacity (default: 80 hours / sprint)
   - Orders tasks by layer dependency (database → backend → frontend → testing)
   - Packs tasks into 2-week sprint buckets
3. Sprint records are created in the `sprints` table
4. Tasks are updated with their `sprintNumber`

---

## Sprint Planning Algorithm

### Core Rules

| Rule | Value |
|---|---|
| Sprint duration | 14 days (2 weeks) |
| Default capacity per developer per sprint | 80 hours |
| Actual effective capacity (meetings, reviews) | 70–80% of stated capacity |
| Layer ordering priority | `database` → `infrastructure` → `backend` → `frontend` → `testing` → `other` |
| Unassigned tasks | Placed in sprint 1 as a pool (flag for review) |

### Algorithm Steps

1. Sort tasks by layer priority (lower = earlier)
2. Group tasks by `developerId`
3. For each developer, bin-pack tasks into sprints (greedy first-fit)
4. For unassigned tasks, distribute across sprints without exceeding global unassigned budget
5. Create `sprint` records for each unique sprint number
6. Update `tasks.sprintNumber` in bulk

---

## API Route: `POST /api/projects/[id]/sprints/generate`

```ts
// app/api/projects/[id]/sprints/generate/route.ts
import { eq, inArray } from "drizzle-orm";
import { db } from "@/db";
import { tasks, sprints, developers, projects } from "@/db/schema";
import { badRequest, internalError, notFound, ok } from "@/lib/api-response";

const LAYER_ORDER: Record<string, number> = {
  database:       1,
  infrastructure: 2,
  backend:        3,
  frontend:       4,
  testing:        5,
  other:          6,
};

const SPRINT_DAYS     = 14;
const DEFAULT_CAPACITY = 80; // hours per developer per sprint

type Params = { params: Promise<{ id: string }> };

export async function POST(_req: Request, { params }: Params) {
  try {
    const { id } = await params;
    const projectId = Number(id);

    // 1. Validate project exists
    const [project] = await db.select().from(projects).where(eq(projects.id, projectId));
    if (!project) return notFound(`Project ${projectId} not found`);

    // 2. Load assigned, estimated tasks for this project
    const storyTaskRows = await db.query.tasks.findMany({
      where: (t, { eq: eqFn, inArray: inFn, sql }) =>
        sql`${t.userStoryId} IN (
          SELECT id FROM user_stories WHERE project_id = ${projectId}
        )`,
    });

    if (storyTaskRows.length === 0) {
      return badRequest("No tasks found. Run 'Analyze & Estimate' first.");
    }

    // 3. Load developer capacity map
    const devRows = await db.select().from(developers).where(eq(developers.projectId, projectId));
    const capacityMap = new Map(devRows.map((d) => [d.id, d.capacityHours ?? DEFAULT_CAPACITY]));

    // 4. Sort tasks by layer priority
    const sortedTasks = [...storyTaskRows].sort(
      (a, b) => (LAYER_ORDER[a.layer] ?? 6) - (LAYER_ORDER[b.layer] ?? 6)
    );

    // 5. Bin-pack tasks into sprints per developer
    // Map: developerId (or "unassigned") → sprint buckets
    const devSprints = new Map<string | number, number[]>();
    // devSprints[devId][sprintIndex] = total hours in that sprint

    const taskSprintMap = new Map<number, number>(); // taskId → sprintNumber (1-based)

    for (const task of sortedTasks) {
      const key      = task.developerId ?? "unassigned";
      const capacity = task.developerId ? (capacityMap.get(task.developerId) ?? DEFAULT_CAPACITY) : DEFAULT_CAPACITY;

      if (!devSprints.has(key)) devSprints.set(key, [0]); // Sprint 0 = sprint #1

      const buckets = devSprints.get(key)!;
      const hours   = task.estimatedHours;

      // Find first sprint with enough remaining capacity
      let placed = false;
      for (let i = 0; i < buckets.length; i++) {
        if (buckets[i] + hours <= capacity) {
          buckets[i] += hours;
          taskSprintMap.set(task.id, i + 1); // 1-based sprint number
          placed = true;
          break;
        }
      }

      if (!placed) {
        // Open a new sprint
        buckets.push(hours);
        taskSprintMap.set(task.id, buckets.length);
      }
    }

    // 6. Determine total number of sprints across all developers
    let maxSprint = 0;
    for (const buckets of devSprints.values()) {
      maxSprint = Math.max(maxSprint, buckets.length);
    }

    // 7. Delete existing draft sprints for this project
    await db.delete(sprints).where(eq(sprints.projectId, projectId));

    // 8. Create sprint records
    const sprintStartDate = new Date();
    const createdSprints = [];
    for (let i = 1; i <= maxSprint; i++) {
      const startDate = new Date(sprintStartDate.getTime() + (i - 1) * SPRINT_DAYS * 86_400_000);
      const endDate   = new Date(startDate.getTime() + SPRINT_DAYS * 86_400_000);

      const [sprint] = await db.insert(sprints).values({
        projectId,
        sprintNumber: i,
        startDate,
        endDate,
        status: "draft",
        totalHours: [...taskSprintMap.entries()]
          .filter(([, s]) => s === i)
          .reduce((sum, [taskId]) => {
            const t = storyTaskRows.find((r) => r.id === taskId);
            return sum + (t?.estimatedHours ?? 0);
          }, 0),
      }).returning();

      createdSprints.push(sprint);
    }

    // 9. Bulk-update tasks with their sprintNumber
    for (const [taskId, sprintNum] of taskSprintMap.entries()) {
      await db.update(tasks)
        .set({ sprintNumber: sprintNum, updatedAt: new Date() })
        .where(eq(tasks.id, taskId));
    }

    return ok({
      sprintCount: maxSprint,
      sprints:     createdSprints,
      taskCount:   taskSprintMap.size,
    });
  } catch (err) {
    console.error("[sprints/generate] error:", err);
    return internalError();
  }
}
```

---

## "Generate Sprints" Button Component

```tsx
// app/components/sprints/generate-sprints-button.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/app/components/ui/button";

interface GenerateSprintsButtonProps {
  projectId: number;
}

export function GenerateSprintsButton({ projectId }: GenerateSprintsButtonProps) {
  const router = useRouter();
  const [isGenerating, setIsGenerating] = useState(false);
  const [error, setError]               = useState<string | null>(null);
  const [result, setResult]             = useState<{ sprintCount: number; taskCount: number } | null>(null);

  async function handleGenerate() {
    setIsGenerating(true);
    setError(null);
    setResult(null);
    try {
      const res  = await fetch(`/api/projects/${projectId}/sprints/generate`, { method: "POST" });
      const json = await res.json();

      if (!res.ok) {
        setError(json?.data?.message ?? "Sprint generation failed");
        return;
      }
      setResult(json.data);
      router.refresh();
    } catch {
      setError("Network error — please try again");
    } finally {
      setIsGenerating(false);
    }
  }

  return (
    <div>
      {error  && <p className="mb-2 text-sm text-red-600">{error}</p>}
      {result && (
        <p className="mb-2 text-sm text-green-700">
          ✅ Generated {result.sprintCount} sprints for {result.taskCount} tasks
        </p>
      )}
      <Button onClick={handleGenerate} isLoading={isGenerating} variant="secondary">
        📅 Generate Sprints
      </Button>
    </div>
  );
}
```

---

## Capacity Summary Component

Show per-developer sprint load summary:

```tsx
// app/components/sprints/capacity-summary.tsx
import type { Developer, Task } from "@/db/schema";

interface CapacitySummaryProps {
  developers: Developer[];
  tasks:      Task[];
  sprintCount: number;
}

export function CapacitySummary({ developers, tasks, sprintCount }: CapacitySummaryProps) {
  return (
    <div className="overflow-x-auto">
      <table className="w-full text-sm">
        <thead>
          <tr className="border-b bg-gray-50">
            <th className="px-4 py-2 text-left">Developer</th>
            {Array.from({ length: sprintCount }, (_, i) => (
              <th key={i} className="px-4 py-2 text-center">Sprint {i + 1}</th>
            ))}
          </tr>
        </thead>
        <tbody>
          {developers.map((dev) => (
            <tr key={dev.id} className="border-b">
              <td className="px-4 py-2 font-medium">{dev.name}</td>
              {Array.from({ length: sprintCount }, (_, i) => {
                const sprintNum  = i + 1;
                const sprintHours = tasks
                  .filter((t) => t.developerId === dev.id && t.sprintNumber === sprintNum)
                  .reduce((sum, t) => sum + t.estimatedHours, 0);
                const pct = Math.round((sprintHours / dev.capacityHours) * 100);
                const color = pct > 90 ? "text-red-600" : pct > 70 ? "text-yellow-600" : "text-green-600";
                return (
                  <td key={i} className={`px-4 py-2 text-center ${color}`}>
                    {sprintHours.toFixed(1)}h
                    <span className="ml-1 text-xs text-gray-400">({pct}%)</span>
                  </td>
                );
              })}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

---

## Sprint Planning Rules

| Rule | Implementation |
|---|---|
| Never over-allocate a developer | Greedy bin-packing respects `capacityHours` |
| Backend tasks before frontend | `LAYER_ORDER` sort before packing |
| Unassigned tasks | Keyed as `"unassigned"` — placed in sprints separately |
| Re-generating sprints | Always delete existing draft sprints first |
| Sprint dates | Calculated from today forward, 14 days each |

---

## Checklist: Sprint Planning Feature

- [ ] `sprints` table in `db/schema.ts` with `sprintNumber`, `startDate`, `endDate`, `totalHours`, `status`
- [ ] `tasks` table has `sprintNumber` column (nullable int)
- [ ] `POST /api/projects/[id]/sprints/generate` route implemented
- [ ] Tasks sorted by `LAYER_ORDER` before packing
- [ ] Per-developer capacity respected using `capacityHours` from `developers` table
- [ ] Existing draft sprints deleted before regenerating
- [ ] `tasks.sprintNumber` updated for every task
- [ ] `GenerateSprintsButton` handles loading + error + success
- [ ] `CapacitySummary` shows per-developer utilization per sprint
- [ ] Over-allocated developers (>90%) shown in red
