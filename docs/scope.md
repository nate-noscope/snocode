# snocode: scope

Status: draft v0 (2026-09-21). Items marked **(proposed)** were suggested, not yet confirmed. Everything under "Open questions" is undecided.

## What this is

A fast, minimal terminal coding agent. It reads code, runs commands, edits files, and reports what happened. It is meant to sit next to other small terminal tools, not replace them.

## Why it exists

- Existing agents bundle features I don't use (voice, web UI, IDE and desktop and cloud surfaces, analytics).
- Agents that commit as part of their normal loop take a decision that belongs to the user.
- After a long chain of commands, output is walls of text where the important part is hard to find.

## Principles

Each principle is followed by what it forces in the design. When a feature request conflicts with one of these, the principle wins.

1. **The user owns git.**
   - Commits are not part of the normal workflow. The agent commits only when asked, through a harness-implemented commit tool that shows the message and asks for approval first.
   - The agent has no other git-write tool. Anything done through the shell that moves HEAD or refs is reported as a verified fact.
   - The harness snapshots for undo and reports any HEAD or ref movement.
2. **Every turn has three layers: intent, activity, report.**
   - Intent is what should happen, activity is what is happening, report is what did happen.
   - Only the report is required reading. Layers are visually and structurally distinct.
3. **Verified facts and model claims never look alike.**
   - The harness generates facts (diffstat, commands run and exit codes, files touched, out-of-plan changes).
   - The model writes claims (summary, rationale, deviations, questions).
   - The model may cite evidence but never supply it. Proof is referenced by block id, and the harness supplies the text.
4. **Dense over prose.**
   - Few straight paragraphs, exposed by default.
   - Text is bulleted and packed with essential information.
   - Long paragraphs auto-collapse past a configurable length.
5. **Full output is stored, display is a view.**
   - Nothing is truncated at rest.
   - The UI condenses to a configurable max lines per block.
   - The model gets its own truncation limit, plus a way to fetch more by range.
6. **Modes are data and permissions, not prompts or names.**
   - A mode is a bundle of tools, permissions, a system prompt, and an output shape, defined as data. Ask and work are defaults loaded through the same path as any user-defined mode.
   - The engine never branches on a mode's name. It reads only the mode's permissions and settings.
   - Restrictions are enforced by the harness, not requested politely.
7. **The engine is a library, the TUI is a client.**
   - Inputs go in, typed events come out. "Command" is reserved for shell commands.
   - Headless mode and future integrations come from the same interface.
8. **Minimal by default.**
   - Nothing is added until its absence hurts in real use.
   - Every addition needs a one-line justification.

## Goals (v1)

- Ask and work as default modes, defined as data. Users can define their own from a fixed set of building blocks in global config. A plan is a persisted artifact that a mode can produce and another can consume.
- A block-based TUI: collapsed by default, expandable, navigable, with jump-to-report.
- Harness-owned undo per turn, plus reporting of any change to HEAD or refs.
- One or two providers behind a thin trait, with streaming and tool calling.
- Small tool set: read, grep/glob, edit, shell, and a commit tool that asks for approval.
- Session log as an append-only event file, replayable.
- Linux first, macOS second, with CI on both from the first commit.

## Non-goals

Anything here needs an explicit decision to move out of this list.

- Voice input, browser UI, help mode, analytics, telemetry.
- IDE plugins, desktop app, hosted or cloud agent, account system.
- Multiple edit formats. One primary format, at most one whole-file fallback for weak models.
- A large provider abstraction layer.
- Agent commits as part of the default workflow. Commits happen only on request, with approval.
- Repo map. Deferred until read/grep/glob prove insufficient in real use.
- Windows. Deferred; a much larger jump than macOS.
- Project-level (per-repo) mode files. Modes load from global config only. Deferred because a cloned repository could otherwise ship a mode with broad permissions.
- Scripting or code in modes. Modes are declarative, and a mode cannot define a tool.
- Plugin systems or extension APIs **(proposed)**.
- Sub-agents or multi-agent orchestration **(proposed)**.

## Turn structure

