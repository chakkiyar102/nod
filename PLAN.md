# nod-grokify: build plan

Fork of n8n-io/nod (`chakkiyar102/nod`), local clone at `~/nod-grokify`, synced to upstream
`93707ae` on 2026-11-18. Goal: a Grok-style personal assistant on Nod. The agent's browser plays
live in chat while it works, the owner can run several bots, and some bots run outside agent CLIs
(Claude Code, Codex) using what mausbot-rs proved.

Model provider: Command Code Provider API (`https://api.commandcode.ai/provider/v1`) as Nod's
OpenAI-compatible connection. Host for development: 8 GB Apple silicon Mac, so the desktop VM gets
4 GB (`desktop/runtime/index.mjs:402`) and two or three browsers is the practical ceiling.

## Ground rules

- Keep upstream mergeable. New code goes in new modules (`src/server/bots.ts`,
  `src/web/components/live-browser.tsx`, `src/providers/cli-engine/`). Edits to big shared files
  (`thread.ts`, `tools.ts`, `runtime/index.ts`, `app.ts`) stay small and are listed below.
  Rebase on `upstream/main` weekly: `git fetch upstream && git rebase upstream/main`.
- Do not weaken the trust model. The trust check, Vault, approvals, untrusted-result marking and
  "provider keys never enter the sandbox" all stay. Every milestone below keeps them.
- Each milestone ends with `pnpm check` (or the CI script set in `.github/scripts/ci-tests.mjs`),
  its own tests, and a short screen recording of the feature working in `pnpm start`.
- One milestone per branch: `grok/m1-live-browser`, `grok/m2-cursor-overlay`, and so on.

## Current code: what exists and what blocks us

| Area | Where | Finding |
| --- | --- | --- |
| Live browser frames | `src/server/app.ts:779` `GET /api/browser-handoff/stream`; `src/providers/container-browser/client.ts:321` `view()`; frames are base64 images (`protocol.ts` `viewFrame`, 4 MB max) | Streaming exists, but the endpoint only opens while a handoff is open (`handoffBrowser(turnId)`). |
| Workspace browser window | `src/web/pages/browser.tsx`, `src/web/components/browser-screen.tsx`, `GET /api/browser/screen` | Polled PNG of the "current" browser in a separate window. Not in chat, not per task. |
| Takeover panel | `src/web/components/browser-handoff.tsx` (653 lines), `src/web/lib/browser-frames.ts` | Interactive panel inside chat. Reuse its frame decoding and layout for the watch-only view. |
| Browser events | `turn_events` (`src/server/db.ts:98`) has `browser_id`, `site`, `call_id`, `run_id`, `parent_run_id`; label like `browser click` built at `src/runtime/index.ts:474` | We know which browser acted and the action name. No element box or coordinates, so no cursor position yet. |
| Subagents | `src/runtime/tools.ts:284` (subagent tool), runs with own browser, `parentRunId` | Parallel browsers already exist (max 4, `use/browser.mdx`). |
| One conversation per owner | `src/server/db.ts:69` `UNIQUE INDEX threads_active_owner ON threads(owner_id) WHERE archived_at IS NULL` | This is the main blocker for multiple bots: one active thread per owner. |
| System prompt | `src/runtime/prompt.ts:34` `systemPrompt(...)` | Single fixed persona. A bot persona is an extra section here. |
| OpenAI-compatible models | `src/providers/pi/models.ts:343-360` | `cost` is hard-coded to 0, so Usage shows $0 for Command Code. No custom headers, so no `x-cmd-zdr: 1`. |
| Channels | `src/server/channels.ts`, `channel-settings.ts` | One Telegram and one WhatsApp pairing per install. |

## Milestones

### M0. Setup (half a day)

