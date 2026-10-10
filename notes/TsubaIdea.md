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

A Pi extension that uses Jev (TypeSafe) as a decision layer: a measured,
calibrated tool-call gate plus a post-execution output judge, both fail-open.

> [!NOTE]
> **Metadata**
> - **Related**: C006, C008, C014, C015, C016

Supporting analysis for Tsuba's Pi integration (C006), generic tool assessment
(C008), assessment method (C014), the third-party-assessment-tool option
(C015), and the low-cost-AI-with-a-prompt option (C016); it does not make the
choice.

It is the most direct reference in the set for a Pi auto-mode gate built on a
third-party classifier: it gates Pi tool calls (C006) through Jev (C015/C016),
and its calibration section is the most concrete treatment of how to turn a
classifier's raw answers into thresholds.

**Shape.** Jev answers *typed* questions about a piece of state — a `noul`
yes/no as a probability, a `choice` as an option plus a distribution, a
`score` as a position on ordered levels — and returns numbers the code
branches on, so nothing parses prose. Three things use it: a gate judging
`bash`, `write`, and `edit` calls before they run; an output judge reading what
a `bash` call printed or a `codemode` script returned; and a `jev_ask` tool the
model can call for the same kind of judgment itself. It is a Pi extension with
no runtime dependencies, and without an API key it loads, says so once, and
stays out of the way.

**The gate.** Four questions go in one request (about 300 ms, versus four
round trips): is the action destructive (`0.90`), does it send local data or
secrets off-machine (`0.70`), does it affect anything beyond what the user
asked for (`0.85`), and how much damage if unwanted (`score` over four levels,
threshold `2.50`). Shadow mode is the default — a flagged call only notifies —
and enforce mode asks for confirmation. Headless runs cannot prompt, so
enforcement falls back to the warning unless `gate.blockWithoutUI` is set.

**Fail-open and caching.** Every error path fails open: a missing key, a
timeout, a 429, or a malformed response produces no verdict and the call
proceeds, with errors rate-limited to once a minute so a dead endpoint cannot
flood the transcript. Identical input is judged once per `cacheSeconds`
(120), and sibling calls from one assistant message share a single in-flight
request. This is the opposite of the fail-closed contract in S001 and S004 —
a deliberate stance, mitigated by shipping in shadow mode.

**Output judge.** The gate sees intent, not what a command printed, so the
extension adds a post-execution judge over `tool_result`: does the output
contain a secret or credential (`noul`, `0.90`) and what kind of failure is
this (`choice` over six classes, confidence floor `0.60`). It appends one line
of fixed advice from a table (retry `transient`, fix the environment for
`environment`, fix the code for `code_bug`, do not retry `permission`, fix the
invocation for `user_error`) and never blocks. This is an assessment placement
beyond the pre-call focus of Tsuba and the other gates: a judgment about text
that exists only after the call.

**Codemode.** Under Pi 0.99 the model can run tools from a `codemode` script.
The gate judges a `bash` call whether the model or a script made it (a blocked
call fails inside the script), while the output judge reads the `codemode`
result — what actually reaches the model — rather than the inner call results,
so a script's intermediate output is left alone.

**Calibration (C014).** The thresholds are measured rather than chosen, and
the findings are the most transferable part of the study. An ordinary
*requested* `sed -i` scored as high as `0.85` on `destructive`, so a `0.7`
threshold would have prompted on every in-scope edit; `0.90` sits above the
whole in-scope band and below the `0.99` a genuinely destructive command
scored. `beyond_scope` cleanly separated in-scope (`0.10–0.74`) from
out-of-scope (`0.90–0.98`), with the threshold mid-gap because the same state
moved by ±0.05 between runs. Question wording mattered: an early destructive
question asking whether data "cannot be recovered from version control" scored
a real `rm -rf` plus force-push at `0.77`, under any useful threshold, because
"it is in git" is a reasoning path the model takes; asking plainly whether the
action is destructive separated the pair `0.03` against `0.99`. The output
judge ran 53 fixtures three times each: the leak question had an empty gap
(`≤0.02` against `≥0.92`) so `0.90` sits in it, and the failure classifier was
right at confidence `0.88–1.00` and wrong once at `0.42`, which is why the
floor is `0.60`. Two retry-shaped questions were dropped as overlapping, and
the advice is derived from the class in code instead of asked. The author
labels all of this a smoke calibration, not a labelled evaluation set — enough
to reject wrong thresholds, not enough to leave shadow mode.

