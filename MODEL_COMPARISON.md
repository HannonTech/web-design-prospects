# Local LLM Model Comparison — Demo Site Generation

Tracking results from testing different llama.cpp-served models against the n8n
"Generate Demo Site" pipeline (`cEqApNqtgbTK3S0B`) and direct curl calls to
`http://10.25.1.128:8081`. Hardware: Debian server, 128GB RAM, Tesla P40 (24GB
VRAM, Pascal — no tensor cores, ~347GB/s bandwidth, bandwidth-bound not
compute-bound).

Test brief used for most comparisons: Clover's Bar & Grill (real tagline, clover
emoji branding, green/gold palette, weekly events, menu items, 3 real photo URLs).

## Results

| Model | Type / size | Speed | Works via n8n? | Verdict |
|---|---|---|---|---|
| **Qwen_Qwen3.6-35B-A3B-Q4_K_M** | MoE, 35B total / ~3B active | 60–140s typical | Yes | **Current default.** Best overall design quality, creative, accurate content. Tends toward full-viewport heroes by default but otherwise reliable. |
| Qwen_Qwen3-8B-Q4_K_M | Dense, 8B | 40–70s | Yes | Fast but generic — kept defaulting to the same navy-gradient template regardless of business type, some bugs (mismatched icons, dead CTA links) before prompt fixes. |
| Qwen2.5-Coder-32B-Instruct-Q4_K_M | Dense, 32B | N/A | No | Timed out in initial direct testing (MCP tool). Never got a result — dense 32B too slow on this bandwidth-bound GPU. Not retried via n8n. |
| zai-org_GLM-4.7-Flash-Q4_K_M | Dense | 213s via n8n (smaller output) / 228s for 8,051 tokens via direct test (35.19 t/s) | Yes (slow via n8n) | Mixed signal — first n8n test seemed slow for output size, but a later direct test (8,051 tokens, 35.19 t/s) looked great. Worth re-testing properly against the n8n pipeline; may have been underrated. Chose 70vh hero (not full-viewport) on its own, used `background-attachment:fixed` (mobile compat risk), font slightly too ornate for a "not upscale" brief. |
| DeepSeek-R1-Distill-Qwen-32B-Q4_K_M | Dense reasoning, 32B | N/A | **No** | Hung indefinitely on the full brief; eventually errored after the full 240s node timeout with no output. Reasoning overhead + dense 32B bandwidth cost = too slow/unreliable. |
| gemma-4-12B-it-Q4_K_M | Dense, 12B, reasoning ("thinking" on by default) | 144s for simplified 3-section brief (clean, good output) / failed twice on full brief (500 proxy error, then 280s hard hang) | **No** (also has a separate n8n/LangChain incompatibility — `reasoning_content` field not handled) | Works for simple prompts, breaks down under full production complexity (photos + full instruction set). Not reliable enough to use. |

## Downloaded, not yet tested

- `DeepSeek-Coder-V2-Lite-Instruct-Q4_K_M.gguf` (~10.4GB, MoE 16B total/2.4B active, code-specialized)
- `mistralai_Mistral-Small-3.2-24B-Instruct-2506-Q4_K_M.gguf` (~14.3GB, dense)
- `phi-4-Q4_K_M.gguf` (~9.05GB, dense)
- `Qwythos-9B-Claude-Mythos-5-1M-Q4_K_M.gguf` (~5.63GB, dense reasoning model fine-tuned on Claude traces — flagged risk: reasoning models have a 0-for-2 track record here so far)

## Notes / gotchas learned along the way

- Cloudflare in front of `n8n.hannontech.net` hard-caps requests at 120s (Error 524) regardless of n8n's own per-node timeout. A 524 doesn't mean failure — check `n8n_executions` afterward; the workflow often completes successfully server-side past that window.
- `presets.ini` (`/opt/llama.cpp/presets.ini`) only needs an entry for *overriding* defaults (e.g. larger context). Models dropped into `/opt/llama.cpp/models/` auto-register under their filename.
- `models-max = 2` in `presets.ini` — only two models can stay loaded in 24GB VRAM at once; large models (~20GB each) can't coexist, so swapping has real cost.
- Reasoning models (anything emitting a `<think>` block or separate `reasoning_content` field) have been the least reliable category so far — both DeepSeek-R1-Distill and Gemma 4 had serious issues under full production complexity.
