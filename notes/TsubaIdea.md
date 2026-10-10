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


## Prior arts

Related work to study one by one; each study informs Tsuba's design.

### S001: pi-automode 💡

Claude Code-style auto mode for Pi, built as an in-process guardrail
extension that intercepts agent tool calls before execution.

> [!NOTE]
> **Metadata**
> - **Related**: C006, C014

Supporting analysis for Tsuba's Pi integration (C006) and assessment
design (C014); it does not make the choice.

It hooks Pi's `tool_call` event and blocks a call on a deterministic
rule, a hard-deny check, or a model classifier's verdict, driven by JSON
configuration. Its guardrail shape is a close reference for the Pi
integration Tsuba contemplates.

**Pipeline.** Each call climbs an ordered ladder from certain-and-cheap
to expensive: auto-mode-enabled check; cancelled-turn block;
`permissions.deny` match (block locally, no model); `permissions.ask`
match (ask the user — acceptance still forces classifier review);
deterministic hard-deny checks; path-deny checks including symlink
aliases and recursive-search scope; allow tiers
(`allowInsideWorkingDirectory`, `permissions.allow`, and a read-only
fast path over `read` / `grep` / `find` / `ls`); then the classifier
sees whatever remains. Explainable rules decide first; the model only
handles the residue.

**Assessment.** The classifier is two-stage and fail-closed: a one-token
filter returns exactly `0` (allow) or `1` (review), and only a `1`
triggers a structured call that must return one internal
`classifier_decision` tool call (`decision` allow/block; `tier`
hard_deny/soft_deny/allow/explicit_intent/none; `reason`). Sampling is
schema-constrained where the provider supports it, validated locally
otherwise. Any failure — no model, provider error, timeout, context
overflow, malformed output — blocks the call. The model is selectable
and can run at low reasoning effort: this is the low-cost-AI-with-a-
prompt assessment method (C016) in production.

**Policy tiers.** Rules are graded — `hard_deny` unconditional,
`soft_deny` overridable by an `allow` exception or explicit user intent,
`allow` exceptions applying to soft denies only. Tool-level
`permissions.deny` / `ask` / `allow` are separate from the prose policy
and parse Bash through `unbash`, so a command chain cannot smuggle an
uncovered command past an allow pattern.

**Trust and limits.** Configuration is source-ranked, and a checked-in
shared project file can only tighten auto mode, never loosen it. The
extension states plainly that it is not a sandbox and does not guard the
user's own `!` shell commands; Claude Code's real classifier and rules
are private, so this reimplements the documented behaviour.

**Bearings for Tsuba.** The deterministic-first/model-last ladder, the
fail-closed contract, the cheap-filter → classifier split, and the
per-generic-tool routing (read-only fast path vs. write/edit and shell
through the model) map onto the tool-assessment (C008) and
assessment-method (C014) concepts.

References: [pi-automode][pi-automode],
[classifier flow][pa-classifier-flow],
[configuration][pa-configuration], [defaults][pa-defaults].

[pi-automode]: https://github.com/czottmann/pi-automode
[pa-classifier-flow]: https://github.com/czottmann/pi-automode/blob/main/docs/automode-classifier-flow.md
[pa-configuration]: https://github.com/czottmann/pi-automode/blob/main/docs/configuration.md
[pa-defaults]: https://github.com/czottmann/pi-automode/blob/main/docs/defaults.md

### S002: cc-safety-net 💡

A deterministic, static pre-execution policy gate that blocks destructive
commands and sensitive-file access before a tool call runs, shipped as thin
adapters across many coding agent CLIs.

> [!NOTE]
> **Metadata**
> - **Related**: C003, C004, C005, C006, C014, C015

Supporting analysis for Tsuba's per-harness integration (C003, C004, C005,
C006), assessment method (C014), and third-party-assessment-tool option
(C015); it does not make the choice.

It parses each tool call before execution and denies the destructive ones,
with no model in the loop. Its breadth — sixteen supported agents, four
integration models — is a close reference for how Tsuba reaches each harness
and for what a deterministic assessment layer looks like.

**Shape.** CC Safety Net is not a classifier and not a sandbox. It reads a
tool call, parses the command with its own POSIX and PowerShell parsers, and
denies calls that match destructive-command or sensitive-path rules. It
states plainly that it does not contain processes, set filesystem
permissions, or filter egress, and it lists its residual bypasses (behaviour
inside arbitrary binaries, `eval` and fetched code, interpreter long-form
flags, a symlink TOCTOU window) as known limitations.

**Integration.** It reaches sixteen agents through four models, which map
directly onto Tsuba's harness-integration concepts. *Stdin hook subprocess* —
the agent runs `cc-safety-net hook <flag>` per call and reads an
agent-specific deny object from stdout — covers Claude Code (`PreToolUse`),
Codex (`PreToolUse`), and nine others (C004, C005). *Agent-loaded plugin*
covers OpenCode via the `@opencode-ai/plugin` contract and a
`tool.execute.before` hook (C003). *In-process extension* covers Pi through
`pi.on('tool_call')` (C006). *Event plugin* covers Amp Code. Each adapter
only normalizes the payload shape and the deny format; the guard itself is
shared, with no per-agent branch. It is also callable as a library
(`checkCommand`) without installing a hook.

