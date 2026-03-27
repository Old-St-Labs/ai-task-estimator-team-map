---
name: create-form
description: Guide for building interactive forms with client-side Zod validation and API submission. Use this when asked to create a create/edit form, input form, or any interactive form component.
---

# Skill: create-form

> Use this skill whenever you need to build a form that submits data to an API route.

---

## Project Context

| Concern    | Choice |
|------------|--------|
| Forms      | Controlled React forms (`useState`) — no form library |
| Validation | Zod 4 client-side (same schemas derived from Drizzle) |
| Submission | `fetch` to Next.js Route Handlers (`/api/...`) |
| Response   | `{ code: "201", data: {...} }` or `{ code: "400", data: { message, details } }` |
| Directive  | `"use client"` always required for forms |

---

## Standard Form Pattern

### Create form

```tsx
// app/components/developers/developer-form.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { z } from "zod";
import { Button } from "@/app/components/ui/button";

// ── 1. Define or import the Zod schema ──────────────────────────────────────
// Mirror the insert schema from lib/schemas/developer.schema.ts
const developerFormSchema = z.object({
  name:          z.string().min(1, "Name is required"),
  role:          z.enum(["frontend", "backend", "fullstack", "devops", "qa"]),
  skillset:      z.string().min(1, "At least one skill is required"), // comma-separated
  capacityHours: z.coerce.number().min(1).max(160).default(80),
  projectId:     z.coerce.number().int().positive(),
});

type DeveloperFormValues = z.infer<typeof developerFormSchema>;

interface DeveloperFormProps {
  projectId: number;
  /** Provide to switch to edit mode */
  initialValues?: Partial<DeveloperFormValues> & { id?: number };
  onSuccess?: () => void;
}

// ── 2. Component ─────────────────────────────────────────────────────────────
export function DeveloperForm({ projectId, initialValues, onSuccess }: DeveloperFormProps) {
  const router = useRouter();
  const isEditing = Boolean(initialValues?.id);

  const [values, setValues] = useState<DeveloperFormValues>({
    name:          initialValues?.name          ?? "",
    role:          initialValues?.role          ?? "fullstack",
    skillset:      initialValues?.skillset      ?? "",
    capacityHours: initialValues?.capacityHours ?? 80,
    projectId,
  });
  const [errors, setErrors]     = useState<Partial<Record<keyof DeveloperFormValues, string>>>({});
  const [isLoading, setIsLoading] = useState(false);
  const [apiError, setApiError]   = useState<string | null>(null);

  // ── 3. Generic field updater ────────────────────────────────────────────
  function update<K extends keyof DeveloperFormValues>(key: K, value: DeveloperFormValues[K]) {
    setValues((prev) => ({ ...prev, [key]: value }));
    setErrors((prev) => ({ ...prev, [key]: undefined }));
  }

  // ── 4. Submit handler ────────────────────────────────────────────────────
  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setApiError(null);

    // Client-side validation
    const result = developerFormSchema.safeParse(values);
    if (!result.success) {
      const fieldErrors: typeof errors = {};
      for (const issue of result.error.issues) {
        const key = issue.path[0] as keyof DeveloperFormValues;
        fieldErrors[key] = issue.message;
      }
      setErrors(fieldErrors);
      return;
    }

    setIsLoading(true);
    try {
      const url = isEditing
        ? `/api/developers/${initialValues!.id}`
        : "/api/developers";
      const method = isEditing ? "PATCH" : "POST";

      const res = await fetch(url, {
        method,
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(result.data),
      });

      const json = await res.json();

      if (!res.ok) {
        setApiError(json?.data?.message ?? "An error occurred");
        return;
      }

      router.refresh();
      onSuccess?.();
    } catch {
      setApiError("Network error — please try again");
    } finally {
      setIsLoading(false);
    }
  }

  // ── 5. JSX ───────────────────────────────────────────────────────────────
  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      {apiError && (
        <div className="rounded-md bg-red-50 p-3 text-sm text-red-700">{apiError}</div>
      )}

      {/* Name */}
      <div>
        <label htmlFor="name" className="block text-sm font-medium text-gray-700">
          Name
        </label>
        <input
          id="name"
          type="text"
          value={values.name}
          onChange={(e) => update("name", e.target.value)}
          className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 text-sm focus:border-blue-500 focus:outline-none"
        />
        {errors.name && <p className="mt-1 text-xs text-red-600">{errors.name}</p>}
      </div>

      {/* Role */}
      <div>
        <label htmlFor="role" className="block text-sm font-medium text-gray-700">
          Role
        </label>
        <select
          id="role"
          value={values.role}
          onChange={(e) => update("role", e.target.value as DeveloperFormValues["role"])}
          className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
        >
          {["frontend", "backend", "fullstack", "devops", "qa"].map((r) => (
            <option key={r} value={r}>{r}</option>
          ))}
        </select>
        {errors.role && <p className="mt-1 text-xs text-red-600">{errors.role}</p>}
      </div>

      {/* Skillset */}
      <div>
        <label htmlFor="skillset" className="block text-sm font-medium text-gray-700">
          Skills <span className="text-gray-400">(comma-separated, e.g. React, Node.js)</span>
        </label>
        <input
          id="skillset"
          type="text"
          value={values.skillset}
          onChange={(e) => update("skillset", e.target.value)}
          placeholder="React, TypeScript, Node.js"
          className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
        />
        {errors.skillset && <p className="mt-1 text-xs text-red-600">{errors.skillset}</p>}
      </div>

      {/* Capacity */}
      <div>
        <label htmlFor="capacity" className="block text-sm font-medium text-gray-700">
          Capacity (hours/sprint)
        </label>
        <input
          id="capacity"
          type="number"
          min={1}
          max={160}
          value={values.capacityHours}
          onChange={(e) => update("capacityHours", Number(e.target.value))}
          className="mt-1 block w-full rounded-md border border-gray-300 px-3 py-2 text-sm"
        />
        {errors.capacityHours && <p className="mt-1 text-xs text-red-600">{errors.capacityHours}</p>}
      </div>

      <div className="flex justify-end gap-3 pt-2">
        <Button type="submit" isLoading={isLoading}>
          {isEditing ? "Save Changes" : "Add Developer"}
        </Button>
      </div>
    </form>
  );
}
```

