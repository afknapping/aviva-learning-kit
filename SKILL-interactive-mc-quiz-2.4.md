---
name: interactive-mc-quiz
description: Creates an interactive multiple-choice quiz for a knowledge check. Use this skill whenever the LERNEN project needs a test, quiz, exam, self-check, or knowledge probe on a topic just covered — even if "multiple choice" isn't said explicitly. Applies to any learning topic in this project, not just retraining prep.
version: 2.4
---

# Interactive Multiple-Choice Quiz

## When to trigger
- The user asks for a test, quiz, probe, or practice questions on a topic
	just discussed.
- The user says "quiz me", "multiple choice", "grade me", or similar.

## Core principle: file pipeline instead of live generation
Waiting for a fresh generation costs time the user doesn't want to lose
on a spontaneous test. So, wherever possible, a finished, verified,
rendered HTML test file already sits ready per topic. A requested test
is served instantly from that file. Right after — still within the same
response — the next test for the same topic is already generated in the
background, verified, and saved as a new file, so something is ready
again next time too.

## Live delivery: auto-scored widget (in-chat) — default alongside the file

A plain `.html` file in `QUIZZES/` has no live connection back to Claude —
there is no tool that can read a static file preview's state, confirmed
by testing `read_widget_context` against it (it returns "no widget
context available"). So for in-chat delivery, render the *same*
`QUESTIONS` data a second time as a live interactive widget (via the
visualize tool's `show_widget`), which auto-reports the result the
moment the user clicks submit — no manual "tell me your score" needed.

This is not a second quiz to write — it's the same verified question
list, formatted twice: once into the portable file (for `QUIZZES/`,
offline use, sharing, taking outside the chat), once into the live
widget (for in-chat use with automatic scoring).

**How the auto-report works:** the widget's submit handler calls the
visualize tool's global `sendPrompt(text)` function — this pushes a
message into the chat exactly as if the user had typed it. The message
must follow this exact, parseable format so it can be recognized
reliably in conversation:

```
Quiz result — <Topic>: X of Y correct.
```

When a message in this format appears in the conversation, treat it
exactly like the manual "test taken" signal described under
"Archiving" below — no need for the user to additionally confirm it.

**Widget requirements** (on top of the visualize skill's own design
system rules — flat design, CSS variables, sentence case, no titles or
prose inside the widget itself):
- Start with a visually-hidden `<h2 class="sr-only">` summarizing the
	quiz for screen readers.
- One question block per item, radio-button options, bare `<input>` /
	`<button>` tags (pre-styled by the host).
- A single submit button whose label live-updates
	`Submit (X of Y answered)` as with the file version.
- On submit: color each question block via `var(--bg-success)` /
	`var(--bg-danger)` and `var(--text-success)` / `var(--text-danger)`
	for right/wrong (these are the CDS role tokens — not the file
	template's own `--success-bg` custom variables, which only exist in
	the standalone HTML file's own stylesheet), lock the radio inputs,
	then call `sendPrompt` with the exact result string above.
- No `sendPrompt` call before the user has actually submitted — only
	fire it once, on the submit click.

**Hide-until-ready (mandatory — prevents lost clicks while streaming):**
`show_widget` content streams into the chat progressively, and the
widget appears to fully re-render on each incoming chunk until the
stream finishes and the `<script>` attaches — so any click registered
before that point is silently lost. There is no way to fix this inside
the widget's own JS timing; the fix is structural, in the markup order:
- The question blocks (`#quiz-root` or equivalent) start with inline
	`style="display: none;"` in the HTML itself — hidden from the very
	first streamed chunk, not hidden later by a script.
- A simple loading indicator (e.g. a thin animated bar, pure CSS
	`@keyframes`, no library) is visible by default in its place, with
	text like "Preparing quiz…".
- The `<script>` block — which only ever executes after the full
	widget has streamed in and attached — is what flips this: hide the
	loading indicator, un-hide the quiz root, wire up the change/submit
	listeners. All in the same script, so the questions only become
	visible at the exact moment they also become clickable.
- Net effect: the user only ever sees either "still preparing" or
	"fully interactive" — never a half-rendered, clickable-but-about-to-
	reset state in between.

This was validated live in-project: a 2-question proof-of-concept
widget correctly auto-reported `Quiz auto-report test: scored 2 out of
2.` into the chat on submit, confirming `sendPrompt` is a reliable
readback channel where `read_widget_context` against a plain file is
not. The hide-until-ready pattern was added after the user reported
losing answers by clicking mid-stream on the real 8-question Chapter 1
quiz, and confirmed fixed on the next render.

## Folder structure (in the LERNEN project)
- `QUIZZES/` — holds exactly one ready-to-take test file per topic.
	Lives in the project root.
- `ARCHIVE/` — taken or finished test files land here (an existing
	project folder, used project-wide as a store for "no longer active",
	not a trash can). Lives in the project root.
- The technical template for rendering new test files has been embedded
	directly in this SKILL.md since version 1.3 (see "Template:
	quiz-template.html" below) — no external `assets/` file needed. That
	lets this skill be shared as a single markdown file, without zipping
	up a whole folder.
- `TEACHING SKILLS/` bundles the project-local copies of the skills used
	in the LERNEN project (e.g. also
	`TEACHING SKILLS/heuristic-teaching-dialogue-aviva/`), kept separate
	from the learning content (`QUIZZES/`, `ARCHIVE/`, `Diagramme/`). The
	folder name carries no version number — versioning happens per skill
	only, via filename and frontmatter (see "Versioning" below).

## Filename (mandatory)
`Quiz – <Topic> <Timestamp>.html`, saved in `QUIZZES/`.
- `<Topic>`: plain-text topic name (e.g. "Mathematik", "Ohmsches Gesetz",
	"Schaltsymbole") — no slugs in the filename. Keep the topic name in
	whatever language the study material uses.
- `<Timestamp>`: format `YYYY-MM-DD_HH-MM` (sortable, no colons for
	filesystem compatibility), time of generation.
- Example: `Quiz – Mathematik 2026-07-28_08-26.html`
- Note: files generated before this version used the spelling
	`Quizz –` (double z) — a historical artifact of the earlier
	German-only version. Existing files are not renamed retroactively;
	only new files use the `Quiz –` spelling. When checking `QUIZZES/`
	for an existing file (step 2 below), match both spellings.

## Flow

1. **Determine the topic.** Derive a plain-text name from the topic just
	discussed or requested.
2. **Check `QUIZZES/`.** Is there already a file there whose name starts
	with `Quiz – <Topic> ` (or the legacy `Quizz – <Topic> `)?
3. **File exists (the normal case):**
	a. This file is the test to deliver — present it directly, without
		modifying it first.
	b. Also render the same verified `QUESTIONS` as a live auto-scoring
		widget per "Live delivery" above, so the in-chat path doesn't
		require a manual score report.
	c. Briefly let the user know the file can be moved to `ARCHIVE` or
		deleted themselves once done, if the test is taken outside the
		chat (directly in the browser) rather than via the live widget
		(see "Archiving" below).
	d. Immediately kick off step 5 (refill the pipeline), without
		waiting for a signal from the user — the delivered file stays
		untouched in `QUIZZES/` until it's demonstrably been taken (see
		Archiving).
4. **No file exists (new topic, or the pipeline happened to be empty):**
	a. Derive questions as usual from the material just discussed (not
		from outside sources) — 5–15 questions depending on scope, mixed
		order across sub-topics.
	b. Run it through verification (step 6) before delivering.
	c. Render it as a new file per the naming convention in `QUIZZES/`
		and deliver it, plus the same questions as a live auto-scoring
		widget per "Live delivery" above.
	d. Immediately kick off step 5, so a pipeline now exists for this
		topic going forward.
5. **Refill the pipeline (after every test delivery):**
	a. Derive a new question list for the same topic from the material
		discussed so far (different questions/emphasis than the
		just-delivered test, as far as the material allows).
	b. Run it through verification (step 6).
	c. Only render it as a new file once it passes verification. This
		file is NOT placed in `QUIZZES/` immediately while the previous
		file for the same topic is still there (see Archiving) — hold it
		internally instead (e.g. as a draft in the response context or
		briefly in the working folder) and move it to `QUIZZES/` only
		once the old file gets archived. Exception: if there was no file
		for the topic at all to begin with (case 4), place it in
		`QUIZZES/` right away.
	d. This runs "in the background" — the user doesn't wait on it. If
		verification rejects questions and too few remain, prefer a
		shorter but clean question list over keeping unreliable
		questions.

## Archiving (when a file disappears from QUIZZES/)

- **Test taken in chat ("inline"):** As soon as the user signals in
	conversation that a specific test is done — e.g. they state a
	result, say "done", ask for the next test on the same topic, or the
	live widget auto-reports via the `Quiz result — <Topic>: X of Y
	correct.` message described under "Live delivery" above — the file
	in question is moved from `QUIZZES/` to `ARCHIVE/` via `mv` (never
	delete; see the project convention "delete means archive", which
	also applies to Claude's own cleanup — never use `rm`). Immediately
	after, move the successor file prepared in step 5 into `QUIZZES/`,
	so a test is ready again right away.
- **Test taken outside the chat** (the user opens the HTML file
	themselves in the browser without mentioning it in chat): Claude has
	no way to detect this. The file stays put until the user moves it to
	`ARCHIVE` or deletes it by hand. Mention this briefly when delivering
	a file (see step 3b).
- Never remove a file from `QUIZZES/` without a concrete signal in
	conversation that it's done.

## Regular check: all topics covered?

Whenever this skill or a learning topic in the LERNEN project comes up
(e.g. skill invocation, start of a new learning session, explicit
question), briefly cross-check:
- Which topics have already been covered, per `PROGRESS.md` (project
	root) and prior conversation?
- Does every one of those topics have a file ready in `QUIZZES/` (name
	starting with `Quiz – <Topic> `)?
- If a topic is missing a file (neither in `QUIZZES/` nor as a prepared
	pipeline draft), generate and verify one per step 4 before it's
	needed.
- If many topics are missing at once (e.g. initial setup), briefly check
	with the user whether to catch up on all of them now, rather than
	doing it unprompted at large scale.

## Verification (mandatory before any delivery and before placing anything in QUIZZES/)

a. Commission an independent subagent (Agent tool) to check every
	question against the material: Is the question factually correct?
	Is the index marked `correct` actually the right answer? Is the
	wording unambiguous (no two plausible answers)?
b. The subagent has no memory of the conversation — so give it, in the
	prompt, both the full question draft (including all options and the
	marked index) and a short, self-contained summary of the underlying
	material.
c. Have the subagent correct or strike faulty or ambiguous questions;
	only use the verified result going forward.
d. For questions on established technical knowledge (formulas, data,
	facts), the subagent may also use a web search for cross-checking if
	needed.
e. Additionally check the distribution of `correct` indices across the
	whole question set (see "Distribution of the correct answer" below)
	and reshuffle the options of affected questions if skewed (change
	option order, adjust the index accordingly) — question text and
	content stay unchanged.

### Distribution of the correct answer
- Distribute the position of the correct answer (index 0–3 among the
	options) as evenly as possible across the whole test — not
	predominantly option 1 (index 0), and not any other position
	throughout either.
- Rule of thumb: no position should be `correct` for more than ~30% of a
	test's questions (so max 3× the same position out of 10 questions).
- Roll the order independently per question, not by rotating through a
	fixed pattern (0,1,2,3,0,1,2,3,…) — that's just as guessable as
	always index 0.

## Format specification (mandatory)
- One question per block, 4 answer options as radio buttons, exactly
	one correct answer.
- One button at the end of all questions. Its text live-shows answering
	progress: `Submit (X of Y answered)`, updating on every selection.
- On clicking the button:
	- Each question is colored as a whole: fully correctly answered →
		green background across the whole question block. Wrong or
		unanswered → red background across the whole question block.
	- Within a wrong or unanswered question, the correct option is
		additionally highlighted in canary yellow.
	- Selection gets locked (radio buttons disabled).
	- The button is hidden (redundant after grading).
	- The page smooth-scrolls to the overall result at the end.
- The overall result and a tiered feedback message appear below the
	questions, where the button used to be.
- Tier message staggered by score, with color coding:
	- 100%: celebratory ("tinsel") — clearly visible.
	- ≥ 80%: positive, but measured.
	- ≥ 50%: neutral, names remaining gaps.
	- < 50%: matter-of-fact, points to targeted practice.
- UI text (button, labels, feedback, tier messages) is in English per
	the template below. The question and answer content itself follows
	the language of the underlying study material — currently German for
	this project's topics.

## Technical implementation
The template in the "Template: quiz-template.html" section below is a
self-contained, functional HTML file (own CSS, no dependency on
claude.ai color variables) that already covers the full format
specification above.

Procedure for rendering a new test file:
1. Use the code block from "Template: quiz-template.html" below as the
	starting point for a new file.
2. Replace the `QUESTIONS` array at the top of the `<script>` block with
	the freshly verified questions (structure: `question`, `options`,
	`correct`, where `correct` is the 0-based index of the correct
	option).
3. Save as a new file per the naming convention above in `QUIZZES/` (or
	hold it internally per the pipeline rule, see step 5c).

Changes to the template (e.g. new feedback or styling rules) apply to
future rendered tests; files already delivered and sitting in
`QUIZZES/` or `ARCHIVE/` are not retroactively updated — migrate to the
new template explicitly if needed. Changes to the template count as a
substantive skill change and therefore fall under "Versioning" below.

## Template: quiz-template.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Knowledge Check</title>
<style>
	:root {
		--bg: #ffffff;
		--text: #1a1a1a;
		--text-muted: #6b6b68;
		--border: #dddad2;
		--success-bg: #eaf3de;
		--success-text: #27500a;
		--warning-bg: #faeeda;
		--warning-text: #633806;
		--danger-bg: #fcebeb;
		--danger-text: #791f1f;
		--canary-bg: #fff9c4;
		--canary-text: #6b5900;
	}
	@media (prefers-color-scheme: dark) {
		:root {
			--bg: #1c1c1a;
			--text: #f0efe9;
			--text-muted: #b4b2a9;
			--border: #444441;
			--success-bg: #173404;
			--success-text: #c0dd97;
			--warning-bg: #412402;
			--warning-text: #fac775;
			--danger-bg: #501313;
			--danger-text: #f7c1c1;
			--canary-bg: #4a4000;
			--canary-text: #ffe97a;
		}
	}
	body {
		font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Arial, sans-serif;
		background: var(--bg);
		color: var(--text);
		max-width: 680px;
		margin: 2rem auto;
		padding: 0 1rem;
		line-height: 1.6;
	}
	.question-block {
		margin-bottom: 1.25rem;
		padding-bottom: 1.25rem;
		border-bottom: 1px solid var(--border);
		border-radius: 8px;
		transition: background 0.15s ease;
	}
	.question-block.question-correct {
		background: var(--success-bg);
		padding: 12px 14px 14px;
		border-bottom: none;
	}
	.question-block.question-incorrect {
		background: var(--danger-bg);
		padding: 12px 14px 14px;
		border-bottom: none;
	}
	.question-text {
		font-weight: 600;
		margin: 0 0 8px;
	}
	.options {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
		gap: 8px;
	}
	label {
		display: flex;
		align-items: center;
		gap: 6px;
		font-size: 14px;
		color: var(--text-muted);
		cursor: pointer;
		padding: 3px 6px;
		margin: -3px -6px;
		border-radius: 6px;
	}
	label.option-canary {
		background: var(--canary-bg);
		color: var(--canary-text);
	}
	.feedback {
		font-size: 13px;
		margin: 8px 0 0;
		display: none;
	}
	button#submit {
		width: 100%;
		padding: 10px;
		margin-top: 1rem;
		border-radius: 8px;
		border: 1px solid var(--border);
		background: transparent;
		color: var(--text);
		font-size: 14px;
		cursor: pointer;
	}
	#result-box {
		display: none;
		flex-direction: column;
		gap: 10px;
		border-radius: 8px;
		padding: 1rem 1.25rem;
		margin-top: 1rem;
	}
	.result-row {
		display: flex;
		align-items: center;
		justify-content: space-between;
	}
	#tier-text {
		font-size: 14px;
		font-weight: 600;
	}
</style>
</head>
<body>

<div id="questions"></div>

<button id="submit">Submit (0 of 0 answered)</button>

<div id="result-box">
	<div class="result-row">
		<span style="font-size:14px; color:var(--text-muted);">Result</span>
		<span id="score-value" style="font-size:22px; font-weight:600;">–</span>
	</div>
	<span id="tier-text"></span>
</div>

<script>
	// Replace the questions here — keep the same structure.
	// "correct" is the 0-based index of the correct option.
	const QUESTIONS = [
		{
			question: "Example: U = 12 V, R = 3 Ω. What is I?",
			options: ["4 A", "36 A", "0.25 A", "9 A"],
			correct: 0
		}
	];

	const container = document.getElementById("questions");

	QUESTIONS.forEach((item, i) => {
		const block = document.createElement("div");
		block.className = "question-block";
		block.id = "block-" + i;

		const questionEl = document.createElement("p");
		questionEl.className = "question-text";
		questionEl.textContent = (i + 1) + ". " + item.question;
		block.appendChild(questionEl);

		const optionsWrap = document.createElement("div");
		optionsWrap.className = "options";

		item.options.forEach((opt, j) => {
			const label = document.createElement("label");
			label.id = "label-" + i + "-" + j;
			const radio = document.createElement("input");
			radio.type = "radio";
			radio.name = "question" + i;
			radio.value = j;
			label.appendChild(radio);
			const span = document.createElement("span");
			span.textContent = opt;
			label.appendChild(span);
			optionsWrap.appendChild(label);
		});

		block.appendChild(optionsWrap);

		const feedback = document.createElement("p");
		feedback.className = "feedback";
		feedback.id = "feedback-" + i;
		block.appendChild(feedback);

		container.appendChild(block);
	});

	function answeredCount() {
		let count = 0;
		QUESTIONS.forEach((item, i) => {
			if (document.querySelector('input[name="question' + i + '"]:checked')) {
				count++;
			}
		});
		return count;
	}

	function updateSubmitButton() {
		document.getElementById("submit").textContent =
			"Submit (" + answeredCount() + " of " + QUESTIONS.length + " answered)";
	}

	container.addEventListener("change", updateSubmitButton);
	updateSubmitButton();

	function tierFor(score, total) {
		const ratio = score / total;
		if (score === total) {
			return {
				text: "Perfect — all " + total + " correct. Clean full score.",
				bg: "var(--success-bg)",
				color: "var(--success-text)"
			};
		}
		if (ratio >= 0.8) {
			return {
				text: "Very strong — just some polishing left.",
				bg: "var(--success-bg)",
				color: "var(--success-text)"
			};
		}
		if (ratio >= 0.5) {
			return {
				text: "Solid foundation, a few gaps remain.",
				bg: "var(--warning-bg)",
				color: "var(--warning-text)"
			};
		}
		return {
			text: "This is worth some targeted practice.",
			bg: "var(--danger-bg)",
			color: "var(--danger-text)"
		};
	}

	document.getElementById("submit").addEventListener("click", () => {
		let score = 0;

		QUESTIONS.forEach((item, i) => {
			const selected = document.querySelector('input[name="question' + i + '"]:checked');
			const feedback = document.getElementById("feedback-" + i);
			const block = document.getElementById("block-" + i);
			feedback.style.display = "block";

			if (selected && parseInt(selected.value) === item.correct) {
				score++;
				block.classList.add("question-correct");
				feedback.textContent = "Correct";
				feedback.style.color = "var(--success-text)";
			} else if (selected) {
				block.classList.add("question-incorrect");
				document.getElementById("label-" + i + "-" + item.correct).classList.add("option-canary");
				feedback.textContent = "Incorrect. The correct answer was: " + item.options[item.correct];
				feedback.style.color = "var(--danger-text)";
			} else {
				block.classList.add("question-incorrect");
				document.getElementById("label-" + i + "-" + item.correct).classList.add("option-canary");
				feedback.textContent = "Not answered. The correct answer was: " + item.options[item.correct];
				feedback.style.color = "var(--text-muted)";
			}

			// Lock selection after grading
			document.querySelectorAll('input[name="question' + i + '"]').forEach((radio) => {
				radio.disabled = true;
			});
		});

		const box = document.getElementById("result-box");
		const tier = tierFor(score, QUESTIONS.length);

		box.style.display = "flex";
		box.style.background = tier.bg;
		document.getElementById("score-value").textContent = score + " of " + QUESTIONS.length;
		document.getElementById("score-value").style.color = tier.color;
		document.getElementById("tier-text").textContent = tier.text;
		document.getElementById("tier-text").style.color = tier.color;

		// Button is redundant after grading
		document.getElementById("submit").style.display = "none";

		// scroll down to the result
		window.scrollTo({ top: document.body.scrollHeight, behavior: "smooth" });
	});
</script>

</body>
</html>
```

## Deprecated names
The account also still has two old, deprecated skill entries that
cannot be deleted: `sokratische-lehre` and `interaktives-mc-quiz`. Both
carry a deprecation notice pointing here and to
`heuristic-teaching-dialogue-aviva`. Ignore them if they ever surface.

## Orchestration
For deciding which topic needs a quiz next across the whole project, or
running a full session (teach → quiz → update progress) end to end
rather than just this one step, see `learning-project-orchestrator`.
This skill only owns quiz generation, verification, and the file
pipeline.

## Versioning
This SKILL.md carries a `version` field in its frontmatter (currently
2.4). The parent folder name `TEACHING SKILLS` deliberately carries no
version number — versioning is skill-level only (filename +
frontmatter), never folder-level. A shared folder-level version number
would have been misleading anyway: if one skill catches up to the
other, the highest value stays the same and the folder name shows no
change — pure noise with no information content.

**Standing rule:** every substantive change to this skill bumps
`version` in the frontmatter (e.g. 2.0 → 2.1). Immediately after:
1. Rename the file to `SKILL-interactive-mc-quiz-<new-version>.md` (via
	`mv`, don't keep the old version file).
2. Update the live skill via `save_skill` (`overwrite: true`) in sync,
	with the same version number.
3. Append a one-line entry to the Changelog section of
	`aviva-learning-kit/README.md` (project root), matching this
	skill's own changelog entry below. This is the shared, cross-skill
	changelog convention — see "Skill versioning" under
	`learning-project-orchestrator`'s "Project-wide conventions" for the
	full rule shared by all three skills in this project.

The `TEACHING SKILLS/` folder stays untouched. Pure typo fixes with no
behavior change can skip a version bump at your discretion — when in
doubt, bump anyway.

**Changelog:**
- 2.4: Added the standing rule (step 3 above) to also append every
	version bump's changelog line to `aviva-learning-kit/README.md`,
	so there's one shared, human-readable changelog across all three
	project skills instead of three separate internal ones nobody
	reads together.
- 2.3: Added the "Hide-until-ready" requirement for live widgets: the
	question blocks now start `display: none` in the raw HTML (hidden
	from the first streamed chunk) with a plain animated loading bar
	shown in their place, and only the post-stream `<script>` swaps
	visibility and wires up the listeners. Fixes a real bug the user
	hit — clicking answers while the widget was still streaming in
	silently lost them, because the widget appears to fully re-render on
	each chunk until the script attaches. This is a markup-order fix,
	not a JS-timing fix, since nothing inside the widget can control the
	host's streaming/re-render behavior.
- 2.2: Added live in-chat delivery alongside the file: the same
	verified `QUESTIONS` data is now also rendered as a `show_widget`
	interactive widget whose submit handler calls `sendPrompt` with a
	fixed-format result string (`Quiz result — <Topic>: X of Y
	correct.`), so scores auto-report into the conversation instead of
	requiring the user to type them back. Confirmed via live test that
	`read_widget_context` cannot read a plain file-preview panel's state
	(no live connection exists there), but `sendPrompt` reliably pushes
	widget state into the chat. The file in `QUIZZES/` is unchanged in
	purpose — still the portable/offline/shareable copy — the widget is
	additive for the in-chat case. The auto-report message counts as a
	completion signal under "Archiving," same as a manually typed score.
- 2.1: "Regular check" now points to the concrete `PROGRESS.md` file
	(project root) instead of vague "progress notes" language. Added a
	pointer to the new `learning-project-orchestrator` skill, which owns
	cross-topic sequencing and the multi-skill session loop. (The
	"Deprecated names" section was added to the project-local file
	during the 2.0 edit but never bumped or pushed live — folded into
	this version instead.)
- 2.0: Renamed from `interaktives-mc-quiz` to `interactive-mc-quiz`.
	Full file translated to English (documentation, template UI strings,
	internal variable/class names). Filename convention prefix changed
	from `Quizz –` to `Quiz –` going forward (old files keep their
	historical spelling — match both when checking `QUIZZES/`). Produced
	quiz question/answer content still follows the language of the study
	material (German for this project), independent of the skill's own
	English documentation.
- 1.3: Template (`quiz-template.html`) embedded directly in this
	SKILL.md instead of an external `assets/` file — lets this skill be
	shared as a single markdown file without zipping a folder. Also
	removed folder-level versioning (`TEACHING SKILLS <x.y>`) again —
	brought no informational value and just caused unnecessary renaming.
- 1.2: Retroactive correction — this skill had gone through more
	revision rounds since initial creation than the teaching-dialogue
	skill (color-coding reworked twice, button behavior changed twice,
	architecture switched from JSON cache to file pipeline), hence a
	version jump past that skill's 1.0 (unchanged since creation).
- 1.1: First versioned release (file-pipeline workflow with
	`QUIZZES/`/`ARCHIVE/`, format spec with question-block coloring).
