# Master: Multi-Session App Builder
> Bootstrap only. After Plan, attach all files from `artifacts/`. Do not re-attach this file.

## Detect Mode
| Context | Mode |
|---|---|
| This file present | Plan |
| `plan.md` + `next.md` present, this file absent | Build |

---

## Script Convention
- One Python script per session, delivered as a single fenced code block in the chat reply only — never written to disk, executed, installed, or attached as a file; the user runs it locally
- Header: `#!/usr/bin/env python3` and `import os`
- Writes: `os.makedirs(..., exist_ok=True)` + `open(path, "w").write(...)` + triple-quoted strings; paths relative to cwd exactly as listed in `files`, no wrapper folder
- Overwrites on re-run (safe to repeat)
- Binary assets (icons, images, compiled resources) aren't written by the script — note required assets/specs in README instead
- `artifacts/` holds only `plan.md`, `next.md`, concise handoff files; source files go to project paths
- All artifact files: flat `key: value` only — no prose, no decorative markdown; LLM-targeted, minimal
---

## Plan

Decide all critical choices (target platform/OS and stack included) by defaulting to the most common choice for the user described scenario. Stack locks on first write to `plan.md`.

Pre-write checklist — resolve all before writing:
- Every imported/referenced file assigned to exactly one session
- No forward dependencies; entry/composition files go to last session that imports from them
- Every dependency in stack with a pinned version; all jointly compatible
- Every file referenced by another generated file (build manifests/project files — e.g. Dockerfile, Gradle/Xcode project files, AndroidManifest.xml, Info.plist) is in the sessions table
- Any external fact (API shapes, SDK versions, model IDs) confirmed via live search — one search plus one fetch of the primary doc page per fact; batch related facts (e.g. a model lineup and its deprecations) into a single page fetch instead of separate searches
- Any package/API call whose name could be ambiguous (ID/hash/date generation, etc.) is written in contracts as a literal import+call line, not a description
- All cross-session contracts fully defined: field names, types, nullability, hazards, async intermediate states
- Design/UI direction stated as one line in contracts; skill or library deep-dives happen only in the build session that authors those files

Session design: distribute work evenly; split overloaded sessions; record `total_sessions`.

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
       part1: new contracts not in plan.md contracts — omit section entirely if none
       part2: file paths produced this session
  6  if is_last=false → overwrite artifacts/next.md:
       session: N+1
       files: sessions[N+1] from plan.md
       handoff_out: artifacts/handoff_[N+1].md
       is_last: (N+1 == total_sessions)
       read_handoffs: handoff_1.md, ..., handoff_N.md (all produced so far)
  7  if is_last=true → write README.md, .gitignore, platform build-ignore file(s); leave next.md unchanged
  delivery: single fenced python code block in the chat reply only — never written to disk, executed, installed, or attached as a file; paths relative to cwd exactly as listed, no wrapper folder; binary assets noted in README, not written; the user runs the script locally
  script_rule: in Markdown file content strings use ~~~ as fence delimiters, not triple backticks — triple backticks inside a Python triple-quoted string break the outer fenced code block display
  on_contradiction: smallest literal fix → log in handoff part1 as flagged_correction → proceed; no redesign
  ownership: write only files in next.md; never re-output prior or future sessions' files
  stack_lock: no dependency outside plan.md stack
  no_verification: trust plan.md completely — no installs, imports, version checks, server starts, or test runs of any kind; the sandbox's network/runtime doesn't reflect the deploy target
  no_narration: execute steps directly; do not restate next.md/plan.md contents in prose first
  write_means: "write" in every step above means emit a Python open(path,"w").write(...) call — never display file contents as raw text, inline code, or with # === filename === separators
```

---

## Build

Read `build_protocol` from `plan.md` and execute steps 1–7 directly — do not restate them, verify, research, or redesign; all decisions are pre-made in `plan.md`.
Output: a single Python script, delivered as one fenced code block in the chat reply — never written to disk, executed, installed, or attached as a file. The user runs it locally. Open a new chat. Attach all files from `artifacts/`.
