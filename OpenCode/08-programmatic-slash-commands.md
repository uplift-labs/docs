# Programmatic Slash Commands

Use this pattern when a slash-style command should run local code immediately instead of turning into an LLM prompt. The practical example is `/worktree`: it creates worktree tabs, reports status in the TUI, and does not need model reasoning.

## Mental Model

OpenCode has two different command surfaces that look similar from the user seat:

| Surface | Runs Where | LLM Turn? | Use For |
| --- | --- | --- | --- |
| TUI `api.command.register` with `onSelect` | TUI plugin process | No | Local actions, navigation, dialogs, toasts, opening tabs, triggering scripts. |
| Server `command.execute.before` | Server plugin hook | Yes | Mutating slash/custom command prompt parts before they are sent to the model. |

If the desired behavior is "user types `/name`, code runs, no assistant response is required", use a TUI command. Do not model this as a prompt template plus `command.execute.before`; that still enters the agent loop.

## Minimal Shape

```ts
import type { TuiPlugin } from "@opencode-ai/plugin/tui"

function argsFromCommandInput(input: any) {
  if (!input) return ""
  if (typeof input === "string") return input
  return input.arguments || input.args || input.query || input.value?.replace(/^\/?worktree\s*/, "") || ""
}

export default {
  id: "uplift.worktree-command",
  tui: async (api) => {
    if (!api.command?.register) return

    api.command.register(() => [
      {
        title: "/worktree",
        value: "worktree",
        description: "Create git worktree tab(s). Supports -n N, --print, --no-dirty.",
        category: "Worktree",
        slash: { name: "worktree" },
        onSelect: async (input) => {
          api.ui.toast({ variant: "info", message: "Creating worktree..." })

          const args = argsFromCommandInput(input)
          const result = await runWorktreeCommand({
            directory: api.state.path.directory,
            args,
          })

          api.ui.toast({
            variant: result.status === 0 ? "success" : "warning",
            message: result.message,
            duration: 8000,
          })
        },
      },
    ])
  },
} satisfies { id: string; tui: TuiPlugin }
```

The exact command implementation can shell out, call a local TypeScript module, use the SDK, or open TUI UI. Keep the command item itself thin: parse user input, run one clear action, then surface the result.

## Design Checklist

| Check | Why |
| --- | --- |
| Register from a TUI plugin with a stable `id`. | Local path TUI plugins rely on plugin identity for loading and enablement. |
| Include `title`, `value`, `description`, `category` and `slash.name`. | Makes the command discoverable from the palette and slash flow. |
| Implement `onSelect`. | This is the programmatic execution path; without it the command is only metadata. |
| Parse `input.arguments`, `input.args`, `input.query` and fallback `input.value`. | Different invocation paths can pass arguments with different property names. |
| Use `api.state.path.directory` or `api.state.path.worktree` instead of `process.cwd()`. | The TUI process cwd is not always the target workspace. |
| Report progress and result with `api.ui.toast` or a dialog. | The command does not create an assistant turn, so user feedback must be explicit. |
| Keep expensive work out of render paths. | Register metadata synchronously, but run filesystem/git/process work only inside `onSelect`. |
| Feature-detect optional APIs. | TUI plugin APIs are source-level/internal; fail open if `api.command` is missing. |

## When Not To Use It

Use a normal slash/custom command or `command.execute.before` when the command's job is to prepare instructions for the model, review context, or produce an assistant answer.

Use a custom tool when the model should decide when to call the capability during an agent run.

Use a permission rule or `tool.execute.before` when the goal is to guard an operation rather than expose a user-invoked action.

## Upgrade Smoke Test

After upgrading OpenCode or changing the plugin:

1. Start OpenCode with the TUI plugin enabled.
2. Open the command palette and confirm the command appears with the expected title/category.
3. Type the slash command, including arguments such as `/worktree -n 2 --print`.
4. Confirm `onSelect` receives the arguments and the command does not create an LLM assistant turn.
5. Confirm failure output is visible to the user through a toast/dialog and does not leave the TUI stuck.
