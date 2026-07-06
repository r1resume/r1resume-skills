# r1resume-skills

Skills for the R1Resume builder web app. Installed via `npx skills add r1resume/r1resume-skills`.

## Structure

```
skills/r1resume/<skill-name>/SKILL.md
```

Each skill is a `SKILL.md` with YAML frontmatter (`name`, `description`).

## Gotcha: YAML frontmatter descriptions must be quoted

The skills CLI uses a YAML parser on frontmatter. Unquoted `description` values containing colons (`:`) — e.g. `e.g. "do this"` — cause silent parse failures. Always wrap descriptions in quotes:

```yaml
---
name: my-skill
description: "Does X — e.g. \"foo\", \"bar\"."
---
```

This affected `cover-letter`, `import-resume`, and `match-resume` (but not `navigate` by luck).