**Privacy accounting.** The gate sends the working directory, the tool name,
the last user message (first 1200 characters), and the tool's arguments to
`api.typesafe.ai`; for `write` / `edit` those arguments contain file content,
and for `codemode` they are the script source. Long strings are truncated with
a marker (400 characters for arguments, 2000 for output), and the omitted text
never leaves the machine — with `gate.tools: ["bash"]` file content stays out
entirely. The key travels in the `Authorization` header, and notification text
is scrubbed of it before reaching the transcript.

**Journal.** Flagged verdicts are stored as Pi custom entries that never enter
the LLM context and survive `/reload`, `/resume`, and `/fork`; reading them
costs no Jev requests, and only flags are journaled.

**Trust and limits.** It is not a sandbox. It fails open on every error, ships
in shadow mode because the calibration is only a smoke test, and its
thresholds are tuned to one model (`jev-latest`). Judging every `read` was
explicitly avoided as one request per file opened.

**Bearings for Tsuba.** Four takeaways. First, this is the closest template to
Tsuba's likely Pi integration plus classifier (C006 + C015/C016), including
how to lay out config (user + project, project wins) and a self-contained
client. Second, its multi-dimension assessment (destructive / exfiltration /
beyond-scope / impact) is a richer C008 breakdown than a single risk score, and
the output judge shows assessment can extend past the pre-call boundary Tsuba
currently targets. Third, the calibration method — measure real in-scope and
out-of-scope bands and place the threshold in the gap, watch for question
wording that lets the model reason the danger away, floor on confidence, batch
the questions — is directly reusable for Tsuba's classifier and its evaluation
(and pairs with the discipline S006 describes). Fourth, its explicit "what
leaves the machine" accounting and truncation is a good model for Tsuba's
privacy posture, and its fail-open choice is the opposite of S001/S004 — so
Tsuba should decide its failure direction deliberately rather than by default.

References: [pi-jev][pj].

[pj]: https://github.com/y0usaf/pi-jev

### S008: jev-shield 💡

An MCP-layer semantic firewall — a ready-made gate *tool* that screens every
tool call, tool result, and tool description through Jev and enforces with
deterministic rules, a semantic battery, and session taint tracking.

> [!NOTE]
> **Metadata**
> - **Related**: C002, C003, C004, C005, C008, C014, C015, C016

Supporting analysis for Tsuba's harness-extension form factor (C002), the
Claude Code / OpenCode / Codex integrations (C004 / C003 / C005), generic tool
assessment (C008), assessment method (C014), the third-party-assessment-tool
option (C015), and the low-cost-AI-with-a-prompt option (C016); it does not
make the choice.

Unlike the other studies, this one is not a component Tsuba would assemble — it
is a finished product that overlaps Tsuba's target: an existing gate. It is the
strongest "adopt versus build" data point in the set.

**Shape.** A proxy that sits between any MCP client and any stdio MCP server and
screens every `tools/list` description, every `tools/call`, and every tool
result. It returns calibrated probabilities and leaves policy to the caller.
It ships in three forms: the MCP firewall itself (`npx jev-shield -- <upstream>`),
a cooperative agent skill (`/verify`, which screens text on demand), and an
*opt-in* hook/plugin layer for tool-by-tool enforcement. The enforcement layers,
not the skill, are what make it a gate.

**Three layers of control.** Structural (deterministic, free): recipient
allowlists and required flags from config. Semantic (Jev, one call per event,
~$0.00003, ~500 ms): exfiltration, destructive actions, injection-in-arguments,
credential/payload sensitivity, tool-result injection, secrets-in-results, and
tool-description poisoning. Session (taint tracking, free): after a result is
flagged, subsequent calls are verified under an escalated policy. The bundled
demo exercised ten verifications for $0.0003 at a median 558 ms.

**Enforcement surfaces.** Claude Code `PreToolUse` returns `deny`/`ask` before
`Bash`/`Write`/`Edit`/`WebFetch`/MCP tools and `PostToolUse` flags poisoned
results; an OpenCode plugin throws from `tool.execute.before` to block and
withholds injected results in `tool.execute.after`; Codex has no hook API so it
is enforced through the MCP firewall wrapper. `JEV_HOOK_OFF=1` is a kill
switch, `JEV_FAIL_MODE=closed` fails closed (default is open), and local reads
(`Read`/`Grep`/`Glob`) are exempt. **It does not cover Pi.**

