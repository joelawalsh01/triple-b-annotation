# GEPA prompt optimization plan

## Goal

Use GEPA (reflective prompt evolution) to optimize prompts for this project. Two candidate optimization targets:

1. **Aya 8B translation prompt** — improve Spanish translation quality / NGSS standard preservation against the rater data we now have.
2. **`czi_connector.py` standards-matching prompt** — tighten the candidate-tag step so it aligns better with human source-confirmation. Currently the rater step rejects ~half of Opus v1's tags (rater 2: 55% No; rater 1: 44% No, κ between raters = 0.243).

Pick (1) first — translation is the project's central artifact and the rater data is more directly applicable.

## Why not Opus 4.6 as the reflector

Rough estimate for a standard GEPA run with Opus 4.6 as both executor and reflector, prompt-cached: **~$50–$100**. Aggressive runs land at $120–$220. Tight pilot is ~$15–$25. Too expensive for the value given the 82-passage training set is small enough that GEPA likely plateaus before the budget is well-spent.

## Reflector model picks (open weights via Ollama)

Already have: gpt-oss 120b.

Top three to try, ranked:

1. **Qwen 2.5 72B Instruct** *(first pick)* — best reasoning-per-VRAM, explicitly multilingual (Spanish matters because the reflector evaluates Aya's Spanish output against English source + NGSS standards). Beats gpt-oss 120b on most reflection-relevant benchmarks. `ollama pull qwen2.5:72b-instruct` (~40GB Q4).
2. **DeepSeek-V3 / V3.1** (or **DeepSeek-R1** for explicit reasoning behavior) — strongest open reasoning model in this size class. R1's think-before-answer pattern is a natural fit for GEPA's reflection step. Heavy footprint; only attempt if Qwen plateaus.
3. **Command R+ 104B** — same family as Aya, so its failure-mode model is likely compatible. Multilingual-strong on Aya-trained languages. Niche but worth a comparison run.

## Training data ("current numbers")

- **Rater 2**: 82 non-zero-std pages, 240 source-confirm decisions (109 Yes / 131 No), all 16 zero-std passages confirmed "no standard applies." Per-translation quality is forced-rank but uses an absolute scale that doesn't align with rater 1 — treat as descriptive only for now.
- **Rater 1 (Ashley)**: 25 pages so far (78 source-confirm decisions, 44 Yes / 34 No); ~67 pages of pre-existing quality ratings carry the duplicated-annotations bug from `annotated_data_done.json`.
- **Inter-rater overlap**: 78 shared standards across 24 pages. Both-agree subset = 48 standards (24 Yes-Yes + 24 No-No). This is the cleanest supervision signal; the 30 disagreements are noise from the rater perspective.

For the standards-matching target, supervise on the both-agree subset to avoid tuning toward one rater's permissiveness baseline. Wait until Ashley reaches ~50–60 pages so the agreement set is bigger (estimate ~120 both-agree standards by then).

For the translation target, use rater 2's preservation ratings (327 ratings across all 82 non-zero-std pages) as the dense per-(standard, translation) signal. Ashley's preservation data adds 129 more on the pages she's done.

## Budget estimates (open-model setup)

Local compute is essentially free past hardware costs. Budget concern shifts to wall-clock:

| Run | Rollouts | Reflections | Approx wall-clock on a single 80GB-class GPU |
|-----|----------|-------------|----------------------------------------------|
| Tight pilot | ~400 | ~30 | a few hours |
| Standard | ~1,200 | ~120 | overnight |
| Aggressive | ~2,500 | ~250 | 1–2 days |

Wall-clock dominated by the 70B+ reflector. Aya 8B as executor runs fast.

## Setup notes

- DSPy ships a GEPA optimizer; talks to Ollama via the OpenAI-compatible endpoint at `http://localhost:11434/v1`. No code changes beyond pointing `dspy.LM` at it.
- Run executor (Aya 8B) and reflector on the same Ollama instance. Aya 8B is small enough to coexist if there's headroom.
- Keep training set ≤50 passages and reflector context short. Open-model reflection quality degrades faster at long contexts than Opus's — don't dump the full standards list into every reflection step; summarize.
- Hold out ~20 passages for validation. If GEPA's optimized prompt doesn't beat the current baseline on the held-out set, the score is the score; longer runs won't fix it.

## Open questions / decisions before running

- Translation prompt vs standards-matching prompt as first target. Plan recommends translation.
- Reflector pick. Plan recommends Qwen 2.5 72B Instruct.
- Whether to wait for Ashley to reach ~50–60 pages before running (gives larger both-agree training set). Worth waiting if standards-matching is the target; less critical if translation is the target.
- How to score translations (preservation rating proxy via judge model, or direct rater-label match).
