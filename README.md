# Confidently Wrong

## Reproducible experiments for AI-augmented threat hunting

This repository contains the benchmark, code, controls, and result files used to study a simple problem:

> Can an AI security workflow become confidently wrong even when the underlying telemetry does not change?

The repository has three connected experiments.

1. **Multi-agent judge test** — A primary model and a blind refuter inspect the same telemetry. A third model resolves disagreements.
2. **False-positive feedback-loop test** — The telemetry stays fixed. Only the history changes across four controlled conditions.
3. **MITRE ATT&CK grounding extension** — Compare an LLM-only condition with an ATT&CK-grounded retrieval condition. An optional environment-context condition is also supported.

The code is designed so other researchers and defenders can replace the model, repeat the run, and compare results.

---

## Why this exists

AI can help with SOC triage, summaries, ATT&CK mapping, and escalation decisions. The risk is not only a single hallucination. A model output can become part of a later model's context. If the system does not preserve provenance, an earlier AI conclusion can look like independent evidence.

This repository tests that failure mode under controlled conditions.

**Important:** The contaminated history in Experiment 2 is an intentional experimental treatment. It does **not** estimate how often this failure occurs in production.

---

## Benchmark

The benchmark contains **46 sanitized security investigation cases**:

- 23 MALICIOUS cases
- 23 BENIGN cases
- 8 telemetry events per case
- ground truth kept separate from the model payload
- ATT&CK technique IDs and direct answer labels removed from the model input
- hostnames, paths, and vendor-specific details normalized where needed to reduce label leakage

The model sees telemetry. It does not see the answer key.

Files:

```text
data/benchmark_dataset_v2.json
data/benchmark_ground_truth_v2.json
```


---

## Repository structure

```text
confidently-wrong-github/
├── README.md
├── LICENSE
├── CITATION.cff
├── DATASET_NOTICE.md
├── SECURITY.md
├── CONTRIBUTING.md
├── requirements.txt
├── .env.example
├── .gitignore
│
├── data/
│   ├── benchmark_dataset_v2.json
│   ├── benchmark_ground_truth_v2.json
│   ├── feedback_loop_frozen_history_public_v1.json
│   ├── environment_context.example.json
│   └── attack/
│       └── .gitkeep
│
├── src/
│   └── openrouter_client.py
│
├── experiments/
│   ├── 01_multi_agent_judge/
│   │   ├── README.md
│   │   └── run.py
│   ├── 02_feedback_loop/
│   │   ├── README.md
│   │   ├── build_public_treatments.py
│   │   └── run.py
│   └── 03_attack_grounding/
│       ├── README.md
│       └── run.py
│
├── scripts/
│   └── fetch_attack_stix.py
│
├── docs/
│   ├── METHODOLOGY.md
│   ├── DATA_DICTIONARY.md
│   ├── REPLICATION_GUIDE.md
│   ├── LIMITATIONS.md
│   └── RESULTS.md
│
└── results/
    ├── published_summary.json
    ├── feedback_loop_v3_summary.csv
    └── feedback_loop_v3_gpt4o_mini_results.json
```

---

## Quick start

### 1. Create a Python environment

```bash
python -m venv .venv
source .venv/bin/activate       # macOS / Linux
# .venv\Scripts\activate        # Windows PowerShell
pip install -r requirements.txt
```

### 2. Add your OpenRouter API key

```bash
export OPENROUTER_API_KEY="YOUR_KEY"
```

On Windows PowerShell:

```powershell
$env:OPENROUTER_API_KEY="YOUR_KEY"
```

Do not commit the key.

### 3. Validate the experiments without making API calls

```bash
python experiments/01_multi_agent_judge/run.py --dry-run
python experiments/02_feedback_loop/run.py --dry-run
```

---

# Experiment 1 — Can an LLM judge improve triage?

Flow:

```text
CASE TELEMETRY
     │
     ├──> PRIMARY AGENT ──┐
     │                    ├── disagreement? ──> LLM JUDGE ──> FINAL VERDICT
     └──> BLIND REFUTER ──┘
```

The refuter does not receive the primary agent's answer. This keeps the two first-stage assessments independent.

Run with one model for all roles:

```bash
python experiments/01_multi_agent_judge/run.py \
  --primary openai/gpt-4o-mini
```

Run with different models:

```bash
python experiments/01_multi_agent_judge/run.py \
  --primary openai/gpt-4o-mini \
  --refuter qwen/qwen3-235b-a22b-2507 \
  --judge z-ai/glm-5.3-flash
```

The script reports TP, FP, TN, FN, false-positive rate, false-negative rate, and attack recall.