- **Intent:** the plan, short, visible before work starts.
  - A mode that produces plans (ask, by default) can produce it on request as a persisted artifact: steps, files affected, verification commands, explicit out-of-scope items.
  - A mode that consumes plans (work, by default) uses it if it exists, otherwise writes a short one first.
- **Activity:** commands, outputs, and thinking, condensed by default.
  - Commands show as one line: command, exit code, duration.
  - Output shows below, capped, as head and tail with an elided-lines count.
  - Thinking shows as a one-line live summary while streaming, expandable afterward.
- **Report:** the part the user reads and answers. See below.

## Modes

A mode is a named bundle of building blocks, defined as data. Ask and work ship as defaults and are loaded through the same path as any user-defined mode. The engine never branches on a mode's name, so guarantees attach to permissions: any mode with no write access is read-only, whatever it is called. The status line shows the permission summary (for example "ask · read-only"), since a name does not state it.

**Building blocks.** A fixed vocabulary. Modes compose these and nothing else.
- Tools available: read, grep/glob, edit, shell, commit.
- Write scope: none, the whole tree, or specific paths.
- Shell policy: none, allowlist, or full (sandboxed once a sandbox exists).
- Approval policy: which actions stop and ask first.
- System prompt: a data file.
- Plan behavior: whether the mode can produce a plan artifact, consumes one, or requires one.
- Report sections: which model-authored sections it fills.
- Optionally, a model or provider per mode.

**Not configurable by any mode.** This is the trust boundary.
- The verified block of the report, and evidence by reference.
- Snapshots for undo, and the session log.
- Reporting of any HEAD or ref movement.

**Rules**
- Global config only. Project-level mode files are out of scope for now.
- Declarative only. A mode cannot define a tool or run code.
- Inheritance ("like ask, but with this prompt") is optional. Start with copy-and-edit.
- Add building blocks only when needed. Each one is maintenance and documentation.

**Ask (default)**
- Read-only tools. Shell limited to an allowlist, and later a sandbox.
- For discussion: asking what the agent did, requesting revisions, exploring options, and discussing architecture before any code exists.
- Longer prose allowed.
- Models are discouraged from writing code. Where they do, it shows intent only. Fenced code is size-capped and collapsed in the UI, since this can't be enforced.
- Can produce a plan artifact on request. Does not force a plan on every turn.

**Work (default)**
- Full tool set, more restricted behavior. Covers building, fixing, testing, refactoring, and removing.
- Follows the plan artifact exactly if one exists, or writes a short plan first.
- The commit tool is available, and always asks for approval.
- The report includes plan-versus-actual per step, so deviations are surfaced, not buried.

## Report

Two visibly different blocks.

- **Verified (harness):** diffstat from the real filesystem, commands run with exit codes, files touched, changes outside the plan, HEAD and ref status (including commits made through the commit tool).
- **Model says:** summary, rationale, deviations, open questions, suggested commit message.

Only the report has an enforced schema. Constraining all output hurts quality, so other output uses a prompted markdown subset (bullets, key-value lines, short tables) plus auto-collapse in the renderer.

The model-authored part of the report has a configurable maximum length in characters, so it stays readable at a glance. Everything else is available on demand. **(proposed: the cap covers the model block only, since the verified block is generated and structured)**

**Evidence**
- A claim in the report can cite a stored output block by id, with an optional line range.
- The harness fetches the stored text and renders it verbatim in the verified style. The model never pastes output into the report, because pasted text is a model claim again.
- Cited excerpts follow the same condensation limits as any other block, expandable on demand.
- The verified block still lists every command with its exit code, whatever is cited. Evidence adds to that list and never replaces it, so a passing run cannot stand in for a failing one.
- A citation naming a missing block, or a range outside the output, is flagged instead of rendered.
- Later: attach evidence to individual claims rather than the report as a whole. Mark state claims (tests pass, file changed) that have no evidence. Label evidence that predates the last edit to the files it concerns.

Quick actions on a report: continue, ask a question, undo this turn.

## Git and safety

