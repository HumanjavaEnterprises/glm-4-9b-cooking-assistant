# How GLM-4-9B Cooking Assistant was checked

## Versions

The model is one product with dated versions (git tags on the Hugging Face repos, tags on Ollama). Every version is
scored on the same 682 questions by the same judges, so the rows compare directly.

| Version | Date | Invents (no recipe, 2 of 3 judges) | Correct (recipe shown, 2 of 3 judges) |
|---|---|---|---|
| **v1** (current) | 2026-11-14 | 9.7% (base 40.1%) | 96.2% (base 83.1%) |

The rest of this page describes v1.

Everything is on the fixed held-out set of 682 questions in 21 languages that was never used in training (the same
set as [Qwen3-8B Cooking Assistant](https://github.com/HumanjavaEnterprises/qwen3-8b-cooking-assistant)): 290 with the
recipe shown, 10 recall questions with no recipe ("how do you make X?"), 300 that need a recipe the model is not
shown, and 82 general cooking questions. Each answer from base
GLM-4-9B-0414 and from this model was judged blind. The full write-up, with confidence intervals and caveats, is on
the [model card](https://huggingface.co/HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant).

## Does it invent recipe details?

Three open-weights judges, none of them from the GLM or the Qwen family, scored every answer on its own:
DeepSeek-V4-Flash (MIT), gpt-oss-120b (Apache-2.0) and Seed-OSS-36B-Instruct (Apache-2.0). Share of no-recipe
questions where the answer invents recipe details:

| counted as "invents" when | GLM-4-9B-0414 | this model |
|---|---|---|
| at least 2 of 3 judges agree | 40.1% [34.7, 45.8] | **9.7%** [6.8, 13.6] |
| all three agree | 28.1% | 3.7% |
| any one judge says so | 58.9% | 27.1% |

And when the recipe **is** shown (290 questions, 2 of 3 judges agree): GLM-4-9B-0414 reads it correctly
83.1% [78.4, 87.0] of the time, this model **96.2%** [93.3, 97.9] (McNemar p
< 1e-8).

A fourth judge, Qwen3.8-27B (Apache-2.0), also scored every answer; it is a Qwen-family model and wrote the training
questions, so it stays out of the headline. With all four and 3 of 4 agreeing: invents 30.1% vs
4.7%, correct with the recipe shown 78.6% vs 95.5%.

Caveats: the judges are models, and model judges tend to share mistakes, so three agreeing is less than three
independent checks. The training data is the Qwen3-8B Cooking Assistant's, unchanged: part of it was written by Qwen3-8B
models, so for this model it is another model's words.

**We are still refining the numbers with people.** Label Test is our blind labelling tool: you see five answers, one at
a time, and decide whether each made recipe details up, asked for the recipe, or gave honest general help. No
account; we keep only a name or initials and your labels. [Label five answers](https://label-test.humanjava.com/?s=cook4&k=cook-9ac741a1e387&src=github-guide-glm).

## General questions

82 general cooking questions graded against a reference answer (2 of 3 judges agree): base GLM-4-9B-0414 56.1%, this
model 46.3% (14 questions only the base gets right, 6 only this model; McNemar p = 0.12, not significant). General
answers are not improved. The quantised files scored 54-56% on the same questions (next section; not compared against the base),
so the gap may be partly a serving or decoding effect (one of three differences has p < 0.05, none after correcting for three comparisons); not settled.

## Do the small files lose quality?

Same questions, same three judges, each quantised file against full precision:

| | Q4_K_M | Q6_K | Q8_0 |
|---|---|---|---|
| Invents with no recipe, 2 of 3 judges (full precision) | 10.7% (8.7%) | 11.7% (9.3%) | 9.4% (10.0%) |
| Correct when the recipe is shown (full precision) | 95.5% (95.9%) | 95.5% (96.2%) | 95.9% (96.2%) |
| Difference significant? (McNemar, inventing) | no, p = 0.26 | no, p = 0.14 | no, p = 0.77 |

With the three headline judges, no quantised file differs significantly from full precision on inventing (p ≥ 0.14) or on reading a shown recipe (p ≥ 0.5). On the 82 general questions every quantised file scored higher than full precision: Q4_K_M 54.9% vs 46.3% (p = 0.039), Q6_K 53.7% vs 47.6% (p = 0.27), Q8_0 56.1% vs 47.6% (p = 0.065). The full-precision answers are the same text in every comparison; only the judging was re-run for each file, which alone moves their scores by up to 1.3 points. The quantised answers come from llama.cpp and the full-precision ones from vLLM, so the full-precision model's lower general score may be partly a serving or decoding effect (one of three differences has p < 0.05, none after correcting for three comparisons); not settled.

## Does it copy its training text?

Training documents from the Wikibooks Cookbook (CC BY-SA) were probed with prefixes of 32 to 128 tokens at several offsets, as raw text and as chat prompts, with greedy decoding. The model never continues a document verbatim (no 50-token match; longest overlap 89 characters, the same as base GLM-4-9B-0414), and its overlap with the true text is no higher than the base's (ROUGE-L 0.158 vs 0.161).
