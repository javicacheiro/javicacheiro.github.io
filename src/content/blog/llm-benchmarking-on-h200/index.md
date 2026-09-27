---
title: 'Serving LLMs on a single 8×H200 server'
description: 'Benchmarking six open-weight models, including MiMo-V2.6, on one 8×H200 server.'
publishDate: 2026-09-24
updatedDate: 2026-09-27
draft: true
tags:
  - ai
  - llm
  - gpu
  - benchmarking
  - inference
---

<style>
.qz {
  --ok: #0e9384;
  --bad: #c2410c;
  --vl: #4f5bd5;
  --grid: hsl(var(--border));
  --ink: hsl(var(--foreground));
  --dim: hsl(var(--muted-foreground));
  margin: 2rem 0;
}
.dark .qz { --ok: #2f9e8f; --bad: #cf6238; --vl: #8b93e8; }
.qz svg { width: 100%; height: auto; display: block; overflow: visible; }
.qz figcaption { font-size: .875rem; line-height: 1.55; color: var(--dim); margin-top: .8rem; }
.qz .lab { font-size: 12px; fill: var(--ink); }
.qz .sub { font-size: 10.5px; fill: var(--dim); }
.qz .val { font-size: 12px; font-weight: 600; fill: var(--ink); }
.qz .ax  { font-size: 11px; fill: var(--dim); }
</style>

Comparison of the performance of six open-weight models — GLM-5.3, GLM-5.3-Flash,
DeepSeek-V4.1-Flash, Qwen3.8-Flash-Next and Xiaomi's MiMo-V2.6 in both its Flash-RL
and Pro-RL sizes — on a H200 node with 8 GPUs, measured on the same seven
workload shapes I used for the [B300 comparison](/blog/throughput-benchmarking-on-b300/).

They are not all the same weight class: MiMo-V2.6-Pro-RL is 1.02T parameters (42B
active) and GLM-5.3 is 743B, against 309B for MiMo-V2.6-Flash-RL. What they have in
common is that each one fits on this single node.

## Method

One node with 8× NVIDIA H200 (SM90, 141 GiB each). TP8 throughout except Qwen's
SGLang cells, which are TP4. The flags used correspond to the ones from each model's published
recipe in the vLLM and SGLang docs. There are only three forced deviations: `--max-model-len 262144` for
GLM-5.3 under vLLM (the published command will not boot otherwise),
`--max-num-seqs 512` for GLM-5.3-Flash under data parallelism, and a raised
engine-start timeout.

The two MiMo-V2.6 checkpoints store their expert weights in MXFP4, which this
hardware has no native path for: H200 is SM90, and FP4 tensor cores arrive with
Blackwell. Their published recipe therefore pins `--moe-runner-backend marlin`
on H200 where it uses `deep_gemm` on B300, so every MiMo number below is MXFP4
dequantised through Marlin rather than computed in FP4. That is a property of
running these models on Hopper, not something to tune away.

Both MiMo checkpoints also ship a **DFlash** speculative drafter inside the
repository under `dflash/`, which turns out to matter more than any other single
flag here — see [Speculative decoding](#speculative-decoding-dflash) below.

GuideLLM 0.7.1, with seven workload shapes from 2,048 to 131,072 input tokens, runs
bounded by duration rather than prompt count. Same workloads as the ones used for the [B300 comparison](/blog/throughput-benchmarking-on-b300/).

The 256-stream runs are provisional due to some issues in the benchmarking procedure.

### The exact commands

Every configuration below, verbatim. Some boilerplate is common to all of them:
the HuggingFace cache is bind-mounted from `/fsx` because the root disk on this
node is too small to hold the weights, and `--ulimit memlock=-1` is there
because several of these models pin host memory for their state tables. Flags
otherwise come straight from each model's published `hw=h200` recipe cell.

### zai-org/GLM-5.3

**SGLang latency** — `sgl-glm53-lowlat`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:latest \
  sglang serve --model-path zai-org/GLM-5.3 \
  --tp 8 --speculative-algorithm EAGLE --speculative-num-steps 5 \
  --speculative-eagle-topk 1 --speculative-num-draft-tokens 6 \
  --mem-fraction-static 0.8 \
  --host 0.0.0.0 --port 30000
```

**SGLang throughput** — `sgl-glm53-hithru`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:latest \
  sglang serve --model-path zai-org/GLM-5.3 \
  --tp 8 --dp 8 --enable-dp-attention --moe-a2a-backend deepep \
  --speculative-algorithm EAGLE --speculative-num-steps 1 \
  --speculative-eagle-topk 1 --speculative-num-draft-tokens 2 \
  --mem-fraction-static 0.85 --chunked-prefill-size 32768 \
  --max-running-requests 256 \
  --host 0.0.0.0 --port 30000
```


### zai-org/GLM-5.3-Flash

**SGLang latency** — `sgl-flash-lowlat`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:glm-5.3-flash \
  sglang serve --model-path zai-org/GLM-5.3-Flash \
  --tp-size 8 --ep-size 8 --mem-fraction-static 0.75 \
  --dsa-prefill-backend tilelang --dsa-decode-backend tilelang \
  --kv-cache-dtype bfloat16 --moe-runner-backend deep_gemm \
  --speculative-algorithm EAGLE --speculative-num-steps 5 \
  --speculative-eagle-topk 1 --speculative-num-draft-tokens 6 \
  --reasoning-parser glm45 --tool-call-parser glm47 \
  --host 0.0.0.0 --port 30000
```

**SGLang throughput** — `sgl-flash-hithru`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:glm-5.3-flash \
  sglang serve --model-path zai-org/GLM-5.3-Flash \
  --tp-size 8 --ep-size 8 --dsa-prefill-backend tilelang \
  --dsa-decode-backend tilelang --kv-cache-dtype bfloat16 \
  --moe-runner-backend deep_gemm --reasoning-parser glm45 \
  --tool-call-parser glm47 \
  --host 0.0.0.0 --port 30000
```


### zai-org/GLM-5.3

**vLLM latency** — `vllm-glm53`

```bash
VLLM_ENGINE_READY_TIMEOUT_S=3600 \
vllm serve zai-org/GLM-5.3 \
  --max-model-len 262144 --kv-cache-dtype fp8 --tensor-parallel-size 8 \
  --speculative-config.method mtp \
  --speculative-config.num_speculative_tokens 5 --tool-call-parser glm47 \
  --reasoning-parser glm47 --enable-auto-tool-choice \
  --served-model-name glm-5.3 \
  --host 0.0.0.0 --port 8000
```


### zai-org/GLM-5.3-Flash

**vLLM latency** — `vllm-flash`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  -e VLLM_ENGINE_READY_TIMEOUT_S=3600 \
  vllm/vllm-openai:nightly \
  --model zai-org/GLM-5.3-Flash \
  --tensor-parallel-size 8 \
  --speculative-config {"method":"mtp","num_speculative_tokens":5} \
  --tool-call-parser glm47 --reasoning-parser glm47 \
  --enable-auto-tool-choice --served-model-name glm-5.3-flash \
  --host 0.0.0.0 --port 8000
```

**vLLM balanced** — `vllm-flash-tep`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  -e VLLM_ENGINE_READY_TIMEOUT_S=3600 \
  vllm/vllm-openai:nightly \
  --model zai-org/GLM-5.3-Flash \
  --tensor-parallel-size 8 --enable-expert-parallel \
  --speculative-config {"method":"mtp","num_speculative_tokens":5} \
  --tool-call-parser glm47 --reasoning-parser glm47 \
  --enable-auto-tool-choice --served-model-name glm-5.3-flash \
  --host 0.0.0.0 --port 8000
```

**vLLM throughput** — `vllm-flash-dep`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  -e VLLM_ENGINE_READY_TIMEOUT_S=3600 \
  vllm/vllm-openai:nightly \
  --model zai-org/GLM-5.3-Flash \
  --max-num-seqs 512 --data-parallel-size 8 --enable-expert-parallel \
  --speculative-config {"method":"mtp","num_speculative_tokens":5} \
  --tool-call-parser glm47 --reasoning-parser glm47 \
  --enable-auto-tool-choice --served-model-name glm-5.3-flash \
  --host 0.0.0.0 --port 8000
```


### zai-org/GLM-5.3

**vLLM balanced** — `vllm-glm53-tep`

```bash
VLLM_ENGINE_READY_TIMEOUT_S=3600 \
vllm serve zai-org/GLM-5.3 \
  --tensor-parallel-size 8 --enable-expert-parallel --max-model-len 262144 \
  --kv-cache-dtype fp8 --speculative-config.method mtp \
  --speculative-config.num_speculative_tokens 5 --tool-call-parser glm47 \
  --reasoning-parser glm47 --enable-auto-tool-choice \
  --served-model-name glm-5.3 \
  --host 0.0.0.0 --port 8000
```

**vLLM throughput** — `vllm-glm53-dep`

```bash
VLLM_ENGINE_READY_TIMEOUT_S=3600 \
vllm serve zai-org/GLM-5.3 \
  --data-parallel-size 8 --enable-expert-parallel --max-model-len 262144 \
  --kv-cache-dtype fp8 --speculative-config.method mtp \
  --speculative-config.num_speculative_tokens 5 --tool-call-parser glm47 \
  --reasoning-parser glm47 --enable-auto-tool-choice \
  --served-model-name glm-5.3 \
  --host 0.0.0.0 --port 8000
```


### deepseek-ai/DeepSeek-V4.1-Flash

**SGLang latency** — `sgl-dsv41-lowlat`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:dev-dsv41 \
  sglang serve --model-path deepseek-ai/DeepSeek-V4.1-Flash \
  --trust-remote-code --tp 8 --ep-size 8 --mem-fraction-static 0.8 \
  --attention-backend dsv4 --moe-runner-backend flashinfer_mxfp4 \
  --cuda-graph-max-bs-decode 64 --reasoning-parser auto \
  --tool-call-parser auto --enable-decoder-swa-bounded-replay \
  --host 0.0.0.0 --port 30000
```

**SGLang throughput** — `sgl-dsv41-hithru`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:dev-dsv41 \
  sglang serve --model-path deepseek-ai/DeepSeek-V4.1-Flash \
  --trust-remote-code --tp 8 --ep-size 8 --mem-fraction-static 0.8 \
  --attention-backend dsv4 --moe-runner-backend flashinfer_mxfp4 \
  --cuda-graph-max-bs-decode 64 --reasoning-parser auto \
  --tool-call-parser auto --max-running-requests 256 \
  --host 0.0.0.0 --port 30000
```


### Qwen/Qwen3.8-Flash-Next

**SGLang latency** — `sgl-qwen38-lowlat`

```bash
docker run --rm --gpus '"device=0,1,2,3"' --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:qwen38flashnext \
  sglang serve --model-path Qwen/Qwen3.8-Flash-Next \
  --tp 4 --mem-fraction-static 0.85 --chunked-prefill-size 8192 \
  --linear-attn-prefill-backend flashinfer \
  --linear-attn-decode-backend flashinfer --mamba-ssm-dtype bfloat16 \
  --reasoning-parser auto --linear-attn-verify-backend triton \
  --speculative-algorithm NEXTN --speculative-num-steps 3 \
  --speculative-eagle-topk 1 --speculative-num-draft-tokens 4 \
  --max-running-requests 96 \
  --host 0.0.0.0 --port 30000
```

**SGLang throughput** — `sgl-qwen38-hithru`

```bash
docker run --rm --gpus '"device=0,1,2,3"' --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:qwen38flashnext \
  sglang serve --model-path Qwen/Qwen3.8-Flash-Next \
  --tp 4 --mem-fraction-static 0.85 --chunked-prefill-size 8192 \
  --linear-attn-prefill-backend flashinfer \
  --linear-attn-decode-backend flashinfer --mamba-ssm-dtype bfloat16 \
  --reasoning-parser auto --ep 4 \
  --host 0.0.0.0 --port 30000
```


### Qwen/Qwen3.8-Flash-Next-FP8

**SGLang latency** — `sgl-qwen38fp8-lowlat`

```bash
docker run --rm --gpus '"device=0,1,2,3"' --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:qwen38flashnext \
  sglang serve --model-path Qwen/Qwen3.8-Flash-Next-FP8 \
  --tp 4 --ep 4 --mem-fraction-static 0.85 --chunked-prefill-size 8192 \
  --linear-attn-prefill-backend flashinfer \
  --linear-attn-decode-backend flashinfer --mamba-ssm-dtype bfloat16 \
  --reasoning-parser auto --linear-attn-verify-backend triton \
  --speculative-algorithm NEXTN --speculative-num-steps 3 \
  --speculative-eagle-topk 1 --speculative-num-draft-tokens 4 \
  --host 0.0.0.0 --port 30000
```

**SGLang throughput** — `sgl-qwen38fp8-hithru`

```bash
docker run --rm --gpus '"device=0,1,2,3"' --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:qwen38flashnext \
  sglang serve --model-path Qwen/Qwen3.8-Flash-Next-FP8 \
  --tp 4 --ep 4 --mem-fraction-static 0.85 --chunked-prefill-size 8192 \
  --linear-attn-prefill-backend flashinfer \
  --linear-attn-decode-backend flashinfer --mamba-ssm-dtype bfloat16 \
  --reasoning-parser auto \
  --host 0.0.0.0 --port 30000
```


### deepseek-ai/DeepSeek-V4.1-Flash

**vLLM latency** — `vllm-dsv41-tp`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  -e VLLM_ENGINE_READY_TIMEOUT_S=3600 \
  vllm/vllm-openai:nightly \
  --model deepseek-ai/DeepSeek-V4.1-Flash \
  --tensor-parallel-size 8 --language-model-only \
  --tokenizer-mode deepseek_v41 --tool-call-parser deepseek_v41 \
  --enable-auto-tool-choice --reasoning-parser deepseek_v41 \
  --gpu-memory-utilization 0.9 \
  --speculative-config {"method":"dspark","num_speculative_tokens":5,"draft_sample_method":"probabilistic","rejection_sample_method":"block","enable_adaptive_verification":false} \
  --max-model-len 262144 --max-num-seqs 128 --max-num-batched-tokens 16384 \
  --served-model-name dsv41 \
  --host 0.0.0.0 --port 8000
```

**vLLM throughput** — `vllm-dsv41-dep`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  -e VLLM_ENGINE_READY_TIMEOUT_S=3600 \
  vllm/vllm-openai:nightly \
  --model deepseek-ai/DeepSeek-V4.1-Flash \
  --data-parallel-size 8 --enable-expert-parallel --language-model-only \
  --tokenizer-mode deepseek_v41 --tool-call-parser deepseek_v41 \
  --enable-auto-tool-choice --reasoning-parser deepseek_v41 \
  --gpu-memory-utilization 0.9 \
  --speculative-config {"method":"dspark","num_speculative_tokens":5,"draft_sample_method":"probabilistic","rejection_sample_method":"block","enable_adaptive_verification":false} \
  --max-model-len 262144 --max-num-seqs 128 --max-num-batched-tokens 16384 \
  --served-model-name dsv41 \
  --host 0.0.0.0 --port 8000
```


### Qwen/Qwen3.8-Flash-Next-FP8

**vLLM balanced** — `vllm-qwen38fp8-tep`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  -e VLLM_ENGINE_READY_TIMEOUT_S=3600 \
  vllm/vllm-openai:qwen38-flash-next \
  --model Qwen/Qwen3.8-Flash-Next-FP8 \
  --tensor-parallel-size 8 --enable-expert-parallel --moe-backend triton \
  --gpu-memory-utilization 0.85 --max-num-seqs 256 --enable-prefix-caching \
  --no-enable-flashinfer-autotune --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder --reasoning-parser qwen3 \
  --served-model-name qwen38fp8 \
  --host 0.0.0.0 --port 8000
```

**vLLM throughput** — `vllm-qwen38fp8-dep`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  -e VLLM_ENGINE_READY_TIMEOUT_S=3600 \
  vllm/vllm-openai:qwen38-flash-next \
  --model Qwen/Qwen3.8-Flash-Next-FP8 \
  --data-parallel-size 8 --enable-expert-parallel --moe-backend triton \
  --gpu-memory-utilization 0.85 --max-num-seqs 256 --enable-prefix-caching \
  --no-enable-flashinfer-autotune --enable-auto-tool-choice \
  --tool-call-parser qwen3_coder --reasoning-parser qwen3 \
  --served-model-name qwen38fp8 \
  --host 0.0.0.0 --port 8000
```


### XiaomiMiMo/MiMo-V2.6-Flash-RL

Stable vLLM cannot load the MXFP4-stored weights at all, so these use the image
published for the series rather than a release tag.

**vLLM latency** — `vllm-flash-tp` (the published command, verbatim)

```bash
docker run --rm --gpus '"device=0,1,2,3"' --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  vllm/vllm-openai:mimo-v26 \
  --model XiaomiMiMo/MiMo-V2.6-Flash-RL \
  --tensor-parallel-size 4 --trust-remote-code --gpu-memory-utilization 0.95 \
  --max-model-len auto --reasoning-parser mimo --tool-call-parser mimo \
  --enable-auto-tool-choice --generation-config vllm \
  --host 0.0.0.0 --port 8000
```

**vLLM latency + DFlash** — `vllm-flash-dflash-r3`

`$DFLASH` is the `dflash/` directory inside the downloaded snapshot. Note the
`--gpu-memory-utilization 0.90`: at the published 0.95 the drafter has nowhere
to allocate and the engine dies with a CUDA OOM.

```bash
DFLASH=$(ls -d /fsx/hf-cache/hub/models--XiaomiMiMo--MiMo-V2.6-Flash-RL/snapshots/*/dflash)

docker run --rm --gpus '"device=0,1,2,3"' --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  vllm/vllm-openai:mimo-v26 \
  --model XiaomiMiMo/MiMo-V2.6-Flash-RL \
  --tensor-parallel-size 4 --trust-remote-code --gpu-memory-utilization 0.90 \
  --max-model-len auto \
  --speculative-config '{"method":"dflash","model":"'"$DFLASH"'","num_speculative_tokens":7}' \
  --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY"}' \
  --reasoning-parser mimo --tool-call-parser mimo \
  --enable-auto-tool-choice --generation-config vllm \
  --host 0.0.0.0 --port 8000
```

**vLLM balanced** — `vllm-flash-tep`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  vllm/vllm-openai:mimo-v26 \
  --model XiaomiMiMo/MiMo-V2.6-Flash-RL \
  --tensor-parallel-size 8 --enable-expert-parallel \
  --trust-remote-code --gpu-memory-utilization 0.95 --max-model-len auto \
  --reasoning-parser mimo --tool-call-parser mimo \
  --enable-auto-tool-choice --generation-config vllm \
  --host 0.0.0.0 --port 8000
```

**SGLang** — `sgl-flash-cell` (the published H200 cell, verbatim)

```bash
docker run --rm --gpus '"device=0,1,2,3"' --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:dev \
  sglang serve --model-path XiaomiMiMo/MiMo-V2.6-Flash-RL \
  --tp 4 --moe-runner-backend marlin --trust-remote-code \
  --reasoning-parser mimo --tool-call-parser mimo \
  --host 0.0.0.0 --port 30000
```

### XiaomiMiMo/MiMo-V2.6-Pro-RL

The 1T checkpoint. Identical flags, TP8, and the Pro repository.

**vLLM latency** — `vllm-pro-tp`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  vllm/vllm-openai:mimo-v26 \
  --model XiaomiMiMo/MiMo-V2.6-Pro-RL \
  --tensor-parallel-size 8 --trust-remote-code --gpu-memory-utilization 0.95 \
  --max-model-len auto --reasoning-parser mimo --tool-call-parser mimo \
  --enable-auto-tool-choice --generation-config vllm \
  --host 0.0.0.0 --port 8000
```

**vLLM latency + DFlash** — `vllm-pro-dflash-r3`

```bash
DFLASH=$(ls -d /fsx/hf-cache/hub/models--XiaomiMiMo--MiMo-V2.6-Pro-RL/snapshots/*/dflash)

docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 8000:8000 -v /fsx/hf-cache:/root/.cache/huggingface \
  vllm/vllm-openai:mimo-v26 \
  --model XiaomiMiMo/MiMo-V2.6-Pro-RL \
  --tensor-parallel-size 8 --trust-remote-code --gpu-memory-utilization 0.90 \
  --max-model-len auto \
  --speculative-config '{"method":"dflash","model":"'"$DFLASH"'","num_speculative_tokens":7}' \
  --compilation-config '{"cudagraph_mode":"FULL_DECODE_ONLY"}' \
  --reasoning-parser mimo --tool-call-parser mimo \
  --enable-auto-tool-choice --generation-config vllm \
  --host 0.0.0.0 --port 8000
```

**SGLang** — `sgl-pro-cell`

```bash
docker run --rm --gpus all --shm-size 32g --ulimit memlock=-1 --ipc=host \
  -p 30000:30000 -v /fsx/hf-cache:/root/.cache/huggingface \
  lmsysorg/sglang:dev \
  sglang serve --model-path XiaomiMiMo/MiMo-V2.6-Pro-RL \
  --tp 8 --moe-runner-backend marlin --trust-remote-code \
  --reasoning-parser mimo --tool-call-parser mimo \
  --host 0.0.0.0 --port 30000
```

## Single request performance

<figure class="qz">
  <svg viewBox="0 0 640 528" role="img" aria-label="Single-stream output throughput for 18 serving configurations. DeepSeek-V4.1-Flash 294, GLM-5.3 195, Qwen3.8-Flash-Next FP8 184, GLM-5.3-Flash 174, DeepSeek-V4.1-Flash 167, Qwen3.8-Flash-Next bf16 164, GLM-5.3-Flash 150, GLM-5.3 147, GLM-5.3-Flash 143, Qwen3.8-Flash-Next FP8 140, GLM-5.3 137, Qwen3.8-Flash-Next bf16 137, Qwen3.8-Flash-Next FP8 137, GLM-5.3-Flash 113, GLM-5.3-Flash 99, GLM-5.3 96, DeepSeek-V4.1-Flash 68, DeepSeek-V4.1-Flash 68.">
    <line x1="331" y1="20" x2="331" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="331" y="16" text-anchor="middle">100</text>
    <line x1="429" y1="20" x2="429" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="429" y="16" text-anchor="middle">200</text>
    <line x1="528" y1="20" x2="528" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="528" y="16" text-anchor="middle">300</text>
    <text class="ax" x="232" y="523" text-anchor="start">One request at a time · CHAT-S · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM latency · 3.32 ms / token</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="529" y="41">294</text>
    <text class="lab" x="224" y="66" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="76" text-anchor="end">SGLang latency · 4.71 ms / token</text>
    <rect x="232" y="56" width="192" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="432" y="67">195</text>
    <text class="lab" x="224" y="92" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="102" text-anchor="end">SGLang latency · 5.03 ms / token</text>
    <rect x="232" y="82" width="182" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="422" y="93">184</text>
    <text class="lab" x="224" y="118" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="128" text-anchor="end">SGLang latency · 5.31 ms / token</text>
    <rect x="232" y="108" width="172" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="412" y="119">174</text>
    <text class="lab" x="224" y="144" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="154" text-anchor="end">vLLM throughput · 5.68 ms / token</text>
    <rect x="232" y="134" width="165" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="405" y="145">167</text>
    <text class="lab" x="224" y="170" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="180" text-anchor="end">SGLang latency · 5.76 ms / token</text>
    <rect x="232" y="160" width="161" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="401" y="171">164</text>
    <text class="lab" x="224" y="196" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="206" text-anchor="end">vLLM latency · 6.10 ms / token</text>
    <rect x="232" y="186" width="148" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="388" y="197">150</text>
    <text class="lab" x="224" y="222" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="232" text-anchor="end">vLLM latency · 6.41 ms / token</text>
    <rect x="232" y="212" width="145" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="385" y="223">147</text>
    <text class="lab" x="224" y="248" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="258" text-anchor="end">vLLM balanced · 6.44 ms / token</text>
    <rect x="232" y="238" width="141" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="381" y="249">143</text>
    <text class="lab" x="224" y="274" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="284" text-anchor="end">vLLM balanced · 7.02 ms / token</text>
    <rect x="232" y="264" width="138" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="378" y="275">140</text>
    <text class="lab" x="224" y="300" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="310" text-anchor="end">vLLM balanced · 7.02 ms / token</text>
    <rect x="232" y="290" width="135" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="375" y="301">137</text>
    <text class="lab" x="224" y="326" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="336" text-anchor="end">SGLang throughput · 7.16 ms / token</text>
    <rect x="232" y="316" width="135" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="375" y="327">137</text>
    <text class="lab" x="224" y="352" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="362" text-anchor="end">SGLang throughput · 7.07 ms / token</text>
    <rect x="232" y="342" width="135" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="375" y="353">137</text>
    <text class="lab" x="224" y="378" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="388" text-anchor="end">SGLang throughput · 8.64 ms / token</text>
    <rect x="232" y="368" width="111" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="351" y="379">113</text>
    <text class="lab" x="224" y="404" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="414" text-anchor="end">vLLM throughput · 9.95 ms / token</text>
    <rect x="232" y="394" width="98" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="338" y="405">99</text>
    <text class="lab" x="224" y="430" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="440" text-anchor="end">SGLang throughput · 10.21 ms / token</text>
    <rect x="232" y="420" width="94" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="334" y="431">96</text>
    <text class="lab" x="224" y="456" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="466" text-anchor="end">SGLang latency · 14.97 ms / token</text>
    <rect x="232" y="446" width="67" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="307" y="457">68</text>
    <text class="lab" x="224" y="482" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="492" text-anchor="end">SGLang throughput · 15.00 ms / token</text>
    <rect x="232" y="472" width="67" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="307" y="483">68</text>
  </svg>
  <figcaption>
    A single request, CHAT-S shape (2,048 in → 512 out). The sublabel is the
    inter-token latency — how long a reader waits between words.
  </figcaption>
</figure>

DeepSeek-V4.1-Flash under vLLM is the fastest single stream in the set at
**294 tok/s and 3.3 ms between tokens** — and the same model under SGLang's
published H200 cell is the *slowest* at 68 tok/s. That 4.3× gap is the one
result here that is not a tuning trade-off: it is the same checkpoint on the
same GPUs, and part of it is speculative decoding, which vLLM runs and the
SGLang H200 cell does not. SGLang's two published DeepSeek cells also return
near-identical numbers at every point, so on this hardware they are effectively
one configuration.

Otherwise the pattern is the one you would expect: low-latency recipes on top,
high-throughput recipes at the bottom, and the largest model (GLM-5.3, 743B)
holding its own at 195 tok/s because its EAGLE 5-step draft is nearly free at
batch 1.

## Concurrent request performance

<figure class="qz">
  <svg viewBox="0 0 640 528" role="img" aria-label="Batch output throughput at 64 concurrent requests for 18 serving configurations. Qwen3.8-Flash-Next FP8 6,475, DeepSeek-V4.1-Flash 5,700, GLM-5.3-Flash 5,433, DeepSeek-V4.1-Flash 5,343, GLM-5.3-Flash 5,025, GLM-5.3-Flash 4,490, Qwen3.8-Flash-Next FP8 4,453, Qwen3.8-Flash-Next bf16 4,446, GLM-5.3-Flash 4,369, Qwen3.8-Flash-Next bf16 4,327, Qwen3.8-Flash-Next FP8 4,276, GLM-5.3-Flash 4,093, GLM-5.3 4,009, DeepSeek-V4.1-Flash 2,184, DeepSeek-V4.1-Flash 2,184, GLM-5.3 2,051, GLM-5.3 2,014, GLM-5.3 1,516.">
    <line x1="321" y1="20" x2="321" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="321" y="16" text-anchor="middle">2,000</text>
    <line x1="411" y1="20" x2="411" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="411" y="16" text-anchor="middle">4,000</text>
    <line x1="500" y1="20" x2="500" y2="491" stroke="var(--grid)"/>
    <text class="sub" x="500" y="16" text-anchor="middle">6,000</text>
    <text class="ax" x="232" y="523" text-anchor="start">Under load · BATCH-D · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM balanced · 64 concurrent</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="529" y="41">6,475</text>
    <text class="lab" x="224" y="66" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM throughput · 64 concurrent</text>
    <rect x="232" y="56" width="255" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="495" y="67">5,700</text>
    <text class="lab" x="224" y="92" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM balanced · 64 concurrent</text>
    <rect x="232" y="82" width="243" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="483" y="93">5,433</text>
    <text class="lab" x="224" y="118" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="128" text-anchor="end">vLLM latency · 64 concurrent</text>
    <rect x="232" y="108" width="239" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="479" y="119">5,343</text>
    <text class="lab" x="224" y="144" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="154" text-anchor="end">vLLM latency · 64 concurrent</text>
    <rect x="232" y="134" width="225" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="465" y="145">5,025</text>
    <text class="lab" x="224" y="170" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="180" text-anchor="end">vLLM throughput · 64 concurrent</text>
    <rect x="232" y="160" width="201" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="441" y="171">4,490</text>
    <text class="lab" x="224" y="196" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="206" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="186" width="199" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="439" y="197">4,453</text>
    <text class="lab" x="224" y="222" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="232" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="212" width="199" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="439" y="223">4,446</text>
    <text class="lab" x="224" y="248" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="258" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="238" width="195" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="435" y="249">4,369</text>
    <text class="lab" x="224" y="274" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="284" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="264" width="193" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="433" y="275">4,327</text>
    <text class="lab" x="224" y="300" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="310" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="290" width="191" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="431" y="301">4,276</text>
    <text class="lab" x="224" y="326" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="336" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="316" width="183" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="423" y="327">4,093</text>
    <text class="lab" x="224" y="352" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="362" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="342" width="179" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="419" y="353">4,009</text>
    <text class="lab" x="224" y="378" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="388" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="368" width="98" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="338" y="379">2,184</text>
    <text class="lab" x="224" y="404" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="414" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="394" width="98" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="338" y="405">2,184</text>
    <text class="lab" x="224" y="430" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="440" text-anchor="end">vLLM balanced · 64 concurrent</text>
    <rect x="232" y="420" width="92" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="332" y="431">2,051</text>
    <text class="lab" x="224" y="456" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="466" text-anchor="end">vLLM latency · 64 concurrent</text>
    <rect x="232" y="446" width="90" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="330" y="457">2,014</text>
    <text class="lab" x="224" y="482" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="492" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="472" width="68" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="308" y="483">1,516</text>
  </svg>
  <figcaption>
    Sustained output on BATCH-D (4,096 in → 8,192 out) at 64 concurrent requests.
    Not the peak across the sweep — peak falls in the 256-stream column, which is
    the one that did not reproduce.
  </figcaption>
</figure>

The order inverts almost completely: the low-latency recipes fall to the bottom
and the throughput-oriented ones rise. Qwen3.8-Flash-Next FP8 under vLLM leads at
6,475 tok/s, and the model that was fastest single-stream — DeepSeek under vLLM's
latency strategy — is now mid-table.


## Complete results

Aggregate output tok/s at 64 concurrent requests — the operating point where a
second repetition agreed within 6%, so these are the soundest numbers here.

| configuration | API-S | CHAT-S | CODE-I | CHAT-L | CODE-A | DOC-L | BATCH-D |
|---|--:|--:|--:|--:|--:|--:|--:|
| **GLM-5.3-Flash** | | | | | | | |
| SGLang latency | 759 | 1,675 | 1,608 | 964 | 957 | 264 | 4,093 |
| SGLang throughput | 1,529 | 2,840 | 1,873 | 1,248 | 868 | 222 | 4,369 |
| vLLM latency | 1,193 | 2,038 | 2,003 | 1,305 | 1,225 | 386 | 5,025 |
| vLLM balanced | 1,257 | 2,238 | 2,097 | 1,308 | 1,216 | 388 | 5,433 |
| vLLM throughput | 1,478 | 2,091 | 1,845 | 1,317 | 1,699 | 587 | 4,490 |
| **GLM-5.3** | | | | | | | |
| SGLang latency | 891 | 1,139 | 616 | 354 | 289 | 94 | 1,516 |
| SGLang throughput | 1,090 | 1,521 | 930 | 468 | 392 | *rej* | 4,009 |
| vLLM latency | 760 | 1,238 | 569 | 320 | 300 | 95 | 2,014 |
| vLLM balanced | 801 | 1,239 | 599 | 328 | 287 | 99 | 2,051 |
| vLLM throughput | — | — | — | — | — | — | — |
| **DeepSeek-V4.1-Flash** | | | | | | | |
| SGLang latency | 437 | 655 | 624 | 624 | *n/c* | *n/c* | 2,184 |
| SGLang throughput | 437 | 655 | 624 | *n/c* | *n/c* | *n/c* | 2,184 |
| vLLM latency | 1,620 | 2,927 | 1,731 | 1,006 | 965 | *n/c* | 5,343 |
| vLLM throughput | 1,331 | 2,095 | 1,333 | 746 | 992 | *rej* | 5,700 |
| **Qwen3.8-Flash-Next bf16** | | | | | | | |
| SGLang latency | 1,032 | 1,685 | 1,936 | 1,189 | 1,155 | 353 | 4,446 |
| SGLang throughput | 1,857 | 2,403 | 1,873 | 1,100 | 1,100 | 359 | 4,327 |
| **Qwen3.8-Flash-Next FP8** | | | | | | | |
| SGLang latency | 1,170 | 1,681 | 1,917 | 1,198 | 1,390 | 342 | 4,453 |
| SGLang throughput | 2,075 | 2,840 | 2,496 | 1,249 | 1,039 | 289 | 4,276 |
| vLLM balanced | 2,435 | 3,398 | 3,102 | 1,838 | 2,082 | 378 | 6,475 |
| vLLM throughput | — | — | — | — | — | — | — |
| **MiMo-V2.6-Flash-RL** | | | | | | | |
| SGLang | 2,075 | 2,838 | 1,986 | 1,043 | 825 | 266 | 4,358 |
| vLLM latency | 1,856 | 3,263 | 2,026 | 1,277 | 1,123 | 335 | 2,924 |
| vLLM latency + DFlash | 1,909 | 3,285 | 2,410 | 1,507 | 1,313 | 370 | 4,154 |
| vLLM balanced | 2,511 | 4,384 | 2,476 | 1,989 | 1,587 | 482 | 4,950 |
| vLLM throughput | 1,527 | 1,960 | 1,735 | 1,238 | 1,278 | 386 | 2,020 |
| **MiMo-V2.6-Pro-RL** | | | | | | | |
| SGLang | 1,311 | 1,960 | 1,285 | 664 | 571 | 195 | 2,869 |
| vLLM latency | 1,295 | 2,393 | 1,438 | 909 | 739 | 231 | 2,359 |
| vLLM latency + DFlash | 1,335 | 2,679 | 1,648 | 987 | 892 | 255 | 3,198 |
| vLLM balanced | 1,205 | 2,392 | 1,317 | 861 | 712 | 224 | 2,096 |
| vLLM throughput | 763 | 1,091 | 663 | 525 | 514 | 193 | 1,103 |

*rej* — the server refused every request. *n/c* — no request finished inside the window.

Adding the MiMo models turns what used to be a clean sweep into a three-way
split. Before they were measured, Qwen3.8-Flash-Next FP8 under vLLM won six of
the seven benchmarks. It now wins three:

| benchmark | best configuration | output tok/s |
|---|---|--:|
| API-S | MiMo-V2.6-Flash-RL, vLLM balanced | 2,511 |
| CHAT-S | MiMo-V2.6-Flash-RL, vLLM balanced | 4,384 |
| CODE-I | Qwen3.8-Flash-Next FP8, vLLM balanced | 3,102 |
| CHAT-L | MiMo-V2.6-Flash-RL, vLLM balanced | 1,989 |
| CODE-A | Qwen3.8-Flash-Next FP8, vLLM balanced | 2,082 |
| DOC-L | GLM-5.3-Flash, vLLM throughput | 587 |
| BATCH-D | Qwen3.8-Flash-Next FP8, vLLM balanced | 6,475 |

MiMo-V2.6-Flash-RL takes the conversational shapes and the 32k one; Qwen keeps
the code shapes and the decode-heavy batch shape; GLM-5.3-Flash still owns the
128k DOC-L column it led before. Three different models, and for five of the
seven the same strategy — vLLM with expert parallelism.

No configuration is good at everything. The best API-S cell is mid-table on
DOC-L; the best DOC-L cell is mid-table on API-S. If your traffic is one shape,
benchmark that shape.

## Speculative decoding: DFlash

The largest single improvement in this whole comparison is not a parallelism
strategy or an engine choice. It is one flag.

Both MiMo-V2.6 checkpoints ship a DFlash drafter inside the model repository.
Pointing `--speculative-config` at it, changing nothing else except dropping
`--gpu-memory-utilization` from 0.95 to 0.90 to leave the drafter room:

| output tok/s | MiMo-Flash | | MiMo-Pro | |
|---|--:|--:|--:|--:|
| | baseline | +DFlash | baseline | +DFlash |
| CHAT-S, 1 stream | 232 | **345** (+48%) | 140 | **273** (+95%) |
| CHAT-S, 64 streams | 3,263 | 3,285 (+1%) | 2,393 | 2,679 (+12%) |
| BATCH-D, 64 streams | 2,924 | 4,154 (+42%) | 2,359 | 3,198 (+36%) |
| DOC-L, 64 streams | 335 | 370 (+10%) | 231 | 255 (+10%) |

**DFlash nearly doubles the 1T model's single-stream throughput** and improves
every shape measured on both models. The shape of the gain is what you would
expect from speculation: largest where the GPU is idle waiting on a sequential
decode (one stream: +48% and +95%), smallest where 64 concurrent requests
already keep it busy (+1% and +12%). It does not disappear under load, though —
BATCH-D, which is decode-heavy at 8,192 output tokens, still gains 36–42%.

If you serve either of these models, this is the first thing to turn on.

### What would not run on H200

Two configurations from the published recipes cannot run on this hardware, and
both trace back to MXFP4 having no native SM90 path:

- **`--moe-a2a-backend deepep` together with Marlin.** SGLang rejects the
  combination outright: `Runner backend MoeRunnerBackend.MARLIN requires a fused
  func for a2a backend deepep, but none is registered`. B300 avoids this because
  it runs `deep_gemm`; H200 is forced onto Marlin, which has no DeepEP kernel.
- **`--attention-backend fa4`.** Fails during startup with
  `ValueError: Expected size in shape to be strictly positive, but got 0`, raised
  from CUTLASS's layout builder. Isolated by changing one variable at a time:
  removing expert parallelism and keeping fa4 still fails; removing fa4 and
  keeping expert parallelism serves normally. This is a narrow claim — this
  model, this image, this node — not a general statement about FA4 on Hopper.

The MiMo cookbook page's prose recommends keeping both of those on H200. Its
machine-readable H200 cell omits both. **The cell is right and the prose is
wrong**, which is a good argument for reading the config rather than the page.
And the reduced version of the prose configuration that H200 *can* run still
loses to the published cell: 167 against 201 tok/s single-stream, 2,151 against
2,838 at 64 streams.

## Throughput vs concurrency

<figure class="qz">
  <svg viewBox="0 0 640 318" role="img" aria-label="Output throughput against concurrency on BATCH-D for three GLM-5.3-Flash configurations. SGLang latency: 205 at c1, 2,158 at c16, 4,093 at c64; SGLang throughput: 171 at c1, 1,638 at c16, 4,369 at c64; vLLM throughput: 137 at c1, 1,403 at c16, 4,490 at c64.">
    <rect x="58" y="8" width="13" height="3" rx="1.5" fill="var(--ok)"/>
    <text class="sub" x="76" y="12.5">SGLang latency</text>
    <rect x="162" y="8" width="13" height="3" rx="1.5" fill="var(--bad)"/>
    <text class="sub" x="180" y="12.5">SGLang throughput</text>
    <rect x="284" y="8" width="13" height="3" rx="1.5" fill="var(--vl)"/>
    <text class="sub" x="302" y="12.5">vLLM throughput</text>
    <line x1="58" y1="87" x2="566" y2="87" stroke="var(--grid)"/>
    <text class="sub" x="49" y="91" text-anchor="end">4k</text>
    <text class="sub" x="58" y="290" text-anchor="middle">1</text>
    <text class="sub" x="312" y="290" text-anchor="middle">16</text>
    <text class="sub" x="566" y="290" text-anchor="middle">64</text>
    <text class="ax" x="312" y="306" text-anchor="middle">concurrent requests</text>
    <polyline points="58,263 312,172 566,83" fill="none" stroke="var(--ok)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="263" r="3.5" fill="var(--ok)"/>
    <circle cx="312" cy="172" r="3.5" fill="var(--ok)"/>
    <circle cx="566" cy="83" r="3.5" fill="var(--ok)"/>
    <polyline points="58,264 312,196 566,70" fill="none" stroke="var(--bad)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="264" r="3.5" fill="var(--bad)"/>
    <circle cx="312" cy="196" r="3.5" fill="var(--bad)"/>
    <circle cx="566" cy="70" r="3.5" fill="var(--bad)"/>
    <polyline points="58,266 312,207 566,65" fill="none" stroke="var(--vl)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="266" r="3.5" fill="var(--vl)"/>
    <circle cx="312" cy="207" r="3.5" fill="var(--vl)"/>
    <circle cx="566" cy="65" r="3.5" fill="var(--vl)"/>
    <text class="val" x="574" y="68" fill="var(--vl)">4,490</text>
    <text class="val" x="574" y="81" fill="var(--bad)">4,369</text>
    <text class="val" x="574" y="94" fill="var(--ok)">4,093</text>
  </svg>
  <figcaption>
    GLM-5.3-Flash on BATCH-D. The low-latency recipe leads at low concurrency and
    is overtaken between 16 and 64 concurrent requests.
  </figcaption>
</figure>

Every model shows this shape. What is **not** portable is where the lines cross:

| model | crossover |
|---|---|
| DeepSeek-V4.1-Flash | below 16 concurrent requests |
| GLM-5.3 | between 16 and 64 |
| GLM-5.3-Flash | between 16 and 64 |
| Qwen3.8-Flash-Next (bf16 and FP8) | above 64 |

A rule of thumb learned on one model picks the wrong recipe for another. If you
serve Qwen at 32 concurrent requests, the low-latency recipe is still the faster
choice; at the same load DeepSeek has already crossed over.

## Qwen3.8-Flash-Next Quantization comparison

Comparison of Qwen3.8-Flash-Next BF16 checkpoint (336 GB) and FP8 (173 GB) checkpoints:

| Qwen3.8-Flash-Next | API-S | CHAT-S | CODE-I | BATCH-D c64 |
|---|--:|--:|--:|--:|
| BF16, SGLang throughput | 1,857 | 2,403 | 1,873 | 4,327 |
| FP8, SGLang throughput | 2,075 | 2,840 | 2,496 | 4,276 |

On the decode-heavy shape the two are within 1.2% — half the weight memory for
no measurable throughput change. On the shorter shapes FP8 is ahead by 12–33%,
which is the more useful result if your traffic looks like API-S or CHAT-S.

(The 256-stream peaks are left out of this comparison on purpose: the only sound
figure for either checkpoint at that concurrency is a re-measurement of the FP8
one, so a BF16-vs-FP8 comparison there would be mixing two different
measurement windows.)

## Conclusion

Pick the tuning for the load you actually expect. Within a single model the
low-latency and throughput recipes are different operating points, not better
and worse versions of each other, and the crossover between them sits somewhere
different for every model — so the choice cannot be carried over from one
deployment to the next.

The engine matters less than the tuning for GLM-5.3-Flash, where SGLang and vLLM
land within a few percent of each other at 64 concurrent requests. It matters a
great deal elsewhere: vLLM is 17–51% ahead on Qwen3.8-Flash-Next FP8 and **2.5–4.5×
ahead on DeepSeek-V4.1-Flash**, on the same checkpoint and the same GPUs. Which
way it goes is model-specific.

Look for a speculative drafter before tuning anything else. Enabling DFlash on
MiMo-V2.6 beat every parallelism change tried on that model — **+95% on
single-stream Pro** — and it is one flag pointing at a directory that is already
inside the checkpoint. That is a better return than any strategy choice in this
comparison.

Read the machine-readable recipe, not the page around it. Both MiMo
configurations that refused to start on H200 were things the cookbook's prose
recommends and its own H200 cell omits, and both traced back to the same cause:
MXFP4 has no native path on SM90, so Marlin is forced, and Marlin rules out the
DeepEP kernel the prose asks for. The published cell had already accounted for
the hardware.

Which is the practical point: none of this transfers. Benchmark your model, your
engine and your workload shape before production, because every one of those
three changed the answer here.
