# Prompt 21 — Typed Frontend API Client

This README explains, step by step, what was done to fulfill the latest prompt:

> Create `apps/frontend/src/api/client.ts`. Implement a generic `requestJson<T>(path, options?)`
> wrapper around `fetch`. On non-2xx responses, throw a typed `ClientApiError` containing
> `status`, `message`, and optional `details`. Export typed functions for the work order API,
> using only native `fetch` and types imported from `@equipment-hub/contract`.

## Steps followed

1. **Inspected the shared contract package first.**
   Read [packages/contract/src/index.ts](packages/contract/src/index.ts) and
   [packages/contract/src/types.gen.ts](packages/contract/src/types.gen.ts) to see exactly which
   types are exported (`Asset`, `Technician`, `WorkOrder`, `NewWorkOrder`, `TransitionCommand`,
   `AssignmentCommand`, `DashboardSummary`, `ApiError`, `WorkOrderState`, `WorkOrderAction`,
   `Priority`). The client must be built only from these — no hand-rolled duplicate types.

2. **Checked how the frontend talks to the backend.**
   Read [apps/frontend/vite.config.ts](apps/frontend/vite.config.ts) and confirmed the dev server
   proxies `/api` to the Fastify backend (`http://127.0.0.1:3001`). This means the client can call
   relative paths like `/api/work-orders` directly with `fetch`, with no base URL configuration
   needed.

3. **Confirmed frontend dependencies.**
   Checked [apps/frontend/package.json](apps/frontend/package.json) to verify
   `@equipment-hub/contract` is already a workspace dependency, so it can be imported with a plain
   `import type { ... } from "@equipment-hub/contract"`.

4. **Wrote `apps/frontend/src/api/client.ts`** containing:
   - `ClientApiError` — an `Error` subclass carrying `status: number`, `message: string`, and an
     optional `details?: Record<string, unknown>`, mirroring the shape of the contract's
     `ApiError` schema.
   - `requestJson<T>(path, options?)` — a private generic wrapper around `fetch` that:
     - Sends `Content-Type: application/json` by default (overridable via `options`).
     - On a non-2xx response, tries to parse the JSON body as `ApiError` to get a real message
       and details, falling back to `response.statusText` if the body isn't valid JSON, then
       throws `ClientApiError`.
     - Returns `undefined` for `204 No Content`, otherwise parses and returns the JSON body as
       `T`.
   - Typed, exported functions built on top of `requestJson`, one per backend endpoint:
     - `listAssets()`
     - `listTechnicians()`
     - `listWorkOrders(filters?: { state?: WorkOrderState; priority?: Priority })` — builds a
       query string only from the filters that are provided.
     - `getWorkOrder(id: string)`
     - `createWorkOrder(input: NewWorkOrder)`
     - `assignTechnician(id: string, technicianId: string)`
     - `transitionWorkOrder(id: string, action: WorkOrderAction)`
     - `getDashboardSummary()`

5. **Attempted a type-check** with `npx tsc --noEmit` inside `apps/frontend`, but found that
   `node_modules` isn't installed yet in this workspace (no local `typescript` binary). Rather
   than installing packages without being asked, this was left for the student/instructor to run
   after `npm install`, since the file was written directly against the verified contract types
   and endpoint shapes from `types.gen.ts`.

## Running the app (Windows Terminal)

The root `dev` script is just a placeholder, so the backend and frontend dev servers must be
started separately. This is an npm workspaces monorepo, so run everything from the repository
root using the `--workspace` flag — open two tabs/panes in Windows Terminal:

**1. Install dependencies once, from the repo root:**

```powershell
npm install
```

**2. Tab/pane 1 — start the backend (Fastify, on port 3001):**

```powershell
npm run dev --workspace apps/backend
```

**3. Tab/pane 2 — start the frontend (Vite, on port 5173):**

```powershell
npm run dev --workspace apps/frontend
```

Then open `http://localhost:5173` in your browser. The Vite dev server proxies any `/api/*`
request to the backend at `http://127.0.0.1:3001` (see step 2 above), so the API client in
`apps/frontend/src/api/client.ts` works out of the box with no extra configuration.

## Try it yourself

After installing dependencies at the repo root (e.g. `npm install`), you can verify the new file
compiles cleanly:

```bash
cd apps/frontend
npx tsc --noEmit -p tsconfig.json
```

And you can exercise it once both the backend (`apps/backend`) and frontend dev servers are
running, by importing functions from `src/api/client.ts` in a component and calling them (e.g.
`listAssets()`, `getDashboardSummary()`).