**Policy is code, not a prompt.** `config/policies.json` holds thresholds
(`review` 0.35, `action` 0.7, `severity_block` 2.0), `hazard_actions`
(`data_exfiltration`/`destructive_action` → `block`), `fail_mode`, and per-tool
rules (`recipient_allowlist`, `require_flag`). The README's framing — probabilities
are fixed, thresholds decide the action, "change policy in code review, not in a
prompt" — is the same separation of mechanism and policy Tsuba needs.

**What the demo catches.** A poisoned tool description is flagged
(`tool_poisoning` 0.9x); an injection inside a search result is *redacted before
the agent reads it* (`contains_injection` 0.98); a novel-phrasing exfiltration
that slips past the allowlist is still blocked (`payload_sensitivity` 0.98); a
destructive call is blocked by severity escalation (2.11 ≥ 2.0); a
secret-bearing result is redacted (`contains_secrets` 0.97); a legitimate email
to an allowed recipient passes — a semantics-not-keywords demonstration.

**Trust and limits.** It states plainly that it is "not a security boundary" —
a probabilistic signal layer on top of sandboxing, least privilege, and human
approval, evadable by a determined attacker, adding ~0.5–1.3 s per call, with a
~32k-token state budget that forces truncation of large outputs. It depends on
TypeSafe access through Vercel AI Gateway, and its enforcement paths exclude Pi.

**Bearings for Tsuba.** First, this is the most direct adopt-versus-build
candidate: a finished gate (C015) already doing MCP-level screening and
harness enforcement across Claude Code, OpenCode, and Codex, so Tsuba's
differentiation would have to be the Pi / Oh My Pi coverage (C006) and whatever
it does beyond a Jev-based signal. Second, its three-layer model — a
deterministic structural floor, a semantic battery, and session taint tracking —
is a concrete C008 architecture richer than single-question classification, and
taint-escalation after a flagged result is a dimension Tsuba's pre-call-only
assessment lacks. Third, the strict policy-in-code split is worth copying
verbatim in spirit. Fourth, its honest "signal, not boundary" stance and its Pi
absence together define exactly what adopting it would and would not buy.

References: [jev-shield][jsh].

[jsh]: https://github.com/caiovicentino/jev-shield

### S009: jev-mcp 💡

An MCP server exposing twelve Jev judgment tools, plus a documented
`PreToolUse` hook gate that turns one `jev_decide` call into a policy-driven
tool-call gate across Claude Code, Codex, OpenCode, and Pi.

> [!NOTE]
> **Metadata**
> - **Related**: C002, C003, C004, C005, C006, C008, C014, C015, C016

Supporting analysis for Tsuba's harness-extension form factor (C002), the
Claude Code / OpenCode / Codex / Pi integrations (C004 / C003 / C005 / C006),
generic tool assessment (C008), assessment method (C014), the
third-party-assessment-tool option (C015), and the low-cost-AI-with-a-prompt
option (C016); it does not make the choice.

It matters to Tsuba twice over: it uses "gate" in two distinct senses — a
completion gate *tool* (`jev_gate`) and a tool-call gate *hook* — and the hook
recipe ships an adapter for Pi, the integration Tsuba targets.

**Shape.** An MCP server (`npx -y @jkudish/jev-mcp`) offering twelve tools that
return typed probabilities in roughly 150–500 ms for a fraction of a cent. It
registers with Claude Code, Codex, OpenCode, Amp, and generic MCP clients
(stdio, or stateless HTTP for a shared/remote server), and can be embedded
in-process via a side-effect-free `createServer()`. It ships an agent skill that
teaches the agent when to reach for each tool — the "registered-but-unused"
problem the whole class of MCP tools faces.

**Tool taxonomy (C008).** The suite is a menu of assessment dimensions:
`jev_verify` (each cited claim against evidence, per-claim verdict, full
distribution, confidence, plus a `same_subject` question so a contradiction
only stands when the evidence is about the same thing), `jev_screen`
(prompt-injection / substance / relevance *before content enters context*,
returning an advisory `pass`/`review`/`block`/`skip` and never blocking
itself), `jev_find` and `jev_rerank` (selection by meaning, no embeddings),
`jev_classify` (batched labeling, auto only when probability ≥ 0.85 *and*
winner margin ≥ 0.5), `jev_decide` (a bounded 2–6-way choice with
`ask_user`/`investigate`/`none` escape hatches and per-requirement checks),
`jev_compare`, `jev_extract` (regex candidates; the model picks a verbatim
match and never writes a value), `jev_audit` (a failure-mode battery —
hallucinated / off-target / incomplete / format — max-gated so one fired flag
cannot be diluted), `jev_review` (four 0–2 rubrics plus a `safe_to_apply`
probability combined into a composite and one `auto`/`review`/`escalate`), and
`jev_gate` (`jev_review` plus completion-claim verification in one call).

