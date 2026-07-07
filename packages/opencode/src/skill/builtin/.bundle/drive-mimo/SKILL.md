---
name: drive-mimo
description: Use when you need to programmatically drive another MiMoCode (mimo) process — supports both headless `mimo run` with JSON events and interactive TUI via tmux for full terminal interaction testing. Reach for it to script, test, or automate a separate mimo instance and validate its behavior from parseable evidence.
---

# Drive MiMo

## Overview

Two modes for driving a separate `mimo` process:

| Mode | Command | Use when |
|------|---------|----------|
| **Headless** | `mimo run --format json` | Scripted tasks, CI, event validation |
| **TUI** | `mimo` in tmux | Interactive flow, permission dialogs, keybindings, visual regression |

**Core principle:** Every run produces parseable evidence. No eyeballing.

Always drive an *isolated* instance: give each run a fresh `MIMOCODE_HOME` and a throwaway workspace so it never touches your own config, memory, or session DB.

## Prerequisites

```bash
# mimo binary must be on PATH
which mimo || echo "mimo not found on PATH"

# tmux (for TUI mode)
which tmux || echo "tmux not found — install it for TUI mode"
```

---

## Part 1: Headless Mode (`mimo run`)

### Launch

```bash
PROMPT=$(mktemp -t mimo-drive.XXXXXX)
cat >"$PROMPT" <<'EOF'
Your task here.
EOF

MIMOCODE_HOME=$(mktemp -d) mimo run \
  --format json \
  --dangerously-skip-permissions \
  --dir "$WORKSPACE" \
  < "$PROMPT" > /tmp/mimo-out.jsonl 2>&1

EXIT=$?
```

### Key flags

| Flag | Purpose |
|------|---------|
| `--format json` | Structured JSONL events to stdout |
| `--dangerously-skip-permissions` | Auto-approve all permissions |
| `--model provider/model` | Override model |
| `--agent compose` | Use compose agent |
| `--session SID` | Continue existing session |
| `--continue` | Continue last session |
| `--file path` | Attach file to message |
| `--dir path` | Working directory |

### JSON event types

```
{"type":"session.id","session":{"id":"ses_abc"}}
{"type":"text","text":"I'll create the file..."}
{"type":"reasoning","reasoning":"The user wants..."}
{"type":"tool_use","tool":{"name":"write","input":{...}}}
{"type":"tool_result","tool":{"name":"write","output":"..."}}
{"type":"error","error":"Permission denied"}
{"type":"session.status","status":{"type":"idle"}}
```

`session.status` with `type: "idle"` = completion signal.

### Validation patterns

```bash
# Exit code
[ $EXIT -eq 0 ] || echo "FAIL: exit $EXIT"

# No errors
grep -q '"type":"error"' /tmp/mimo-out.jsonl && echo "FAIL: errors found"

# Reached completion
grep -q '"session.status"' /tmp/mimo-out.jsonl || echo "FAIL: never completed"

# Tool was used
grep -q '"name":"write"' /tmp/mimo-out.jsonl || echo "FAIL: write tool not called"

# Text output contains expected string
grep '"type":"text"' /tmp/mimo-out.jsonl | jq -r '.text' | grep -q "expected"
```

### Timeout

```bash
timeout 120 mimo run --format json --dangerously-skip-permissions < "$PROMPT"
[ $? -eq 124 ] && echo "FAIL: timed out"
```

---

## Part 2: TUI Mode (tmux)

### Isolation variables

| Variable | Purpose |
|---|---|
| `MIMOCODE_HOME` | **Required.** Fresh `mktemp -d` per run. Sandboxes DB, config, cache. |
| `MIMOCODE_PURE=true` | Disable external plugins. |
| `MIMOCODE_DISABLE_GIT=true` | Skip git ops if workspace isn't a real repo. |

### Launch TUI in tmux

```bash
# Create isolated environment
MHOME=$(mktemp -d)
WORKSPACE=$(mktemp -d)
SESSION="mimo-drive-$$"

# Launch mimo TUI in tmux (workspace is a positional arg, NOT --dir)
tmux new-session -d -s "$SESSION" -x 120 -y 30 \
  "MIMOCODE_HOME=$MHOME MIMOCODE_PURE=true mimo $WORKSPACE; sleep 999"

# Wait for the TUI to render its input prompt
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "$PROMPT_RE" -T 15
```

Where `PROMPT_RE` is a language-neutral pattern for the input prompt. The
prompt placeholder is localized, so match on the stable markers rather than a
localized string:

```bash
# Matches the prompt line regardless of UI language:
#   the ">" input caret, the "Ask" English placeholder, or a "/" command hint
PROMPT_RE='>|Ask|/[a-z]'
```

If a run's UI language is known and fixed, you may match its literal
placeholder instead — but prefer the neutral pattern for portability.