- The agent has one git-write tool, commit. It shows the message and the changes and asks for approval. It exists only in modes that include it, and nothing in the prompts tells the model to commit unless the user asks.
- Before each turn the harness records HEAD and refs. After, it compares. Any movement is reported as a verified fact: which commits were made, and whether they came through the commit tool or the shell. Movement that did not go through the commit tool bypassed approval, so it is flagged loudly, with a one-key restore.
- Undo snapshots are tree objects written from a temporary index, stored under a private ref namespace. HEAD, the index, and `git log` are untouched. Snapshotting the whole tree, not just files the edit tool touched, catches changes made through the shell.
- Undo restores the working tree only. If HEAD moved during the turn, undo says so and offers to move HEAD back with confirmation. It never rewrites history silently.
- The bypass check is the v1 answer and is not real enforcement. Real enforcement is an OS sandbox with `.git` read-only for the agent's shell (also making read-only modes' guarantee real). It comes later behind a sandbox interface, with a detect-and-report default backend. The commit tool runs in the harness, outside the sandbox, so approved commits still work.

## Platforms

- Linux first, macOS second. Windows is out of scope.
- Platform-specific code lives in one small module. The main such piece is sandboxing, which needs separate backends (Linux and macOS mechanisms differ).
- Tell the model which platform it is on, since GNU and BSD tools behave differently.
- Decide path comparison rules early (macOS filesystems are case-insensitive by default).
- Test keybindings on macOS terminals early, especially anything using Alt.

## Architecture

- Cargo workspace with separate crates: **protocol**, **engine**, **tui**. The compiler enforces that the engine cannot depend on the UI. Layout is an open question below.
- **Protocol:** serializable inputs and events. Inputs are what goes into the engine (send a prompt, approve, undo, cancel), and "command" is reserved for shell commands. Blocks are first-class, with an id, a kind (thinking, command, output, plan, report), and an open, append, close lifecycle.
  - It includes an approval request and response pair from the start, because the engine will ask the user things mid-turn (commit approval, plan approval, later sandbox escalation).
  - The crate is plain data plus serde, with no async runtime and no I/O.
- **Session log:** an append-only JSON Lines file, one event per line. The first line is a header record with a schema version, so a reader can read, convert, or clearly refuse an old log. It provides replay, undo bookkeeping, and full-output storage in one mechanism.
- **Modes** are data files loaded through one general path (see Modes).
- **Prompts** live as data files, not in code.
- **Provider layer:** a thin trait over streaming HTTP APIs with tool calling.

## Milestones

1. Scope doc, names, license, CI on Linux and macOS. **Done.**
2. Protocol crate and hand-written fixture event streams (ask turn that produces a plan, multi-command work turn, giant output, failing test, long thinking, report with a deviation).
3. TUI driven by replays only, on alt-screen: block rendering, condensation, navigation, report styling, and the transcript dump.
4. Headless engine: one provider, four tools, snapshots, HEAD and ref movement reporting. Connect the TUI and start using it on real work.
5. Ask and work as default mode data loaded through the general path, then user-defined modes from global config. Plan artifact, and plan-versus-actual in work mode.
6. Harness-generated facts in the report, evidence citations by block id, and the commit tool with approval.
7. Sandbox interface and backends, vendored from existing projects.

## Decisions

- **Name:** snocode. The crate name is reserved on crates.io with a 0.0.1 placeholder release.
- **Workspace layout:** one repository, with the root staying the `snocode` package and `protocol`, `engine`, and `tui` as members under `crates/`. Chosen over a virtual workspace so the README and license ship with the published crate and the binary stays first-class.
  - Sub-crates are separate packages with independent versions, not separate repositories. Their names are reserved on crates.io at 0.0.1 (`snocode-protocol`, `snocode-engine`, `snocode-tui`), with no `publish = false`.
  - A published crate cannot depend on unpublished path crates, so the sub-crates must be published before `snocode` ships a release that uses them; otherwise distribution stays `cargo install --git` or prebuilt binaries. Real code ships as a bumped version.
  - CI runs clippy and tests with `--workspace`, because a workspace with a root package operates on the root package alone by default.
  - Sub-crates omit `readme`: crates.io resolves a sub-crate's relative README links against its own directory (`crates/<name>/`), so pointing them at the root README would break its `LICENSE` link.
