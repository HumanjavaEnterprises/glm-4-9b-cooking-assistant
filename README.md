# GLM-4-9B Cooking Assistant

A multilingual cooking assistant you can run on your own machine, down to a 4 GB GPU.
Fine-tuned from GLM-4-9B-0414 on public-domain and openly licensed cookbooks.

It answers from the recipe you give it, usually asks for the recipe when an answer depends on one it cannot see, and answers
general cooking questions. On questions it could not answer without a recipe, base GLM-4-9B-0414
invented recipe details 40.1% of the time; this model does it 9.7% of the time. Given the
recipe, it reads it correctly 96.2% of the time (base: 83.1%). General answers are not improved: on our 82
general questions the full-precision model scored lower than the base (46.3% vs 56.1%; not statistically
significant). Three open-weights judges from other model families, see [EVALS.md](EVALS.md).

*Unofficial fine-tune of GLM-4-9B-0414 by Humanjava Enterprises; not made or endorsed by Zhipu AI / Z.ai.*

| | |
|---|---|
| **Run it on a small GPU** | [SMALL-GPU.md](SMALL-GPU.md): pick a file for your card, install, start, test, troubleshoot |
| **GGUF** (llama.cpp, Ollama, LM Studio) | [HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant-GGUF](https://huggingface.co/HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant-GGUF) |
| **Full precision** (Transformers, vLLM) | [HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant](https://huggingface.co/HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant) |
| **Evaluation set** (every question, answer and judge verdict behind the numbers) | [HumanjavaEnterprises/cooking-assistant-eval](https://huggingface.co/datasets/HumanjavaEnterprises/cooking-assistant-eval), config `glm-4-9b-v1` |
| **Results** | [EVALS.md](EVALS.md) |
| **The same recipe on Qwen3-8B** | [qwen3-8b-cooking-assistant](https://github.com/HumanjavaEnterprises/qwen3-8b-cooking-assistant) |
| **Help refine it** | [Label five answers](https://label-test.humanjava.com/?s=cook4&k=cook-9ac741a1e387&src=github-guide-glm): blind, no account, a few minutes |

## Quick start

With [Ollama](https://ollama.com):

```bash
ollama run vveerrgg/glm-4-9b-cooking-assistant
```

With llama.cpp on an 8 GB GPU:

```bash
pip install -U huggingface_hub
hf download HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant-GGUF glm-4-9b-cooking-assistant-v1-Q4_K_M.gguf --local-dir .
llama-server -m glm-4-9b-cooking-assistant-v1-Q4_K_M.gguf --jinja -ngl 99 -c 8192 -fa on -ctk q8_0 -ctv q8_0 --port 8080
examples/ask.sh          # -> an answer of 900 ml, e.g. "You would need 900 ml of milk. This is calculated by multiplying the original 300 ml by 3."
```

Other card sizes: `python tools/vram_estimate.py --vram 6` (or 8, 12, ...) prints the largest setup it estimates will
fit, and the flags; it can suggest more context or more GPU layers than the measured, conservative table below.

## Which file?

| GPU memory | File | Measured memory use |
|---|---|---|
| 4 GB | `Q4_K_M`, 18 of 40 layers on the GPU | ~3.0 GB |
| 6 GB | `Q4_K_M`, 27 of 40 layers on the GPU | ~4.1 GB |
| 8 GB | `Q4_K_M`, context 8,192 | 6.0 GB |
| 10 GB and up | `Q8_0`, context 8,192 | 9.2 GB |

On our checks none of the three loses significantly against full precision in inventing details or reading a recipe. Details and speeds: [SMALL-GPU.md](SMALL-GPU.md).

## How to use it

The GLM-4-0414 chat format (built into the files) and this system prompt (the one it was trained with):

```
You are a cooking assistant. You answer questions about dishes, ingredients, techniques and food safety. When you are not sure, you say so.
```

Put the recipe in the same message as the question. Without a recipe it usually asks for one; it may add a typical range, marked as typical. The design
idea: **the fine-tune for behaviour, your app for facts** (look up the recipe and pass it in) **and code for
numbers** (scaling, unit conversion).

## In this repo

| Path | What |
|---|---|
| [SMALL-GPU.md](SMALL-GPU.md) | the full process for a small-GPU machine, with measured memory and speed |
| [EVALS.md](EVALS.md) | how the model was checked, and the results |
| [examples/](examples/) | `ask.sh` (curl), `chat.py` (Python, any OpenAI-compatible server), an Ollama `Modelfile` |
| [CONTRIBUTING.md](CONTRIBUTING.md) · [SECURITY.md](SECURITY.md) | report a wrong answer, a bug or a security issue |
| [tools/vram_estimate.py](tools/vram_estimate.py) | memory planner, fitted to measurements (within 0.04 GiB on 21 setups); `python -m pytest tests` |

## Licence and credits

Code and docs in this repo: MIT. Model weights: MIT, same as the base model (the base model's copyright notice,
"Copyright (c) 2025 Zhipu AI", ships in the `LICENSE` file of each model repo). Evaluation set text: CC BY-SA 4.0
(per-row attribution in the dataset).

Base model: [GLM-4-9B-0414](https://huggingface.co/zai-org/GLM-4-9B-0414) by Zhipu AI / Z.ai (MIT). Training data and
recipe: shared with [Qwen3-8B Cooking Assistant](https://huggingface.co/HumanjavaEnterprises/Qwen3-8B-Cooking-Assistant)
(teacher Qwen3.8-27B, Apache-2.0; part of the general answers and the preference-stage answers were written by Qwen3-8B
models, Apache-2.0). The headline scores are from the full-precision model served with vLLM (greedy decoding); the quantised-file scores
and the memory and speed figures are from [llama.cpp](https://github.com/ggml-org/llama.cpp) (MIT).
Cookbook sources: Wikibooks Cookbook contributors (CC BY-SA 4.0), Project Gutenberg and Internet Archive public-domain
books, NHLBI (US government).

Built by [Humanjava Enterprises](https://humanjava.com).
