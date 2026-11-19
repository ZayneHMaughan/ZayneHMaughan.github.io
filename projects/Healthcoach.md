# healthcoach

Desktop, on-device health guidance tool. Not a diagnostic device. Rules and thresholds are drafts for clinician review.

## Layout
```
src/healthcoach/
  schema.py      Intake, Vitals, Tier
  rules.py       hard-coded red-flag layer (runs first, can only raise urgency)
  risk.py        loads models/risk_model/ bundle, returns score + tier
  safety.py      post-checks on LLM output
  pipeline.py    rules -> classifier -> retrieval + LLM -> safety check
  resources.py   region-specific emergency/crisis contacts
  intake/
    flow.py      data-driven question flow + IntakeSession (UI-agnostic)
    cli.py       terminal demo: python -m healthcoach.intake.cli
    data/        questions.json (screener, follow-ups, context, readings),
                 symptoms.json (patient-facing labels/synonyms), cc_symptom_mapping.csv
  retrieval/
    ingest.py    MedlinePlus XML (health-topic summaries only) + curated Markdown -> chunks
    index.py     hybrid BM25 + optional dense vectors (RRF), on-disk, integrity-checked
    retriever.py Intake + tier -> cited, token-budgeted passages; reports uncovered symptoms
    embed.py     hashing (tests), sentence-transformers, llama.cpp adapters
    build.py     python -m healthcoach.retrieval.build ...
    data/topic_map.csv   symptom group -> MedlinePlus titles (UNVERIFIED candidates; the build reports mismatches)
  warning_signs.py  plain-language "call emergency services if..." lines per symptom (draft, mirrors rules.py)
  finetune/
    sampler.py       synthetic intakes drawn from the intake schema (never from the ER dataset)
    targets.py       extractive baseline targets: format + safety behaviour, stilted prose by design
    build_dataset.py python -m healthcoach.finetune.build_dataset --index models/kb --out data/ft
    validate.py      checks any chat JSONL (ours or rewritten targets): tier, citations, dosing, grounding
evals/
  cases/*.jsonl  test cases (seed set; grow to 100+, clinician-reviewed)
  run_evals.py   harness, exits 1 on any under-triage
tests/           unit tests
```
Planned (not included): `training/` (feature build + classifier), `finetune/` (synthetic data + LoRA),
`retrieval/` (vetted-source index), `app/` (desktop UI), `models/` (git-ignored artifacts).

## Quickstart
```
pip install -e ".[dev]"
python evals/run_evals.py
pytest
```

## Intake flow
Screener (emergencies first) -> age/sex -> symptoms -> severity/duration -> red-flag follow-ups ->
context -> optional readings -> optional free text. Rules re-run after every answer; once they
reach EMERGENCY the flow stops asking. "Not sure" answers are treated conservatively on
high-stakes questions and recorded in `IntakeResult.assumptions`.
Edit `questions.json` / `symptoms.json` (no code changes needed) and run the tests: they check that
every modifier the rules read can be collected, that no question is dead, and that every seed eval case
can be expressed through the flow.

## Retrieval
```
# 1. download "MedlinePlus Health Topic XML" from https://medlineplus.gov/xml.html into data/medlineplus/
#    (project root, git-ignored; NOT inside src/). Run commands from the project root.
python -m healthcoach.retrieval.build --out models/kb      # picks the newest XML in data/medlineplus/
python -m healthcoach.retrieval.build --medlineplus data/medlineplus/mplus_topics_YYYY-MM-DD.xml --out models/kb
python evals/run_retrieval_evals.py --index models/kb --coverage     # which symptoms have no grounding?
python evals/run_retrieval_evals.py --index models/kb                # seed cases (verify titles first)
python evals/run_evals.py --index models/kb --llm mymod:generate     # LLM runs require an index
```
- LLM is only called when relevant passages exist; its JSON must include `sources` citing only passages it was given.
- UI must display `Response.attribution` and may show `Response.sources`.
- Licensing: only MedlinePlus health-topic summaries are ingested. Do NOT ingest A.D.A.M. encyclopedia, ASHP drug
  monographs, or images (copyrighted). Re-read https://medlineplus.gov/about/using/usingcontent/ before shipping.
- Refresh the corpus periodically (`Index.stale()`); MedlinePlus republishes daily.

