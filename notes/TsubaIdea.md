# Tsuba - Idea

Tsuba provides mechanism to ensure *tool calls* by AI agents make no harm.


## Related Documents

- Spec: `TsubaSpec.md` (not yet created)
- Plan: `TsubaPlan.md` (not yet created)


## C001: Auto mode as a vendor-lock-in factor 💡

Efficient AI usage depends on automatic tool-call gating, but Claude Code's
*auto mode* is backed by a classifier built into Claude models, which locks
non-Claude models and other harnesses out of that capability.

Claude Code supports a feature called [*auto mode*][claude-automode] which
could smartly assess tool calls made by agents, and block dangerous actions if
they would go beyond user requests and/or would make any harm. It is backed by
a classifier model that assesses every tool call before they run, so that user
does not have to respond to repeated permission requests.

However, the classifier model is built into Claude models like Haiku, Sonnet,
Opus, and Fable. This makes using non-Claude model in Claude Code impossible
to use auto mode.

In other AI agents like OpenCode, although they would also support permission
gates for tool calls, many of them manage permissions heuristically or through
inflexible configurations - like Claude Code without *auto mode*.

We can say that one of the key factors for efficient AI usage is auto mode.
Without auto mode, the speed of problem solving and task execution with AI
would degrade a lot. Thus, we need an auto mode alternative, on Claude Code
and/or other AI agent harness. Otherwise, auto mode would be a strong factor
of vendor locking.

[claude-automode]: https://code.claude.com/docs/en/permission-modes#eliminate-prompts-with-auto-mode


## C002: Harness-extension form factor 💡

Tsuba leverages the *harness extension* mechanisms provided by each AI agent
system to add tool-call gating, so its packaging differs per harness.

For example, on Claude Code Tsuba would be a [*Claude Plugin*][claude-plugins]
(bundle of agent skills, scripts, hooks, ..., etc.) that [*hooks*][claude-hooks]
the `PreToolUse` event. In OpenCode, Tsuba would be an OpenCode plugin (a npm
package that runs within OpenCode process) that hooks `tool:execute` event.

Other AI agent harnesses would have different mechanisms to enable
customization. And form factor of Tsuba on them depends on their extension
mechanisms.

[claude-plugins]: https://code.claude.com/docs/en/plugins/
[claude-hooks]: https://code.claude.com/docs/en/hooks
[opencode-plugin]: https://opencode.ai/docs/plugins/


## C007: Harness integration 💡

Each AI agent harness exposes its own extension mechanism for gating tool calls,
so Tsuba's integration is developed and tracked per target harness.

> [!NOTE]
> **Metadata**
> - **Related**: C002

### C003: Auto mode on OpenCode 💡

OpenCode loads TypeScript plugins into its process and fires a
`tool.execute.before` event before each tool call, giving Tsuba a
model-agnostic point to gate actions.

> [!NOTE]
> **Metadata**
> - **Related**: C002

OpenCode extensions are *plugins*: an npm package (or a
`.opencode/plugins/*.ts` file) that runs inside the OpenCode process and
subscribes to lifecycle events. The `tool.execute.before` event fires before a
tool runs, so a plugin can read the tool name and arguments and abort the call;
`tool.execute.after` fires afterwards.

OpenCode also has a permission system with `allow` / `ask` / `deny` effects and
an `--auto` flag that approves everything not explicitly denied. Tsuba supplies
the missing judgment: a classifier on `tool.execute.before` that blocks calls
the static rules cannot assess, so a session can run with `--auto` while
dangerous actions are still caught.

References: [OpenCode plugins][opencode-plugin], [OpenCode permissions][opencode-permissions].

[opencode-permissions]: https://opencode.ai/docs/permissions/


### C004: Auto mode on Claude Code 💡

Claude Code's built-in auto mode is backed by a Claude-only classifier, so a
`PreToolUse` hook that classifies tool calls restores auto mode when a
non-Claude model drives the harness.

> [!NOTE]
> **Metadata**
> - **Related**: C002

Claude Code supports *plugins*, which bundle skills, scripts, and hooks. A
`PreToolUse` hook fires before every tool call, ahead of every permission-mode
check, and can return a `permissionDecision` of `allow`, `deny`, or `ask`; a
`deny` blocks the call even under `bypassPermissions`. That makes `PreToolUse`
the extension point Tsuba uses.

Native auto mode is model-side: a classifier inside Claude models reviews each
call, so it disappears when a non-Claude model or a model router is used. A
harness-side `PreToolUse` hook does not depend on the model, so Tsuba can supply
the classifier as a plugin instead.

References: [Claude Code plugins][claude-plugins], [Claude Code hooks][claude-hooks], [Claude Code permission modes][claude-permission-modes].

[claude-permission-modes]: https://code.claude.com/docs/en/permission-modes


### C005: Auto mode on Codex 💡

Codex exposes `PreToolUse` hooks that can block or rewrite a tool call, giving
Tsuba a model-agnostic gate for auto mode.

> [!NOTE]
> **Metadata**
> - **Related**: C002

Codex has a hooks framework with lifecycle events including `PreToolUse`,
`PostToolUse`, and `PermissionRequest`. `PreToolUse` fires before Bash commands,
`apply_patch` file edits, MCP tool calls, and other local function tools run. A
hook returns `permissionDecision: "deny"` to block a call, or
`permissionDecision: "allow"` with `updatedInput` to rewrite it. Hooks are
declared in Codex's hooks configuration and may be command hooks or MCP-tool
hooks.

