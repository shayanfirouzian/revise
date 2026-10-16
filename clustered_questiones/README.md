I reviewed the saved notebook and the results JSON in [`clustered_questiones`](https://github.com/shayanfirouzian/revise/tree/main/clustered_questiones?utm_source=chatgpt.com), checked their consistency, and compared the findings against the reviewer concerns and the manuscript-revision history.

The revised run substantially addresses the sampling and statistical problems raised by Reviewer 2.4, but it changes the Social Reasoning results enough that several manuscript claims must be updated. It also does not, by itself, resolve the reviewers’ concerns about the construct validity and cross-tokenizer comparability of full-sequence loss.

## 1. What the new run establishes

The saved [notebook](https://github.com/shayanfirouzian/revise/blob/main/clustered_questiones/SocialReasoning_benchmark_revised.ipynb?utm_source=chatgpt.com) and [results JSON](https://github.com/shayanfirouzian/revise/blob/main/clustered_questiones/social_reasoning_baseline_revised.json?utm_source=chatgpt.com) agree on the following design:

Distinct questions sampled

# 307

Records per model

# 1,228

Random seed

# 42

Bootstrap replicates

# 5,000

The sample contains 307 distinct question IDs, with exactly four records per question for each model. All four models were evaluated on the same question set.

The uploaded `ProlificAI.jsonl` also matches the source-file SHA-256 checksum documented in the previously archived example manifest, so this result is consistent with the dataset version used in the earlier audit.&#x20;

social_reasoning_tested_example_manifest.csv



The power calculation now targets 80% power for a standardized paired effect \\(d_z=0.20\\), using a conservative Bonferroni planning level of \\(0.05/6\\) for the six model comparisons. This yields 307 question clusters. The subsequent analysis uses paired question-level \\(t\\)-tests, question-bootstrap confidence intervals, and Holm correction.

### Updated model results

These are the values from the saved run. The confidence intervals below are the 95% percentile-bootstrap intervals over question-level scores.

| Model                   | Mean full-sequence loss | Native-tokenizer PPL\* |
| ----------------------- | ----------------------- | ---------------------- |
| Qwen3 (8B) Base         | 2.2579 [2.2140, 2.3013] | 9.563                  |
| Llama 3.1 (8B) Base     | 2.3161 [2.2738, 2.3575] | 10.136                 |
| Llama 3.1 (8B) Instruct | 2.4138 [2.3713, 2.4561] | 11.176                 |
| Qwen3 (8B) Instruct     | 2.5514 [2.5049, 2.5965] | 12.825                 |

\*Perplexity is the exponentiated mean loss for each model's native tokenizer. It is descriptive only and should not be interpreted as directly comparable across tokenizers.

The observed ordering from lowest to highest loss is Qwen3 Base, Llama Base, Llama Instruct, Qwen3 Instruct.

### Updated pairwise comparisons

All six model comparisons have 95% paired question-bootstrap confidence intervals that exclude zero and remain statistically significant after Holm correction. The table gives each difference as Model A minus Model B; a negative difference means Model A has lower loss.

| Comparison                      | Mean difference [95% CI]   | Paired \\(d_z\\) |
| ------------------------------- | -------------------------- | ---------------- |
| Qwen3 Instruct − Qwen3 Base     | +0.2935 [0.2838, 0.3032]   | +3.359           |
| Qwen3 Instruct − Llama Base     | +0.2353 [0.2225, 0.2478]   | +2.042           |
| Qwen3 Instruct − Llama Instruct | +0.1376 [0.1266, 0.1488]   | +1.360           |
| Qwen3 Base − Llama Base         | −0.0582 [−0.0677, −0.0488] | −0.682           |
| Qwen3 Base − Llama Instruct     | −0.1559 [−0.1650, −0.1465] | −1.847           |
| Llama Base − Llama Instruct     | −0.0977 [−0.1055, −0.0896] | −1.387           |

The most consequential interpretive change is the base-to-base comparison. Qwen3 Base has a lower mean full-sequence loss than Llama Base by 0.0582 loss units, with a paired 95% interval excluding zero and paired \\(d_z=-0.682\\).

That is not the same effect-size statistic as the earlier reported \\(d=0.124\\). The earlier result came from the old sample and effect-size calculation; the new value is a paired standardized difference over question-level outcomes. They must not be compared as though they were the same estimate.

The raw loss difference and its confidence interval should lead the interpretation. The new result supports a difference on this benchmark's operational metric; it does not, on its own, establish that the difference is practically consequential or that Qwen3 Base has generally superior social competence.

## 2. The manuscript revisions I recommend

I would prioritize the following changes in this order.

### Revision 1 — Replace the Social Reasoning sampling and statistical-method description

This addresses the core of Reviewer 2.4: the original use of 393 records from only 99 question families made the apparent sample size misleading. The reviewer also explicitly requests paired comparisons and multiplicity control.&#x20;

Pasted text(2).txt



The Methods section should distinguish question clusters from source records. Do not write \\(N=1{,}228\\) as if the experiment had 1,228 independent observations.

Suggested replacement text:

```
For Social Reasoning, the distinct question text was treated as the
sampling and inferential unit. The ProlificAI dataset contained 955
distinct questions, each represented by four preference-comparison
records. We drew a simple random sample without replacement of 307
complete question groups using a fixed random seed (42), retaining all
four records for each selected question (1,228 records per model).
The sample-size calculation targeted 80\% power for a standardized
paired effect of $d_z=0.20$, using a Bonferroni planning level of
$\alpha=0.05/6$ for the six pairwise model comparisons.

For each record, the primary and sole Social Reasoning metric was
mean causal-language-model loss over the full sequence formed by
concatenating the question, a space, and the chosen response before
tokenization. Loss was evaluated over the complete tokenized sequence;
the rejected response was retained in the source records but was not
scored. The four record-level losses were averaged within each question
before calculating model means. Model comparisons used paired
$t$-tests on the question-level means, with 95\% percentile-bootstrap
confidence intervals obtained by resampling questions and Holm
correction across the six pairwise tests. Perplexity was retained only
as a descriptive transformation of each model's native-tokenizer loss.
```

This wording accurately reflects the current code: the question and chosen response are concatenated as text and tokenized together. The older description that tokenizes the two parts separately and concatenates token sequences should be removed if it remains in the working manuscript.

One additional technical point: `MAX_SEQUENCE_LENGTH=512` appears in the notebook, but the scoring function does not enforce a fixed 512-token cap when the model or tokenizer supplies a higher context limit. It raises an error only when a sequence exceeds the detected model-specific limit. The manuscript should describe that actual policy, not claim that every example was restricted to 512 tokens.

### Revision 2 — Replace the main Social Reasoning results table

The main table should use the new estimates and the question-bootstrap confidence intervals. It should retain a clear distinction between full-sequence loss, which is the sole metric, and native-tokenizer perplexity, which is descriptive.

Suggested table content:

```
\begin{table}[t]
\centering
\caption{Full-sequence loss on the ProlificAI Social Reasoning
benchmark. Results use a reproducible random sample of 307 distinct
question clusters, each represented by four source records
(1,228 records per model). Entries are mean full-sequence losses
with 95\% percentile-bootstrap confidence intervals obtained by
resampling question clusters. Lower loss is lower under the
operational metric. Native-tokenizer perplexity is included for
descriptive reference only and is not directly comparable across
tokenizers.}
\label{tab:social_loss}
\begin{tabular}{lcc}
\toprule
\textbf{Model} &
\textbf{Mean full-sequence loss [95\% CI]} &
\textbf{Native-tokenizer PPL} \\
\midrule
Qwen3 (8B) Base
& $2.258$ [$2.214$, $2.301$] & $9.563$ \\
Llama 3.1 (8B) Base
& $2.316$ [$2.274$, $2.357$] & $10.136$ \\
Llama 3.1 (8B) Instruct
& $2.414$ [$2.371$, $2.456$] & $11.176$ \\
Qwen3 (8B) Instruct
& $2.551$ [$2.505$, $2.597$] & $12.825$ \\
\bottomrule
\end{tabular}
\end{table}
```

The manuscript should not retain the old values based on the first 393 dataset entries. In particular, update every occurrence of the old \\(N=393\\) sample description and the old power calculation. Also make sure the table caption and any surrounding paragraph say that the 307 units are distinct questions, not merely 307 randomly chosen rows.

### Revision 3 — Replace the main Social Reasoning Results paragraph

Suggested replacement:

```
\textbf{Social Reasoning.} On the random sample of 307 question
clusters (1,228 chosen-response records per checkpoint), Qwen3 (8B)
Base had the lowest mean full-sequence loss (2.258; 95\% question-
bootstrap CI [2.214, 2.301]), followed by Llama 3.1 (8B) Base
(2.316 [2.274, 2.357]), Llama 3.1 (8B) Instruct
(2.414 [2.371, 2.456]), and Qwen3 (8B) Instruct
(2.551 [2.505, 2.597]). All six paired question-level comparisons
remained statistically significant after Holm correction. In the
base-to-base comparison, the mean difference for Qwen3 Base minus
Llama 3.1 Base was $-0.0582$ (95\% paired question-bootstrap
CI [$-0.0677$, $-0.0488$]; paired $d_z=-0.682$).
These findings describe differences in full-sequence loss on this
benchmark; they do not establish a tokenizer-invariant ranking of
general social competence.
```

This resolves an important inconsistency between the previous analysis and the new run. In the old manuscript, some wording characterized the base-model Social Reasoning difference as negligible. The new paired result requires that language to be reconsidered. The correct replacement, however, is not the broader claim that Qwen3 Base is definitively “better at social reasoning.” It is that it has lower observed full-sequence loss on the sampled benchmark, with uncertainty assessed at the question level.

### Revision 4 — Replace Supplementary Table S10

The supplementary table should report the six paired contrasts from the new run. I recommend reporting the mean differences, paired bootstrap confidence intervals, and paired \\(d_z\\), while stating that all six tests remain significant after Holm correction. There is little benefit in printing a column of extremely small adjusted \\(p\\)-values.

Use the new six contrasts above, label the difference direction explicitly as Model A minus Model B, and label the effect size as paired \\(d_z\\). Do not retain the older 99-cluster values or present the earlier unpaired/pooled Cohen's \\(d\\) values as if they were equivalent to these paired effect sizes.

## 3. Update the Discussion, Conclusion, and Abstract

### Discussion and Conclusion

Any sentence claiming that the Social Reasoning difference between Qwen3 Base and Llama Base is negligible should be updated. The new results show a lower loss for Qwen3 Base and a paired interval that excludes zero.

I suggest using wording like this:

> On the sampled ProlificAI benchmark, Qwen3 (8B) Base obtained lower full-sequence loss than Llama 3.1 (8B) Base under paired question-level analysis. This is a benchmark-specific difference in the chosen-sequence likelihood measure, rather than independent evidence of greater general social competence. Together with the behavioral and analytical-reasoning results, the finding illustrates why model selection for GABM should consider multiple dimensions rather than rely on a single universal ranking.

This preserves the paper's broader multi-dimensional argument, without making the result stronger than the metric allows.

### Abstract

The abstract should no longer say that the base models have negligible Social Reasoning differences. It should summarize the updated finding in terms of the operational loss measure, while keeping the methodological contribution and multi-dimensional trade-off as the main message.

I would avoid giving the abstract a claim like “Qwen3 Base demonstrates superior social reasoning.” The current run supports a lower full-sequence-loss score on this particular benchmark; it does not validate that stronger interpretation.

## 4. Which reviewer concerns are resolved—and which remain

The new run is a substantial improvement, but not every reviewer request is addressed by changing the sample and inference method.

Reviewer 2.4 — Repeated observations and inflated certainty

The new design addresses the principal concern: random sampling of complete question groups, one question-level outcome per model, paired comparisons, bootstrap intervals, and Holm correction. The remaining reproducibility improvement is to publish an explicit selected-question manifest. The code identifies exact repeated questions; it does not establish that semantic paraphrases were detected.

Reviewer 2.2 — Missing original code and mismatched implementation

The revised notebook and saved output provide a reproducible starting point. The Methods section must now match its implementation, especially joint tokenization, model context limits, and the exact model/tokenizer revisions used.

Reviewer 3.5 — Inconsistent claims about base-model Social Reasoning

The revised results give a direct answer: Qwen3 Base has lower measured loss than Llama Base, and the paired interval excludes zero. Replace the old negligible-difference wording, but do not translate a loss difference into a general competence claim.

Reviewers 2.3 and 3.2 — Construct validity and tokenization

These remain only partly addressed. The saved run still measures full-sequence loss on the question plus chosen response, using each model's own tokenizer. The statistical correction improves uncertainty estimation; it cannot make the score tokenizer-invariant or establish that it measures social competence.&#x20;

Pasted text(2).txt



This last point matters. Reviewer 3.2 explicitly requested response-only conditional loss and suggested validation against independent judgments. You have chosen to retain full-sequence loss as the sole metric, so the paper should present that as a deliberate operationalization of joint scenario-plus-response sequence modeling and narrow its conclusions accordingly. The current run cannot be described as satisfying the reviewer’s explicit request for a response-only measure or external validation.

To address the concern as far as this design permits, the Limitations section should say that the reported score includes likelihood assigned to the scenario itself as well as the chosen response; that tokenization differs between Qwen and Llama; and that the study does not validate the score against independent judgments of generated answers. Also avoid calling publisher-preference labels culturally universal gold standards. Those limitations support the current framework-demonstration framing without adding another metric.

The archived revision audit already records the literature comparison with Social IQa, SOTOPIA, and SocialBench as addressed; check the current LaTeX source for that paragraph rather than duplicating it.&#x20;

revision_status_audit_2026-10-11.md



## 5. Reproducibility additions before final submission

I recommend two small repository additions alongside the manuscript edits.

First, add a `selected_question_manifest.csv` with one row per sampled question: the hashed question ID, source line numbers for its four records, and the source dataset checksum. The results JSON records the sampled question IDs, but an explicit manifest makes it easier for reviewers to establish exactly what was evaluated.

Second, add a README to `clustered_questiones` specifying that this is the primary question-clustered analysis, including seed 42, the 307-question sample, the 1,228 records per model, and the paired-inference method. Also explain that [`unique_questions`](https://github.com/shayanfirouzian/revise/tree/main/unique_questions?utm_source=chatgpt.com) contains a different analysis based on one derived most-preferred answer per question, with a different sample and estimand. Its numbers should not be mixed with this run or presented as a direct replication without a pre-specified sensitivity-analysis rationale.

Finally, enrich the result metadata with model and tokenizer repository identifiers and revision hashes, software versions, the source-file checksum, and per-model token-length summaries. The present run did not fail the context-length guard, but it does not save all of these details in the results JSON.

Recommended order: update the Methods and main results table first, then Supplementary Table S10, and then revise the Discussion, Conclusion, and Abstract to reflect the new estimates. After those results are frozen, update the reviewer response to distinguish concerns resolved by the new clustered design from the construct-validity and tokenization concerns that remain limitations.
