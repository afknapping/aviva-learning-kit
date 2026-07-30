---
name: interactive-mc-quiz
description: Creates an interactive multiple-choice quiz for a knowledge check. Use this skill whenever the LERNEN project needs a test, quiz, exam, self-check, or knowledge probe on a topic just covered — even if "multiple choice" isn't said explicitly. Applies to any learning topic in this project, not just retraining prep.
version: 2.7
---

# Interactive Multiple-Choice Quiz

## When to trigger
- The user asks for a test, quiz, probe, or practice questions on a topic
	just discussed.
- The user says "quiz me", "multiple choice", "grade me", or similar.

## Core principle: verified quiz-data banks, combined on demand
Waiting for a fresh generation costs time the user doesn't want to lose
on a spontaneous test. So, wherever possible, every already-covered
chapter/module ("unit") of a topic already has its own small bank of
verified questions sitting ready in `MATERIALS/<Topic>/quiz-data/` (see
"Module quiz-data banks" below) — prepared in the background as soon as
that unit is known, not generated fresh each time a quiz is requested.
Delivering a quiz, whether for one unit or several combined, is then
just: pull the relevant unit banks, combine and reshuffle them, render.
No subagent call and no waiting is needed at delivery time as long as
every requested unit already has a bank; only a genuinely new,
never-before-banked unit falls back to live generation. Right after any
delivery, the units that were actually used get their banks quietly
refreshed in the background, so repeat exposure to the same questions
stays rare.

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

**Widget is the delivery; the file is silent backup.** Render the
widget immediately — don't wait on the file, don't ask whether the
user wants to see it, and don't call `present_files` on the quiz file
by default. The file still gets saved into `QUIZZES/` (it's the
portable/offline copy and the archiving anchor), but surfacing it via
`present_files` every time is redundant noise once a widget already
exists — only do that if the user specifically asks to see the file,
open it outside the chat, or share/download it. Anticipate this
ahead of time rather than blocking on a question: if a quiz is being
delivered at all, default straight to rendering the widget.

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
- `QUIZZES/` — holds exactly one ready-to-take test file per topic (or
	per unit combination currently in play). Lives in the project root.
- `MATERIALS/<Topic>/quiz-data/` — one verified question bank per
	chapter/module ("unit") of that topic, named to match the
	corresponding file in `MATERIALS/<Topic>/`. See "Module quiz-data
	banks" below. Lives inside `MATERIALS/`, whose overall shape
	`learning-project-orchestrator` owns.
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

## Module quiz-data banks

Each already-covered chapter/module of a topic ("unit") has its own
small, verified bank of questions, prepared ahead of time rather than
regenerated every time a quiz touches that unit.

**Location and filename.** `MATERIALS/<Topic>/quiz-data/<Unit>.json`,
where `<Unit>` matches the stem of that unit's file in
`MATERIALS/<Topic>/` (e.g. `MATERIALS/GarageBand and Trip Hop/Module 3
Beat and Drums.md` ↔ `MATERIALS/GarageBand and Trip Hop/quiz-data/Module
3 Beat and Drums.json`). `<Topic>` follows the same convention as
`MATERIALS/<Topic>/` itself — see `learning-project-orchestrator`'s
"Source material persistence".

**Schema:**
```json
{
	"topic": "GarageBand and Trip Hop",
	"unit": "Module 3 Beat and Drums",
	"material_file": "MATERIALS/GarageBand and Trip Hop/Module 3 Beat and Drums.md",
	"generated_at": "2026-07-29T15:00:00+02:00",
	"verified": true,
	"questions": [
		{
			"question": "...",
			"options": ["...", "...", "...", "..."],
			"correct": 0
		}
	]
}
```
5–15 questions per unit depending on scope, same shape as delivered
questions (`question`/`options`/`correct`), already run through
Verification below before being saved here.

