# four-ai-one-repo

> Alright Chat, here's the deal. I am trying to get AI agents (maybe even you!) to work together on a project.
> 
> GOAL: Gemini, ChatGPT, Claude (I guess), Crazy wild card AIs, all communicating with each other in a wild group chat.
> 
> GOAL: The AI have a structured debate where they propose an app idea, discuss ideas for 4 rounds, then vote on an app to make (I will split ties if needed).
> 
> GOAL: The AI each take the app idea and split up the design work, producing a design document complete with: UX mock ups, user journey, user stories, and tech specs, complete with a development roadmap and task breakdowns that say how to make the app step-by-step, grouped in milestone deliverables: PoC, MVP, Vertical Slice, Alpha, Beta. Final release.
> 
> GOAL: The AI decide amongst themselves who is assigned what tasks and work together towards each milestone.
> 
> GOAL: PoC milestone is achieved and delivered and actually works when a human uses it.
> 
> THE SETUP: This will all be happening locally on my machine. We'll make a public git repo. Each agent will have their own git worktree and work out of branches that are named {agent}/{task}.
> 
> GOAL: The AI agents will code review each other's PR before merging into main.
> 
> Given the above goals and setup, what technologies would YOU PERSONALLY try to use to achieve this? Some kind of database? Some kind of message queue? Agents would need to broadcast each other, I guess? They would need a way to track progress or share information. 
> 
> This is my wild experiment. Your thoughts?

