---
name: learn
description: Adaptive learning system — ingests source documents, builds a prerequisite knowledge graph, generates Bloom's-aligned questions with rubrics, and tutors via Socratic dialogue with spaced repetition
---

# /learn — Adaptive Learning System

## Overview

This skill turns source documents into an adaptive, interactive learning experience grounded in evidence-based learning science: retrieval practice, spaced repetition, Socratic dialogue, and mastery-based progression via a prerequisite knowledge graph.

**A source file is always required.** Never generate learning content from your own knowledge — the source document is the ground truth for all questions, rubrics, and feedback.

---

## Commands

| Invocation | Behavior |
|---|---|
| `/learn` | Start a new topic or continue an existing one |
| `/learn review` | Run a spaced repetition review session |
| `/learn status` | Display the knowledge map for the current topic |
| `/learn add <file>` | Add a source document to the current topic |

---

## State Location

All state is stored as JSON files under a topic directory:

```
~/.claude/learn/<topic-slug>/
├── meta.json            ← topic name, documents, timestamps
├── dag.json             ← concept nodes and prerequisite edges
├── questions.json       ← question bank with rubrics and hints
├── student-model.json   ← BKT mastery state per concept
└── srs-schedule.json    ← SM-2 spaced repetition schedule
```

**Global vs. local mode:** Use `~/.claude/learn/` by default. If a `.learn/` directory exists in the current working directory, use it instead (project-local mode).

---

## Step 1 — Entry

When `/learn` is invoked:

1. Determine state root: if `.learn/` exists in cwd, use it; otherwise use `~/.claude/learn/`
2. List existing topic directories. If any exist, ask: "Continue an existing topic, or start a new one?"
3. **New topic:**
   - Ask for a topic name → derive `topic-slug` (lowercase, hyphens)
   - Ask for one or more source document file paths
   - Run **Step 2 — Ingest Pipeline**
4. **Existing topic:** load state files and run **Step 3 — Study Session**

---

## Step 2 — Ingest Pipeline

Run when processing documents for a new topic or when adding a source via `/learn add`.

### 2a. Read and segment the document

Read the full document. Identify discrete, learnable concepts — these become nodes in the DAG. A concept is a unit that can be independently mastered (e.g., "gradient descent", "the Pythagorean theorem", "TCP three-way handshake"). Aim for 5–30 concepts per document depending on length and density.

For each concept record:
- `name`: short identifier
- `description`: 1–2 sentence summary drawn directly from the source
- `difficulty`: float 0.1–1.0 (0.1 = trivial, 1.0 = very hard); estimate from conceptual density and abstraction level
- `source_passages`: verbatim excerpts from the document that define or explain the concept
- `sources`: list of document filenames that mention this concept

### 2b. Build the prerequisite DAG

For each concept, identify which other concepts must be mastered before it is learnable. Record as directed edges: `prerequisite_id → concept_id`.

If a required prerequisite concept is not covered by any source document, mark the concept `status: "blocked"` and add it to `missing_prerequisites` in `dag.json`.

After building the DAG, handle each missing prerequisite by asking the learner:

> "To learn **[concept]**, you'll need **[prerequisite]**, which isn't in your sources. Do you already know this, or would you like to provide a source document for it?"

- **Learner knows it:** set `p_learn: 0.9` in the student model for that concept (self-reported mastery); mark `status: "available"`
- **Learner provides a file:** ingest the file, add its concepts to the DAG, resolve the blocked status
- **Neither:** leave as `status: "blocked"` — this branch is unreachable until resolved

### 2c. Detect conflicts across documents

When the same concept appears in multiple source documents:

- **Agreement:** merge into one node; list all sources in the `sources` field
- **Conflict:** mark `has_conflict: true`; record both claims verbatim with their source document in `conflict_details`. Agreed-upon aspects are taught normally first. The conflict surfaces later as a research task at the Evaluate Bloom's level (see Step 3h).

### 2d. Generate questions and rubrics

For each concept, generate questions spanning all 6 Bloom's levels. Use only the concept's `source_passages` as the basis — do not introduce information not present in the source.

**Bloom's level coverage:**

| Level | Question types | Format |
|---|---|---|
| Remember | Define, list, identify — factual recall | MCQ or open |
| Understand | Explain in own words, summarize, compare | MCQ or open |
| Apply | Use this concept to solve X, apply to scenario Y | Open response only |
| Analyze | Identify components, explain relationships, find evidence | Open response only |
| Evaluate | Critique, compare approaches, assess validity | Open response only |
| Create | Design, propose, construct, synthesize | Open response only |

