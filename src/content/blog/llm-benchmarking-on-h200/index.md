---
title: 'Serving LLMs on a single 8×H200 server'
description: 'Benchmarking six open-weight models, including MiMo-V2.6, on one 8×H200 server.'
publishDate: 2026-09-24
updatedDate: 2026-09-27
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
/* Model series. Colour follows the model, never its rank, so a chart that
   drops a series never repaints the survivors. Slots 1-3 reuse the strategy
   colours above. Both orders pass the categorical checks (lightness band,
   chroma floor, CVD separation, normal-vision floor, contrast >= 3:1). */
.qz { --s1: #0e9384; --s2: #c2410c; --s3: #4f5bd5; --s4: #a16207;
      --s5: #0284c7; --s6: #be185d; --s7: #6941c6; }
.dark .qz { --ok: #2f9e8f; --bad: #cf6238; --vl: #8b93e8;
      --s1: #2f9e8f; --s2: #cf6238; --s3: #6b76d4; --s4: #b0841c;
      --s5: #2f90c8; --s6: #d1598d; --s7: #9468d4; }
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
  <svg viewBox="0 0 640 788" role="img" aria-label="Single-stream output throughput for 28 serving configurations. MiMo-V2.6-Flash-RL 345, DeepSeek-V4.1-Flash 294, MiMo-V2.6-Pro-RL 273, MiMo-V2.6-Flash-RL 232, MiMo-V2.6-Flash-RL 201, GLM-5.3 195, Qwen3.8-Flash-Next FP8 184, GLM-5.3-Flash 174, MiMo-V2.6-Flash-RL 174, DeepSeek-V4.1-Flash 167, Qwen3.8-Flash-Next bf16 164, GLM-5.3-Flash 150, GLM-5.3 147, GLM-5.3-Flash 143, Qwen3.8-Flash-Next FP8 140, MiMo-V2.6-Pro-RL 140, GLM-5.3 137, Qwen3.8-Flash-Next bf16 137, Qwen3.8-Flash-Next FP8 137, MiMo-V2.6-Pro-RL 123, GLM-5.3-Flash 113, MiMo-V2.6-Pro-RL 113, MiMo-V2.6-Flash-RL 109, GLM-5.3-Flash 99, GLM-5.3 96, DeepSeek-V4.1-Flash 68, DeepSeek-V4.1-Flash 68, MiMo-V2.6-Pro-RL 55.">
    <line x1="316" y1="20" x2="316" y2="751" stroke="var(--grid)"/>
    <text class="sub" x="316" y="16" text-anchor="middle">100</text>
    <line x1="400" y1="20" x2="400" y2="751" stroke="var(--grid)"/>
    <text class="sub" x="400" y="16" text-anchor="middle">200</text>
    <line x1="484" y1="20" x2="484" y2="751" stroke="var(--grid)"/>
    <text class="sub" x="484" y="16" text-anchor="middle">300</text>
    <text class="ax" x="232" y="783" text-anchor="start">One request at a time · CHAT-S · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM latency + DFlash · 2.70 ms / token</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="529" y="41">345</text>
    <text class="lab" x="224" y="66" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM latency · 3.32 ms / token</text>
    <rect x="232" y="56" width="246" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="486" y="67">294</text>
    <text class="lab" x="224" y="92" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM latency + DFlash · 3.38 ms / token</text>
    <rect x="232" y="82" width="229" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="469" y="93">273</text>
    <text class="lab" x="224" y="118" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="128" text-anchor="end">vLLM latency · 4.10 ms / token</text>
    <rect x="232" y="108" width="195" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="435" y="119">232</text>
    <text class="lab" x="224" y="144" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="154" text-anchor="end">SGLang · 4.83 ms / token</text>
    <rect x="232" y="134" width="169" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="409" y="145">201</text>
    <text class="lab" x="224" y="170" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="180" text-anchor="end">SGLang latency · 4.71 ms / token</text>
    <rect x="232" y="160" width="163" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="403" y="171">195</text>
    <text class="lab" x="224" y="196" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="206" text-anchor="end">SGLang latency · 5.03 ms / token</text>
    <rect x="232" y="186" width="155" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="395" y="197">184</text>
    <text class="lab" x="224" y="222" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="232" text-anchor="end">SGLang latency · 5.31 ms / token</text>
    <rect x="232" y="212" width="146" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="386" y="223">174</text>
    <text class="lab" x="224" y="248" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="258" text-anchor="end">vLLM balanced · 5.60 ms / token</text>
    <rect x="232" y="238" width="146" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="386" y="249">174</text>
    <text class="lab" x="224" y="274" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="284" text-anchor="end">vLLM throughput · 5.68 ms / token</text>
    <rect x="232" y="264" width="140" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="380" y="275">167</text>
    <text class="lab" x="224" y="300" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="310" text-anchor="end">SGLang latency · 5.76 ms / token</text>
    <rect x="232" y="290" width="137" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="377" y="301">164</text>
    <text class="lab" x="224" y="326" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="336" text-anchor="end">vLLM latency · 6.10 ms / token</text>
    <rect x="232" y="316" width="126" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="366" y="327">150</text>
    <text class="lab" x="224" y="352" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="362" text-anchor="end">vLLM latency · 6.41 ms / token</text>
    <rect x="232" y="342" width="123" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="363" y="353">147</text>
    <text class="lab" x="224" y="378" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="388" text-anchor="end">vLLM balanced · 6.44 ms / token</text>
    <rect x="232" y="368" width="120" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="360" y="379">143</text>
    <text class="lab" x="224" y="404" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="414" text-anchor="end">vLLM balanced · 7.02 ms / token</text>
    <rect x="232" y="394" width="117" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="357" y="405">140</text>
    <text class="lab" x="224" y="430" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="440" text-anchor="end">vLLM latency · 6.84 ms / token</text>
    <rect x="232" y="420" width="117" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="357" y="431">140</text>
    <text class="lab" x="224" y="456" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="466" text-anchor="end">vLLM balanced · 7.02 ms / token</text>
    <rect x="232" y="446" width="115" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="355" y="457">137</text>
    <text class="lab" x="224" y="482" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="492" text-anchor="end">SGLang throughput · 7.16 ms / token</text>
    <rect x="232" y="472" width="115" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="355" y="483">137</text>
    <text class="lab" x="224" y="508" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="518" text-anchor="end">SGLang throughput · 7.07 ms / token</text>
    <rect x="232" y="498" width="115" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="355" y="509">137</text>
    <text class="lab" x="224" y="534" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="544" text-anchor="end">SGLang · 7.28 ms / token</text>
    <rect x="232" y="524" width="103" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="343" y="535">123</text>
    <text class="lab" x="224" y="560" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="570" text-anchor="end">SGLang throughput · 8.64 ms / token</text>
    <rect x="232" y="550" width="95" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="335" y="561">113</text>
    <text class="lab" x="224" y="586" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="596" text-anchor="end">vLLM balanced · 8.51 ms / token</text>
    <rect x="232" y="576" width="95" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="335" y="587">113</text>
    <text class="lab" x="224" y="612" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="622" text-anchor="end">vLLM throughput · 9.05 ms / token</text>
    <rect x="232" y="602" width="92" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="332" y="613">109</text>
    <text class="lab" x="224" y="638" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="648" text-anchor="end">vLLM throughput · 9.95 ms / token</text>
    <rect x="232" y="628" width="83" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="323" y="639">99</text>
    <text class="lab" x="224" y="664" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="674" text-anchor="end">SGLang throughput · 10.21 ms / token</text>
    <rect x="232" y="654" width="80" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="320" y="665">96</text>
    <text class="lab" x="224" y="690" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="700" text-anchor="end">SGLang latency · 14.97 ms / token</text>
    <rect x="232" y="680" width="57" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="297" y="691">68</text>
    <text class="lab" x="224" y="716" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="726" text-anchor="end">SGLang throughput · 15.00 ms / token</text>
    <rect x="232" y="706" width="57" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="297" y="717">68</text>
    <text class="lab" x="224" y="742" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="752" text-anchor="end">vLLM throughput · 19.15 ms / token</text>
    <rect x="232" y="732" width="46" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="286" y="743">55</text>
  </svg>
  <figcaption>
    A single request, CHAT-S shape (2,048 in → 512 out). The sublabel is the
    inter-token latency — how long a reader waits between words.
  </figcaption>
</figure>

MiMo-V2.6-Flash-RL with DFlash is the fastest single stream in the set at
**345 tok/s**, and MiMo-V2.6-Pro-RL with DFlash is third at 273 — a 1T model
sitting above every 300–700B configuration here, on the strength of one
speculative-decoding flag. Without it the same two checkpoints fall to 232 and
140.

Behind them, DeepSeek-V4.1-Flash under vLLM reaches **294 tok/s at 3.3 ms
between tokens** — and the same model under SGLang's published H200 cell is the
*slowest* in the chart at 68 tok/s. That 4.3× gap is the one
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
  <svg viewBox="0 0 640 788" role="img" aria-label="Batch output throughput at 64 concurrent requests for 28 serving configurations. Qwen3.8-Flash-Next FP8 6,475, DeepSeek-V4.1-Flash 5,700, GLM-5.3-Flash 5,433, DeepSeek-V4.1-Flash 5,343, GLM-5.3-Flash 5,025, MiMo-V2.6-Flash-RL 4,950, GLM-5.3-Flash 4,490, Qwen3.8-Flash-Next FP8 4,453, Qwen3.8-Flash-Next bf16 4,446, GLM-5.3-Flash 4,369, MiMo-V2.6-Flash-RL 4,358, Qwen3.8-Flash-Next bf16 4,327, Qwen3.8-Flash-Next FP8 4,276, MiMo-V2.6-Flash-RL 4,154, GLM-5.3-Flash 4,093, GLM-5.3 4,009, MiMo-V2.6-Pro-RL 3,198, MiMo-V2.6-Flash-RL 2,924, MiMo-V2.6-Pro-RL 2,869, MiMo-V2.6-Pro-RL 2,359, DeepSeek-V4.1-Flash 2,184, DeepSeek-V4.1-Flash 2,184, MiMo-V2.6-Pro-RL 2,096, GLM-5.3 2,051, MiMo-V2.6-Flash-RL 2,020, GLM-5.3 2,014, GLM-5.3 1,516, MiMo-V2.6-Pro-RL 1,103.">
    <line x1="321" y1="20" x2="321" y2="751" stroke="var(--grid)"/>
    <text class="sub" x="321" y="16" text-anchor="middle">2,000</text>
    <line x1="411" y1="20" x2="411" y2="751" stroke="var(--grid)"/>
    <text class="sub" x="411" y="16" text-anchor="middle">4,000</text>
    <line x1="500" y1="20" x2="500" y2="751" stroke="var(--grid)"/>
    <text class="sub" x="500" y="16" text-anchor="middle">6,000</text>
    <text class="ax" x="232" y="783" text-anchor="start">Under load · BATCH-D · 64 concurrent · output tok/s</text>
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
    <text class="lab" x="224" y="170" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="180" text-anchor="end">vLLM balanced · 64 concurrent</text>
    <rect x="232" y="160" width="221" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="461" y="171">4,950</text>
    <text class="lab" x="224" y="196" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="206" text-anchor="end">vLLM throughput · 64 concurrent</text>
    <rect x="232" y="186" width="201" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="441" y="197">4,490</text>
    <text class="lab" x="224" y="222" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="232" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="212" width="199" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="439" y="223">4,453</text>
    <text class="lab" x="224" y="248" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="258" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="238" width="199" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="439" y="249">4,446</text>
    <text class="lab" x="224" y="274" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="284" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="264" width="195" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="435" y="275">4,369</text>
    <text class="lab" x="224" y="300" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="310" text-anchor="end">SGLang · 64 concurrent</text>
    <rect x="232" y="290" width="195" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="435" y="301">4,358</text>
    <text class="lab" x="224" y="326" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="336" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="316" width="193" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="433" y="327">4,327</text>
    <text class="lab" x="224" y="352" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="362" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="342" width="191" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="431" y="353">4,276</text>
    <text class="lab" x="224" y="378" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="388" text-anchor="end">vLLM latency + DFlash · 64 concurrent</text>
    <rect x="232" y="368" width="186" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="426" y="379">4,154</text>
    <text class="lab" x="224" y="404" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="414" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="394" width="183" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="423" y="405">4,093</text>
    <text class="lab" x="224" y="430" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="440" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="420" width="179" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="419" y="431">4,009</text>
    <text class="lab" x="224" y="456" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="466" text-anchor="end">vLLM latency + DFlash · 64 concurrent</text>
    <rect x="232" y="446" width="143" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="383" y="457">3,198</text>
    <text class="lab" x="224" y="482" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="492" text-anchor="end">vLLM latency · 64 concurrent</text>
    <rect x="232" y="472" width="131" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="371" y="483">2,924</text>
    <text class="lab" x="224" y="508" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="518" text-anchor="end">SGLang · 64 concurrent</text>
    <rect x="232" y="498" width="128" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="368" y="509">2,869</text>
    <text class="lab" x="224" y="534" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="544" text-anchor="end">vLLM latency · 64 concurrent</text>
    <rect x="232" y="524" width="105" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="345" y="535">2,359</text>
    <text class="lab" x="224" y="560" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="570" text-anchor="end">SGLang throughput · 64 concurrent</text>
    <rect x="232" y="550" width="98" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="338" y="561">2,184</text>
    <text class="lab" x="224" y="586" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="596" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="576" width="98" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="338" y="587">2,184</text>
    <text class="lab" x="224" y="612" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="622" text-anchor="end">vLLM balanced · 64 concurrent</text>
    <rect x="232" y="602" width="94" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="334" y="613">2,096</text>
    <text class="lab" x="224" y="638" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="648" text-anchor="end">vLLM balanced · 64 concurrent</text>
    <rect x="232" y="628" width="92" height="14" rx="3" fill="var(--vl)"/>
    <text class="val" x="332" y="639">2,051</text>
    <text class="lab" x="224" y="664" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="674" text-anchor="end">vLLM throughput · 64 concurrent</text>
    <rect x="232" y="654" width="90" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="330" y="665">2,020</text>
    <text class="lab" x="224" y="690" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="700" text-anchor="end">vLLM latency · 64 concurrent</text>
    <rect x="232" y="680" width="90" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="330" y="691">2,014</text>
    <text class="lab" x="224" y="716" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="726" text-anchor="end">SGLang latency · 64 concurrent</text>
    <rect x="232" y="706" width="68" height="14" rx="3" fill="var(--ok)"/>
    <text class="val" x="308" y="717">1,516</text>
    <text class="lab" x="224" y="742" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="752" text-anchor="end">vLLM throughput · 64 concurrent</text>
    <rect x="232" y="732" width="49" height="14" rx="3" fill="var(--bad)"/>
    <text class="val" x="289" y="743">1,103</text>
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


## Results per benchmark

The same 64-stream measurement, one panel per workload shape, each model shown
with whichever of its configurations was fastest on that shape. The point of
splitting it out is that the ordering genuinely changes between panels — no
model wins everywhere, and two of them win nothing.

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="Best output throughput per model on API-S at 64 concurrent requests. MiMo-V2.6-Flash-RL 2,511, Qwen3.8-Flash-Next FP8 2,435, Qwen3.8-Flash-Next bf16 1,857, DeepSeek-V4.1-Flash 1,620, GLM-5.3-Flash 1,529, MiMo-V2.6-Pro-RL 1,335, GLM-5.3 1,090.">
    <line x1="347" y1="20" x2="347" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="347" y="16" text-anchor="middle">1,000</text>
    <line x1="462" y1="20" x2="462" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="462" y="16" text-anchor="middle">2,000</text>
    <text class="ax" x="232" y="237" text-anchor="start">API-S · 2k in / 256 out · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--s6)"/>
    <text class="val" x="529" y="41">2,511</text>
    <text class="lab" x="224" y="66" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="56" width="280" height="14" rx="3" fill="var(--s5)"/>
    <text class="val" x="520" y="67">2,435</text>
    <text class="lab" x="224" y="92" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="102" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="82" width="214" height="14" rx="3" fill="var(--s4)"/>
    <text class="val" x="454" y="93">1,857</text>
    <text class="lab" x="224" y="118" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="128" text-anchor="end">vLLM latency</text>
    <rect x="232" y="108" width="187" height="14" rx="3" fill="var(--s3)"/>
    <text class="val" x="427" y="119">1,620</text>
    <text class="lab" x="224" y="144" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="154" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="134" width="176" height="14" rx="3" fill="var(--s2)"/>
    <text class="val" x="416" y="145">1,529</text>
    <text class="lab" x="224" y="170" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="180" text-anchor="end">vLLM latency + DFlash</text>
    <rect x="232" y="160" width="154" height="14" rx="3" fill="var(--s7)"/>
    <text class="val" x="394" y="171">1,335</text>
    <text class="lab" x="224" y="196" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="206" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="186" width="126" height="14" rx="3" fill="var(--s1)"/>
    <text class="val" x="366" y="197">1,090</text>
  </svg>
  <figcaption>
    <strong>API-S</strong> · 2,048 in → 256 out. Short API calls. The shape where prefill dominates and the engine has least room to hide.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="Best output throughput per model on CHAT-S at 64 concurrent requests. MiMo-V2.6-Flash-RL 4,384, Qwen3.8-Flash-Next FP8 3,398, DeepSeek-V4.1-Flash 2,927, GLM-5.3-Flash 2,840, MiMo-V2.6-Pro-RL 2,679, Qwen3.8-Flash-Next bf16 2,403, GLM-5.3 1,521.">
    <line x1="364" y1="20" x2="364" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="364" y="16" text-anchor="middle">2,000</text>
    <line x1="496" y1="20" x2="496" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="496" y="16" text-anchor="middle">4,000</text>
    <text class="ax" x="232" y="237" text-anchor="start">CHAT-S · 2k in / 512 out · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--s6)"/>
    <text class="val" x="529" y="41">4,384</text>
    <text class="lab" x="224" y="66" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="56" width="224" height="14" rx="3" fill="var(--s5)"/>
    <text class="val" x="464" y="67">3,398</text>
    <text class="lab" x="224" y="92" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM latency</text>
    <rect x="232" y="82" width="193" height="14" rx="3" fill="var(--s3)"/>
    <text class="val" x="433" y="93">2,927</text>
    <text class="lab" x="224" y="118" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="128" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="108" width="187" height="14" rx="3" fill="var(--s2)"/>
    <text class="val" x="427" y="119">2,840</text>
    <text class="lab" x="224" y="144" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="154" text-anchor="end">vLLM latency + DFlash</text>
    <rect x="232" y="134" width="177" height="14" rx="3" fill="var(--s7)"/>
    <text class="val" x="417" y="145">2,679</text>
    <text class="lab" x="224" y="170" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="180" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="160" width="159" height="14" rx="3" fill="var(--s4)"/>
    <text class="val" x="399" y="171">2,403</text>
    <text class="lab" x="224" y="196" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="206" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="186" width="100" height="14" rx="3" fill="var(--s1)"/>
    <text class="val" x="340" y="197">1,521</text>
  </svg>
  <figcaption>
    <strong>CHAT-S</strong> · 2,048 in → 512 out. A short chat turn — the most common interactive shape.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="Best output throughput per model on CODE-I at 64 concurrent requests. Qwen3.8-Flash-Next FP8 3,102, MiMo-V2.6-Flash-RL 2,476, GLM-5.3-Flash 2,097, Qwen3.8-Flash-Next bf16 1,936, DeepSeek-V4.1-Flash 1,731, MiMo-V2.6-Pro-RL 1,648, GLM-5.3 930.">
    <line x1="325" y1="20" x2="325" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="325" y="16" text-anchor="middle">1,000</text>
    <line x1="419" y1="20" x2="419" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="419" y="16" text-anchor="middle">2,000</text>
    <line x1="512" y1="20" x2="512" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="512" y="16" text-anchor="middle">3,000</text>
    <text class="ax" x="232" y="237" text-anchor="start">CODE-I · 16k in / 2k out · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--s5)"/>
    <text class="val" x="529" y="41">3,102</text>
    <text class="lab" x="224" y="66" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="56" width="231" height="14" rx="3" fill="var(--s6)"/>
    <text class="val" x="471" y="67">2,476</text>
    <text class="lab" x="224" y="92" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="82" width="196" height="14" rx="3" fill="var(--s2)"/>
    <text class="val" x="436" y="93">2,097</text>
    <text class="lab" x="224" y="118" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="128" text-anchor="end">SGLang latency</text>
    <rect x="232" y="108" width="181" height="14" rx="3" fill="var(--s4)"/>
    <text class="val" x="421" y="119">1,936</text>
    <text class="lab" x="224" y="144" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="154" text-anchor="end">vLLM latency</text>
    <rect x="232" y="134" width="161" height="14" rx="3" fill="var(--s3)"/>
    <text class="val" x="401" y="145">1,731</text>
    <text class="lab" x="224" y="170" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="180" text-anchor="end">vLLM latency + DFlash</text>
    <rect x="232" y="160" width="154" height="14" rx="3" fill="var(--s7)"/>
    <text class="val" x="394" y="171">1,648</text>
    <text class="lab" x="224" y="196" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="206" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="186" width="87" height="14" rx="3" fill="var(--s1)"/>
    <text class="val" x="327" y="197">930</text>
  </svg>
  <figcaption>
    <strong>CODE-I</strong> · 16,384 in → 2,048 out. Code completion with a file of context in the prompt.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="Best output throughput per model on CHAT-L at 64 concurrent requests. MiMo-V2.6-Flash-RL 1,989, Qwen3.8-Flash-Next FP8 1,838, GLM-5.3-Flash 1,317, Qwen3.8-Flash-Next bf16 1,189, DeepSeek-V4.1-Flash 1,006, MiMo-V2.6-Pro-RL 987, GLM-5.3 468.">
    <line x1="377" y1="20" x2="377" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="377" y="16" text-anchor="middle">1,000</text>
    <line x1="523" y1="20" x2="523" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="523" y="16" text-anchor="middle">2,000</text>
    <text class="ax" x="232" y="237" text-anchor="start">CHAT-L · 32k in / 2k out · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--s6)"/>
    <text class="val" x="529" y="41">1,989</text>
    <text class="lab" x="224" y="66" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="56" width="267" height="14" rx="3" fill="var(--s5)"/>
    <text class="val" x="507" y="67">1,838</text>
    <text class="lab" x="224" y="92" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM throughput</text>
    <rect x="232" y="82" width="192" height="14" rx="3" fill="var(--s2)"/>
    <text class="val" x="432" y="93">1,317</text>
    <text class="lab" x="224" y="118" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="128" text-anchor="end">SGLang latency</text>
    <rect x="232" y="108" width="173" height="14" rx="3" fill="var(--s4)"/>
    <text class="val" x="413" y="119">1,189</text>
    <text class="lab" x="224" y="144" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="154" text-anchor="end">vLLM latency</text>
    <rect x="232" y="134" width="146" height="14" rx="3" fill="var(--s3)"/>
    <text class="val" x="386" y="145">1,006</text>
    <text class="lab" x="224" y="170" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="180" text-anchor="end">vLLM latency + DFlash</text>
    <rect x="232" y="160" width="144" height="14" rx="3" fill="var(--s7)"/>
    <text class="val" x="384" y="171">987</text>
    <text class="lab" x="224" y="196" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="206" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="186" width="68" height="14" rx="3" fill="var(--s1)"/>
    <text class="val" x="308" y="197">468</text>
  </svg>
  <figcaption>
    <strong>CHAT-L</strong> · 32,768 in → 2,048 out. A long conversation carrying its own history forward.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="Best output throughput per model on CODE-A at 64 concurrent requests. Qwen3.8-Flash-Next FP8 2,082, GLM-5.3-Flash 1,699, MiMo-V2.6-Flash-RL 1,587, Qwen3.8-Flash-Next bf16 1,155, DeepSeek-V4.1-Flash 992, MiMo-V2.6-Pro-RL 892, GLM-5.3 392.">
    <line x1="371" y1="20" x2="371" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="371" y="16" text-anchor="middle">1,000</text>
    <line x1="510" y1="20" x2="510" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="510" y="16" text-anchor="middle">2,000</text>
    <text class="ax" x="232" y="237" text-anchor="start">CODE-A · 64k in / 4k out · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--s5)"/>
    <text class="val" x="529" y="41">2,082</text>
    <text class="lab" x="224" y="66" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM throughput</text>
    <rect x="232" y="56" width="236" height="14" rx="3" fill="var(--s2)"/>
    <text class="val" x="476" y="67">1,699</text>
    <text class="lab" x="224" y="92" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="82" width="221" height="14" rx="3" fill="var(--s6)"/>
    <text class="val" x="461" y="93">1,587</text>
    <text class="lab" x="224" y="118" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="128" text-anchor="end">SGLang latency</text>
    <rect x="232" y="108" width="161" height="14" rx="3" fill="var(--s4)"/>
    <text class="val" x="401" y="119">1,155</text>
    <text class="lab" x="224" y="144" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="154" text-anchor="end">vLLM throughput</text>
    <rect x="232" y="134" width="138" height="14" rx="3" fill="var(--s3)"/>
    <text class="val" x="378" y="145">992</text>
    <text class="lab" x="224" y="170" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="180" text-anchor="end">vLLM latency + DFlash</text>
    <rect x="232" y="160" width="124" height="14" rx="3" fill="var(--s7)"/>
    <text class="val" x="364" y="171">892</text>
    <text class="lab" x="224" y="196" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="206" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="186" width="55" height="14" rx="3" fill="var(--s1)"/>
    <text class="val" x="295" y="197">392</text>
  </svg>
  <figcaption>
    <strong>CODE-A</strong> · 65,536 in → 4,096 out. Whole-repository analysis: a large prompt and a substantial answer.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 216" role="img" aria-label="Best output throughput per model on DOC-L at 64 concurrent requests. GLM-5.3-Flash 587, MiMo-V2.6-Flash-RL 482, Qwen3.8-Flash-Next FP8 378, Qwen3.8-Flash-Next bf16 359, MiMo-V2.6-Pro-RL 255, GLM-5.3 99.">
    <line x1="331" y1="20" x2="331" y2="179" stroke="var(--grid)"/>
    <text class="sub" x="331" y="16" text-anchor="middle">200</text>
    <line x1="429" y1="20" x2="429" y2="179" stroke="var(--grid)"/>
    <text class="sub" x="429" y="16" text-anchor="middle">400</text>
    <line x1="528" y1="20" x2="528" y2="179" stroke="var(--grid)"/>
    <text class="sub" x="528" y="16" text-anchor="middle">600</text>
    <text class="ax" x="232" y="211" text-anchor="start">DOC-L · 128k in / 2k out · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM throughput</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--s2)"/>
    <text class="val" x="529" y="41">587</text>
    <text class="lab" x="224" y="66" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="56" width="238" height="14" rx="3" fill="var(--s6)"/>
    <text class="val" x="478" y="67">482</text>
    <text class="lab" x="224" y="92" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="82" width="186" height="14" rx="3" fill="var(--s5)"/>
    <text class="val" x="426" y="93">378</text>
    <text class="lab" x="224" y="118" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="128" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="108" width="177" height="14" rx="3" fill="var(--s4)"/>
    <text class="val" x="417" y="119">359</text>
    <text class="lab" x="224" y="144" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="154" text-anchor="end">vLLM latency + DFlash</text>
    <rect x="232" y="134" width="126" height="14" rx="3" fill="var(--s7)"/>
    <text class="val" x="366" y="145">255</text>
    <text class="lab" x="224" y="170" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="180" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="160" width="49" height="14" rx="3" fill="var(--s1)"/>
    <text class="val" x="289" y="171">99</text>
  </svg>
  <figcaption>
    <strong>DOC-L</strong> · 131,072 in → 2,048 out. The 128k document shape. Prefill dominates completely, and the ranking here looks unlike every other panel.
  </figcaption>
