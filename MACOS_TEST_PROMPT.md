# macOS Integration Test Session — AlphaClaw

## Context

The `claude/macos-support-improvements-JvuCN` branch on `brandongilchrist/alphaclaw` adds full macOS support to AlphaClaw. Previously, AlphaClaw was Linux/Docker-only due to hardcoded cgroup paths, `/etc/cron.d/`, systemctl shims, and `/usr/local/bin/` install targets.

The changes span 7 files (230 insertions) across:

- **`lib/server/constants.js`** — New exports: `kPlatform`, `kIsMacOS`, `kIsLinux`
- **`lib/server/system-resources.js`** — macOS `vm_stat` memory parsing, skip cgroups on darwin
- **`bin/alphaclaw.js`** — LaunchAgent plist instead of `/etc/cron.d/`, git/gog shims install to `~/.alphaclaw/bin/`, systemctl shim skipped on macOS
- **`lib/scripts/git`** — Uses `$TMPDIR` instead of hardcoded `/tmp/`
- **`lib/scripts/systemctl`** — Uses `$ALPHACLAW_ROOT_DIR` for lock paths
- **`lib/setup/hourly-git-sync.sh`** — Adds Homebrew + `~/.alphaclaw/bin` to PATH on macOS, portable `date -u`
- **`README.md`** — Updated platform support docs

## Your Task

You are running on a local Mac. Clone this branch and systematically validate every macOS-specific feature. For each test, report PASS or FAIL with details. If something fails, diagnose the root cause and fix it on-the-spot (commit fixes to the same branch).

## Prerequisites

- Node.js >= 22.12.0 (`node --version`)
- Git
- macOS (verify with `uname -s`)

## Setup

```bash
git clone https://github.com/brandongilchrist/alphaclaw.git
cd alphaclaw
git checkout claude/macos-support-improvements-JvuCN
npm install
```

## Tests to Run (in order)

### 1. Test Suite (GIL-55)
```bash
npm test
```
All 297 tests should pass. Report any failures.

### 2. Platform Detection (GIL-56)
```bash
node -e "
const c = require('./lib/server/constants');
console.log('kPlatform:', c.kPlatform);
console.log('kIsMacOS:', c.kIsMacOS);
console.log('kIsLinux:', c.kIsLinux);
console.assert(c.kPlatform === 'darwin', 'Expected darwin');
console.assert(c.kIsMacOS === true, 'Expected kIsMacOS true');
console.assert(c.kIsLinux === false, 'Expected kIsLinux false');
console.log('PASS');
"
```

### 3. System Resources (GIL-57)
```bash
node -e "
const { getSystemResources } = require('./lib/server/system-resources');
const r = getSystemResources();
console.log(JSON.stringify(r, null, 2));
console.assert(r.memory.usedBytes > 0, 'memory.usedBytes should be > 0');
console.assert(r.memory.totalBytes > 0, 'memory.totalBytes should be > 0');
console.assert(r.memory.percent > 0, 'memory.percent should be > 0');
console.assert(r.cpu.percent !== null, 'cpu.percent should not be null');
console.assert(r.cpu.cores > 0, 'cpu.cores should be > 0');
console.assert(r.disk.usedBytes > 0, 'disk.usedBytes should be > 0');
console.assert(r.disk.totalBytes > 0, 'disk.totalBytes should be > 0');
console.log('PASS');
"
```

### 4. LaunchAgent Plist (GIL-58)

This requires running the startup code path. Create a minimal test:
```bash
# Create a temp workspace to avoid touching real config
export ALPHACLAW_ROOT_DIR="$(mktemp -d)/alphaclaw-test"
mkdir -p "$ALPHACLAW_ROOT_DIR/.openclaw/cron"
echo '{}' > "$ALPHACLAW_ROOT_DIR/.openclaw/cron/system-sync.json"

# The startup code in bin/alphaclaw.js installs the plist.
# After running startup (or extracting the plist logic), check:
PLIST="$HOME/Library/LaunchAgents/com.alphaclaw.hourly-sync.plist"

# Verify plist is valid XML
plutil -lint "$PLIST" 2>/dev/null && echo "PASS: valid plist" || echo "FAIL: invalid plist"

# Verify launchctl knows about it
launchctl list | grep alphaclaw && echo "PASS: launchd loaded" || echo "INFO: not loaded (expected if startup wasn't run)"

# Cleanup
launchctl bootout gui/$(id -u) "$PLIST" 2>/dev/null || true
rm -f "$PLIST"
rm -rf "$ALPHACLAW_ROOT_DIR"
```