Generate at least 2 questions per Bloom's level per concept. Target 3 for Apply and above.

**For each open-response question, generate a rubric with 3–5 criteria:**

Each criterion must test one specific, verifiable thing present in the source passage. Avoid vague criteria like "shows understanding." Criteria should be pass/fail or point-scored against what the source actually says.

```json
{
  "criteria": [
    {
      "name": "criterion name",
      "description": "what a complete answer must include, with reference to the source",
      "points": 1
    }
  ],
  "total_points": N,
  "passing_score": "ceil(N * 0.7)"
}
```

**For each question, generate 3 tiered hints:**

- **Hint 1 (directional):** points toward the relevant concept or passage without revealing the answer
- **Hint 2 (procedural):** describes the reasoning process or approach to use
- **Hint 3 (near-complete):** gives almost the full answer structure, leaving only the final conclusion to the learner

### 2e. Initialize state files

After ingest, write the following files to the topic directory.

**meta.json:**
```json
{
  "topic": "Human-readable topic name",
  "slug": "topic-slug",
  "documents": ["path/to/file1.pdf"],
  "created": "ISO timestamp",
  "last_session": "ISO timestamp"
}
```

**dag.json:**
```json
{
  "concepts": {
    "<concept-id>": {
      "name": "",
      "description": "",
      "difficulty": 0.5,
      "sources": [],
      "source_passages": [],
      "prerequisites": [],
      "status": "available",
      "has_conflict": false,
      "conflict_details": null
    }
  },
  "missing_prerequisites": []
}
```

**questions.json:**
```json
{
  "<question-id>": {
    "concept_id": "",
    "bloom_level": "remember | understand | apply | analyze | evaluate | create",
    "question_type": "open_response | mcq",
    "question": "",
    "source_passage": "",
    "rubric": {
      "criteria": [],
      "total_points": 0,
      "passing_score": 0
    },
    "hints": ["hint1", "hint2", "hint3"],
    "mcq_options": null
  }
}
```

**student-model.json:**
```json
{
  "concepts": {
    "<concept-id>": {
      "p_learn": 0.0,
      "p_transit": 0.1,
      "p_slip": 0.1,
      "p_guess": 0.2,
      "attempts": 0,
      "correct": 0,
      "time_started": null,
      "mastered_at": null
    }
  },
  "global_learning_rate": 0.1,
  "session_count": 0
}
```

**srs-schedule.json:**
```json
{
  "<concept-id>": {
    "ease_factor": 2.5,
    "interval": 1,
    "repetitions": 0,
    "next_review": null,
    "last_reviewed": null
  }
}
```

After writing all files, proceed immediately to Step 3.

---

## Step 3 — Study Session

### 3a. Select the next concept

1. Load `dag.json` and `student-model.json`
2. Find the **outer fringe**: concepts where:
   - All prerequisites have `p_learn >= 0.95`
   - The concept itself has `p_learn < 0.95`
   - `status` is not `"blocked"`
3. From the outer fringe, target a ~70% success rate for the learner. Prefer concepts whose `difficulty` is close to `global_learning_rate * 0.7`. If the learner has been failing recently, bias toward easier concepts; if succeeding easily, bias toward harder ones.

### 3b. Select a question

Filter `questions.json` for questions belonging to the selected concept. Choose Bloom's level based on current `p_learn`:

| p_learn range | Bloom's levels to use |
|---|---|
| 0.0 – 0.4 | Remember, Understand |
| 0.4 – 0.7 | Apply, Analyze |
| 0.7 – 0.95 | Evaluate, Create |

Do not repeat a question the learner has answered correctly in the current session.

### 3c. Ask the question — Socratic mode

Present the question. Do not offer hints unless the learner asks or answers incorrectly.

**Hint escalation protocol:**
1. First incorrect answer or explicit "I don't know": offer Hint 1
2. Second incorrect answer: offer Hint 2
3. Third incorrect answer: offer Hint 3
4. Fourth incorrect answer: reveal the answer with a full explanation citing the source passage; record as incorrect for BKT

**Never give the answer before Hint 3 is exhausted.**

**If the learner mentions a concept not in the DAG mid-session:**
Note it. After the current question resolves, ask: "You mentioned [concept] — would you like to add it as a prerequisite? If so, provide a source document for it." If yes, run ingest on the new file and update the DAG.

### 3d. Grade the response

**Open-response questions:**
1. Load the question's `rubric` and `source_passage`
2. Evaluate the learner's response against each rubric criterion strictly — a criterion is met only if the learner's answer includes the substance described, as verifiable against the source passage
3. Sum points; compare to `passing_score`
4. For each criterion not met, give specific feedback explaining what was missing and cite the relevant part of the source passage