1. Install Node 24.21+, Colima with Docker, Compose v2 and Buildx (`brew install node@24 colima docker docker-compose docker-buildx`), `colima start --cpu 4 --memory 5`.
2. `.env`: `LLM_PROVIDER=compatible`, `OPENAI_COMPATIBLE_BASE_URL=https://api.commandcode.ai/provider/v1`, `OPENAI_COMPATIBLE_API_KEY=...`, `LLM_MODEL=` a tool-capable model from `/provider/v1/models`.
3. `corepack enable && pnpm start`. Run one browser errand end to end. Note memory use with `docker stats`.
4. (Done 2026-11-18) Fork `main` synced with upstream on GitHub.
5. Trust checks on Jev: add `TYPESAFE_API_KEY` (already in `~/.zshrc`) to `.env` and set Trust to
   `decision` mode. Trust checks then use the decision model instead of falling back to a small chat
   model, which is the cheapest per-action saving available. Nod calls TypeSafe directly
   (`src/providers/decisions.ts:23`); Command Code's `/v1/systemone` also serves Jev but would need a
   one-line URL override, so leave it on TypeSafe for now.
6. Tracing with Phoenix: run Phoenix locally (`docker run -p 6006:6006 -p 4317:4317 arizephoenix/phoenix`)
   and set `OTEL_EXPORTER_OTLP_ENDPOINT` in `.env`. Nod exports model-request metadata only (no
   prompts or results), which is enough to measure cost and latency per finished task. Stop the
   container when not measuring; it uses memory.

Done when: a browser task completes on Command Code, the takeover panel works, a trust check shows
as a Jev decision, and the task's model calls appear in Phoenix.

### M1. Live browser in chat, watch-only (2 to 3 days)

The single most visible Grok feature.

- Server: new `GET /api/browser/live?turnId=&runId=` in a new `src/server/live-browser.ts`, registered
  from `app.ts`. Allowed while the turn is `running` and the browser belongs to that turn's run (look
  up `turn_events.browser_id` for the run). Reuses `dependencies.computer.view(browserId, signal)`.
  No input endpoint: the stream is read-only by construction.
- Frame budget: cap to about 5 fps and drop frames when the client is behind (mirror the
  `frame-ack` back-pressure in `/api/browser-handoff/frame-ack`).
- Web: new `src/web/components/live-browser.tsx`. It appears under the running task's message when
  the first `browser` event arrives, shows the site and the last action label, collapses when the
  turn finishes, and switches to the existing takeover panel if a handoff opens. A "Take over" button
  asks the agent to hand off (reuse the existing handoff request path).
- Settings toggle: "Show the browser while Nod works" (default on, desktop and web only; Telegram
  and WhatsApp unchanged).

Done when: asking "find the cheapest flight X to Y" shows the page moving in chat in real time,
with no input possible until takeover. Tests: endpoint refuses finished turns, other owners' turns,
and a browser not tied to the run.

### M2. Cursor, click highlight and step captions (2 days)

- Runtime: for `browser` actions that target an element (`click`, `fill`, `select`, `check`, `drag`,
  see `tools.ts:740`), have the browser provider return the element's bounding box with the result.
  Record it on the event as `command.box = {x,y,w,h,viewport}` (the `command jsonb` column exists).
  Do not record typed text for `fill`, since it may be a secret; record only the box.
- Web: overlay an animated pointer that eases to the box centre, a click ring, and a one-line
  caption ("Clicking Search flights"). Port the spring-pointer approach from the `animated-pointer`
  skill.
- Vault-filled fields show only "Entering saved login" with no box.

Done when: the overlay tracks real clicks within one frame of the action. Test that no Vault value
or typed text ever reaches `turn_events`.

### M3. Parallel browsers grid (1 to 2 days)

- When subagents run, the live panel becomes a grid of their browsers (one tile per `run_id` with a
  browser), labelled with the subagent's instruction summary. Click a tile to enlarge.
- Respect the 4-browser cap; on the 8 GB Mac set the cap to 2 via config.

Done when: "compare prices at three shops" shows three tiles that update at once.

### M4. LiteLLM gateway in front of Command Code (1 day, no Nod code)

Nod's OpenAI-compatible connection points at a local LiteLLM proxy instead of Command Code
directly. LiteLLM forwards to Command Code and fixes three gaps without touching Nod:

- Claude models: LiteLLM takes Nod's OpenAI-format requests and calls Command Code's Anthropic
  `/v1/messages` for Claude models (`anthropic/<model>` with `api_base` set to Command Code), and
  `/v1/chat/completions` for the rest.
