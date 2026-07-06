---
name: r1resume-cover-letter
description: "Use when the user wants to write or generate a cover letter in the R1Resume builder — e.g. \"write a cover letter\", \"generate a cover letter for this job\", \"draft a cover letter\". The agent acts as the LLM that the Go API's `/ai/cover-letter` endpoint would otherwise call: it reads the matched resume and job description via WebMCP, writes the cover letter itself, and writes it back via `set_cover_letter`."
---

# R1Resume — Step 4: Generate Cover Letter

## What this skill does

In the regular app flow, step 4 calls `POST /ai/cover-letter` (see `apps/api/internal/ai/controller.go` and `apps/api/internal/ai/gemini_client.go`), which feeds the matched resume + job description to Gemini with a system prompt and expects plain cover-letter text back.

**The agent replaces that endpoint.** You are the LLM. You read the matched resume and job description via WebMCP, write the cover letter yourself, and write it back via `set_cover_letter`.

## Prerequisites

See `r1resume-navigate`. Confirm `chrome-devtools_list_webmcp_tools` shows `get_matched_resume`, `get_original_resume`, `get_job_description`, `set_cover_letter`, `get_cover_letter`, `set_current_step`.

## Workflow

1. **Read the resume.**
   - `get_matched_resume`. If it returns empty/null, fall back to `get_original_resume`.
   - If both are empty, the user must run extract (and optionally match) first — see `r1resume-import-resume` and `r1resume-match-resume`. Stop and tell them.

2. **Ensure a job description exists.**
   - `get_job_description`.
   - If empty, ask the user to provide it. If the user supplies it in the chat, call `set_job_description({ jobDescription: <text> })` and verify with `get_job_description`. Do **not** type into the textarea (see `r1resume-navigate`'s form-filling rule).
   - If still empty, stop and tell the user a job description is required.

3. **Write the cover letter.** Apply this system prompt verbatim to yourself, using the resume and the job description as input:

   > You are an expert cover letter writer. Analyze the resume and job description to create a compelling, one-page cover letter (200-250 words).
   > Structure:
   >
   > Opening: Hook with specific enthusiasm and one strong fit
   > Body: 2-3 key qualifications with quantifiable examples matching job requirements
   > Closing: Express interest with confident call-to-action
   >
   > Style:
   >
   > Professional but conversational
   > Specific achievements with metrics, not generic claims
   > Active voice, strong verbs
   > No clichés or resume repetition
   >
   > Format:
   >
   > Standard business letter with proper greeting and sign-off
   > Clear paragraph breaks
   >
   > Critical: Never use placeholders like [Date], [Name], [Company]. Extract all information from the provided resume and job description. If information cannot be determined, skip that element entirely. Output must be complete and ready to send.

   Output a single plain-text cover letter (200–250 words). Additional rules:
   - **No placeholders.** Never emit `[Date]`, `[Name]`, `[Company]`, `[Hiring Manager]`, etc. If the recipient's name cannot be determined from the resume or job description, omit the name line / use a generic greeting like "Dear Hiring Team,".
   - **No clichés** and do not repeat the resume verbatim — synthesize.
   - The output must be **complete and ready to send**.

4. **Write the cover letter.**
   - `set_cover_letter({ coverLetter: <text> })` — input is the full cover-letter text.

5. **Verify and advance.**
   - `get_cover_letter` — read back and confirm it matches what you wrote.
   - If the read-back matches, call `set_current_step({ step: 5 })` to move to the Export step, mirroring the app's `onSuccess` (which calls `setCurrentStep(5)`).
   - If the read-back does **not** match, retry `set_cover_letter` once. On a second mismatch, report it to the user and stop — do not advance.

## Skip behavior

The app exposes a **Skip** button on step 4 that calls `setCurrentStep(5)` without generating. If the user explicitly says "skip the cover letter", just call `set_current_step({ step: 5 })` — do not invoke this skill's write workflow.

## Guards

- Never POST to `/ai/cover-letter`. You are replacing it.
- Never type into the cover-letter field; use `set_cover_letter`.
- Never use placeholders; if information is missing, skip that element entirely.
- Never advance to step 5 until `get_cover_letter` confirms the text landed.

## Reference

- System prompt source: `apps/api/internal/ai/gemini_client.go:46` (`generateCoverLetterSystemPrompt`).
- WebMCP tools: `apps/rna/lib/store/mcp/register-get-matched-resume.ts`, `register-get-original-resume.ts`, `register-get-job-description.ts`, `register-set-cover-letter.ts`, `register-get-cover-letter.ts`, `register-set-current-step.ts`.