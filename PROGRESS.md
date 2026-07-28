# Progress — LERNEN

Tracks what's been covered, how confident the learner is, and what's
still open. One section per topic. Update the existing section in
place after every teaching session and every quiz result — don't create
a duplicate section for a topic that already has one.

Each topic section starts with a fenced yaml block for the mechanical,
scriptable fields (`last_worked`, `confidence`, `status`, `next_topic`,
`active_quiz`, `material_files`), followed by short prose for anything
needing nuance (`Confidence note`, `Open gaps`). See
`learning-project-orchestrator`'s "PROGRESS.md structured fields"
convention for the schema.

This copy is a demo — it shows the file's format, not this project's
real progress data. See "Demo-safe PROGRESS.md export" in
`learning-project-orchestrator`'s conventions.

## Mathematik
```yaml
last_worked: 2026-07-28
confidence: unknown
status: in_progress
next_topic: null
active_quiz: QUIZZES/Quizz – Mathematik 2026-07-28_08-26.html
material_files: []
```
**Confidence note:** not yet recorded — this entry was backfilled when
`PROGRESS.md` was introduced. A quiz round is already archived
(`Quizz – Mathematik 2026-07-28_08-24.html`) and a fresh one is queued
in `QUIZZES/`. Confirm the actual score and update this section at the
next session.
**Open gaps:** unknown — confirm at next session.
