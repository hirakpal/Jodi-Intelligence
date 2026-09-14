# Jodi Intelligence — Project Documentation

## 1. What this project is

Jodi Intelligence is an evidence-grounded life-planning intelligence layer for matrimonial platforms and professional matchmakers.

It helps two people who have mutually expressed interest discuss practical expectations before commitment.

The product does **not** decide whether two people should marry. It does not produce a “marriage success prediction,” diagnose personality, or label anyone compatible/incompatible.

Instead, it does this:

```text
Two structured profiles
        ↓
Find areas of alignment and expectation gaps
        ↓
Show the exact answers behind each finding
        ↓
Create neutral discussion scenarios
        ↓
Optionally use Gemini to explain one validated gap
        ↓
Validate safety and evidence before showing the AI output
```

The business value is not “AI compatibility scoring.”

The business value is:

- fewer hidden expectation gaps
- more meaningful conversations after mutual interest
- useful matchmaker review material
- better progression from interest to informed decision
- auditable AI output instead of black-box advice

---

## 2. The problem Jodi Intelligence solves

Most matrimonial products focus on helping people discover profiles.

The difficult part often begins after mutual interest:

- How should career relocation be handled?
- When should children be discussed?
- How should family-care responsibilities work?
- How should major financial decisions be made?
- How should household work be divided?

People may agree on surface preferences while having different expectations about daily married life.

Jodi Intelligence gives those expectations a respectful structure.

---

## 3. Level 1 MVP scope

### Included

1. Five structured compatibility dimensions
2. Controlled answer choices
3. Importance levels from 1 to 5
4. Deterministic alignment and gap detection
5. Important-versus-low-priority distinction
6. Evidence-backed guided scenarios
7. Gemini-generated neutral wording
8. Safety, grounding, and evidence-link checks
9. Synthetic evaluation dataset
10. Error logging and version tracking
11. Colab Gradio prototype

### Not included yet

1. Real user data
2. Login or authentication
3. Database storage
4. Recommendation/ranking of potential matches
5. Marriage-success prediction
6. Conversation history or couple memory
7. RAG or retrieval
8. Matchmaker dashboard
9. Family graph
10. Voice/video analysis
11. Production deployment

---

## 4. Product safety boundary

Jodi Intelligence is a conversation-support system.

### It may say

- “You selected different approaches to this topic.”
- “This may be worth discussing.”
- “Here is a neutral question you can explore together.”
- “The explanation is based on these stated answers.”

### It must not say

- “You are incompatible.”
- “You should marry.”
- “You should not continue this match.”
- “This person is controlling.”
- “This couple will fail.”
- “This predicts marriage success.”

This boundary is essential because marriage is a high-impact life decision.

---

## 5. Architecture

```text
┌───────────────────────────────────────────────────────┐
│                 Structured Profile Input              │
│  question_id + controlled value + importance (1–5)    │
└─────────────────────────────┬─────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────┐
│            Deterministic Jodi Engine                   │
│  Alignment detection + gap detection + priority rules  │
└─────────────────────────────┬─────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────┐
│                 Evidence Layer                         │
│  Exact question ID + both selected answers + finding ID│
└─────────────────────────────┬─────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────┐
│              Scenario Generation Layer                 │
│  Neutral, topic-specific guided discussion scenarios   │
└─────────────────────────────┬─────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────┐
│              Optional Gemini AI Layer                  │
│  Writes wording only; does not create findings         │
└─────────────────────────────┬─────────────────────────┘
                              ↓
┌───────────────────────────────────────────────────────┐
│                 Validation Layer                       │
│  Structure + safety + grounding + evidence links       │
└───────────────────────────────────────────────────────┘
```

The deterministic engine is the source of truth.

Gemini is a language layer, not a decision layer.

---

# 6. Step-by-step build history

## Step 1 — Create structured data models

### Code concept

```python
@dataclass
class Answer:
    question_id: str
    topic: str
    answer: str
    importance: int

@dataclass
class Profile:
    name: str
    answers: List[Answer]
```

### What this code does

`@dataclass` tells Python to create a simple data container.

`Answer` represents one response to one question.

It stores:

