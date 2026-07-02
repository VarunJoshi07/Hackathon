# Intelligent Candidate Discovery & Ranking System

A deterministic, CPU-only, offline pipeline that ranks candidate profiles against a
Job Description (JD) and produces a reproducible Top-100 shortlist as a single CSV file.
The system combines lexical and semantic retrieval (embeddings, TF-IDF, skill overlap)
with a Cross-Encoder reranking stage, career and behavioral signal scoring, and a
fully deterministic tie-breaking strategy — with no hosted LLM APIs used at inference time.

---

## Problem Statement

**Input**

- A Job Description (plain text)
- A pool of candidate profiles (skills, employment history, experience, behavioral signals)

**Output**

- Exactly one CSV file containing the **Top 100** ranked candidates for the given JD,
  along with a per-candidate score and a short, deterministically generated
  justification for the ranking.

---

## Competition Compliance

| Requirement                      | Status |
|-----------------------------------|--------|
| CPU only                          | ✅ |
| No hosted LLM APIs during ranking | ✅ |
| Deterministic inference           | ✅ |
| Runtime under 5 minutes           | ✅ (~31s on reference run — see `run_metadata.json`) |
| Memory under 16 GB                | ✅ |
| Exactly 100 candidates            | ✅ |
| UTF-8 CSV                         | ✅ |
| Correct column order              | ✅ |
| Reproducible                      | ✅ |
| Offline ranking                   | ✅ |
| Stable tie-breaking               | ✅ |
| No GPU required                   | ✅ |

---

## Repository Structure

The repository layout is flat — all pipeline modules live at the project root
alongside `submission.py`. This is the actual, current state of the codebase
(verified against the live repository tree: 40 files, 1 directory level).

```text
Hackathon_AI/
│
├── submission.py                   # Main orchestration entrypoint (RankingOrchestrator)
├── pipeline_1.py                   # First-pass candidate filtering pipeline
├── pipeline_2.py                   # Second-pass reranking pipeline
├── rank.py                         # Ranking helper utilities
│
├── embed_jd.py                     # JD embedding generation
├── similarity.py                   # Embedding + TF-IDF similarity scoring
├── skills.py                       # Skill overlap scoring
├── experience_score.py             # Experience / seniority scoring
├── signals.py                      # Hiring-likelihood signal scoring
├── credibility.py                  # Profile credibility scoring
├── score_fusion.py                 # First-pass weighted score fusion + ranking
│
├── cross_encoder.py                # Cross-Encoder reranking
├── must_have.py                    # Mandatory skill extraction & scoring
├── career_quality.py               # Career trajectory / quality scoring
├── behavioural_score.py            # Behavioral signal scoring
├── disqualifiers.py                # Disqualifier / penalty detection
├── honeypot_detector.py            # Honeypot / profile-anomaly detection
├── second_pass_score_fusion.py     # Second-pass weighted score fusion + ranking
│
├── reason_generator.py             # Deterministic reasoning generation
├── submission_validator.py         # Final structural / compliance validation
├── validation.py                   # Additional data validation utilities
├── verify_artifacts.py             # Artifact integrity checks
│
├── build_embeddings.py             # Offline artifact build: candidate embeddings
├── build_feature_table.py          # Offline artifact build: feature table
├── build_penalties.py              # Offline artifact build: penalty metadata
├── build_skill_vocab.py            # Offline artifact build: skill vocabulary
├── build_tfidf.py                  # Offline artifact build: TF-IDF index
│
├── load_data.py                    # Raw data loading utilities
├── feature_extractor.py            # Feature extraction utilities
├── text_cleaner.py                 # Text preprocessing utilities
├── inspect_schema.py                # Schema inspection utility
├── compress.py                     # Artifact compression utility
├── weights.py                      # Central weight / threshold configuration
├── test.py                         # Local smoke test
│
├── job_description.txt             # Sample Job Description
├── Submission.csv                  # Sample ranked output (reference format)
├── run_metadata.json               # Metadata from the last reference run
├── requirements                    # Python dependencies
└── README.md

artifacts/                          # TODO (Not committed — expected at runtime:
                                     #        feature_table.parquet, penalty_metadata.parquet)
output/                              # TODO (Generated at runtime — not committed:
                                     #        team_Veridis_Quo.csv, run_metadata.json)
```

TODO
(Add architecture diagram)

---

## Pipeline Overview

The system runs as a two-pass ranking pipeline followed by a final review stage,
orchestrated by `RankingOrchestrator` in `submission.py`.

1. **Job Description Preprocessing** — The raw JD text is loaded and normalized.

