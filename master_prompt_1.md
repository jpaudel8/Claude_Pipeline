# Master: Multi-Session App Builder
> Bootstrap only. After Plan, attach all files from `artifacts/`. Do not re-attach this file.
> Pipeline: Plan → Build ×K → Consolidate (`Master_Continue.md` → `continue.md`) → run via README command (logs to `artifacts/log.txt`) → Fix/Add loop per `continue.md`.

## Detect Mode
First matching row wins.

| Context | Mode |
|---|---|
| This file present, `plan.md` absent | Plan |
| User says `rebuild session N` and `plan.md` present | Repair |
| `next.md` contains `done: true` | Consolidation — follow the instruction inside next.md |
| `plan.md` + `next.md` present, this file absent | Build |
| Any other combination | Halt; state what is attached and ask |

---

## Script Convention
- One Python script per session, delivered as a single fenced code block in the chat reply only — never written to disk, executed, installed, or attached as a file; the user runs it locally
- Script body: only `os.makedirs`, `open().write()`, and one final `print` — no other imports, no subprocess, no network; auditable at a glance
- Header: `#!/usr/bin/env python3` and `import os`
- Writes: `os.makedirs(..., exist_ok=True)` + `open(path, "w", encoding="utf-8", newline="\n").write(...)` + triple-quoted strings; paths relative to cwd exactly as listed in `files`, no wrapper folder
- Escaping: escape any embedded triple-quote sequence or switch that string to the other triple-quote delimiter; backslash-heavy content (regexes, Windows paths) → raw strings or doubled backslashes; emitted bytes must round-trip exactly
- Final statement: `print("OK: N files")` with N hardcoded to the write-call count — a missing final line signals a truncated script
- Overwrites on re-run (safe to repeat)
- Binary assets (icons, images, compiled resources) aren't written by the script — note required assets/specs in README instead
- `artifacts/` holds only `plan.md`, `next.md`, concise handoff files; source files go to project paths
- All artifact files: flat `key: value` only — no prose, no decorative markdown; LLM-targeted, minimal

---

## Plan

Decide all critical choices (target platform/OS and stack included) by defaulting to the most common choice for the user described scenario. Stack locks on first write to `plan.md`.

Pre-write checklist — resolve all before writing:
- Every imported/referenced file assigned to exactly one session
- No forward dependencies: every file's session ≥ every session containing a file it imports; entry/composition files land in the final sessions
- Every dependency in stack with a pinned version; all jointly compatible
- Every file referenced by another generated file (build manifests/project files — e.g. Dockerfile, Gradle/Xcode project files, AndroidManifest.xml, Info.plist) is in the sessions table
- Any external fact (API shapes, SDK versions, model IDs) confirmed via live search — one search plus one fetch of the primary doc page per fact (one extra fetch permitted if the primary page lacks it); batch related facts (e.g. a model lineup and its deprecations) into a single page fetch instead of separate searches
- Any package/API call whose name could be ambiguous (ID/hash/date generation, etc.) is written in contracts as a literal import+call line, not a description
- All cross-session contracts fully defined: field names, types, nullability, hazards, async intermediate states
- Design/UI direction stated as one line in contracts; skill or library deep-dives happen only in the build session that authors those files

Session design: distribute work evenly; cap ≈1,200 lines of emitted file content per session; split overloaded sessions; record `total_sessions`.

Output: a single Python script, delivered as one fenced code block in the chat reply — never written to disk or run. The user runs it locally; running it writes `artifacts/plan.md` and `artifacts/next.md` (session 1 row; `read_handoffs:` empty). Open a new chat. Attach all files from `artifacts/`.

---

## plan.md Template

*Written once by Plan; never modified. Flat key:value — no prose, no decorative markdown.*