**The completion gate.** `jev_gate` is not a tool-call gate: it gates a *done*
claim. It is `auto` only when the patch review is accepted *and* every "tests
pass" claim is verified against supplied evidence; a confidently contradicted
claim escalates. It is the sharpest expression in the set of self-report honesty
as an assessment dimension — catching exactly the success claim an agent is most
tempted to hand-wave.

**The hook gate (the tool-call gate).** `examples/hook-gate.md` wires a
`PreToolUse` hook to one `jev_decide` call: candidates `proceed`/`block`,
escape hatches on, the policy in the `priorities` field, and the tool call
framed as untrusted facts. The routing discipline is the reusable part —
the harness matcher is the free router; local `--skip` regexes and a
4,000-character size cap defer at zero cost (exit 0, so the harness's own flow
applies); one model call judges only what matched; and a verdict is emitted
*only* on a confident, valid judgment (`deny` needs `block` selected at or above
`--block-threshold`, default 0.85). Provider errors, timeouts,
`invalid_response`, and escapes all defer, and `proceed` always defers — the
gate never fabricates an allow.

**Cross-harness wiring.** The same runner serves Claude Code and Codex (shared
`PreToolUse` JSON on stdin, `hookSpecificOutput.permissionDecision` on stdout).
OpenCode intercepts in-process via a plugin that throws from
`tool.execute.before` (with the caveat that it has not always fired for MCP
calls); **Pi** intercepts via an extension returning `{ block, reason }` from
the `tool_call` event. `--ask` maps the `ask_user` escape to `ask`, supported on
Claude Code only.

**Honesty and calibration.** The README states that a policy gate is not a
security boundary, that the tool-call text is untrusted input framed as facts
(framing is mitigation, not a guarantee), and that deferring is not approval. It
prescribes a reproducible threshold experiment — capture real payloads, label
each `allow`/`block`/`unsure`, sweep `--block-threshold`, count false denies and
misses, pin `JEV_MCP_MODEL` — and notes ~1 s per judged call, which is why the
matcher and skip layers exist.

**Trust and limits.** Providers are pluggable (TypeSafe / OpenRouter /
Cloudflare / Vercel / compatible); inputs leave the environment. It notes that
confidence measures how peaked a choice distribution is, not the probability the
answer is correct, so thresholds are starting points to tune against your own
traffic. Typed output guarantees the interface, not the truth.

**Bearings for Tsuba.** First, the hook gate is the closest published recipe to
Tsuba's own enforcement shape, and it is the only listed tool with a ready Pi
adapter (C006) alongside Claude Code, Codex, and OpenCode — directly usable as
the gate mechanism (C002). Second, its routing and its "deny only on
confidence, otherwise defer, never auto-allow" principle are the clearest
statement of a fail direction, and are worth adopting over S007's fail-open /
fail-closed binary. Third, its eleven non-gate tools are a menu of C008
dimensions well beyond destructive-command checks — injection screening,
fabrication auditing, diff blast-radius review, claim verification — several
judging text and results rather than the call, extending assessment past the
pre-call boundary. Fourth, its validation strictness (`invalid_response`,
probability-and-margin gates, confidence floors, max-gated audit) and its
"measure your own traffic, pin the model" stance reinforce the calibration
discipline in S006 and S007.

References: [jev-mcp][jm], [hook gate][jmh], [skill][jms].

[jm]: https://github.com/jkudish/jev-mcp
[jmh]: https://github.com/jkudish/jev-mcp/blob/main/examples/hook-gate.md
[jms]: https://github.com/jkudish/jev-mcp/blob/main/skills/jev/SKILL.md


## Metaphor

According to [Tsuba](https://en.wikipedia.org/wiki/Japanese_sword_mountings#Tsuba)
on Wikipedia:

> The tsuba (鍔; or 鐔) is usually a round (or occasionally squarish) guard at
> the end of the grip of bladed Japanese weapons, like the katana and its
> variations, tachi, wakizashi, tantō, naginata etc. They contribute to the
> balance of the weapon and to the protection of the hand. The tsuba was mostly
> meant to be used to prevent the hand from sliding onto the blade during
> thrusts as opposed to protecting from an opponent's blade.