- `question_id`: machine-readable identifier, for example `career_location`
- `topic`: display label, for example `Career and location`
- `answer`: the selected or written answer
- `importance`: how strongly the person feels about it, from 1 to 5

`Profile` represents one participant and their list of answers.

### Why this matters

A product cannot reliably compare loose chat messages alone. Structured data makes answers:

- comparable
- testable
- serializable to JSON
- usable by a web app
- usable by an API later

---

## Step 2 — Create fictional profiles

### Code concept

```python
aarav = Profile(...)
meera = Profile(...)
```

### What this code does

It creates fictional participants used only for testing.

Aarav and Meera answer the same questions differently in some areas.

For example:

- Aarav is open to relocation for a major opportunity.
- Meera prefers staying in the current city.
- Aarav prefers discussing children sooner.
- Meera prefers waiting longer.

### Why this matters

Real matrimonial data is sensitive. Development began with fictional data so the engine could be tested safely.

---

## Step 3 — Build the first expectation-gap detector

### Code concept

```python
def analyze_expectation_gaps(person_a, person_b):
    answers_a = {
        answer.question_id: answer
        for answer in person_a.answers
    }
```

### What this code does

The dictionary comprehension turns a list of answers into a lookup table.

For example:

```python
answers_a["career_location"]
```

quickly finds the career-location answer for Person A.

The engine then:

1. finds questions both people answered
2. compares their answers
3. creates an `alignment` if answers match
4. creates an `expectation_gap` if answers differ
5. marks the finding important if either person selected importance 4 or 5

### Initial limitation

The first version compared full written sentences.

That was useful for learning, but not reliable enough for a real product.

Two people could mean the same thing using different words.

---

## Step 4 — Add guided scenarios

### Code concept

```python
SCENARIO_TEMPLATES = {
    "Career and location": {
        "scenario": "...",
        "discussion_questions": [...]
    }
}
```

### What this code does

This dictionary stores neutral guided scenarios for a topic.

Each template has:

- a realistic situation
- three discussion questions

The scenario generator receives the findings and only creates a scenario when:

```python
finding["type"] == "expectation_gap"
```

and:

```python
finding["priority"] == "important conversation"
```

### Why this matters

The product does not stop at “difference detected.”

It helps a couple discuss the difference constructively.

---

## Step 5 — Create a structured JSON report

### Code concept

```python
report = {
    "schema_version": "0.1",
    "participants": {...},
    "findings": findings,
    "scenarios": scenarios,
    "limitations": [...]
}
```

### What this code does

A Python dictionary is converted into JSON using:

```python
json.dumps(report, indent=2)
```

The report can later be displayed in:

- a web interface
- a matchmaker dashboard
- an API response
- a downloaded report
- an evaluation system

### Important product principle

The report includes limitations.

This prevents the product from presenting itself as a marriage recommendation engine.

---

## Step 6 — Move from free text to controlled answer keys

### Code concept

```python
QUESTION_CATALOG = {
    "career_location": {
        "topic": "Career and location",
        "options": {
            "relocate_if_needed": "...",
            "stay_in_current_city": "..."
        }
    }
}
```

### What this code does

The product stores internal values such as:

```text
relocate_if_needed
```

instead of comparing entire sentences.

It displays friendly wording such as:

```text
Relocate if an important career opportunity makes sense
```

### Why this matters

Controlled values are better for:

- reliable comparison
- analytics
- tests
- APIs
- future recommendation systems
- avoiding spelling or wording variation

---

## Step 7 — Validate structured input

### Code concept

```python
def create_answer(question_id, value, importance):
    if question_id not in QUESTION_CATALOG:
        raise ValueError(...)

    if value not in valid_options:
        raise ValueError(...)

    if importance not in range(1, 6):
        raise ValueError(...)
```

### What this code does

The function rejects invalid input.

It ensures:

- the question exists
- the selected answer belongs to the question
- importance is only from 1 through 5

### Why this matters

Validation prevents bad data from entering the engine.

A production system must never silently accept invalid values.

---

## Step 8 — Build the structured comparison engine

### Code concept

```python
if a.value == b.value:
    finding_type = "alignment"
else:
    finding_type = "expectation_gap"
```

