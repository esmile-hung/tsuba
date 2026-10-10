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

A minimal Pi permission gate inspired by Claude Code's auto mode: a
three-state verdict (allow / ask / deny) from a deterministic floor, user
rules, then a context-aware classifier, fail-closed throughout.

> [!NOTE]
> **Metadata**
> - **Related**: C002, C006, C008, C014, C016, C017

Supporting analysis for Tsuba's harness-extension form factor (C002), Pi
integration (C006), generic tool assessment (C008), assessment method (C014),
the low-cost-AI-with-a-prompt option (C016), and the keyword-catcher floor
(C017); it does not make the choice.

It is the third Pi auto-mode gate studied (after S001 and S003) and the one
whose README explicitly documents how it *converged* with S001 on the same
architecture — deny floor, then user rules, then classifier, fail-closed. Its
additions over the others are worth weighing: a classifier that can itself
return `ask`, a self-protection layer no configuration can disable, and a
context-aware classifier rather than a command-only one.

**Shape.** The core concept is the three-state *verdict*: `allow`, `deny`, or
`ask` (routed to the human, degrading to `deny` in non-interactive sessions).
The design goal is stated as safe automation, not maximum automation, because
both approval fatigue and silent unsafe execution lose. It is a ~2k-line
single-file extension with zero runtime dependencies. It is a permission gate,
not a sandbox, and does not guard manual `!` shell escapes.

**Pipeline.** Every call runs a fixed ladder. Stage 0 is self-protection
(below). Stage 1 is the deterministic rule layer at zero latency: the built-in
deny floor (bash danger regexes plus path sensitivity tiers S0–S5), user
`allow` / `deny` regexes (deny beats allow, both beat the classifier),
`denyPaths` (a declared protected path triggers a terminal ask before any user
allow, and the classifier is told only that a protected path exists, never
its name), and `ignoreTools` (declared uncovered tools pass with zero model
calls). Stage 2 is the gray zone: nested codemode calls may be routed to
rules-only, otherwise the call goes to the classifier. Stage 3 is the
three-state adjudication. Any classifier exception, timeout (25s), or contract
violation becomes a `deny`.

**Assessment.** The classifier sees the *conversation context* — recent user
intent plus prior tool calls — not just the command string, which is the key
contrast with S003's command-only classifier. The default is "self-reflection"
(the session model adjudicates); `classifierModel` pins another model, and
`"typesafe/jev-latest"` opts into Pi's native `classify()` catalog (choice +
probabilities + confidence). This is the low-cost-AI-with-a-prompt method
(C016) again, but with two refinements: the model can emit `ask` as a verdict
rather than only a risk score, and a confidence floor
(`classifierMinConfidence`) with a fallback cascade (`classifierFallbackModel`)
demotes low-confidence native verdicts to a deeper model or to the user,
without ever auto-relaxing a demoted deny or ask. Design decisions are
measurement-backed and the lab notes ship in-repo — e.g. a verdict cache was
declined after replaying 1.2k real verdicts measured a 3.2% hit rate, and a
tree-sitter rule engine was rejected after it absorbed zero gray-zone calls on
746 real commands.

**Self-protection.** The gate guards its own files (the config and installed
extension copy) against in-process writes — reads pass, writes hard-deny, on
the sudoers / visudo precedent that the owner's editor never passes through
the gate. It is not disableable by any config, and tamper detection snapshots
watched files at session start and re-verifies before every verdict, auto-
restoring a changed extension copy and failing closed. This is a governance
idea the earlier studies lack.

**Trust and limits.** The path-sensitivity floor applies to file tools only,
so `cat ~/.ssh/id_rsa` in bash goes to the classifier while `read
~/.ssh/id_rsa` denies deterministically. On Windows the floor covers
bash-shaped patterns only, so PowerShell destructive commands rely on the
classifier. `denyPaths` bash extraction is token-level and obfuscatable
(substitution, base64, external script contents, spaces in paths, recursive
searches), which the author frames as acceptable because a hit routes to the
human rather than deciding silently. There is deliberately no built-in
allowlist — every "always allow" claim is the user's. AGENTS.md is not passed
to the classifier as intent evidence.

