---
name: ai-task-analysis
description: Guide for implementing the AI-powered task analysis and estimation pipeline — including the Analyze & Estimate button, prompt construction, result parsing, and storing AI-generated tasks. Use this when asked to implement or extend the core AI analysis feature.
---

# Skill: ai-task-analysis

> Use this skill when implementing the "Analyze & Estimate" feature — the core AI pipeline that takes user stories + developer roster and produces gap analysis, task breakdowns, assignments, and hour estimates.

**Read the `llm-integration` skill first** — this skill builds on top of it.

---

## Feature Overview (from PRD)

1. User clicks **"Analyze & Estimate"** on a project
2. Backend collects all user stories + developer roster for the project
3. LLM performs:
   - **Gap analysis** — identifies implicit/missing requirements
   - **Task breakdown** — decomposes stories into granular sub-tasks
   - **Smart assignment** — matches tasks to developers by skill
   - **Time estimation** — estimates hours per task
4. Results are stored in the `tasks` table with `isAiGenerated = 1`
5. User is presented with the generated task list for review

---

## Database Tables Involved

- `user_stories` — input (read only)
- `developers` — input for skill-matching (read only)
- `tasks` — output (AI-generated rows inserted here)
- `analysis_runs` — audit log for every AI call

See `add-drizzle-schema` skill for the full schema definitions.

---

## API Route: `POST /api/projects/[id]/analyze`

### File: `app/api/projects/[id]/analyze/route.ts`

```ts
import { z } from "zod";
import { eq } from "drizzle-orm";
import { db } from "@/db";
import { userStories, developers, tasks, analysisRuns, projects } from "@/db/schema";
import { getJsonModel } from "@/lib/gemini";
import { badRequest, internalError, notFound, ok } from "@/lib/api-response";

// ── LLM Response Zod Schema ──────────────────────────────────────────────────
const llmTaskSchema = z.object({
  userStoryId:         z.number().int(),
  title:               z.string().min(1),
  description:         z.string(),
  layer:               z.enum(["backend", "frontend", "database", "infrastructure", "testing", "other"]),
  estimatedHours:      z.number().min(0.5).max(40),
  assignedDeveloperId: z.number().int().nullable(),
  assignmentReason:    z.string(),
});

const llmGapSchema = z.object({
  title:       z.string().min(1),
  description: z.string(),
  storyId:     z.number().int(),
});

const llmResponseSchema = z.object({
  gaps:  z.array(llmGapSchema),
  tasks: z.array(llmTaskSchema).min(1),
});

type Params = { params: Promise<{ id: string }> };

export async function POST(_req: Request, { params }: Params) {
  try {
    const { id } = await params;
    const projectId = Number(id);

    // 1. Validate project exists
    const [project] = await db.select().from(projects).where(eq(projects.id, projectId));
    if (!project) return notFound(`Project ${projectId} not found`);

    // 2. Load stories and developers
    const stories   = await db.select().from(userStories).where(eq(userStories.projectId, projectId));
    const devRoster = await db.select().from(developers).where(eq(developers.projectId, projectId));

    if (stories.length === 0)   return badRequest("Add at least one user story before running analysis");
    if (devRoster.length === 0) return badRequest("Add at least one developer before running analysis");

    // 3. Build prompt
    const prompt = buildAnalysisPrompt(project.name, stories, devRoster);

    // 4. Create audit record
    const [run] = await db.insert(analysisRuns).values({ projectId, status: "running", prompt }).returning();

    // 5. Call Gemini
    let rawText = "";
    try {
      const model  = getJsonModel();
      const result = await model.generateContent(prompt);
      rawText      = result.response.text();
    } catch (apiErr) {
      await failRun(run.id, rawText, String(apiErr));
      return internalError("Gemini API call failed — check your API key and quota");
    }

    // 6. Parse and validate LLM response
    let llmData: z.infer<typeof llmResponseSchema>;
    try {
      const json      = JSON.parse(rawText);
      const zodResult = llmResponseSchema.safeParse(json);
      if (!zodResult.success) {
        await failRun(run.id, rawText, zodResult.error.message);
        return internalError("AI returned an unexpected response format");
      }
      llmData = zodResult.data;
    } catch {
      await failRun(run.id, rawText, "JSON parse error");
      return internalError("AI returned malformed JSON");
    }

    // 7. Clear previous AI-generated tasks for this project
    const storyIds = stories.map((s) => s.id);
    for (const storyId of storyIds) {
      await db.delete(tasks)
        .where(eq(tasks.userStoryId, storyId));
        // Only delete AI-generated tasks; manually created ones would need: .where(and(eq(...), eq(tasks.isAiGenerated, 1)))
    }

    // 8. Insert new tasks
    const insertedTasks = await db.insert(tasks).values(
      llmData.tasks.map((t) => ({
        userStoryId:    t.userStoryId,
        developerId:    t.assignedDeveloperId ?? undefined,
        title:          t.title,
        description:    t.description,
        layer:          t.layer,
        estimatedHours: t.estimatedHours,
        status:         "unassigned" as const,
        isAiGenerated:  1,
      }))
    ).returning();

    // 9. Mark stories as analyzed
    for (const s of stories) {
      await db.update(userStories).set({ status: "analyzed", updatedAt: new Date() }).where(eq(userStories.id, s.id));
    }

    // 10. Complete the audit record
    await db.update(analysisRuns)
      .set({ status: "completed", rawResponse: rawText, updatedAt: new Date() })
      .where(eq(analysisRuns.id, run.id));

    return ok({
      analysisRunId: run.id,
      tasks:         insertedTasks,
      gaps:          llmData.gaps,
      taskCount:     insertedTasks.length,
      gapCount:      llmData.gaps.length,
    });
  } catch (err) {
    console.error("[analyze] unexpected error:", err);
    return internalError();
  }
}

// ── Prompt Builder ─────────────────────────────────────────────────────────────
function buildAnalysisPrompt(
  projectName: string,
  stories: Array<{ id: number; title: string; description: string; acceptanceCriteria: string | null; priority: string }>,
  devRoster: Array<{ id: number; name: string; role: string; skillset: string; capacityHours: number }>
): string {
  const storiesText = stories.map((s) =>
    `Story ID: ${s.id}
