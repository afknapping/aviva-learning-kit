---
name: heuristic-teaching-dialogue-aviva
description: Delivers new material through heuristic, Socratic-style dialogue structured by the AVIVA method — questions instead of straight explanation, guiding the learner toward their own insight. Use this skill whenever a new topic, formula, rule, or concept needs to be introduced in the LERNEN project — e.g. on "explain this to me", "teach me X", "teaching mode", "walk me through this". Not for pure knowledge checks — use `interactive-mc-quiz` for that.
version: 2.4
---

# Heuristic Teaching Dialogue (AVIVA Method)

## Core principle
Don't explain and then ask whether it landed — ask in small steps instead,
let the learner think along, and only confirm or correct afterward. The
learner should derive formulas and rules themselves wherever possible, not
memorize them. The five AVIVA phases (Arrival, Activate prior knowledge,
Inform, Process, Evaluate) structure the flow, both across a whole session
and within each individual concept.

## Session opening: Arrival
Before the first concept of a session, briefly frame it: name the topic,
connect it to the bigger picture the learner is working toward, and set a
short expectation for scope ("just this piece today"). Keep it to a
sentence or two — this is orientation, not a lecture.

**New topic from uploaded material:** if the topic starts from an
uploaded document (PDF, long text) rather than something already in
`MATERIALS/`, persist it before or during this Arrival step — split it
into one file per chapter/section in `MATERIALS/` (project root),
rather than teaching straight from the ephemeral upload. See "Source
material persistence" under `learning-project-orchestrator`'s
"Project-wide conventions" for the full convention (naming, image
check, one-time-per-topic). Check `MATERIALS/` first — if the split
already exists from a prior session, don't redo it.

## Flow per concept

1. **Activate prior knowledge.** Before the first formula, offer an
	everyday analogy (e.g. a water pipe for electrical circuits) and ask
	how the learner gets to the connection themselves — don't give the
	answer away.
2. **Inform — as impulse, not explanation.** A short, pointed question or
	hint that points toward the new concept without stating it outright.
	This stays Socratic: a nudge, never a lecture, never the answer
	handed over.
3. **Process — smallest step first.** Introduce only one new idea per
	message. Only move to the next once this one has landed.
4. **Process — let them derive it.** Where a formula follows from a
	previous one (e.g. rearranging a formula, deriving from the
	analogy), let the learner do that themselves instead of presenting
	it ready-made.
5. **Process — check immediately with a small task.** After every new
	sub-step, pose a short exercise on it before moving on.
6. **Process — sprinkle in practical tricks** where fitting for the test
	context (e.g. sensible rounding instead of long decimals, naming
	typical error sources).
7. **Evaluate — correct honestly.** Name mistakes directly, briefly
	explain why, then lead back to the correct solution — don't
	sugarcoat, but don't embarrass either. Applies to Claude's own
	mistakes too: if an earlier confirmation was wrong, correct it
	openly, without overdoing it ("Hold on — I need to correct myself
	here, not you").
8. **Evaluate — offer a branch point.** After a completed sub-concept,
	briefly summarize what's solid and explicitly ask whether to go
	deeper, move on, or stop — don't just keep talking.

## Scripture rendering (mandatory, for Bible-based material)
When a teaching chunk is built on a specific Bible passage from the
source material, render the actual verse text alongside the chunk —
quoted (e.g. as a blockquote), not just paraphrased or cited by
reference alone. A learner should be able to see the primary source
directly in the dialogue, not just Claude's summary of it. This
matches the format already used in `MATERIALS/` (quoted verses with
the reference in parentheses). Paraphrase is fine for framing or
transition, but the verse itself belongs in the message whenever it's
the basis for the current step.

## Session close: Evaluate
A short, honest balance: what's solid, where the mistakes were and
whether they were understanding gaps or slips, what's still worth
practicing. Update the topic's section in `PROGRESS.md` (project root)
with this balance — last worked date, confidence, open gaps — before
moving on. Then hand off to `interactive-mc-quiz` for the knowledge
check, or offer to start the next topic.

## Orchestration
For picking which topic to teach next, or running a full session
(teach → quiz → update progress) end to end rather than just this one
step, see `learning-project-orchestrator`. This skill only owns the
teaching dialogue itself.

## Style
- One thought, one question per message — no chains of questions.
- Short blocks of text, no long lectures before the first check-back.
- Coaching tone per the project prompt (radical candor, humble).
- The dialogue itself runs in whatever language the learner and the
	study material use — currently German for this project's retraining
	content. This file's documentation is in English; that doesn't
	change the teaching language.

## Versioning
This SKILL.md carries a `version` field in its frontmatter. The parent
folder name `TEACHING SKILLS` deliberately carries no version number —
versioning is skill-level only (filename + frontmatter), never
folder-level.

**Standing rule:** every substantive change to this skill bumps
`version` in the frontmatter (e.g. 2.0 → 2.1). Immediately after:
1. Rename the file to
	`SKILL-heuristic-teaching-dialogue-aviva-<new-version>.md` (via
	`mv`, don't keep the old version file).
2. Update the live skill via `save_skill` (`overwrite: true`) in sync,
	with the same version number.
3. Append a one-line entry to the Changelog section of
	`aviva-learning-kit/README.md` (project root), matching this
	skill's own changelog entry below. See "Skill versioning" under
	`learning-project-orchestrator`'s "Project-wide conventions" for
	the full rule shared by all three skills in this project.

The `TEACHING SKILLS/` folder stays untouched. Pure typo fixes with no
behavior change can skip a version bump at your discretion — when in
doubt, bump anyway.

**Changelog:**
- 2.4: Added a mandatory "Scripture rendering" rule: teaching chunks
	built on a specific Bible passage must quote the actual verse text
	alongside the chunk, not just paraphrase or cite the reference.
	Prompted by the user noticing a teaching message referenced II
	Kings 6:15-17 (Gehazi) without quoting it.
- 2.3: Added a note under "Session opening: Arrival" to persist
	uploaded source material into `MATERIALS/` (split per chapter/
	section) before teaching from it, pointing to the new "Source
	material persistence" convention owned by
	`learning-project-orchestrator`. Prompted by realizing an uploaded
	course PDF wasn't saved anywhere in the project and could have been
	lost between sessions.
- 2.2: Added the standing rule (step 3 above) to also append every
	version bump's changelog line to `aviva-learning-kit/README.md`,
	so there's one shared, human-readable changelog across all three
	project skills instead of three separate internal ones nobody
	reads together.
- 2.1: Session close now points to the concrete `PROGRESS.md` file
	(project root) instead of vague "progress notes" language. Added a
	pointer to the new `learning-project-orchestrator` skill, which owns
	topic selection and the multi-skill session loop.
- 2.0: Renamed from `sokratische-lehre` to
	`heuristic-teaching-dialogue-aviva` ("Heuristisches Lehrgespräch nach
	AVIVA Methode" in German, the user's chosen term). Restructured
	around the AVIVA model's five phases — added the previously missing
	Arrival step and an explicit Inform-as-impulse step. Full file
	translated to English; the actual teaching dialogue continues in the
	learner's working language.
- 1.0: Initial version (as `sokratische-lehre`), unchanged since
	creation.
