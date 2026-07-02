# Master_Continue.md

## Role
You have received either `plan.md` or a prior `continue.md` (which supersedes `plan.md`), plus
all available handoff files. Produce a single `continue.md` that gives any future session
everything it needs to fix bugs or add features, plus the exact protocol that session must follow.

---

## Instructions for building continue.md

### Part A — Project Snapshot

Distil from `plan.md` or prior `continue.md` (whichever was supplied), plus every handoff, into these sections, in order:

1. **Overview** — one paragraph: what the app does, how it runs (local dev vs Docker), entry points.
2. **Stack** — backend packages, frontend packages, infra/tooling; one line each.
3. **Ports & volumes** — reproduce the port and volume tables verbatim.
4. **File manifest** — every file in the project, one-line purpose each. No session numbers or history.
5. **Contracts** — env vars, DB schema, REST routes, SSE event format, shared TypeScript types. Reproduce verbatim.
6. **Implementation decisions** — flat deduplicated list of decisions that affect behaviour or aren't obvious from the code. No handoff attribution, no session references.

### Part B — Fix/Feature Protocol

Embed the block below **verbatim** (do not summarise or reword it):

---
<!-- BEGIN PROTOCOL — copy into continue.md unchanged -->

## Fix / Feature Protocol

When this file is loaded with an error description or feature request, follow the two phases below
in strict order.

### Phase 1 — Triage

1. Identify the **set of files** needed to diagnose or implement the change.
   Use the file manifest above; prefer precision over breadth.
2. Output **only** the bash script — no prose before or after it.
   Minimal, no comments. Each `cp` must warn to stderr and exit non-zero if the file is missing,
   and echo a concise confirmation line (file + destination) on success so progress is verifiable
   at a glance.

3. **Stop.** Do not analyse anything yet. Wait for the user to upload the temp files.

### Phase 2 — Resolution

After the user uploads the temp files:

1. Determine the root cause or exact change points.
2. Output a patch bash script. Rules:
   - **Surgical edits only.** Target the exact word, token, expression, or
     line that must change. Never rewrite a whole function or block when only
     one line inside it differs.
   - Prefer `sed -i 's/old/new/'` for single-token or single-line changes.
   - Use `awk` only when the change spans multiple consecutive lines or
     requires line-number arithmetic.
   - Use `patch -p1 << 'EOF' … EOF` for multi-line, multi-hunk changes where
     a unified diff is more readable than chained `sed` calls.
   - Use `grep -q … || sed -i …` (or equivalent) guards to keep every edit
     idempotent.
   - A heredoc (`cat > path/to/file << 'EOF'`) is permitted for creating a
     brand-new file when a feature requires one. Existing files must always
     be edited surgically (sed/awk/patch) — a heredoc must never be used to
     replace or rewrite an existing file wholesale.
   - **After every edit, verify and report it.** Immediately after each
     `sed`/`awk`/`patch` block, check the result (e.g. `grep -q` for the new
     value) and echo a one-line `OK: <description>` or `FAILED: <description>`
     to stderr. No edit may land silently — the output must make it obvious
     whether each change fully applied, partially applied, or failed.
3. At the **end** of the patch script (before `rm -rf ./temp`), append a
   `cat` heredoc that writes a handoff file to the project root.
   Name the file according to the type of change and the next available
   sequence number:
   - Bug fix → `handoff_fix_N.md`  (count existing `handoff_fix_*.md` files to get N)
   - Feature addition → `handoff_add_N.md`  (count existing `handoff_add_*.md` files to get N)

   Determine N inside the script itself so it is always correct regardless
   of how many prior handoffs exist:

```bash
# … all surgical edits above …

# Write handoff (fix example — swap 'fix' for 'add' and adjust title for features)
mkdir -p Artifacts
_n=$(ls Artifacts/handoff_fix_*.md 2>/dev/null | wc -l)
_n=$(( _n + 1 ))
cat > "Artifacts/handoff_fix_${_n}.md" << 'EOF'
# Fix Handoff — <title>

## Changes
| File | Change |
|------|--------|
| path/to/file | one-sentence description of what changed and why |

## Notes
- <any follow-up caveats, e.g. "docker compose up --build required">
EOF

rm -rf ./temp
```

   Keep the handoff file focused: the changes table plus only notes that
   a future session could not infer from the table alone. No prose padding.

### Hard rules
- Never skip Phase 1.
- Never output full source files in either phase.
- New files are created inside the patch script via heredoc, not as separate artifacts.
- Existing files are always edited surgically (sed/awk/patch); never replaced wholesale.
- Handoff files are always `handoff_fix_N.md` (bug fix) or `handoff_add_N.md` (feature); never any other name.
- N is always computed at runtime by counting existing files of that prefix, not hardcoded.
- Every `cp` (Phase 1) and every `sed`/`awk`/`patch` edit (Phase 2) must emit a concise
  success or failure confirmation; no operation's outcome may be left unstated.
- If the fix touches config that affects Docker, note the required `docker compose up --build`
  in a comment at the top of the patch script and repeat it in the handoff's Notes section.

<!-- END PROTOCOL -->

---

## Output instructions
Write `continue.md` as a single markdown file. Section order must be:
Overview → Stack → Ports & Volumes → File Manifest → Contracts →
Implementation Decisions → Fix/Feature Protocol.

Keep prose tight. The goal is a file a future session can fully rely on without
re-reading plan.md or any handoff.