Title: ${s.title}
Priority: ${s.priority}
Description: ${s.description}
Acceptance Criteria: ${s.acceptanceCriteria ?? "not specified"}`
  ).join("\n\n---\n\n");

  const devsText = devRoster.map((d) =>
    `Developer ID: ${d.id}
Name: ${d.name}
Role: ${d.role}
Skills: ${d.skillset}
Capacity: ${d.capacityHours}h per sprint`
  ).join("\n\n");

  return `You are an expert software engineering project planner for project "${projectName}".

Your job is to:
1. Identify implicit requirements and missing tasks (gap analysis)
2. Break each user story into granular, actionable sub-tasks
3. Assign each sub-task to the most appropriate developer based on their skills
4. Estimate realistic effort in hours

## User Stories
${storiesText}

## Development Team
${devsText}

## Instructions

For tasks:
- Break each story into 3–8 sub-tasks covering all layers (backend API, frontend UI, database migrations, tests, etc.)
- Include implicit tasks not mentioned in the story (e.g., input validation, error states, database indexes, email triggers)
- Each task should take between 0.5 and 16 hours
- Assign developers based on skill match — a React story should go to a frontend/fullstack developer
- If no developer has the right skill, set assignedDeveloperId to null

For gaps:
- List any business requirements, edge cases, or technical concerns that are implied but not stated
- Examples: "Password reset flow not specified", "Email verification required", "Rate limiting on auth endpoints"

## Required JSON Output

Return ONLY valid JSON with no markdown, no code blocks, no explanation:

