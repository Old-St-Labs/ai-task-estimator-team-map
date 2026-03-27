---
name: llm-integration
description: Guide for integrating Google Gemini API into Next.js API routes with structured JSON output and Zod validation. Use this when asked to add any AI/LLM feature, call the Gemini API, or build a structured prompt.
---

# Skill: llm-integration

> Use this skill whenever you need to call the Google Gemini API from a Next.js Route Handler.

---

## Project Context

| Concern    | Choice |
|------------|--------|
| LLM        | Google Gemini via `@google/generative-ai` SDK |
| Output     | Structured JSON — always request `responseMimeType: "application/json"` |
| Parsing    | Zod 4 — validate LLM JSON before touching the database |
| Location   | Server-side only — API routes (`app/api/`) never client components |
| Key        | `GEMINI_API_KEY` environment variable |

---

## Setup

### 1. Install the SDK

```bash
yarn add @google/generative-ai
```

### 2. Add environment variable

Add to `.env.local`:
```
GEMINI_API_KEY=your_key_here
```

Add to `.env.local.example`:
```
GEMINI_API_KEY=        # Google Gemini API key — get from https://aistudio.google.com
```

### 3. Create a shared Gemini client

```ts
// lib/gemini.ts
import { GoogleGenerativeAI } from "@google/generative-ai";

if (!process.env.GEMINI_API_KEY) {
  throw new Error("GEMINI_API_KEY is not set in environment variables");
}

export const gemini = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);

/** Returns a model instance configured for structured JSON output */
export function getJsonModel(modelName = "gemini-1.5-flash") {
  return gemini.getGenerativeModel({
    model: modelName,
    generationConfig: {
      responseMimeType: "application/json",
      temperature: 0.2,         // Low temperature for consistent structured output
      maxOutputTokens: 8192,
    },
  });
}
```

---

## Prompt Engineering Patterns

### Rule: Always request JSON output

Always instruct the model to respond with valid JSON. Include a concrete schema description in the prompt.

```ts
const prompt = `
You are a software project planning assistant. Analyze the following user stories and return a structured JSON response.

## User Stories
${userStoriesText}

## Developer Team
${developersText}

## Required JSON Output Schema
Return ONLY valid JSON matching this exact structure — no markdown, no explanation:
{
  "gaps": [
    { "title": string, "description": string, "storyId": number }
  ],
  "tasks": [
    {
      "userStoryId": number,
      "title": string,
      "description": string,
      "layer": "backend" | "frontend" | "database" | "infrastructure" | "testing" | "other",
      "estimatedHours": number,
      "assignedDeveloperId": number | null,
      "assignmentReason": string
    }
  ]
}

Rules:
- estimatedHours must be between 0.5 and 40
- assignedDeveloperId must be one of the developer IDs provided, or null if no suitable developer exists
- Each user story should produce 3-8 sub-tasks
- Include implicit tasks the user stories don't mention (migrations, error handling, tests)
`;
```

---

## Making an LLM Call

### Complete Route Handler Template