```
app: [name]
platform: [target OS/runtime, e.g. web, android, ios, macos, windows, linux]
total_sessions: [K]

overview: [purpose, features, UX notes — single compact paragraph]

contracts: |
  [All cross-session interfaces: data schemas/types, shared exchange formats, inter-component
   or inter-process communication (env vars, ports, IPC, intents, delegates, notifications,
   deep links — whichever apply to the target platform), permissions/entitlements, build/signing
   configuration, export signatures. Exact field names, types, nullability. Known hazards.
   Valid intermediate states during async transitions.
   Before writing any entry, ask: does this exact content need to appear identically in files
   owned by two or more sessions? If no, skip it — it belongs in that session's files.
   Only cross-session shared types/interfaces qualify for verbatim blocks.
   Everything else is flat key:value or a single hazard line.
   One fact, one place: once a value is defined, later entries reference its key by name.]

stack: |
  [one line per dependency; exact pinned version; e.g. react@18.2.0, kotlin@2.0.0,
   swift-tools-version@5.10 — whatever the target platform's package/build system uses]

sessions:
  1: [file list]
  2: [file list]
  K: [file list]

next_schema:
  session: [N]
  files: [file list from sessions[N]]
  handoff_out: artifacts/handoff_[N].md
  is_last: [true|false]
  read_handoffs: [comma-separated prior concise handoff paths; empty for session 1]

build_protocol steps:
  1  read next.md → {session=N, files, handoff_out, is_last, read_handoffs}
  2  read plan.md → {stack, contracts, total_sessions}
  3  read each path in read_handoffs in order (skip if empty)
  4  emit open(path,"w").write(...) calls for every file in `files` — full content, no stubs; never output file contents as raw text or with separator comments
  5  write concise handoff_out (flat key:value):
       part1: new contracts not already in plan.md contracts or any prior handoff — omit section entirely if none
       part2: file paths produced this session
  6  if is_last=false → overwrite artifacts/next.md:
       session: N+1
       files: sessions[N+1] from plan.md
       handoff_out: artifacts/handoff_[N+1].md
       is_last: (N+1 == total_sessions)
       read_handoffs: handoff_1.md, ..., handoff_N.md (all produced so far)
  7  if is_last=true → write README.md (exact local commands: install deps, build/typecheck, run —
     the user performs all verification; the run command must capture all output to
     ./artifacts/log.txt, e.g. docker compose up --build 2>&1 | tee ./artifacts/log.txt, or the
     platform equivalent; plus binary-asset specs), .gitignore (must ignore artifacts/ and build
     outputs), platform build-ignore file(s); overwrite artifacts/next.md with exactly two lines:
       done: true
       next: consolidation — in a new chat attach Master_Continue.md together with every artifacts/ file; if Master_Continue.md is absent, ask the user for it and do nothing else
  delivery: single fenced python code block in the chat reply only — never written to disk,
     executed, installed, or attached as a file; script body only os.makedirs +
     open(path,"w",encoding="utf-8",newline="\n").write(...) + one final print — no other
     imports, no subprocess, no network; paths relative to cwd exactly as listed, no wrapper
     folder; binary assets noted in README, not written; final statement print("OK: N files")
     with N hardcoded to the write-call count (missing line = truncated script); the user runs
     the script locally
  script_rule: in Markdown file content strings use ~~~ as fence delimiters, not triple
     backticks — triple backticks inside a Python triple-quoted string break the outer fenced
     code block display; escape any embedded triple-quote sequence or switch that string to the
     other triple-quote delimiter; use raw strings or doubled backslashes for backslash-heavy
     content — emitted bytes must round-trip exactly
  on_contradiction: smallest literal fix → log in handoff part1 as flagged_correction → proceed; no redesign
  ownership: write only files in next.md; never re-output prior or future sessions' files
  stack_lock: no dependency outside plan.md stack
  no_verification: trust plan.md completely — no installs, imports, version checks, server starts, or test runs of any kind; the sandbox's network/runtime doesn't reflect the deploy target — README's commands relocate all verification to the user
  no_narration: execute steps directly; do not restate next.md/plan.md contents in prose first
  write_means: "write" in every step above means emit a Python open(path,"w").write(...) call — never display file contents as raw text, inline code, or with # === filename === separators
```

---

## Build

Read `build_protocol` from `plan.md` and execute steps 1–7 directly — do not restate them, verify, research, or redesign; all decisions are pre-made in `plan.md`.
Output: a single Python script, delivered as one fenced code block in the chat reply — never written to disk, executed, installed, or attached as a file. The user runs it locally. Open a new chat. Attach all files from `artifacts/`.

---

## Repair

User attaches `plan.md` plus all existing handoffs and says `rebuild session N`. Reconstruct the next.md row for N from `plan.md` (files: sessions[N]; handoff_out: artifacts/handoff_[N].md; is_last: N == total_sessions; read_handoffs: handoff_1 … handoff_[N-1]), then execute Build steps 2–7 with that row. Available until consolidation (requires `plan.md`).