### Send input

```bash
# Type a message (literal text, then Enter)
tmux send-keys -t "$SESSION:0.0" -l -- "Create a file called hello.txt with content 'world'"
tmux send-keys -t "$SESSION:0.0" Enter

# Special keys
tmux send-keys -t "$SESSION:0.0" C-c          # Cancel
tmux send-keys -t "$SESSION:0.0" C-d          # EOF
tmux send-keys -t "$SESSION:0.0" Escape       # Escape
tmux send-keys -t "$SESSION:0.0" Tab          # Tab completion
tmux send-keys -t "$SESSION:0.0" Up           # History up
tmux send-keys -t "$SESSION:0.0" Down         # History down
tmux send-keys -t "$SESSION:0.0" Enter        # Submit
```

### Capture output

```bash
# Current screen
tmux capture-pane -t "$SESSION:0.0" -p

# Full scrollback
tmux capture-pane -t "$SESSION:0.0" -p -S -

# Last 50 lines
tmux capture-pane -t "$SESSION:0.0" -p -S -50
```

### Wait for state changes

```bash
# Wait for agent to start processing
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "thinking\|reading\|writing" -T 30

# Wait for tool permission prompt
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "Allow\|Deny\|permission\|approve" -T 30

# Wait for completion
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "completed\|done\|finished\|idle" -T 120

# Wait for error
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "error\|failed\|Error" -T 30

# Custom regex
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "hello\.txt.*world" -T 30
```

### Handle permission dialogs

```bash
# Wait for permission prompt, then approve
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "Allow\|approve" -T 30
tmux send-keys -t "$SESSION:0.0" -l -- "y"
tmux send-keys -t "$SESSION:0.0" Enter

# Or deny
tmux send-keys -t "$SESSION:0.0" -l -- "n"
tmux send-keys -t "$SESSION:0.0" Enter
```

### Multi-turn interaction

```bash
# Turn 1: send initial message
tmux send-keys -t "$SESSION:0.0" -l -- "Create a TypeScript file that adds two numbers"
tmux send-keys -t "$SESSION:0.0" Enter
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "completed\|done" -T 120

# Turn 2: follow-up
tmux send-keys -t "$SESSION:0.0" -l -- "Now add a test for it"
tmux send-keys -t "$SESSION:0.0" Enter
scripts/wait-for-text.sh -t "$SESSION:0.0" -p "completed\|done" -T 120

# Verify result
tmux capture-pane -t "$SESSION:0.0" -p -S - | grep -i "pass\|fail"
```

### Cleanup

```bash
tmux kill-session -t "$SESSION" 2>/dev/null
rm -rf "$MHOME" "$WORKSPACE"
```

---

## Part 3: Scenarios

### S1: Smoke (headless)

```bash
test_smoke() {
  local P=$(mktemp) MHOME=$(mktemp -d)
  echo "Say hello" > "$P"
  MIMOCODE_HOME=$MHOME mimo run --format json --dangerously-skip-permissions \
    < "$P" > /tmp/s1.jsonl 2>&1
  local E=$?
  rm -rf "$MHOME" "$P"
  [ $E -eq 0 ] && grep -q '"type":"text"' /tmp/s1.jsonl && echo "PASS" || echo "FAIL"
}
```

### S2: Tool use (headless)

```bash
test_tool_use() {
  local P=$(mktemp) MHOME=$(mktemp -d) WS=$(mktemp -d)
  echo 'Create file test.txt with content "hello"' > "$P"
  MIMOCODE_HOME=$MHOME mimo run --format json --dangerously-skip-permissions --dir "$WS" \
    < "$P" > /tmp/s2.jsonl 2>&1
  local E=$?
  local OK=true
  [ $E -ne 0 ] && OK=false
  ! grep -q '"name":"write"' /tmp/s2.jsonl && OK=false
  [ ! -f "$WS/test.txt" ] && OK=false
  rm -rf "$MHOME" "$WS" "$P"
  $OK && echo "PASS" || echo "FAIL"
}
```

### S3: TUI interactive flow

```bash
test_tui_interactive() {
  local MHOME=$(mktemp -d) WS=$(mktemp -d) SID="mimo-s3-$$"
  local PROMPT_RE='>|Ask|/[a-z]'
  tmux new-session -d -s "$SID" -x 120 -y 30 "MIMOCODE_HOME=$MHOME MIMOCODE_PURE=true mimo $WS; sleep 999"

  # Wait for prompt
  scripts/wait-for-text.sh -t "$SID:0.0" -p "$PROMPT_RE" -T 15 || { echo "FAIL: no prompt"; tmux kill-session -t $SID; return 1; }

  # Send task
  tmux send-keys -t "$SID:0.0" -l -- "Create hello.txt with content 'world'"
  tmux send-keys -t "$SID:0.0" Enter

  # Wait for completion (agent shows elapsed time like "· 9.8s")
  scripts/wait-for-text.sh -t "$SID:0.0" -p "· [0-9]" -T 120 || { echo "FAIL: no completion"; tmux kill-session -t $SID; return 1; }

  # Verify file
  [ -f "$WS/hello.txt" ] && echo "PASS" || echo "FAIL"
  tmux kill-session -t "$SID" 2>/dev/null
  rm -rf "$MHOME" "$WS"
}
```

