# Code OSS + Copilot Chat Extension: Local Development Setup

## Critical Requirements

- Node.js **22.22.0** (not 22.16.0) — VS Code's build scripts use `.ts` files directly, which requires native type stripping (unflagged in 22.22.0)
- Both repos must be **siblings** on disk — launch configs and scripts hardcode `../vscode`
- `product.overrides.json` must include **three blocks**: `defaultChatAgent`, `extensionGallery`, and `trustedExtensionAuthAccess`
- A GitHub Copilot subscription is required at runtime (free tier works)

## Directory Layout

```
~/
  vscode/                  # microsoft/vscode fork
  vscode-copilot-chat/     # this extension repo
```

## Setup Steps

### 1. Install Node 22.22.0

The VS Code repo's `.nvmrc` specifies 22.22.0. Earlier 22.x versions fail on `.ts` preinstall scripts with `ERR_UNKNOWN_FILE_EXTENSION`.

```bash
nvm install 22.22.0
nvm alias default 22.22.0
node --version  # v22.22.0
```

### 2. Clone and install VS Code

```bash
git clone https://github.com/<your-org>/vscode.git ~/vscode
cd ~/vscode
npm install
```

### 3. Create `product.overrides.json`

Create `~/vscode/product.overrides.json` with all three required blocks:

```json
{
  "defaultChatAgent": {
    "extensionId": "GitHub.copilot",
    "chatExtensionId": "GitHub.copilot-chat",
    "documentationUrl": "https://aka.ms/github-copilot-overview",
    "termsStatementUrl": "https://aka.ms/github-copilot-terms-statement",
    "privacyStatementUrl": "https://aka.ms/github-copilot-privacy-statement",
    "skusDocumentationUrl": "https://aka.ms/github-copilot-plans",
    "publicCodeMatchesUrl": "https://aka.ms/github-copilot-match-public-code",
    "manageSettingsUrl": "https://aka.ms/github-copilot-settings",
    "managePlanUrl": "https://aka.ms/github-copilot-manage-plan",
    "manageOverageUrl": "https://aka.ms/github-copilot-manage-overage",
    "upgradePlanUrl": "https://aka.ms/github-copilot-upgrade-plan",
    "signUpUrl": "https://aka.ms/github-sign-up",
    "provider": {
      "default": { "id": "github", "name": "GitHub" },
      "enterprise": { "id": "github-enterprise", "name": "GHE.com" },
      "google": { "id": "google", "name": "Google" },
      "apple": { "id": "apple", "name": "Apple" }
    },
    "providerUriSetting": "github-enterprise.uri",
    "providerScopes": [
      ["user:email"],
      ["read:user"],
      ["read:user", "user:email", "repo", "workflow"]
    ],
    "entitlementUrl": "https://api.github.com/copilot_internal/user",
    "entitlementSignupLimitedUrl": "https://api.github.com/copilot_internal/subscribe_limited_user",
    "chatQuotaExceededContext": "github.copilot.chat.quotaExceeded",
    "completionsQuotaExceededContext": "github.copilot.completions.quotaExceeded",
    "walkthroughCommand": "github.copilot.open.walkthrough",
    "completionsMenuCommand": "github.copilot.toggleStatusMenu",
    "completionsRefreshTokenCommand": "github.copilot.signIn",
    "chatRefreshTokenCommand": "github.copilot.refreshToken",
    "completionsAdvancedSetting": "github.copilot.advanced",
    "completionsEnablementSetting": "github.copilot.enable",
    "nextEditSuggestionsSetting": "github.copilot.nextEditSuggestions.enabled"
  },
  "extensionGallery": {
    "serviceUrl": "https://marketplace.visualstudio.com/_apis/public/gallery",
    "itemUrl": "https://marketplace.visualstudio.com/items",
    "cacheUrl": "https://vscode.blob.core.windows.net/gallery/index",
    "controlUrl": "",
    "recommendationsUrl": ""
  },
  "trustedExtensionAuthAccess": {
    "github": [
      "github.copilot-chat"
    ]
  }
}
```