**Creating or refreshing a bank** (new unit, or refreshing after use —
see below): derive questions from that unit's `MATERIALS/<Topic>/
<Unit>.md` file itself, not from conversation memory — the file is the
source of truth, which also makes this work for units taught in a past
session. Run the draft through Verification below, then save. Never
save an unverified bank.

**Refresh after use.** Once a unit's bank has been included in any
delivered quiz (solo or combined), regenerate a fresh bank for that
specific unit right after delivery — same no-repeat principle as
before, just scoped to the unit actually used instead of the whole
topic. Cheaper than before: a combined quiz across four units now only
refreshes those four units' banks, not a whole re-derivation of
everything covered in the topic.

**Bulk creation must not block the conversation.** Whenever more than
one or two units need a bank at once — e.g. an initial backfill for an
already-covered topic, or catching up after several sessions — issue
one `Agent`-tool call per missing unit and batch all of them into a
single message so they run in parallel, never sequentially. A bulk
backfill isn't gating any specific delivery, so there's no reason to
make the user wait through a queue of subagent calls one at a time.
Each unit's generation and its Verification is a self-contained
subagent hand-off (it needs the material file path, not conversation
context), so parallelizing across units is safe.

## Filename (mandatory)
`Quiz – <Topic> <Timestamp>.html`, saved in `QUIZZES/`.
- `<Topic>`: plain-text topic name (e.g. "Mathematik", "Ohmsches Gesetz",
	"Schaltsymbole") — no slugs in the filename. Keep the topic name in
	whatever language the study material uses.
- `<Timestamp>`: format `YYYY-MM-DD_HH-MM` (sortable, no colons for
	filesystem compatibility), time of generation.
- Example: `Quiz – Mathematik 2026-07-28_08-26.html`
- When a delivered quiz combines more than one unit, append the unit
	range in parentheses before the timestamp: `Quiz – <Topic>
	(<Unit range>) <Timestamp>.html`, e.g. `Quiz – GarageBand and Trip
	Hop (Modules 1–4) 2026-07-29_15-11.html`. A quiz covering only a
	single unit keeps the plain `Quiz – <Topic> <Timestamp>.html` form,
	no unit suffix.
- Note: files generated before this version used the spelling
	`Quizz –` (double z) — a historical artifact of the earlier
	German-only version. Existing files are not renamed retroactively;
	only new files use the `Quiz –` spelling. When checking `QUIZZES/`
	for an existing file (step 2 below), match both spellings.

## Flow

1. **Determine the topic and the unit(s) in scope.** Default to
	whichever unit(s) were just discussed; if the user names an explicit
	range ("quiz me on modules 1 through 4", "everything so far"), scope
	to that instead.
2. **Check each in-scope unit's bank** in `MATERIALS/<Topic>/quiz-data/`
	(see "Module quiz-data banks" above).
	- Bank exists → reuse its verified `questions` directly.
	- Bank missing (brand-new unit, or one that slipped through) →
		derive questions from that unit's material now, run them through
		Verification below, and save the bank before continuing — this
		is the only case where delivery has to wait on live generation.
3. **Combine.** Concatenate the question lists of all in-scope units
	into one working list.
4. **Interleave.** Shuffle the order across units — not all of one
	unit's questions followed by all of another's, same "mixed order
	across sub-topics" principle as a single-unit quiz.
5. **Re-check the combined distribution.** Even though each unit's bank
	was already balanced on its own, recombining several can still skew
	the overall test — recount the `correct`-index distribution across
	the *combined* list (see "Distribution of the correct answer" under
	Verification) and reshuffle affected questions' option order if it
	drifts past the ~30% rule. This is plain counting/reshuffling, not a
	factual check, so it doesn't need a fresh subagent call — do it
	directly.
6. **Render and deliver.** Render the combined, rebalanced list as a
	live auto-scoring widget per "Live delivery" above — this is the
	delivery. Save the same list into `QUIZZES/` as a file too, per the
	naming convention (single- or multi-unit form) — a silent
	backup/offline copy, not surfaced via `present_files` by default
	(see "Widget is the delivery" above). Only mention the file
	explicitly if the test ends up taken outside the chat rather than
	via the widget (see "Archiving" below).
7. **Refresh what was used.** Right after delivery, for every unit that
	was actually included, regenerate and re-verify a fresh bank for
	that unit (see "Refresh after use" under "Module quiz-data banks"
	above) — this runs after the user already has their quiz, so it
	never delays delivery.

## Archiving (when a file disappears from QUIZZES/)

- **Test taken in chat ("inline"):** As soon as the user signals in
	conversation that a specific test is done — e.g. they state a
	result, say "done", ask for the next test on the same topic, or the
	live widget auto-reports via the `Quiz result — <Topic>: X of Y
	correct.` message described under "Live delivery" above — the file
	in question is moved from `QUIZZES/` to `ARCHIVE/` via `mv` (never
	delete; see the project convention "delete means archive", which
	also applies to Claude's own cleanup — never use `rm`). The unit
	bank(s) behind it get refreshed per "Refresh after use" above, so
	the next request — for this topic or any combination touching these
	units — assembles fresh from already-ready banks; there's no
	pre-rendered successor file to move in anymore.
- **Test taken outside the chat** (the user opens the HTML file
	themselves in the browser without mentioning it in chat): Claude has
	no way to detect this. The file stays put until the user moves it to
	`ARCHIVE` or deletes it by hand. Mention this briefly when delivering
	a file (see "Flow" above).
- Never remove a file from `QUIZZES/` without a concrete signal in
	conversation that it's done.

## Regular check: all units banked?

Whenever this skill or a learning topic in the LERNEN project comes up
(e.g. skill invocation, start of a new learning session, explicit
question), briefly cross-check:
- Which chapters/modules ("units") have already been covered, per
	`PROGRESS.md` (project root) and prior conversation?
- Does every one of those units have a verified bank in
	`MATERIALS/<Topic>/quiz-data/` (see "Module quiz-data banks" above)?
- If a unit is missing a bank, generate and verify one per "Module
	quiz-data banks" above before it's needed.
- If several units are missing at once (e.g. initial backfill for an
	already-covered topic), check with the user whether to catch up on
	all of them now, then do so via the parallel-batch approach under
	"Bulk creation must not block the conversation" above — never
	sequentially for a large batch.

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
	content stay unchanged. This applies per unit bank at creation time;
	when several units are later combined into one delivered quiz, the
	combined set gets re-checked too (see "Flow" above) — that recheck
	is plain counting, not a factual check, so it doesn't need a fresh
	subagent call.
f. Additionally check each question against "No structural tells"
	above — flag any question where the correct option is noticeably
	longer, more detailed, or structurally different from the three
	distractors, and rewrite the distractors (or trim the correct
	option) so length/structure give no hint.

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
- **No structural tells.** All 4 options for a question must be
	similar in length and sentence structure — the correct answer must
	not be identifiable simply because it's noticeably longer, more
	detailed, more specific, or grammatically different from the three
	distractors. If a correct answer naturally wants to be longer/more
	precise, either trim it or lengthen/sharpen the distractors to
	match, so length and structure carry no signal at all. This is
	checked again during verification (see below).
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
3. Save as a new file per the naming convention above in `QUIZZES/`.

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
2.7). The parent folder name `TEACHING SKILLS` deliberately carries no
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
- 2.7: Replaced the "one combined file regenerated per topic" pipeline
	with per-chapter/module quiz-data banks: each already-covered unit
	gets its own small, verified question bank in
	`MATERIALS/<Topic>/quiz-data/<Unit>.json`, prepared ahead of time.
	Delivering a quiz — whether for one unit or several combined —
	pulls the relevant banks, combines and reshuffles them, and
	renders, with no subagent wait as long as every requested unit is
	already banked. After delivery, only the units actually used get
	refreshed, not the whole topic, which is cheaper as topics grow.
	Bulk backfill for an already-covered topic now happens via parallel
	`Agent`-tool calls in a single message, one per missing unit,
	instead of a sequential queue. Drops the old "hold a rendered
	successor file internally until the current one is archived"
	complexity — nothing needs to be pre-rendered anymore since
	assembly from verified banks is already instant. Prompted directly
	by user feedback: "data for quizzes should be background prepared
	for all known modules. if quiz is taken for several modules at
	once, just combine the modules for the quiz" — paired with
	`learning-project-orchestrator`'s new per-topic `MATERIALS/`
	subfolders (1.6) so bank files can mirror material files 1:1.
- 2.6: Added a "No structural tells" rule to the format spec and
	verification checklist: all 4 options must be similar in length and
	sentence structure, so the correct answer can't be spotted just by
	being longer, more detailed, or grammatically different from the
	distractors. Direct user feedback: "the correct answer is too easy
	to spot. make sure wrong choices have similar length and sentence
	structure."
- 2.5: Widget is now the default delivery, full stop — stopped calling
	`present_files` on the quiz file whenever a widget is also rendered
	(redundant, and the user found it blocking/noisy), and stopped
	waiting on any confirmation before rendering the widget. The file
	in `QUIZZES/` is still saved every time as a silent backup/offline
	copy, just not surfaced unless asked for. Prompted directly by user
	feedback: "i don't need to see the file if you make a widget...
	i do not want to be blocked. anticipate where you can and
	prioritise widget – file is just backup."
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
