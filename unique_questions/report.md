# Review of the revised Social Reasoning results

The new GitHub run materially improves the Social Reasoning analysis, particularly in response to Reviewer 2's concerns about repeated questions, non-independent observations, and multiple comparisons. It also changes the numerical results enough that the previous Social Reasoning tables, effect sizes, and interpretive statements must be updated together.

I reviewed the saved notebook outputs and `social_reasoning_baseline_revised.json` in [`shayanfirouzian/revise`](https://github.com/shayanfirouzian/revise?utm_source=chatgpt.com).

## 1. What the new run establishes

The evaluation completed for all four checkpoints, with 393 selected items per model and no reported scoring errors. The saved results confirm random sampling without replacement, using seed 42, from a dataset of 955 question–answer records.

| Model checkpoint      | Mean full-sequence loss | 95% bootstrap CI |
| --------------------- | ----------------------- | ---------------- |
| Qwen3-8B              | 2.4739                  | [2.4118, 2.5358] |
| Qwen3-8B-Base         | 2.1802                  | [2.1211, 2.2387] |
| Llama 3.1 8B Base     | 2.2212                  | [2.1663, 2.2755] |
| Llama 3.1 8B Instruct | 2.3284                  | [2.2711, 2.3856] |

Lower loss means that the checkpoint assigns higher length-normalized likelihood to the complete scenario–reference-response sequences under its own tokenizer. It does not, by itself, mean that the checkpoint produces more socially appropriate answers.

The new results rank Qwen3-8B-Base lowest in full-sequence loss, followed by Llama 3.1 8B Base, Llama 3.1 8B Instruct, and Qwen3-8B. That ranking is a result about this particular metric and dataset—not a general ranking of social competence.

### The six paired comparisons

The notebook compares the four checkpoints question by question, reports paired \\(t\\)-tests, uses bootstrap confidence intervals for paired mean differences, and adjusts the six \\(p\\)-values with Holm's procedure.

| Comparison, model 1 minus model 2 | Mean loss difference | 95% paired bootstrap CI | Paired \\(d_z\\) |
| --------------------------------- | -------------------- | ----------------------- | ---------------- |
| Qwen3-8B − Qwen3-8B-Base          | +0.2937              | [0.2825, 0.3051]        | +2.581           |
| Qwen3-8B − Llama Base             | +0.2528              | [0.2379, 0.2678]        | +1.689           |
| Qwen3-8B − Llama Instruct         | +0.1455              | [0.1324, 0.1588]        | +1.092           |
| Qwen3-8B-Base − Llama Base        | −0.0409              | [−0.0522, −0.0296]      | −0.359           |
| Qwen3-8B-Base − Llama Instruct    | −0.1482              | [−0.1592, −0.1376]      | −1.355           |
| Llama Base − Llama Instruct       | −0.1072              | [−0.1161, −0.0984]      | −1.187           |

All six comparisons remain significant after Holm correction. The adjusted \\(p\\)-values range from approximately \\(5.30\times10^{-12}\\) to \\(8.13\times10^{-175}\\).

The important interpretive change concerns the two base checkpoints. Qwen3-8B-Base has lower loss than Llama 3.1 8B Base, but the mean difference is only about 0.041 loss units, approximately 1.8% relative to the Llama Base mean. The paired standardized effect is \\(d_z=-0.359\\), which is small by conventional thresholds. It should not be described as the previous \\(d=0.124\\): the current run uses a different set of items and a paired effect-size definition.

Thus, the revised results support a statistically detectable difference in the metric, but they do not establish that one base checkpoint is generally better at social reasoning.

## 2. The manuscript changes I recommend

The key is to update the entire Social Reasoning reporting chain—not just insert the new means. The prior analysis based on 393 records from 99 question clusters is superseded by this run, which selects 393 distinct question records and compares checkpoints on the same questions.

### A. Methods: replace the old sampling and inference description

In the Social Reasoning Methods subsection, replace references to selecting the first 393 records, the 99-cluster structure, CR1-style cluster-robust standard errors, and \\(98\\) degrees of freedom with the current procedure.

Suggested replacement text:

\*\*Sampling and scoring.\*\* The reduced Social Reasoning corpus contains 955 records, each pairing a distinct scenario with one reference response selected using the Copeland aggregation procedure described in the Appendix. We selected 393 records by simple random sampling without replacement, using a fixed random seed of 42. The same selected records were evaluated under all four model checkpoints.  The target sample size of 393 was retained from the initial planning calculation, which assumed a standardized effect size of \\(d=0.20\\), nominal power of 80%, and a two-sided significance level of 0.05. Because this calculation used a two-independent-group approximation, it is treated as the planning basis for the item count rather than as a guarantee of 80% power for the paired comparisons reported below.  For each record, the scenario and reference response were concatenated with a single space and tokenized together using the evaluated checkpoint's native tokenizer. The resulting token sequence was supplied as both model input and labels, so loss was calculated over the complete sequence rather than over the response alone. The primary outcome was the mean item-level full-sequence next-token negative log-likelihood (NLL). No truncation was applied. The longest tokenized sequence contained 553–554 tokens, depending on the checkpoint, and no evaluated example required sliding-window scoring.

This paragraph corrects the description of the actual loss computation, retains full-sequence loss as the sole evaluation metric, and documents the current item-selection process.

One important qualification: the code uses a common plain-text serialization (`question + " " + answer`), not a model-specific chat template. The Methods section should say so explicitly. Otherwise, the paper could still appear to imply that it evaluates each instruction-tuned model using its native conversational interface. The current results are a comparison under a standardized textual serialization and each checkpoint's native tokenizer—not a test of the best possible deployment setup for each model.

### B. Methods: replace the old clustered-statistics paragraph

Use this in the statistical analysis subsection:

For Social Reasoning, each sampled scenario contributes one item-level loss per model, and all four checkpoints are evaluated on the same 393 items. We estimate each model's mean full-sequence loss and its 95% percentile bootstrap confidence interval using 10,000 resamples and seed 2026. For each pair of checkpoints, we calculate within-item loss differences and conduct a two-sided paired \\(t\\)-test (\\(df=392\\)). We estimate 95% confidence intervals for the paired mean differences by bootstrap resampling of the matched items. The standardized paired effect is reported as Cohen's \\(d_z\\), calculated as the mean paired difference divided by the standard deviation of those differences. Holm's procedure controls family-wise error across the six pairwise checkpoint comparisons.

This directly addresses the core of Reviewer R2.4: repeated rows sharing the same exact question no longer inflate the apparent number of independent observations, comparisons are paired item by item, and the six-test comparison family is corrected for multiplicity. The former \\(G=99\\), \\(df=98\\) cluster-based reporting should be removed from the Social Reasoning analysis because it describes the earlier sampling design, not this run.&#x20;

Pasted text(2).txt



### C. Results: replace the old Social Reasoning results paragraph

Use the new means and confidence intervals reported above. The main narrative can remain concise:

\*\*Social Reasoning.\*\* The four checkpoints were evaluated on the same random sample of 393 distinct scenario–reference-response records drawn without replacement from the 955-record reduced corpus. Qwen3-8B-Base exhibited the lowest mean full-sequence loss, followed by Llama 3.1 8B Base, Llama 3.1 8B Instruct, and Qwen3-8B. All six paired model comparisons remained statistically significant after Holm correction (all adjusted \\(p \leq 5.30 \times 10^{-12}\\)).  The difference between Qwen3-8B-Base and Llama 3.1 8B Base was \\(-0.0409\\) loss units when calculated as Qwen3-8B-Base minus Llama 3.1 8B Base (95% paired bootstrap CI \\([-0.0522,-0.0296]\\); paired \\(d_z=-0.359\\)). This is a small difference in absolute loss units, despite strong statistical evidence against a zero mean difference. The results characterize likelihood assigned to complete scenario–reference-response records under the implemented serialization and each model's native tokenizer; they do not establish a definitive ranking of general social competence.

The revised text should replace any earlier claim that the base-model difference is statistically non-significant or that its effect size is \\(d=0.124\\). Both statements refer to the previous analysis, not these results.

### D. Main table and Supplementary Table S10

For the main Social Reasoning table, update all four means and confidence intervals to the new values. The confidence intervals should be labelled as 95% bootstrap CIs, with \\(N=393\\) distinct scenario records.

I recommend retaining full-sequence loss as the only reported evaluation metric, as you specified. Perplexity is just an exponential transformation of NLL, not a second independent result; omitting its separate column will keep the operational measure unambiguous.

For Supplementary Table S10, update the six mean differences, paired bootstrap CIs, and paired Cohen's \\(d_z\\) values to the new results. Keep the sign convention explicit: positive values indicate that model 1 has greater (worse) loss than model 2. To avoid cluttering the table with extremely small \\(p\\)-values, it can state in its caption that all six paired comparisons remained significant after Holm correction, with all adjusted \\(p \leq 5.30\times10^{-12}\\). The saved JSON should continue to preserve the exact \\(p\\)-values for reproducibility.

The loss-comparison figure and effect-size heatmap should also be regenerated from this output. The existing repository already contains those two figures; verify that the copies in the manuscript match the present result file, and keep the p-value heatmap removed.

### E. Discussion and Conclusion: revise the interpretation, not just the numbers

The new findings do not justify a general conclusion that Qwen3-8B-Base has superior social reasoning. They do support a more specific statement: it has the lowest observed full-sequence loss under this experiment's scoring procedure.

Suggested replacement for the affected base-model comparison:

On the full-sequence Social Reasoning metric, Qwen3-8B-Base assigns higher length-normalized likelihood to the evaluated scenario–reference-response records than Llama 3.1 8B Base. Their mean loss difference is \\(0.0409\\) in absolute magnitude, with a small paired standardized effect (\\(|d_z|=0.359\\)). Although the paired comparison is statistically significant after multiplicity correction, full-sequence loss combines scenario modeling with reference-response prediction and is affected by each checkpoint's native tokenization. The finding should therefore be interpreted as a difference in likelihood assigned to these records, not as evidence of superior general social competence. The Social Reasoning result remains one component of the broader model-selection framework rather than a standalone criterion for choosing a universal winner.

Use this interpretation consistently in the Abstract, Results Summary, Practical Summary, and Conclusion. In particular, replace the earlier \\(d=0.124\\) statement wherever it remains, and remove wording such as “better social reasoning overall” where it implies a validated construct-level ranking.

The current result can inform model choice, but its role in the framework should remain complementary to the other dimensions rather than decisive on its own.

## 3. What this resolves for the reviewers—and what it does not

Reviewer R2.4 — repeated observations and inference

The new design addresses the main concern: it uses one record per distinct question, a random sample rather than a file-order prefix, matched comparisons across the models, paired uncertainty estimates, and Holm correction.

Reviewer R2.2 — scoring procedure and reproducibility

The preserved notebook provides an inspectable evaluation implementation, and the method description can now be reconciled with the actual full-sequence loss computation. The current run also confirms that no evaluated example was truncated or scored using a sliding window.

Reviewer R2.3 — construct validity and comparison across tokenizers

The revised analysis does not satisfy the request for validation against independent assessments of generated answers. Under your decision to retain full-sequence loss as the sole metric and not conduct a new validation experiment, the defensible response is to narrow the claims, explain the metric's distinct purpose relative to Social IQa, SOTOPIA, and SocialBench, and acknowledge the absence of independent construct validation. This is a reasoned scope decision, but it may not fully satisfy a reviewer who considers the requested validation essential.

The exact-text deduplication also does not establish that no two questions are semantically reworded versions of one another. The manuscript and response letter should say that the sample contains distinct question records according to the reduction procedure, rather than claiming that semantic near-duplicates have been ruled out. Likewise, the concern about rare behavioral events belongs to the separate behavioral-classification analyses; this continuous-loss evaluation does not resolve it.

## 4. Two reproducibility items to fix before finalizing the response letter

I found two inconsistencies between the repository artifacts that should be reconciled before claiming that the saved JSON is the direct output of the precise notebook revision now in the repository.

First, the notebook uses the relative input filename `most_preferred.jsonl`, whereas the saved JSON's metadata contains the absolute path `/media/external20/shayan_firouzian/most_preferred.jsonl`. The current repository root listing includes the notebook, result JSON, and figures, but not `most_preferred.jsonl`. The source dataset must either be included or made accessible through a documented, reproducible location.

Second, the notebook source lists short model paths (`Qwen3-8B`, `Qwen3-8B-Base`, and the two Llama directory names), whereas the saved JSON uses absolute model paths. The numerical values in the result file agree with the notebook's displayed results, but the path mismatch weakens the evidence that the artifacts were produced by the same final code state. Reconcile the metadata and record the exact checkpoint identifiers/revisions and tokenizer pairing.

These are provenance fixes, not reasons by themselves to reject the reported numbers. They matter for Reviewer R2.2, however, so I would resolve them before asserting that the published script and result file are an exact reproducible pair.

Recommended order: first reconcile those artifact details; then replace the Social Reasoning Methods and Results text, update the two tables and figures, and finally propagate the corrected interpretation through the Abstract and Conclusion. That sequence will make the manuscript's account of the new run internally consistent while keeping the full-sequence metric and the existing scope of the study intact.
