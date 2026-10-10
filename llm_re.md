# Getting LLM-Assisted Reverse Engineering Right: Why Same-Model Voting Fails and What to Use Instead

The premise that "many agents normalize on a usually-correct answer" is mostly false for the setup you are considering: running several copies of the same model produces correlated errors, so majority voting mainly makes a shared mistake look more confident, and on hard functions it can make answers worse. What actually produces correct RE analysis is a proposer whose claims are checked against oracles that do not share the model's blind spots: the debugger, symbols, signatures, diffing, and re-execution. A verifier from a different model family adds a smaller second layer of protection.

## TL;DR

- **Same-model voting cancels noise, not blind spots.** Across 350+ LLMs, when two models both err they pick the same wrong answer 60% of the time (Kim et al., ICML 2025), and the more accurate the models, the more their errors correlate. In multi-step agent tasks, voting adds only +0–2 points, against +5–17 for single-turn chain-of-thought, because agent errors are systematic: Claude Sonnet 4.5 went from 81.5% with one run to 83.5% with a 10-run vote, and Gemini 3 Pro fell 0.4 points. For Claude Sonnet 4.5, 61% of its errors were "consistent-wrong", meaning the runs agreed on the wrong answer.
- **Verification beats voting only when the verifier has independent information.** LLMs mostly cannot find their own reasoning errors (Huang et al., ICLR 2024; Tyen et al., ACL 2024), but they correct well once told where the error is. Tool-grounded critique works (CRITIC, ICLR 2024). Debate between copies of one model rarely beats plain self-consistency. A verifier from a different model family helps somewhat, but frontier models increasingly share failure modes.
- **For RE specifically, the oracle is the verifier.** LLM symbol recovery on unseen stripped code is weak. On REBench, most models score F1 below 0.1. ReSym's exact-match accuracy on unseen functions is 37.5% for names and 49.9% for types. LLM decompiler "refinement" raises Ghidra's build rate from 75% to 90% while lowering semantic match from 74% to 62%. Run one strong agent on one function or cluster at a time. Have a separate adversarial verifier, preferably from another model family, try to refute each claim using WinDbg, PDBs, FLIRT/Lumina and BinDiff. Use agreement between runs only as an abstention signal, never as proof.

## Key Findings

### 1. Same-model ensembles: errors are correlated, and voting can backfire

**Errors across models and samples are correlated.**
- *Correlated Errors in Large Language Models* (Kim, Garg, Peng, Garg; ICML 2025, arXiv 2506.07962) evaluated more than 350 LLMs. "On one leaderboard dataset, models agree 60% of the time when both models err."
  - Correlation is driven by shared architecture and shared provider.
  - "Larger and more accurate models have highly correlated errors, even with distinct architectures and providers."
  - Multiple samples from a single model are the extreme case of shared architecture and provider, so you should expect their errors to correlate even more strongly than errors across these model pairs.
- *Great Models Think Alike and this Undermines AI Oversight* (Goel et al., ICML 2025, arXiv 2502.04313) proposed CAPA, a chance-adjusted measure of how much two models' mistakes overlap.
  - Model mistakes become more similar as capability rises.
  - LLM judges favor models similar to themselves. Across 9 judges and 39 models on MMLU-Pro, the paper reports "a significant (p < 0.01) positive correlation (average Pearson r=0.84) between LLM-as-a-judge scores and model similarity (κp) for all judges."

**More votes is not monotonically better.**
- *Are More LLM Calls All You Need?* (Chen et al., Stanford/Berkeley, arXiv 2403.02419) found that Vote and Filter-Vote performance is often non-monotonic in the number of calls.
  - "More LM calls lead to higher performance on 'easy' queries, but lower performance on 'hard' queries."
  - The reason: on a hard query, the model's most likely single answer is wrong, and voting converges on it.
- *When Self-Consistency Backfires* (arXiv 2608.11403, August 2026; small open models) tested this on GPQA Diamond.
  - Majority voting lowered per-problem accuracy on 56.6% of problems for Qwen2.5-7B and 65.7% for Llama-3-8B.
  - Aggregate accuracy barely moved (0.342 → 0.369 for Qwen).
  - "High self-consistency is therefore confidently wrong about as often as confidently right."
  - These are small models, so the result does not directly describe frontier models. The mechanism does generalize to any function where the model's dominant interpretation is wrong.

