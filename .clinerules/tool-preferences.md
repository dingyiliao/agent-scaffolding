---
name: tool-preferences
description: Hard rules for which tools Cline must prefer in this workspace. Native file-IO tools are slow and context-hungry; bash via execute_command is the default. This rule is binding — deviations require explicit reasons.
---

# Tool Preferences (binding)

## Why this exists

Cline's native `read_file` / `write_to_file` / `search_files` / `list_files` are convenient but cost real tokens and wall-time:

- `read_file` defaults to reading **up to 1000 lines** of any file — without `start_line` / `end_line` it floods context.
- `write_to_file` requires the model to emit the **complete file content** and Cline streams + renders a diff live — overhead is linear in file size even when only a few lines change.
- `search_files` / `list_files` go through extra XML formatting and UI rendering for results.

`execute_command` (bash) bypasses all of that — output goes through Cline's terminal capture and is much leaner. This rule makes bash the default and pins down the few cases where native tools are still right.

**Shell assumption**: examples below use **bash** (Git Bash / WSL / Linux / macOS). The user's primary shell is bash. PowerShell-specific differences are called out where they exist. **Never run bash-only syntax (heredoc, `$(...)`, `2>&1`) in PowerShell** — check `$SHELL` or `echo $0` if uncertain.

---

## Reading files

### MUST

- **NEVER call `read_file` without `start_line` AND `end_line`** unless you have already confirmed the file is ≤ 200 lines. Calling `read_file` with no range on an unknown-size file is forbidden.
- For files of unknown size, **probe first**:
  ```bash
  wc -l path/to/file
  ```
  Then decide:
  - ≤ 200 lines → `read_file` is fine (no range needed)
  - 200–1000 lines → `read_file` with explicit `start_line` / `end_line`
  - \> 1000 lines → use bash to read targeted ranges only (see below)

- **Extracting a specific line range** (preferred for large files):
  ```bash
  sed -n '120,180p' path/to/file
  ```

- **Reading around a symbol / pattern** (faster than reading then scanning):
  ```bash
  grep -n -B 3 -A 20 'func MyHandler' path/to/file.go
  ```

- **Reading file head / tail**:
  ```bash
  head -n 50 path/to/file
  tail -n 50 path/to/file
  ```

### NEVER

- Do not read a file "to get oriented" without first checking its size — large config / generated files (lock files, snapshots, transcripts) can be 10k+ lines and will blow context in one call.
- Do not chain multiple full-file `read_file` calls when a single `grep -rn` would find what you need.

### PowerShell note
- `sed` / `head` / `tail` are present on Git Bash and WSL. PowerShell native equivalent: `Get-Content path -TotalCount 50` (head) / `-Tail 50` (tail). Range: `Get-Content path | Select-Object -Index (120..180)`. **Prefer staying in bash.**

---

## Searching

### MUST

- **Use `rg` (ripgrep) if available; otherwise `grep -rn`**. Avoid `search_files` for routine code search.
- Examples:
  ```bash
  # Find a function definition across the repo
  rg -n 'func MyHandler\(' --type go

  # Find all callers of an exported symbol
  rg -n '\bMyHandler\b' --type-add 'src:*.{go,ts,tsx}' --type src

  # Count occurrences
  rg -c 'TODO' --type go
  ```

- Always include `-n` (line numbers) and a file-type filter — unfiltered searches in `node_modules` / `vendor` / `dist` waste output.

### When `search_files` IS allowed

- The result needs to be rendered in the chat UI as a structured artifact (rare).
- You're working with model variants that don't reliably parse bash output (rare).

In practice this means **almost never** — default to `rg` / `grep`.

---

## Listing directories

### MUST

- Use `ls -la` for flat listings, `find` for filtered recursion, `tree` (if available) for trees.
- Examples:
  ```bash
  ls -la src/
  find src -type f -name '*.ts' -not -path '*/node_modules/*' | head -50
  tree -L 3 -I 'node_modules|dist|build' src/
  ```