### 4. Fix pre-existing build error in VS Code

The file `.vscode/extensions/vscode-extras/src/extension.ts` has an unused `context` parameter that fails TypeScript strict checks. Prefix it with an underscore:

Replace `context` with `_context` on lines 13, 44, 45, 46 of that file.

### 5. Build and first-launch Code OSS

```bash
cd ~/vscode
./scripts/code.sh
```

First run downloads Electron (~100MB) and compiles the full VS Code source. Subsequent launches are fast. Close this window after verifying it opens.

### 6. Install the Copilot Chat extension

```bash
cd ~/vscode-copilot-chat
npm install
npm run get_token  # GitHub OAuth device flow, writes to .env
```

### 7. Start the extension watch build

In one terminal tab:

```bash
nvm use 22.22.0
cd ~/vscode-copilot-chat
npm run watch
```

Leave running. This compiles TypeScript and bundles with esbuild incrementally.

### 8. Launch Code OSS with the extension

In a second terminal tab:

```bash
nvm use 22.22.0
~/vscode/scripts/code.sh --extensionDevelopmentPath=$HOME/vscode-copilot-chat
```

### 9. Sign in to GitHub Copilot

In the Code OSS window: Accounts icon (bottom-left) > Sign in with GitHub. Requires an active Copilot subscription.

## Gotchas

| Gotcha | Symptom | Fix |
|--------|---------|-----|
| Wrong Node version | `ERR_UNKNOWN_FILE_EXTENSION ".ts"` during `npm install` or `npm run watch` | `nvm use 22.22.0` — must be done in **each terminal tab** unless set as default |
| Missing `defaultChatAgent` in overrides | "An error occurred while setting up chat" dialog | Add the full `defaultChatAgent` block to `product.overrides.json` |
| Missing `extensionGallery` in overrides | "No extension gallery service configured" | Add `extensionGallery` block with Marketplace URLs to `product.overrides.json` |
| Only `trustedExtensionAuthAccess` in overrides | Both errors above | CONTRIBUTING.md only shows auth access; you need all three blocks |
| Unused variable in vscode-extras | `TS6133: 'context' is declared but its value is never read` | Rename `context` to `_context` in the file |
| `sed` backslash on macOS | `TS1127: Invalid character` — literal backslashes inserted | macOS `sed` interprets `\_` differently; use a text editor or `sed 's/context/_context/'` without escaping |
| DNS failure downloading Electron | `getaddrinfo ENOTFOUND github.com` | Transient network issue; check VPN/proxy, retry |
| `code` command opens Cursor | Cursor overwrites the `code` symlink | Use full path `~/vscode/scripts/code.sh` or set an alias |
| `npm run get_token` not found | Running in `~/vscode` instead of `~/vscode-copilot-chat` | Script exists only in the extension repo |

## Quick Reference

| Task | Command |
|------|---------|
| Set Node version | `nvm use 22.22.0` |
| Install VS Code deps | `cd ~/vscode && npm install` |
| Install extension deps | `cd ~/vscode-copilot-chat && npm install` |
| Get OAuth token | `cd ~/vscode-copilot-chat && npm run get_token` |
| Start extension watch | `cd ~/vscode-copilot-chat && npm run watch` |
| Launch Code OSS + extension | `~/vscode/scripts/code.sh --extensionDevelopmentPath=$HOME/vscode-copilot-chat` |
| Build Code OSS without launching | `cd ~/vscode && node build/lib/preLaunch.ts` |
| Skip pre-launch (use existing build) | `VSCODE_SKIP_PRELAUNCH=1 ~/vscode/scripts/code.sh --extensionDevelopmentPath=$HOME/vscode-copilot-chat` |

## Architecture Note

The Copilot Chat UI cannot run independently from VS Code. All UI surfaces (chat panel, inline chat, suggestions) are rendered by VS Code via its native `vscode.chat.*` proposed APIs. The extension provides logic; VS Code provides rendering. Development requires running the extension inside Code OSS.