- Costs and budgets: LiteLLM tracks spend per model and per key. Give each bot (M5) its own LiteLLM
  virtual key with a budget. Nod's own Usage page still shows $0 for this connection, so LiteLLM's
  dashboard is the spend source of truth.
- Zero data retention: add `x-cmd-zdr: 1` as an extra header on every Command Code route.

Steps:

1. `litellm-config.yaml` listing the chosen Command Code models (chat models via `openai/<id>`,
   Claude via `anthropic/<id>`), `extra_headers: {x-cmd-zdr: "1"}`, and the Command Code key read
   from an environment variable, never written in the file.
2. Run LiteLLM bound to loopback only (`127.0.0.1:4000`) with a master key. On the desktop VM or
   Docker setup, make sure Nod's server can reach it (Nod rejects private addresses only for the
   workspace, not for model connections; verify).
3. In Nod: `OPENAI_COMPATIBLE_BASE_URL=http://<litellm-host>:4000/v1`,
   `OPENAI_COMPATIBLE_API_KEY=<litellm virtual key>`.
4. Docs note in `docs/src/content/docs/connect/` describing the setup, for the fork.

Fallback if LiteLLM's memory use (roughly 200 to 400 MB, measure it) is too much on the 8 GB Mac:
the original in-Nod change. Read per-model prices from `/models` in `src/providers/pi/models.ts`
(cost is hard-coded to 0 at lines 343-360) and add an `OPENAI_COMPATIBLE_HEADERS` setting for the
ZDR header. That fallback does not give Claude models.

Done when: Nod runs one task on a Claude model and one on an open model through LiteLLM, both
requests carry `x-cmd-zdr: 1`, and LiteLLM's spend matches Command Code's dashboard within rounding.

### M5. Multiple bots inside one install (5 to 8 days)

Turns Nod from one assistant into several, each with its own name, persona, model and channel.

- Schema (in `db.ts`, additive only):
  - `bots(id uuid pk, owner_id, name, persona text, model_connection text, model_id text,
    memory_scope text default 'shared', created_at, archived_at)`.
  - `threads.bot_id uuid` nullable. A default bot row is created per owner and existing threads are
    pointed at it.
  - Replace `threads_active_owner` with a unique index on `(owner_id, bot_id) WHERE archived_at IS NULL`.
    This is the change most likely to conflict with upstream; keep it in one migration block.
- Runtime: pass the bot into `systemPrompt()` (`prompt.ts:34`) as an extra "You are <name>.
  <persona>" section, and select the bot's model instead of the global default.
- Memory: phase 1 is shared memory with a `bot_id` tag on new notes. Phase 2 (later) filters recall
  by bot when `memory_scope='own'`.
- Channels: one Telegram bot token per bot. `channel-settings.ts` stores `bot_id` with each pairing;
  inbound messages route to that bot's thread.
- Web: bot switcher in the sidebar, a "New bot" dialog (name, persona, model), and a bot badge on
  tasks and schedules.
- Trust: owner rules and spending limits stay per owner and apply to every bot. Approvals show which
  bot is asking.

Done when: two bots ("Errands", "Research") on different Command Code models hold separate
conversations, each reachable from its own Telegram bot, with approvals labelled by bot.

### M6. CLI engine bots: Claude Code and Codex (5 to 7 days)

Ports the proven mausbot-rs design (`~/Documents/mausbot-rs/crates/harness/src/{claude,codex,broker}.rs`,
`spikes/m1/RESULTS.md`) into Nod as a bot type `engine = 'nod' | 'claude-code' | 'codex'`.

- Where it runs: inside the workspace container, so file and shell access stay sandboxed and behind
  the workspace firewall. Install `claude` and `codex` in `Dockerfile.workspace`.
- Keys stay out of the sandbox. Add a server-side model proxy reachable from the workspace that
  accepts a per-turn token and adds the real key. Point the CLI at it with `ANTHROPIC_BASE_URL`
  (Claude Code to Command Code's `/v1/messages`, which also unlocks Claude models through Command
  Code) or Codex's OpenAI base URL. The token expires with the turn.