```text
James, this is cat-herding with lasers. Fun. Here’s the clean, ship-it stack I’d use to make your “agents build a real app together” experiment actually work on a single machine without turning into glue-spaghetti.

TL;DR: my opinionated stack
	•	Orchestrator / runtime: LangGraph (Python) to encode the debate → planning → execution state machine. It’s boring, stable, and built for multi-agent graphs.
	•	LLM vendor fan-out: LiteLLM as a single proxy to OpenAI/Anthropic/Google; one API for all models + per-agent routing and rate limits.
	•	Message bus (broadcast + pub/sub): NATS JetStream (tiny, fast, dead-simple, durable). Better than Redis Streams for multi-topic fan-out and replay.
	•	Source of truth DB: Postgres (Docker) with pgvector (if you want RAG/memory). Keep it relational: tasks, runs, artifacts, votes, PRs.
	•	Disk artifacts & protocol: Git repo as the canonical artifact store; JSON Lines events in /events/ for full audit; Markdown deliverables in /docs/.
	•	Job queue: BullMQ (Node + Redis) OR just use NATS Work Queues. If you want one broker, stick to NATS (request/reply + queue groups).
	•	Repo automation / PRs: GitHub App (Probot) or plain gh CLI tokens per agent. Each agent works in a git worktree on branch agents/{name}/{task}.
	•	Code execution sandbox: Docker Compose services per agent with read/write bind-mount to its worktree. Optional Firecracker later if you get spicy.
	•	Design outputs: Use Mermaid + Excalidraw JSON + SVG; generate a lightweight Storybook for UX comps; no Figma API dependency for PoC.
	•	Testing/CI: GitHub Actions with required reviews by other agents (bot accounts), plus a human override (you).
	•	Observability: OpenTelemetry traces → Tempo/Loki (optional). For sanity, log to files and ship later.

⸻

Architecture (simple & tough)

                 ┌─────────────────────────┐
                 │       Orchestrator      │  (LangGraph)
                 │  debate→plan→assign→run │
                 └───────────┬─────────────┘
                             │
                    (commands/events)
                             │
     ┌───────────────────────┴──────────────────────────┐
     │                   NATS JetStream                 │
     │  subjects: debate.*, plan.*, tasks.*, votes.*,   │
     │            pr.*, runs.*, artifacts.*             │
     └───────┬───────────────┬───────────────┬──────────┘
             │               │               │
     ┌───────▼───────┐ ┌─────▼────────┐ ┌────▼─────────┐
     │ Agent: ChatGPT│ │ Agent: Claude│ │ Agent: Gemini│  …(+ Wildcard)
     │ (container)   │ │ (container)  │ │ (container)   │
     │ LiteLLM -> LLM│ │ LiteLLM ->   │ │ LiteLLM ->    │
     └─┬─────────────┘ └─────┬────────┘ └────┬──────────┘
       │   git worktree       │               │
       │   branch: agents/{name}/{task}       │
       │                                      │
       └─────────────── file ops ─────────────┘
                     Git monorepo (origin: GitHub)
                     /docs /events /apps /packages …

                    Postgres (tasks, runs, votes, assignments)


⸻

Core ideas that keep this sane
	1.	One brain, many mouths.
Let one Orchestrator own the phase clock and compliance (who speaks when). Agents are pure workers that react to NATS subjects and write artifacts to their branches. No emergent chaos.
	2.	Events are law.
Everything is an event on NATS and a JSONL append in /events/{yyyymmdd}.jsonl. If it didn’t get emitted, it didn’t happen. You can replay and audit.
	3.	Git is the artifact registry.
Designs, mockups (SVG/PNG), specs (MD), code, roadmaps—everything lands in branches. PRs are the merge gateway with agent cross-review requirements.
	4.	Vendors are swappable.
LiteLLM gives you a single client; agents only know “model=claude-3.X” or “model=gpt-4.X” etc. Swap providers without rewriting code.

⸻

Concrete: how the protocol flows

Phase 1 — Structured Debate (4 rounds + vote)
	•	Subjects:
	•	debate/round.{1..4}: orchestrator publishes the prompt + rules + timebox.
	•	debate/round.{n}.submission: each agent posts a message blob {agent_id, content_md, citations}.
	•	debate/vote: each agent emits {agent_id, ranked_choices:[id1,id2,id3]}.
	•	Orchestrator closes round on timeout, aggregates, tallies Condorcet fallback to IRV; you break ties.

Deliverables: /docs/debate/{round}/agent-{name}.md + /docs/debate/vote-results.md

Phase 2 — Design Split & Work Allocation
	•	Orchestrator issues work packages:
	•	plan/tasks.create with payloads like:

{
  "task_id":"T-UX-001",
  "domain":"design",
  "title":"Wireflow + user journeys",
  "artifacts":["/docs/design/wireflow.mmd","/docs/design/journeys.md"],
  "acceptance":["storyboard complete","2 alt flows","png export"]
}


	•	Agents bid or get assigned: plan/tasks.claim → {agent_id, task_id}
	•	DB persists tasks, assignments, and SLA (deadline). (Schema below.)

Phase 3 — Deliverables & PRs
	•	Each agent writes in its worktree branch agents/{name}/{task}.
	•	When ready: push branch, open PR (GitHub App or gh pr create).
	•	Publish pr/open event with pr_number, task_id, checks.
	•	Other agents must approve (CODEOWNERS can require cross-agent review).
	•	CI runs lint/tests/build; orchestrator listens to pr/merged to advance milestones.

Phase 4 — Milestones
	•	Milestones as state gates in Postgres:
	•	ProofOfConcept → MVP → VerticalSlice → Alpha → Beta → Release
	•	Entry criteria = checks of concrete artifacts + passing tests; orchestrator enforces.

⸻

Minimal schemas (Postgres)

-- agents
create table agents (
  id text primary key,
  display_name text not null,
  model text not null,     -- e.g., gpt-4.1, claude-3.5, gemini-2.0
  capabilities jsonb       -- sketch of strengths
);

-- tasks
create type task_status as enum ('queued','claimed','in_progress','review','merged','blocked');
create table tasks (
  id text primary key,
  title text not null,
  domain text not null,                 -- design, backend, frontend, testing
  milestone text not null,              -- PoC, MVP, etc.
  spec_path text,                       -- where spec lives in repo
  status task_status not null default 'queued',
  created_at timestamptz default now(),
  due_at timestamptz
);

-- assignments
create table assignments (
  task_id text references tasks(id),
  agent_id text references agents(id),
  assigned_at timestamptz default now(),
  primary key(task_id, agent_id)
);

-- runs & events (for audit)
create table agent_runs (
  id bigserial primary key,
  agent_id text not null references agents(id),
  task_id text,
  phase text not null,                  -- debate, plan, build, review
  input jsonb,
  output jsonb,
  started_at timestamptz default now(),
  finished_at timestamptz
);

-- votes
create table votes (
  id bigserial primary key,
  agent_id text references agents(id),
  ballot jsonb not null,                -- ranked choices
  created_at timestamptz default now()
);


⸻

NATS subjects (stable contract)
	•	debate.round.{n}.open / .submission / .close
	•	debate.vote.open / .ballot / .result
	•	plan.tasks.create / .claim / .status
	•	build.task.{id}.start / .artifact / .done
	•	pr.open / pr.review / pr.approve / pr.merge
	•	milestone.enter / .exit

Keep payloads as JSON; version with schema_version.

⸻

File/Repo layout (Git + worktrees)

/ (origin: GitHub)
├─ apps/
│  ├─ web/            # Next.js or SvelteKit
│  └─ api/            # FastAPI/Express
├─ packages/
│  ├─ ui/             # component lib + Storybook
│  └─ sdk/            # TypeScript client for NATS + DB
├─ docs/
│  ├─ debate/round-1/agent-*.md
│  ├─ design/wireflow.mmd
│  ├─ journeys/*.md
│  └─ tech/specs/*.md
├─ events/            # append-only JSONL
│  └─ 2025-09-07.jsonl
├─ .github/workflows/
│  └─ ci.yml
└─ tools/
   ├─ orchestrator/   # LangGraph app
   ├─ broker/         # NATS config
   ├─ github-bot/     # Probot or scripts using gh
   └─ docker-compose.yml

Each agent gets:

git worktree add ../worktrees/chatgpt-T-UX-001 agents/chatgpt/T-UX-001
# container mounts ../worktrees/chatgpt-T-UX-001 as /workspace


⸻

Docker Compose (services you actually need)
	•	nats (with JetStream enabled)
	•	postgres (+ pgvector)
	•	orchestrator (Python: LangGraph + NATS client + Postgres)
	•	litellm (proxy)
	•	agent-chatgpt, agent-claude, agent-gemini, agent-wildcard (Node or Python wrappers)
	•	github-bot (optional if you prefer gh in orchestrator)

⸻

Guardrails that prevent chaos
	•	Phase gates: Only orchestrator publishes debate.round.N.open and accepts late submissions → rejected.
	•	Deterministic timeouts: real-time is cute; deadlines are kinder.
	•	One-shot merges: PR requires ✅ CI + ≥2 cross-agent approvals (not author).
	•	Schema-first events: Put JSONSchema in packages/sdk/schemas/ and validate on ingest.
	•	Disk quotas: Agents love to dump; limit artifact size and clean old branches nightly.

⸻

PoC plan (make something real in a day)
	1.	Bootstrap infra
docker compose up -d nats postgres → run small setup SQL → start orchestrator.
	2.	Run a mock debate
Orchestrator opens round-1; each agent posts a one-paragraph app pitch. Close, tally, publish result.
	3.	Create 6 tasks for PoC milestone
	•	UX wireflow (Mermaid)
	•	User journeys (MD)
	•	API spec (OpenAPI YAML)
	•	DB schema (SQL + migration)
	•	Web app skeleton (Next.js route, 1 form)
	•	E2E smoke test (Playwright)
	4.	Agents claim & deliver
They generate artifacts into their worktree branches; open PRs; other agents review.
	5.	Merge & prove “actually works”
Spin apps/web, hit the simple flow (e.g., submit a “note”, see it persisted). That’s your PoC.

⸻

Why not Kafka/Temporal/etc.?
	•	Kafka: Overkill for one box; NATS gives you 95% of what you need with 5% of the pain.
	•	Temporal: Great, but adds a lot of ceremony. LangGraph state machine + NATS is enough.
	•	Vector DB: Only if you need heavy RAG; pgvector in Postgres is plenty to start.

⸻

“Do this now” starter list
	•	Spin nats, postgres in Compose.
	•	Create packages/sdk/ with:
	•	nats.ts: thin client with typed publish/subscribe
	•	events.ts: JSONSchemas + Zod runtime guards
	•	Write the orchestrator (Python or TS) with a 6-state enum:
	•	DEBATE_R1..R4 → VOTING → PLANNING → ASSIGNMENT → BUILD → REVIEW
	•	Add gh auth and a tiny PR helper.
	•	Register 3 agent containers wired to LiteLLM.

If you want, I can slam out the initial docker-compose.yml, the NATS subject constants, the Postgres DDL, and a skeleton orchestrator that runs Round 1 of debate and tallies votes. Then we layer on tasks and branches. Your call.
```
