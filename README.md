# /learn — Adaptive Learning Skill for Claude Code

A Claude Code skill that turns any document into an adaptive, interactive learning experience. Grounded in evidence-based learning science: retrieval practice, spaced repetition, Socratic dialogue, and mastery-based progression.

**A source file is always required.** The document you provide is the ground truth — the skill never generates learning content from Claude's own knowledge.

---

## Installation

1. Clone this repository:
   ```bash
   git clone <repo-url>
   cd learn
   ```

2. Copy the skill into your Claude Code skills directory:
   ```bash
   cp .claude/skills/learn.md ~/.claude/skills/learn.md
   ```

3. Restart Claude Code (or open a new session). The `/learn` command is now available globally.

**Project-local mode:** If you want learning state scoped to a specific project, create a `.learn/` directory in that project's root. The skill will detect it and store state there instead of `~/.claude/learn/`.

---

## Usage

### Start a new topic
```
/learn
```
Claude will ask for a topic name and one or more source documents. On first run it ingests the documents — extracting concepts, building a prerequisite knowledge graph, and generating questions with rubrics at all 6 Bloom's levels. Then it starts a study session immediately.

### Add a source to an existing topic
```
/learn add path/to/another-source.pdf
```
Ingests the new document and merges its concepts into the existing knowledge graph. If the new source conflicts with an existing one, agreed-upon content is taught first and the disagreement is surfaced as a research task.

### Review spaced repetition due items
```
/learn review
```
Runs a review session for any concepts due today based on the SM-2 spaced repetition schedule.

### Check your knowledge map
```
/learn status
```
Displays which concepts are mastered, in progress, ready to learn, or blocked by missing prerequisites.

---

## How it works

### Ingest (runs once per document)

When you provide a source file, the skill:

1. **Extracts concepts** and assigns each a difficulty score
2. **Builds a prerequisite DAG** — a directed graph where edges mean "must know A before learning B"
3. **Handles gaps** — if a prerequisite isn't covered by your sources, it asks whether you already know it (self-report) or want to provide a source for it
4. **Detects conflicts** between sources — teaches the consensus first, surfaces the disagreement as a research task after you've mastered the concept
5. **Generates questions** at all 6 Bloom's levels (Remember → Create) per concept
6. **Generates a rubric** for each open-response question — specific, verifiable criteria tied to the source passage

### Study session

The skill selects questions from your **outer fringe** — concepts whose prerequisites you've mastered but you haven't mastered yet. It targets a ~70% success rate to keep you in the flow channel.

Questions are asked Socratically: the skill never gives you the answer directly. If you're stuck, you can ask for a hint. There are three hint levels (directional → procedural → near-complete) before the answer is revealed.

Your responses are graded against the question's rubric, criterion by criterion, with the source passage cited in feedback.

### Student model

Mastery is tracked using **Bayesian Knowledge Tracing (BKT)** — a per-concept probability of mastery that updates after each response. You advance to new concepts when `P(mastered) ≥ 0.95`.

If you're progressing slower than expected on a concept (accounting for its difficulty and your personal learning rate), the skill asks whether a missing prerequisite might explain it and offers to add one.

### Spaced repetition

Mastered concepts are scheduled for review using the **SM-2 algorithm**. Intervals expand as your mastery strengthens — 1 day, 6 days, then longer based on how well you perform at each review. Run `/learn review` to work through due items.

---

## State files

All state is stored as plain JSON in `~/.claude/learn/<topic-slug>/`:

| File | Contents |
|---|---|
| `meta.json` | Topic name, source documents, timestamps |
| `dag.json` | Concept nodes, prerequisite edges, conflict flags |
| `questions.json` | Question bank with rubrics and hints |
| `student-model.json` | BKT mastery state per concept |
| `srs-schedule.json` | SM-2 spaced repetition schedule |

These files are human-readable and editable if you need to correct something.

---

## Design principles

This skill is built on peer-reviewed learning science research:

- **Retrieval over re-reading** — every interaction is test-first (Roediger & Karpicke, 2006; d≈0.5)
- **Bloom's taxonomy** — questions span all 6 cognitive levels, not just recall (Anderson & Krathwohl, 2001)
- **Socratic by default** — tiered hints, never direct answers (Lepper et al., 1993; Aleven et al., 2004)
- **Spaced repetition** — SM-2 scheduling before the forgetting curve hits (Wozniak, 1990; Ebbinghaus, 1885)
- **Mastery-based progression** — outer fringe targeting via Knowledge Space Theory (Falmagne & Doignon, 1985)
- **Adaptive difficulty** — ~70% success rate target, the ZPD / flow channel (Vygotsky; Csikszentmihalyi, 1990)
- **Bayesian Knowledge Tracing** — probabilistic mastery model per concept (Corbett & Anderson, 1994)

Full research notes are in `/research/`.

---

## Future todos

- IRT (Item Response Theory) for per-question difficulty calibration alongside BKT
- Hint abuse detection (rapid sequential hint-clicking pattern)
- Hattie & Timperley self-regulation feedback level ("what should I do differently next time")
- FSRS algorithm as an upgrade path from SM-2
