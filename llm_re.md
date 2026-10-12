# Getting LLM-Assisted Reverse Engineering Right: Consistency, Focus, and Correctness

*Companion to the multi-agent architecture doc. Last updated 2026-10-11.*

The intuition that "many agents normalize on a usually-correct answer" is mostly **false** for same-model ensembles: running several copies of one model produces *correlated* errors, so majority voting mainly makes a shared mistake look more confident, and on hard functions it can make answers worse. What actually produces correct RE analysis is a **proposer whose claims are checked against oracles** that do not share the model's blind spots — the debugger, symbols, signatures, diffing, re-execution. A verifier from a different model family adds a smaller second layer.

---

## TL;DR

- **Same-model voting cancels noise, not blind spots.** Across 350+ LLMs, when two models both err they pick the *same* wrong answer ~60% of the time, and more accurate models' errors correlate *more*. In multi-step agent tasks, voting adds only +0–2 points (vs +5–17 for single-turn chain-of-thought) because agent errors are systematic. Claude Sonnet 4.5 went 81.5% → 83.5% from a 10-run vote; 61% of its errors were "consistent-wrong" (the runs agreed and were wrong).
- **Verification beats voting only when the verifier has independent information.** LLMs mostly can't find their own reasoning errors, but correct well once *told where* the error is. An oracle supplies exactly that location. Debate between copies of one model rarely beats plain self-consistency; a different model family helps somewhat, but frontier models increasingly share failure modes.
- **For RE, the oracle is the verifier.** LLM symbol recovery on unseen stripped code is weak (REBench: most models F1 < 0.1; ReSym exact-match 37.5% names / 49.9% types on *unseen* functions). LLM decompiler "refinement" can raise build rate while *lowering* correctness. Run one strong agent on one function/cluster at a time; have a separate adversarial verifier — preferably a different model family — try to *refute* each claim using WinDbg, PDBs, FLIRT/Lumina and BinDiff; use agreement between runs only as an abstention signal, never as proof.

---

## 1. Same-model ensembles: errors are correlated, and voting can backfire

**Errors across models and samples are correlated.**