- **Publishing:** the published crate carries the root `README.md`, set explicitly as `readme = "README.md"` so a missing file fails the publish instead of silently dropping it. crates.io resolves relative README links when the `repository` host is GitHub, GitLab, or Bitbucket, rewriting them to `blob/HEAD/<path>` in the repo, so `[GPL (version 3.0 only)](LICENSE)` becomes `https://github.com/nate-noscope/snocode/blob/HEAD/LICENSE`. This depends on `repository` staying set and on one of those hosts. Published versions are immutable, and the 0.0.1 placeholder predates the README, so it renders none; the next publish must bump the version.
- **License:** GPL-3.0-only. This is compatible with vendoring Apache-2.0 code, which GPLv2-only would not be. Relicensing later needs consent from every copyright holder, so settle contribution terms (for example, a note in `CONTRIBUTING`) before accepting outside contributions.
- **Commits:** not part of the normal workflow, but allowed when asked. Implemented as a harness commit tool with approval, plus reporting of any commit that bypasses it through the shell. The harness knows exactly what was committed, and the user sees the message first.
- **Modes are data, not built in.** Ask and work are defaults, and the engine never branches on a mode's name. Custom modes come from global config only, and are declarative. The trust boundary (verified block, evidence, snapshots, log, HEAD reporting) is not configurable.
- **Resolved mode permissions are not recorded in the log for now.** Guarded actions record their outcomes (allowed, denied, asked and approved), so the effective policy can be reconstructed from behavior. Revisit if resuming sessions after editing a mode file becomes confusing.
- **Default mode names:** ask (read-only) and work.
  - "Work" over "build" because much of what the agent does is fixing, testing, refactoring, and removing, and because "build mode" reads as running `cargo build` in a Rust tool. "Code" and "write" were rejected for the same narrowness.
  - "Ask" over "plan" and "talk" because the read-only mode is mostly used for asking what the agent did, requesting revisions, and getting suggestions, not formal planning. It is short, a verb like "work", and has a different initial for keybindings.
  - Watch for a naming overlap: "ask a question" is also a quick action on a report, and the agent may ask the user for approval. Keep UI strings for those distinct from the mode name.
  - A plan is an artifact, not a mode.
- **Screen mode:** alt-screen first, inline deferred. Inline can be added later if scrollback and tmux habits are missed.
  - The transcript dump to stdout on exit is built early. It renders blocks to plain lines with no screen, so it doubles as the basis for inline mode and is the simplest thing to snapshot-test.
  - Block rendering is a pure function of the block, a width, an expanded flag, and a line limit. It returns styled lines and does not know where they are drawn.
  - UI state (focus, scroll offset, which blocks are expanded) lives in the TUI layer, never in block data.
  - Widgets do not assume they own the full screen.
  - Before building each interaction, decide what it degrades to in inline mode, so inline does not end up a second-class afterthought.

## Open questions

- **Protocol details, with leans:**
  - Output encoding. Lean: lossy text at capture, with ANSI escape codes handled in the renderer, so line counts ignore them.
  - Block ids. Lean: per-session sequential integers, which keep fixtures readable and citations short.
  - Block structure. Lean: flat blocks with an optional parent id, not a tree.
  - Time. Events carry timestamps or durations, and tests must not depend on the values.
- **Does work mode pause for approval of a self-written plan,** or proceed visibly and interruptibly?
- **Checkpoint storage:** git refs or a separate store.
- **Token counting:** exact per-model tokenizers or a cheap approximation for budgeting.
- **First provider(s).**
- **Git shell-out or a git library.** Shelling out is slower but matches user-visible behavior exactly, and is a reasonable v1.

## Prior art and attribution

- **aider:** edit-matching logic and prompts are worth studying. Apache-2.0.
- **Codex CLI:** the sandbox backends are the main thing to vendor. Apache-2.0.
- **opencode:** compare how it splits plan and build permissions.
- Any ported code keeps its original license header and notice. Ported files are listed in a `NOTICE` file, and changes are marked.
- Before adding a feature, check the non-goals list. If it's there, that's the answer.
