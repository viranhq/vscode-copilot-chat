# Copilot Chat Customization: Code Changes Reference

## Date: 2026-03-02

## Change 1: Hide "Configure Tools" Button (VS Code Core)

### File

`~/vscode/src/vs/workbench/contrib/chat/browser/actions/chatToolActions.ts`

### Location

`ConfigureToolsAction` class constructor, line ~128

### Original Code

```typescript
menu: [{
    when: ContextKeyExpr.and(
        ChatContextKeys.chatModeKind.isEqualTo(ChatModeKind.Agent),
        ChatContextKeys.lockedToCodingAgent.negate(),
    ),
    id: MenuId.ChatInput,
    group: 'navigation',
    order: 100,
}]
```

### Modified Code

```typescript
menu: [{
    when: ContextKeyExpr.false(),
    id: MenuId.ChatInput,
    group: 'navigation',
    order: 100,
}]
```

### Effect

- Hides the wrench/screwdriver "Configure Tools" button from the chat input toolbar
- Tools remain fully functional in agent mode — only the manual configuration UI is hidden
- The `ConfigureToolsAction` class and its `run()` method are preserved (action can still be invoked programmatically)

### How to Apply

```bash
cd ~/vscode
# Edit the source file
# Then recompile:
npm run compile
# Launch with extension:
./scripts/code.sh --extensionDevelopmentPath=$HOME/vscode-copilot-chat
```

### How to Revert

Restore the original `when` clause:

```bash
cd ~/vscode
git checkout -- src/vs/workbench/contrib/chat/browser/actions/chatToolActions.ts
npm run compile
```

---

## Change 2 (Reverted): Empty languageModelToolSets (Extension)

This change was attempted first but did **not** hide the tools button.

### File

`~/vscode-copilot-chat/package.json`

### What Was Changed

Replaced the `languageModelToolSets` array (containing 6 tool set objects: edit, execute, read, search, vscode, web) with an empty array `[]`.

### Why It Didn't Work

The tools button is rendered by VS Code core based on the existence of `languageModelTools` (individual tools), not `languageModelToolSets` (groupings). Emptying the tool sets only removes the categories from the dropdown — the button itself persists as long as any tools are registered.

### Lesson

UI chrome in the chat panel is controlled by VS Code core's menu/action system, not by extension contributions. To hide UI elements, modify VS Code core.

---

## Other VS Code Change (Pre-existing, Not Ours)

The diff also showed a change in `~/vscode/.vscode/extensions/vscode-extras/src/extension.ts` — renaming `context` to `_context` (unused variable prefix). This was **not** part of our changes.

---

## Build Requirements

| Requirement | Value |
|---|---|
| Node.js | 22.22.0 (per `~/vscode/.nvmrc`) |
| Package manager | npm (VS Code repo uses gulp internally) |
| Compile command | `npm run compile` (~4 min) |
| Watch command | `npm run watch` (incremental) |
| Launch command | `~/vscode/scripts/code.sh --extensionDevelopmentPath=$HOME/vscode-copilot-chat` |