**Hosts.** It requires Pi ≥ 0.99 for the native classifier; older Pi and Oh My
Pi keep a 0.12.x line that self-anchors to whichever agent tree it is
installed in. That host-split install story is a concrete instance of the
packaging difference Tsuba faces (C002) and the Pi / Oh My Pi pairing (C006).

**Bearings for Tsuba.** The convergence with S001 is the strongest signal yet
that the floor → rules → classifier → fail-closed shape is the settled design
for a Pi auto-mode gate. Beyond that, three ideas extend Tsuba's concept set:
a classifier permitted to return `ask` (so ambiguity is a first-class verdict,
not a low score); self-protection of the gate's own configuration and code;
and the confidence-floor-plus-fallback cascade for layering a fast classifier
over a deeper one. It is also a concrete third-party tool for C015 and C016,
and its regex deny floor is the C017 keyword-catcher idea used as the
deterministic bottom layer.

References: [pi-verdict][pv], [pi-verdict repo][pv-repo],
[security principles][pv-security], [configuration][pv-config].

[pv]: https://pi.dev/packages/pi-verdict
[pv-repo]: https://github.com/jesset/pi-verdict
[pv-security]: https://cdn.jsdelivr.net/npm/pi-verdict@0.15.0/docs/security-principles.md
[pv-config]: https://cdn.jsdelivr.net/npm/pi-verdict@0.15.0/docs/configuration.md

### S005: jev-use 💡

A harness plugin that hands decisions needing no text output to Jev — a
purpose-built judgment model — and, relevant to Tsuba, wraps it as a gate that
risk-checks each tool call before it runs.

> [!NOTE]
> **Metadata**
> - **Related**: C004, C005, C006, C008, C014, C015, C016

Supporting analysis for Tsuba's Claude Code (C004), Codex (C005), and Pi
(C006) integrations, generic tool assessment (C008), assessment method (C014),
the third-party-assessment-tool option (C015), and the low-cost-AI-with-a-
prompt option (C016); it does not make the choice.

It is less a full auto-mode gate than the others (S001, S003, S004) and more a
tooling layer for one judgment model, but it is the reference for that model's
*wire protocol* and for wrapping a classifier as a gate. Its Jev model is the
same one S003 and S004 name as a classifier, so this study fills in what that
model actually is and how it is called.

**Shape.** The project splits decisions by whether they need new text. A
deterministic router (no request spent) sends writing tasks and open-ended
judgments to the LLM, and everything enumerable to Jev. It ships several
surfaces over one engine: two MCP tools (`jev_judge`, `jev_gate`), a CLI
(`jev-use judge`, `jev-use hook gate`), a Claude Code plugin carrying a
routing skill, a Codex plugin, and a PreToolUse hook adapter for Claude Code
and Codex. The install step wires whichever of Claude Code, Codex, and Pi it
finds.

**The gate.** `jev_gate` takes one proposed action (tool + input, verbatim)
and returns `allow` / `deny` / `escalate`. The semantics are "tighten only":
`deny` denies with a reason, an unsure verdict asks, and `allow` stays silent
so the host's normal permission flow still decides — the gate can never grant
anything by itself. An unreachable provider never becomes an allow: the tool
reports `escalate` with reason `unreachable` (proceed as if Jev were absent)
and the hook defers to the normal flow. On the hook path it spends zero LLM
tokens on allow — a deny or ask feeds its reason back to the model, which is
the point — at roughly one ~100 ms round trip per gated call, so the matcher
is scoped to tools worth gating.

