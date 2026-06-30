---
name: mtd
description: Transcribe a meeting/voice recording, save a prettified transcript as an Odysseus document, then store a very short recap in memory that references that document.
version: 1.0.0
category: productivity
tags: [whisper, transcription, meeting, recap, document, memory, russian, stt, pipeline]
platforms: [linux, macos]
requires_toolsets: [shell]
status: published
confidence: 0.85
source: taught
created: 2026-06-30T00:00:00Z
---

## When to Use

Use this when the user gives a meeting / call / voice recording (uploaded in
chat or a file path) and wants the full workflow, not just raw text:

- "transcribe this meeting and save it"
- "make a document from this recording and remember the gist"
- "расшифруй встречу, сохрани в документ и запомни кратко"
- Any request to turn a recording into a saved transcript **+** a short,
  searchable recap.

This skill chains three Odysseus capabilities end to end:
1. **Transcribe** the audio (Whisper, Russian-capable).
2. **Create a document** with a prettified transcript.
3. **Write a very short memory** recap that references that document.

For plain "just give me the text" requests, use the `transcribe-russian-audio`
skill instead.

> Depends on the `transcribe-russian-audio` skill for the transcription step
> (Whisper at `http://whisper:8000`, model `Systran/faster-whisper-large-v3`).
> Load that skill too if details are needed.

## Procedure

Do the steps strictly in order. Do not skip the document step before writing
memory — the memory recap must reference the created document.

### Step 0 — Determine the recording timestamp and the name

1. **Recording datetime.** Use the recording's own time if known (file mtime,
   upload time, or what the user states); otherwise use the current time.
   Format it as `YYYY-MM-DD HH-MM-SS`.
   - **Colons are forbidden** in the name, so the time uses **hyphens**
     (`HH-MM-SS`), never `HH:MM:SS`.
   - Get it via shell if needed:
     ```sh
     date '+%Y-%m-%d %H-%M-%S'
     ```
2. **Base name.** Ask yourself, in this priority order:
   - If the **user provided a name** → use it as the base.
   - Else, try to **derive a name from the transcription** (e.g. the meeting
     topic / first clear subject, kept short, ≤ 60 chars). Only use this if it
     is reasonably clear.
   - Else → there is **no base name**.
3. **Final name** (this is the document title AND the memory reference):
   - If there is a base name:  `"<base> <YYYY-MM-DD HH-MM-SS>"`
   - If there is no base name:  `"<YYYY-MM-DD HH-MM-SS>"`  (the datetime alone)
   - Sanitize: strip/replace any `:` `/` `\` and trim whitespace. The datetime
     suffix is **always** appended, even when the user supplied a name.

   Examples:
   - User says name "Sprint Planning", recorded 2026-06-30 14:05:09 →
     `Sprint Planning 2026-06-30 14-05-09`
   - No name, recorded 2026-06-30 14:05:09 →
     `2026-06-30 14-05-09`

### Step 1 — Transcribe

Transcribe the recording to text following the `transcribe-russian-audio`
skill. For a chat-uploaded file with the shell tool ON:

```sh
curl -sS http://whisper:8000/v1/audio/transcriptions \
  -F "file=@/app/data/uploads/<YYYY>/<MM>/<DD>/<uuid>.<ext>" \
  -F "model=Systran/faster-whisper-large-v3" \
  -F "language=ru" \
  -F "vad_filter=true" \
  -F "response_format=verbose_json"
```

Keep the raw transcript text. Prefer `verbose_json` so you have segment
timestamps for the prettified version; `response_format=text` is fine if you
only need plain text.

### Step 2 — Create the document (prettified transcript)

Use the **`create_document`** tool (editor panel). Do NOT use shell/`write_file`
for this — the user wants a real Odysseus document.

- **Title:** the Final name from Step 0.
- **Content:** a prettified Markdown transcript, not the raw blob:
  - `# <Final name>` heading.
  - A short metadata block: recording date/time, source filename (from
    `uploads.json` if known), language, duration if available.
  - The transcript with light cleanup: paragraph breaks, fixed obvious spacing,
    speaker turns if distinguishable, and `[mm:ss]` markers from the segments
    when using `verbose_json`. Do not invent content or change meaning —
    cleanup only.

Example skeleton:

```markdown
# Sprint Planning 2026-06-30 14-05-09

- **Recorded:** 2026-06-30 14-05-09
- **Source:** test.mp3
- **Language:** ru
- **Duration:** 00:52

---

[00:00] ...первый абзац транскрипта...

[00:14] ...следующий абзац...
```

Capture the returned **document id / title** — Step 3 references it.

### Step 3 — Write a VERY short memory recap

Use the **`manage_memory`** tool with `action="add"` (it is always available to
the agent; do NOT call `app_api`/`/api/memory/add` directly).

- **Keep it tiny** — 1–2 sentences max (≈ the gist + the single most important
  outcome/decision). This is a recap, not a summary.
- **Reference the document** by its Final name (and id if available) so the
  memory is traceable back to the transcript.

Memory content template:

```
Meeting "<Final name>": <one-sentence gist>. <one key decision/outcome>.
See document "<Final name>".
```

Example:

```
Meeting "Sprint Planning 2026-06-30 14-05-09": agreed scope for the next
sprint; backend API work prioritized over UI polish. See document
"Sprint Planning 2026-06-30 14-05-09".
```

### Step 4 — Confirm to the user

Briefly report: the document title created, and that a short recap was saved to
memory referencing it. Offer the transcript inline if they want it.

## Pitfalls

- **Order matters.** Transcript → document → memory. The memory recap must name
  the document, so create the document first and reuse its exact title.
- **Datetime suffix is mandatory and colon-free.** Always append
  `YYYY-MM-DD HH-MM-SS` (hyphens, not `:`), even when the user gives a name.
  With no name, the name IS that datetime string.
- **Use the right tools, not shell, for doc/memory.** `create_document` for the
  document and `manage_memory` (action=add) for memory. `write_file`/`bash`
  write to disk, not the Odysseus editor; `app_api` to `/api/memory/add` 422s.
  Shell is only for the transcription step.
- **Memory must stay short.** Do not dump the transcript or a long summary into
  memory — that pollutes retrieval. One or two sentences plus the document
  reference.
- **Prettify, don't rewrite.** Clean spacing/paragraphs/timestamps only; never
  alter the speaker's meaning or fabricate content.
- **Russian accuracy.** Pass `language=ru` (and `vad_filter=true`) to Whisper;
  see the `transcribe-russian-audio` skill for the 500 / batched-mode caveat.
- **Name from transcription is best-effort.** Only derive a topic-name if it is
  clearly stated early; otherwise fall back to the datetime-only name rather
  than guessing.

## Verification

- A new document exists (via `manage_documents` action=list) whose title equals
  the Final name and whose body is the prettified transcript.
- A memory exists (via `manage_memory` action=search on the Final name or
  "Meeting") that is 1–2 sentences and references the document title.
- The name contains no `:` and ends with a `YYYY-MM-DD HH-MM-SS` suffix; if the
  user gave a name, it is `"<name> <datetime>"`, otherwise just `"<datetime>"`.
