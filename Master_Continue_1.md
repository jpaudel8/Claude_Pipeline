# Master_Continue.md
> Consolidation master. Attach in a new chat together with every file from `artifacts/`. Re-attach later only to re-consolidate after fix/add handoffs accumulate.

## Role
Inputs: either `plan.md` + `handoff_*.md` (first consolidation) or a prior `continue.md` + `handoff_fix_*.md`/`handoff_add_*.md` (re-consolidation; `continue.md` supersedes `plan.md`). If required inputs are missing, ask for them and do nothing else. If a `next.md` lacking `done: true` is attached, the build is unfinished — say so and stop.
Output: one Python script (conventions below) that writes `artifacts/continue.md` and compacts `artifacts/`.

## Script conventions (shared by this file and the protocol below)
- Deliver a single fenced python code block in the chat reply only — never written to disk, executed, installed, or attached by you; the user pastes it into update.py and runs it locally
- Imports: `os`, `glob` only; no subprocess, no network
- All writes: `open(path, "w", encoding="utf-8", newline="\n")`
- Escaping: escape any embedded triple-quote sequence or switch the outer delimiter (the protocol block below contains double-triple-quotes — delimit the continue.md content string with triple single quotes); use `~~~` (never triple backticks) for fences inside markdown content strings; raw strings or doubled backslashes for backslash-heavy content; emitted bytes must round-trip exactly
- Final statement: a summary `print` — a missing final line signals a truncated script

## Consolidation script
1. Write `artifacts/continue.md` (content per Parts A + B) — write first, so a truncated script never deletes state
2. Then delete, each guarded by `os.path.exists`: `artifacts/plan.md`, `artifacts/next.md`, every `artifacts/handoff*.md` (via glob), `artifacts/log.txt`, `artifacts/review.txt`
3. `print("OK: continue.md written; artifacts compacted")`

## Part A — Project snapshot
Distil from the supplied plan/continue plus every handoff; deduplicate; no session numbers, no handoff attribution. Flat `key: value` style, LLM-targeted, minimal. continue.md must stand alone — every handoff is deleted after this.
1. overview: one paragraph — what the app does, how it runs, entry points
2. stack: one line per dependency, exact pinned versions
3. runtime_config: ports, volumes, env vars, permissions/entitlements — whichever the platform uses; values verbatim
4. file_manifest: every project file, one-line purpose each
5. contracts: schemas, routes, event/IPC formats, shared types — plan contracts plus all handoff part1 additions, merged verbatim
6. decisions: flat list of behaviour-affecting or non-obvious choices, including flagged_corrections

## Part B — embed the block below in continue.md verbatim (do not summarise or reword)

<!-- BEGIN PROTOCOL — copy into continue.md unchanged -->

## Fix / Feature Protocol

Context per request: this file + `artifacts/log.txt` (if present) + the user's error or feature description. Two phases, strict order.

Script conventions (both phases): reply with a single fenced python code block — never write, execute, or attach it yourself; the user runs it locally as update.py. Imports `os`, `glob` only; no subprocess, no network. All writes use `open(path, "w", encoding="utf-8", newline="\n")`. Escape embedded triple-quote sequences or switch delimiter; use `~~~` (never triple backticks) for fences inside markdown content strings; raw strings or doubled backslashes for backslash-heavy content. End with a summary print — a missing final line signals truncation.

### Phase 1 — Triage
1. From the file manifest, log.txt, and the description, choose the minimal file set needed to diagnose or implement. Precision over breadth.
2. Reply with only this script, FILES filled in — no prose before or after:

~~~python
#!/usr/bin/env python3
import os
FILES = ["<path>", "<path>"]
os.makedirs("artifacts", exist_ok=True)
out = open("artifacts/review.txt", "w", encoding="utf-8", newline="\n")
n = 0
for p in FILES:
    if os.path.exists(p):
        out.write(f"##### {p}\n" + open(p, encoding="utf-8").read() + "\n")
        n += 1; print(f"OK: {p}")
    else:
        print(f"MISSING: {p}")
out.close()
print(f"review.txt: {n}/{len(FILES)} files")
~~~

3. Stop. Do not analyse anything yet. Wait for the user to upload `artifacts/review.txt`.
Skip Phase 1 only when the change creates new files exclusively and edits nothing existing.

### Phase 2 — Resolution
After review.txt arrives: one line stating the root cause or change points, then the patch script, nothing after. Rules:
- Open with this helper verbatim; every change to an existing file goes through it:

~~~python
#!/usr/bin/env python3
import os, glob
ERR = []
def edit(path, old, new, desc, count=1):
    try: s = open(path, encoding="utf-8").read()
    except OSError: print(f"FAILED missing {path}: {desc}"); ERR.append(desc); return
    if new in s: print(f"SKIP applied: {desc}"); return
    c = s.count(old)
    if c != count: print(f"FAILED {c}x vs {count}: {desc}"); ERR.append(desc); return
    open(path, "w", encoding="utf-8", newline="\n").write(s.replace(old, new, count))
    print(f"OK: {desc}")
~~~

- Surgical only: `old` is the exact minimal snippet (a token, a line, a few lines) copied verbatim from review.txt, padded with just enough surrounding context to be unique. Never rewrite a whole function or block when one line differs. The helper is idempotent and self-reporting — no edit lands silently.
- Brand-new files (features): `os.makedirs(dir, exist_ok=True)` + one full `open(path, "w", encoding="utf-8", newline="\n").write(...)`. Never use a full write on an existing file.
- End of script, in order:

~~~python
n = len(glob.glob("artifacts/handoff_fix_*.md")) + 1   # feature: count handoff_add_* instead
open(f"artifacts/handoff_fix_{n}.md", "w", encoding="utf-8", newline="\n").write("""# Fix — <title>
| File | Change |
|---|---|
| <path> | <what changed and why, one sentence> |

notes: <only what the table cannot convey; omit the line if nothing>
""")
for f in ("artifacts/review.txt", "artifacts/log.txt"):
    if os.path.exists(f): os.remove(f)
print("DONE" if not ERR else f"ERRORS: {len(ERR)}")
~~~

(log.txt is stale the moment a patch lands — deleting it forces the next run to produce fresh evidence.)

### Hard rules
- Never skip Phase 1 except the new-files-only case; never analyse before review.txt arrives.
- Never output the full source of an existing file in any phase.
- Existing files: edit() only. New files: one full write. Handoffs: `artifacts/handoff_fix_N.md` (bug fix) or `artifacts/handoff_add_N.md` (feature), N computed at runtime by counting; never any other name.
- Every operation prints OK / SKIP / FAILED / MISSING; no outcome may be left unstated.
- If the change touches container/build config, put the rebuild command (e.g. `docker compose up --build`) in a comment at the top of the patch script and repeat it in the handoff notes.
- When several fix/add handoffs have accumulated, tell the user to re-consolidate: new chat, every `artifacts/` file + `Master_Continue.md`.

<!-- END PROTOCOL -->

## Output instructions
continue.md section order: Overview → Stack → Runtime Config → File Manifest → Contracts → Decisions → Fix/Feature Protocol. Keep prose tight — a future session must rely on continue.md alone, without plan.md or any handoff.
