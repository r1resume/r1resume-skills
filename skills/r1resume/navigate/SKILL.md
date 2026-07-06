---
name: r1resume-navigate
description: Use when the user wants to navigate the R1Resume builder workflow, move between steps, or fill resume/job-description/cover-letter forms in the running RNA web app. Triggers on phrases like "navigate resume builder", "go to step", "fill resume form", "set job description", "move to match step". Also the umbrella skill that documents the full WebMCP tool inventory and delegates the AI-replacement steps (extract / match / cover-letter) to their dedicated skills.
---

# R1Resume Builder Navigation & Form-Filling

## Context

The R1Resume RNA web app (`apps/rna`) exposes a set of **WebMCP tools** on `document.modelContext` while the resume builder is open. opencode reaches them through the `chrome-devtools` MCP server, which is already enabled in `.opencode/opencode.jsonc` with `--categoryExperimentalWebmcp=true`.

The agent (you) is the LLM that powers the app's AI features. Where the regular app flow would `POST` to the Go API at `/ai/extract-resume`, `/ai/match-resume`, `/ai/cover-letter` (see `apps/api/internal/ai/gemini_client.go`), **you instead perform the work directly** by reading state via `get_*` tools and writing state via `set_*` tools. You never call the `/ai/*` HTTP endpoints.

## Prerequisites

- The RNA web dev server is already running and the resume-builder page is open in a Chrome tab that opencode is attached to.
- Before doing anything else, call `chrome-devtools_list_webmcp_tools` to confirm the RNA tab is attached and the tools below are live. If the list is empty or missing the `*_resume` / `*_cover_letter` tools, the resume builder is not mounted — tell the user to open the builder and try again.

All WebMCP tools are invoked via `chrome-devtools_execute_webmcp_tool` with `toolName` and a JSON-stringified `input`.

## The 6-step workflow

| Index | Step        | What lives here                                                                 |
| ----- | ----------- | ------------------------------------------------------------------------------- |
| 0     | Import      | Pick a file and extract it into structured resume data                         |
| 1     | Build       | Edit the parsed resume (form-driven, user task)                                |
| 2     | Match Job   | Paste a job description and tailor the resume to it                             |
| 3     | Review      | Review the diff between original and matched resume (user task)                |
| 4     | Cover Letter| Generate a tailored cover letter                                               |
| 5     | Export      | Download the resume / cover letter as PDF                                       |

Read the current position with `get_current_step`; move with `set_current_step({ step: <0-5> })`. Do not skip forward blindly — confirm a step's required state exists before advancing (see per-step rules below).

## WebMCP tool inventory

All tools are registered in `apps/rna/lib/store/mcp/` and `apps/rna/lib/resume-importer/mcp/`. The input/output shapes described here are authoritative for skill use.

### Navigation

- `get_current_step` — no input; returns the current step index as text.
- `set_current_step` — `{ step: number (0-5) }`.

### File (step 0)

- `pick_file` — no input; opens the browser file picker (PDF/DOCX). Returns `Selected: <name>` or `No file selected`.
- `get_selected_file` — no input; returns `{ hasFile, fileName }`.
- `extract_raw_text` — no input; returns the raw text content of the selected file (PDF/DOCX).

### Original resume (steps 0 → 1)

- `get_original_resume` — no input; returns the current `Resume` as JSON text.
- `set_original_resume` — input is a `Resume` object matching `$defs.Resume` in `apps/rna/lib/api/schemas/resume.schema.json`.

### Job description (step 2)

- `get_job_description` — no input; returns the current job-description text.
- `set_job_description` — `{ jobDescription: string }`.

### Matched resume (steps 2 → 3)

- `get_matched_resume` — no input; returns the current matched `Resume` as JSON text.
- `set_matched_resume` — input is a `Resume` object (same schema as `set_original_resume`).

### Cover letter (step 4)

- `get_cover_letter` — no input; returns the current cover-letter text.
- `set_cover_letter` — `{ coverLetter: string }`.

### Export (step 5)

