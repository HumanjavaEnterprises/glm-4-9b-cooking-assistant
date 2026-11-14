# Running GLM-4-9B Cooking Assistant on a small GPU

Step by step, from "I have a gaming laptop" to asking the model about a recipe. Every number here was measured
with llama.cpp. Your speeds will differ; the memory figures should be close on other NVIDIA cards (we measured on one RTX PRO 4000).

## 1. Check what you have

- **NVIDIA:** run `nvidia-smi` and read the memory total, e.g. `8188MiB` is an 8 GB card.
- **Apple Silicon (M1–M4):** the GPU shares system memory. Leave a few GB of the shared memory free: the Q4_K_M file is 6.2 GB, Q6_K 8.3 GB.
- **AMD or Intel GPU:** use a llama.cpp Vulkan build (or LM Studio, which bundles one). The memory figures should be close (we measured only CUDA).
- **No GPU:** it runs on the CPU, slowly. Use Q4_K_M with 16 GB of RAM.

Your desktop, browser and other apps use some GPU memory too. Leave about 0.5 GB free. (Memory figures in this
guide are GiB, the unit `nvidia-smi` reports in MiB: an "8 GB" card shows about 8,000 MiB.)

## 2. Pick a file and settings

| Your GPU memory | File | Context | KV cache | Layers on GPU | Uses (measured) |
|---|---|---|---|---|---|
| 4 GB | Q4_K_M | 4,096 | q8_0 | 18 of 40 (`-ngl 18`) | ~3.0 GB |
| 6 GB | Q4_K_M | 4,096 | q8_0 | 27 of 40 (`-ngl 27`) | ~4.1 GB |
| 8 GB | Q4_K_M | 8,192 | q8_0 | all (`-ngl 99`) | 6.0 GB |
| 10 GB | Q8_0 | 8,192 | q8_0 | all | 9.2 GB |
| 12 GB+ | Q8_0 | 16,384 | f16 | all | 9.7 GB |

These are measured settings with about 0.5 GB of headroom, chosen conservatively. Not sure?
`python tools/vram_estimate.py --vram 8` prints the largest setup it estimates will fit and the exact flags; it can
suggest more context or more GPU layers than this table.

What the settings mean:
- **File (quantisation).** Smaller files hold the weights in fewer bits. On our checks, Q4_K_M, Q6_K and Q8_0 show no significant difference from full precision in inventing details or reading a shown recipe (see [EVALS.md](EVALS.md)). Use the biggest that fits.
- **Context** is how much text the model sees at once: system prompt + recipe + question + answer. A long recipe is
  about 1,000 tokens, so 4,096 is plenty for one recipe; 8,192 leaves room for a conversation.
- **KV cache** is the model's working memory for that context. `q8_0` halves it. It needs
  flash attention (`-fa on`).
- **Layers on GPU.** The model has 40 layers. If they don't all fit, put some on the GPU and the rest run on the CPU.
  Slower, but it works.

### Measured: memory and speed

llama.cpp (build `ca8daad`, CUDA) on an NVIDIA RTX PRO 4000 (24 GB), one request. "Memory" is what the llama-server
process used on the GPU.

| File | Context | KV cache | Memory | Prompt (tokens/s) | Answer (tokens/s) |
|---|---|---|---|---|---|
| Q4_K_M | 4,096 | f16 | 5.9 GB | 1,162 | 56 |
| Q4_K_M | 4,096 | q8_0 | 5.9 GB | 1,271 | 56 |
| Q4_K_M | 8,192 | f16 | 6.1 GB | 1,281 | 58 |
| Q4_K_M | 8,192 | q8_0 | 6.0 GB | 1,274 | 56 |
| Q4_K_M | 16,384 | f16 | 6.4 GB | 1,308 | 58 |
| Q4_K_M | 16,384 | q8_0 | 6.1 GB | 1,194 | 55 |
| Q6_K | 4,096 | f16 | 7.7 GB | 1,199 | 45 |
| Q6_K | 4,096 | q8_0 | 7.7 GB | 1,246 | 44 |
| Q6_K | 8,192 | f16 | 7.9 GB | 1,333 | 46 |
| Q6_K | 8,192 | q8_0 | 7.8 GB | 1,256 | 45 |
| Q6_K | 16,384 | f16 | 8.2 GB | 1,337 | 46 |
| Q6_K | 16,384 | q8_0 | 7.9 GB | 1,256 | 45 |
| Q8_0 | 4,096 | f16 | 9.2 GB | 1,524 | 39 |
| Q8_0 | 4,096 | q8_0 | 9.1 GB | 1,468 | 38 |
| Q8_0 | 8,192 | f16 | 9.4 GB | 1,519 | 39 |
| Q8_0 | 8,192 | q8_0 | 9.2 GB | 1,458 | 39 |
| Q8_0 | 16,384 | f16 | 9.7 GB | 1,548 | 39 |
| Q8_0 | 16,384 | q8_0 | 9.4 GB | 1,463 | 38 |

Partial offload (Q4_K_M, context 4,096, q8_0 cache). The CPU here is a workstation Xeon; a laptop CPU is slower.

| Layers on GPU | Memory | Prompt (tokens/s) | Answer (tokens/s) |
|---|---|---|---|
| 27 of 40 | 4.1 GB | 299 | 32 |
| 18 of 40 | 3.0 GB | 214 | 26 |
| 9 of 40 | 1.9 GB | 158 | 22 |