</figure>

<figure class="qz">
  <svg viewBox="0 0 640 242" role="img" aria-label="Best output throughput per model on BATCH-D at 64 concurrent requests. Qwen3.8-Flash-Next FP8 6,475, DeepSeek-V4.1-Flash 5,700, GLM-5.3-Flash 5,433, MiMo-V2.6-Flash-RL 4,950, Qwen3.8-Flash-Next bf16 4,446, GLM-5.3 4,009, MiMo-V2.6-Pro-RL 3,198.">
    <line x1="321" y1="20" x2="321" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="321" y="16" text-anchor="middle">2,000</text>
    <line x1="411" y1="20" x2="411" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="411" y="16" text-anchor="middle">4,000</text>
    <line x1="500" y1="20" x2="500" y2="205" stroke="var(--grid)"/>
    <text class="sub" x="500" y="16" text-anchor="middle">6,000</text>
    <text class="ax" x="232" y="237" text-anchor="start">BATCH-D · 4k in / 8k out · 64 concurrent · output tok/s</text>
    <text class="lab" x="224" y="40" text-anchor="end">Qwen3.8-Flash-Next FP8</text>
    <text class="sub" x="224" y="50" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="30" width="289" height="14" rx="3" fill="var(--s5)"/>
    <text class="val" x="529" y="41">6,475</text>
    <text class="lab" x="224" y="66" text-anchor="end">DeepSeek-V4.1-Flash</text>
    <text class="sub" x="224" y="76" text-anchor="end">vLLM throughput</text>
    <rect x="232" y="56" width="255" height="14" rx="3" fill="var(--s3)"/>
    <text class="val" x="495" y="67">5,700</text>
    <text class="lab" x="224" y="92" text-anchor="end">GLM-5.3-Flash</text>
    <text class="sub" x="224" y="102" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="82" width="243" height="14" rx="3" fill="var(--s2)"/>
    <text class="val" x="483" y="93">5,433</text>
    <text class="lab" x="224" y="118" text-anchor="end">MiMo-V2.6-Flash-RL</text>
    <text class="sub" x="224" y="128" text-anchor="end">vLLM balanced</text>
    <rect x="232" y="108" width="221" height="14" rx="3" fill="var(--s6)"/>
    <text class="val" x="461" y="119">4,950</text>
    <text class="lab" x="224" y="144" text-anchor="end">Qwen3.8-Flash-Next bf16</text>
    <text class="sub" x="224" y="154" text-anchor="end">SGLang latency</text>
    <rect x="232" y="134" width="199" height="14" rx="3" fill="var(--s4)"/>
    <text class="val" x="439" y="145">4,446</text>
    <text class="lab" x="224" y="170" text-anchor="end">GLM-5.3</text>
    <text class="sub" x="224" y="180" text-anchor="end">SGLang throughput</text>
    <rect x="232" y="160" width="179" height="14" rx="3" fill="var(--s1)"/>
    <text class="val" x="419" y="171">4,009</text>
    <text class="lab" x="224" y="196" text-anchor="end">MiMo-V2.6-Pro-RL</text>
    <text class="sub" x="224" y="206" text-anchor="end">vLLM latency + DFlash</text>
    <rect x="232" y="186" width="143" height="14" rx="3" fill="var(--s7)"/>
    <text class="val" x="383" y="197">3,198</text>
  </svg>
  <figcaption>
    <strong>BATCH-D</strong> · 4,096 in → 8,192 out. Decode-heavy batch generation — the shape that rewards raw token production.
  </figcaption>
