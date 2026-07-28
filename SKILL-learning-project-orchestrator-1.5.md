---
name: learning-project-orchestrator
description: Bootstraps a new AVIVA-based learning project (folder skeleton, progress file, teaching/quiz skills) and runs the ongoing session loop in an existing one — deciding the next topic, handing off to `heuristic-teaching-dialogue-aviva` and `interactive-mc-quiz`, and updating the progress file. Use whenever the user wants to set up a new learning project from scratch, or wants a full learning session run end to end ("what's next", "let's continue learning", "run my session") rather than just teaching or just quizzing one thing.
version: 1.5
---

# Learning Project Orchestrator

## Two jobs
1. **Bootstrap** — one-time setup of a brand-new learning project.
2. **Orchestrate** — the recurring per-session loop in an existing
	project (e.g. this LERNEN project).

Neither job duplicates what `heuristic-teaching-dialogue-aviva` or
`interactive-mc-quiz` already do — this skill decides *when* and *what*,
then hands off to them for the actual teaching or quizzing.

## Bootstrap: setting up a new learning project

Trigger: the user wants to start a new project structured like this one
(possibly for a completely different subject).

1. Clarify (if not already obvious) the project's learning focus and
	confirm the target folder — don't assume scope.
2. Create the folder skeleton in that project:
	- `TEACHING SKILLS/` — project-local copies of the teaching/quiz
		skills, one subfolder per skill.
	- `QUIZZES/` — one ready-to-take test file per topic.
	- `ARCHIVE/` — finished/inactive files. Never a trash can — see
		"Delete means archive" below.
	- `MATERIALS/` — persisted, chapter-split source study material. See
		"Source material persistence" below.
	- `PROGRESS.md` — the project's single progress-tracking file.
3. Install the two companion skills into `TEACHING SKILLS/`:
	- `heuristic-teaching-dialogue-aviva`
	- `interactive-mc-quiz`
	Copy their current SKILL.md content as project-local files, named
	`SKILL-<name>-<version>.md`, matching the versioning convention
	below. Register all three skills (these two plus this orchestrator)
	account-wide via `save_skill` if they aren't already available
	there.
4. Create `PROGRESS.md` from this template:
	```
	# Progress — <Project Name>

	Tracks what's been covered, how confident the learner is, and what's
	still open. One section per topic. Update the existing section in
	place after every teaching session and every quiz result — don't
	create a duplicate section for a topic that already has one.

	Each topic section starts with a fenced yaml block for the
	mechanical, scriptable fields, followed by short prose for anything
	needing nuance. See "PROGRESS.md structured fields" below for the
	schema.

	## <Topic Name>
	```yaml
	last_worked: <YYYY-MM-DD>
	confidence: unknown
	status: not_started
	next_topic: null
	active_quiz: null
	material_files: []
	```
	**Confidence note:** <prose — caveats, partial results, anything not
	captured by a single enum value>
	**Open gaps:** <prose — specific shaky sub-concepts, in plain terms>
	```
5. Confirm the finished setup with the user before treating it as done.

## Orchestrate: running a session in an existing project

Trigger: the user wants to continue learning generally (not a specific
teach-this or quiz-this request) — e.g. "what's next", "let's continue",
"run my learning session".

1. **Read `PROGRESS.md`.** Determine what's covered, confidence levels,
	and open gaps.
2. **Pick the next topic.** Prioritize open gaps over brand-new topics,
	unless the user asks for something specific. If several gaps are
	open, name them and let the user choose rather than picking silently.
3. **Hand off to `heuristic-teaching-dialogue-aviva`** for the teaching
	dialogue (it owns its own Arrival framing — don't duplicate that
	here).
4. **After the topic is worked through, hand off to
	`interactive-mc-quiz`** for the knowledge check.
5. **Confirm `PROGRESS.md` was updated** with the session's outcome
	(both skills above update it as part of their own close-out — this
	step is a check, not a re-write).
6. **Offer a branch point** at the session level: continue with another
	topic, stop, or revisit a specific open gap.
7. **Anticipate the next "recommended path" item(s)** per "Anticipatory
	readiness" below, so the next session or next in-chat quiz request
	starts instantly rather than triggering live generation.

## Project-wide conventions

These apply across all three skills in this project, not just this one.

- **Delete means archive.** Never delete a file in the project's
	working folders (`QUIZZES/`, `TEACHING SKILLS/`, etc.) — move it to
	`ARCHIVE/` via `mv` instead. Never use `rm`, including during
	Claude's own cleanup.
- **Skill versioning.** Any substantive change to
	`heuristic-teaching-dialogue-aviva`, `interactive-mc-quiz`, or this
	orchestrator bumps that skill's `version`, renames its file to
	match (`SKILL-<name>-<new-version>.md`, via `mv`), and gets pushed
	live via `save_skill` (`overwrite: true`). The `TEACHING SKILLS/`
	folder itself never carries a version number.