**Guard pipeline.** Every call runs one fixed ordered sequence: bounded
tool-input extraction (bounds fail closed), semantic facts, parser budget
checks, **policy-file protection**, **Git-metadata protection**, policy
snapshot load, **sensitive-path protection**, non-command allow, empty-command
fail-closed, then destructive-command analysis. Two properties stand out.
First, the always-on protections (policy files in both scopes, Git metadata)
deny *before* the policy snapshot is loaded, so no preset or override can
switch them off. Second, sensitive-path protection sits *after* the snapshot
and is the only one of the hard-stop protections policy can tune. A thrown
error anywhere in the guard becomes a fail-closed deny attributed to its
stage; an invalid *configuration* source is dropped, not converted into a
denial.

**Assessment.** Stage 12 is the analyzer: it splits a command on shell
operators, walks each segment, strips env assignments and known wrappers, and
dispatches by head command to per-command analyzers (`git`, `rm` /
`Remove-Item`, `find` / `xargs` / `parallel`, device tools), recursing into
shell and interpreter bodies with a depth cap. It tracks `cd` / `pushd` and
propagates environment assignments. This is the deterministic pole of C014:
the opposite of S001's low-cost-model classifier (C016). Semantic parsing is
chosen over wildcard deny rules precisely to resist flag reordering, shell
wrapping, and interpreter hiding.

**Policy.** Rules come from built-in rules, custom rulebooks (fetched and
vendored, integrity-checked, strictly additive), and overrides. Three presets
— standard, strict, paranoid — trade coverage for interruption, with separate
capabilities (`fail_closed`, `paranoid_rm`, `paranoid_interpreters`). Secret
protection runs before command analysis and covers shell, path, search, and
patch shapes through a 138-rule built-in set plus configured deny paths, with
a credential tier on by default. An agent's denial is phrased to keep it on
task, carrying an intent that names a safer alternative.

**Trust and limits.** `policy.json` is protected in both the user and project
scopes (though the project chain stops at its own `.cc-safety-net` directory).
Custom rules can only tighten, never loosen. Several hosts are fail-open by
design (Grok Build, Factory Droid, Devin CLI), and per-host gaps are
documented (Codex `write_stdin` and `workdir`, OpenClaw session cwd, DeepSeek
Harness `run_code`). It positions itself as one layer in a defense-in-depth
stack alongside deny rules and OS sandboxing, not a replacement for either.

**Bearings for Tsuba.** Three takeaways. The four integration models are a
ready map of how each target harness is reached (C003–C006), and its
OpenCode/Pi coverage confirms those extension points are viable for a
model-agnostic gate. Its fail-closed-on-guard-error / fail-open-on-config
split, and the always-on-protections-before-config ordering, are policy
shapes worth mirroring. And it is a concrete instance of the third-party
assessment tool option (C015): Tsuba could depend on it as the deterministic
layer rather than reimplement parsing, while a classifier (C016) handles the
residue deterministic rules cannot judge.

References: [cc-safety-net][ccsn], [architecture][ccsn-architecture],
[integration architecture][ccsn-integration], [design principles][ccsn-design],
[known limitations][ccsn-limits], [modes][ccsn-modes],
[secret protection][ccsn-secret].

[ccsn]: https://github.com/kenryu42/cc-safety-net
[ccsn-architecture]: https://ccsafetynet.com/docs/guides/architecture
[ccsn-integration]: https://ccsafetynet.com/docs/guides/integration-architecture
[ccsn-design]: https://ccsafetynet.com/docs/guides/design-principles
[ccsn-limits]: https://ccsafetynet.com/docs/guides/known-limitations
[ccsn-modes]: https://ccsafetynet.com/docs/configuration/modes
[ccsn-secret]: https://ccsafetynet.com/docs/reference/secret-protection

### S003: pi-automode-classifier 💡

A Pi auto-mode extension that decides each tool call by deterministic rules
first and a small classifier model last, prompting the user for the risky
remainder.

> [!NOTE]
> **Metadata**
> - **Related**: C006, C008, C014, C016

Supporting analysis for Tsuba's Pi integration (C006), generic tool
assessment (C008), assessment method (C014), and the low-cost-AI-with-a-prompt
option (C016); it does not make the choice.

It is the clearest production example of the classifier method (C016) on a Pi
target, and it shows how the method is pinned to a model and threshold rather
than left abstract. Built for the same reason as Tsuba — Pi runs every tool
call without approval, and the author wanted Claude Code's auto mode — it
differs from S001 in focus: S001 is a broad rule-based auto-mode extension,
while this one centres on shell-command classification behind a small,
swappable model.

