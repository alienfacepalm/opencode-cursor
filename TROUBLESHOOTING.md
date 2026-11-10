# Troubleshooting Guide

## Issue: OpenCode doesn't load the local plugin

### Symptom
After setting `"plugins": ["file:C:/Users/brand/PROJECTS/AlienFacepalm/opencode-cursor"]`, OpenCode doesn't show Cursor models when running `opencode models`.

### Solutions

1. **Check the path format**
   - Must use forward slashes: `file:C:/Users/brand/...` (not `file:C:\Users\brand\...`)
   - Must be an absolute path
   - Must point to the repo root (where `package.json` is), not the `dist/` folder

2. **Verify the build completed**
   ```bash
   cd C:\Users\brand\PROJECTS\AlienFacepalm\opencode-cursor
   npm run build
   ```
   Check that `dist/plugin/index.js` exists

3. **Restart OpenCode completely**
   - Exit OpenCode fully (not just close the window)
   - Restart it

4. **Check OpenCode logs**
   - On Windows: `%LOCALAPPDATA%\opencode\logs\`
   - Look for plugin loading errors

5. **Try a relative path (project-specific)**
   If you're working in a project that's IN the opencode-cursor repo:
   ```json
   {
     "plugins": ["."]
   }
   ```

## Issue: Build fails with "vitest: not found"

### Solution
Run `npm install` first:
```bash
cd C:\Users\brand\PROJECTS\AlienFacepalm\opencode-cursor
npm install
npm run build
```

## Issue: Still seeing "Duplicate tool call" errors

### Diagnostics

1. **Verify the local build is being used**
   
   Check OpenCode is loading from your local path:
   ```bash
   # Enable debug mode
   $env:OPENCODE_CURSOR_DEBUG="1"
   opencode
   ```
   
   You should see startup logs showing the plugin is loaded.

2. **Check if duplicates are actually being skipped**
   
   With debug enabled, when a duplicate occurs, you should see:
   ```
   skipping duplicate tool-call-started: <id>
   ```
   
   If you DON'T see this message but still get the error, the issue might be different.

3. **Verify you're using OpenCode v2**
   
   Check your OpenCode version:
   ```bash
   opencode --version
   ```
   
   The fix is for OpenCode v2.x (tested on 2.2.6).

4. **Try a clean slate**
   
   Clear OpenCode's plugin cache:
   
   **Windows:**
   ```powershell
   Remove-Item -Recurse "$env:LOCALAPPDATA\opencode\cache\*"
   ```
   
   Then restart OpenCode.

## Issue: Tool results don't pair with tool calls

### This Should Not Happen

The fix only deduplicates `tool-call` events, not `tool-result` events. Tool results should still pair correctly.

If you see tool results without corresponding calls:
1. Enable debug logging
2. Check if there are multiple `tool-result` events with the same ID
3. Report this as a new issue—this might indicate a different bug

## Issue: Different tool call IDs but still getting duplicates

### Example
```
Error: Duplicate tool call: toolu_ABC123
Error: Duplicate tool call: toolu_XYZ789
```

If you're seeing duplicates with DIFFERENT IDs each time, this might be a different issue where:
- The Cursor SDK is generating duplicate IDs in a different way
- OpenCode is tracking tool calls differently than expected

### Diagnostic
1. Enable debug logging
2. Check if the skipping messages appear
3. If yes: The plugin is deduping correctly, but duplicates are still reaching OpenCode somehow
4. If no: The IDs might be different at the plugin level but appearing as duplicates in OpenCode

## Issue: Performance degradation

### Symptom
Long sessions feel slower after applying the fix.

### Diagnosis
The deduplication Set should have minimal overhead. Each tool call ID is a small string, and Set lookup is O(1).

If performance is an issue:
1. Check how many unique tool calls are in your session (`Set.size` would grow unboundedly without clearing)
2. Verify the Set is being cleared on new runs (it should be)
3. The Set is NOT persisted between OpenCode restarts, so closing and reopening OpenCode resets it

## Getting Help

If none of these solutions work:

1. **Capture debug output**
   ```powershell
   $env:OPENCODE_CURSOR_DEBUG="1"
   opencode 2>&1 | Tee-Object -FilePath debug.log
   ```

2. **Note the exact error message** and when it occurs (e.g., "after 3rd bash tool call")

3. **Check the tool call IDs** in the error messages—are they the same each time or different?

4. **Minimal reproduction case**: Try to find the smallest session that triggers the issue

## Reverting the Fix

If you need to revert to the published version:

1. **Edit opencode.json**
   ```json
   {
     "plugins": ["@stablekernel/opencode-cursor@latest"]
   }
   ```

2. **Clear the local plugin**
   ```powershell
   Remove-Item -Recurse "$env:LOCALAPPDATA\opencode\cache\*"
   ```

3. **Restart OpenCode**

The published version will be re-installed from npm.
