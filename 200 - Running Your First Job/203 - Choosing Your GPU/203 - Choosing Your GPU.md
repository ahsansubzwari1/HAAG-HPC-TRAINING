# Choosing Your GPU

The fastest GPU is not automatically the best GPU for your job. On a shared cluster, the best first choice is usually the smallest compatible GPU configuration that fits your measured VRAM requirement with safe headroom.

This page uses three questions to turn a VRAM estimate into a practical ICE GPU request.

---

## Before choosing a GPU

You should already know:

- Whether your code actually uses a GPU
- Peak GPU VRAM from a small representative run or a recipe estimate
- Whether the program supports more than one GPU
- Whether your framework requires NVIDIA CUDA or supports AMD ROCm

If your program is CPU-only, stop here and omit the `--gres=gpu...` line. A GPU reservation does not automatically convert CPU code into GPU code.

---

## The three-question funnel

### Question 1: Does the job fit on one 16 GB V100?

If the peak requirement plus headroom is at or below 16 GB and the software supports the V100, request one V100. This is a common, relatively plentiful tier on ICE.

```bash
#SBATCH --gres=gpu:V100:1
```

If you must distinguish the 32 GB version, use the exact GPU name accepted by ICE rather than assuming the generic request will select it.

### Question 2: If not, what is the smallest tier that fits?

Work upward from the smallest tier. Current ICE hardware is grouped approximately like this:

| Required VRAM | Candidate tier | Check before choosing |
|---:|---|---|
| Up to 16 GB | V100 16 GB | Framework and model compatibility |
| Up to 24 GB | Quadro RTX 6000 | Older hardware may run more slowly |
| Up to 32 GB | V100 32 GB | Request the specific 32 GB type when needed |
| Up to 40 GB | A100 40 GB | Verify the exact `--gres` name |
| Up to 48 GB | A40, L40S, or RTX 6000 Pro Blackwell | Compare availability and multi-GPU layout |
| Up to 64 GB | MI210 | AMD ROCm compatibility is required |
| Up to 80 GB | A100 80 GB or H100 80 GB | Prefer the smallest practical, available option |
| Up to 142 GB | H200 142 GB | Scarce capacity; use when the memory is genuinely required |

Hardware inventory and accepted names can change. Check the current ICE resources before finalizing a long-running job.

### Question 3: Do you truly need multiple GPUs?

Request multiple GPUs only when at least one of these is true:

- A single GPU cannot hold the model or workload.
- The application is configured for tensor, pipeline, or data parallelism.
- A measured scaling test shows that additional GPUs reduce elapsed time enough to justify them.

Multiple GPUs do not behave like one automatically pooled memory device. Your framework and code must distribute the work. A script that only selects `cuda:0` will not use the other GPUs.

---

## Decision tree

```text
Does the program use GPU-enabled code?
├── No → Submit a CPU-only job; do not request --gres=gpu.
└── Yes
    ├── Does peak VRAM + headroom fit on one 16 GB V100?
    │   └── Yes → Request one V100 and profile the real run.
    └── No
        ├── Does the workload fit on one larger ICE GPU?
        │   └── Yes → Choose the smallest compatible tier that fits.
        └── No
            ├── Can the application distribute work across multiple GPUs?
            │   └── Yes → Test 2 GPUs first; scale only with evidence.
            └── No → Reduce memory use, change the method, or escalate.
```

---

## Why not request an H100 or H200 every time?

H100 and H200 capacity available to general ICE users is more limited than the total inventory suggests because many nodes are reserved. A job that also fits on an A100 may start sooner there.

Use an H-class GPU when you need its memory capacity, bandwidth, supported numeric formats, or demonstrated performance. Do not choose one simply because it is newer.

---

## Turning the decision into Slurm

The GPU request has this general form:

```bash
#SBATCH --gres=gpu:<GPU-TYPE>:<COUNT>
```

Example request forms include:

```bash
#SBATCH --gres=gpu:V100:1
#SBATCH --gres=gpu:V100-32GB:1
#SBATCH --gres=gpu:A100:2
#SBATCH --gres=gpu:H200:4
```

Do not copy an example until you confirm the name and count appropriate for your job. On ICE, you can inspect the GPU resources advertised to Slurm with:

```bash
sinfo -o "%P %G"
```

ICE partitions are routed according to access, so do not hard-code a partition merely to reach a particular GPU. Request the GPU type with `--gres` and follow current PACE documentation for account-specific requirements.

---

## Worked example

A small profiling run peaks at 18 GB of GPU VRAM.

1. Add about 20% headroom: `18 GB × 1.2 = 21.6 GB`.
2. It does not fit safely on a 16 GB V100.
3. A 24 GB GPU is mathematically large enough, but leaves only about 10% headroom relative to the 21.6 GB plan.
4. A 32 GB V100 provides a more comfortable first request.
5. Request one GPU because the application has not demonstrated multi-GPU support.

```bash
#SBATCH --gres=gpu:V100-32GB:1
```

After the real run, compare measured peak VRAM and utilization with the reservation. Revise the next request rather than keeping the first estimate forever.

---

## Compatibility checks

VRAM capacity is necessary, but it is not the only requirement.

- **CUDA versus ROCm:** NVIDIA-oriented code may not run unchanged on the AMD MI210.
- **Numeric formats:** Some models depend on hardware support for BF16, FP8, or particular kernels.
- **Multi-GPU transport:** NVLink and PCIe configurations have different communication performance.
- **Software environment:** Confirm that the PyTorch, TensorFlow, CUDA, ROCm, or serving-framework version supports the selected hardware.
- **GPU count per node:** Do not request more GPUs than a suitable node can provide.

---

## Next step

For LLM **inference serving**, continue to **Using the VRAM Calculator**. For training or fine-tuning, use the relevant sizing recipe instead.

For current hardware names and availability, consult [PACE ICE Resources](https://gatech.service-now.com/home?id=kb_article_view&sysparm_article=KB0042095).
