---
name: learn-visualise
description: Trace a code flow across one or more repos and render it as an interactive, scrollable, dark-mode diagram. Use to understand or re-learn how a feature or data flow works end to end.
---

# learn-visualise

Flow to trace: $ARGUMENTS

Turn a natural-language question about a code flow into an interactive diagram. The renderer is fixed (`template.html`); your job is to **trace the real flow through the actual code** and produce accurate JSON. Do not invent the flow — read the files.

## What you produce

A single self-contained HTML file at `.learn-visualise/out/<slug>.html`, where `<slug>` is derived from the flow (e.g. `auth-flow.html`). It is `template.html` with the `FLOW` object replaced by your traced JSON. Reference bundled files with `${CLAUDE_SKILL_DIR}/template.html` and `${CLAUDE_SKILL_DIR}/schema.json` so paths resolve regardless of where the skill is installed.

## Process

### 1. Resolve the project (stack + repo layout)

Work out two things before tracing: **where the code lives** and **which systems it uses**. Resolve in this order:

1. **Read the config block if present.** Look for a `PROJECT_CONFIG` block (an HTML comment, see below) in this SKILL.md or in a `.learn-visualise/config.yml` at the repo root. If it exists, it is authoritative — use its repo paths and stack entries and skip detection for anything it specifies.

2. **Otherwise auto-detect** from manifest files. This is usually enough; don't ask the user unless detection is genuinely ambiguous. Read whatever exists:
   - `package.json` — dependencies reveal the framework (`fastify`, `express`, `next`, `@nestjs/*`, `react`, `react-native`, `expo`, `vue`, `svelte`, `@angular/*`), the ORM (`prisma`, `drizzle-orm`, `typeorm`, `mongoose`), and services (`stripe`, `@aws-sdk/*`, `firebase`, `@supabase/*`).
   - `prisma/schema.prisma` — the `provider` line gives the DB (`postgresql`, `mysql`, `sqlite`, `mongodb`).
   - `app.json` / `app.config.{js,ts}` — Expo / React Native mobile app.
   - `requirements.txt` / `pyproject.toml` — `django`, `flask`, `fastapi`.
   - `go.mod`, `Gemfile`, `composer.json`, `pom.xml`, `*.csproj` — Go, Rails, Laravel, Spring, .NET.
   - `docker-compose.yml` — often surfaces `redis`, `postgres`, `minio`/`s3`, message queues.

3. **Find the repos.** A flow may span sibling folders (e.g. `api/`, `mobile/`, `web/`) or live in one. Identify the relevant ones from the config block, or by locating the manifest files above. Do not assume specific folder names — discover them.

### 2. Map detected tech to system keys

Each step's `system` field must be a key in the `SYSTEMS` registry in `template.html` (open it to see the full list — it covers most common frameworks, databases, BaaS, storage, and services). Map what you detected to the closest key:
- Fastify → `fastify`, Express → `express`, NestJS → `nest`, Django → `django`, etc.
- Postgres → `postgres`, Prisma calls → `prisma` (or just `postgres` if you don't want to split ORM from DB).
- Firebase Auth → `firebase`, Supabase → `supabase`, Clerk → `clerk`.
- S3 → `s3`, Cloudflare R2 → `r2`, GCS → `gcs`.
- A generic layer with no specific match → `mobile`, `web`, `backend`, or `external`.

**If a system genuinely isn't in the registry**, append a new entry to the `SYSTEMS` object in your output copy of `template.html` before rendering: `key: { label: "...", color: "#hex", icon: "ti-..." }`. Use a Tabler icon name (https://tabler.io/icons) and a color distinct from neighbours. Don't force a bad fit onto an unrelated key.

### 3. Trace it for real — this is the hard part, do not skip
Start at the entry point (a screen tap, an API route, a cron job) and follow the actual call chain:
- `grep`/search for the entry point across the resolved repos.
- **Open each file and read the relevant function body** before writing a step for it. Do not infer behaviour from a function name alone.
- Follow imports and calls across file and repo boundaries. When the frontend calls the backend, find the route handler. When the handler calls a service, open the service. When it queries the DB, note the model/query.
- Note where data crosses a system boundary (device → auth provider, app → backend, backend → cache/DB/storage) — each crossing is usually a new step.
- Capture real failure branches you see in the code (a thrown 401, an early return, a cache miss) — these become `branch` objects.

If you genuinely cannot find part of the chain, say so in the step `detail` rather than fabricating. An honest "this calls an external service we don't have source for" is correct; a confident invented function name is not.

### 4. Build the JSON
Conform to `schema.json`. Rules:
- One step = one meaningful unit of work in one system. Aim for 4–9 steps; collapse trivial plumbing.
- `system` must be a key in the `SYSTEMS` registry (see step 2).
- `file` and `fn` must be the **real** path and function/route you read. Use `(external)` for third-party-only steps.
- `data` = what concretely flows to the next step (a token, a user record, a list of keys). Keep it short; it renders on the connector.
- `detail` = 2–4 plain-language sentences. This is the explanation box that travels with the step. Explain *why*, not just *what*.
- Add a `branch` only where the code has a real alternate/failure path.
- Populate the top-level `systems` array in first-appearance order (drives the legend).

### 5. Render
- Copy `template.html` to the output path.
- Replace the entire `const FLOW = { ... };` literal with your JSON, assigned to `FLOW`.
- If step 2 required new systems, add them to the `SYSTEMS` registry in that same copy.
- Do not touch anything else in the template — the renderer and styles are fixed.
- Tell the user the file path and that they can open it in a browser. Navigation: arrow keys, Prev/Next, or click any node.

## Validate before finishing
- The JSON parses and every `system` value exists in the `SYSTEMS` registry of the output file.
- Every `file`/`fn` corresponds to something you actually opened.
- Step `id`s are sequential from 1.
- Open the file mentally: does the flow read correctly start to finish?

## Notes
- The template loads the Tabler icon webfont from a CDN; the user needs to be online the first time they open it.
- Keep `detail` text honest about uncertainty. The value of this tool is trust — a diagram that confidently lies about the code is worse than no diagram.

---

## PROJECT_CONFIG (optional)

The skill auto-detects your stack and repo layout from manifest files, so **you usually don't need to fill this in.** Edit it only to override wrong detection, or to help with an unusual / monorepo layout.

To use it: uncomment the block below (remove the `<!--` and `-->`) and keep only the keys you need. Paths are relative to where you run the skill.

```
<!-- PROJECT_CONFIG
repos:
  - path: ./api        # a backend repo/folder
    role: backend
  - path: ./mobile     # a mobile app folder
    role: mobile
  - path: ./web        # a web frontend folder
    role: web

stack:
  mobile:  React Native (Expo)
  web:     Next.js
  backend: Fastify
  db:      Postgres
  orm:     Prisma
  cache:   Redis
  storage: Cloudflare R2
  auth:    Firebase Auth
PROJECT_CONFIG -->
```