### What this code does

The engine compares controlled values, not full sentences.

Each finding includes:

```python
{
    "finding_id": "gap:career_location",
    "type": "expectation_gap",
    "topic": "Career and location",
    "priority": "important conversation",
    "evidence": {...}
}
```

### Important fields

`finding_id`

A stable identifier, for example:

```text
gap:career_location
```

This allows future scenarios, AI explanations, feedback, and database records to point to the exact finding.

`evidence`

Stores:

- the source question ID
- both internal answer keys
- both display answers

This makes every result auditable.

---

## Step 9 — Fix finding order

### Original issue

The first engine used a Python `set` to loop over shared question IDs.

Sets do not preserve a guaranteed display order.

### Fix

```python
for question_id in QUESTION_CATALOG:
```

### What this does

The engine now processes questions in the order defined by the catalogue.

This creates stable output for:

- users
- tests
- reports
- future APIs

Stable ordering is a small but important production-quality detail.

---

## Step 10 — Expand to five Level 1 dimensions

The final Level 1 dimensions are:

1. Career and location
2. Children timeline
3. Family care
4. Money decisions
5. Household responsibilities

### Why these dimensions were selected

They are practical daily-life topics that often need direct discussion.

They are not personality diagnoses or predictions.

---

## Step 11 — Add deterministic scenarios for all dimensions

The final `SCENARIO_TEMPLATES` dictionary includes guidance for:

- career relocation
- children timing
- family-care support
- major money decisions
- household responsibilities

### Why deterministic templates are useful

The scenario itself does not need an LLM.

A carefully authored scenario is:

- predictable
- inexpensive
- easy to review
- easy to test
- less likely to invent facts

Gemini is used later only for personalized wording.

---

## Step 12 — Build deterministic evaluation tests

### Code concept

```python
assert actual_gap_topics == expected_gap_topics
```

### What this code does

`assert` means the condition must be true.

If it is false, the notebook stops and reveals a regression.

The first deterministic tests verified:

- expected gaps were found
- expected alignments were found
- important gaps produced scenarios
- every scenario traced back to a source finding

### Why this matters

Without automated evaluation, a small code edit can silently break important product behavior.

---

## Step 13 — Add edge-case evaluation

The edge cases tested were:

1. Same answers create alignments
2. Low-importance differences stay low priority
3. Missing answers are skipped instead of guessed

### Missing-answer rule

```python
if question_id not in answers_a or question_id not in answers_b:
    continue
```

### What this does

If either person skipped a question, the engine does not create a conclusion.

This is safer than guessing.

---

# 7. Gemini AI layer

## Step 14 — Create a limited AI request

### Code concept

```python
def build_llm_explanation_request(finding):
```

### What this code does

It sends Gemini only one already-approved expectation gap.

It includes:

- topic
- priority
- the two allowed display answers
- strict safety rules
- required JSON structure

It does not send:

- the entire profile
- unrelated answers
- hidden system data
- a request to score the relationship

---

## Step 15 — Define the AI safety rules

The Gemini request includes rules such as:

```text
Use only the evidence supplied below.
Do not infer personality, values, intent, or relationship outcome.
Do not give a marriage recommendation.
Do not say the people are compatible or incompatible.
Use respectful, non-judgmental language.
```

### Why this matters

The LLM should explain, not decide.

---

## Step 16 — Validate structured Gemini output

### Code concept

```python
class GeminiExplanation(BaseModel):
    plain_language_explanation: str
    discussion_question: str
    unsupported_claims: List[str]
    evidence_used: List[EvidenceUsed]
```

### What this code does

Pydantic validates that Gemini returns the required JSON fields.

`BaseModel` describes the expected data shape.

If Gemini returns invalid JSON or missing fields, validation fails.

### Required Gemini response fields

- `plain_language_explanation`
- `discussion_question`
- `unsupported_claims`
- `evidence_used`

---

## Step 17 — Safety validator

### Code concept

```python
BANNED_VERDICT_PHRASES = [
    "compatible",
    "incompatible",
    "should marry",
    "should not marry"
]
```

### What this code does

The validator blocks known relationship-verdict language.

It also checks:

