<p align="center">
  <em>A long-running AI session is a Turing-complete computer. <strong>Prose is a programming language for it.</strong></em>
</p>

<p align="center">
  <a href="skills/prose/prose.md">Language spec</a> •
  <a href="skills/prose/compiler.md">Compiler</a> •
  <a href="skills/prose/examples/">Examples</a> •
  <a href="skills/prose/lib/">Standard library</a>
</p>

<p align="center">
  <strong>⚠️ Beta Software</strong> — <a href="#legal">Read before using</a>
</p>

---

```prose
# Research and write workflow
agent researcher:
  model: sonnet
  skills: ["web-search"]

agent writer:
  model: opus

parallel:
  research = session: researcher
    prompt: "Research quantum computing breakthroughs"
  competitive = session: researcher
    prompt: "Analyze competitor landscape"

loop until **the draft meets publication standards** (max: 3):
  session: writer
    prompt: "Write and refine the article"
    context: { research, competitive }
```

## What it is

**OpenClaw Prose** is the `prose` skill for [OpenClaw](https://github.com/openclaw). It's a structured language for orchestrating AI agents from inside an OpenClaw agent session. You declare agents, control flow, and intent — the running OpenClaw session becomes the interpreter and wires the rest up. The session itself is the Inversion-of-Control container.

This repo is the OpenClaw-native continuation of [openprose/prose](https://github.com/openprose/prose).

### Fork point

Forked from upstream at **[v0.7.1](https://github.com/openprose/prose/releases/tag/v0.7.1)** (January 2026). Since then, upstream has pivoted away from the embodied in-session VM model that this fork keeps:

- **[skill-v0.15.0](https://github.com/openprose/prose/releases/tag/skill-v0.15.0)** — *Intelligent React overhaul*: the in-session `judge → verdict → pressure → fulfillment` loop was replaced by a deterministic reconciler (`runtime_contract: 1 → 2`).
- **[v0.16.0](https://github.com/openprose/prose/releases/tag/v0.16.0)** — the Reactor harness moved to [openprose/reactor](https://github.com/openprose/reactor); `prose react`, the legacy `@openprose/prose-cli`, and `tools/cli/` were removed.
- **[v0.17.0](https://github.com/openprose/prose/releases/tag/v0.17.0)** — added guided `prose init` and `prose compose` over an obligation-centered `std/ops/compose` package layout.

This fork keeps the v0.7.1 **embodied in-session VM**. The **core language surface** — agents, sessions, control flow, `**...**` fourth wall, pipelines, blocks, persistence — carries forward, as do the filesystem / sqlite / postgres state backends. Anything from upstream's post-v0.7.1 direction doesn't apply here: not the `runtime_contract: 2` reconciler, not the obligation-centered `std/ops/compose` authoring layout, not `prose init` / `prose compose`, not the `### Maintains` / `### Requires` contract blocks, not the `prose compile` → `prose serve` → `prose run` host-process topology. Treat `.prose` programs written here as **language-compatible with v0.7.1** — not with anything upstream released after January 2026. The runtime mapping and the standard library are OpenClaw-specific. For upstream's later work, see the [CHANGELOG](https://github.com/openprose/prose/blob/main/CHANGELOG.md).

## Install

Clone into your OpenClaw skills directory and reload:

```bash
git clone https://github.com/BottaniCals/openclaw-prose.git \
  ~/.openclaw/skills/prose
```

Then try a sample program from inside the cloned skill:

```
cd ~/.openclaw/skills/prose
prose run examples/01-hello-world.prose
```

> **By installing, you agree to the [Terms of Service](TERMS.md).**

## The OpenClaw Runtime Mapping

The upstream OpenProse spec is harness-agnostic. Under OpenClaw, the VM uses these primitives:

| OpenProse concept        | OpenClaw runtime                                            |
| ------------------------ | ----------------------------------------------------------- |
| Spawn a subagent         | `sessions_spawn(task, label, runtime, model, agentId)`      |
| Read / write files       | `read` / `write`                                            |
| Shell execution          | `exec`                                                      |
| Fetch a URL              | `web_fetch`                                                 |
| Model name (`sonnet` etc)| Any OpenClaw-registered model identifier                    |

You never call these yourself when writing `.prose` files. The `prose` VM handles every `sessions_spawn` and every tool boundary. Treat the table above as the substrate — useful for debugging and for understanding what a `session` actually does.

### What changed in the OpenClaw fork

- Skill renamed `open-prose` → **`prose`**. Update any activation rules or marketplace config accordingly. The legacy `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` still label themselves `open-prose` — update their plugin entries if you're publishing through the plugin marketplace.
- Skill files live under `skills/prose/`.
- Remote registry (`@handle/slug`) and direct URL fetching are gone. The `use` statement now resolves only to **local file paths**.
- Telemetry removed — nothing leaves your machine.
- New `lib/` standard library (see below).
- User-scoped agent persistence at `~/.prose/agents/` and `~/.prose/agents.db`.

## The Intelligent Inversion of Control

Traditional orchestration requires explicit coordination code. Prose inverts this — you declare agents and control flow, and the OpenClaw session wires them up.

### 1. The session as runtime

Other frameworks orchestrate agents from outside. Prose runs *inside* an OpenClaw agent session — the session itself is both interpreter and runtime. It doesn't just match names; it understands context and intent.

### 2. The fourth wall (`**...**`)

When you need AI judgment instead of strict execution, break out of structure:

```prose
loop until **the code is production ready**:
  session "Review and improve"
```

The `**...**` syntax lets you speak directly to the VM. It evaluates this semantically — deciding what "production ready" means based on context.

### 3. Structure + flexibility

**Why not just plain English?** You can — that's what `**...**` is for. But complex workflows need unambiguous structure for control flow. The AI shouldn't have to guess whether you want sequential or parallel execution.

**Why not rigid frameworks?** They're inflexible. Prose gives you structure where it matters (control flow, agent definitions) and natural language where you want flexibility (conditions, context passing).

## Language features

| Feature           | Example                                                                |
| ----------------- | ---------------------------------------------------------------------- |
| Agents            | `agent researcher: model: sonnet`                                      |
| Sessions          | `session "prompt"` or `session: agent`                                 |
| Persistent agents | `persist: true` / `persist: user` / `persist: project` / `resume:`     |
| Parallel          | `parallel:` blocks with join strategies (`first`, `any`, `count`)      |
| Variables         | `let x = session "..."` / `const y = session "..."`                    |
| Context           | `context: [a, b]` or `context: { a, b }`                               |
| Fixed loops       | `repeat 3:` and `for item in items:`                                   |
| Unbounded loops   | `loop until **condition**:`                                            |
| Error handling    | `try` / `catch` / `finally`, `retry` with `backoff`                    |
| Pipelines         | `items \| map: session "..."`                                          |
| Conditionals      | `if **condition**:` / `choice **criteria**:`                           |
| Blocks            | `block name(params):` / `do name(args)`                                |
| Imports           | `use "./lib/inspector.prose"` — local file paths only                  |

See the [Language reference](skills/prose/compiler.md) for the full grammar.

## Standard library

`skills/prose/lib/` ships production-quality `.prose` programs. Import with `use`:

```prose
use "./lib/inspector.prose"
use "./lib/user-memory.prose"
```

### Evaluation & improvement

| Program                  | Purpose                                                                |
| ------------------------ | ---------------------------------------------------------------------- |
| `inspector.prose`        | Post-run analysis — runtime fidelity and task effectiveness            |
| `profiler.prose`         | Performance profiling and bottleneck identification                    |
| `vm-improver.prose`      | Reads inspections and proposes PRs to improve the VM                   |
| `program-improver.prose` | Reads inspections and proposes PRs to improve your `.prose` source     |
| `cost-analyzer.prose`    | Token usage and cost pattern analysis                                  |
| `calibrator.prose`       | Validates light evaluations against deep evaluations                   |
| `error-forensics.prose`  | Root-cause analysis for failed runs                                    |

### Memory

| Program                | Purpose                                          | Recommended backend |
| ---------------------- | ------------------------------------------------ | ------------------- |
| `user-memory.prose`    | Cross-project personal memory (`persist: user`)  | `--state=sqlite+`   |
| `project-memory.prose` | Project-scoped institutional memory (`persist: project`) | `--state=sqlite+` |

```bash
prose run skills/prose/lib/inspector.prose
prose run skills/prose/lib/user-memory.prose --state=sqlite+
```

The improvement loop:

```
   Run Program  ──►  Inspector  ──►  VM-Improver ──► PR
        ▲                │
        │                ▼
        │         Program-Improver ──► PR
        └────────────────┘
```

`cost-analyzer`, `calibrator`, and `error-forensics` are the supporting cast — where the money goes, whether cheap evaluations proxy for expensive ones, and why a run failed.

## Examples

48 example programs in `skills/prose/examples/`:

| Range  | Category                                                                            |
| ------ | ----------------------------------------------------------------------------------- |
| 01-08  | Basics (hello world, research, code review, debugging)                              |
| 09-12  | Agents and skills                                                                   |
| 13-15  | Variables and composition                                                           |
| 16-19  | Parallel execution                                                                  |
| 20-21  | Loops and pipelines                                                                 |
| 22-23  | Error handling                                                                      |
| 24-27  | Advanced (choice, conditionals, blocks, interpolation)                              |
| 28     | Gas Town (multi-agent orchestration)                                                |
| 29-31  | Captain's chair (persistent orchestrator)                                           |
| 33-38  | Production workflows (PR auto-fix, content pipeline, feature factory, bug hunter, The Forge, skill scan) |
| 39     | Architect by simulation                                                             |
| 40-43  | Recursive language models (RLM)                                                    |
| 44-49  | Meta / self-hosting (UX test, plugin release, workflow crystallizer, language self-improvement, habit miner, run retrospective) |

Start with `01-hello-world.prose`. When you're ready to see the language flex, try `37-the-forge.prose` — AI builds a browser from scratch.

## State backends

Pick a state mode per run with `--state=<mode>`:

| Mode         | When to use                                                  | State location               |
| ------------ | ------------------------------------------------------------ | ---------------------------- |
| `filesystem` | Default. Complex programs, resumption, debugging             | `.prose/runs/{id}/`          |
| `in-context` | Simple programs (<30 statements), no persistence              | Conversation history         |
| `sqlite`     | Queryable, transaction-safe (experimental)                   | `.prose/runs/{id}/state.db`  |
| `sqlite+`    | Memory programs (`user-memory`, `project-memory`)            | `~/.prose/agents.db`         |
| `postgres`   | True concurrent writes, external dashboards (experimental)   | PostgreSQL (BYO database)    |

`sqlite` and `sqlite+` need the `sqlite3` CLI; `postgres` needs `psql`. Without them, fall back to `filesystem`.

## How it works

### The Prose VM

LLMs are simulators. Given a detailed system description, they don't just describe it — they *simulate* it. The `prose.md` specification describes a virtual machine with enough fidelity that a Prose-capable OpenClaw session reading it *becomes* that VM. Each `session` triggers a real `sessions_spawn`, outputs are real artifacts, and state persists in conversation history or files. Simulation with sufficient fidelity is implementation.

| Aspect               | Behaviour                                                  |
| -------------------- | ---------------------------------------------------------- |
| Execution order      | **Strict** — follows the program exactly                   |
| Session creation     | **Strict** — creates what the program specifies            |
| Parallel coordination | **Strict** — executes as specified                        |
| Context passing      | **Intelligent** — summarizes/transforms as needed          |
| Condition evaluation | **Intelligent** — interprets `**...**` semantically        |
| Completion detection | **Intelligent** — determines when "done"                  |

### Documentation files

| File                                  | Purpose                              | When to load                                |
| ------------------------------------- | ------------------------------------ | ------------------------------------------- |
| `skills/prose/prose.md`               | VM / interpreter                     | Always, to run programs                     |
| `skills/prose/compiler.md`            | Compiler / validator                 | Only when compiling or validating           |
| `skills/prose/help.md`                | Help, onboarding                     | On `prose help`                             |
| `skills/prose/state/filesystem.md`    | File-based state (default)           | With the VM                                 |
| `skills/prose/state/in-context.md`    | In-context state                     | On request                                  |
| `skills/prose/state/sqlite.md`        | SQLite state (experimental)          | On `--state=sqlite` or `--state=sqlite+`    |
| `skills/prose/state/postgres.md`      | PostgreSQL state (experimental)      | On `--state=postgres`                       |
| `skills/prose/guidance/patterns.md`   | Best practices                       | When writing new `.prose`                   |
| `skills/prose/guidance/antipatterns.md` | What to avoid                     | When writing new `.prose`                   |
| `skills/prose/guidance/system-prompt.md` | VM-dedicated system prompts      | When launching a fresh VM sub-session       |
| `skills/prose/primitives/session.md`   | Session context & compaction guidelines | When tuning how sessions carry context |
| `skills/prose/alts/*.md`              | Narrative style packs                | Optional flavor for the VM                  |
| `skills/prose/lib/`                   | Standard library programs            | When you want to `use` one                  |

## FAQ

**Why not LangChain / CrewAI / AutoGen?**
Those are orchestration libraries — they coordinate agents from outside. Prose runs inside the OpenClaw session — the session itself is the IoC container. Zero external dependencies, no SDK lock-in.

**Why not just plain English?**
You can use `**...**` for that. But complex workflows need unambiguous structure for control flow — the AI shouldn't guess whether you want sequential or parallel execution.

**What's "intelligent IoC"?**
Traditional IoC containers (Spring, Guice) wire up dependencies from configuration. Prose's container is an OpenClaw agent session that wires up agents using *understanding*. It doesn't just match names — it understands context, intent, and can make intelligent decisions about execution.

**Why OpenClaw-specific?**
OpenClaw has first-class primitives (`sessions_spawn`, `read`/`write`, `exec`, `web_fetch`) that map cleanly onto the v0.7.1 embodied VM. So this fork ships with a tighter runtime mapping and an OpenClaw-native standard library. We deliberately stayed on the pre-pivot embodied model — the post-v0.7.1 reconciler (`runtime_contract: 2`) and obligation-centered authoring layout don't fit how OpenClaw sessions are structured.

**Can I run a `.prose` file written against modern upstream?**
No, not unmodified. Programs written against the `runtime_contract: 2` reconciler, that use the `std/ops/compose` package layout, or that depend on `prose init` / `prose compose` / the `### Maintains` / `### Requires` contract blocks need upstream's reconciler topology. Programs written against the v0.7.1 embodied surface (agents, sessions, control flow, `**...**`, pipelines, blocks) run here with the OpenClaw runtime mapping applied on top. If a program needs the pivot-era model, use upstream.

**Where's my agent state?**
- **Project-scoped** agents live under `.prose/agents/` in your working directory.
- **User-scoped** agents live under `~/.prose/agents/` (and `~/.prose/agents.db` for `--state=sqlite+`).

## Legal

OpenClaw Prose is in **beta**. Expect rough edges. Report issues at [github.com/BottaniCals/openclaw-prose/issues](https://github.com/BottaniCals/openclaw-prose/issues).

You are responsible for all actions performed by AI agents you spawn through Prose. Review your `.prose` programs before execution and verify all outputs.

- [MIT License](LICENSE)
- [Terms of Service](TERMS.md)
- [Contributing](CONTRIBUTING.md)
