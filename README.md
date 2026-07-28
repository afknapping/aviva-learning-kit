# AVIVA Learning Kit

A small, portable system of three Claude skills plus one progress file
for running self-directed learning: heuristic (Socratic-style) teaching
structured by the AVIVA didactic model, an interactive multiple-choice
quiz generator, and an orchestrator that ties the two together and
tracks progress over time.

## What's in this folder

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

## Use

To set up a new learning project with this kit, point
`learning-project-orchestrator`'s "Bootstrap" section at the target
folder — it creates the `TEACHING SKILLS/`, `QUIZZES/`, `ARCHIVE/`
structure, installs the two content skills, and creates a fresh
`PROGRESS.md` from the template in that same file.

## Sources

- Städeli, C., Grassi, A., Rhiner, K., Obrist, W. (2010). *Kompetenzorientiert
	unterrichten: das AVIVA-Modell.* hep Verlag, Bern. —
	[hep-verlag.ch/das-aviva-modell](https://www.hep-verlag.ch/das-aviva-modell)
- [AVIVA (didaktisches Muster) – Wikipedia](https://de.wikipedia.org/wiki/AVIVA_(didaktisches_Muster))
- Hessisches Landesamt für Fachschulen (HLFS). *Kompetenzorientiert
	ausbilden – das AVIVA-Modell* (PDF) —
	[hlfs.hessen.de](https://hlfs.hessen.de/sites/hlfs.hessen.de/files/2024-02/fb-ausbilder_lernu_9_aviva_modell_01.00.00.pdf)