- response is a dictionary
- required fields exist
- text fields are strings
- `unsupported_claims` is an empty list

### Limitation

Keyword blocking alone cannot prove that all model claims are supported.

That is why grounding and evidence-link checks were added.

---

## Step 18 — Evidence grounding validator

### Code concept

```python
if (person, stated_answer) not in expected_evidence:
    return False
```

### What this does

Gemini must cite answers that exactly match the engine’s allowed evidence.

It cannot cite:

```text
Meera refuses to relocate
```

because that is not her actual selected answer.

It must cite:

```text
Meera: Prefer to remain in the current city
```

### Why this matters

The AI output remains tied to the structured source data.

---

## Step 19 — Evidence-link validator

Prompt Version `1.1.0` introduced stronger requirements.

### Code concept

```python
class EvidenceLink(BaseModel):
    output_field: Literal[
        "plain_language_explanation",
        "discussion_question"
    ]
    cited_people: List[str]
```

### What this does

Gemini must link both:

- its explanation
- its discussion question

to both participants.

### Validation rule

```python
if cited_people != allowed_people:
    return False
```

This means each output field must cite both people.

---

# 8. Gemini setup in Colab

## API key handling

The key is stored in Colab Secrets under:

```text
GEMINI_API_KEY
```

### Why this matters

API keys must never be:

- pasted into notebook code
- printed in output
- saved in a report
- committed to GitHub

### Client creation

```python
gemini_client = genai.Client(api_key=gemini_api_key)
```

This creates the Gemini client using the secret stored in Colab.

---

# 9. Product hardening

## Deterministic evaluation dataset

A 30-case fictional evaluation dataset was created.

### Dataset coverage

| Test category | Number of cases |
|---|---:|
| Alignment cases | 5 |
| High-priority one-topic gaps | 10 |
| Low-priority gaps | 5 |
| Missing-answer safety cases | 5 |
| Multi-gap cases | 5 |
| Total | 30 |

### Why the dataset matters

It ensures changes do not accidentally break:

- gap detection
- alignment detection
- priority logic
- missing-answer behavior
- scenario generation
- finding counts
- scenario IDs

---

## Error log

### Code concept

```python
error_log = [
    result
    for result in evaluation_results
    if not result["passed"]
]
```

### What this does

It collects only failed test cases into a structured error log.

The error log is saved as:

```text
jodi_level_1_error_log.json
```

### Important learning result

The first hardening run reported:

```text
Passed: 24/30
Failures logged: 6
```

The test suite correctly revealed missing scenario templates for:

- Family care
- Money decisions

Those templates were added to `jodi_engine.py`.

After the fix, the evaluation suite passed.

This is exactly why automated evaluation is valuable.

---

## Prompt and engine version tracking

### Versions created

```python
ENGINE_VERSION = "1.0.1"
PROMPT_VERSION = "1.1.0"
EVALUATION_DATASET_VERSION = "1.0.0"
```

### What each version means

`ENGINE_VERSION`

Tracks deterministic logic changes.

`PROMPT_VERSION`

Tracks Gemini instruction changes.

`EVALUATION_DATASET_VERSION`

Tracks which test dataset was used to validate a release.

### Why this matters

If Gemini behavior changes after a prompt edit, the team can identify:

- which prompt changed
- which engine produced the finding
- which test suite validated the release

This is a basic LLMOps practice.

---

## Finding fingerprint

### Code concept

```python
hashlib.sha256(...).hexdigest()[:16]
```

### What this does

It creates a short, privacy-preserving fingerprint for a structured finding.

The operational log stores the fingerprint rather than raw answers.

### Why this matters

In production, logs should minimize sensitive personal content.

---

## AI run log

Each Gemini run records:

- run ID
- timestamp
- status
- finding ID
- version metadata
- latency
- rejection reason if applicable

It does not record:

- API keys
- raw full prompts
- unrelated profile data

---

## AI prompt evaluation

Prompt Version `1.1.0` was tested against the three important fictional gaps:

1. Career and location
2. Children timeline
3. Household responsibilities

Result:

```text
Prompt evaluation passed: 3/3
Gemini Prompt Version 1.1.0 passed all synthetic AI checks.
```