**Agentic settings are the worst case for voting.** This is the result closest to a Claude Code RE workflow.
- *When Agents Disagree With Themselves* (Mehta, Snowflake; ICML 2026 workshop, arXiv 2602.11619) ran 8,000 ReAct-agent runs: Claude Sonnet 4.5, GPT-5, Gemini 3 Pro and Llama 3.1 70B, 10 runs each on 200 HotpotQA questions at temperature 0.7.
  - "Majority voting yields only +0–2pp (vs. +5–17pp for single-turn CoT) because errors are systematic: an early wrong action commits the agent to an incorrect trajectory." In Mehta's Table 5, Claude Sonnet 4.5 went from 81.5% at k=1 to 82.2% at k=3 (+0.7 points) and 83.5% at k=10 (+2.0 points), and Gemini 3 Pro dropped to 71.7% at k=10 (−0.4 points).
  - "61% of Claude's errors are consistent-wrong vs. only 25% of Llama's." The strongest, most consistent model is the one voting helps least.
  - On hard tasks, 52 of 61 consistent task–model pairs (85.2%) were scored wrong, though Mehta's single-annotator check found "approximately 30%" of those were surface-form false negatives (e.g., "Pasek and Paul" vs. gold "Pasek & Paul"); scored by token F1 ≥ 0.5 instead, all four models still show Hard-task consistency gaps of +9 to +16 points.
  - The same paper found the useful alternative: treat agreement as a *filter*. Answering only when all k = 3 runs agree gave 87–88% accuracy at 54–62% coverage, a gain of 6–14 points.
  - Agreement between runs detected failures better than the model's verbalized confidence (AUROC 0.65–0.74 vs. 0.48–0.55).
- *When LLMs Agree, Are They Right?* (arXiv 2607.08065) found that self-consistency is a positive but weak correlate of correctness: no cell exceeds ρ = 0.6. GPT-4.1 had the highest mean agreement (0.89) but the lowest GPQA correlation (0.20). I could only read this paper's search snippet, not its full text.

**When same-model voting helps:**
- short, single-step answers where errors come from random slips such as arithmetic or a misread constant;
- easy items, where the model's most likely answer is already correct.

**When it hurts or only adds confidence:**
- hard items where the model's prior points the wrong way;
- multi-step agent trajectories that commit early;
- strong, low-variance models whose errors are mostly consistent.

Most "what does this undocumented kernel routine do" questions fall into the second group.

### 2. Cross-model and diverse ensembles: diversity helps, but less than hoped

- **Mixing models is not automatically better.** *Rethinking Mixture-of-Agents* (Li, Lin, Xia, Jin; Princeton, arXiv 2502.00674) found that Self-MoA, which aggregates samples from only the single best model, beat mixed-model MoA by 6.6% on AlpacaEval 2.0 and 3.8% on average across MMLU, CRUX and MATH.
  - Reason: "mixing different LLMs often lowers the average quality."
  - Mixing helps only when the models are close in quality and genuinely diverse.
  - The original MoA (Wang et al., ICLR 2025) reached 65.1% on AlpacaEval 2.0 using only open-source models, but its benchmarks reward synthesis, not refutation.
- **Heterogeneity is the one lever that reliably helps debate.** *Stop Overvaluing Multi-Agent Debate* (arXiv 2502.08788) tested 5 debate methods × 9 benchmarks × 4 models.
  - Debate "often fail[s] to outperform simple single-agent baselines such as Chain-of-Thought and Self-Consistency, even when consuming significantly more inference-time computation."
  - The authors' remedy is model heterogeneity (Heter-MAD).
  - Huang et al. (ICLR 2024) likewise found that, at an equal number of responses, debate between copies of one model underperforms plain majority-vote self-consistency. What gains it shows come from "self-consistency," not from "self-correction."
- **The limit of diversity.** Because frontier models' errors converge (Kim et al.; Goel et al.), a verifier from another vendor removes some blind spots but not all of them. Kim et al. note that correlation persists "even with distinct architectures and providers." Treat a different-family verifier as partial decorrelation, not independence.

