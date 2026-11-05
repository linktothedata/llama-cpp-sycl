# llama-sycl

Auto-built llama.cpp with the SYCL backend for the Arc B65, published to
`ghcr.io/linktothedata/llama-sycl`. Same pattern as `whisper-sycl`.

## Layout

| File | Goes to |
|---|---|
| `llama-sycl.yml` | `.github/workflows/llama-sycl.yml` in the repo |
| `docker-compose.llama-sycl.yml` | fireball, e.g. `/home/fireball/docker/llama-sycl/` |

The workflow checks out **ggml-org/llama.cpp** and builds its in-tree
`.devops/intel.Dockerfile` (`server` target). Nothing is forked or patched —
upstream's build stage uses oneAPI with `icx`/`icpx`, and its runtime stage
installs pinned Intel IGC + compute-runtime + GMM `.deb`s.

## Running it

1. Run the workflow once (`Actions → Build llama.cpp SYCL → Run workflow`).
   First build is slow — the oneAPI image is enormous and the compile is on
   4 vCPUs. Budget an hour or more.
2. Make the package visible to fireball: GHCR packages default to private, so
   either `docker login ghcr.io` with a read:packages PAT on fireball, or flip
   the package to public in its GitHub settings.
3. Drop a GGUF in `models/` and `docker compose up -d`.

## Getting the GGUF without re-downloading 18 GB

Ollama blobs *are* GGUF files, just without an extension. On the ollama-sycl
container:

```bash
# find the model layer (the big one, usually ~18 GB for a 27B Q4)
ls -lhS /root/.ollama/models/blobs/ | head
```

Symlink or copy that blob into `models/` with a `.gguf` name. No re-pull.

## Benchmarking against ollama-sycl

The point of the exercise. Same blob, same host, same session:

```bash
# in the llama-sycl container
docker exec -it llama-sycl /app/llama-bench \
  -m /models/qwen3.6-27b-Q4_K_M.gguf -p 512 -n 128 -fa 1
```

Compare `pp512` and `tg128` against what ollama-sycl does on the same file.
If llama.cpp is inside ~15%, staying on Ollama is the right call — you keep
`OLLAMA_MAX_LOADED_MODELS=2` and the API that immich-analyze and Open WebUI
already point at.

## Knobs worth sweeping before concluding anything

- `-ub` (512 / 1024 / 2048) — biggest single lever on prefill for SYCL
- `--cache-type-k/v` — `q8_0` buys context headroom on a 31 GiB card
- `--flash-attn on|off` — not always a win on SYCL; measure it
- `GGML_SYCL_F16=OFF` build arg — FP32 fallback if F16 output looks wrong

## If the GPU isn't detected

Startup should log `Found 1 SYCL devices` with the B65 as `SYCL0`. If it
reports CPU-only, the `/usr/lib/wsl` mount or `LD_LIBRARY_PATH` ordering is
wrong — that's the same failure mode as the `ipex-llm-inference-cpp-xpu`
dead end, except here the compute runtime genuinely is in the image.

## Driver pin escape hatch

Upstream currently pins compute-runtime 26.18. There's a known multi-GPU bug
in the 26.x series (llama.cpp#21747), and upstream leaves the older 25.40
pins commented in the Dockerfile. Single GPU, so this shouldn't bite — but if
you hit driver-level weirdness, those ARGs are overridable via `build-args`
in the workflow without forking anything.
