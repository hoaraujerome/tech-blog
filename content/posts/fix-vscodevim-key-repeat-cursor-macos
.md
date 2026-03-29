+++
date = '2025-07-13T09:47:01-04:00'
title = 'Fix Vim key repeat in Cursor, Zed, VS Code on macOS'
+++

Vim extension in Cursor, Zed, or VS Code on macOS — arrow keys and held `hjkl` don't repeat. macOS disables key repeat for some Electron apps by default.

**Fix:** disable `ApplePressAndHoldEnabled` for the app (keeps accent popup off, enables key repeat).

1. Get the app's bundle ID:

```bash
osascript -e 'id of app "Cursor"'   # or "Zed" or "Visual Studio Code"
```

2. Enable key repeat for that app:

```bash
defaults write <BUNDLE_ID> ApplePressAndHoldEnabled -bool false
```

Examples:
- **Cursor:** `com.todesktop.230313mzl4w4u92`
- **Zed:** `dev.zed.Zed`
- **VS Code:** `com.microsoft.VSCode`

3. Restart the app.

To reset a global override: `defaults delete -g ApplePressAndHoldEnabled`