- Approvals: port mausbot's broker. Claude Code runs with `--permission-prompt-tool` pointed at a
  stdio MCP server that forwards each request to Nod's existing approval cards (`src/server/approvals.ts`),
  including `AskUserQuestion`. Codex uses `codex app-server` JSON-RPC approval requests, with an
  isolated `CODEX_HOME`.
- Isolation flags from mausbot: `--setting-sources project --strict-mcp-config`, `--plugin-dir` per
  selected plugin, `--resume <session>` per thread.
- Output: map stream-json events onto `turn_events` so the chat shows tool steps like Nod's own runs.
- First target: a "Kissflow builder" bot with the `app-agents@kissflow` plugin running
  `/app-agents:author-app`, as already proven live in mausbot-rs on 2026-11-13.
- Not in scope: giving CLI bots Nod's browser. They keep their own tools; browsing errands go to a
  Nod-engine bot.

Done when: the Kissflow builder bot builds a test app from a one-line brief, each risky step shows
a Nod approval card, and no provider key appears in the workspace (`env` and the filesystem checked).

### M7. Optional: several agents on one answer (3 days)

Grok-Heavy style. A `fan_out` mode on the subagent tool: same instruction to N subagents on
different models, then a final step compares and merges. Off by default, with a cost estimate shown
before it runs, because it multiplies spend by N.

### M8. Optional cost trims (1 day)

- Fetch before browser: check `src/runtime/prompt.ts` and the tool descriptions in
  `src/runtime/tools.ts` so read-only page lookups use the web fetch tool (`pi-web-access`, returns
  text) instead of opening a browser and reading screenshots. Measure in Phoenix before and after on
  three read-only errands.
- RTK in the workspace: install `rtk` in `Dockerfile.workspace` and wrap common noisy commands, so
  shell output reaching the model is shorter. Low gain, since Nod runs fewer shell commands than a
  coding agent; do it only if Phoenix shows large bash outputs.

Source: review of a cost-cutting thread (x.com/simplifyinAI/status/2122869119586607258, 2026-11-18).
Its three habits (skip checks that find nothing, no full-history reloads, capped retries) are
already built into Nod. Its queue, memory, caching and prompt-compression tools were reviewed and
skipped because Nod has equivalents or because caching personal results risks stale answers.

## Order and effort

| # | Milestone | Effort | Depends on |
| --- | --- | --- | --- |
| M0 | Setup on Command Code, Jev trust checks, Phoenix | 0.5 to 1 d | none |
| M1 | Live watch-only browser | 2 to 3 d | M0 |
| M2 | Cursor and captions | 2 d | M1 |
| M3 | Parallel browser grid | 1 to 2 d | M1 |
| M4 | LiteLLM gateway: Claude, costs, ZDR | 1 d | M0 |
| M5 | Multiple bots | 5 to 8 d | M0 (M4 for per-bot budgets) |
| M6 | Claude Code and Codex bots | 5 to 7 d | M5 |
| M7 | Multi-agent answers | 3 d | M5 |
| M8 | Optional cost trims | 1 d | M0 (Phoenix) |

M1 to M4 give the Grok feel in about a week. M5 and M6 are the large structural changes.

## Risks

- Upstream drift: `thread.ts`, `tools.ts` and `runtime/index.ts` change often. Keep edits small and
  rebase weekly. M5's index change is the most likely conflict.
- Memory on the 8 GB Mac: the live view adds frame encoding load; M3 and M6 add browsers and CLI
  processes. Move to a 16 GB machine or a VPS before M6 if `docker stats` shows swapping.
- Secrets in the live view: the frames show whatever the page shows, including filled forms. The
  view is owner-only, never shared, never stored. Do not add recording without masking.
- Extra processes: LiteLLM and Phoenix both cost memory on the 8 GB Mac. Run Phoenix only while
  measuring, and measure LiteLLM's footprint before keeping it.
- LiteLLM is now in the model path: if it is down, Nod has no model. Start it with Nod, or keep a
  direct Command Code connection as a second connection in Nod.
- Command Code model quality: weaker tool-calling models fail long errands. Benchmark two or three
  models in M0 on the same errand before picking defaults.
- Licence: Apache 2.0. Keep `LICENSE` and `NOTICE`, and mark changed files.