- `download_resume` — `{ content: string, filename?: string }`. **`content` is Typst markup** produced by the app's `parseResume(resumeData, template)` helper, which is **not** exposed as a WebMCP tool. Therefore the agent cannot produce valid input for `download_resume`. Treat step 5's resume export as user-driven; only step 4's cover-letter export is agent-callable:
- `download_cover_letter` — wraps the PDF downloader for the cover letter (see `apps/rna/lib/store/mcp/register-download-cover-letter.ts`).

## Form-filling rule (IMPORTANT)

When filling any field in the builder, **do not type into the textareas via keyboard automation**. Use the dedicated `set_*` tool so the app's state store updates atomically:

| Field                  | Write tool              | Read-back tool         |
| ---------------------- | ----------------------- | ---------------------- |
| Original resume        | `set_original_resume`   | `get_original_resume`  |
| Matched resume         | `set_matched_resume`    | `get_matched_resume`   |
| Job description        | `set_job_description`   | `get_job_description`  |
| Cover letter           | `set_cover_letter`      | `get_cover_letter`     |

After every `set_*` call, call the matching `get_*` to verify the value landed. If the read-back doesn't match what you wrote, retry once; on a second mismatch, report it to the user and stop.

## Form-error feedback

Form validation errors are **not yet exposed to WebMCP** (planned for a later commit). That means a `set_*` call that the app would have rejected client-side may appear to succeed even when the form is invalid. Until error feedback is wired up, the agent must **self-validate** every input against the schema/shape documented here before calling `set_*`, and must verify by reading back.

## Delegating the AI steps

Three steps (0, 2, 4) call the Go API's `/ai/*` endpoints in the regular app flow. The agent replaces those endpoints by performing the LLM work itself. **Use the dedicated skills** for those steps rather than improvising:

| Step | Skill                        | SKILL.md                                                       |
| ---- | ---------------------------- | ------------------------------------------------------------- |
| 0    | `r1resume-extract-resume`    | `.opencode/skills/r1resume-extract-resume/SKILL.md`           |
| 2    | `r1resume-match-resume`      | `.opencode/skills/r1resume-match-resume/SKILL.md`             |
| 4    | `r1resume-cover-letter`      | `.opencode/skills/r1resume-cover-letter/SKILL.md`             |

This `r1resume-navigate` skill covers only navigation, form-filling, and the user-driven steps (1 Build, 3 Review, 5 Export). When the user asks for extraction / matching / cover-letter generation, hand off to the relevant skill — do not inline its workflow.

## Step-by-step behavior for this skill

### "Go to step N" / "move to step N"

1. `get_current_step` to confirm where the user is.
2. If the target step depends on prior state (see below), check it exists before moving:
   - Step 1 (Build) requires `get_original_resume` to return non-empty data.
   - Step 2 (Match Job) requires `get_original_resume` non-empty. `get_job_description` may be empty (user can paste it on step 2).
   - Step 3 (Review) requires `get_matched_resume` non-empty.
   - Step 4 (Cover Letter) requires `get_matched_resume` non-empty (fallback: `get_original_resume`) and `get_job_description` non-empty.
   - Step 5 (Export) has no hard prerequisite; both `download_*` tools are best-effort.
3. `set_current_step({ step: N })`.

### "Fill the job description with X" / "set the job description to X"

1. Confirm current step or move to step 2.
2. `set_job_description({ jobDescription: X })` (do **not** type into the textarea).
3. `get_job_description` to verify.

### "Put my resume into the editor" / "fill in the resume form"

1. If the user has raw structured data, validate it against `apps/rna/lib/api/schemas/resume.schema.json` (`$defs.Resume`) before writing.
2. Use `set_original_resume` for the Build step (step 1) and `set_matched_resume` for the Review step (step 3). Matched-resume tailoring to a job description belongs to `r1resume-match-resume`, not here.
3. Verify with the matching `get_*` tool.

### Export

- For the cover letter: `download_cover_letter` is callable by the agent (step 5).
- For the resume: `download_resume` requires Typst content the agent cannot produce. Tell the user to click the Export button themselves.

## Guards

- Never POST to `/ai/*`. The agent replaces those endpoints.
- Never drive textareas by typing; use `set_*`.
- Never advance past a step whose prerequisite state is missing without first producing it (via the dedicated skill or a `set_*` call).
- If a WebMCP tool returns an error string, surface it verbatim to the user and stop.