Tsuba supplies a `PreToolUse` command hook that classifies each call and denies
the harmful ones, so Codex can run without per-call prompts while dangerous
actions are still gated.

References: [Codex hooks][codex-hooks].

[codex-hooks]: https://developers.openai.com/codex/hooks


### C006: Auto mode on Pi / Oh My Pi 💡

Pi and its Oh My Pi fork are minimal harnesses shaped by TypeScript extensions
that can observe and block tool calls, which is where Tsuba gates auto mode.

> [!NOTE]
> **Metadata**
> - **Related**: C002

Pi is a minimal agent harness designed to be adapted by *extensions*: TypeScript
modules loaded from `~/.pi/agent/extensions/` (global) or `.pi/extensions/`
(project), and shareable as packages over npm or git. An extension subscribes to
lifecycle events; the tool-call event can return a `{ block: true, reason }`
result to stop a tool before it runs.

Oh My Pi (`omp`) is a fork of Pi by Mario Zechner maintained by can1357, rebuilt
as a coding-first surface while keeping the same TypeScript extension model, so
a Tsuba extension written against Pi's extension API applies to both. The Pi
ecosystem already hosts third-party hook packages, including one that adapts
Claude Code's hook format.

References: [Pi][pi], [Pi extensions][pi-extensions], [Oh My Pi][omp].

[pi]: https://pi.dev/
[pi-extensions]: https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md
[omp]: https://github.com/can1357/oh-my-pi


## C008: Tool assessment 💡

Different AI tools have different protocols, usage, and concerns, so Tsuba
assesses each kind of typical tool use under a *generic* name rather than by a
harness-specific tool.

> [!NOTE]
> **Metadata**
> - **Related**: C007

Harnesses name and invoke tools differently — Claude Code's `Read` and
OpenCode's `read` are the same kind of action, as are Claude Code's `Bash` and
Codex's shell execution. Assessment is defined once per generic tool so the
rules are shared across harnesses and only the name mapping stays
harness-specific.

### C009: glob-tool 💡

File discovery by path pattern.

Examples: Claude Code `Glob`, OpenCode `glob`.

Read-only. Assessment centres on path scope — whether the pattern reaches
outside the working directory or enumerates sensitive locations. Only file
names reach the model, not contents.

### C010: grep-tool 💡

Content search across files.

Examples: Claude Code `Grep`, OpenCode `grep`.

Read-only. Assessment centres on path scope and on whether the query would
surface sensitive material (keys, tokens) into context.

### C011: read-tool 💡

Read a file's contents.

Examples: Claude Code `Read`, OpenCode `read`.

Read-only, but can pull sensitive material (credentials, `.env`, private keys)
into context. Assessment centres on path scope and on sensitive file patterns.

### C012: edit-tool 💡

Create, overwrite, or patch file contents.

Examples: Claude Code `Edit` / `Write`, OpenCode `edit` / `write` / `patch`,
Codex `apply_patch`.

A write action. Assessment centres on the target path (inside the project vs
outside it, or a protected file) and on the nature of the change (destructive
overwrite vs additive edit).

### C013: exec-tool 💡

Run a shell command.

Examples: Claude Code `Bash`, OpenCode `bash`, Codex `codex exec`.

The highest-risk kind — arbitrary command execution. Assessment centres on the
command's effect: destructive operations, data exfiltration, privilege
escalation, and whether it runs outside the sandbox.


## C014: Assessment method 💡

There are several ways to judge whether a tool call is harmful, and different
situations favour different ones.

> [!NOTE]
> **Metadata**
> - **Related**: C008

The methods differ in cost, latency, and accuracy, and none is universally
best; the choice depends on how much scrutiny the situation warrants.

### C015: 3rd-party assessment tool 💡

Delegate the judgment to an existing third-party tool or service that already
assesses tool calls.

Reuses available tooling instead of building a classifier. Trade-off: adds an
external dependency and its cost and latency, but inherits whatever policy and
coverage that tool provides.

### C016: Low-cost AI with special prompt 💡

Classify each call with a cheap model driven by a purpose-built prompt.

A small or inexpensive model rates the call against a rubric carried in the
prompt, tailored per generic tool. Trade-off: moderate cost and latency, with
quality bound by the prompt and the model's capability.

### C017: Keyword-catcher dumb assessment 💡

Flag calls with a trivial keyword or pattern matcher, kept mainly for testing.

A substring or regex check that catches obviously dangerous tokens without real
judgment. Not a serious classifier; its value is as a baseline and test
stand-in so the harness plumbing can be exercised before a smarter method is
wired in.


## Metaphor

According to [Tsuba](https://en.wikipedia.org/wiki/Japanese_sword_mountings#Tsuba)
on Wikipedia:

> The tsuba (鍔; or 鐔) is usually a round (or occasionally squarish) guard at
> the end of the grip of bladed Japanese weapons, like the katana and its
> variations, tachi, wakizashi, tantō, naginata etc. They contribute to the
> balance of the weapon and to the protection of the hand. The tsuba was mostly
> meant to be used to prevent the hand from sliding onto the blade during
> thrusts as opposed to protecting from an opponent's blade.