### S4: TUI permission handling

```bash
test_tui_permission() {
  local MHOME=$(mktemp -d) WS=$(mktemp -d) SID="mimo-s4-$$"
  local PROMPT_RE='>|Ask|/[a-z]'
  tmux new-session -d -s "$SID" -x 120 -y 30 "MIMOCODE_HOME=$MHOME MIMOCODE_PURE=true mimo $WS; sleep 999"

  scripts/wait-for-text.sh -t "$SID:0.0" -p "$PROMPT_RE" -T 15

  # Ask for something that needs permission (no --dangerously-skip-permissions in TUI)
  tmux send-keys -t "$SID:0.0" -l -- "Run the command: echo hello"
  tmux send-keys -t "$SID:0.0" Enter

  # Wait for permission prompt
  scripts/wait-for-text.sh -t "$SID:0.0" -p "Allow\|approve\|permission\|y/n" -T 30 || { echo "FAIL: no permission prompt"; tmux kill-session -t $SID; return 1; }

  # Approve
  tmux send-keys -t "$SID:0.0" -l -- "y"
  tmux send-keys -t "$SID:0.0" Enter

  # Wait for completion
  scripts/wait-for-text.sh -t "$SID:0.0" -p "· [0-9]" -T 60

  tmux capture-pane -t "$SID:0.0" -p | grep -q "hello" && echo "PASS" || echo "FAIL"
  tmux kill-session -t "$SID" 2>/dev/null
  rm -rf "$MHOME" "$WS"
}
```

### S5: TUI keybindings

```bash
test_tui_keybindings() {
  local MHOME=$(mktemp -d) WS=$(mktemp -d) SID="mimo-s5-$$"
  local PROMPT_RE='>|Ask|/[a-z]'
  tmux new-session -d -s "$SID" -x 120 -y 30 "MIMOCODE_HOME=$MHOME MIMOCODE_PURE=true mimo $WS; sleep 999"

  scripts/wait-for-text.sh -t "$SID:0.0" -p "$PROMPT_RE" -T 15

  # Test Ctrl+C cancels input
  tmux send-keys -t "$SID:0.0" -l -- "some partial input"
  tmux send-keys -t "$SID:0.0" C-c

  # Screen should still show prompt (not exit)
  sleep 1
  tmux capture-pane -t "$SID:0.0" -p | grep -qE "$PROMPT_RE" && echo "PASS: Ctrl+C didn't exit" || echo "FAIL: Ctrl+C exited"

  # Test Escape
  tmux send-keys -t "$SID:0.0" Escape
  sleep 0.5

  tmux kill-session -t "$SID" 2>/dev/null
  rm -rf "$MHOME" "$WS"
}
```

---

## Batch Runner

```bash
run_all() {
  local PASS=0 FAIL=0 RESULTS=()
  for fn in test_smoke test_tool_use test_tui_interactive test_tui_permission test_tui_keybindings; do
    echo "--- $fn ---"
    if $fn 2>/dev/null; then PASS=$((PASS+1)); else FAIL=$((FAIL+1)); RESULTS+=("$fn: FAIL"); fi
  done
  echo "=== $PASS passed, $FAIL failed ==="
  [ ${#RESULTS[@]} -gt 0 ] && printf '  %s\n' "${RESULTS[@]}"
  [ $FAIL -eq 0 ]
}
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Headless run | `MIMOCODE_HOME=$(mktemp -d) mimo run --format json --dangerously-skip-permissions "prompt"` |
| TUI launch | `tmux new-session -d -s test -x 120 -y 30 "MIMOCODE_HOME=$(mktemp -d) mimo $WORKSPACE; sleep 999"` |
| Send text | `tmux send-keys -t test:0.0 -l -- "text" && tmux send-keys -t test:0.0 Enter` |
| Capture screen | `tmux capture-pane -t test:0.0 -p -S -` |
| Wait for text | `scripts/wait-for-text.sh -t test:0.0 -p "pattern" -T 30` |
| Send Ctrl+C | `tmux send-keys -t test:0.0 C-c` |
| Approve permission | `tmux send-keys -t test:0.0 -l -- "y" && tmux send-keys -t test:0.0 Enter` |
| Cleanup | `tmux kill-session -t test && rm -rf $MHOME` |
