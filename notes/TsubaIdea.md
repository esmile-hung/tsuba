# Tsuba - Idea

Tsuba provides mechanism to ensure *tool calls* by AI agents make no harm.


## Why we need Tsuba?

Claude Code supports a feature called [*auto mode*][claude-automode] which
could smartly assess tool calls made by agents, and block dangerous actions if
they would beyond user requests and/or would make any harm. It is backed by
a classifier model that assess every tool calls before they run, so that user
does not have to respond repeated permission requests.

However, the classifier model is built into Claude models like Haiku, Sonnet,
Opus, and Fable. This makes using non-Claude model in Claude Code impossible to
use auto mode.

In other AI agents like OpenCode, although they would also support permission
gates for tool calls, many of them manage permissions heuristically or though
inflexible configurations - like Claude Code without *auto mode*.

We can say that, one of the key factor for efficient AI usage is auto mode.
Without auto mode, the speed of problem solving and task execution with AI
would degarde a lot. Thus, we need auto mode alternative, on Claude Code and/or
other AI agent harness. Otherwise, auto mode would be a strong factor of vendor
locking.

[claude-automode]: https://code.claude.com/docs/en/permission-modes#eliminate-prompts-with-auto-mode


## Form Factor

Tsuba leverage *harness extension* mechanisms provided by each AI agent system
to customize agent behavior and add tool call gating feature. Its form factor
depends on how AI agent system enable harness customization.

For example, on Claude Code Tsuba would be a [*Claude Plugin*][claude-plugins]
(bundle of agent skills, scripts, hooks, ..., etc.) that [*hooks*][claude-hooks]
the `PreToolUse` event. In OpenCode, Tsuba would be an OpenCode plugin (a npm
package that runs within OpenCode process) that hooks `tool:execute` event.

Other AI agent harness would have different mechanism to enable customization.
And form factor of Tsuba on them depends on their extension mechanisms.

[claude-plugins]: https://code.claude.com/docs/en/plugins/
[claude-hooks]: https://code.claude.com/docs/en/hooks
[opencode-plugin]: https://opencode.ai/docs/plugins/


## Metaphor

According to [Tsuba](https://en.wikipedia.org/wiki/Japanese_sword_mountings#Tsuba)
on Wikipedia:

> The tsuba (鍔; or 鐔) is usually a round (or occasionally squarish) guard at
> the end of the grip of bladed Japanese weapons, like the katana and its
> variations, tachi, wakizashi, tantō, naginata etc. They contribute to the
> balance of the weapon and to the protection of the hand. The tsuba was mostly
> meant to be used to prevent the hand from sliding onto the blade during
> thrusts as opposed to protecting from an opponent's blade.