</figure>

Two things are worth pulling out. **DeepSeek-V4.1-Flash is missing from the
DOC-L panel** — not because it was slow, but because no request finished inside
the window on any of its configurations, so there is no rate to plot. And
MiMo-V2.6-Pro-RL, the 1T model, places mid-table or last on every shape except
the single-stream chart further up: parameter count buys quality, not tokens
per second.

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
  <svg viewBox="0 0 640 366" role="img" aria-label="Output throughput against concurrency on BATCH-D, best configuration for each of 7 models. GLM-5.3: 137 at c1, 1,068 at c16, 4,009 at c64; GLM-5.3-Flash: 205 at c1, 1,848 at c16, 5,433 at c64; DeepSeek-V4.1-Flash: 205 at c1, 2,407 at c16, 5,700 at c64; Qwen3.8-Flash-Next bf16: 307 at c1, 2,206 at c16, 4,446 at c64; Qwen3.8-Flash-Next FP8: 171 at c1, 2,160 at c16, 6,475 at c64; MiMo-V2.6-Flash-RL: 205 at c1, 2,163 at c16, 4,950 at c64; MiMo-V2.6-Pro-RL: 341 at c1, 1,510 at c16, 3,198 at c64.">
    <rect x="58" y="8" width="13" height="3" rx="1.5" fill="var(--s1)"/>
    <text class="sub" x="76" y="12.5">GLM-5.3</text>
    <rect x="121" y="8" width="13" height="3" rx="1.5" fill="var(--s2)"/>
    <text class="sub" x="139" y="12.5">GLM-5.3-Flash</text>
    <rect x="219" y="8" width="13" height="3" rx="1.5" fill="var(--s3)"/>
    <text class="sub" x="237" y="12.5">DeepSeek-V4.1-Flash</text>
    <rect x="353" y="8" width="13" height="3" rx="1.5" fill="var(--s4)"/>
    <text class="sub" x="371" y="12.5">Qwen3.8-Flash-Next bf16</text>
    <rect x="58" y="24" width="13" height="3" rx="1.5" fill="var(--s5)"/>
    <text class="sub" x="76" y="28.5">Qwen3.8-Flash-Next FP8</text>
    <rect x="209" y="24" width="13" height="3" rx="1.5" fill="var(--s6)"/>
    <text class="sub" x="227" y="28.5">MiMo-V2.6-Flash-RL</text>
    <rect x="337" y="24" width="13" height="3" rx="1.5" fill="var(--s7)"/>
    <text class="sub" x="355" y="28.5">MiMo-V2.6-Pro-RL</text>
    <line x1="58" y1="285" x2="562" y2="285" stroke="var(--grid)"/>
    <text class="sub" x="49" y="289" text-anchor="end">1k</text>
    <line x1="58" y1="250" x2="562" y2="250" stroke="var(--grid)"/>
    <text class="sub" x="49" y="254" text-anchor="end">2k</text>
    <line x1="58" y1="216" x2="562" y2="216" stroke="var(--grid)"/>
    <text class="sub" x="49" y="219" text-anchor="end">3k</text>
    <line x1="58" y1="181" x2="562" y2="181" stroke="var(--grid)"/>
    <text class="sub" x="49" y="184" text-anchor="end">4k</text>
    <line x1="58" y1="146" x2="562" y2="146" stroke="var(--grid)"/>
    <text class="sub" x="49" y="149" text-anchor="end">5k</text>
    <line x1="58" y1="111" x2="562" y2="111" stroke="var(--grid)"/>
    <text class="sub" x="49" y="115" text-anchor="end">6k</text>
    <line x1="58" y1="76" x2="562" y2="76" stroke="var(--grid)"/>
    <text class="sub" x="49" y="80" text-anchor="end">7k</text>
    <text class="sub" x="58" y="338" text-anchor="middle">1</text>
    <text class="sub" x="310" y="338" text-anchor="middle">16</text>
    <text class="sub" x="562" y="338" text-anchor="middle">64</text>
    <text class="ax" x="310" y="354" text-anchor="middle">concurrent requests</text>
    <polyline points="58,315 310,283 562,180" fill="none" stroke="var(--s1)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="315" r="3.5" fill="var(--s1)"/>
    <circle cx="310" cy="283" r="3.5" fill="var(--s1)"/>
    <circle cx="562" cy="180" r="3.5" fill="var(--s1)"/>
    <polyline points="58,313 310,256 562,131" fill="none" stroke="var(--s2)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="313" r="3.5" fill="var(--s2)"/>
    <circle cx="310" cy="256" r="3.5" fill="var(--s2)"/>
    <circle cx="562" cy="131" r="3.5" fill="var(--s2)"/>
    <polyline points="58,313 310,236 562,122" fill="none" stroke="var(--s3)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="313" r="3.5" fill="var(--s3)"/>
    <circle cx="310" cy="236" r="3.5" fill="var(--s3)"/>
    <circle cx="562" cy="122" r="3.5" fill="var(--s3)"/>
    <polyline points="58,309 310,243 562,165" fill="none" stroke="var(--s4)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="309" r="3.5" fill="var(--s4)"/>
    <circle cx="310" cy="243" r="3.5" fill="var(--s4)"/>
    <circle cx="562" cy="165" r="3.5" fill="var(--s4)"/>
    <polyline points="58,314 310,245 562,95" fill="none" stroke="var(--s5)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="314" r="3.5" fill="var(--s5)"/>
    <circle cx="310" cy="245" r="3.5" fill="var(--s5)"/>
    <circle cx="562" cy="95" r="3.5" fill="var(--s5)"/>
    <polyline points="58,313 310,245 562,148" fill="none" stroke="var(--s6)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="313" r="3.5" fill="var(--s6)"/>
    <circle cx="310" cy="245" r="3.5" fill="var(--s6)"/>
    <circle cx="562" cy="148" r="3.5" fill="var(--s6)"/>
    <polyline points="58,308 310,267 562,209" fill="none" stroke="var(--s7)" stroke-width="2" stroke-linejoin="round" stroke-linecap="round"/>
    <circle cx="58" cy="308" r="3.5" fill="var(--s7)"/>
    <circle cx="310" cy="267" r="3.5" fill="var(--s7)"/>
    <circle cx="562" cy="209" r="3.5" fill="var(--s7)"/>
    <text class="val" x="570" y="98" fill="var(--s5)">6,475</text>
    <text class="val" x="570" y="125" fill="var(--s3)">5,700</text>
    <text class="val" x="570" y="138" fill="var(--s2)">5,433</text>
    <text class="val" x="570" y="151" fill="var(--s6)">4,950</text>
    <text class="val" x="570" y="169" fill="var(--s4)">4,446</text>
    <text class="val" x="570" y="184" fill="var(--s1)">4,009</text>
    <text class="val" x="570" y="212" fill="var(--s7)">3,198</text>
  </svg>
  <figcaption>
    BATCH-D (4,096 in → 8,192 out), each model shown with its best configuration
    at 64 concurrent requests. Every model gains an order of magnitude from 1 to
    64 streams; they do not gain it at the same rate, and the ordering at c1 is
    not the ordering at c64.
  </figcaption>
</figure>

Every model shows the same shape, and none of them is close to saturated at 16
streams. Qwen3.8-Flash-Next FP8 starts second-slowest of the seven at one stream
and finishes first at 64; MiMo-V2.6-Pro-RL does the opposite, leading at c1 and
ending last. Ranking a model on single-stream numbers tells you very little
about how it will serve a loaded endpoint.

Within a model, the low-latency and throughput recipes cross somewhere — and
where they cross is **not** portable either:

| model | crossover |
|---|---|
| DeepSeek-V4.1-Flash | below 16 concurrent requests |
| GLM-5.3 | between 16 and 64 |
| GLM-5.3-Flash | between 16 and 64 |
| Qwen3.8-Flash-Next (bf16 and FP8) | above 64 |
| MiMo-V2.6-Flash-RL | between 1 and 16 — but only against the balanced recipe |
| MiMo-V2.6-Pro-RL | never — the latency recipe leads at every concurrency measured |

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