**Assessment.** Jev answers *typed* questions, each a small primitive: `noul`
(the probability a statement is true), `choice` (one enumerated option), and
`score` (a position on an ordered list of levels). Questions about one state
are batched into a single call, and latency is flat in question count
(measured 1 / 4 / 8 questions ≈ 274 ms). Every verdict carries a confidence
and a `confidenceFrom` field saying whether it was reported by Jev's own head
or estimated from the answer distribution, with a per-source escalation
threshold. The escalation contract is explicit and typed: pre-call reasons
(`writing`, `open_ended`, `oversized`) come from the deterministic router,
post-call reasons (`unsure`, `unreachable`) from the model, and every handback
is a normal verdict with a hint, never an exception — the LLM stays the
planner and writer. This is the low-cost-model method (C016) expressed as a
reusable protocol rather than one hard-coded prompt.

**Redaction.** The gated action is the one part of the judged state the user
did not write, so its credentials are stripped before the call — URL
passwords, auth and cookie headers, `-u user:pass`, `--token=` / `SECRET=`
values, and known key shapes become `[redacted]`, while a value that is only a
reference (`$GITHUB_TOKEN`) is left alone. Everything the verdict depends on
(the tool, flags, host, path) still goes as-is. This is a concrete answer to
the privacy concern Tsuba faces when it sends a tool call to a classifier.

**Trust and limits.** It is not a sandbox. The published cost is a *rate* win,
not a token win — Jev spends more tokens per decision, so the saving is real
only when the decision leaves the conversation; routing bulk data by reference
(a file piped to the CLI) is part of the design. The gate only ever tightens,
so an outage cannot silently grant permission.

**Bearings for Tsuba.** Four takeaways. First, the typed verdict contract
(`noul` / `choice` / `score`, per-verdict confidence, typed escalation reasons)
is a cleaner interface than a raw risk score and directly suits C014/C016.
Second, the deterministic pre-call router — deciding what never reaches the
model — is a reusable way to keep the classifier off the easy cases. Third,
credentials-redacted-before-send is worth adopting for any model-backed
assessment. Fourth, the "tighten only, never grant, defer on failure" gate
semantics is a third stance between S003's fail-to-prompt and the fail-closed
contract elsewhere. As a concrete tool it is a strong C015 candidate: Tsuba
could call `jev_gate` (or the underlying model) rather than build a classifier,
though that accepts a third-party dependency and sends tool calls to a
provider.

References: [jev-use][jevu], [reference][jevu-ref],
[Claude Code harness][jevu-cc], [routing skill][jevu-skill].

[jevu]: https://github.com/shitianfang/jev-use
[jevu-ref]: https://github.com/shitianfang/jev-use/blob/main/docs/reference.md
[jevu-cc]: https://github.com/shitianfang/jev-use/blob/main/harness/claude-code/README.md
[jevu-skill]: https://github.com/shitianfang/jev-use/blob/main/skills/jev-use/SKILL.md

### S006: jev-skill 💡

A community collection of Jev use-case skills for coding agents — a catalogue
of what an agent can hand to a judgment model, and a skill-based delivery
mechanism, rather than a tool-call gate.

> [!NOTE]
> **Metadata**
> - **Related**: C002, C008, C014, C015, C016

Supporting analysis for Tsuba's harness-extension form factor (C002), generic
tool assessment (C008), assessment method (C014), the third-party-assessment-
tool option (C015), and the low-cost-AI-with-a-prompt option (C016); it does
not make the choice.

Unlike the other studies it is not a gate: it does not gate tool calls and is
not a security boundary. Its value to Tsuba is different in kind — it
enumerates the *kinds of decisions* an agent can delegate to a classifier, and
it demonstrates the advisory skill form factor next to the enforcing hook form
factor the other studies use.