**Implication for RE:** a cross-model verifier is worth having mainly because it is *less* correlated with the proposer, and it is most valuable when it is also handed different evidence. The decorrelation that matters most is a different information source (debugger trace, PDB, signature hit), not a different set of weights.

### 3. Verification versus voting: verification wins only with independent signal

- **Intrinsic self-correction does not work and can hurt.** Huang et al., *Large Language Models Cannot Self-Correct Reasoning Yet* (Google DeepMind, ICLR 2024, arXiv 2310.01798): "LLMs struggle to self-correct their responses without external feedback, and at times, their performance even degrades after self-correction." In their tests, accuracy fell across all benchmarks after self-correction.
- **The bottleneck is finding errors, not fixing them.** Tyen et al., *LLMs cannot find reasoning errors, but can correct them given the error location* (ACL Findings 2024): models "generally struggle" to locate logical mistakes "even in highly objective, unambiguous cases." Given the error location, however, their "correction abilities are robust."
  - This is the most actionable result for RE. An oracle that says "the debugger shows this branch is never taken" or "the PDB says this field is at +0x38, not +0x30" supplies exactly the error location the model cannot find on its own.
- **Tool-grounded critique works.** CRITIC (Gou et al., ICLR 2024, arXiv 2305.11738) finds that "feedback from external tool interaction is crucial for consistent self-improvement" and highlights "the unreliability of LLMs in self-verification." The TACL critical survey *When Can LLMs Actually Correct Their Own Mistakes?* reaches the same conclusion: self-correction succeeds with reliable external feedback and generally fails without it.
- **Adversarial separation matters, but only when grounded.** In *When Helping Hurts and How to Fix It* (arXiv 2606.02866, Claude 4 Sonnet, data cleaning):
  - Majority voting over 5 samples (F1 0.176) did *worse* than a single pass (0.191).
  - "Self-verification with identical tools fails, while a separate adversarial Critic succeeds."
  - "Code-execution grounding with evidence-gated generation" beat the single agent.
  - This is a single, recent, non-RE study, but its design is close to the proposer/refuter split recommended here.
- **Generator–verifier gap.** *Large Language Monkeys* (Brown et al., arXiv 2407.21787) showed coverage, the chance that at least one sample is correct, grows steadily with sample count. SWE-bench Lite rose from 15.9% solved with 1 sample to 56% with 250 samples.
  - That gain is only realized with an automatic verifier such as unit tests.
  - "In domains without automatic verifiers... majority voting and reward models plateau beyond several hundred samples."
  - RE sits in between. Some claims are mechanically checkable (struct offsets, call targets, return values, IOCTL codes, whether a WFP callout is registered), and for those, sampling plus an oracle is very effective. Semantic claims ("this function validates the caller's token") are not mechanically checkable, so you need to break them into checkable sub-claims.

**Where verification is easier than generation in RE:** confirming a struct offset by watching a memory access in WinDbg; checking a call graph; confirming a name against a PDB or a FLIRT/Lumina hit; confirming that a function compiles and matches behavior; diffing against a symbolized build.

**Where it is not:** intent, security implications, and anything that requires the same global understanding the generator lacked. For these, an LLM verifier is roughly as blind as the proposer.

### 4. Ground-truth anchoring in RE: the evidence

**LLM symbol recovery on genuinely unseen stripped code is weak, and benchmarks flatter it.**

- *REBench* (Won, Jin, Ma, Lin; Ohio State, arXiv 2604.27319, 2026) tested seven 13–15B models (CodeLlama 13B, Llama2 13B, WizardCoder 15B, Phi4 Unsloth 14B, Qwen2.5 14B, DeepSeek-R1-Qwen 14B and Qwen3 14B), LoRA-fine-tuned on 160,000 Ghidra-decompiled functions. "Across all tasks, most LLMs demonstrate limited capability, with the majority achieving F1 scores below 0.1."
  - Best function-name F1 was 0.3816 on x86 (Qwen3, O3) and 0.1617 on x64 (Phi4 Unsloth, O3).
  - Variable names fell from about 0.19–0.24 at O0 to "around 0.05" at O1–O3 on x64.
  - No model exceeded a type-inference F1 of 0.0589 (Phi4 Unsloth, MIPS, O1).
  - On 500 real ARM firmware functions, F1 was "near-zero" for function names and types.
  - Models trained on Ghidra output did not transfer to IDA output.
