# snocode: scope

Status: draft v0 (2026-09-20). Items marked **(proposed)** were suggested, not yet confirmed. Everything under "Open questions" is undecided.

## What this is

A fast, minimal terminal coding agent. It reads code, runs commands, edits files, and reports what happened. It is meant to sit next to other small terminal tools, not replace them.

## Why it exists

- Existing agents bundle features I don't use (voice, web UI, IDE and desktop and cloud surfaces, analytics).
- Agents that create their own commits take a decision that belongs to the user.
- After a long chain of commands, output is walls of text where the important part is hard to find.

## Principles

Each principle is followed by what it forces in the design. When a feature request conflicts with one of these, the principle wins.

1. **The user owns git.**
   - The agent has no git-write capability and never commits, branches, or resets.
   - The harness snapshots for undo. The user commits.
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
6. **Modes are permissions, not prompts.**
   - A mode is a bundle of tool permissions, a system prompt, and an output shape.
   - Restrictions are enforced by the harness, not requested politely.
7. **The engine is a library, the TUI is a client.**
   - Commands go in, typed events come out.
   - Headless mode and future integrations come from the same interface.
8. **Minimal by default.**
   - Nothing is added until its absence hurts in real use.
   - Every addition needs a one-line justification.

## Goals (v1)

- Plan mode and build mode, with a persisted plan artifact passed between them.
- A block-based TUI: collapsed by default, expandable, navigable, with jump-to-report.
- Harness-owned undo per turn, plus a tripwire for any change to HEAD or refs.
- One or two providers behind a thin trait, with streaming and tool calling.
- Small tool set: read, grep/glob, edit, shell.
- Session log as an append-only event file, replayable.
- Linux first, macOS second, with CI on both from the first commit.

## Non-goals

Anything here needs an explicit decision to move out of this list.

- Voice input, browser UI, help mode, analytics, telemetry.
- IDE plugins, desktop app, hosted or cloud agent, account system.
- Multiple edit formats. One primary format, at most one whole-file fallback for weak models.
- A large provider abstraction layer.
- Agent-authored commits, in any form.
- Repo map. Deferred until read/grep/glob prove insufficient in real use.
- Windows. Deferred; a much larger jump than macOS.
- Plugin systems or extension APIs **(proposed)**.
- Sub-agents or multi-agent orchestration **(proposed)**.

## Turn structure

- **Intent:** the plan, short, visible before work starts.
  - Plan mode produces it as a persisted artifact: steps, files affected, verification commands, explicit out-of-scope items.
  - Build mode consumes it if it exists, otherwise writes a short one first.
- **Activity:** commands, outputs, and thinking, condensed by default.
  - Commands show as one line: command, exit code, duration.
  - Output shows below, capped, as head and tail with an elided-lines count.
  - Thinking shows as a one-line live summary while streaming, expandable afterward.
- **Report:** the part the user reads and answers. See below.

## Modes

Names are tentative (see open questions).

**Plan**
- Read-only tools. Shell limited to an allowlist, and later a sandbox.
- Longer prose allowed, since it discusses architecture before any code exists.
- Models are discouraged from writing code. Where they do, it shows intent only. Fenced code is size-capped and collapsed in the UI, since this can't be enforced.
- Output is the plan artifact.

**Build**
- Full tool set, more restricted behavior.
- Follows the plan artifact exactly, or writes a short plan first.
- The report includes plan-versus-actual per step, so deviations are surfaced, not buried.

## Report

Two visibly different blocks.

- **Verified (harness):** diffstat from the real filesystem, commands run with exit codes, files touched, changes outside the plan, HEAD and ref tripwire status.
- **Model says:** summary, rationale, deviations, open questions, suggested commit message.

Only the report has an enforced schema. Constraining all output hurts quality, so other output uses a prompted markdown subset (bullets, key-value lines, short tables) plus auto-collapse in the renderer.

**Evidence**
- A claim in the report can cite a stored output block by id, with an optional line range.
- The harness fetches the stored text and renders it verbatim in the verified style. The model never pastes output into the report, because pasted text is a model claim again.
- Cited excerpts follow the same condensation limits as any other block, expandable on demand.
- The verified block still lists every command with its exit code, whatever is cited. Evidence adds to that list and never replaces it, so a passing run cannot stand in for a failing one.
- A citation naming a missing block, or a range outside the output, is flagged instead of rendered.
- Later: attach evidence to individual claims rather than the report as a whole. Mark state claims (tests pass, file changed) that have no evidence. Label evidence that predates the last edit to the files it concerns.

Quick actions on a report: continue, ask a question, undo this turn.

## Git and safety

- The agent gets no git-write tool.
- Before each turn the harness records HEAD and refs. After, it compares. Any change is flagged loudly, with a one-key restore.
- Undo snapshots are tree objects written from a temporary index, stored under a private ref namespace. HEAD, the index, and `git log` are untouched. Snapshotting the whole tree, not just files the edit tool touched, catches changes made through the shell.
- The tripwire is the v1 answer and is not real enforcement. Real enforcement is an OS sandbox with `.git` read-only (also making plan mode's read-only guarantee real). It comes later behind a sandbox interface, with a tripwire-only default backend.

## Platforms

- Linux first, macOS second. Windows is out of scope.
- Platform-specific code lives in one small module. The main such piece is sandboxing, which needs separate backends (Linux and macOS mechanisms differ).
- Tell the model which platform it is on, since GNU and BSD tools behave differently.
- Decide path comparison rules early (macOS filesystems are case-insensitive by default).
- Test keybindings on macOS terminals early, especially anything using Alt.

## Architecture

- Cargo workspace with separate crates: **protocol**, **engine**, **tui**. The compiler enforces that the engine cannot depend on the UI.
- **Protocol:** serializable commands and events. Blocks are first-class, with an id, a kind (thinking, command, output, plan, report), and an open, append, close lifecycle.
- **Session log:** the append-only event file. It provides replay, undo bookkeeping, and full-output storage in one mechanism.
- **Prompts** live as data files, not in code.
- **Provider layer:** a thin trait over streaming HTTP APIs with tool calling.

## Milestones

1. Scope doc, names, license, CI on Linux and macOS.
2. Protocol crate and hand-written fixture event streams (plan turn, multi-command build turn, giant output, failing test, long thinking, report with a deviation).
3. TUI driven by replays only: block rendering, condensation, navigation, report styling. Decide alt-screen versus inline here.
4. Headless engine: one provider, four tools, snapshots, tripwire. Connect the TUI and start using it on real work.
5. Plan mode, plan artifact, build mode with plan-versus-actual.
6. Harness-generated facts in the report, and evidence citations by block id.
7. Sandbox interface and backends, vendored from existing projects.

## Decisions

- **Name:** snocode. The crate name is reserved on crates.io with a 0.0.1 placeholder release.
- **License:** GPL-3.0-only. This is compatible with vendoring Apache-2.0 code, which GPLv2-only would not be. Relicensing later needs consent from every copyright holder, so settle contribution terms (for example, a note in `CONTRIBUTING`) before accepting outside contributions.

## Open questions

- **Second mode name:** "build" or "work."
- **Does build mode pause for approval of a self-written plan,** or proceed visibly and interruptibly?
- **Alt-screen or inline.** Alt-screen allows re-collapsing past output. Inline preserves native scrollback. Current lean: alt-screen plus a transcript dump to stdout on exit.
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