**MCQ:**
Correct or incorrect only; explain why the correct answer is right, citing the source passage.

### 3e. Update BKT

After each response, update `p_learn` for the concept in `student-model.json`.

**If correct:**
```
p_learn_posterior = (p_learn * (1 - p_slip)) /
                    ((p_learn * (1 - p_slip)) + ((1 - p_learn) * p_guess))
p_learn_new = p_learn_posterior + (1 - p_learn_posterior) * p_transit
```

**If incorrect:**
```
p_learn_posterior = (p_learn * p_slip) /
                    ((p_learn * p_slip) + ((1 - p_learn) * (1 - p_guess)))
p_learn_new = p_learn_posterior + (1 - p_learn_posterior) * p_transit
```

Increment `attempts`. If correct, increment `correct`.

**Mastery threshold — `p_learn >= 0.95`:**
When a concept crosses this threshold:
1. Set `mastered_at` to current ISO timestamp
2. Recompute `global_learning_rate` as the mean `p_transit` across all mastered concepts
3. Initialize or update the concept's SRS entry (Step 3f)
4. Notify the learner: "You've mastered **[concept]**."

### 3f. Update spaced repetition schedule (SM-2)

When a concept is mastered or reviewed, update its entry in `srs-schedule.json`.

**Map performance to SM-2 quality score (q):**

| Performance | q |
|---|---|
| Did not answer / gave up | 1 |
| Answered with Hint 3 | 2 |
| Passed narrowly (score at passing_score) | 3 |
| Passed well | 4 |
| Perfect score, no hints | 5 |

**SM-2 update:**
```
if q >= 3:
    if repetitions == 0: interval = 1
    elif repetitions == 1: interval = 6
    else: interval = round(interval * ease_factor)
    repetitions += 1
    ease_factor = ease_factor + 0.1 - (5 - q) * (0.08 + (5 - q) * 0.02)
    ease_factor = max(1.3, ease_factor)
else:
    repetitions = 0
    interval = 1

next_review = today + interval days
last_reviewed = today
```

### 3g. Check struggle signal

After each question attempt, evaluate whether the learner is progressing anomalously slowly on the current concept:

1. `expected_attempts = concept.difficulty / student.global_learning_rate`
2. If `attempts > expected_attempts * 2` and `p_learn < 0.95`:
   - Ask: "You're finding **[concept]** harder than expected. Is there a related concept you're unfamiliar with that might be a prerequisite? Or would you like to add another source?"
   - If the learner names a concept: add it as a new prerequisite node, ask for a source document
   - If they provide a file: ingest it and wire the new concepts as prerequisites
   - Update `dag.json` accordingly

### 3h. Handle conflicts

When presenting a concept with `has_conflict: true`:
1. Teach only the agreed-upon aspects first — standard question flow above
2. After the concept is mastered, surface the conflict:
   > "Sources disagree on **[specific point]**. [Source A] says: *[claim]*. [Source B] says: *[claim]*. Research this disagreement and come back with your conclusion."
3. Add an Evaluate-level question about the conflict to the session queue:
   > "Having reviewed both sources, which claim do you find better supported, and why?"

---

## Step 4 — Review Session (`/learn review`)

1. Load `srs-schedule.json`
2. Filter for concepts where `next_review <= today`
3. For each due concept, select a question — prefer Evaluate or Create level for already-mastered concepts
4. Run the Socratic questioning and grading flow (Steps 3c–3f)
5. Update `srs-schedule.json` after each item

If no items are due:
> "Nothing due for review. Next review: **[earliest next_review date]**."

---

## Step 5 — Status (`/learn status`)

Display the knowledge map:

```
Topic: [topic name]
Sources: [document list]

Knowledge Map:
✓  [concept]   mastered — next review [date]
◉  [concept]   in progress — p_learn: 0.67
○  [concept]   ready to learn (outer fringe)
✗  [concept]   blocked — missing prerequisite: [name]
⚠  [concept]   conflict flagged for research

Progress: X / Y concepts mastered
Next SRS review due: [date or "none scheduled"]
```

---

## Invariants — Never Violate

1. **Never give an answer directly** — exhaust all three hints first
2. **Never use your own knowledge as a source** — all questions, rubrics, and feedback must be grounded in the source passages
3. **Never advance past a blocked concept** — prerequisites must be resolved before the branch is unlocked
4. **Never use vague rubric scoring** — every criterion must be verifiable against the source passage
5. **Always cite the source passage in feedback** — the learner must be able to verify every correction
6. **Never repeat a correctly-answered question** within the same session