- *ReSym* (Xie et al., CCS 2024, Distinguished Paper) combines fine-tuned LLMs with program analysis and a Prolog step that cross-checks multiple LLM queries.
  - Variable exact-match: 56.4% names and 64.6% types overall, but **88.2%/89.5% on functions seen in training versus 37.5%/49.9% on unseen ones.** That gap reflects memorization of seen functions.
  - The cross-checking step raised struct-layout recall (16.8 → 28.9) and F1 (27.4 → 41.4) but slightly *lowered* precision (74.9 → 72.9). Aggregation found more structure; it did not make individual claims more trustworthy.
  - An ungrounded end-to-end model "altered the original semantics" in 68% of 100 sampled outputs.
  - GPT-4 few-shot reached only 18.7% on names and 30.6% on types.
- *SymGen* (Jiang, Jin, Lin; NDSS 2025) fine-tuned code LLMs on 2,237,915 functions across four ISAs and O0–O3. It reports "up to 409.3%, 553.5%, and 489.4%" gains in precision, recall and F1 over prior work. These are relative gains over weak baselines, not evidence that names are reliable.
- *GenNm* (Xu et al., NDSS 2025; earlier versions of the same arXiv paper were titled LmPa) uses program analysis to validate name consistency across callers and callees.
  - In the hardest setting, where ground-truth names never appeared in training, accuracy rose only from 17.3% to 22.8%.
  - The LmPa version's user study rated 75% of its names "good" against 6% for DIRTY, but automatic precision was only 33.85%.
- *BinSum* (Jin et al., arXiv 2312.09601; 557,664 functions) found that stripping symbols cut summary semantic similarity "from 0.449 to 0.202 (55.0% decrease)."
  - Removing only function names cut it by 30.3%.
  - Summaries of stripped code have "only partial or elementary semantics."
  - Summaries "are susceptible to malicious manipulation through intentional alterations of function names." A misleading symbol misleads the model.
- *When LLM Decompilers Recompile More and Preserve Less* (Liu, Raff, Micinski; arXiv 2609.05370, September 2026) is the starkest hallucination result.
  - On 291 real GitHub library functions, the strongest refinement LLM raised Ghidra's build rate from 75% to 90% while its behavior-matched rate fell from 74% to 62%.
  - Among candidates that passed every shipped test, 4.9% still diverged under fuzzing, up to 13% for one system.
  - On 287 CVE-grounded functions, 8.7% of outputs lost the crash, meaning the vulnerability disappeared.
  - Outputs that introduced invented fields, types or callees diverged 53% of the time versus 20% for outputs that did not (odds ratio 4.6).
  - The system that stayed closest to the decompiler, DeGPT-style in-place editing, stayed at or below 5% divergence with 0% crash loss.
  - **Lesson: "it compiles" and "it reads better" are not correctness oracles, and every symbol the LLM introduces needs a check.**
- *DeGPT* (Hu, Liang, Chen; NDSS 2024) improves readability (reported as a 24.4% cognitive-load reduction) and includes an explicit check that function semantics are not changed. That check is the reason it held up in the divergence study.
- *Reveree* (Benkraouda, Cai, Celik, Wang; arXiv 2609.01185, September 2026) tested 9 frontier Claude and GPT-5 models with 4 prompting strategies on 88 CTF RE challenges.
  - "The base model dominates performance, whereas prompting strategy is a secondary, model-dependent effect." Reflexion-style self-reflection is no fix.
  - Failures "concentrate at the comprehension stages," and "extra budget, persistence, or reasoning effort rescues few of them, pointing to a competence limit rather than a resource limit."
  - Successful runs pivot from analysis into *building a solver*, an executable artifact, instead of re-analyzing.
  - Models reproduce picoCTF flags from challenge descriptions alone, so memorization is real.
- *Automatically Attacking Software Reverse Engineering AI Agents* (Naval Postgraduate School, arXiv 2605.30667) showed that strings embedded in a binary can prompt-inject GhidraMCP-style agents into misreading it. For malware or adversarial drivers, the binary itself is an untrusted input to your agent.

