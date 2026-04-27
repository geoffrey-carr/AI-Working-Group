# Claude Code Tips and Tricks

## CLAUDE.md Best Practices

- **CLAUDE.md is a router, not a rulebook.** Its job is to be a lean, high-level map that points the agent to the right "skills," documentation, or sub-directories for the task at hand.
- **Enforce, don't instruct.** Instead of telling Claude to run a typecheck, use a hook (like a git hook, custom eslint rule, or a script that runs on file-save) that forces the typecheck. As one user put it: "The rule tells the model what you think is true; the tooling shows it what is true."
- **Solve documentation staleness.** One user shared a tool that watches git diffs and auto-updates documentation, while another system has agents document their own work as a byproduct of completing tasks.
- **Progressive disclosure is key.** Instead of a massive context dump, give the agent a map and let it pull the specific documents or tools it needs for a given task. One user even wraps tools in simple curl scripts to expose them gradually without needing a full MCP server.
- **There's an official plugin for that.** Before building your own system from scratch, Anthropic has an official `claude-md-management` plugin in the marketplace that audits your CLAUDE.md for quality and helps capture session learnings.

---

## 10 Workflow Tips
Hey look a test!
### 1. Set up the `cc` alias

Add this to your `~/.zshrc` or `~/.bashrc`:

```bash
alias cc='claude --dangerously-skip-permissions'
```

Run `source ~/.zshrc` to load it. You type `cc` instead of `claude` and skip every permission prompt. The flag name is intentionally scary — only use it after you fully understand what Claude Code can and will do to your codebase.

### 2. Give Claude a way to check its own work

Include test commands or expected outputs directly in your prompt:

```
Refactor the auth middleware to use JWT instead of session tokens.
Run the existing test suite after making changes.
Fix any failures before calling it done.
```

Claude runs the tests, sees failures, and fixes them without you stepping in. Boris Cherny says this alone gives a 2–3x quality improvement.

### 3. Install a code intelligence plugin for your language

Language Server Protocol plugins give Claude automatic diagnostics after every file edit. This is the single highest-impact plugin you can install.

```bash
/plugin install typescript-lsp@claude-plugins-official
/plugin install pyright-lsp@claude-plugins-official
/plugin install rust-analyzer-lsp@claude-plugins-official
/plugin install gopls-lsp@claude-plugins-official
```

Run `/plugin` and go to the Discover tab to browse the full list.

### 4. Stop interpreting bugs for Claude — paste the raw data

Pipe output directly from the terminal:

```bash
cat error.log | claude "explain this error and suggest a fix"
npm test 2>&1 | claude "fix the failing tests"
```

Your interpretation adds abstraction that often loses the detail Claude needs. Give Claude the raw data and get out of the way.

### 5. Use `.claude/rules/` for rules that only apply sometimes

To make a rule load only when Claude works on specific files, add a `paths` frontmatter block:

```yaml
---
paths:
  - "**/*.ts"
---
# TypeScript conventions
Prefer interfaces over types.
```

TypeScript rules load when Claude reads `.ts` files, Go rules when it reads `.go` files.

### 6. Auto-format with a PostToolUse hook

Add a `PostToolUse` hook in `.claude/settings.json` that runs Prettier on any file after Claude edits or writes it:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_FILE_PATH\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

The `|| true` prevents hook failures from blocking Claude. Add `npx eslint --fix` as a second hook entry to chain tools.

### 7. Block destructive commands with PreToolUse hooks

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "type": "command",
        "command": "if echo \"$TOOL_INPUT\" | grep -qE 'rm -rf|drop table|truncate'; then echo 'BLOCKED: destructive command' >&2; exit 2; fi"
      }
    ]
  }
}
```

The hook fires before Claude executes the tool. Destructive commands get caught before they cause damage. Add to `.claude/settings.json` or tell Claude to set it up via `/hooks`.

### 8. Let Claude interview you when you can't fully spec a feature

```
I want to build [brief description]. Interview me in detail
using the AskUserQuestion tool. Ask about technical implementation,
edge cases, concerns, and tradeoffs. Don't ask obvious questions.
Keep interviewing until we've covered everything,
then write a complete spec to SPEC.md.
```

Once the spec is done, start a fresh session to execute with clean context and a complete spec.

### 9. Play a sound when Claude finishes

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          {
            "type": "command",
            "command": "/usr/bin/afplay /System/Library/Sounds/Glass.aiff"
          }
        ]
      }
    ]
  }
}
```

Kick off a task, switch to something else, hear a ping when it's done.

### 10. Fan-out with `claude -p` for batch operations

```bash
for file in $(cat files-to-migrate.txt); do
  claude -p "Migrate $file from class components to hooks" \
    --allowedTools "Edit,Bash(git commit *)" &
done
wait
```

`--allowedTools` scopes what Claude can do per file. Run in parallel with `&` for maximum throughput. Good for converting file formats, updating imports across a codebase, and repetitive migrations where each file is independent.

---

## Further Resources

- [Ruflo](https://github.com/ruvnet/ruflo) — an agent orchestration package for Claude Code
- [claude-code-guide](https://github.com/OriNachum/claude-code-guide) — a community-maintained repo updated with the latest Claude Code tips and tricks