Each response passed:

- structured-output validation
- safety validation
- evidence grounding validation
- field-level evidence-link validation

---

# 10. Colab Gradio prototype

## Purpose

The Gradio interface creates a visible prototype without building a full backend.

It allows two fictional profiles to select answers and view:

- alignments
- expectation gaps
- guided scenarios
- optional validated Gemini wording

## Two-button design

### Button 1

```text
Create deterministic discussion map
```

This runs only the deterministic engine.

It does not call Gemini.

### Button 2

```text
Generate validated AI wording
```

This calls Gemini for only one selected topic.

It only runs when:

1. the topic is an expectation gap
2. the gap is marked important
3. Gemini output passes validation

### Why two buttons are important

The deterministic core remains independently useful and testable.

Gemini is optional, traceable, and cannot silently replace deterministic logic.

## Temporary public link warning

Gradio used:

```python
demo.launch(share=True, debug=True)
```

`share=True` creates a temporary public link.

Only fictional data should be used in this prototype.

---

# 11. Current files

```text
PROJECT_DOCUMENTATION.md
    This project explanation document.

README.md
    Short GitHub-facing project overview.

requirements.txt
    Python dependency list.

jodi_engine.py
    Level 1 deterministic engine.

Jodi_Intelligence_Level_1_MVP.ipynb
    Colab development notebook.

jodi_level_1_evaluation_dataset.json
    30 fictional deterministic cases.

jodi_level_1_error_log.json
    Deterministic evaluation failures, if any.

jodi_ai_prompt_evaluation_v1_1_0.json
    Gemini prompt-evaluation results.

jodi_ai_prompt_error_log_v1_1_0.json
    Gemini prompt-evaluation failures, if any.
```

---

# 12. Dependencies

```text
google-genai
    Gemini API SDK

pydantic
    Structured-output validation

gradio
    Colab web prototype

pytest
    Future GitHub test runner
```

---

# 13. Current release status

```text
Jodi Intelligence Level 1.0.1

Deterministic engine:
- Five dimensions
- Evidence-grounded findings
- Guided scenarios
- 30 synthetic evaluation cases
- Hardened

Gemini layer:
- Structured JSON output
- Safety validation
- Evidence grounding
- Evidence links
- Prompt Version 1.1.0 tested
- Hardened on synthetic cases

Data policy:
- Fictional data only
- No production user data
```

---

# 14. Next roadmap

## Phase 2 — GitHub packaging

Move stable Colab code into a repository:

```text
jodi-intelligence/
├── src/
│   ├── engine.py
│   ├── ai_guardrails.py
│   ├── gemini_client.py
│   └── report.py
├── tests/
│   ├── test_engine.py
│   ├── test_guardrails.py
│   └── fixtures/
├── data/
│   └── synthetic/
├── app.py
├── requirements.txt
├── README.md
└── PROJECT_DOCUMENTATION.md
```

## Phase 3 — Conversation intelligence

With explicit consent and synthetic data first:

- extract discussion topics
- identify unresolved questions
- identify stated decisions
- create follow-up prompts
- build couple interaction memory

## Phase 4 — RAG and retrieval

Later, retrieve consented prior context:

- previous answers
- approved summaries
- matchmaker notes
- unresolved decisions

Use hybrid search and reranking to surface the most relevant evidence.

## Phase 5 — Matchmaker workflow

Build:

- matchmaker dashboard
- human review queue
- escalation for sensitive conflicts
- feedback and correction workflow

## Phase 6 — Production readiness

Before real user data:

- explicit consent flow
- data-minimization policy
- encryption
- role-based access
- deletion controls
- audit logging
- quality monitoring
- human escalation process

---

# 15. Key lessons learned

1. Start with structured data before adding an LLM.
2. Use deterministic rules for core findings.
3. Use LLMs for language and support, not high-impact decisions.
4. Preserve evidence for every finding.
5. Test missing data instead of guessing.
6. Build evaluation before scaling features.
7. Log version numbers for engine, prompt, and tests.
8. Treat privacy and consent as product requirements, not future polish.
9. Keep a human review path for complex or sensitive cases.
10. A commercial AI product is a system, not only a chatbot prompt.
