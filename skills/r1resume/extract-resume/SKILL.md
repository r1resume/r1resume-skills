---
name: r1resume-extract-resume
description: Use when the user wants to import or extract a resume in the R1Resume builder — e.g. "import my resume", "extract this resume file", "parse my PDF resume into the builder". The agent acts as the LLM that the Go API's `/ai/extract-resume` endpoint would otherwise call: it reads raw text via WebMCP tools and writes structured Resume JSON back into the app.
---

# R1Resume — Step 0: Extract Resume

## What this skill does

In the regular app flow, step 0 calls `POST /ai/extract-resume` (see `apps/api/internal/ai/controller.go` and `apps/api/internal/ai/gemini_client.go`), which feeds the raw text to Gemini with a system prompt and expects a `Resume` JSON back, then calls `resume.CorrectOrder()`.

**The agent replaces that endpoint.** You are the LLM. You read the raw text via WebMCP, transform it into the `Resume` shape yourself, and write it back via `set_original_resume`.

## Prerequisites

See `r1resume-navigate`. Confirm the RNA tab is attached by calling `chrome-devtools_list_webmcp_tools` and seeing `extract_raw_text`, `set_original_resume`, `get_original_resume`, `set_current_step`. The user must be on (or be sent to) step 0 — confirm with `get_current_step`; if not 0, do not jump steps unless the user explicitly asks — just perform the extraction and set state.

## Workflow

1. **Ensure a file is selected.**
   - `pick_file` — opens the browser's native picker (PDF/DOCX). The user must choose a file; you cannot pick for them.
   - `get_selected_file` — confirm `hasFile: true` and read `fileName`. If `hasFile: false`, ask the user to select a file and call `pick_file` again.

2. **Extract raw text.**
   - `extract_raw_text` — no input. Returns the raw text content of the selected file.
   - If it returns `No file selected` or `File type not supported for web extraction`, surface the message to the user and stop.

3. **Transform raw text → `Resume` JSON.**
   Apply this system prompt verbatim to yourself, using the raw text as the input:

   > Populate resume data based on the given prompt. IMPORTANT: Never omit any content — if it doesn't fit a standard section, place it in a custom section.

   Output a single `Resume` object matching `$defs.Resume` in `apps/rna/lib/api/schemas/resume.schema.json`. Rules:
   - Capture **everything** in the raw text — never omit content. If a piece of information doesn't fit a standard section (summary, experience, education, skills, projects, certifications, languages, awards, publications, custom sections, etc.), put it in a custom section so nothing is lost.
   - Infer sensible field placement (e.g. dates into `startDate`/`endDate`, bullets into `entries[].description`, contact info into `email`/`phone`/`website`/`location`/`socialNetworks`).
   - If the raw text genuinely lacks a field, leave it out (do not invent content). The "never omit" rule applies to information *present* in the source.
   - Apply the same final step the Go server does (`resume.CorrectOrder()`): order standard sections in the conventional order and preserve any explicit `order` array the schema allows.

4. **Validate before writing.** Self-validate the JSON against the `Resume` schema — form-error feedback is not yet exposed via WebMCP (see `r1resume-navigate`), so a malformed `Resume` may appear to succeed. Fix any schema violations before calling the next step.

5. **Write the resume.**
   - `set_original_resume({ ...the Resume object })` — input is the full `Resume` object (not a wrapped key).

6. **Verify and advance.**
   - `get_original_resume` — read back and confirm it matches what you wrote (compare the JSON, not just the success text).
   - If the read-back matches, call `set_current_step({ step: 1 })` to move to the Build step, mirroring the app's `onSuccess` (which calls `setCurrentStep(1)`).
   - If the read-back does **not** match, retry `set_original_resume` once. On a second mismatch, report it to the user and stop — do not advance.

## Guards

- Never POST to `/ai/extract-resume`. You are replacing it.
- Never type into form fields; always use `set_original_resume`.
- Never fabricate content that is not in the source text.
- Never advance to step 1 until `get_original_resume` confirms the data landed.
- If `pick_file` returns `No file selected` or `File picker cancelled`, do not attempt extraction — ask the user to pick a file.

## Reference

- System prompt source: `apps/api/internal/ai/gemini_client.go:17` (`extractResumeSystemPrompt`).
- Resume schema: `apps/rna/lib/api/schemas/resume.schema.json` (`$defs.Resume`).
- WebMCP tools: `apps/rna/lib/resume-importer/mcp/register-pick-file.ts`, `register-get-picked-file.ts`, `register-extract-raw-text.ts`; `apps/rna/lib/store/mcp/register-set-original-resume.ts`, `register-get-original-resume.ts`, `register-set-current-step.ts`.