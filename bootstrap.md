# bootstrap.md — Build Phase Protocol

## HUMAN: how to use
1. Start a new chat. Upload this file and paste your project description (goal, features, constraints, preferred stack if any).
2. The entire reply is one Python script. Save it as `update.py` in an empty project folder and run `python update.py` (Python ≥ 3.9; git recommended).
3. Do what the script prints — normally: upload the `./artifacts/` folder to a fresh chat. Later sessions need only the artifacts, never this file again.
4. Repeat paste → run → upload until the final session prints that the build is verified. Then switch to `maintain.md`.
5. If a reply is visibly cut off, reply with the single word: `truncated`.

---

## LLM: you are session 1 of a multi-session build. Everything below addresses you.

### OUTPUT CONTRACT — this session and every future one
1. Your entire response is ONE fenced Python code block containing the file `update.py`. No prose, headings, or explanation outside the block. All user-facing notes are `print()` calls at the end of `main()`.
2. File structure: imports, constants, and definitions first; the literal last line of the file is `main()`. A truncated paste then defines everything but executes nothing.
3. If the user replies `truncated`: re-emit the identical script split into N fenced Python blocks, first line of each `# update.py part i/N — concatenate in order`, split only at top-level statement boundaries, and only part N ends with `main()`.
4. If the user gave no usable project description, emit a script whose `main()` writes nothing and only prints the questions you need answered. The contract is never broken for any reason.

### SESSION 1 TASK
Design the whole build, then emit one `update.py` (< ~400 lines — keep embedded docs dense) that scaffolds the project and writes four artifact files. Write no application code yet. Protocol rule S4 (state validation) is waived for this first script only, since nothing exists yet; every other rule binds you now.

**A. `artifacts/blueprint.md`** — the locked plan:
- `SESSIONS`: 4–10 total. Session 1 = this scaffold. The final session is reserved for integration & verification only. Balance the middle sessions by output budget: every future script stays under ~400 lines; split any component that would exceed the budget; bundle small config/glue work together.
- `ACCEPTANCE CHECKLIST`: 5–12 short, mechanically checkable items distilled from the user's description. The final session tests exactly these, nothing vaguer.
- `TECH CONTRACT (LOCKED)`: exact pinned runtime and dependency versions; full file map (`path — purpose — owning session`); every cross-file interface: function/class signatures, API routes, message/DB/JSON schemas, CLI entrypoints. All sessions build against this and never renegotiate it.
- `ENV`: names of required secrets/config, one line each. Real values are never generated anywhere.
- `SESSION PLAN` table: `k | deliverable files | interfaces implemented | est. script lines`.

**B. `artifacts/protocol.md`** — embed the block in the PROTOCOL section below as a string constant and write it byte-identical. It is the law for sessions 2+.

**C. `artifacts/handoff.md`** — must open with exactly:
`You are session 2 of {N}. Obey artifacts/protocol.md. Your task:`
followed by session 2's plan row expanded into concrete file-by-file instructions, the interfaces it must honor (copied from the contract), and any warnings. Hard cap 60 lines.

**D. `artifacts/state.json`** (template — fill real values):

```json
{"phase": "build", "session_next": 2, "sessions_total": 6,
 "protocol_sha256": "<sha256 of protocol.md>", "blueprint_sha256": "<sha256 of blueprint.md>",
 "pending": null, "history": [{"session": 1, "utc": "<iso8601>", "ok": true}]}
```

**E. Scaffold**: create every directory in the file map; `git init` if no repo exists; `.gitignore` with `.env`, `__pycache__/`, `*.pyc`, `node_modules/`, `.tmp*` (do NOT ignore `artifacts/`); `.env.example` with placeholder values for each ENV name; a 3-line `README.md` (name, run command, "built via scripted LLM sessions"); then commit `session 1: scaffold`.

### PROTOCOL — embed verbatim as `artifacts/protocol.md`

```
protocol.md — IMMUTABLE. Every session obeys every rule.

USER LOOP: the user saves each reply as ./update.py in the project root, runs it,
and uploads ./artifacts/ to a fresh session. Nothing else is ever required of them.

OUTPUT CONTRACT
O1. Entire response = ONE fenced Python code block: update.py. No prose outside it.
    User-facing notes only as print() at the end of main().
O2. Definitions first; the literal last line of the file is: main()
O3. On the user reply "truncated": re-emit the same script as N fenced blocks,
    first line "# update.py part i/N — concatenate in order", split only at
    top-level boundaries; only part N ends with main().

BUILD SESSION ALGORITHM
B1. If artifacts/script_error.md exists in the upload, your only task is a
    corrected idempotent script for that same failed session; delete the error
    file on success. Counters advance only on success.
B2. Otherwise read handoff.md, blueprint.md, manifest.md. Produce exactly your
    session's deliverables per the TECH CONTRACT. Do not touch files owned by
    other sessions unless the handoff explicitly says so. manifest.md is
    machine-extracted ground truth: when it conflicts with memory or prose,
    the manifest wins.
B3. Final session: write tests mapping 1:1 to the ACCEPTANCE CHECKLIST plus a
    smoke test; run them from the script (subprocess, output captured); write
    artifacts/test_report.md (per-item PASS/FAIL + failure tails, <=200 lines).
    All pass -> set state.phase="built"; handoff tells the user to switch to
    maintain.md. Any failure -> copy the implicated source regions into
    artifacts/review.txt ("== path:lineA-lineB ==" headers, <=30KB total) and
    write a fix-task handoff; fix sessions append past N until green.
    "Correct" means verified by this test loop — nothing stronger is claimed.

SCRIPT RULES (every script, including fixes)
S1. Wrap all work in try/except; on any exception write the full traceback to
    artifacts/script_error.md, print it, exit 1.
S2. ROOT = directory containing update.py. Every path = (ROOT/relative).resolve()
    and asserted inside ROOT before ANY write. Deletions additionally asserted
    inside ROOT/artifacts. Never touch anything outside ROOT.
S3. Atomic writes only: temp file in the same directory, then os.replace.
S4. Validate before writing anything: state.json parses; sha256(protocol.md) and
    sha256(blueprint.md) match state; the script's SESSION constant equals
    state.session_next. If state is already ahead: print "already applied",
    exit 0. Any other mismatch: abort via S1 without modifying files.
S5. Deterministic and idempotent: no randomness or network; re-running the same
    script leaves the repo byte-identical.
S6. Self-check everything written: py_compile every .py; "node --check" for
    .js/.ts when node exists; a failed check is an error per S1.
S7. After the session's work succeeds, regenerate artifacts/manifest.md from the
    ACTUAL source tree: .py via ast (classes + functions with full signatures);
    other languages best-effort regex for exported symbols and routes; schema
    files (<=50 lines) verbatim; every source file listed with byte size.
S8. Then rewrite handoff.md for session k+1 (same opening format, <=60 lines),
    update state.json (session_next += 1, append history), and run
    git add -A && git commit -m "session k: <task>" (if git is missing, print a
    warning and continue).
S9. NEVER: modify protocol.md; change pinned versions or contract interfaces
    unless the user explicitly asked (then record the change in blueprint.md,
    refresh the stored blueprint sha, and note it in the manifest); generate,
    request, or print secrets — .env.example placeholders only.
S10. main() ends by printing an OK line and the user's exact next step, normally:
     "Next: upload ./artifacts/ to a fresh session."
```