## Run the app (local web page, http://localhost:8888)
```
python -m healthcoach.app --index models/kb --gguf models/gguf/tuned-q4_k_m.gguf --open
python -m healthcoach.app            # rules + standard messages only; works before you have a model or knowledge base
```
Optional: `--risk-model models/risk_model`, `--port 9000` (8888 is also Jupyter's default), `--budget-tokens N`
(read automatically from `data/ft/stats.json` if present; must match training). Binds to localhost only.
Answers live in memory for the session only (30 min idle timeout) and are never written to disk or logged.
Front end: `src/healthcoach/app/static/` (plain HTML/CSS/JS, no external resources, text inserted via textContent only).

## Fine-tuning data
```
python -m healthcoach.finetune.build_dataset --index models/kb --out data/ft --n 3000 --seed 7
python -m healthcoach.finetune.validate data/ft/train.jsonl data/ft/eval.jsonl
```
- Needs a REAL index (the builder refuses an index containing synthetic test fixtures unless `--allow-synthetic`).
- Examples are produced by the real rules, retriever, and prompt builder; the shared `SYSTEM_PROMPT` and
  `build_messages()` in `pipeline.py` must also be used at inference time.
- Train/eval are split by scenario signature. The hand-written cases in `evals/cases` are the true held-out test.
- `stats.json` is the data card: tier/category/symptom mix, corpus gaps, skips, index manifest, seed.
- Targets are an extractive baseline. Upgrade path: rewrite targets (person or stronger model, check its terms),
  then re-run `finetune.validate` before training.

## LoRA runbook
Two paths. `llm.py` builds a JSON schema per prompt (urgency pinned to the assigned tier, sources limited to the
given ids) and runs on llama.cpp (Metal on Apple Silicon) for both.

### Apple Silicon (M-series, e.g. M2 with 8 GB): MLX path. NOT run in the authoring environment (no Mac).
```
pip install "mlx-lm[train]"            # training/fusing
pip install -e ".[llm]"                # llama-cpp-python for evals/inference (build it with Metal)
# 1. shorter sequences fit 8 GB: build data with a smaller retrieval budget; use the SAME budget at inference
python -m healthcoach.finetune.build_dataset --index models/kb --out data/ft --n 2000 --budget-tokens 700 --max-tokens 1500
# 2. smoke run first (50 iterations): check speed and peak memory in Activity Monitor
python -m healthcoach.finetune.train_mlx --model <quantized 4-bit MLX instruct model, ~1-1.5B> --data data/ft --work models/mlx_run --smoke
# 3. full run (add --dry-run to see the exact mlx-lm command)
python -m healthcoach.finetune.train_mlx --model <same> --data data/ft --work models/mlx_run
# 4. fuse into the UNQUANTIZED original model, then convert + quantize with llama.cpp (commands are printed).
#    Find the original id on the MLX model's Hugging Face page: README says "converted to MLX format from <id>".
#    e.g. mlx-community/Llama-3.2-1B-Instruct-4bit -> meta-llama/Llama-3.2-1B-Instruct (gated: accept the license, `huggingface-cli login`)
python -m healthcoach.finetune.fuse_mlx --base-model <original fp16 HF id> --adapters models/mlx_run/adapters --out models/fused_hf
# 5. GGUF: one-time llama.cpp setup (use a SEPARATE venv for the converter's torch/transformers requirements)
git clone https://github.com/ggml-org/llama.cpp ~/llama.cpp
cmake -S ~/llama.cpp -B ~/llama.cpp/build -DGGML_METAL=ON && cmake --build ~/llama.cpp/build --config Release -j 8
python3 -m venv ~/convert-venv && ~/convert-venv/bin/pip install -r ~/llama.cpp/requirements.txt
python -m healthcoach.finetune.make_gguf --fused models/fused_hf --llama-cpp ~/llama.cpp --out models/gguf --name tuned \
    --convert-python ~/convert-venv/bin/python
# baseline GGUF without converting: hf download unsloth/Llama-3.2-1B-Instruct-GGUF Llama-3.2-1B-Instruct-Q4_K_M.gguf --local-dir models/gguf
# 6. evaluate the quantized GGUF you will ship, against the un-tuned baseline (see below)
```
Out of memory? In order: `--batch-size 1`, `--num-layers 4`, lower `--max-seq-length` and rebuild data with a smaller
`--budget-tokens`, a smaller model, close other apps.

### CUDA GPU: TRL/PEFT path
```
pip install -e ".[lora,llm]"           # + bitsandbytes for --qlora
python -m healthcoach.finetune.train_lora --base-model <hf id> --data data/ft --out models/lora --check-only
python -m healthcoach.finetune.train_lora --base-model <hf id> --data data/ft --out models/lora
python -m healthcoach.finetune.merge_export --base-model <hf id> --adapter models/lora/adapter --out models/merged
```

### Evaluate (both paths): baseline first, then tuned
```
HEALTHCOACH_GGUF=models/gguf/base-q4_k_m.gguf python -m healthcoach.finetune.eval_generation \
    --data data/ft --llm healthcoach.llm:default_llm --report evals/reports/base.json
HEALTHCOACH_GGUF=models/gguf/tuned-q4_k_m.gguf python -m healthcoach.finetune.eval_generation \
    --data data/ft --llm healthcoach.llm:default_llm --report evals/reports/tuned.json --baseline evals/reports/base.json
HEALTHCOACH_GGUF=models/gguf/tuned-q4_k_m.gguf python evals/run_evals.py --index models/kb --llm healthcoach.llm:default_llm
```

## Rules of the road
- Final tier = max(rules, classifier). The LLM can explain a tier but never lower it.
- Emergency, crisis, and out-of-scope responses never wait on the LLM.
- Every rule change must keep `python evals/run_evals.py` passing; add a case for each new rule.
- Never commit the ER dataset or patient-level derivatives.