**Shape.** Three actors decide, in order. Built-in rules parse the shell
command — including `$(...)`, `bash -c` strings, and `sudo` / `timeout` /
`uv run` wrappers — and return one of four verdicts: `deny` (blocked, no
prompt), `ask` (prompt), `allow`, or `classify` (hand to the model). The model
sees only the `classify` commands and answers one yes/no question with a
probability; below `askAbove` the command runs, at or above it the user is
prompted, and if the model cannot answer (down, timeout, over context) the
user is prompted too. The model never blocks anything by itself. Like S001, it
is not a sandbox and does not guard the user's own `!` commands.

**Assessment.** This is the low-cost-model method (C016) in concrete form. The
classifier is swappable across three shapes: a local llama.cpp server (Kev
0.8B, Laya, Julia-1), a hosted model through Pi's own model registry (Jev from
TypeSafe, over OpenRouter), or an arbitrary process speaking a JSON
line protocol on stdin/stdout (LANCET Nano). `askAbove` is tuned per model,
and `classifier.model` is pinned so a silently swapped model forces every
classify command to a prompt rather than answering with another model's
threshold. The prompt itself is a rubric — a main yes/no question with `risky`
and `safe` descriptions, optionally replaced by graded `levels`, plus optional
`extraQuestions` each with its own `askAbove`. It carries its own 50-command
benchmark and reports AUC per model, and it documents that its author wrote
the benchmark while writing the rules — a sanity check, not an accuracy
claim.

**Tool routing.** `bash` (and any tool with a string `command`) goes through
the rules and model. `write` and `edit` are allowed inside the working
directory and ask outside it. `read` and `grep` are not checked except for
credential files: a read-only `cat ~/.aws/credentials` still asks, because the
contents reach the transcript. Credential detection is by path (any home,
`.ssh`, `.aws`, keyrings, shell history, `/etc/shadow`) and by name (`.env`
variants, `id_rsa`, `*.pem`, `*.key`, `secrets.yaml`, service-account JSON);
naming-only commands (`ls`, `stat`, `test -f`, `chmod`) and writes do not
ask. The working directory is the Pi run directory plus its git checkout and
existing worktrees.

**Script provenance.** Because `./scripts/build.sh` says nothing about what it
does, the rules judge by where the file came from: a project file or a
script the agent wrote this session runs; a file `curl` / `wget` saved this
session asks; inline code, new files of another repo, and any script with
`deploy` / `publish` / `release` / `upload` / `prod` go to the model. This
provenance rule is a notable refinement over a pure path-scope check.

**Prompts and learning.** Every prompt offers *Allow once*, *Allow for this
session*, and *Block*. Two kinds widen the choice so the next similar call is
not asked again: an unknown command is offered a pattern with its subcommand
or first flag (`Allow "terraform plan *" for this session`, optionally saved to
config), and an out-of-directory change is offered the whole folder. Inline
code and the built-in ask rules are never offered a pattern. Rules can also be
supplied as `deny` / `ask` / `classify` / `allow` patterns, matched per simple
command (not the whole line), with the most severe match winning and
`allow` rules unable to override the built-in denies, the extension's
own-file prompt, or out-of-directory redirects.

**Trust and limits.** No UI (print mode, most subagents) means there is nobody
to ask, so those calls are blocked by default (`withoutUi` can be set to
`allow`). The model sees only the command and the working directory, so it
cannot tell whether the user asked for the action. Script files are never
read. Paths or programs held in shell variables cannot be resolved, so they
ask or go to the model. Long commands always prompt — and a bigger context
does not help, as the author measured a 3,700-character `rm -rf ~/Documents`
tail scoring 0.15. Credential detection is path-and-name only, so `openssl
pkey -in server.key` asks and `printenv` is missed. Only English commands and
paths were tested.

**Bearings for Tsuba.** Three takeaways. First, this is the C016 method
operationalized for a Pi target (C006), so it is the closest reference to
Tsuba's likely Pi integration plus classifier. Second, the swappable-backend
design — local server, hosted registry model, or external process — is a clean
way to keep the assessment method (C014) and the concrete tool (C015) separate
from the harness integration. Third, its ordering (certain rules first, model
only on the residue, prompt on model failure) matches S001 and cc-safety-net,
which strengthens that as the shape Tsuba should adopt; its fail-to-prompt
behaviour on model failure is a deliberate fail-safe rather than the
fail-closed contract the other studies use.

References: [pi-automode-classifier][pac].

[pac]: https://github.com/deepu105/pi-automode-classifier

### S004: pi-verdict 💡

https://pi.dev/packages/pi-verdict

### S005: jev-use 💡

https://github.com/shitianfang/jev-use

### S006: jev-skill 💡

https://github.com/wuyoscar/jev-skill

### S007: pi-jev 💡

https://github.com/y0usaf/pi-jev


## Metaphor

According to [Tsuba](https://en.wikipedia.org/wiki/Japanese_sword_mountings#Tsuba)
on Wikipedia:

> The tsuba (鍔; or 鐔) is usually a round (or occasionally squarish) guard at
> the end of the grip of bladed Japanese weapons, like the katana and its
> variations, tachi, wakizashi, tantō, naginata etc. They contribute to the
> balance of the weapon and to the protection of the hand. The tsuba was mostly
> meant to be used to prevent the hand from sliding onto the blade during
> thrusts as opposed to protecting from an opponent's blade.