- **Changelog README.** Every one of those version bumps also gets a
	one-line entry appended to the "Changelog" section of
	`aviva-learning-kit/README.md` (project root) — grouped under that
	skill's name, newest entry on top, matching the wording of the
	entry already added to the skill's own internal changelog. This
	keeps one shared, human-readable history across all three skills in
	one place, instead of three separate internal changelogs nobody
	reads side by side. Do this every time, immediately after the
	skill-versioning step above — not just when explicitly asked.
- **Language.** Skill documentation is in English. The actual teaching
	dialogue, quiz content, and `PROGRESS.md` entries follow the
	language of the study material — currently German for this
	project's retraining content.
- **Source material persistence.** Uploaded study material (PDFs, long
	documents) lives only in the temporary uploads folder by default —
	it can be gone by the next session. The first time a new topic's
	source material is introduced (typically during
	`heuristic-teaching-dialogue-aviva`'s session opening), persist it
	into `MATERIALS/` in the project root, one file per chapter/section
	of the source (not one giant file): plain-text `.md`, preserving
	headings, key verses/quotes, and all references, named
	`<Chapter/Section Title>.md` in the language of the study material.
	Splitting keeps each file cheap to read later and mirrors the
	one-file-per-topic pattern already used by `QUIZZES/`.
	Before converting, check the source for embedded illustrations
	(e.g. `pdfimages -list` on a PDF) — plain-text markdown silently
	drops anything that isn't extractable text. If the source has real
	diagrams, scanned figures, or tables-as-images, keep the original
	file as the source of truth alongside the markdown (or extract and
	link the images separately) rather than relying on text extraction
	alone. Do this once per topic, not per session — check `MATERIALS/`
	for an existing split before re-deriving it.
- **Anticipatory readiness ("the recommended path never waits").** As a
	learner, taking the recommended next step — the next queued quiz, or
	the next lesson — should never require waiting on live generation.
	Both arms of "what's next" stay pre-staged ahead of the ask, ideally
	via background subagents so prep doesn't consume the main
	conversation:
	- **Quiz arm.** `interactive-mc-quiz`'s own pipeline already
		guarantees a verified, ready quiz file per topic in `QUIZZES/` (see
		that skill's refill step). Delegate that refill's generation and
		verification to a general-purpose `Agent` subagent whenever the
		material and prior-question context are self-contained enough to
		hand off in a single prompt — it doesn't need conversational
		back-and-forth, so it shouldn't consume the teaching dialogue's
		turns either.
	- **Lesson arm.** Unlike a quiz, a lesson is an adaptive Socratic
		dialogue — its actual content depends on how the learner answers,
		so it can't be pre-scripted in full the way a quiz can. "Ready"
		for a lesson instead means three cheaper, non-adaptive things are
		already done ahead of the ask: (a) the next topic is already
		decided and named in `PROGRESS.md`'s "Open gaps" rather than
		picked live when the user asks "what's next"; (b) its source
		material is already split into `MATERIALS/` per "Source material
		persistence" above, done proactively rather than deferred until
		the topic actually starts; (c)
		`heuristic-teaching-dialogue-aviva`'s opening move — a brief
		Arrival framing plus the first Activate-prior-knowledge question —
		is drafted in advance from the persisted material, since that
		first move doesn't depend on any learner response yet. Everything
		past that first exchange stays live and adaptive, as the method
		requires — this is deliberately not "pre-write the whole lesson."
	- This zero-wait guarantee only covers the next queued item(s) on the
		recommended path. Out-of-path requests (quizzing an untouched
		chapter, jumping to a topic with no prepped material) are
		generated on demand as before, not held to the same bar.
- **PROGRESS.md structured fields.** Each topic section in
	`PROGRESS.md` starts with a fenced ```yaml``` block holding the
	fields that are mechanical/scriptable rather than narrative:
	- `last_worked` — ISO date of the most recent session or quiz.
	- `confidence` — one of `high` / `medium` / `low` / `unknown`.
	- `status` — one of `not_started` / `in_progress` /
		`chapter_complete` / `course_complete`.
	- `next_topic` — name of the next queued topic/chapter, or `null`.
	- `active_quiz` — path (relative to project root) of the currently
		ready quiz file for this topic in `QUIZZES/`, or `null`.
	- `material_files` — list of paths (relative to project root) of
		this topic's persisted files in `MATERIALS/`.
	Below the YAML block, keep free-text prose for anything needing
	nuance beyond a single value: `**Confidence note:**` (caveats,
	partial results, "needed a few nudges" type detail) and
	`**Open gaps:**` (specific shaky sub-concepts, in plain terms). This
	split lets a script or subagent check "what's next" or "is anything
	still shaky" from a quick, reliable read of the YAML block, without
	re-interpreting prose — while nuance that doesn't fit a single value
	stays readable prose. It mirrors the split Claude's own memory
	system already uses (structured frontmatter + prose body). Update
	both the YAML block and the relevant prose line together whenever a
	topic's section changes — they should never disagree with each
	other.
- **Demo-safe PROGRESS.md export.** `aviva-learning-kit/PROGRESS.md`
	exists only as a format example for people installing the kit (see
	"What's in this folder" in the README) — it is not a live mirror of
	this project's real `PROGRESS.md`. Never copy the project-root
	`PROGRESS.md` into `aviva-learning-kit/` wholesale. Only ever include
	topics explicitly approved as public/demo content (currently just
	`Mathematik`) — never a real personal-topic section (e.g. the
	Bible-study course content), even accidentally via a blanket file
	copy. If a new demo-worthy topic shows up, ask before adding it
	rather than assuming any given topic is fine to share. This applies
	only to `PROGRESS.md`; the three `SKILL-*.md` files and the README
	are documentation, not personal data, and sync normally. Prompted
	directly after the project-root `PROGRESS.md` (including real
	Bible-study progress) got copied into `aviva-learning-kit/` as a
	side effect of the 1.4 update — user feedback: "keep PROGRESS.md in
	demo mode when exporting – none of my personal learning data should
	be in there. the math example is fine, that was just to try sth out
	anyways."

## Versioning
This SKILL.md carries a `version` field in its frontmatter. The parent
folder name `TEACHING SKILLS` deliberately carries no version number —
versioning is skill-level only (filename + frontmatter), never
folder-level.

**Standing rule:** every substantive change to this skill bumps
`version` in the frontmatter (e.g. 1.0 → 1.1). Immediately after:
1. Rename the file to
	`SKILL-learning-project-orchestrator-<new-version>.md` (via `mv`,
	don't keep the old version file).
2. Update the live skill via `save_skill` (`overwrite: true`) in sync,
	with the same version number.

The `TEACHING SKILLS/` folder stays untouched. Pure typo fixes with no
behavior change can skip a version bump at your discretion — when in
doubt, bump anyway.

**Changelog:**
- 1.5: Added the "Demo-safe PROGRESS.md export" convention:
	`aviva-learning-kit/PROGRESS.md` is a format demo only and must
	never receive a wholesale copy of the project-root `PROGRESS.md` —
	only explicitly approved demo topics (`Mathematik`) belong there,
	never real personal-topic data. Fixes a privacy slip from the 1.4
	update, where the real file (including personal Bible-study
	progress) got copied into the shared kit folder as a side effect of
	adding the new structured-fields example. Prompted directly by user
	feedback: "keep PROGRESS.md in demo mode when exporting – none of my
	personal learning data should be in there."
- 1.4: Added the "PROGRESS.md structured fields" convention: each
	topic section now starts with a fenced YAML block for mechanical
	fields (`last_worked`, `confidence`, `status`, `next_topic`,
	`active_quiz`, `material_files`), with prose kept only for nuance
	(`Confidence note`, `Open gaps`). Updated the bootstrap template to
	match. Prompted directly by user feedback: "could you speed things
	up by keeping some things in a more machine-readable format? eg:
	progress as json etc" — resolved as hybrid frontmatter rather than
	full JSON after discussing trade-offs (JSON doesn't actually speed
	up reading since no parser is being run, and would force nuance into
	a flat schema), confirmed via `AskUserQuestion`.
- 1.3: Added the "Anticipatory readiness" project-wide convention: the
	next queued quiz and the next lesson's groundwork (topic decided,
	material persisted, opening move drafted) stay pre-staged so the
	recommended path never waits on live generation — quiz refills can
	be delegated to a background `Agent` subagent since they're
	self-contained, while a lesson's actual dialogue stays adaptive and
	un-scripted past its opening move. Prompted directly by user
	feedback: "as long as i take the recommended path, i never have to
	wait for stuff to be generated. next lessons / quiz(zes) mostly."
- 1.2: Added the "Source material persistence" project-wide
	convention and `MATERIALS/` to the folder skeleton: uploaded study
	material gets split into one file per chapter/section and saved
	into the project (not left dependent on the temporary uploads
	folder), with an explicit check for embedded illustrations before
	relying on plain-text extraction. Prompted by splitting the
	"Strategies For Spiritual Harvest" course PDF into per-chapter
	files after realizing the original upload wasn't persisted anywhere
	in the project.
- 1.1: Added the "Changelog README" project-wide convention: every
	version bump across any of the three skills now also gets a
	one-line entry appended to `aviva-learning-kit/README.md`'s
	Changelog section, so the three skills' internal changelogs are
	mirrored into one shared, readable place.
- 1.0: Initial version. Formalizes and supersedes the informal
	"Standard-Loop pro Thema" and "Fortschritt festhalten" sections from
	the project's original `PROJEKT-PROMPT.md` (now archived) — this
	time with a concrete progress file (`PROGRESS.md`) instead of prose
	instructions to "kurz notieren" with no defined artifact, and with
	the "delete means archive" / versioning conventions written into the
	project itself rather than living only in Claude's memory.
