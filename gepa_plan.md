# GEPA prompt optimization plan

## Goal

Use GEPA (reflective prompt evolution) to optimize prompts for this project. Two candidate optimization targets:

1. **Aya 8B translation prompt** — improve Spanish translation quality / NGSS standard preservation against the rater data we now have.
2. **`czi_connector.py` standards-matching prompt** — tighten the candidate-tag step so it aligns better with human source-confirmation. Currently the rater step rejects ~half of Opus v1's tags (rater 2: 55% No; rater 1: 44% No, κ between raters = 0.243).

Pick (1) first — translation is the project's central artifact and the rater data is more directly applicable.

## Why not Opus 4.6 as the reflector

Rough estimate for a standard GEPA run with Opus 4.6 as both executor and reflector, prompt-cached: **~$50–$100**. Aggressive runs land at $120–$220. Tight pilot is ~$15–$25. Too expensive for the value given the 82-passage training set is small enough that GEPA likely plateaus before the budget is well-spent.

## Reflector model picks (open weights via Ollama)

Hardware: Dell GB10 (Grace Blackwell, 128 GB unified memory, FP4-native).

Already have: gpt-oss 120b.

Top picks, ranked:

1. **Qwen 3.6-27B dense** *(first pick)* — released April 2026, ~17 GB. Reportedly outperforms a 397B MoE on agentic coding benchmarks, which is the closest public proxy for the read-trace / diagnose / propose-mutation loop GEPA's reflector runs. 256K context (useful — GEPA reflection packs traces + history). 201-language multilingual coverage (the reflector evaluates Aya's Spanish output against English source + NGSS standards). Apache 2.0. `ollama pull qwen3.6:27b`.
2. **Qwen 3.6-35B-A3B** (MoE, 35B total / 3B active) — released April 2026, ~24 GB, Apache 2.0. Faster inference per token than the 27B dense thanks to 3B active params; smaller capability ceiling. Hold in reserve as a faster fallback if reflection latency becomes a bottleneck during long runs. `ollama pull qwen3.6:35b-a3b`.
3. **DeepSeek-V3.1** or **DeepSeek-R1** — strongest open reasoning in this size class. Heavy footprint but the GB10 handles it. Use as a tiebreaker run if Qwen 3.6-27B plateaus.

Rationale for 27B dense over the larger Qwen 3 235B-A22B-Thinking on this hardware: 17 GB leaves enormous memory headroom for batched rollouts and Aya 8B coexisting; dense outperforms small-active MoE on the short-prompt reflection calls GEPA actually makes; and the 27B's reasoning benchmarks beat Qwen 3 generation models head-to-head.

## Training data ("current numbers")

- **Rater 2**: 98/98 pages complete. 240 source-confirm decisions (109 Yes / 131 No, 45% Yes). 327 preservation ratings, 294 forced-rank quality labels. All 16 zero-std passages confirmed "no standard applies."
- **Rater 1 (Ashley)**: 63 pages with input (62 complete on the non-zero-std rubric). 186 source-confirm decisions (76 Yes / 110 No, 41% Yes). 228 preservation ratings, 181 forced-rank quality overrides. Has not yet started the 16 zero-standards passages.
- **Quality scale (Worst/Middle/Best) is now aligned**: both raters use forced-rank. Cohen's κ on per-translation labels rose from ≈0 (when rater 1 was using an absolute scale) to **0.183 unweighted / 0.347 quadratic-weighted** — fair agreement.
- **Asymmetric signal worth noting for the paper**: agreement on the *worst* translation per page is strong (68%), but agreement on the *best* is weak (35%). Frame quality claims as "least-likely-to-fail" rather than "most-likely-best."

For the **standards-matching** target, supervise on the both-agree subset (Yes-Yes ∪ No-No across the ~63 overlapping pages) to avoid tuning toward one rater's permissiveness baseline. Discard the disagreements as noise.

For the **translation** target, use rater 2's 327 preservation ratings as the dense per-(standard, translation) signal; Ashley's 228 add coverage on the overlapping pages. Forced-rank quality labels can serve as a coarser per-page model-comparison signal.

## Budget estimates (open-model setup on GB10)

Local compute is essentially free past hardware costs. Budget concern is wall-clock. With Qwen 3.6-27B as the reflector (~17 GB, dense, fast on Blackwell FP4), wall-clock is much shorter than the 70B+ pick this plan originally assumed:

| Run | Rollouts | Reflections | Approx wall-clock on GB10 with Qwen 3.6-27B |
|-----|----------|-------------|----------------------------------------------|
| Tight pilot | ~400 | ~30 | ~1 hour |
| Standard | ~1,200 | ~120 | a few hours |
| Aggressive | ~2,500 | ~250 | overnight |

If you swap the reflector for the 35B-A3B MoE, expect somewhat slower per-rollout wall-clock but lower memory pressure. DeepSeek-V3.1 / R1 as reflector pushes wall-clock back into the overnight range even for tight pilots.

## Setup notes

- DSPy ships a GEPA optimizer; talks to Ollama via the OpenAI-compatible endpoint at `http://localhost:11434/v1`. No code changes beyond pointing `dspy.LM` at it.
- Run executor (Aya 8B) and reflector on the same Ollama instance. Aya 8B is small enough to coexist if there's headroom.
- Keep training set ≤50 passages and reflector context short. Open-model reflection quality degrades faster at long contexts than Opus's — don't dump the full standards list into every reflection step; summarize.
- Hold out ~20 passages for validation. If GEPA's optimized prompt doesn't beat the current baseline on the held-out set, the score is the score; longer runs won't fix it.

## Open questions / decisions before running

- Translation prompt vs standards-matching prompt as first target. Plan recommends translation.
- Reflector pick. Plan recommends Qwen 3.6-27B dense.
- How to score translations (preservation rating proxy via judge model, or direct rater-label match).
- Whether to wait for Ashley to reach 98/98 (currently 63/98). The both-agree subset is already substantial at 63 overlapping pages; not a blocker for the translation target.