**Shape.** The project is a curated "awesome" collection: five installable
skills (`jev`, `jev-triage`, `jev-documents`, `jev-eval`, `jev-act`), 108
scenarios, and 66 linked community projects and models. The division of labour
is the same as S005: Jev chooses, classifies and scores through its three
typed primitives (`noul` yes/no, `choice`, `score`), while the agent supplies
the evidence and takes the action. It installs as agent skills into Codex,
Claude Code, and OpenCode, and also ships an optional CLI (`jev-decide`). The
skill is *instructional*: it teaches the agent when and how to consult the
model, and the agent decides whether to comply.

**Decision taxonomy (C008).** The collection is effectively a catalogue of
candidate assessment dimensions, several of which bear on gating rather than
on the destructive-command checks the other studies focus on: action-risk
triage, plan-versus-action consistency (is this call consistent with the
stated plan?), test-weakening / reward-gaming detection ("does this edit
weaken a required check?"), suspicious tool-output instructions, completion-
evidence checks (does the evidence support the success claim?), goal-drift
checkpoints, and stuck-loop recovery. This broadens "tool assessment" beyond
"is this command destructive?" toward intent-conformance and self-report
honesty — dimensions Tsuba could consider for C008.

**Skill versus hook (C002).** The collection delivers its guidance as a skill:
the prompt pattern tells the agent to consult Jev "when failures repeat, a
route needs choosing, or you are about to claim completion," and to leave
trivial steps alone. This is the advisory pole of the harness-extension form
factor, where the enforcement (the other studies' hooks and extensions) is
separate. For Tsuba it suggests a possible split: enforcement stays in the
harness hook, while a companion skill carries the policy and the places the
model should *expect* to be gated.

**Calibration pitfalls (C014 / C016).** The pitfalls guide is a useful
counterweight to a naive classifier layer: distinguish probability,
confidence, and score rather than acting on any number above a fixed 0.9;
keep every answer and measure repeatability separately from accuracy against
independent labels; supply *enough* context, not the largest context; treat
missing evidence as unknown; and treat saved judgments as expiring when the
evidence, question, or policy version changes. A reported pilot had Jev match
24/24 preassigned labels against DeepSeek's 23/24 on a 24-record synthetic
set, but the authors state plainly that it did not establish production
accuracy or calibration.

**Safety posture.** The setup skill asks the user to choose a real Jev service
(OpenRouter or TypeSafe) or an explicit simulation, never switches between
them silently, never asks for a key in chat, and verifies offline before any
paid call. A simulation is labelled (`jev_called: false`, null probabilities)
and the agent is told not to invent numbers — a small but relevant precedent
for a classifier-backed tool that must stay honest about whether a verdict was
really produced.

**Trust and limits.** It is not a security boundary; the collection lists
community projects without claiming to have installed or tested them, and Jev
is an external service that receives the judged state. Nothing here enforces a
block.

**Bearings for Tsuba.** Three takeaways. First, the decision taxonomy gives
C008/C014 candidate assessment dimensions beyond destructive commands —
plan-vs-action consistency, self-report honesty, and test-weakening — that a
Tsuba gate could flag even when no command is dangerous. Second, the
skill-versus-hook distinction is worth keeping explicit: Tsuba enforces in the
harness, but an advisory skill could carry the policy the model should follow
and the expectation that certain calls will be gated. Third, the calibration
discipline (separate probability / confidence / score, calibrate locally
against independent labels, treat saved verdicts as expiring) is directly
applicable to Tsuba's classifier method and to how it evaluates that method.

References: [jev-skill][jevs], [general skill][jevs-jev],
[pitfalls][jevs-pitfalls], [installation][jevs-install].

[jevs]: https://github.com/wuyoscar/jev-skill
[jevs-jev]: https://github.com/wuyoscar/jev-skill/blob/main/skills/jev/SKILL.md
[jevs-pitfalls]: https://github.com/wuyoscar/jev-skill/blob/main/skills/jev/references/pitfalls.md
[jevs-install]: https://github.com/wuyoscar/jev-skill/blob/main/docs/install.md

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
