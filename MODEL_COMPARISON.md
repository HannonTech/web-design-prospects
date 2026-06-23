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
| **zai-org_GLM-4.7-Flash-Q4_K_M** | Dense | 162–228s, ~35 tokens/sec consistently | Yes | **Strong contender, possibly better than the current default.** Full standardized Clover's brief test: correct green/gold palette, clover emoji worked into every section heading (not just hero), real photo as hero background with overlay, all 6 weekly events listed cleanly, all 4 menu items *with descriptive taglines* ("The Reuben Sandwich — Corned beef & kraut") — best copywriting of any model tested. All 3 photos in a gallery with a tasteful hover effect, correct/matching icons, every `tel:` link correct, bonus secondary CTA, no dead links, clean responsive breakpoint. Earlier n8n test (before this full retest) had chosen a 70vh hero with `background-attachment:fixed` (mobile compat risk) and a slightly too-ornate font — worth watching for in future runs but not disqualifying. |
| DeepSeek-R1-Distill-Qwen-32B-Q4_K_M | Dense reasoning, 32B | N/A | **No** | Hung indefinitely on the full brief; eventually errored after the full 240s node timeout with no output. Reasoning overhead + dense 32B bandwidth cost = too slow/unreliable. |
| gemma-4-12B-it-Q4_K_M | Dense, 12B, reasoning ("thinking" on by default) | 144s for simplified 3-section brief (clean, good output) / failed twice on full brief (500 proxy error, then 280s hard hang) | **No** (also has a separate n8n/LangChain incompatibility — `reasoning_content` field not handled) | Works for simple prompts, breaks down under full production complexity (photos + full instruction set). Not reliable enough to use. |

## Inconclusive

| Model | Type / size | Notes |
|---|---|---|
| `phi-4-Q4_K_M.gguf` | Dense, 14B (~9.05GB) | Two attempts both timed out (280s, then 400s) with no response at all. Server's `/v1/models` showed `status: "loading"` stuck for 11+ minutes — not a speed issue, the model never finished loading into VRAM. Possible causes: corrupted/incomplete download, disk space, or a llama-server crash not surfaced in the status report. Not retested — would need server-side log checking to diagnose (file integrity, disk space, process logs). Set aside for now. |

## Ruled out

| Model | Type / size | Result |
|---|---|---|
| `Qwythos-9B-Claude-Mythos-5-1M-Q4_K_M` | Dense reasoning, 9B, fine-tuned on Claude traces | At 8192 max_tokens, burned the entire budget on `reasoning_content` with empty `content`. Retested at 20,000 max_tokens with an explicit "don't mention Qwythos/Empero AI" instruction — that worked (no vendor branding in output, footer correctly said only the business name), but the actual page had real defects: phone/location/clock icon `<span>` elements were left completely empty (no SVG inside, despite explicit verbatim-path instructions), the clover emoji placeholder was also empty, address/phone/hours were redundantly duplicated across two sections, and weekly events/menu items were dumped as plain sentences instead of formatted lists. Took ~5 minutes (297s) with 25K+ characters of reasoning overhead for a weaker result than Qwen3.6 produces in a third the time. Confirmed not competitive. |
| `DeepSeek-Coder-V2-Lite-Instruct-Q4_K_M` | MoE, 16B total / 2.4B active, code-specialized | Clean response (`finish_reason: stop`, no reasoning overhead), but only 12.7 tokens/sec (171s wall time) — slower than expected for an MoE model on this hardware. Worse, multiple content-completeness failures: used a purple/teal gradient despite an explicit green-and-gold requirement, omitted the clover emoji entirely, **dropped all 3 provided photos** (no gallery at all), omitted weekly events and menu items completely, and produced malformed `tel:(708) 555-0100` links (parentheses/spaces in a `tel:` URI risk breaking click-to-call on real phones). Icons were at least geometrically correct. Structure was also messy/redundant (address/phone/hours repeated across 5 blocks). Confirmed not competitive. |

## Promising, not yet a clear winner

| Model | Type / size | Notes |
|---|---|---|
| `mistralai_Mistral-Small-3.2-24B-Instruct-2506-Q4_K_M` | Dense, 24B | 16.2 tokens/sec (170s wall time). Clean, correct execution overall: right green/gold palette, clover emoji worked in via CSS `::after`, one approved photo used as hero background with dark overlay, all 3 photos in a proper gallery, correct/matching icons, working `tel:` links, bonus Google Maps link. **But**: weekly events and menu items were both completely omitted despite being explicitly required, and there's a dead link (`<a href="#menu">` pointing to a nonexistent anchor). Strongest of the new contenders so far, but not yet matching Qwen3.6's content completeness. |

## Notes / gotchas learned along the way

- Cloudflare in front of `n8n.hannontech.net` hard-caps requests at 120s (Error 524) regardless of n8n's own per-node timeout. A 524 doesn't mean failure — check `n8n_executions` afterward; the workflow often completes successfully server-side past that window.
- `presets.ini` (`/opt/llama.cpp/presets.ini`) only needs an entry for *overriding* defaults (e.g. larger context). Models dropped into `/opt/llama.cpp/models/` auto-register under their filename.
- `models-max = 2` in `presets.ini` — only two models can stay loaded in 24GB VRAM at once; large models (~20GB each) can't coexist, so swapping has real cost.
- Reasoning models (anything emitting a `<think>` block or separate `reasoning_content` field) have been the least reliable category so far — both DeepSeek-R1-Distill and Gemma 4 had serious issues under full production complexity.
