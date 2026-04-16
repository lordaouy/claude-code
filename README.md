
# Claude Code (Unofficial Source Extraction)

Follow me at https://x.com/paidev



> **This is NOT an official Anthropic repository.**

This repository contains the extracted TypeScript source code of [Anthropic's Claude Code](https://www.anthropic.com/) CLI tool — Anthropic's official CLI that lets you interact with Claude directly from the terminal to perform software engineering tasks like editing files, running commands, searching codebases, managing git workflows, and more.

The source was obtained by unpacking the source map (`cli.js.map`) bundled with the officially published npm package.

- **npm package:** [@anthropic-ai/claude-code v2.1.88](https://www.npmjs.com/package/@anthropic-ai/claude-code/v/2.1.88)
- **Official homepage:** [github.com/anthropics/claude-code](https://github.com/anthropics/claude-code)

## How It Leaked

The source code leak was discovered by [Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) and posted publicly on March 31, 2026:

> *"Claude code source code has been leaked via a map file in their npm registry!"*
>
> — [@Fried_rice](https://x.com/Fried_rice), March 31, 2026

The published npm package (`@anthropic-ai/claude-code`) included a source map file (`cli.js.map`) containing the full, unobfuscated TypeScript source code. The `sourcesContent` field of the source map held every original `.ts`/`.tsx` file that was bundled into `cli.js`, making the entire codebase trivially extractable.

## Why does this exist?

Anthropic publishes Claude Code as a bundled JavaScript CLI on npm. The published package includes a source map file (`cli.js.map`) that contains the original TypeScript source. This repository simply extracts and preserves that source for easier reading and reference.

## How to get it yourself

### Clone this repository

```bash
git clone git@github.com:chatgptprojects/claude-code.git
cd claude-code
```

### Or extract it yourself from npm

1. **Install the package:**

```bash
mkdir claude-code-extract && cd claude-code-extract
npm pack @anthropic-ai/claude-code@2.1.88
tar -xzf anthropic-ai-claude-code-2.1.88.tgz
cd package
```

2. **Run the unpack script:**

Create a file called `unpack.mjs`:

```js
import { readFileSync, writeFileSync, mkdirSync } from "fs";
import { dirname, join } from "path";

const mapFile = join(import.meta.dirname, "cli.js.map");
const outDir = join(import.meta.dirname, "unpacked");

console.log("Reading source map...");
const map = JSON.parse(readFileSync(mapFile, "utf-8"));

const sources = map.sources || [];
const contents = map.sourcesContent || [];

console.log(`Found ${sources.length} source files.`);

let written = 0;
let skipped = 0;

for (let i = 0; i < sources.length; i++) {
  const src = sources[i];
  const content = contents[i];

  if (content == null) {
    skipped++;
    continue;
  }

  const outPath = join(outDir, src.replace(/^\.\.\//g, ""));
  mkdirSync(dirname(outPath), { recursive: true });
  writeFileSync(outPath, content);
  written++;
}

console.log(`Done! Wrote ${written} files to ${outDir}`);
if (skipped > 0) console.log(`Skipped ${skipped} files with no content.`);
```

3. **Run it:**

```bash
node unpack.mjs
```

The extracted source will be in the `unpacked/` directory.

## Project Structure

```
src/
├── cli/           # CLI entrypoint and argument parsing
├── commands/      # Command implementations
├── components/    # UI components (Ink/React)
├── constants/     # App constants and configuration
├── context/       # Context management
├── hooks/         # React hooks
├── ink/           # Terminal UI (Ink framework)
├── services/      # Core services
├── skills/        # Skill definitions
├── tools/         # Tool implementations (file editing, search, etc.)
├── types/         # TypeScript type definitions
├── utils/         # Utility functions
├── main.tsx       # Main application entry
├── query.ts       # Query handling
└── ...
```

## GitHub Copilot CLI Port Spec: Agent Teams

If this feature is ever ported to GitHub Copilot CLI, it should be treated as a
small distributed system inside the CLI rather than as a terminal UI feature.
The current implementation is fundamentally a leader/worker coordination model
with shared persistent state.

### Core behavioral model to preserve

- A team has exactly one leader and one or more named workers.
- The leader owns the session-level coordination flow.
- Workers may run in-process or out-of-process.
- All team members share:
  - a team registry
  - a task queue
  - a mailbox/message bus
  - permission and approval routing
- The expected lifecycle is:
  1. create a team
  2. spawn workers
  3. assign or claim tasks
  4. exchange messages
  5. route approvals through the leader
  6. mark workers idle between turns
  7. gracefully shut workers down
  8. delete team state only after workers are inactive

### Current implementation map in this repository

The current Claude Code implementation is spread across these areas:

- team creation and deletion
  - `src/tools/TeamCreateTool/TeamCreateTool.ts`
  - `src/tools/TeamDeleteTool/TeamDeleteTool.ts`
- worker spawning
  - `src/tools/AgentTool/AgentTool.tsx`
  - `src/tools/shared/spawnMultiAgent.ts`
- team persistence
  - `src/utils/swarm/teamHelpers.ts`
- shared task system
  - `src/utils/tasks.ts`
- mailbox and messaging
  - `src/utils/teammateMailbox.ts`
  - `src/tools/SendMessageTool/SendMessageTool.ts`
- permission routing
  - `src/utils/swarm/permissionSync.ts`
  - `src/utils/swarm/inProcessRunner.ts`
- idle and teammate initialization
  - `src/utils/swarm/teammateInit.ts`
- backend selection
  - `src/utils/swarm/backends/registry.ts`
- feature gate
  - `src/utils/agentSwarmsEnabled.ts`

### Required user-facing capabilities

Any Copilot CLI port should expose the equivalent of the following behaviors,
even if the exact command names differ:

- `team create`
  - Creates a leader-managed team and persistent team state.
- `team delete`
  - Deletes team state, but only after active workers have exited or been
    marked inactive.
- `agent spawn`
  - Creates a named worker with inherited execution context.
- `agent list` or `team status`
  - Shows the roster, worker state, backend type, and current activity.
- `task create`, `task list`, `task claim`, `task update`
  - Operates on a shared task queue visible to all team members.
- `message send`
  - Sends durable asynchronous messages by worker name.
- `agent shutdown`
  - Requests graceful worker termination through the coordination layer.

The names are less important than the model: agent teams must remain
leader-controlled, name-addressable, and task-oriented.

### Required persisted state

The port needs durable shared state so workers can coordinate across turns and
across process boundaries.

#### Team record

The team record should persist at least:

- team name
- leader session id
- leader agent id
- member list
- member metadata:
  - internal id
  - human-readable name
  - role or type
  - model
  - cwd or repo root
  - backend type
  - active, idle, or stopped state
  - creation time
  - last heartbeat or last activity time

The most important rule is to preserve both:

- a stable internal unique id for bookkeeping
- a stable human-readable name for routing and UX

Workers should continue to be addressable by name. The user and other workers
must be able to say "send this to tester" or "assign this to reviewer" without
using opaque ids.

#### Shared task store

The task system should remain a first-class coordination primitive, not a UI
extra. The shared task store should keep:

- task id
- subject
- description
- owner
- status
- dependencies or blockers
- arbitrary task metadata

The Copilot CLI port should preserve these behaviors:

- the team shares one task queue
- tasks can be claimed by a worker
- tasks can block other tasks
- task completion is durable
- a new team can start with a fresh task namespace

If the port skips the shared task queue, the feature collapses into loosely
coordinated parallel chat instead of actual multi-agent execution.

#### Mailbox or message bus

The current implementation uses durable file-backed inboxes. A Copilot CLI port
may use files, sqlite, sockets, or another IPC layer, but it still needs
durable asynchronous delivery semantics.

Required message categories:

- plain text worker-to-worker messages
- worker-to-leader messages
- shutdown request and shutdown response
- permission request and permission response
- plan approval response
- idle notifications
- optional system events such as worker crashed or worker resumed

The storage mechanism is replaceable. Durable asynchronous routing is not.

#### Suggested Copilot CLI storage layout

For an MVP, a simple durable layout would be enough. One reasonable shape is:

- `{copilot-state}/agent-teams/{team-name}/team.json`
  - team metadata and member roster
- `{copilot-state}/agent-teams/{team-name}/tasks/`
  - one durable task record per task, or a single append-safe task store
- `{copilot-state}/agent-teams/{team-name}/inboxes/{worker-name}.json`
  - durable mailbox per named worker
- `{copilot-state}/agent-teams/{team-name}/permissions/`
  - pending and resolved permission requests
- `{copilot-state}/agent-teams/{team-name}/runtime/`
  - heartbeats, pid mapping, crash markers, or ephemeral worker state

The exact storage engine can change later. What matters is that workers can
reconstruct shared state after process boundaries, CLI restarts, or short-lived
worker exits.

### Runtime architecture

The recommended Copilot CLI runtime shape is:

- leader session
  - owns user-facing coordination
  - resolves permissions
  - keeps the authoritative team view
- worker runtime
  - executes delegated work
  - reports progress, completion, and idle state
  - does not independently invent its own approval model
- shared state layer
  - stores team records, task state, and mailbox traffic
- backend layer
  - decides whether workers run in-process or as subprocesses
  - is separate from coordination logic

This separation matters. The coordination model should work the same way even
if the worker backend changes later.

### Execution backend guidance

The current code supports tmux, iTerm2, and in-process execution. A Copilot CLI
port should not start by copying the pane-management layer.

Recommended rollout order:

1. MVP
   - in-process workers or subprocess workers only
2. Later
   - richer terminal integration such as tmux or pane orchestration

This keeps the first port focused on correctness and portability instead of
terminal-specific UX.

### Context inheritance requirements

Spawned workers must inherit enough context to behave like a continuation of the
leader session. At minimum, propagate:

- cwd
- repo context
- model selection
- permission mode
- CLI settings or config
- authentication or session context, if Copilot CLI separates that from process
  environment

If workers do not inherit the same execution context, they will make different
tool choices, resolve different files, and break team coordination.

### Permission model requirements

One of the most important architectural constraints to preserve is
leader-mediated permissions.

The expected flow is:

1. a worker reaches a permission boundary
2. the worker sends a permission request to the leader
3. the leader resolves the request through the main approval flow
4. the response is routed back to the worker

This prevents workers from surfacing inconsistent approval prompts or silently
performing privileged actions. If Copilot CLI already has a centralized approval
system, worker actions should plug into that system instead of creating a second
one.

Workers that can edit files, run shell commands, or access the network should
not bypass this approval path.

### Critical runtime flows to implement

#### Team creation flow

1. create a new team record
2. register the leader as the first member
3. initialize task, mailbox, and permission storage
4. mark the leader session as attached to that team

#### Worker spawn flow

1. validate requested worker name
2. create worker metadata with internal id and backend type
3. inherit leader execution context
4. start the worker runtime
5. persist worker state before handing it work

#### Task execution flow

1. leader or worker creates a task
2. a worker claims the task
3. ownership becomes visible in shared state
4. the worker reports progress or completion
5. the worker returns to idle when the turn ends

#### Messaging flow

1. sender resolves a target by worker name
2. the message is durably recorded
3. the recipient observes the message on the next poll or event tick
4. the recipient can acknowledge, act, or respond later

#### Permission flow

1. worker hits a gated operation
2. worker emits a permission request to the leader channel
3. leader resolves through the main CLI approval path
4. leader writes back approval, rejection, or updated policy
5. worker resumes or aborts based on the response

#### Shutdown flow

1. leader or user requests worker shutdown
2. the request is durably recorded and delivered
3. worker exits cleanly or is marked inactive after timeout handling
4. team deletion remains blocked until active workers are gone

### Idle semantics

Worker idle state is part of normal operation. Idle should mean:

- the worker finished its current turn
- the worker is still available
- the worker is waiting for a new task or message

Idle must not be treated as failure, disappearance, or shutdown.

### Shutdown and cleanup requirements

Cleanup must be explicit and graceful.

Required behavior:

- shutdown should be requested, not assumed
- workers should acknowledge or be marked inactive after timeout handling
- team deletion should be blocked while workers are still active
- deleting a team should also clean shared task and mailbox state

This avoids orphaned workers, stale task queues, and inconsistent team state.

### MVP scope for a Copilot CLI port

The minimum useful version should include:

- a feature flag for agent teams
- persistent team records
- leader plus named worker model
- in-process or subprocess worker spawning
- shared task queue
- durable mailbox or equivalent IPC
- worker idle reporting
- graceful shutdown
- leader-routed permissions

The MVP does not need terminal panes, fancy UI badges, or advanced visual
rendering.

### Later-phase scope

After the coordination layer is stable, the port can add:

- richer team roster views
- task progress rendering
- worker message previews
- worker badges or colors where supported
- crash recovery and worker restart logic
- tmux or pane integration

These are improvements to usability, not the architectural core.

### Compatibility notes with the current Claude Code implementation

The following ideas are Claude-specific and should be abstracted rather than
copied directly:

- Anthropic model plumbing
- current AppState and REPL integration
- attachment rendering details
- teammate color and pane decoration logic
- Claude-specific feature gates
- some plan-mode approval hooks

The following ideas are implementation-agnostic and should be preserved as
closely as possible:

- team registry
- shared task queue
- mailbox or message bus
- leader and worker architecture
- permission proxying through the leader
- idle notifications
- graceful shutdown
- backend abstraction

### Recommended implementation priority

The safest order for a GitHub Copilot CLI port is:

1. port the coordination model
   - team registry
   - worker spawn
   - shared tasks
   - durable messaging
   - leader-mediated permissions
   - shutdown flow
2. validate correctness and failure handling
3. add richer UX and terminal integrations later

The core value of agent teams is not the pane UI. It is coordinated execution
with shared durable state and a single approval authority.

## Disclaimer

All code in this repository is the intellectual property of [Anthropic](https://www.anthropic.com/). This repository is provided for **educational and reference purposes only**. Please refer to Anthropic's [license terms](https://www.npmjs.com/package/@anthropic-ai/claude-code/v/2.1.88) for usage restrictions.

This is **not** affiliated with, endorsed by, or supported by Anthropic.