- Do not call `list_files` unless you specifically need its directory-tree-with-types format (rare).

### NEVER

- Do not call `list_files` recursively on a repo root without filters — it walks the whole tree.

---

## Writing files

There are two distinct cases. Pick the right one.

### Case A — Creating a new file or full overwrite

**MUST use bash heredoc, not `write_to_file`**:

```bash
cat > path/to/new-file.ts <<'EOF'
import { foo } from './foo'

export function bar() {
  return foo()
}
EOF
```

- Use `<<'EOF'` (single-quoted) to disable variable expansion — content goes in literally.
- For content with `$` / backticks meant to be literal, single-quoted heredoc is mandatory.
- For files containing the literal sequence `EOF`, change the delimiter (e.g. `<<'END_OF_FILE'`).

**Why**: `write_to_file` requires the model to re-emit the entire file in XML format and Cline streams + renders it. Heredoc bypasses both — bytes go straight to disk.

### Case B — Small edits to an existing file (1–20 changed lines)

**MUST use `replace_in_file`, not bash `sed`**:

- `replace_in_file` matches exact text and is robust against whitespace / escape errors.
- `sed -i 's/.../.../'` on real code is fragile (regex escaping, path separators, multiline limits) and routinely breaks files. Don't.

### Case C — Larger edits to an existing file (> 30% rewritten)

Read the file first (using ranges as per the Reading section), construct the full new content, and use **heredoc (Case A)** to overwrite. Faster than chaining many `replace_in_file` blocks.

### NEVER

- Never use `write_to_file` for a file > 100 lines unless explicitly requested by the user (e.g. "regenerate this from scratch"). The streaming overhead is the reason this rule exists.
- Never use `sed -i` for editing source code. It's a trap.

### PowerShell note

- Single-quoted heredoc `<<'EOF'` is bash-only. PowerShell equivalent is here-string `@'...'@`. **Prefer bash** for file content — heredoc semantics are stable across Git Bash / WSL / Linux / macOS.

---

## When native tools are still right

The following are the **only** exceptions to the bash-first default:

| Native tool | When to use it | Why |
|---|---|---|
| `read_file` (with ranges) | Files ≤ 1000 lines, when you want Cline's line-numbered output for citing in chat | Line labels (`42 | const x = 1`) are convenient for human readers |
| `replace_in_file` | Small targeted edits (1–20 changed lines) on existing files | Safer than `sed`; no escape pitfalls |
| `apply_patch` | When a unified diff is already in hand | Designed for this case |
| `browser_action` | Browser automation tasks | No bash substitute |
| `web_fetch` / `web_search` | External lookups | Native is required |
| `use_mcp_tool` / `use_skill` | MCP / skill invocation | Native is required |
| `ask_followup_question` / `attempt_completion` | Conversation control | Native is required |

Everything else (reading large files, searching, listing, creating files, full rewrites) **defaults to `execute_command`**.

---

## Quick decision table

| Task | Default | Fallback / when |
|---|---|---|
| Read ≤ 200 lines of known file | `read_file` (no range) | — |
| Read 200–1000 lines | `read_file` with range | — |
| Read > 1000 lines | `sed -n 'M,Np'` / `head` / `tail` | — |
| Read around a symbol | `grep -n -B/-A` | `read_file` after locating |
| Search code | `rg -n` / `grep -rn` | `search_files` (rare) |
| List dir | `ls -la` / `find` / `tree` | — |
| Create new file | `cat > path <<'EOF'` | — |
| Small edit (1–20 lines) | `replace_in_file` | — |
| Large rewrite | heredoc overwrite | — |
| Apply pre-made diff | `apply_patch` | — |

---

## Violations

If a tool call violates this rule, **revise before sending**. Do not "explain why this once it's OK" — the rule is the rule. Real exceptions:

- The user explicitly asks for a specific native tool ("use write_to_file for this").
- The environment lacks bash entirely (rare).
- The task is one the table above marks as native-only.

Otherwise: bash wins.