**Grounding mechanisms that work in practice.** The academic literature on these is thin; most of the support is established RE practice.
- **Exact-symbol oracles:** PDBs (Microsoft's public symbol server covers ntoskrnl, fltmgr, netio/WFP components, tcpip and many inbox drivers), FLIRT and Lumina signatures, and BinDiff/Diaphora porting names from a symbolized build or an adjacent version.
- **Behavioral oracles:** WinDbg breakpoints, memory-access watchpoints, ETW traces, and recompile-and-run or emulation checks.

These oracles do something no ensemble can: they turn a semantic claim into a check whose errors are independent of the model's errors. I found no peer-reviewed study that measures LLM accuracy with and without FLIRT, Lumina or BinDiff grounding specifically. ReSym, GenNm and DeGPT are the closest evidence that program-analysis cross-checks help.

### 5. Focus and context: keep each agent's context small

- *Lost in the Middle* (Liu et al., TACL 2024): accuracy follows a U shape with position. It is highest when the relevant information is at the start or end of the context and "significantly degrades" when it is in the middle, and "performance substantially decreases as the input context grows longer, even for explicitly long-context models."
- Chroma's *Context Rot* report (Hong et al., July 2025) tested 18 frontier models, including Claude 4, GPT-4.1 and Gemini 2.5.
  - All of them degraded as input length grew, even on simple tasks, long before the context window was full.
  - Distractors (semantically similar but irrelevant content) caused extra degradation beyond what length alone explains.
  - This report is an industry technical report, not peer-reviewed, but it has been widely replicated informally.
- **Why this matters for RE:** decompiler output is full of near-duplicate distractors: similar dispatch routines, inlined helpers, IRP handlers that look alike, `sub_14000XXXX` placeholders.
- **Practical implications:**
  - Analyze one function or tight cluster per agent context: target, direct callers and callees, and the relevant types.
  - Keep the tool surface narrow and read-only by default.
  - Keep durable state (hypothesis ledger, confirmed symbols) outside the context and re-inject only what was *confirmed*.
  - Discard failed exploration rather than compacting it into the working context. This is also what Claude Code subagents are good for: isolating context, not voting.
- **Caveat:** Reveree found prompting strategy, including a hypothesis ledger, to be a secondary effect. Context hygiene reduces avoidable errors but will not lift a model past its comprehension limit.

## Recommendations

**Compare the three architectures:**

| Setup | What it fixes | What it cannot fix | Verdict for RE |
|---|---|---|---|
| Same-model N-sample voting | Random slips on easy, single-step items | Shared priors, early trajectory commitment, hard functions; can *lower* accuracy (Chen et al.; 2608.11403) | Use only as an **abstention filter**: unanimous runs mean "proceed to verification"; disagreement means "flag for human" |
| Single strong agent + context-isolated subagents | Context rot and distraction; keeps each function analysis focused | Model's comprehension limits; self-verification blindness | **Good base layer**; subagents are for decomposition, not for consensus |
| Proposer + different-family adversarial verifier anchored on oracles | Converts claims into independent checks; supplies error locations the model can't find itself (Tyen et al.) | Unverifiable intent-level claims | **Best correctness per dollar**; the oracle does most of the work, the second model family adds partial decorrelation |

**Concrete operating procedure:**
1. **Anchor first.** Before any LLM reasoning, apply PDBs, FLIRT/Lumina, and BinDiff/Diaphora from the nearest symbolized build. Remember that BinSum showed function names alone account for about 30% of summary quality. Never let the LLM rename something that already has a ground-truth symbol.
2. **Decompose claims.** Make the proposer output atomic, checkable claims: "arg2 is a `FWPS_INCOMING_VALUES0*`", "returns STATUS_ACCESS_DENIED when X", "field +0x48 is a LIST_ENTRY". Do not accept a free-form story as output.
3. **Refute, don't re-derive.** Give the verifier, a different model family where feasible, the claims plus oracle access (WinDbg scripts, symbol lookups, xrefs). Instruct it to *find evidence each claim is false*. A claim is "confirmed" only by oracle evidence, never by the verifier's agreement.
4. **Gate invented symbols.** Any field, type or callee the LLM introduces gets a mandatory oracle check. The 2609.05370 data show that introduced symbols are where semantic divergence concentrates.
5. **Use agreement as triage.** Run k = 3 cheap proposer samples. If they disagree, route the function to deeper analysis or a human; if they agree, still verify. Mehta's data show consistent-wrong answers are a hard floor that filtering cannot remove.
6. **Treat binary content as untrusted.** Sanitize strings before they reach the agent's instructions, especially for malware and third-party drivers.
7. **Track model versions.** Error correlations, model rankings and context behavior change with each release. Reveree found that newer or costlier models are not reliably stronger.

**Cheap experiment: measuring your own correlated-error and confident-wrong rate on about 20 functions**

- **Corpus (20 functions, ground-truthable):**
  - 10 from your own or open-source drivers compiled with a PDB, analyzed stripped. These are low-contamination.
  - 10 from Windows binaries with public PDBs (e.g., netio.sys, fltmgr.sys), analyzed with symbols withheld. Expect contamination here: the model may recall names from training. Report the two halves separately; the gap between them is your memorization estimate, mirroring ReSym's 88% seen vs. 37.5% unseen.
  - Include roughly 5 "hard" ones: optimized, inlined, or with indirect calls.
- **Ground truth:** real name, prototype and types from the PDB, plus 3–5 behavioral facts per function confirmed in WinDbg, for about 80 atomic claims in total.
- **Runs:**
  - (A) Claude, 5 independent samples per function, fresh context each time.
  - (B) a second model family, 5 samples.
  - (C) Claude proposer plus a different-family refuter with oracle access.
  - (D) the same as C, but the refuter has no oracle access.
- **Metrics:**
  1. Per-sample accuracy on names (semantic match), types (exact) and behavior claims.
  2. Majority-vote accuracy versus single-sample accuracy: does voting help, nothing, or hurt?
  3. **Confident-wrong rate:** the share of functions where 4 or 5 of the 5 samples agree *and* are wrong.
  4. **Same-wrong-answer rate:** when two samples or two models are both wrong, how often they give the same wrong answer (Kim et al.'s 60% metric). Compare within-Claude, within-model-B, and cross-model.
  5. Verifier catch rate (wrong claims rejected) and false-reject rate (correct claims rejected) for C versus D. The difference isolates the oracle's value from the second model's.
- **Decision rule:**
  - If within-Claude same-wrong-answer agreement is high (above about 50%) and the confident-wrong rate is not near zero, voting is laundering errors, and your budget belongs in C.
  - If D catches few errors that C catches, the second model adds little and the oracle is doing the work.
- **Statistics caveat:** with 20 functions, a 95% confidence interval on a proportion is roughly ±20 points. Treat the results as directional, pool at the claim level (about 80 claims) for tighter estimates, and repeat after major model updates.

## Caveats

- **Most correlation and voting evidence is general-LLM, not RE-specific.** The tasks are MCQ, GPQA, HotpotQA and data cleaning. I found no study that directly measures same-model error correlation on binary analysis; the experiment above is meant to fill that gap for your workload.
- **Several key papers are recent preprints and not yet peer-reviewed:** 2608.11403, 2607.08065, 2609.05370, 2609.01185, 2606.02866 and 2604.27319, all dated 2026. The 2607.08065 numbers come from a search snippet only, not the full text.
- **RE benchmarks mostly use Linux/ELF, GNU projects, or CTFs.** Windows kernel code (PE, MSVC, heavy use of SEH and IRQL-dependent paths, WDM/KMDF/WFP callback patterns) is underrepresented. REBench's finding that models transfer poorly from Ghidra to IDA output suggests results may not carry over to your toolchain.
- **Fast-changing:** model capabilities, inter-model error correlation, Claude Code subagent features, and MCP integrations with IDA, Ghidra and WinDbg all change from release to release. Re-validate any conclusion here against current models, as of October 2026.
- **Not every cross-model or debate result is negative.** Heterogeneous debate and MoA can help when the models are close in quality and diverse. The claim here is narrower: neither replaces an oracle for correctness.