A previous run produced the following conference summary:

| Configuration | Accuracy | Attacks caught | False positives |
|---|---:|---:|---:|
| Single agent A | 70% | 61% | 22% |
| Single agent B / refuter | 83% | 91% | 25% |
| A + B + LLM judge | 70% | 61% | 22% |

There were 12 disagreements. In that run, the judge sided with the weaker model 9 times.

Treat these numbers as a recorded result, not a guaranteed result. Models and hosted endpoints change. Re-run the experiment for your model and date.

---

# Experiment 2 — False-positive feedback loops

The telemetry stays fixed. Only the history changes.

| Condition | Context | Purpose |
|---|---|---|
| A — Baseline | Current telemetry only | Measure the model without history |
| B — Matched factual | Factual history from the same telemetry lineage | Control for more context and repetition |
| C — AI contaminated | Suspicious AI-generated history | Measure sensitivity to inherited AI claims |
| D — Provenance aware | Same text as C, marked `AI_GENERATED_UNVERIFIED` | Test whether provenance labels reduce the effect |

Controls:

- 23 BENIGN cases
- 4 repeated rounds
- telemetry held constant
- B, C, and D have matched history counts
- C and D have the same substantive AI history
- D changes provenance metadata only

Run:

```bash
python experiments/02_feedback_loop/run.py \
  --model openai/gpt-4o-mini
```

Try another model:

```bash
python experiments/02_feedback_loop/run.py \
  --model qwen/qwen3-235b-a22b-2507
```

The repository includes the original v3 `gpt-4o-mini` result file and a transparent public treatment generator. The public treatment file is **not** claimed to be byte-for-byte identical to the original private frozen-history file. It is included so other people can run the same controlled method without hidden treatment generation.

---

# Experiment 3 — ATT&CK-grounded evaluation

This extension asks a different question:

> Does ATT&CK grounding make the security decision more correct, or only more confident?

The retrieval corpus is official MITRE ATT&CK STIX data. AI-generated conclusions are not added to the trusted retrieval corpus.

### Download a pinned ATT&CK release

```bash
python scripts/fetch_attack_stix.py --version 19.2
```

### Run LLM-only vs ATT&CK RAG

```bash
python experiments/03_attack_grounding/run.py \
  --attack-stix data/attack/enterprise-attack-19.2.json \
  --model openai/gpt-4o-mini
```

### Add environment context

```bash
python experiments/03_attack_grounding/run.py \
  --attack-stix data/attack/enterprise-attack-19.2.json \
  --environment-context data/environment_context.example.json
```

The included benchmark has decision ground truth. It does not include a human-reviewed ATT&CK technique answer key. Therefore, do not claim technique-mapping accuracy until you add and review such a key. See `docs/METHODOLOGY.md`.

---

## Primary metrics

Use the same metrics across model runs:

- True positive (TP)
- False positive (FP)
- True negative (TN)
- False negative (FN)
- false-positive rate
- false-negative rate
- attack recall
- review / escalation rate
- abstention (`UNSURE`) rate
- confidence
- cross-model disagreement
- ATT&CK mapping accuracy, only when a human-reviewed mapping key exists
- unsupported attribution rate

For Experiment 2, also compare the change across rounds.

---

## Reproduce before you compare

Record these items with every run:

- date and time
- model ID
- provider
- prompt version
- temperature
- ATT&CK release
- dataset SHA-256
- ground-truth SHA-256
- treatment-file SHA-256
- number of cases
- number of rounds

The current dataset hashes are documented in `docs/METHODOLOGY.md`.

---

## What this repository does not prove

This benchmark does not prove that all LLMs behave the same way. It does not measure the natural frequency of feedback-loop contamination in real SOC products. It does not show that RAG always improves security decisions. It does not treat ATT&CK coverage as the same thing as ATT&CK correctness.

The goal is narrower: provide a controlled test method that other people can repeat, challenge, and extend.

---

## How to contribute a replication

Run the experiment with another model or architecture. Save the raw result file. Do not overwrite prior results. Open a GitHub issue with:

- model and provider
- run date
- experiment number
- dataset hash
- prompt changes, if any
- result file
- short summary of differences

See `CONTRIBUTING.md` and `.github/ISSUE_TEMPLATE/replication.yml`.

---

## Research and defensive-use note

Use this repository for AI assurance, DFIR research, SOC testing, training, and defensive evaluation. Use synthetic or authorized telemetry. Do not upload sensitive production logs to a public repository.

## Citation

Use `CITATION.cff` or GitHub's **Cite this repository** feature after the repository is published.