{
  "gaps": [
    {
      "storyId": <number — must be a valid Story ID from above>,
      "title": "<short gap title>",
      "description": "<explanation of the missing requirement>"
    }
  ],
  "tasks": [
    {
      "userStoryId": <number — must be a valid Story ID from above>,
      "title": "<specific actionable task title>",
      "description": "<what needs to be implemented and why>",
      "layer": "<backend|frontend|database|infrastructure|testing|other>",
      "estimatedHours": <number between 0.5 and 16>,
      "assignedDeveloperId": <number from developer list, or null>,
      "assignmentReason": "<why this developer was chosen>"
    }
  ]
}`;
}

async function failRun(id: number, rawResponse: string, error: string) {
  await db.update(analysisRuns)
    .set({ status: "failed", rawResponse, errorMessage: error, updatedAt: new Date() })
    .where(eq(analysisRuns.id, id));
}
```

---

## Trigger UI: "Analyze & Estimate" Button

```tsx
// app/components/projects/analyze-button.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/app/components/ui/button";

interface AnalyzeButtonProps {
  projectId: number;
  onComplete?: (result: { taskCount: number; gapCount: number }) => void;
}

export function AnalyzeButton({ projectId, onComplete }: AnalyzeButtonProps) {
  const router = useRouter();
  const [isRunning, setIsRunning] = useState(false);
  const [error, setError]         = useState<string | null>(null);

  async function handleAnalyze() {
    setIsRunning(true);
    setError(null);
    try {
      const res  = await fetch(`/api/projects/${projectId}/analyze`, { method: "POST" });
      const json = await res.json();

      if (!res.ok) {
        setError(json?.data?.message ?? "Analysis failed");
        return;
      }

      onComplete?.(json.data);
      router.refresh();
    } catch {
      setError("Network error — please try again");
    } finally {
      setIsRunning(false);
    }
  }

  return (
    <div>
      {error && <p className="mb-2 text-sm text-red-600">{error}</p>}
      <Button
        onClick={handleAnalyze}
        isLoading={isRunning}
        disabled={isRunning}
        size="lg"
      >
        {isRunning ? "Analyzing…" : "✨ Analyze & Estimate"}
      </Button>
      {isRunning && (
        <p className="mt-2 text-xs text-gray-500">
          This may take 20–60 seconds depending on the number of stories.
        </p>
      )}
    </div>
  );
}
```

---

## Gap Analysis Display Component

```tsx
// app/components/analysis/gap-list.tsx
import { Badge } from "@/app/components/ui/badge";

interface Gap {
  storyId: number;
  title:   string;
  description: string;
}

export function GapList({ gaps }: { gaps: Gap[] }) {
  if (gaps.length === 0) return null;

  return (
    <div className="rounded-lg border border-yellow-200 bg-yellow-50 p-4">
      <h3 className="flex items-center gap-2 font-semibold text-yellow-800">
        <span>⚠️</span>
        Identified Gaps ({gaps.length})
      </h3>
      <ul className="mt-3 space-y-3">
        {gaps.map((gap, i) => (
          <li key={i} className="rounded bg-white p-3 shadow-sm">
            <div className="flex items-center gap-2">
              <Badge variant="warning">Story #{gap.storyId}</Badge>
              <span className="font-medium text-sm">{gap.title}</span>
            </div>
            <p className="mt-1 text-sm text-gray-600">{gap.description}</p>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

---

## Checklist: AI Task Analysis Feature

- [ ] `llm-integration` skill steps completed (Gemini client, env var)
- [ ] All required tables exist: `user_stories`, `developers`, `tasks`, `analysis_runs`
- [ ] `POST /api/projects/[id]/analyze` route created
- [ ] Prompt clearly requests the exact JSON schema with Story IDs from input
- [ ] Zod validates LLM response before any DB write
- [ ] Both JSON parse failure and Zod validation failure return `internalError()`
- [ ] Analysis audit record created before LLM call, updated after
- [ ] Previous AI-generated tasks cleared before inserting new ones
- [ ] Stories updated to `status: "analyzed"` after successful run
- [ ] `AnalyzeButton` client component handles loading, error, and success states
- [ ] `router.refresh()` called on success to re-render task list
- [ ] Gap list component rendered when `gaps.length > 0`
