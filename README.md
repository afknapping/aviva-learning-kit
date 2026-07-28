# AVIVA Learning Kit

A small, portable system of three Claude skills plus one progress file
for running self-directed learning: heuristic (Socratic-style) teaching
structured by the AVIVA didactic model, an interactive multiple-choice
quiz generator, and an orchestrator that ties the two together and
tracks progress over time.

<details>
  <summary>What's in this folder</summary>

- `SKILL-heuristic-teaching-dialogue-aviva-2.1.md` — teaches new
	material through guided questions rather than explanation, structured
	around the AVIVA model's five phases (Arrival, Activate prior
	knowledge, Inform, Process, Evaluate).
- `SKILL-interactive-mc-quiz-2.1.md` — generates a self-contained,
	single-file HTML multiple-choice quiz for any topic just covered,
	with a file pipeline so a fresh test is always ready to go.
- `SKILL-learning-project-orchestrator-1.0.md` — bootstraps a new
	learning project (folder skeleton, progress file, the two skills
	above) and runs the ongoing per-session loop in an existing one:
	pick topic → teach → quiz → update progress.
- `PROGRESS.md` — a live example of the progress-tracking file the
	orchestrator creates and the other two skills update: one section
	per topic, with last-worked date, confidence, and open gaps.

</details>

## Install (once per Claude account)

Skills live per Claude account, not per repo — cloning or downloading
this folder doesn't make them available on its own. Add them once:

1. Open a Cowork/Claude chat and attach the three `SKILL-*.md` files
	(or connect this whole folder), then ask Claude to save them as
	skills — it has a `save_skill` tool for exactly this.
2. Alternatively, zip each skill's file into a `.skill` archive and
	share it in chat — Cowork renders a one-click "Save skill" install
	button for files of that type.

Do this once; after that, all three skills are available in every
project on that account, the same as any other Claude skill.

## Use (per new learning project)

Once the three skills are installed on your account:

1. Connect or select the folder you want as the new project's root.
2. In that project's chat, just say what you want to learn — e.g. "set
	up a new learning project here for Spanish vocabulary."
3. `learning-project-orchestrator`'s description matches that request
	and triggers automatically. It creates the `TEACHING SKILLS/`,
	`QUIZZES/`, `ARCHIVE/` structure, writes a fresh `PROGRESS.md` from
	its template, and drops project-local copies of the two content
	skills into `TEACHING SKILLS/`.

If it doesn't trigger on its own, name it directly: "use
`learning-project-orchestrator` to set this up."

## Sources

- Städeli, C., Grassi, A., Rhiner, K., Obrist, W. (2010). *Kompetenzorientiert
	unterrichten: das AVIVA-Modell.* hep Verlag, Bern. —
	[hep-verlag.ch/das-aviva-modell](https://www.hep-verlag.ch/das-aviva-modell)
- [AVIVA (didaktisches Muster) – Wikipedia](https://de.wikipedia.org/wiki/AVIVA_(didaktisches_Muster))
- Hessisches Landesamt für Fachschulen (HLFS). *Kompetenzorientiert
	ausbilden – das AVIVA-Modell* (PDF) —
	[hlfs.hessen.de](https://hlfs.hessen.de/sites/hlfs.hessen.de/files/2024-02/fb-ausbilder_lernu_9_aviva_modell_01.00.00.pdf)
