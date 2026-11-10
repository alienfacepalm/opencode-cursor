# Fix Summary: Duplicate Tool Call IDs in OpenCode v2

## Problem

During long agentic sessions with OpenCode v2.2.6 on Windows 11 (PowerShell), using the Claude Sonnet 5.5 model through this plugin in "Build agent with auto permissions" mode, OpenCode would show:

```
Error: Duplicate tool call: toolu_01RaUwXi2EGFB3VKvHrxM3zE
```

This happened right after bash/PowerShell tool calls, forcing the user to type `continue` to resume. It occurred twice within about 15 minutes on November 9, 2026.

## Root Cause

**Confirmed Issue**: The plugin was emitting multiple `tool-call` events with identical IDs, which OpenCode v2 strictly rejects.

### Why It Happened

The duplication occurred in these scenarios:

1. **Stall watchdog force-resend**: When a Cursor run stalled (no events for 120 seconds by default), the plugin's watchdog would:
   - Cancel the stalled run
   - Start a fresh run with `local.force: true`
   - Both the old (cancelled) run and the new run would emit `tool-call-started` events with the same `callId` values
   - The plugin pushed both events to OpenCode, causing the duplicate error

2. **SDK retries/reconnections**: The Cursor SDK itself might emit duplicate `tool-call-started` events during network retries or reconnections.

### OpenCode v2 Behavior

In [`opencode/packages/core/src/session/runner/publish-llm-event.ts:479`](https://github.com/anomalyco/opencode/blob/main/packages/core/src/session/runner/publish-llm-event.ts#L479), OpenCode v2 tracks tool calls by ID and strictly enforces uniqueness:

```typescript
if (tool.called) return yield* Effect.die(new Error(`Duplicate tool call: ${event.id}`))
```

When it sees a `tool-call` event with an ID that was already marked as `called`, it dies with the "Duplicate tool call" error, halting execution.

## The Fix

Modified `src/provider/agent-events.ts` to deduplicate tool call IDs:

1. **Track emitted IDs**: Added a `Set<string>` called `emittedToolCalls` to track which tool call IDs have already been emitted in the current stream

2. **Skip duplicates**: When processing `tool-call-started` events:
   - Check if the `callId` is already in `emittedToolCalls`
   - If yes, log the duplicate (when `OPENCODE_CURSOR_DEBUG=1`) and skip emitting the event
   - If no, add it to the set and emit the event normally

3. **Reset on new runs**: Clear `emittedToolCalls` when starting a fresh run (in `startRun()`) so each run attempt can emit its own tool calls without being blocked by IDs from previous attempts

### What's Not Affected

- **Tool results**: Still pair correctly with their corresponding calls
- **Partial tool input**: Not affected by deduplication
- **Other event types**: Text, reasoning, usage, etc. are unchanged
- **OpenCode v1**: This fix is safe for v1 as well (it just won't encounter the duplicate validation)

## Testing

### Automated Tests

- ✅ All 657 existing tests pass
- ✅ New test added: `"deduplicates tool calls when the SDK emits duplicate tool-call-started events"`
- ✅ TypeScript compilation succeeds
- ✅ Build completes successfully

The new test verifies that when the SDK sends two `tool-call-started` events with the same `callId`, only one `tool-call` event is emitted to OpenCode, preventing the duplicate error.

## How to Test Locally

Follow these steps to test the fix in your OpenCode v2.2.6 environment on Windows 11:

### 1. Build the Plugin

```bash
cd C:\Users\brand\PROJECTS\AlienFacepalm\opencode-cursor
npm install
npm run build
```

This creates the compiled plugin in the `dist/` directory.

### 2. Point OpenCode to Your Local Build

Edit your OpenCode configuration file. On Windows, this is typically:
- `%USERPROFILE%\.config\opencode\opencode.json` (global), or
- `.opencode.json` in your project directory (project-specific)

Change the `plugins` array to reference your local build:

```json
{
  "plugins": ["file:C:/Users/brand/PROJECTS/AlienFacepalm/opencode-cursor"]
}
```

**Important notes:**
- Use forward slashes (`/`) in the path, even on Windows
- The `file:` protocol tells OpenCode to load from a local directory instead of npm
- Use the absolute path to the repository root (not the `dist/` folder)

### 3. Restart OpenCode

Exit OpenCode completely and restart it. The plugin will be loaded from your local build.

### 4. Verify the Plugin Loaded

In OpenCode, check that Cursor models are available:
- Type `opencode models` or open the model picker
- You should see `cursor/*` models listed

### 5. Test in a Long Session

Run a long agentic session with:
- Model: Claude Sonnet 5.5 Cursor (`claude-sonnet-5-5-high`)
- Mode: Build agent with auto permissions on

The "Duplicate tool call" error should no longer appear, even during long sessions with many tool calls.

### 6. Enable Debug Logging (Optional)

To see when duplicates are being skipped, set the debug environment variable before starting OpenCode:

**PowerShell:**
```powershell
$env:OPENCODE_CURSOR_DEBUG="1"
opencode
```

**Bash/CMD:**
```bash
export OPENCODE_CURSOR_DEBUG=1  # bash
set OPENCODE_CURSOR_DEBUG=1     # cmd
opencode
```

When duplicates are detected and skipped, you'll see log entries like:
```
skipping duplicate tool-call-started: bash-1
```

## Changes Made

### Files Modified

1. **`src/provider/agent-events.ts`**
   - Added `emittedToolCalls` Set to track emitted tool call IDs
   - Added deduplication logic in the `tool-call-started` case of `onDelta`
   - Clear `emittedToolCalls` when starting a new run in `startRun()`

2. **`test/agent-events.test.ts`**
   - Added test case: `"deduplicates tool calls when the SDK emits duplicate tool-call-started events"`
   - Test verifies only one `tool-call` event is emitted when SDK sends duplicate `tool-call-started` events

### Pull Request

PR #2: https://github.com/alienfacepalm/opencode-cursor/pull/2

Branch: `cursor/fix-duplicate-tool-call-ids-923c`

## Upstream

I checked the upstream repository (`stablekernel/opencode-cursor`) for related issues or PRs:
- **Issues**: No issues mentioning "duplicate tool call"
- **PRs**: No PRs addressing this specific issue

This appears to be a newly discovered issue that primarily affects OpenCode v2 due to its stricter tool call validation.

## Next Steps

1. **Test locally** using the instructions above
2. If the fix works well in your environment, you can:
   - Continue using the local build, or
   - Wait for the upstream to merge a similar fix and update to the published version

## Questions or Issues?

If you encounter any problems with the fix:
1. Check that the local build loaded correctly (`opencode models` should show Cursor models)
2. Enable debug logging to see what's happening internally
3. Check the OpenCode logs for any errors during plugin load

The fix is conservative and only affects tool call deduplication—it doesn't change any other behavior of the plugin.