### 5. systemctl Shim Skipped (GIL-59)
```bash
# Before running startup, note the state of /usr/local/bin/systemctl
ls -la /usr/local/bin/systemctl 2>/dev/null && echo "WARNING: systemctl exists before test" || echo "OK: no systemctl before test"

# After startup, verify it was NOT created
ls -la /usr/local/bin/systemctl 2>/dev/null && echo "FAIL: systemctl was created on macOS" || echo "PASS: systemctl correctly skipped"
```

### 6. Git Auth Shim (GIL-60)
```bash
# After startup, verify:
GIT_SHIM="$HOME/.alphaclaw/bin/git"
test -f "$GIT_SHIM" && echo "PASS: git shim exists" || echo "FAIL: git shim not found"
test -x "$GIT_SHIM" && echo "PASS: git shim is executable" || echo "FAIL: git shim not executable"

# Verify it found a valid real git
grep -o 'REAL_GIT="[^"]*"' "$GIT_SHIM" || echo "FAIL: REAL_GIT not set"

# Verify /usr/local/bin/git was NOT overwritten (if it didn't exist before)
```

### 7. GOG CLI (GIL-61)
```bash
GOG_BIN="$HOME/.alphaclaw/bin/gog"
test -f "$GOG_BIN" && echo "PASS: gog binary exists" || echo "FAIL: gog not found"
test -x "$GOG_BIN" && echo "PASS: gog is executable" || echo "FAIL: gog not executable"
file "$GOG_BIN" | grep -i "mach-o" && echo "PASS: native macOS binary" || echo "FAIL: wrong binary format"
```

### 8. hourly-git-sync.sh (GIL-62)
```bash
# Verify the script uses portable date
grep 'date -u' lib/setup/hourly-git-sync.sh && echo "PASS: uses portable date -u" || echo "FAIL"
grep 'date --utc' lib/setup/hourly-git-sync.sh && echo "FAIL: uses GNU-only --utc" || echo "PASS: no GNU-only flags"

# Verify macOS PATH additions
grep '/opt/homebrew/bin' lib/setup/hourly-git-sync.sh && echo "PASS: Homebrew in PATH" || echo "FAIL"
grep '.alphaclaw/bin' lib/setup/hourly-git-sync.sh && echo "PASS: ~/.alphaclaw/bin in PATH" || echo "FAIL"

# Test that the script parses without errors
bash -n lib/setup/hourly-git-sync.sh && echo "PASS: valid bash syntax" || echo "FAIL: syntax errors"
```

### 9. Watchdog Loop (GIL-63)
```bash
# Start the server briefly and check logs for watchdog activity
SETUP_PASSWORD=test timeout 10 node bin/alphaclaw.js start 2>&1 | tee /tmp/alphaclaw-test.log || true
grep -i "watchdog" /tmp/alphaclaw-test.log || echo "INFO: no watchdog messages (expected without gateway)"
grep -i "error\|exception\|cannot find" /tmp/alphaclaw-test.log && echo "REVIEW: check errors above" || echo "PASS: no errors"
```

### 10. Full Startup (GIL-64)
```bash
SETUP_PASSWORD=test node bin/alphaclaw.js start &
SERVER_PID=$!
sleep 5

# Test that the server is running
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/ | grep -q "200\|302" && echo "PASS: server responds" || echo "FAIL: server not responding"

# Test login endpoint exists
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/auth/status | grep -q "200\|401" && echo "PASS: auth endpoint works" || echo "FAIL: auth endpoint broken"

kill $SERVER_PID 2>/dev/null
```

## Reporting

After all tests, update each Linear sub-issue (GIL-55 through GIL-64) status:
- **Done** if PASS
- **In Progress** if you had to fix something (commit the fix first)
- Leave as **Todo** if blocked

Use the Linear API (key is in the `LINEAR_API_KEY` environment variable):
```
Authorization: $LINEAR_API_KEY
POST https://api.linear.app/graphql
```

Workflow state IDs:
- Done: `17c49c27-83ca-4272-a148-9d8393d441ff`
- In Progress: `f192ba44-bbde-40c0-b6e7-c5dcfc619483`
- Todo: `85fa80b4-7eb7-4129-ba93-1625f4e0600b`

If you need to fix anything, commit to the same branch (`claude/macos-support-improvements-JvuCN`) and push.

At the end, comment a summary on the parent issue GIL-54 (id: `d08a5927-4786-4727-a7f4-e780eebc6bb8`).