- *Correlated Errors in Large Language Models* (Kim et al., ICML 2025, [arXiv 2506.07962](https://arxiv.org/abs/2506.07962)) evaluated 350+ LLMs: on one leaderboard, models agreed ~60% of the time **when both were wrong**. Correlation is driven by shared architecture and provider, and **larger, more accurate models have more correlated errors**, even across providers. Multiple samples from *one* model are the extreme case of shared architecture/provider — expect their errors to correlate even more strongly.
- *Great Models Think Alike and this Undermines AI Oversight* (Goel et al., ICML 2025, [arXiv 2502.04313](https://arxiv.org/abs/2502.04313)): model mistakes grow more similar as capability rises, and LLM judges favor models similar to themselves (avg Pearson r = 0.84 between judge scores and model similarity).

**More votes is not monotonically better.**

- *Are More LLM Calls All You Need?* (Chen et al., [arXiv 2403.02419](https://arxiv.org/abs/2403.02419)): voting performance is often *non-monotonic* in the number of calls. More calls help on easy queries but **hurt on hard ones** — on a hard query the single most-likely answer is wrong, and voting converges on it.
- *When Self-Consistency Backfires* ([arXiv 2608.11403](https://arxiv.org/abs/2608.11403), small open models): majority voting lowered per-problem accuracy on 56.6% (Qwen2.5-7B) and 65.7% (Llama-3-8B) of problems while aggregate accuracy barely moved. "High self-consistency is confidently wrong about as often as confidently right." (Small models — the *mechanism* generalizes; the magnitudes may not.)

**Agentic settings are the worst case for voting** — the closest analogue to a Claude Code RE workflow.

- *When Agents Disagree With Themselves* (Mehta, [arXiv 2602.11619](https://arxiv.org/abs/2602.11619)): 8,000 ReAct runs (Claude Sonnet 4.5, GPT-5, Gemini 3 Pro, Llama 3.1 70B), 10 runs each on 200 HotpotQA questions.
  - Majority voting: **+0–2 points** vs +5–17 for single-turn CoT, "because errors are systematic: an early wrong action commits the agent to an incorrect trajectory." Claude Sonnet 4.5: 81.5% (k=1) → 83.5% (k=10); Gemini 3 Pro *dropped* 0.4.
  - **61% of Claude's errors were "consistent-wrong"** vs 25% of Llama's — the strongest, most consistent model is the one voting helps least.
  - The useful alternative: treat agreement as a **filter**. Answering only when all k=3 runs agree gave 87–88% accuracy at 54–62% coverage (+6–14 points). Agreement detected failures better than verbalized confidence (AUROC 0.65–0.74 vs 0.48–0.55).

**When same-model voting helps:** short, single-step answers where errors are random slips (a misread constant, arithmetic); easy items where the top answer is already right.

**When it hurts or only adds confidence:** hard items where the model's prior points the wrong way; multi-step agent trajectories that commit early; strong, low-variance models whose errors are consistent.

> Most "what does this undocumented kernel routine do?" questions fall into the second group.

---

## 2. Cross-model and diverse ensembles: diversity helps, but less than hoped

- **Mixing models is not automatically better.** *Rethinking Mixture-of-Agents* (Li et al., [arXiv 2502.00674](https://arxiv.org/abs/2502.00674)): aggregating samples from only the single best model (Self-MoA) beat mixed-model MoA by 6.6% on AlpacaEval 2.0, because "mixing different LLMs often lowers the average quality." Mixing helps only when models are close in quality *and* genuinely diverse.
- **Heterogeneity is the one lever that reliably helps debate.** *Stop Overvaluing Multi-Agent Debate* ([arXiv 2502.08788](https://arxiv.org/abs/2502.08788)): debate often fails to beat plain chain-of-thought or self-consistency despite much higher cost; the remedy that works is model heterogeneity. Huang et al. (ICLR 2024) found debate between copies of one model underperforms plain majority-vote self-consistency at equal compute.
- **The limit of diversity.** Because frontier models' errors converge (Kim et al.; Goel et al.), a different-vendor verifier removes *some* blind spots, not all. Treat it as **partial decorrelation, not independence**.

> **Implication for RE:** a cross-model verifier is worth having because it is *less* correlated with the proposer, and it is most valuable when handed *different evidence*. The decorrelation that matters most is a different information source (debugger trace, PDB, signature hit), not a different set of weights.

---

## 3. Verification vs voting: verification wins only with independent signal

- **Intrinsic self-correction doesn't work and can hurt.** Huang et al., *LLMs Cannot Self-Correct Reasoning Yet* (ICLR 2024, [arXiv 2310.01798](https://arxiv.org/abs/2310.01798)): without external feedback, self-correction left performance flat or *worse*.
- **The bottleneck is finding errors, not fixing them.** Tyen et al. (ACL Findings 2024, [arXiv 2311.08516](https://arxiv.org/abs/2311.08516)): models "generally struggle" to *locate* mistakes even in unambiguous cases, but given the error location their correction is robust. **An oracle that says "the PDB puts this field at +0x38, not +0x30" supplies exactly the location the model cannot find on its own.**
- **Tool-grounded critique works.** CRITIC (Gou et al., ICLR 2024, [arXiv 2305.11738](https://arxiv.org/abs/2305.11738)): external tool interaction is crucial for consistent self-improvement; LLM self-verification alone is unreliable.
- **Adversarial separation, grounded.** *When Helping Hurts* ([arXiv 2606.02866](https://arxiv.org/abs/2606.02866), data cleaning with Claude 4 Sonnet): 5-sample majority voting (F1 0.176) did *worse* than a single pass (0.191); "self-verification with identical tools fails, while a separate adversarial Critic succeeds"; code-execution grounding with evidence-gated generation beat the single agent. (One recent non-RE study, but its design is close to the proposer/refuter split recommended here.)
- **Generator–verifier gap.** *Large Language Monkeys* (Brown et al., [arXiv 2407.21787](https://arxiv.org/abs/2407.21787)): coverage (at least one correct sample) grows steadily with sample count — SWE-bench Lite 15.9% (1 sample) → 56% (250) — **but only realizable with an automatic verifier** such as unit tests. Without one, voting plateaus.

**RE sits in between.** Mechanically checkable claims (struct offsets, call targets, return values, IOCTL codes, whether a WFP callout is registered) are where sampling + an oracle is very effective. Semantic claims ("this function validates the caller's token") are not mechanically checkable — break them into checkable sub-claims.

- **Easier to verify than generate:** confirming a struct offset by watching a memory access in WinDbg; checking a call graph; confirming a name against a PDB or FLIRT/Lumina hit; confirming code compiles and matches behavior; diffing against a symbolized build.
- **Not easier:** intent, security implications, anything needing the same global understanding the generator lacked. Here an LLM verifier is roughly as blind as the proposer.

---

## 4. Ground-truth anchoring for RE: the evidence

**LLM symbol recovery on genuinely unseen stripped code is weak, and benchmarks flatter it.**

- *REBench* ([arXiv 2604.27319](https://arxiv.org/abs/2604.27319)): across seven 13–15B models fine-tuned on 160k Ghidra functions, "most LLMs demonstrate limited capability, with the majority achieving F1 scores below 0.1." Variable names collapsed from ~0.19–0.24 (O0) to ~0.05 (O1–O3); on real ARM firmware, F1 was near-zero. Models trained on Ghidra output did **not** transfer to IDA output.
- *ReSym* (Xie et al., CCS 2024, [paper](https://www.cs.purdue.edu/homes/lintan/publications/resym-ccs24.pdf)): variable exact-match 88.2%/89.5% on **seen** functions vs **37.5%/49.9% on unseen** — the gap is memorization. A Prolog cross-check across multiple LLM queries raised struct-layout recall (16.8 → 28.9) but slightly *lowered* precision (74.9 → 72.9): aggregation found more structure, it did not make individual claims more trustworthy. An ungrounded end-to-end model "altered the original semantics" in 68% of sampled outputs.
- *GenNm/LmPa* (NDSS 2025, [arXiv 2306.02546](https://arxiv.org/abs/2306.02546)): in the hardest no-leakage setting, accuracy rose only 17.3% → 22.8%.
- *BinSum* ([arXiv 2312.09601](https://arxiv.org/abs/2312.09601)): stripping symbols cut summary semantic similarity 55%; function names alone account for ~30%; summaries are "susceptible to malicious manipulation" via altered names — **a misleading symbol misleads the model.**
- *When LLM Decompilers Recompile More and Preserve Less* ([arXiv 2609.05370](https://arxiv.org/abs/2609.05370)): the starkest hallucination result. The strongest refinement LLM raised Ghidra's build rate 75% → 90% while behavior-match *fell* 74% → 62%. Among candidates passing every shipped test, 4.9% still diverged under fuzzing. On CVE-grounded functions, 8.7% of outputs lost the crash (the vuln disappeared). Outputs that introduced invented fields/types/callees diverged 53% vs 20% (odds ratio 4.6). The system that stayed closest to the decompiler (DeGPT-style in-place editing) stayed ≤5% divergence with 0% crash loss. **Lesson: "it compiles" and "it reads better" are not correctness oracles, and every symbol the LLM introduces needs a check.**
- *Reveree* ([arXiv 2609.01185](https://arxiv.org/abs/2609.01185), 9 frontier Claude/GPT-5 models on 88 CTF RE challenges): "the base model dominates performance, whereas prompting strategy is a secondary, model-dependent effect." Reflexion-style self-reflection is no fix; failures concentrate at comprehension and "extra budget, persistence, or reasoning effort rescues few of them." Successful runs pivot into *building a solver* (an executable artifact) rather than re-analyzing.
- *Automatically Attacking Software RE AI Agents* ([arXiv 2605.30667](https://arxiv.org/abs/2605.30667)): strings embedded in a binary can prompt-inject GhidraMCP-style agents. **For malware or adversarial drivers, the binary itself is untrusted input to your agent.**

**Grounding mechanisms that work** (the academic literature here is thin; most support is established RE practice):

- *Exact-symbol oracles:* PDBs (Microsoft's public symbol server covers ntoskrnl, fltmgr, netio/WFP, tcpip and many inbox drivers), FLIRT/Lumina signatures, BinDiff/Diaphora porting names from a symbolized or adjacent build.
- *Behavioral oracles:* WinDbg breakpoints, memory-access watchpoints, ETW traces, recompile-and-run or emulation checks.

> These oracles do something no ensemble can: they turn a semantic claim into a check whose errors are **independent** of the model's errors.

---

## 5. Focus and context: keep each agent's context small

- *Lost in the Middle* (Liu et al., TACL 2024): accuracy follows a U-shape with position — highest when relevant info is at the start/end, "significantly degrades" in the middle, and drops as the input grows, even for long-context models.
- *Context Rot* (Chroma, 2025): 18 frontier models (incl. Claude 4, GPT-4.1, Gemini 2.5) all degraded as input length grew, before the window was full; semantically similar distractors caused extra degradation. (Industry report, widely replicated informally.)

**Why this bites in RE:** decompiler output is full of near-duplicate distractors — similar dispatch routines, inlined helpers, look-alike IRP handlers, `sub_14000XXXX` placeholders.

**Practical implications:**

- Analyze **one function or tight cluster per agent context**: target, direct callers/callees, relevant types.
- Keep the tool surface narrow and read-only by default.
- Keep durable state (hypothesis ledger, confirmed symbols) *outside* the context; re-inject only what was *confirmed*.
- Discard failed exploration rather than compacting it into the working context. (This is what subagents are for: isolating context, not voting.)

*Caveat:* Reveree found prompting strategy a secondary effect. Context hygiene removes avoidable errors; it won't lift a model past its comprehension limit.

---

## 6. Real-world instances of the right pattern

Two shipping tools have independently converged on "propose, then refute against ground truth":

- **Binary Ninja Sidekick 26.0** ships a *validation agent* that cross-checks findings against IL, assembly and memory traces, and presents hypotheses backed by evidence. Its multi-provider support (Anthropic/OpenAI plus local Ollama/vLLM) makes a **different-model verifier** practical, and local models keep malware off the cloud.
- **"Decompiler as a database"** (RECON 2026) exposes IDA/Ghidra/Binary Ninja over SQL with cross-tool annotation transfer — a clean substrate for porting *ground-truth* symbols onto an unknown binary (the strongest oracle there is).

Both still carry the standard warning: LLMs hallucinate names and even vulnerabilities, so verify.

---

## Recommendation

| Setup | What it fixes | What it cannot fix | Verdict for RE |
| --- | --- | --- | --- |
| Same-model N-sample voting | Random slips on easy, single-step items | Shared priors, early trajectory commitment, hard functions; can *lower* accuracy | Use only as an **abstention filter** — unanimous = proceed to verification, disagreement = flag for human |
| Single strong agent + context-isolated subagents | Context rot and distraction; keeps each analysis focused | Comprehension limits; self-verification blindness | **Good base layer**; subagents are for decomposition, not consensus |
| Proposer + different-family adversarial verifier anchored on oracles | Converts claims into independent checks; supplies the error locations the model can't find itself | Unverifiable intent-level claims | **Best correctness per dollar** — the oracle does most of the work, the second model family adds partial decorrelation |

**Operating procedure:**

1. **Anchor first.** Before any LLM reasoning, apply PDBs, FLIRT/Lumina, and BinDiff/Diaphora from the nearest symbolized build. Never let the LLM rename something that already has a ground-truth symbol.
2. **Decompose claims.** Make the proposer output atomic, checkable claims ("arg2 is a `FWPS_INCOMING_VALUES0*`", "returns STATUS_ACCESS_DENIED when X", "field +0x48 is a LIST_ENTRY"). Don't accept a free-form story.
3. **Refute, don't re-derive.** Give the verifier (different model family where feasible) the claims plus oracle access and instruct it to *find evidence each claim is false*. A claim is "confirmed" only by oracle evidence, never by the verifier's agreement.
4. **Gate invented symbols.** Any field/type/callee the LLM introduces gets a mandatory oracle check — that's where semantic divergence concentrates.
5. **Use agreement as triage.** Run k=3 cheap proposer samples; disagreement routes to deeper analysis or a human; agreement still gets verified. Consistent-wrong answers are a floor filtering cannot remove.
6. **Treat binary content as untrusted.** Sanitize strings before they reach the agent's instructions, especially for malware and third-party drivers.
7. **Track model versions.** Error correlations, rankings and context behavior shift each release; newer/costlier is not reliably stronger.

---

## Cheap experiment: measure your own correlated-error / confident-wrong rate (~20 functions)

**Corpus (20 functions, ground-truthable):**

- 10 from your own or open-source drivers compiled with a PDB, analyzed stripped (low contamination).
- 10 from Windows binaries with public PDBs (e.g. netio.sys, fltmgr.sys), analyzed with symbols withheld (expect contamination — the model may recall names). **Report the two halves separately; the gap is your memorization estimate** (mirrors ReSym's 88% seen vs 37.5% unseen).
- Include ~5 "hard" ones: optimized, inlined, or with indirect calls.

**Ground truth:** real name, prototype and types from the PDB, plus 3–5 behavioral facts per function confirmed in WinDbg (~80 atomic claims total).

**Runs:**

- **A** — Claude, 5 independent samples/function, fresh context each.
- **B** — a second model family, 5 samples.
- **C** — Claude proposer + different-family refuter *with* oracle access.
- **D** — same as C but refuter *without* oracle access.

**Metrics:**

1. Per-sample accuracy: names (semantic match), types (exact), behavior claims.
2. Majority-vote vs single-sample accuracy — does voting help, nothing, or hurt?
3. **Confident-wrong rate:** share of functions where 4–5 of 5 samples agree *and* are wrong.
4. **Same-wrong-answer rate:** when two samples/models are both wrong, how often they give the *same* wrong answer (Kim et al.'s 60% metric). Compare within-Claude, within-B, and cross-model.
5. Verifier catch rate (wrong claims rejected) and false-reject rate for C vs D — isolates the oracle's value from the second model's.

**Decision rule:**

- If within-Claude same-wrong agreement is high (>~50%) and confident-wrong isn't near zero → voting is laundering errors; spend the budget on C.
- If D catches few of the errors C catches → the second model adds little and the oracle is doing the work.

*Statistics caveat:* with 20 functions a 95% CI on a proportion is ~±20 points. Treat results as directional, pool at the claim level (~80 claims) for tighter estimates, and repeat after major model updates.

---

## Caveats

- **Most correlation/voting evidence is general-LLM, not RE-specific** (MCQ, GPQA, HotpotQA, data cleaning). No study directly measures same-model error correlation on binary analysis — the experiment above is meant to fill that gap for your workload.
- **Several key papers are recent (2026) preprints**, not yet peer-reviewed: 2608.11403, 2609.05370, 2609.01185, 2606.02866, 2604.27319.
- **RE benchmarks skew Linux/ELF, GNU, and CTFs.** Windows kernel code (PE, MSVC, SEH, IRQL-dependent paths, WDM/KMDF/WFP callbacks) is underrepresented; REBench's poor Ghidra→IDA transfer suggests results may not carry to your toolchain.
- **Fast-changing:** model capabilities, inter-model error correlation, and tool integrations all shift release to release. Re-validate against current models.
- **Not every cross-model/debate result is negative.** Heterogeneous debate and MoA can help when models are close in quality and diverse. The narrow claim here: neither replaces an oracle for correctness.

---

## Sources

- Correlated Errors in LLMs — Kim et al., ICML 2025 — https://arxiv.org/abs/2506.07962
- Great Models Think Alike (CAPA) — Goel et al., ICML 2025 — https://arxiv.org/abs/2502.04313
- Are More LLM Calls All You Need? — Chen et al. — https://arxiv.org/abs/2403.02419
- When Self-Consistency Backfires — https://arxiv.org/abs/2608.11403
- When Agents Disagree With Themselves — Mehta — https://arxiv.org/abs/2602.11619
- Rethinking Mixture-of-Agents (Self-MoA) — Li et al. — https://arxiv.org/abs/2502.00674
- Stop Overvaluing Multi-Agent Debate — https://arxiv.org/abs/2502.08788
- LLMs Cannot Self-Correct Reasoning Yet — Huang et al., ICLR 2024 — https://arxiv.org/abs/2310.01798
- LLMs Cannot Find Reasoning Errors, but Can Correct Them Given the Location — Tyen et al., ACL 2024 — https://arxiv.org/abs/2311.08516
- CRITIC — Gou et al., ICLR 2024 — https://arxiv.org/abs/2305.11738
- When Helping Hurts (adversarial Critic + grounding) — https://arxiv.org/abs/2606.02866
- Large Language Monkeys — Brown et al. — https://arxiv.org/abs/2407.21787
- REBench — https://arxiv.org/abs/2604.27319
- ReSym — Xie et al., CCS 2024 — https://www.cs.purdue.edu/homes/lintan/publications/resym-ccs24.pdf
- GenNm / LmPa — NDSS 2025 — https://arxiv.org/abs/2306.02546
- BinSum — https://arxiv.org/abs/2312.09601
- When LLM Decompilers Recompile More and Preserve Less — https://arxiv.org/abs/2609.05370
- Reveree — https://arxiv.org/abs/2609.01185
- Automatically Attacking Software RE AI Agents — https://arxiv.org/abs/2605.30667
- Lost in the Middle — Liu et al., TACL 2024 — https://arxiv.org/abs/2307.03172
- Binary Ninja Sidekick 26.0 — https://sidekick.binary.ninja/blog/sidekick-26-0-a-whole-new-experience-in-reversing-with-ai