```ts
// app/api/analysis/run/route.ts
import { z } from "zod";
import { getJsonModel } from "@/lib/gemini";
import { db } from "@/db";
import { userStories, developers, tasks, analysisRuns } from "@/db/schema";
import { eq } from "drizzle-orm";
import { badRequest, internalError, ok } from "@/lib/api-response";

// ── 1. Zod schema for the LLM response ──────────────────────────────────────
const llmTaskSchema = z.object({
  userStoryId:        z.number().int(),
  title:              z.string().min(1),
  description:        z.string(),
  layer:              z.enum(["backend", "frontend", "database", "infrastructure", "testing", "other"]),
  estimatedHours:     z.number().min(0.5).max(40),
  assignedDeveloperId: z.number().int().nullable(),
  assignmentReason:   z.string(),
});

const llmGapSchema = z.object({
  title:       z.string(),
  description: z.string(),
  storyId:     z.number().int(),
});

const llmResponseSchema = z.object({
  gaps:  z.array(llmGapSchema),
  tasks: z.array(llmTaskSchema),
});

type LlmResponse = z.infer<typeof llmResponseSchema>;

// ── 2. Request body schema ───────────────────────────────────────────────────
const requestSchema = z.object({
  projectId: z.number().int().positive(),
});

// ── 3. Route Handler ─────────────────────────────────────────────────────────
export async function POST(request: Request) {
  try {
    const body: unknown = await request.json();
    const parsed = requestSchema.safeParse(body);
    if (!parsed.success) return badRequest("Validation failed", parsed.error.flatten());

    const { projectId } = parsed.data;

    // Fetch data to send to the LLM
    const stories    = await db.select().from(userStories).where(eq(userStories.projectId, projectId));
    const devRoster  = await db.select().from(developers).where(eq(developers.projectId, projectId));

    if (stories.length === 0) return badRequest("No user stories found for this project");

    // Build the prompt
    const prompt = buildAnalysisPrompt(stories, devRoster);

    // Create an audit record before calling LLM
    const [analysisRun] = await db.insert(analysisRuns).values({
      projectId,
      status: "running",
      prompt,
    }).returning();

    // Call Gemini
    const model = getJsonModel();
    const result = await model.generateContent(prompt);
    const rawText = result.response.text();

    // Parse and validate
    let llmData: LlmResponse;
    try {
      const jsonParsed = JSON.parse(rawText);
      const zodResult  = llmResponseSchema.safeParse(jsonParsed);
      if (!zodResult.success) {
        await markAnalysisFailed(analysisRun.id, rawText, "LLM response did not match expected schema");
        return internalError("AI returned an unexpected response format");
      }
      llmData = zodResult.data;
    } catch {
      await markAnalysisFailed(analysisRun.id, rawText, "Failed to parse LLM JSON response");
      return internalError("AI returned malformed JSON");
    }

    // Persist tasks
    const newTasks = await db.insert(tasks).values(
      llmData.tasks.map((t) => ({
        userStoryId:    t.userStoryId,
        developerId:    t.assignedDeveloperId ?? undefined,
        title:          t.title,
        description:    t.description,
        layer:          t.layer,
        estimatedHours: t.estimatedHours,
        isAiGenerated:  1,
        status:         "unassigned" as const,
      }))
    ).returning();

    // Mark analysis complete
    await db.update(analysisRuns)
      .set({ status: "completed", rawResponse: rawText, updatedAt: new Date() })
      .where(eq(analysisRuns.id, analysisRun.id));

    return ok({ tasks: newTasks, gaps: llmData.gaps, analysisRunId: analysisRun.id });
  } catch (err) {
    console.error("Analysis run failed:", err);
    return internalError("Analysis failed — please try again");
  }
}

// ── Helpers ───────────────────────────────────────────────────────────────────
function buildAnalysisPrompt(
  stories: Array<{ id: number; title: string; description: string; acceptanceCriteria: string | null }>,
  devRoster: Array<{ id: number; name: string; role: string; skillset: string }>
): string {
  const storiesText = stories
    .map((s) => `ID: ${s.id}\nTitle: ${s.title}\nDescription: ${s.description}\nAcceptance Criteria: ${s.acceptanceCriteria ?? "none"}`)
    .join("\n\n");

  const devsText = devRoster
    .map((d) => `ID: ${d.id}\nName: ${d.name}\nRole: ${d.role}\nSkills: ${d.skillset}`)
    .join("\n\n");

  return `You are a software project planning assistant...

## User Stories
${storiesText}

## Developer Team
${devsText}

Return ONLY valid JSON:
{ "gaps": [...], "tasks": [...] }`;
}

async function markAnalysisFailed(id: number, rawResponse: string, error: string) {
  await db.update(analysisRuns)
    .set({ status: "failed", rawResponse, errorMessage: error, updatedAt: new Date() })
    .where(eq(analysisRuns.id, id));
}
```

---

## Error Handling for LLM Calls

Always handle these failure modes:

| Failure | Handling |
|---|---|
| `GEMINI_API_KEY` missing | Throw at module load time in `lib/gemini.ts` |
| Gemini API unreachable (network) | Caught by outer try/catch → `internalError()` |
| Gemini returns non-JSON | `JSON.parse` fails → caught → `internalError()` |
| JSON doesn't match schema | Zod `safeParse` fails → `internalError()` with details |
| Partial response (truncated) | Detected by Zod failing → retry or `internalError()` |

**Never expose raw LLM responses or API keys to the client.**

---

## Prompt Engineering Best Practices

1. **Start with a clear role**: `"You are a software project planning assistant."`
2. **Provide structured input**: Format user stories and developers as labelled sections
3. **Show the exact JSON schema**: Include field names, types, and allowed enum values
4. **Add constraints**: `"estimatedHours must be between 0.5 and 40"` prevents hallucinations
5. **End with**: `"Return ONLY valid JSON — no markdown, no explanation, no code blocks"`
6. **Use low temperature** (`0.2`) for consistent, structured output
7. **Always Zod-parse the output** — never trust raw LLM JSON directly

---

## Environment Variables

| Variable | Purpose |
|---|---|
| `GEMINI_API_KEY` | Google Gemini API key |

**Required in:**
- `.env.local` (development, gitignored)
- `.env.local.example` (committed, blank value)
- Vercel/deployment environment variables (production)

---

## Checklist: New LLM Feature

- [ ] `@google/generative-ai` installed (`yarn add @google/generative-ai`)
- [ ] `GEMINI_API_KEY` in `.env.local` and `.env.local.example`
- [ ] `lib/gemini.ts` client singleton created
- [ ] Zod schema defined for the expected LLM response shape
- [ ] Prompt ends with "Return ONLY valid JSON"
- [ ] `responseMimeType: "application/json"` set on the model
- [ ] `JSON.parse` wrapped in try/catch
- [ ] Zod `safeParse` applied to parsed JSON
- [ ] Both parse failures return `internalError()` (never raw error to client)
- [ ] Analysis audit record created before calling LLM
- [ ] LLM call happens in an API route — never in a Client Component or Server Component
