---
name: system-design-review
description: Review a flow in the Meels codebase and evolve it under load. Grounds in the real code, then produces 2-5 incremental architecture "versions" (each fixes the previous one's failure) calibrated to the next 10x of scale, rendered as an interactive version-toggle diagram plus a written report. Use when asked to review, scale, or harden a flow (signup, upload, feed, notifications, payments, etc.).
---

# system-design-review

Flow to review: $ARGUMENTS

Take a flow that already exists in this codebase, understand how it works **today from the real code**, then evolve it version-by-version the way a system-design walkthrough does: v1 → where it breaks at scale → v2 fixes it → where *that* breaks → v3, and so on. The renderer (`template.html`) is fixed; your job is to trace the real flow, reason honestly about scale, and produce accurate JSON + a written report.

Two hard rules that separate this from generic advice:
1. **Ground everything in the actual code.** Every claim about the *current* state must trace to a file you opened (cite `file:line`). Never describe a strawman.
2. **Right-size for the next 10x, not infinity.** Default posture: recommend what's worth doing before the *next* realistic growth stage. Aggressively resist over-engineering — see the guardrail below.

## What you produce

A folder `.claude/design-output/<slug>/` (create it; `<slug>` is kebab-case from the flow, e.g. `mobile-signup`). Inside:

- **`<slug>.html`** — a copy of `${CLAUDE_SKILL_DIR}/template.html` with the `const REVIEW = { ... };` object replaced by your JSON. This is the interactive, version-toggle diagram. Do not touch anything else in the template — the renderer, styles, and `SYSTEMS` registry are fixed (except appending new systems, see step 4).
- **`<slug>.md`** — the written report (structure below). This is the readable companion to the diagram.
- Any supporting resources the review needs (rare — e.g. a query plan you captured). Most reviews are just the two files.

Reference bundled files as `${CLAUDE_SKILL_DIR}/template.html` and `${CLAUDE_SKILL_DIR}/schema.json` so paths resolve wherever the skill lives.

## Process

### 1. Ground in the real flow — do not skip

**First, inventory the stack from env.** Before tracing, read the three `.env.example` files — `meels-api/.env.example`, `meels/.env.example`, `meels-web/.env.example`. The keys are the fastest, most reliable map of what external services are actually wired up (and, just as importantly, what is *not*). Use them to anchor which `SYSTEMS` a flow can touch and to keep verdicts honest. Read the example files, never real `.env` — never print secret values. What they currently reveal:
- **DB**: `DATABASE_URL` (pooled) + `DIRECT_URL` (migrations) → Postgres/Prisma.
- **Media**: `R2_*` → Cloudflare R2. **Cache/ephemeral**: `UPSTASH_REDIS_REST_*` → Upstash Redis.
- **Auth**: `FIREBASE_*` (admin) + `EXPO_PUBLIC_FB_*` / `NEXT_PUBLIC_FB_*` (clients) + Google OAuth; plus `JWT_SECRET`, `MAGIC_LINK_*` for admin magic-link login.
- **Email**: `RESEND_API_KEY` + `EMAIL_FROM` → **Resend** is the email provider (use the `email` system; note "Resend" as sublabel).
- **Analytics/LLM**: `POSTHOG_*` → PostHog; `ANTHROPIC_API_KEY` → Anthropic.
- **Absent = not built yet**: there are **no** queue/SQS/Kafka/broker keys. So any "add a queue" version is a *new* component today — say so, and prefer Postgres/Redis-backed queuing (see ledger) rather than assuming managed infra exists.
If a key exists that isn't in this list, the stack changed — trust the env file over this doc and reconcile.

Then, **trace the flow.** Start at the entry point (a screen tap, an API route, a cron job) and follow the actual call chain across the relevant repos (`meels/`, `meels-api/`, `meels-web/`).
- `grep`/search for the entry point, then **open each file and read the function body** before describing it. Do not infer from names.
- Follow it across boundaries: mobile hook → `lib/api.ts` → Fastify route → service → Prisma query → Postgres / Redis / R2 / Firebase / push. Note every system crossing.
- Note the real denormalized counters, transactions, and failure branches you see. Capture `file:line`.
- Decide **which version ≈ "you are here"** — the architecture the current code most closely matches. This is `grounding.current_version` and gets the `here-today` verdict. Be honest: Meels may already be at v2 or v3 for a given flow, in which case v1 is historical context and the real work is the later versions.

If you genuinely can't find part of the chain, say so in the report rather than inventing it.

### 2. Establish the target scale
Default to **next 10x** from where Meels plausibly is today (early-stage consumer app). State it concretely in `target_scale` with numbers (DAU, writes/sec, uploads/day, feed reads/sec — whatever this flow stresses). Every `breaks` and every `verdict` is judged against this number. Do not design for millions unless the flow is already near that.

### 3. Evolve it — 2 to 5 versions
Build the version chain. Each version:
- **fixes the previous version's `breaks`**, and
- usually **introduces a `new_problem`** the next version solves (the terminal version may have none, or only a residual/operational one).
- carries a **`verdict`** (`here-today` / `adopt-now` / `defer` / `target`) grounded in the stack ledger.

**Use exactly as many versions as the flow needs — 2, 3, 4, or 5. Do NOT pad to 4.** A simple read path might be 2 versions; a write-heavy pipeline might be 5. The count is driven by how many distinct failure modes actually appear at target scale, not by a template.

Anchor each version to a **concrete trigger** in `breaks`: "at ~X concurrent Y, Z saturates" — not "this doesn't scale."

### 4. Lay out the graph
For each version, position nodes on a grid (`x` = column, `y` = row, fractional allowed; entry point at top, `y` increasing downward). Keep columns tidy (typically 0 / 1.5 / 3). **Reuse the same node `id` and position across versions for a component that persists** so it stays put when the user toggles — only new components move. Mark introduced components `status: "new"` (glow + NEW badge) and behavior changes `status: "changed"`.
- Each node's `system` must be a key in the `SYSTEMS` registry in `template.html`. Map Meels tech to keys: Fastify→`fastify`, Postgres→`postgres`, Prisma→`prisma`, Redis/Upstash→`redis`, R2→`r2`, Firebase→`firebase`, Expo→`mobile`/`expo`, push→`push`, PostHog→`posthog`, a queue→`queue` (or `sqs`/`kafka` if you specifically recommend them), a worker→`worker`, DLQ→`dlq`, outbox→`outbox`, read replica→`replica`.
- **If a system genuinely isn't in the registry**, append an entry to the `SYSTEMS` object *in your output copy* of `template.html`: `key: { label: "...", color: "#hex", icon: "ti-..." }` (Tabler icon name, distinct color).
- Edges carry `kind`: `sync` (blocking, gray), `async` (non-blocking, blue marching-ants), `write` (purple), `read` (cyan), `fail` (red dashed, e.g. → DLQ). Add short labels.
- Use `groups` for a boundary that matters (a transaction, a VPC/region) — e.g. `{ "label": "One transaction", "nodes": ["db","outbox"], "kind": "transaction" }`.

Conform to `schema.json`.

### 5. Write the report (`<slug>.md`)
Structure:
- **Title + one-line summary + target scale.**
- **Current state** — how the flow works today, with `file:line` citations, and which version that maps to.
- **Per version** (`## V1 — Name`): what it adds, where it breaks (with the scale trigger), the new problem, and the verdict with justification.
- **Recommendation summary** — a short table: each proposed change → `adopt-now` / `defer until <threshold>`, and the concrete first step in Meels' stack (which file/service). Lead with what to do next.

### 6. Render + validate
- Copy `template.html` to `.claude/design-output/<slug>/<slug>.html`, replace the `REVIEW` object, append any new `SYSTEMS`.
- Validate: JSON parses; every node `system` exists in `SYSTEMS`; every edge `from`/`to` is a node id in that version; every group node id exists; version `id`s are sequential from 1; 2–5 versions.
- Tell the user both file paths and that they can `open` the HTML. Navigation: arrow keys / tabs switch versions, click a node for detail, drag to pan.

## Stack ledger — what Meels runs, and how far each piece stretches
Ground every `verdict` in this, and cross-check it against the `.env.example` files (step 1) — those are the live source of truth; this ledger is the interpretation. The point is to recommend the *smallest* change that clears the next 10x.

- **API**: Fastify (`meels-api`), Node ESM, tsx dev. Vertical scaling + more instances behind a balancer covers a lot before anything exotic is needed.
- **DB**: PostgreSQL via Prisma, pooled `DATABASE_URL` + `DIRECT_URL`. **Reach for these in order before sharding:** missing indexes → fix N+1 / `select` narrowing → the existing denormalized counters → read replica for read-heavy paths → partitioning. Sharding is a last resort, well past next-10x.
- **Cache / ephemeral**: Upstash **Redis** already in use (`videoFeed.service.ts`, viewed-sets). A queue/worker can piggyback here (BullMQ) or on Postgres (pg-boss) — **prefer these over SQS/Kafka** until job volume is genuinely high (thousands/min sustained).
- **Media**: Cloudflare **R2**, presigned-URL upload→commit flow. Bytes never proxy through the API — preserve that. The upload→commit path is a real dual-write (R2 object + DB row) and is a prime outbox/reconciliation candidate.
- **Auth**: Firebase (identity provider; `User.id` == Firebase uid). Don't propose replacing it.
- **Email**: **Resend** (`RESEND_API_KEY`, `EMAIL_FROM`). A real third-party you call synchronously today unless proven otherwise — a prime candidate to push behind a queue/outbox.
- **Async today**: `notification.service.ts` (push), PostHog capture, dd-trace. These are the existing seams to route background work through. There is **no** message broker in env, so "worker/queue" versions are net-new infra — default to pg-boss (Postgres) or BullMQ (existing Redis).

**When a stack change IS justified** — name the trigger, don't hand-wave: e.g. "add a read replica once read QPS on the feed sustains >N and replica lag is acceptable," "move from pg-boss to SQS only past ~X jobs/min or when you need cross-service fan-out." If the current stack clears next-10x, the verdict is `defer` with the threshold stated.

## Pattern catalog — map these onto the code, don't recite them
Reach for the one that fits the failure you found: **transactional outbox** (dual-write between DB and queue/other system), **idempotency keys** (at-least-once → duplicate side effects), **visibility timeout / lease** (worker crash safety), **dead-letter queue** (poison jobs), **read replica / CQRS** (read-heavy), **caching + TTL / cache-aside** (hot reads), **rate limiting + backpressure** (spiky writes, third-party limits), **fan-out on write vs read** (feeds/timelines), **pagination / keyset** (large lists), **debounce/batch** (counter updates, notifications). Each becomes a version only if the failure that motivates it actually appears at target scale.

## Anti-over-engineering guardrail (important)
This tool's credibility dies if it tells an early-stage app to adopt Kafka and microservices. Before every `adopt-now`, ask: *does the failure it fixes actually occur within target scale?* If not, it's `defer` with a named threshold. Favor: one Postgres table over a new datastore; the existing Redis/cron over new infra; an index or query fix over architecture. A correct review often concludes "you're fine until X — here's the one thing to add now." That is a *success*, not a thin result.

## Honesty notes
- The template loads the Tabler icon webfont from a CDN; the user must be online the first time they open the HTML.
- Keep `note`, `breaks`, and `verdict` honest about uncertainty. A diagram that confidently invents a bottleneck is worse than one that says "measure this first."
- The demo `REVIEW` shipped in `template.html` (sign-up / background jobs) is an illustrative sample, not a Meels-grounded review. Replace it entirely.
