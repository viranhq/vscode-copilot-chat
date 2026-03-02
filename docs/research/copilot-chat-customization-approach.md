# Customizing GitHub Copilot Chat: Research & Approach

## Date: 2026-03-02

## Objective

Strip, restyle, and customize the GitHub Copilot Chat VS Code extension by removing UI elements (starting with the "Configure Tools" button in the chat input toolbar).

## Architecture Discovery

### Two-Repo Architecture

Customizing Copilot Chat requires changes across **two repositories**:

| Repo | Role | What It Controls |
|------|------|-----------------|
| `vscode-copilot-chat` | Extension | Chat participants, tools, prompts, agent logic, `package.json` contributions |
| `vscode` | VS Code core | Chat UI rendering, toolbar buttons, input area, menu system |

### Key Finding: UI Ownership

The chat input toolbar buttons are **not controlled by the extension**. They are rendered by VS Code core's menu system (`MenuId.ChatInput`). The extension only contributes *data* (tools, tool sets, chat participants) — the *UI chrome* is owned by VS Code.

This means:
- **Extension-side changes** (e.g., emptying `languageModelToolSets` in `package.json`) affect what data is available but don't reliably hide UI elements
- **VS Code-side changes** are required to hide/show toolbar buttons, change layouts, or alter the chat input area

### Chat Input Toolbar Rendering

The chat input bar buttons are rendered via VS Code's action/menu system:

1. Actions register themselves to `MenuId.ChatInput` in their constructor
2. `chatInputPart.ts` creates a `MenuWorkbenchToolBar` that renders all actions registered to `MenuId.ChatInput`
3. Each action has a `when` clause (a `ContextKeyExpr`) that controls visibility

**Key file:** `~/vscode/src/vs/workbench/contrib/chat/browser/widget/input/chatInputPart.ts` (line ~2167)

### Extension Modularity (vscode-copilot-chat)

The extension is highly modular via a contribution system:

- **Master list:** `src/extension/extension/vscode-node/contributions.ts` — ~30+ features registered as contributions
- **Easy to remove:** Comment out a line to disable notebooks, surveys, MCP, BYOK, etc.
- **Config-driven:** Many features gated by settings (e.g., `github.copilot.chat.reviewAgent.enabled`)
- **Chat-gated:** ~30 features auto-disable when user is unauthenticated

### Extension package.json Contributions

| Contribution | Purpose | Hides UI? |
|---|---|---|
| `languageModelTools` (15+) | Register individual tools | No — VS Code still shows button if tools exist |
| `languageModelToolSets` (6) | Group tools into categories | No — only affects dropdown content |
| `chatParticipants` (6+) | Register chat agents | Partially — affects agent picker |
| `commands` (200+) | Register VS Code commands | No — commands are invisible until invoked |
| `menus` | Context menus, editor menus | Yes — controls extension-contributed menu items |

## Approach for Hiding UI Elements

### Pattern: `ContextKeyExpr.false()`

To hide a VS Code core UI element while keeping its functionality:

1. Find the action class that registers the button (search for the icon codicon or action ID)
2. Locate its `menu` registration with `MenuId.ChatInput`
3. Change the `when` clause to `ContextKeyExpr.false()` — the button never renders but the action still exists

This is non-destructive: tools remain functional for agent mode, they just can't be configured via the UI button.

### Build & Test Workflow

```
1. Edit .ts source in ~/vscode/src/
2. Compile: cd ~/vscode && npm run compile (~4 min full, or use watch mode)
3. Launch: ~/vscode/scripts/code.sh --extensionDevelopmentPath=$HOME/vscode-copilot-chat
4. Verify in Extension Development Host window
```

**Requirements:** Node 22.22.0 (from `~/vscode/.nvmrc`), nvm for version switching.

**Gotcha:** `npm run compile` runs `clean-out` which wipes `~/vscode/out/`. Editing compiled `.js` files directly works for quick iteration but gets overwritten on recompile. Always edit the `.ts` source.

## Chat Input Toolbar Button Inventory

All buttons in the chat input bar are registered to `MenuId.ChatInput`. To find them:

```bash
grep -r "MenuId.ChatInput" ~/vscode/src/ --include="*.ts" -l
```

The "Configure Tools" button is defined in:
`src/vs/workbench/contrib/chat/browser/actions/chatToolActions.ts`

Other buttons (attach, model picker, mode selector, etc.) follow the same pattern in other action files under `src/vs/workbench/contrib/chat/browser/`.

## Recommendations for Future Work

1. **Start with VS Code core** when hiding/changing UI chrome
2. **Use `ContextKeyExpr.false()`** for clean removal — preserves functionality
3. **Use watch mode** (`npm run watch` in vscode repo) to avoid 4-min full compiles
4. **Keep extension changes minimal** — prefer VS Code core changes for UI, extension changes for behavior/features
5. **Document each hidden element** with the original `when` clause for easy restoration