For reading speed, 15 tokens/s is already faster than most people read.

## 3. Install a runtime

Pick one.

**Ollama** (easiest). Install from [ollama.com](https://ollama.com), then:

```bash
ollama run vveerrgg/glm-4-9b-cooking-assistant          # Q4_K_M, 8 GB cards
ollama run vveerrgg/glm-4-9b-cooking-assistant:q6_k     # Q6_K, 7.9 GB at context 8,192 (Ollama's default f16 KV cache)
ollama run vveerrgg/glm-4-9b-cooking-assistant:q8_0     # Q8_0, 10 GB and up
```

Or straight from Hugging Face: `ollama run hf.co/HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant-GGUF:Q4_K_M`.

Ollama picks how many layers fit by itself. The model repo carries the chat template, the trained system prompt and
the sampling settings, so this one line is all you need. (Tested with Ollama 0.12.6 and this template: it reads the recipe, and when the recipe is missing it asks for it (adding a typical range, marked as typical). If you build it from the GGUF file yourself, use
[examples/Modelfile](examples/Modelfile), step 4.)

**LM Studio** (point and click). Search for `HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant-GGUF` in the model browser,
download the file from step 2, set **GPU offload** and **context length** from the table, and paste the system prompt
from step 5.

**llama.cpp** (most control; the commands below). Download a prebuilt release for your platform from
[github.com/ggml-org/llama.cpp/releases](https://github.com/ggml-org/llama.cpp/releases) (CUDA for NVIDIA, Vulkan for
AMD / Intel, macOS for Apple Silicon), or `brew install llama.cpp` on a Mac.

## 4. Download the file

```bash
pip install -U huggingface_hub
hf download HumanjavaEnterprises/GLM-4-9B-Cooking-Assistant-GGUF glm-4-9b-cooking-assistant-v1-Q4_K_M.gguf --local-dir .
```

Check it against `SHA256SUMS` in the model repo if you like: `sha256sum glm-4-9b-cooking-assistant-v1-Q4_K_M.gguf`.

For Ollama with the trained system prompt:

```bash
cp examples/Modelfile .      # edit FROM if you picked another file
ollama create cooking -f Modelfile
ollama run cooking
```

The Modelfile sets the GLM-4 chat template, the trained system prompt, the context length and the stop tokens.

## 5. Start the server (llama.cpp)

8 GB card, everything on the GPU:

```bash
llama-server -m glm-4-9b-cooking-assistant-v1-Q4_K_M.gguf --jinja \
  -ngl 99 -c 8192 -fa on -ctk q8_0 -ctv q8_0 --port 8080
```

4 GB card, part on the GPU:

```bash
llama-server -m glm-4-9b-cooking-assistant-v1-Q4_K_M.gguf --jinja \
  -ngl 18 -c 4096 -fa on -ctk q8_0 -ctv q8_0 --port 8080
```

| Flag | What it does |
|---|---|
| `-ngl N` | layers on the GPU (`99` = all 40) |
| `-c N` | context length in tokens |
| `-fa on` | flash attention: less memory, and needed for the q8_0 cache |
| `-ctk q8_0 -ctv q8_0` | store the KV cache in 8 bits (about half the memory) |
| `--jinja` | use the chat template inside the file (the default in recent llama.cpp builds) |

Open `http://127.0.0.1:8080` for a chat page, or use the API.

**The system prompt.** The model was trained with this one and works best with it:

```
You are a cooking assistant. You answer questions about dishes, ingredients, techniques and food safety. When you are not sure, you say so.
```

## 6. Try it

```bash
examples/ask.sh
```

Expected: an answer of 900 ml, e.g. "You would need 900 ml of milk. This is calculated by multiplying the original 300 ml by 3." (it reads the recipe in the prompt and triples 300 ml).

Now ask without the recipe:

```bash
examples/ask.sh "" "If I triple my pancake recipe, how much milk do I need?"
```

It should ask you for the original amount (it may add a typical range, marked as typical) rather than present a number
as your recipe's. That is the behaviour this model was built for.
From Python: [examples/chat.py](examples/chat.py) (works with llama-server, Ollama, LM Studio or vLLM).

## 7. Using it well

- **Give it the recipe.** Put the recipe text in the same message as the question. It answers from that text; it
  was not trained to recall recipes from memory.
- **Let your app do the maths.** Scaling and unit conversion are best calculated by code; the model is good at
  reading the recipe and explaining.
- **Not a food-safety authority.** It repeats standard guidance but can be wrong. Check official sources for
  temperatures and storage.

## Troubleshooting

| Problem | Fix |
|---|---|
| `out of memory` / `failed to allocate` | Lower `-c`, add `-ctk q8_0 -ctv q8_0 -fa on`, use a smaller file, or lower `-ngl` |
| Very slow answers | Some layers are on the CPU. Raise `-ngl` as far as memory allows, or use a smaller file so all layers fit |
| Ollama answer never ends, or asks itself questions | Ollama is using a raw template. Pull from `hf.co/...` (step 3) or build with [examples/Modelfile](examples/Modelfile) |
| It invents quantities | It has no recipe to read. Paste the recipe into the message |
| Strange characters or loops | Use the trained system prompt; keep temperature around 0.7 |
| Memory still too tight | Close the browser (it uses GPU memory), or run on the CPU with `-ngl 0` |
