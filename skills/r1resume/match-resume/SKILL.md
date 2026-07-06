---
name: r1resume-match-resume
description: "Use when the user wants to tailor or match their resume to a job description in the R1Resume builder — e.g. \"match my resume to this job\", \"tailor my resume for this posting\", \"optimize my resume for ATS\". The agent acts as the LLM that the Go API's `/ai/match-resume` endpoint would otherwise call: it reads the original resume and job description via WebMCP, rewrites the resume itself, and writes the matched result back via `set_matched_resume`."
---

# R1Resume — Step 2: Match Resume to Job Description

## What this skill does

In the regular app flow, step 2 calls `POST /ai/match-resume` (see `apps/api/internal/ai/controller.go` and `apps/api/internal/ai/gemini_client.go`), which feeds the original resume + job description to Gemini with a system prompt and expects a `Resume` JSON back, then calls `resume.CorrectOrder()`.

**The agent replaces that endpoint.** You are the LLM. You read the original resume and the job description via WebMCP, rewrite the resume yourself, and write it back via `set_matched_resume`.

Note: the Go service wraps the AI call with a `skimResumeFields` / `restoreResumeFields` pair that strips personal-contact fields before the rewrite and restores them after, to avoid the AI rewriting contact info. This skill **does not** replicate that wrapper — per project decision, the agent rewrites the resume in place. Preserve the personal-contact fields directly in your output (do not modify `location`, `phone`, `website`, `socialNetworks`, `order`, or the entries of `references`/`interests`/`volunteer`/`awards`).

## Prerequisites

See `r1resume-navigate`. Confirm `chrome-devtools_list_webmcp_tools` shows `get_original_resume`, `get_job_description`, `set_job_description`, `get_matched_resume`, `set_matched_resume`, `set_current_step`.

## Workflow

1. **Ensure a job description exists.**
   - `get_job_description`.
   - If empty, ask the user to provide it. If the user supplies it in the chat, call `set_job_description({ jobDescription: <text> })` and verify with `get_job_description`. Do **not** type into the textarea (see `r1resume-navigate`'s form-filling rule).
   - If still empty, stop and tell the user a job description is required.

2. **Read the original resume.**
   - `get_original_resume`. If it returns empty/null, the user must run the extract step first (see `r1resume-import-resume`). Stop and tell them.

3. **Rewrite the resume.** Apply this system prompt verbatim to yourself, using the original resume and the job description as input:

   > You are an expert resume optimization agent for editing resume data to maximize ATS compatibility and job-match relevance.
   >
   > CORE TASK:
   > Rewrite resume content to emphasize relevant experience and incorporate keywords from the job posting. The total length of the resume must remain the same as the original.
   >
   > OPTIMIZATION RULES:
   > - Rewrite descriptions with strong action verbs and job-specific terminology
   > - Incorporate job posting keywords naturally throughout all sections
   > - Quantify achievements with metrics where supported by candidate's background
   > - Lead with impact: "Architected X resulting in Y" rather than "Responsible for X"
   > - Include both acronyms and full terms (e.g., "CI/CD (Continuous Integration)")
   > - Keep content concise while improving relevance
   >
   > LENGTH CONSTRAINT (CRITICAL):
   > The total length of the resume output must match the total length of the resume input.
   >
   > PRESERVE:
   > - Factual accuracy - never invent experience or skills
   > - Core tech stack and timeline integrity
   > - Existing section structure
   >
   > MODIFY:
   > - Wording, phrasing, and emphasis of all descriptions
   > - Order of skills and accomplishments
   > - Summary statement language
   >
   > OUTPUT:
   > Return the complete optimized resume data in the same structure.

   Output a single `Resume` object matching `$defs.Resume` in `apps/rna/lib/api/schemas/resume.schema.json`. Additional rules:
   - **Preserve length.** The output should be roughly the same size as the input (do not pad or trim materially).
   - **Preserve contact fields.** Do not modify `location`, `phone`, `website`, `socialNetworks`, or the `order` array. Do not modify the entries of `references`, `interests`, `volunteer`, or `awards` custom sections.
   - **Factual accuracy.** Never invent experience or skills not in the original resume; only reword and reorder.
   - Apply the same final step the Go server does (`resume.CorrectOrder()`): preserve conventional section ordering and any explicit `order` array.

4. **Validate before writing.** Self-validate against the `Resume` schema. Form-error feedback is not yet exposed via WebMCP (see `r1resume-navigate`), so a malformed `Resume` may appear to succeed — fix schema violations before calling the next step.

5. **Write the matched resume.**
   - `set_matched_resume({ ...the Resume object })` — input is the full `Resume` object.

6. **Verify and advance.**
   - `get_matched_resume` — read back and confirm it matches what you wrote.
   - If the read-back matches, call `set_current_step({ step: 3 })` to move to the Review step, mirroring the app's `onSuccess` (which calls `setCurrentStep(3)`).
   - If the read-back does **not** match, retry `set_matched_resume` once. On a second mismatch, report it to the user and stop — do not advance.

## Skip behavior

The app exposes a **Skip** button on step 2 that calls `setMatchedResume(originalResume)` and `setCurrentStep(3)` — i.e. it passes the original resume through unchanged. If the user explicitly says "skip matching", you may call `set_matched_resume` with the content of the current `get_original_resume`, verify, and `set_current_step({ step: 3 })`. Do not invoke this skill's rewrite workflow for a skip.

## Guards

- Never POST to `/ai/match-resume`. You are replacing it.
- Never type into the job-description textarea; use `set_job_description`.
- Never invent experience, skills, or metrics not in the original resume.
- Never modify personal-contact fields during the rewrite.
- Never advance to step 3 until `get_matched_resume` confirms the data landed.

## Reference

- System prompt source: `apps/api/internal/ai/gemini_client.go:18` (`matchResumeSystemPrompt`).
- Service wrapping (not replicated here): `apps/api/internal/ai/service.go:69` (`MatchResume`, `skimResumeFields`, `restoreResumeFields`).
- Resume schema: `apps/rna/lib/api/schemas/resume.schema.json` (`$defs.Resume`).
- WebMCP tools: `apps/rna/lib/store/mcp/register-get-original-resume.ts`, `register-get-job-description.ts`, `register-set-job-description.ts`, `register-get-matched-resume.ts`, `register-set-matched-resume.ts`, `register-set-current-step.ts`.