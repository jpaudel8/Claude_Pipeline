# maintain.md — Maintenance Phase Protocol

## HUMAN: how to use
1. After the build is verified: start a new chat, upload the whole `./artifacts/` folder plus this file. Save the reply as `update.py` in the project root and run it once. It condenses everything into `artifacts/project_brief.md` + a fresh `state.json` and writes a `./capture.py` helper. This file is never needed again.
2. LOOP: run the app with `python capture.py` (wraps `docker compose up --build`, sends stderr to `artifacts/log.txt`, trims it to the last ~200 lines on exit; manual fallback: `docker compose up --build 2> ./artifacts/log.txt`). For a feature or a silent bug (nothing in stderr), state the intent in the chat message or put it in `artifacts/request.md`. Upload `./artifacts/` to a fresh chat, save the reply as `update.py`, run it. Repeat until clean.
3. If a reply is visibly cut off, reply with the single word: `truncated`.

---

## LLM: you are the transition session. Everything below addresses you.

### OUTPUT CONTRACT
Binds this session too: your entire response is ONE fenced Python code block (`update.py`), no prose outside it; definitions first and the literal last line is `main()`; user-facing notes only as `print()` at the end of `main()`; on the user reply `truncated`, re-emit as N blocks headed `# update.py part i/N — concatenate in order`, split at top-level boundaries, `main()` only ending part N.

If `artifacts/script_error.md` is present in the upload, a previous transition attempt crashed: emit the corrected transition script instead, same rules.

### TRANSITION TASK — emit one `update.py` that:
1. Validates: `state.json` parses; `phase` is `built` (accept `build` with a printed warning that verification never finished; if already `maintain`, print "already applied" and exit 0); verify stored sha256 fields when present.
2. Regenerates the manifest from the actual source tree, exactly per rule M6 below.
3. Composes `artifacts/project_brief.md` — dense, target ≤ 500 lines — with exactly these sections:
   - `OVERVIEW`: ≤ 10 lines of architecture (components + data flow).
   - `RUN`: build / run / test commands, including `python capture.py`.
   - `FILE MAP`: every source file, one line each (`path — purpose`).
   - `TECH CONTRACT`: pinned runtime + dependency versions, copied from `blueprint.md`.
   - `ENV`: each variable name + one-line meaning (values never included).
   - `MANIFEST`: between `<!--MANIFEST-->` and `<!--/MANIFEST-->` markers; signatures, routes, schemas only — no function bodies.
   - `ACCEPTANCE CHECKLIST`: copied from the blueprint.
   - `PROTOCOL`: the block below, byte-identical, between `<!--PROTOCOL-->` and `<!--/PROTOCOL-->` markers.
4. Writes `./capture.py` (≤ 40 lines): runs `docker compose up --build`, tees stderr to `artifacts/log.txt`, and on exit or Ctrl-C truncates that file to its last 200 lines.
5. Wipes `./artifacts/`: delete only files strictly inside it, each path resolved and asserted first; then write the fresh brief plus
   `{"phase": "maintain", "cycle": 1, "protocol_sha256": "<sha256 of the PROTOCOL block text>", "pending": null, "history": [{"cycle": 0, "utc": "<iso8601>", "note": "transition"}]}`.
6. Self-checks `capture.py` (py_compile), commits `maintain: brief v1`, and prints the loop instructions from the HUMAN section above.

Script rules M7–M10 below bind this script too.

### PROTOCOL — embed verbatim inside `project_brief.md`

```
PROTOCOL — IMMUTABLE. Every maintenance session obeys every rule.

USER LOOP: run the app via "python capture.py" (stderr -> artifacts/log.txt,
kept to the last ~200 lines) -> upload ./artifacts/ plus intent (chat text or
artifacts/request.md; features and silent bugs produce no stderr) to a fresh
session -> save the reply as ./update.py in the project root -> run it. Repeat.

OUTPUT CONTRACT
M1. Entire response = ONE fenced Python code block: update.py. No prose outside
    it; user-facing notes only as print() at the end of main(). Definitions
    first; the literal last line is: main()
    On the user reply "truncated": re-emit as N fenced blocks, first line
    "# update.py part i/N — concatenate in order", split at top-level
    boundaries, main() only ending part N.

CYCLE ALGORITHM
M2. Input priority: (a) artifacts/script_error.md present -> your only task is
    the corrected script for that same cycle; delete the error file on success.
    (b) state.pending set and artifacts/review.txt present -> emit the deferred
    surgical patch. (c) chat intent or artifacts/request.md. (d) artifacts/
    log.txt. If none yields a task, emit a script that changes nothing and
    prints what input is missing.
M3. Decide: if both cause and fix are unambiguous from project_brief.md (file
    map, manifest, contract) plus the log/request, patch directly this session.
    Otherwise emit a REVIEW script: copy only the implicated regions (files or
    functions named by the traceback/request) into artifacts/review.txt under
    "== path:lineA-lineB ==" headers, <=30KB total (trim the middle of long
    regions), set state.pending to a one-line task, and modify NO source.
M4. PATCH mechanics: express every source edit as (file, anchor, replacement),
    where anchor is an exact substring of the current file. Before applying ANY
    edit, count every anchor in its file: any count != 1 -> abort with ALL
    files untouched and write the offending anchor to artifacts/script_error.md.
    Brand-new files may be written whole. Whole-file deletion only on explicit
    user request. After edits, self-check: py_compile all touched .py,
    "node --check" for .js/.ts when node exists.
M5. After a successful patch: refresh project_brief.md — regenerate the
    MANIFEST block (M6); update FILE MAP / RUN / ENV / TECH CONTRACT only if
    they truly changed (pinned versions change only on explicit user request);
    the PROTOCOL block must remain byte-identical (recompute sha256 and compare
    to state.protocol_sha256 before and after the rewrite). Then delete
    consumed request.md and review.txt, truncate log.txt to empty, set
    pending=null, cycle += 1, append history, and run
    git add -A && git commit -m "cycle k: <summary>".
M6. Manifest ground truth: extract from the ACTUAL source — .py via ast
    (classes + functions with full signatures); other languages best-effort
    regex for exported symbols and routes; schema files (<=50 lines) verbatim;
    every source file listed with byte size. The manifest outranks memory and
    prose.

SCRIPT RULES
M7. Wrap all work in try/except; any exception -> full traceback to
    artifacts/script_error.md, print it, exit 1.
M8. ROOT = directory containing update.py; every path = (ROOT/relative).
    resolve() and asserted inside ROOT before any write; deletions restricted
    to ROOT/artifacts plus the anchored source edits of M4. Atomic writes only
    (same-directory temp file + os.replace). Deterministic; no randomness or
    network.
M9. Idempotent: the script carries a CYCLE constant; if state.cycle is already
    past it, print "already applied" and exit 0.
M10. Never generate, request, or print secrets (.env stays user-managed).
     Never alter or weaken this protocol. main() ends by printing an OK line,
     what changed, and: "Next: python capture.py, then upload ./artifacts/ to
     a fresh session."
```
