---
title: 'doctor-bench: grading LLM fixes by whether the build passes'
description: 'A benchmark of 4,983 broken setups, from C++ builds to Kubernetes, where a model fix only counts if the real build and tests pass afterwards.'
publishDate: 2026-10-16
tags:
  - ai
  - llm
  - benchmarking
  - hpc
  - devops
---

A fix that reads well is not the same as a fix that works. Most ways of
grading a model's debugging answer either compare it with a reference solution
or ask another model whether it looks right. Both reward a plausible
explanation and a confident patch, even when the patch wouldn't apply and the
build would still fail.

If you've ever been on call, you know the only question that matters: does it
work now? So that's the only question doctor-bench asks. The model's edits are
applied to a real broken setup, the setup's own build and tests run, and if
they pass the problem is solved. No LLM judge, no string matching.

I built it to measure small expert models: 4B-parameter LoRA adapters, each
trained to troubleshoot one domain. I wanted to put them next to frontier
models on exactly the same problems. The first results are in
[a companion post](/blog/frontier-models-on-doctor-bench).

## What a problem looks like

Every problem starts as a working setup with its own build and a
`tests/run.sh`. It might be a CMake library, an nginx site, an MPI program and
its Slurm job script, a vLLM deployment of a tiny model, a GROMACS simulation,
a Slurm cluster configuration or a set of Kubernetes manifests.

Then exactly one thing gets broken. A config key with a typo. A file with the
wrong permissions. Libraries linked in the wrong order. A timestep too large
for the physics. A QOS limit that silently starves a partition.

The model sees what a user would show a colleague: the failing log, the files
the log points at and the project's key files. It answers once, in JSON with a
fixed schema: a diagnosis, an explanation, and the exact edits (file, old
text, new text) or permission changes. Those edits get applied, and the build
and tests run in a Docker container with no network. Pass or fail.

Here's a made-up example in the spirit of the suite (not one of its problems).
A GROMACS input sets `dt = 0.02`, a 20 fs timestep that makes an unconstrained
all-atom simulation blow up. A good answer says so and returns one edit,
`dt = 0.02` to `dt = 0.002`. The test runs the simulation and checks that it
stays stable. An answer that explains the physics beautifully but quotes the
line wrong can't be applied, so it fails. I think that's correct. A fix you
can't apply isn't a fix.

## Seven categories

There are 14 canonical benchmarks across seven categories:

| category | what breaks | benchmarks | items |
|---|---|---|---:|
| build | C/C++ builds of open-source libraries (fmt, zstd, cJSON, Eigen, spdlog, libgit2…) with CMake, Make and the linker | transfer + in-distribution | 1,456 |
| linux | a web service and an nginx site: configuration, permissions and ownership, secrets | in-distribution | 311 |
| hpc | MPI and OpenMP programs, Lmod stacks, Slurm job scripts, including 80 real scripts from the HPC-Instruct dataset | transfer + in-distribution | 929 |
| ml-systems | tiny models deployed on vLLM, SGLang, Megatron, PyTorch, transformers and Evo2 | transfer + in-distribution | 530 |
| gromacs | simulation inputs: .mdp parameters, topologies, index groups, pulling, FEP, REMD | transfer + in-distribution | 491 |
| slurm-admin | slurm.conf, slurmdbd, QOS and associations, fair share, Lua job filters, GPU nodes | 2 × transfer + in-distribution | 722 |
| kubernetes | manifests, Kustomize, Helm, CRDs and operators, admission policies, RBAC, NetworkPolicy | transfer + in-distribution | 544 |

Every problem is tagged easy, medium or hard. Easy means the tool tells you
the cause. Medium means the error names the kind of problem but not where it
is. Hard is the nasty kind: everything builds and runs, the result is wrong,
and only the test's checks notice.

Some ml-systems problems need a GPU to validate, because they start a real
vLLM or SGLang server or run a Megatron step. When the model you're scoring
already occupies every GPU on the node, `--cpu-only` scores the subset that
doesn't need one.

## Where the problems come from

Writing thousands of debugging problems by hand doesn't scale. Mining public
bug trackers invites contamination, since those trackers are in everybody's
training data. So I went the other way: start from setups that work and break
them on purpose.

There are about 650 small working projects across the seven domains. Each has
a build and a test that checks real behaviour, not just that something runs.
Hundreds of fault operators then inject one fault each. A few are generic
(permissions, a missing file). Most are specific to their domain, like a
GROMACS cutoff longer than half the box, or a Slurm partition pointing at a
QOS that doesn't exist.

A large teacher model then attempts every broken setup, and a problem is kept
only if a teacher fixed it and the fix passed the real test. That filter
throws out faults that are ambiguous, or impossible to repair from what the
model is shown. It also has a side effect, which I come back to in the results
post: the teacher has, by construction, already solved its part of the
benchmark.

Finally the problems are split into training banks for the expert models and
benchmarks, and the benchmarks are frozen.

## Keeping it honest

Each benchmark is a frozen file with a fingerprint, stored byte for byte. The
harness refuses to score if that file, or any of the 650 seed projects,
differs from its recorded hash.

Each problem stores its broken tree as an overlay on top of its seed: changed
files with their modes, new directories, deleted paths, symlinks. I checked
every overlay against the tree the fault operator actually produced. That
mattered more than I expected. Git doesn't keep empty directories or most file
modes, and a fault built around a mode-000 file is useless if the file comes
back readable.

Benchmarks come in two kinds. *Transfer* benchmarks use setups no expert model
was trained on. *In-distribution* benchmarks put new faults into training
setups, which favours the experts by design. When I compare an expert with a
frontier model, the transfer numbers are the fair ones.

The repository is private for now. Once problems leak into training sets, a
benchmark measures memory.

And it reproduces. Re-scoring Qwen3-4B from a fresh clone matched the
original harness within vLLM's own run-to-run noise. Two runs of the old
harness disagree on 13 of 170 problems; the old harness and doctor-bench
disagree on 14.

## Running it

doctor-bench talks to anything with an OpenAI-compatible API: vLLM, SGLang or
a hosted endpoint.

```bash
uv venv -p 3.12 .venv && uv pip install -p .venv/bin/python -e .
.venv/bin/bench setup        # third-party sources, file modes, seed check
.venv/bin/bench run --model my-model --base-url http://localhost:8000/v1 --suites canonical
.venv/bin/bench report
```

One pass is one greedy attempt per problem: temperature 0, at most 4,096
output tokens, JSON output enforced by the server. You need Docker with the
domain images and plenty of CPU cores, because validation runs real builds and
real clusters, 16 at a time by default.

## Next

I want a gromacs-workflow track, for recovering failed steps of multi-stage
GROMACS workflows. It exists in another project against a mock `gmx` and needs
porting to the real GROMACS image. A long-context track of download recipes
for third-party datasets is also on the list, though those datasets are partly
LLM-judged, so they'll stay out of the executable score. And a lite tier of
about 50 problems per benchmark, because a full run is a lot of GPU time when
all you want is a quick comparison.

The first full run, seven open frontier models against a 4B model and its
experts, is in [Seven open frontier models on doctor-bench](/blog/frontier-models-on-doctor-bench).