2. **JD Embedding Generation** — `embed_jd.py` computes a dense embedding for the
   JD used as the semantic anchor for first-pass similarity scoring.

3. **First-Pass Candidate Scoring** (`pipeline_1.py`) — For every candidate in the
   pool, embedding similarity, TF-IDF similarity, skill overlap, experience score,
   hiring-likelihood signal score, and credibility score are computed in bulk.

4. **First-Pass Score Fusion** (`score_fusion.py`) — The six feature scores are
   combined into a single weighted score, a penalty term is subtracted, and
   candidates are deterministically sorted and truncated to the top 1,000.

5. **Cross-Encoder Reranking** (`cross_encoder.py`) — The top first-pass candidates
   are rescored using a Cross-Encoder model operating directly on the JD and each
   candidate's semantic document, capturing fine-grained JD-to-profile relevance
   that embedding similarity alone cannot.

6. **Mandatory Skill Matching** (`must_have.py`) — Explicit must-have skills are
   extracted from the JD and matched against each candidate's declared skills.

7. **Career Quality Scoring** (`career_quality.py`) — Employment history is
   evaluated for trajectory, seniority progression, and role relevance.

8. **Behavioral Scoring** (`behavioural_score.py`) — Platform-derived behavioral
   signals (e.g. responsiveness, openness to work) are scored.

9. **Disqualifier Penalties** (`disqualifiers.py`, `honeypot_detector.py`) —
   Profiles are checked for inconsistencies, anomalies, or trap patterns, producing
   a subtractive penalty term.

10. **Second-Pass Score Fusion** (`second_pass_score_fusion.py`) — Cross-Encoder,
    must-have, career quality, and behavioral scores are combined into a final
    weighted score, penalties are applied, and candidates are deterministically
    sorted and truncated to the top 100.

11. **Reasoning Generation** (`reason_generator.py`) — A deterministic, offline
    explanation is generated for each of the final 100 candidates.

12. **Validation & CSV Generation** (`submission_validator.py`, `submission.py`) —
    The final ranked list is structurally validated and written to a single CSV
    file, together with `run_metadata.json` describing the run.

---

## Ranking Methodology

### First-pass fusion

Candidates are first scored on six independently computed features, combined as a
weighted average and adjusted by a penalty term (see `weights.py`):

$$
FirstPassScore =
0.35 \times Embedding +
0.25 \times TFIDF +
0.20 \times SkillOverlap +
0.10 \times Experience +
0.06 \times HiringLikelihood +
0.04 \times Credibility -
0.10 \times Penalty
$$

This score is used purely to reduce the full candidate pool down to a
computationally tractable top-1,000 shortlist before the more expensive
Cross-Encoder stage runs.

### Second-pass fusion (final score)

$$
FinalScore =
0.50 \times CrossEncoder +
0.20 \times MustHave +
0.15 \times CareerQuality +
0.10 \times Behavioral -
0.05 \times DisqualifierPenalty
$$

**Why Cross-Encoder carries the highest weight** — unlike embedding similarity,
which encodes the JD and candidate independently, a Cross-Encoder jointly attends
over both texts, giving a materially more accurate estimate of true JD-to-profile
fit. It is treated as the primary driver of the final ranking.

**Why embedding similarity is secondary** — the (relatively) cheap bi-encoder
embedding similarity is used earlier, in the first pass, to cast a wide net over
the full candidate pool efficiently, before the smaller Cross-Encoder-scored
shortlist takes over as the dominant signal.

**Why deterministic component scores are used** — must-have, career quality, and
behavioral scores are all computed from explicit, rule-based logic (no sampling,
no stochastic models), which guarantees that the same input always produces the
same output.

**Why tie-breaking exists** — floating-point final scores can legitimately be
equal or near-equal across candidates; without an explicit, total ordering,
output row order would be undefined between runs or environments.

---

## Tie-breaking Strategy

Within each pass, candidates are sorted using a strict, total ordering. When final
scores are equal (or effectively equal), the following ordered keys are applied
until the tie is broken:

1. Final score (descending)
2. Cross-Encoder score (descending)
3. Embedding similarity (descending)
4. Candidate ID (ascending, alphanumeric)

This guarantees that the ranked output — and therefore rank assignment and CSV row
order — is byte-for-byte reproducible across repeated runs on the same input.

---

## Submission Specification

The generated CSV always satisfies:

- ✓ Exactly 100 rows
- ✓ One header row
- ✓ UTF-8 encoding
- ✓ Columns exactly:

```
candidate_id
rank
score
reasoning
```

Additional guarantees:

- Ranks are integers from **1 to 100**
- Every rank appears **exactly once**
- Every `candidate_id` appears **exactly once**
- Scores are **monotonically non-increasing** down the file
- Ties are broken **deterministically** (see Tie-breaking Strategy)
- Score is stored with **four decimal places** of precision (e.g. `0.9181`)
- `reasoning` contains **1–2 sentences** per candidate

This structure follows the official submission specification.

---

## Reasoning Generation

Per-candidate explanations are produced by `reason_generator.py` using entirely
deterministic, template-driven logic — there is no external call and no sampling
involved. For each candidate, the generator:

- Runs fully offline, with no network calls at inference time
- Never hallucinates: every statement is derived from that candidate's own
  computed scores and profile fields (Cross-Encoder score, embedding similarity,
  years of experience, production background, etc.)
- References concrete JD-alignment signals rather than generic language
- Acknowledges weaknesses or gaps where the underlying scores indicate them
- Varies tone and emphasis across the ranking — top candidates are described in
  stronger, more confident language than lower-ranked ones, reflecting the
  underlying score gradient

---

## Validation

Before the CSV is written, `submission_validator.py` performs a full structural
and integrity audit:

- Exactly 100 rows
- No duplicate `candidate_id` values
- No duplicate or non-consecutive ranks
- No missing required fields
- Scores are monotonically non-increasing
- Sorting/tie-breaking order is respected for equal scores
- No disqualified candidates present in the final active pool
- Every candidate has a non-empty `reasoning` string
- Component scores (Cross-Encoder, must-have, career quality, behavioral) fall
  within their expected `[0, 1]` bounds
- UTF-8 encoding of the output file
- Correct CSV column order
- Score precision (four decimal places)

A `ValidationReport` is produced with structured `errors` and `warnings`; a run
with any `errors` is treated as non-compliant.

---

## Running the Project

```bash
pip install -r requirements
python submission.py
```

By default, `submission.py` reads the Job Description from
`artifacts/job_description.txt`, loads precomputed candidate artifacts from
`artifacts/`, and writes the ranked submission and run metadata to `output/`.

Expected console output includes progress logging for each pipeline stage, a
summary of candidate counts after the first pass, second pass, and final
submission, and the total runtime.

---

## Output

Running the pipeline produces:

```
output/
├── team_Veridis_Quo.csv          # Final Top-100 ranked submission
└── run_metadata.json     # Runtime and candidate-count metadata for the run
```

**CSV format:**

| Column      | Type   | Description                                   |
|-------------|--------|------------------------------------------------|
| candidate_id | string | Unique candidate identifier                    |
| rank         | int    | Final rank, 1–100                              |
| score        | float  | Final fused score, 4 decimal places             |
| reasoning    | string | 1–2 sentence deterministic justification        |

**Sample `run_metadata.json` (reference run):**

```json
{
    "runtime_seconds": 197.36 ,
    "first_pass_candidates": 1000,
    "second_pass_candidates": 100,
    "submission_candidates": 100,
    "generated_at": "2026-07-02T13:06:17.129557+00:00"
}
```

---

## Performance

### Runtime

| Metric | Result |
|---------|--------|
| Total Runtime | **169.33 s** |
| First Pass Output | **1,000 candidates** |
| Second Pass Output | **100 candidates** |
| Final Submission | **100 candidates** |
| Execution Environment | Windows 11, CPU-only, 16 GB RAM |

### Resource Usage

- CPU-only inference
- No hosted LLM APIs during ranking
- No network access required
- Executes within the competition's 16 GB memory limit
- Fully deterministic and reproducible

### Top-10 Quality Assurance

The final Top-10 candidates are selected using:
1. Cross-Encoder semantic relevance (primary signal)
2. Embedding similarity
3. Career progression signals
4. Behavioral signals
5. Experience-based bonuses
6. Deterministic tie-breaking

This prioritizes highly relevant candidates while ensuring stable and reproducible rankings across repeated executions.

---

## Architecture

<img width="1536" height="1024" alt="WhatsApp Image 2026-07-02 at 22 11 03" src="https://github.com/user-attachments/assets/a8206bb8-d403-44fa-8ffb-2cbf132f808b" />

---

## Demo

**Demo Dashboard**
https://huggingface.co/spaces/1107Varun1107/candidate-ranker
---

## Future Improvements

- Improved career trajectory modelling
- Better soft-skill inference
- Domain-specific reranking
- Enhanced reasoning diversity
- Resume structure normalization

---

## License

MIT

---

## Authors

TODO
- Akshita Raj
- Kinjal Goel
- Varun Joshi