---

## Confirmation / Destructive Actions

For delete confirmations, use an inline state toggle rather than `window.confirm`:

```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { Button } from "@/app/components/ui/button";

interface DeleteButtonProps {
  id: number;
  resourceName: string;
  apiPath: string;
}

export function DeleteButton({ id, resourceName, apiPath }: DeleteButtonProps) {
  const router = useRouter();
  const [confirming, setConfirming] = useState(false);
  const [isDeleting, setIsDeleting] = useState(false);

  async function handleDelete() {
    setIsDeleting(true);
    try {
      await fetch(`${apiPath}/${id}`, { method: "DELETE" });
      router.refresh();
    } finally {
      setIsDeleting(false);
      setConfirming(false);
    }
  }

  if (confirming) {
    return (
      <div className="flex items-center gap-2">
        <span className="text-sm text-gray-600">Delete {resourceName}?</span>
        <Button variant="danger" size="sm" isLoading={isDeleting} onClick={handleDelete}>
          Confirm
        </Button>
        <Button variant="ghost" size="sm" onClick={() => setConfirming(false)}>
          Cancel
        </Button>
      </div>
    );
  }

  return (
    <Button variant="danger" size="sm" onClick={() => setConfirming(true)}>
      Delete
    </Button>
  );
}
```

---

## Modal Dialog Pattern

For forms that open in a modal, use a simple state-based overlay:

```tsx
"use client";

import { useState } from "react";
import { Button } from "@/app/components/ui/button";

interface ModalProps {
  title: string;
  trigger: React.ReactNode;
  children: React.ReactNode;
}

export function Modal({ title, trigger, children }: ModalProps) {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <>
      <span onClick={() => setIsOpen(true)}>{trigger}</span>
      {isOpen && (
        <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
          <div className="w-full max-w-md rounded-lg bg-white p-6 shadow-xl">
            <div className="mb-4 flex items-center justify-between">
              <h2 className="text-lg font-semibold">{title}</h2>
              <button
                onClick={() => setIsOpen(false)}
                className="text-gray-400 hover:text-gray-600"
                aria-label="Close"
              >
                ✕
              </button>
            </div>
            {children}
          </div>
        </div>
      )}
    </>
  );
}
```

---

## Response Handling Reference

Always check `res.ok` before reading `json.data`:

```ts
const res  = await fetch("/api/...", { method: "POST", body: JSON.stringify(data), headers: { "Content-Type": "application/json" } });
const json = await res.json();  // shape: { code: string, data: unknown }

if (!res.ok) {
  // json.data = { message: string, details?: unknown }
  setApiError(json?.data?.message ?? "Request failed");
  return;
}
// json.data = the created/updated record
```

---

## Form Validation Rules

- **Always validate client-side** with Zod `safeParse` before `fetch` — never send invalid data to the server
- **Mirror the API's Zod schema** in the form — they must agree on required vs optional fields
- **Show per-field errors** mapped from `result.error.issues`
- **Show a top-level API error** banner for server-returned errors
- **Disable the submit button** while `isLoading` is true
- **Never use `parse()`** — always `safeParse()` to avoid unhandled exceptions

---

## Checklist: New Form

- [ ] `"use client"` directive at the top
- [ ] `useState` for form values, field errors, loading, and API error
- [ ] Zod `safeParse` before any `fetch` call
- [ ] Per-field error messages from `result.error.issues`
- [ ] `fetch` called with `Content-Type: application/json` header
- [ ] Response checked with `res.ok` — API error shown in banner
- [ ] `router.refresh()` called on success to re-render Server Component data
- [ ] Submit button uses `<Button isLoading={isLoading}>`
- [ ] Edit mode: `PATCH` to `/api/{resource}/{id}`; create mode: `POST` to `/api/{resource}`
