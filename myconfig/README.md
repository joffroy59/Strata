# `myconfig/` — run configs for the Q2_0 model

This folder holds the run configuration that the Strata server uses to launch the
engine for `qwen3.8-flash-next-q2_0`. Two files are present:

| File | What it is |
|---|---|
| `strata-q2_0.json` | The **live** config. This is the one the server reads when it starts. |
| `strata-q2_0.json.bak` | The **previous** config, kept automatically by `setup.py` whenever it rewrites the live file (see `write_setup_config` in `setup.py`). It is a backup only; nothing reads it at runtime. |

The `.bak` is never deleted by setup. If you want to go back to the old settings, copy
`strata-q2_0.json.bak` over `strata-q2_0.json`.

---

## What the two files say

Both files point at the same engine, the same pack, the same model shards, the same
tokenizer, and the same vision setup. They differ in four places.

| Setting | `strata-q2_0.json` (live) | `strata-q2_0.json.bak` (old) | Effect |
|---|---|---|---|
| `port` | `8181` | `8080` | The HTTP API listens on `http://127.0.0.1:8181` now, previously `:8080`. Update the base URL in any client that pointed at the old port. |
| `--max-context` | `65536` | `131072` | The live config caps the context at 64K tokens. The old one allowed 128K, which needs more VRAM for the KV cache and is slower at the top end. |
| `--pcie-frac` | absent (engine default) | `0.00` | The old config pinned the PCIe share of expert misses to zero, which forces byte-identical repeats for A/B runs but costs decode speed. The live config lets the engine choose the fraction, so decoding is faster but repeated prompts are not bit-identical. See `docs/DETAILS.md` on `--pcie-frac`. |
| `--spec-min-p` | `0.5` | `0.50` | Same value, written differently by a different setup run. No behavioural difference. |

Everything else is identical in both files:

| Setting | Value (both files) |
|---|---|
| `exe` | `I:\IA\Strata\engine\strata.exe` |
| `--pack` | `I:\IA\Strata-data\packs\q2_0` |
| `--native` | `I:\IA\Strata-data\models\Q2_0\Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00001-of-00002.gguf` |
| `--ple-gguf` | `I:\IA\Strata-data\models\Q2_0\Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00002-of-00002.gguf` |
| `--expert-profile` | `I:\IA\Strata\data\expert-profile.bin` |
| `--expert-cache` | `auto` |
| `--prefill` | `auto` |
| `--spec` | `4` |
| `--mtp` | `I:\IA\Strata-data\mtp\rt` |
| `--kv` | `int8` |
| `--kv-resident` | `32768` |
| `--vision` | enabled (flag, no value) |
| `--vram-reserve-mib` | `700` |
| `cwd` | `I:\IA\Strata` |
| `tokenizer` | `I:\IA\Strata-data\packs\q2_0\tokenizer` |
| `model_name` | `qwen3.8-flash-next-q2_0` |
| `log` | `I:\IA\Strata\strata-q2_0.log` |
| `lib_dirs` | `I:\IA\Strata\.venv\Lib\site-packages\nvidia\cu13\bin\x86_64` |
| `gpu` | `0` |
| `gpus_asked` | `true` |
| `vision.exe` | `I:\IA\Strata\engine\strata-vision.exe` |
| `vision.mmproj` | `I:\IA\Strata-data\models\mmproj-Qwen3.8-Flash-Next-BF16.gguf` |
| `vision.model` | `I:\IA\Strata-data\models\Q2_0\Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00001-of-00002.gguf` |
| `vision.gpu` | `true` |
| `vision.max_tokens` | `1024` |

---

## Key-by-key reference

Top-level keys:

- `exe` — path to the engine binary (`engine\strata.exe`).
- `args` — the engine command line, as a flat list of flag/value pairs.
- `cwd` — working directory the engine is started in.
- `tokenizer` — directory holding the tokenizer files.
- `model_name` — the name the API reports for the model.
- `log` — where the engine writes its log.
- `lib_dirs` — extra directories searched for CUDA DLLs.
- `port` — the port the server binds to.
- `gpu` / `gpus_asked` — which GPU index to use, and whether multi-GPU was requested.
- `vision` — the separate vision engine: its `exe`, the `mmproj` file, the base model,
  whether it uses the GPU, and `max_tokens` for image captions.

Engine flags in `args`:

- `--pack` — the pack directory (blobs, ring slots, pack metadata).
- `--native` — the main model shard.
- `--ple-gguf` — the second shard (the PLE/long-context part).
- `--expert-profile` — the profile used to place experts.
- `--expert-cache auto` — let the engine size the expert cache for the available VRAM.
- `--prefill auto` — read prompts in chunks of up to 8,192 tokens.
- `--spec 4` — speculative decoding draft length.
- `--spec-min-p 0.5` — how sure the draft must be before adding another guess.
- `--mtp` — multi-token-prediction model directory.
- `--max-context` — maximum context length in tokens.
- `--kv int8` — 8-bit KV cache.
- `--kv-resident 32768` — how much KV stays resident in VRAM before streaming starts.
- `--vision` — enable the vision path.
- `--vram-reserve-mib 700` — keep 700 MiB of VRAM free for the expert cache.
- `--pcie-frac` — share of expert misses copied over PCIe (only in the `.bak`).

---

## Notes

- The live config was written by a later setup run, which dropped `--pcie-frac` and
  lowered `--max-context` to 65536. Setup keeps keys it does not write, but it chooses
  the engine options itself, so those two are the ones that changed.
- There are also copies of these files at the workspace root (`strata-q2_0.json`,
  `strata-q2_0.json.bak`). `myconfig/` is the folder setup reads and writes; keep the
  root copies in sync if you edit one by hand.
- To restore the 128K context and the deterministic `--pcie-frac 0.00` behaviour, copy
  the `.bak` over the live file and restart the server.
