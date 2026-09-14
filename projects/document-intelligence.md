# Verifiable Document Intelligence for Building Energy Audits

**Cedrus Solutions · 2025 – present · Production system**
**Role:** design and implementation — sole author of the pipeline described here.

> This case study describes the engineering approach at a high level. Source code,
> customer data and proprietary implementation details remain private.

---

## Problem

A building-energy platform needs a structured technical model of every asset it
simulates: geometry, envelope composition, HVAC equipment, operating schedules,
metered consumption. That model was assembled by an engineer reading technical
audit PDFs by hand — dense, inconsistent French documents mixing prose, regulatory
tables and equipment nameplates.

An earlier retrieval-augmented implementation already existed: a single top-K vector
search feeding one prompt per group of parameters. It produced values, but nothing
in it could answer the two questions that decide whether an extraction pipeline is
usable in production:

1. **Is this value actually supported by the documents?**
2. **When a value is missing or wrong, which part of the system lost it?**

Without answers, every output needs full manual re-verification, and the pipeline
saves nothing.

## Context

Audit documents defeat naive retrieval in specific, diagnosable ways:

- **A single query cannot serve every field.** The question that finds a wall
  U-value is not the question that finds a ventilation schedule.
- **Half of what an audit hangs its facts on is not semantic.** Equipment tags,
  physical symbols, coded references and raw figures are *tokens*. Dense embeddings
  are the wrong instrument for them.
- **Audits describe two worlds at once.** The building as it stands, and the building
  as a retrofit study proposes it should become. Both are stated in the same register,
  often in the same sentence.
- **A missing value and a wrong value look identical downstream.** A plausible-looking
  number in the wrong field is worse than an empty field, because nothing flags it.

## My contribution

I designed and built a second-generation reading pipeline alongside the existing one —
roughly **19 new modules, ~7,700 lines of implementation and ~3,400 lines of unit
tests**, on a path that previously had no test coverage at all.

I also built the R&D evaluation harness that made the work measurable rather than
estimated (described under *Evaluation*).

## Approach / Architecture

The pipeline is organised into seven layers. The point of the decomposition is that
each layer can fail **separately and observably**:

| Layer | Question it answers |
|---|---|
| **Structure** | What the building *is* — functional uses, identity |
| **Inventory** | What it *contains* — each machine and envelope element, with its own identity |
| **Scan** | What it *states* — scalar values, read across every page in bounded batches |
| **Targeted retrieval** | What the scan *missed* — one query per field still empty |
| **Evidence** | What the documents *actually support* |
| **Normalisation** | Source expression → domain value + the rule that produced it |
| **Routing** | Which use or which machine each value belongs to |

Exhaustive page coverage replaces the single top-K search: every page is read, in
bounded batches, before any field-specific query is issued.

## Engineering decisions

### 1. Evidence, not citation

A model asked for a citation always returns one — whether it read it or wrote it.
So the citation is not the evidence; **the corpus is**. Every claimed value passes
three cascading checks:

- the cited document and page exist;
- the quoted sentence is actually present in that page (tolerant to accents, casing
  and table-cell separators);
- the value appears in the sentence **and** the sentence describes the *current* state.

The last check matters most. *"Replace the 100 kW boiler with a 150 kW unit"* contains
"150 kW", is quoted faithfully, and proves no 150 kW boiler is installed. The gate
resolves to one of nine distinct verdicts rather than a boolean, so a rejection says
*why* it was rejected — wrong page, value absent from the quotation, future state, and
so on.

### 2. A closed vocabulary

An extractor that invents a destination field is worse than one that finds nothing:
a plausible-but-nonexistent path makes the value disappear without raising an error.
Every claimed path is resolved against the business schema, with enumeration, bounds
and unit checked. An unresolved path becomes an explicitly *unsupported field* —
never the nearest plausible neighbour.

### 3. The model reads, the code converts

The model is asked for the expression **as written**, not for "the value in kW".
Conversion happens in deterministic code, and the stored value keeps its source
expression, its conversion factor and the rule applied.

Ten numeric forms are recognised — scalar, range, tolerance, inequality, sum,
dimensions, list, schedule/time range, absence — because `re.findall(...)[0]` reads
`50 + 24 kW` as `50` and `45–50 %` as `45`.

### 4. Hybrid, field-level retrieval

Each query names its own field: the concept, the expected unit, French domain
synonyms, the machine or zone it belongs to, and the words that would signal a
*recommendation* rather than a statement of fact.

That query is answered by three searches — dense, full-text and literal — fused by
rank (reciprocal rank fusion), reranked, and deepened in steps (short → medium → wide)
**only when the shallow step returns nothing**. Depth is spent where it is needed
instead of uniformly.

### 5. Per-layer diagnostics

Every decision is counted, so a missing value is attributed to the layer that lost it
— retrieval, extraction, evidence, normalisation or routing — instead of the useless
verdict "the value is wrong". This is what turns an evaluation run into a work queue.

## Evaluation

I built an R&D harness around the pipeline: upstream and downstream adapters, scoring
against reference models filled in by an engineer, failure attribution by responsible
layer, multi-building aggregation (pooled cells, not an average of percentages), and
two exports — an exhaustive one for development and a readable one for the engineer,
where every value carries its provenance and its fate.

**Pipeline economics** on a representative 33-page audit: about **2 minutes**, about
**32 model calls**, roughly **310k input tokens**, for a model cost around **$0.10**
per building. A complete equipment inventory, each machine carrying its own identity.

**Before/after bench.** I replay the production path and the new one over the same
PDFs, the same vector table and the same pinned model, scored by the same projector.
Across three sites: recall is at least equivalent and identification of system families
improves. **Exact-match is not yet a net win** — the new pipeline answers more often,
so it is also visibly wrong more often. I am deliberately not converting that into a
"+X % accuracy" claim; the honest summary is that coverage and traceability improved
while exact-match parity is still being worked.

A finding worth recording: the targeted layer — one query per field the exhaustive
scan left empty — contributed more accepted values than the exhaustive scan itself.
The gain lives in knowing what you are missing, not in reading more.

## Technologies

Python · Pydantic · PostgreSQL · pgvector · dense retrieval · PostgreSQL full-text
search · literal/token retrieval · reciprocal rank fusion · reranking · LLM APIs
(Gemini-class models via OpenRouter) · retry/backoff for batched model calls ·
AWS Lambda · Amazon S3 · DynamoDB · pytest

## What I learned

- **A verification layer is worth more than a better prompt.** Most of the reliability
  came from refusing values, not from extracting more of them.
- **Uniform retrieval depth is waste.** Laddering depth per field, only on failure,
  bought coverage at roughly constant cost.
- **Attribution changes the conversation.** Once a failure has an owner layer, model
  quality stops being the explanation for everything.
- **Answering more often makes you look worse before it makes you look better.** A
  system that declines to answer has an artificially clean exact-match score. Reporting
  that honestly is part of the engineering.

## Links

Private — company codebase. No source, prompts or customer data are published.
