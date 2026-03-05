# Ryzen AI Software — Upgrade Strategy

**System:** Fedora 43 / AMD Ryzen AI MAX+ 395 (Strix Halo) / 96 GB LPDDR5X
**Hardware:** 16C/32T CPU, Radeon 8060S iGPU (gfx1151), NPU (AIE2P, amdxdna)
**Goal:** Leverage all three compute targets (CPU, GPU, NPU) for AI inference on Linux.
**Updated:** 2026-03-05 (post kernel upgrade + NPU verification)

---

## Strategic Priority: Kernel First, OS Second, Kubernetes Third

The foundation is not broken. Fedora 43, kubeadm, containerd, ROCm-in-containers, Vulkan ASR, VAAPI Plex, ArkCase/Qdrant/MinIO — this is a working stack. ~~The loudest problem was kernel 6.18.3 with its SVA regression~~ — **resolved**: kernel upgraded to **6.18.13-200.fc43**, NPU detected and operational (RyzenAI-npu5, FW 1.0.0.166, dmesg clean). AMD's official Ryzen ROCm matrix for Linux is more conservative and officially lists **Ubuntu 24.04.3** for ROCm 7.2 on gfx1151 / Ryzen AI MAX+ 395.

The upgrade path follows this order:

1. ~~**Upgrade the kernel.**~~ **DONE** — 6.18.13-200.fc43 installed, NPU operational. No need to escalate to 6.19.2+.
2. **Validate NPU natively on the host, outside Kubernetes.** Do not containerize NPU experiments until native validation is boring.
3. **Clean up Kubernetes around workload classes and resource policy.** Add quotas, priorities, and better resource modeling.
4. **Only add another OS if you want a vendor-reference lane**, not as your first rescue move.

---

## Current State

| Component | Status | Details |
|---|---|---|
| **CPU** | Working | ArkCase (core, LDAP, Solr, PostgreSQL, ActiveMQ), Qdrant, Music API, Speech Gateway, MinIO |
| **GPU** (ROCm) | Working in K8s | Ollama, ComfyUI, JupyterHub, ACE-Step, BS-RoFormer (5 services sharing one GPU) |
| **GPU** (Vulkan) | Working in K8s | whisper.cpp streaming ASR in music-intelligence |
| **GPU** (VAAPI) | Working in K8s | Plex video transcoding (H.264/HEVC/VP9); Mesa 25.2.7 driver confirmed |
| **NPU** | **Working** | Kernel 6.18.13 fixed SVA regression; `xrt-smi examine` detects RyzenAI-npu5, FW 1.0.0.166, no dmesg errors |
| **XRT** | Working | v2.19.0 — `xrt-smi` requires root (passwordless sudo rule installed), detects RyzenAI-npu5 |
| **CVML Libs** | Available | 67 shared libs + 5 xclbin overlays in `ryzen14/`; VitisAI EP present for C++ |
| **Build Tools** | Ready | GCC 15.2.0, cmake 4.2.3, make 4.4.1 (all via conda-forge) |
| **K8s Cluster** | Healthy | v1.35.0, containerd 2.1.6, all namespaces running, 33 pods across 8 namespaces |

---

## OS Decision: Should You Switch?

**Short answer: No, not first.** Stay on Fedora. Add Ubuntu only as a reference lane if you want a more officially supported Linux test path.

### Decision Matrix

| Goal | Best OS | Rationale |
|---|---|---|
| **Keep current Linux workstation/server stack working** | Fedora 43 (stay) | AMD's Strix Halo optimization doc is unusually friendly to Fedora — it's called out as having the required kernel fixes via native packaging |
| **Have an AMD-blessed Linux validation path** | Ubuntu 24.04.3 (second install) | ROCm 7.2 officially supports Ubuntu 24.04.3 on gfx1151; PyTorch 2.9 + Python 3.12 is the official production lane |
| **Whisper NPU acceleration right now** | Windows | AMD documents Whisper NPU as Windows-only today, with Linux support "planned" |

### Why Fedora Is Not the Villain

AMD's [Strix Halo System Optimization Guide](https://rocm.docs.amd.com/en/latest/how-to/system-optimization/strixhalo.html) is unusually positive about Fedora 43:

- Fedora 43 is explicitly called out as including the required kernel fixes for Strix Halo via native packaging
- AMD recommends treating Strix Halo as a **unified-memory** system where GTT-backed shared memory is the real pool to manage, not "fake discrete VRAM"
- They recommend keeping dedicated BIOS VRAM reservation small and using shared memory/TTM tuning instead

At the same time, AMD's Linux support story is stitched together. The docs show Linux flows for OGA/LLM and CVML, and even Linux examples for `xrt-smi`, but the main installation page for Ryzen AI 1.7 is explicitly Windows-only. A distro switch to Ubuntu would get you closer to AMD's officially supported Linux ROCm path, but it **would not magically unlock Whisper-on-NPU on Linux today**.

### If You Add Ubuntu

Add it on a second SSD/partition for testing — don't migrate everything. ROCm 7.2 officially supports Ubuntu 24.04.3 on the GPU/APU family, and PyTorch 2.9 + Python 3.12 is the official production lane there. Use it as a reference environment to test AMD's documented Linux paths, not as your primary server OS.

---

## ~~Problem 1: Kernel 6.18.3 Breaks the NPU Driver~~ RESOLVED

**Impact:** ~~NPU is non-functional — highest priority blocker.~~ **RESOLVED** (2026-03-05) — Kernel upgraded to 6.18.13-200.fc43. NPU fully operational.

Kernels 6.18.0 through 6.18.7 have a documented IOMMU SVA regression that completely breaks the `amdxdna` NPU driver. The driver loads and `/dev/accel/accel0` exists with correct permissions (666), but workloads fail. Kernel dmesg confirms:

```
amdxdna 0000:c8:00.1: [drm] *ERROR* aie2_get_info: Not supported request parameter 4
```

NPU firmware is present (`17f0_11/npu.sbin.1.0.0.166.xz`) and XRT 2.19.0 is installed, so the rest of the stack is ready — only the kernel regression blocks execution.

**Fix:** Upgrade to kernel **6.19.2+**, which resolves both the SVA regression and a separate firmware protocol incompatibility introduced in 6.18.8.

### Kernel Availability (as of 2026-03-04)

Fedora 43 repos do **not** ship kernel 6.19.x. The available versions are:

| Repo | Version | NPU Status |
|---|---|---|
| `fedora` | 6.17.1-300.fc43 | Pre-regression (NPU may work, but older driver support) |
| `updates` | **6.18.13-200.fc43** | SVA regression fixed (6.18.8+), but firmware protocol issue may persist |
| *needed* | **6.19.2+** | Both issues resolved — clean target |

This means the original "5 min + reboot" estimate was wrong. The upgrade is a two-step decision:

**Step A — Try 6.18.13 first (low risk, available now):**

```bash
sudo dnf upgrade kernel
# Installs 6.18.13-200.fc43 — reboot and test xrt-smi examine
```

Kernel 6.18.13 fixes the SVA regression (6.18.0-6.18.7) but may still have the firmware protocol incompatibility introduced in 6.18.8. Fedora often backports fixes, so 6.18.13 *might* include the fix — **test before assuming it doesn't work.** If `xrt-smi examine` detects the NPU and dmesg is clean, you're done.

**Step B — If 6.18.13 doesn't fix NPU, escalate to 6.19.2+:**

| Option | Effort | Risk |
|---|---|---|
| **Wait for Fedora 44** | Zero effort, unknown timeline | NPU stays blocked until release |
| **Koji / Bodhi build** | `koji download-build kernel-6.19.x` + manual install | Medium — unsupported kernel, no auto-updates |
| **Fedora Rawhide kernel** | `sudo dnf --enablerepo=rawhide upgrade kernel` | Higher — pulls bleeding-edge deps |
| **Build from source** | `make -j32` with Fedora config + amdxdna patches | 30+ min, requires ongoing maintenance |

> **Recommendation:** Try Step A first. If 6.18.13 works for the NPU, the problem is solved with zero complexity. Only escalate to Step B if `xrt-smi examine` still fails after the 6.18.13 upgrade.

> **Note:** Kernel 6.18.4+ also introduced `amdgpu.cwsr_enable=0` requirements for ROCm stability on gfx1151. Your Kubernetes Ollama deployment may need this kernel parameter if you see MES firmware hangs after a kernel upgrade.

---

## Problem 2: No ONNX Runtime with VitisAI EP for Linux

**Impact:** Cannot run CNN/Transformer models on the NPU via Python.

There are no Linux wheels for `onnxruntime-vitisai` on PyPI. Building from source with `--use_vitisai` fails on GCC 13+ due to template errors ([onnxruntime#27097](https://github.com/microsoft/onnxruntime/issues/27097), marked stale with no fix). The system has GCC 15.2.0 (conda-forge), which is also affected.

No Python AI packages are installed in the active conda env: `onnxruntime`, `torch`, `numpy`, `onnx` — all absent. The base env (Python 3.13) is too new for the Ryzen AI stack.

Two ORT C++ library sets are available:

| Location | ORT Version | VitisAI EP |
|---|---|---|
| `~/onnxruntime/onnxruntime-linux-x64-1.20.1/lib` | 1.20.1 (vanilla) | No |
| `~/RyzenAI-SW/Ryzen-AI-CVML-Library/linux/onnx/ryzen14` | 1.x (CVML) | Yes |

The CVML set includes `libonnxruntime_vitisai_ep.so`, `libvaip-core.so`, and `libxcompiler-core-without-symbol.so` — a complete C++ inference stack minus the `vaiml_partition` pass (see Problem 3) and Python bindings.

### Options

| Option | Pros | Cons |
|---|---|---|
| **A. Ryzen AI SDK 1.7 (now GA)** | Bundles a working Linux ORT+VitisAI EP | Ubuntu 24.04 only; Fedora unsupported |
| **B. CVML C++ libs (already have)** | No download needed; C++ API works today | No Python; missing `vaiml_partition` pass (see Problem 3) |
| **C. Wait for public Linux wheels** | Zero friction when available | No timeline from AMD |

**Recommended path:** Download Ryzen AI SDK 1.7 (now GA, option A) while using CVML C++ libs (option B) for what works today.

---

## Problem 3: whisper.cpp NPU Offload — Missing VAIP Pass

**Impact:** whisper.cpp VitisAI ORT backend compiles and runs but all ops fall back to CPU.

The `WHISPER_VITISAI_ORT` backend we built works end-to-end: VitisAI EP loads, the model is parsed (413 ops), and transcription is correct via CPU fallback. The blocker is a single missing VAIP plugin: `libvaip-pass_vaiml_partition.so`.

This pass is responsible for partitioning BF16 Whisper encoder ops onto the NPU via the VAIML toolchain. The CVML Linux libs ship the compiler core (`libxcompiler-core-without-symbol.so`, 118 MB) and the VAIP framework (`libvaip-core.so`) but not the partition pass that invokes them.

**Status:** AMD's Ryzen AI 1.7 docs list Whisper NPU on Linux as "planned." AMD's whisper.cpp NPU support page explicitly states: **Whisper NPU acceleration is currently supported on Windows only, with Linux support planned.** The pass has not been shipped standalone for Linux.

### Options

| Option | Pros | Cons |
|---|---|---|
| **A. Ryzen AI SDK 1.7 (now GA)** | May include the pass as part of the bundled runtime | Unconfirmed; check GA release contents |
| **B. Request from AMD directly** | Targeted ask; backend code is ready | Depends on AMD's roadmap |
| **C. CPU fallback (current)** | Works now; ~158ms encode on Ryzen AI MAX+ | Not using NPU hardware |

**Backend code is complete** — see `whisper.cpp/src/vitisai/`. Zero code changes needed once the pass is available.

> **Cross-project note:** `~/music-intelligence/` runs Whisper large-v3-turbo on the Vulkan GPU backend today. Once NPU offload works, migrating the music-intelligence ASR service to VitisAI ORT would free GPU memory for the generative workloads (ACE-Step, ComfyUI) that are GPU-only. The music-intelligence project already includes an NPU probe (`deploy/kustomize/base/npu-probe/`) that detects `/dev/accel0`.

---

## Problem 4: ROCm — Not Officially Supported on gfx1151

**Impact:** Low — GPU inference works in practice but is unsupported.

gfx1151 (Strix Halo) is **not** in AMD's official ROCm 7.2 compatibility matrix. It works via the `gfx11-generic` ISA target and community workarounds (`HSA_OVERRIDE_GFX_VERSION=11.0.0`, `OLLAMA_FLASH_ATTENTION=1`, `OLLAMA_NO_MMAP=1`).

**You already handle this.** Your `setup-linux.sh` deploys Ollama with the correct Strix Halo env vars in Kubernetes. Fedora 43 ships ROCm 6.4.3 natively; the community [recommends Fedora as the best platform for gfx1151](https://community.frame.work/t/linux-rocm-january-2026-stable-configurations-update/79876).

### If You Need ROCm Outside Kubernetes

```bash
# Fedora native ROCm 6.4.3
sudo dnf install rocm
rocminfo  # verify GPU detection

# PyTorch with gfx1151 nightly wheels (no system ROCm needed)
pip install --pre torch torchvision torchaudio \
  --index-url https://rocm.nightlies.amd.com/v2/gfx1151/
```

> **Kernel parameter for stability:** If MES firmware hangs occur, add `amdgpu.cwsr_enable=0` to your kernel cmdline.

---

## Problem 5: Ryzen AI SDK Not Configured

**Impact:** `RYZEN_AI_INSTALLATION_PATH` and `XLNX_VART_FIRMWARE` are unset. This blocks any workflow that depends on the SDK's config files, xclbin overlays, or runtime libs.

**Blocked by:** Problem 2 (no SDK installed yet).

Once the Early Access SDK is installed:
```bash
export RYZEN_AI_INSTALLATION_PATH=<path>/ryzen_ai-1.7.0/venv
export XLNX_VART_FIRMWARE=<path>/voe/1x4.xclbin
```

---

## Problem 6: Python Environment & Runtime Library Mismatch

**Impact:** Low — all libraries exist on disk, just need `LD_LIBRARY_PATH`.

The CVML VitisAI EP requires specific shared libraries at runtime. All are present but two are not on the default library search path:

| Library | Required By | Status |
|---|---|---|
| `libpython3.10.so.1.0` | VitisAI EP (CVML) | Present in `py310` env at `$CONDA_PREFIX/lib/` — needs `LD_LIBRARY_PATH` |
| `libboost_filesystem.so.1.74.0` | VitisAI EP (CVML) | Present in `py310` env at `$CONDA_PREFIX/lib/` — needs `LD_LIBRARY_PATH` |
| `libboost_filesystem.so.1.83.0` | (system) | Found at `/lib64/` — wrong version for CVML (needs 1.74) |
| `libgomp.so.1` | ORT parallel execution | Found at `/lib64/libgomp.so.1` |
| `libstdc++.so.6` | C++ runtime | Found at `/lib64/libstdc++.so.6` |

### Conda Environments

| Environment | Python | Purpose |
|---|---|---|
| base | 3.13.12 | System default — too new for Ryzen AI |
| py310 | 3.10.19 | CVML runtime deps — has libpython3.10 + boost 1.74 |
| ryzenai-test | 3.12.12 | SDK 1.7 target (sweet spot: broad compatibility) |

**To fix:** Activate `py310` and set `LD_LIBRARY_PATH`. Both required libraries are already installed — no `conda install` needed.

```bash
# Make CVML runtime deps discoverable (both libs already present)
conda activate py310
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib:$LD_LIBRARY_PATH

# Verify
ldd /path/to/libonnxruntime_vitisai_ep.so | grep -E 'python|boost'
```

---

## Unified Memory Architecture

Strix Halo is a **unified-memory APU** — CPU, GPU, and NPU all share the same 96 GB LPDDR5X pool. AMD's [Strix Halo optimization guide](https://rocm.docs.amd.com/en/latest/how-to/system-optimization/strixhalo.html) makes several key recommendations:

### Memory Model

```
┌─────────────────────────────────────────────────────────────────┐
│            96 GB LPDDR5X (Physical, 8000 MT/s Micron)           │
│            93.9 GiB visible to OS (rest: BIOS/firmware reserve) │
│                                                                 │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │  CPU (host)  │  │  GPU (VRAM)  │  │   GPU (GTT / shared)   │ │
│  │  72.9 GiB    │  │  32 GiB BAR  │  │   124 GiB addressable  │ │
│  │  available   │  │  2.6 GiB used│  │   9.2 GiB used         │ │
│  └─────────────┘  └──────────────┘  └────────────────────────┘ │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  NPU (amdxdna) — RyzenAI-npu5, FW 1.0.0.166                 ││
│  │  DMA from host memory — no dedicated pool                   ││
│  │  Operational: xrt-smi detects device (sudo required)        ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

> **Memory accounting note:** dmidecode reports 7 × 16 GB LPDDR5 entries (112 GB), but the OS sees 93.9 GiB (~100.8 GB). The ~11 GB gap is BIOS-reserved (VRAM carve-out + firmware). GTT's 124 GiB addressable space exceeds physical RAM because it includes swap-backed virtual ranges — actual physical backing is shared with CPU memory.

**Key principles from AMD:**

- **GTT (shared) memory is the real pool to manage**, not "fake discrete VRAM." On unified-memory APUs, GTT-backed allocations draw from the same physical LPDDR5X as CPU memory.
- **Keep dedicated BIOS VRAM reservation small.** Use shared memory / TTM tuning instead of large carved-out VRAM.
- **Do not model the iGPU as a discrete exclusive GPU.** Multiple workloads can and do share it concurrently via different access paths (ROCm, Vulkan, VAAPI).
- **NPU uses DMA from host memory** — it has no dedicated pool. Resource contention is about host memory bandwidth, not allocation.

### Implications for Kubernetes

This memory model means Kubernetes resource modeling needs to reflect reality:

1. **Pod `limits.memory` constrains the container's host memory allocation**, which is the same physical pool the GPU and NPU draw from.
2. **There is no separate "GPU memory" to limit** — `GPU_MAX_HEAP_SIZE` is a hint to the ROCm runtime, not an OS-level enforcement.
3. **Total workload memory footprint = host allocations + GTT allocations + VRAM carve-out.** Over-provisioning host memory to pods can starve GPU/NPU workloads indirectly.

---

## Kubernetes Architecture

### Current State: The Architectural Smell

The cluster structure is not the immediate blocker, but it can be improved. The core problem is not "k8s bad" — it is that one shareable integrated GPU/NPU/unified-memory host runs several very different classes of workloads, while Kubernetes mostly sees a pile of containers and some host devices. The scheduler is clever, but not clairvoyant.

### Before: Current Layout (Flat, No Policy)

```
┌─ Kubernetes Cluster (kubeadm) ─────────────────────────────────────────────┐
│  Node: fedora · Flannel CNI · local-path storage                           │
│                                                                            │
│  Namespaces:                                                               │
│  ┌─ default ──────────────────────────────────────────────────────────────┐│
│  │  ArkCase core, Portal, LDAP, Solr, PostgreSQL, ActiveMQ               ││
│  │  Ollama (ROCm GPU), Qdrant, Plex (VAAPI GPU)                         ││
│  │  [No resource quotas · No priority classes · Mixed workload types]    ││
│  └────────────────────────────────────────────────────────────────────────┘│
│  ┌─ music-intelligence ──────────────────────────────────────────────────┐│
│  │  ACE-Step (ROCm GPU), BS-RoFormer (ROCm GPU), Whisper (Vulkan GPU)   ││
│  │  Music API (CPU), Speech Gateway (CPU), PostgreSQL (CPU), MinIO (CPU) ││
│  │  [GPU_MAX_HEAP_SIZE env vars · No quotas · No priorities]             ││
│  └────────────────────────────────────────────────────────────────────────┘│
│  ┌─ Other namespaces ────────────────────────────────────────────────────┐│
│  │  cert-manager, ingress-controller, kube-flannel, kube-system,         ││
│  │  local-path-storage, monitoring                                       ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  Problems:                                                                 │
│  · No workload classification — infra and AI in same namespace             │
│  · No ResourceQuota or LimitRange on any namespace                         │
│  · No PriorityClass — all pods equal, no preemption policy                 │
│  · GPU modeled as exclusive device but actually shared concurrently        │
│  · No node labels/taints for accelerator affinity                          │
│  · NPU experiments would add another layer without native validation       │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                     ┌──────────────┴──────────────┐
                     │       Fedora 43 Host         │
                     │  Kernel 6.18.13 (NPU working) │
                     │  96 GB unified LPDDR5X        │
                     │  No quotas, no priorities     │
                     └──────────────────────────────┘
```

### After: Target Layout (Workload Classes + Policy)

Split the world into four workload classes, each with its own policy:

| Class | Namespaces | Characteristics | Policy |
|---|---|---|---|
| **Stable Infra** | `default` (ArkCase stack) | CPU-only, long-running, must survive GPU pressure | High priority, memory-capped, no GPU access |
| **Online AI** | `ai-online` | Latency-sensitive inference (Ollama, Whisper ASR) | Medium-high priority, GPU access, preempts batch |
| **Batch/Interactive AI** | `ai-batch` | Throughput-oriented (ComfyUI, ACE-Step, BS-RoFormer, Jupyter) | Medium priority, GPU access, preemptable |
| **Media/Sidecar** | `media` | Plex (VAAPI only), low priority | Low priority, VAAPI-only, no ROCm |

```
┌─ Kubernetes Cluster (kubeadm) ─────────────────────────────────────────────┐
│  Node: fedora · Flannel CNI · local-path storage                           │
│  Labels: accelerator/gpu=radeon-8060s, accelerator/npu=aie2p-17f0          │
│                                                                            │
│  PriorityClasses:                                                          │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │  cluster-infra: 1000  (system + cert-manager + ingress)               ││
│  │  stable-infra:   900  (ArkCase, LDAP, Solr, PostgreSQL, ActiveMQ)     ││
│  │  online-ai:      700  (Ollama, Whisper ASR — latency-sensitive)       ││
│  │  batch-ai:       500  (ComfyUI, ACE-Step, BS-RoFormer, Jupyter)       ││
│  │  media:          300  (Plex)                                          ││
│  │  experiment:     100  (NPU dev/test — preemptable by everything)      ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  ┌─ default (Stable Infra) ──────────────────────────────────────────────┐│
│  │  ResourceQuota: requests.memory=40Gi, limits.memory=48Gi              ││
│  │  LimitRange: default 512Mi-4Gi per container                          ││
│  │  PriorityClass: stable-infra (900)                                    ││
│  │                                                                        ││
│  │  ┌────────────┐ ┌──────┐ ┌──────┐ ┌──────────┐ ┌─────────┐           ││
│  │  │arkcase-core│ │ LDAP │ │ Solr │ │PostgreSQL│ │ActiveMQ │           ││
│  │  └─────┬──────┘ └──────┘ └──────┘ └──────────┘ └─────────┘           ││
│  │        ├──→ Ollama (ai-online)                                        ││
│  │        └──→ Qdrant (stable-infra)                                     ││
│  │  ┌────────────┐ ┌──────────┐                                          ││
│  │  │  Portal    │ │  Qdrant  │                                          ││
│  │  └────────────┘ └──────────┘                                          ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  ┌─ ai-online (Online AI Services) ─────────────── /dev/dri + /dev/kfd ─┐│
│  │  ResourceQuota: requests.memory=48Gi, limits.memory=64Gi              ││
│  │  PriorityClass: online-ai (700)                                       ││
│  │                                                                        ││
│  │  ┌──────────────┐  ┌──────────────┐                                   ││
│  │  │   Ollama     │  │ Whisper ASR  │                                   ││
│  │  │  gemma3      │  │ large-v3-    │                                   ││
│  │  │  functiongem │  │ turbo :8080  │                                   ││
│  │  │  nomic-embed │  │ (Vulkan→NPU) │                                   ││
│  │  │  :11434      │  │              │                                   ││
│  │  └──────────────┘  └──────────────┘                                   ││
│  │                                                                        ││
│  │  Future (NPU validated):                                              ││
│  │  ┌──────────────┐  ┌──────────────┐                                   ││
│  │  │ Whisper ASR  │  │  LLM (OGA)   │  ← /dev/accel/accel0             ││
│  │  │ VitisAI ORT  │  │  NPU offload │                                   ││
│  │  └──────────────┘  └──────────────┘                                   ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  ┌─ ai-batch (Batch/Interactive AI) ─────────────── /dev/dri + /dev/kfd ─┐│
│  │  ResourceQuota: requests.memory=32Gi, limits.memory=48Gi              ││
│  │  PriorityClass: batch-ai (500) — preemptable by online-ai             ││
│  │                                                                        ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 ││
│  │  │   ComfyUI    │  │  ACE-Step    │  │ BS-RoFormer  │                 ││
│  │  │  Stable Diff │  │  Music Gen   │  │  Source Sep  │                 ││
│  │  │  txt2img     │  │  v1.5        │  │  6-stem      │                 ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘                 ││
│  │  ┌──────────────┐                                                      ││
│  │  │  JupyterHub  │  GPU/CPU profile select; idle-culler: 3600s         ││
│  │  └──────────────┘                                                      ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  ┌─ music-intelligence (App Services) ───────────────────────────────────┐│
│  │  ResourceQuota: requests.memory=8Gi, limits.memory=12Gi               ││
│  │  PriorityClass: stable-infra (900) — these are app backends           ││
│  │                                                                        ││
│  │  ┌──────────┐ ┌────────────────┐ ┌──────────────┐ ┌──────────────┐   ││
│  │  │Music API │ │Speech Gateway  │ │  PostgreSQL  │ │    MinIO     │   ││
│  │  │ FastAPI  │ │ Swift/gRPC     │ │  + pgvector  │ │  S3 storage  │   ││
│  │  └──────────┘ └────────────────┘ └──────────────┘ └──────────────┘   ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  ┌─ media (Media/Sidecar) ─────────────────────────── /dev/dri only ────┐│
│  │  PriorityClass: media (300) — lowest non-experiment priority          ││
│  │                                                                        ││
│  │  ┌──────────────┐                                                      ││
│  │  │    Plex      │  H.264/HEVC/VP9 VAAPI transcode · :32400           ││
│  │  └──────────────┘                                                      ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                            │
│  ┌─ Cluster Operations (~/projects/dev/) ────────────────────────────────┐│
│  │  systemd: arkcase-startup.service (post-reboot recovery)              ││
│  │  Scripts: setup-linux.sh · arkcase-shutdown.sh · reset-k8s-arkcase.sh ││
│  │  Dev:     arkcase-jkube-deploy.sh · arkcase-dev-profile.sh            ││
│  │  Data:    populate-foia-sample-data.sh                                ││
│  │  Qdrant:  setup-qdrant.sh (start/stop/port-forward)                   ││
│  │  ECR:     aws-arkcase-pull secret (auto-refreshed, 12h expiry)        ││
│  └────────────────────────────────────────────────────────────────────────┘│
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                     ┌──────────────┴──────────────┐
                     │       Fedora 43 Host         │
                     │  Kernel 6.19.2+ (target)     │
                     │  amdgpu · amdxdna · VAAPI    │
                     │  96 GB unified LPDDR5X        │
                     │  kubeadm · containerd         │
                     │  Flannel · local-path-storage │
                     └──────────────────────────────┘
```

### Workload Class Design Principles

**1. PriorityClass + Preemption**

Define priority classes so that when resources are tight, the cluster knows what to sacrifice:

```yaml
# Example: online AI can preempt batch AI, but not stable infra
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: online-ai
value: 700
preemptionPolicy: PreemptLowerPriority
description: "Latency-sensitive AI inference (Ollama, Whisper ASR)"
```

This means: if Ollama needs memory and ACE-Step (batch-ai, 500) is using it, the scheduler can evict ACE-Step. ArkCase (stable-infra, 900) is never evicted by AI workloads.

**2. LimitRange — Set Defaults and Ceilings**

Prevent any single container from consuming unlimited resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: ai-batch
spec:
  limits:
  - default:
      memory: 8Gi
      cpu: "4"
    defaultRequest:
      memory: 2Gi
      cpu: "1"
    type: Container
```

**3. ResourceQuota — Cap Namespace-Wide Consumption**

Prevent any one workload class from starving the others:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ai-batch-quota
  namespace: ai-batch
spec:
  hard:
    requests.memory: 32Gi
    limits.memory: 48Gi
    requests.cpu: "16"
    limits.cpu: "24"
```

**4. iGPU Resource Modeling — Don't Fake Exclusivity**

For a single shareable integrated GPU, do **not** pretend Kubernetes can perfectly schedule unified-memory contention with a single `gpu=1` token. Extended resources are integer resources, cannot be overcommitted, and devices are not shared between containers. That works for discrete "give me one GPU" cases, but is a clumsy fit for a single shareable iGPU that multiple workloads already use concurrently.

Instead:
- For the **iGPU**, centralize access through long-running services and queues. Use namespace quotas + priorities to control blast radius. Pass `/dev/dri` and `/dev/kfd` to GPU namespaces via `hostPath` or a simple device plugin, but don't model it as `1` exclusive GPU.
- For the **NPU** (once validated), use a clean explicit resource model or device plugin because you'll likely want deliberate, low-concurrency use. One workload at a time on the NPU is a reasonable starting model.

**5. Node Labels and Affinity**

Even on a single node, labels document capabilities and prepare for future multi-node:

```yaml
# Apply to the node
kubectl label node fedora \
  accelerator/gpu=radeon-8060s \
  accelerator/npu=aie2p-17f0 \
  topology.kubernetes.io/zone=local

# Use in pod specs
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: accelerator/gpu
          operator: Exists
```

---

## Solution Strategy

### Already in Place

These components are confirmed working and require no action:

- XRT 2.19.0 installed with all CLI tools (`xrt-smi`, `xbutil`, `xclbinutil`); passwordless sudo + ldconfig configured
- NPU device node `/dev/accel/accel0` with correct permissions (666) — **NPU detected and operational**
- NPU firmware `17f0_11/npu.sbin.1.0.0.166.xz` present and loaded
- Kernel 6.18.13-200.fc43 — SVA regression resolved, dmesg clean
- CVML C++ libs (67 .so + 5 .xclbin) including VitisAI EP
- Build toolchain (GCC 15.2.0, cmake 4.2.3)
- Kubernetes cluster healthy (v1.35.0, all workloads running)
- GPU workloads stable (ROCm, Vulkan, VAAPI all operational)
- whisper.cpp VitisAI ORT backend code complete

### Phase 0 — Kernel Upgrade + Immediate Fixes (do now)

| # | Action | Effort |
|---|---|---|
| ~~1~~ | ~~**Upgrade kernel** — 6.18.13 installed, NPU working~~ | **DONE** (2026-03-05) |
| ~~2~~ | ~~**Register for AMD Early Access**~~ — EA program closed; all deliverables are now **GA (Generally Available)**. SDK should be downloadable directly. | **N/A** |
| 3 | **Fix CVML runtime deps** — activate `py310`, set `LD_LIBRARY_PATH` (deps already installed, just not on path) | 2 min |
| 4 | **Add `amdgpu.cwsr_enable=0`** to kernel cmdline if ROCm instability occurs after kernel upgrade — **not needed so far** (K8s GPU pods running normally on 6.18.13) | 2 min (if needed) |
| 4a | **Clean up completed K8s jobs** — `hook-image-awaiter` (default ns) and `bs-roformer-sw-benchmark` (music-intelligence) | 1 min |

### Phase 1 — Native NPU Validation (after kernel upgrade, before K8s)

> **Principle:** Do not containerize NPU experiments until native validation is boring. AMD's Linux paths still involve manual library paths and config edits for some flows. That is not where you add another layer of abstraction.

#### ~~Current Baseline~~ Verified Post-Upgrade State (2026-03-05)

Kernel 6.18.13 resolved the SVA regression. Verified:

```
$ sudo xrt-smi examine
NPU Firmware Version : 1.0.0.166
Device(s) Present
|BDF             |Name          |
|----------------|--------------|
|[0000:c8:00.1]  |RyzenAI-npu5  |

$ dmesg | grep amdxdna
amdxdna 0000:c8:00.1: enabling device (0000 -> 0002)
[drm] Initialized amdxdna_accel_driver 0.1.0 for 0000:c8:00.1 on minor 0
# No *ERROR* lines — clean
```

> **Note:** XRT 2.19.0 requires `euid == 0` to enumerate NPU devices. Fixed with passwordless sudo rule (`/etc/sudoers.d/xrt-smi`) and shell alias. XRT libs added to system ldconfig (`/etc/ld.so.conf.d/xrt.conf`).

| # | Action | Effort |
|---|---|---|
| ~~5~~ | ~~**Reboot and verify NPU** — RyzenAI-npu5 detected, FW 1.0.0.166, dmesg clean~~ | **DONE** (2026-03-05) |
| 6 | **Test CVML C++ inference** — run a simple CNN model through VitisAI EP with CVML libs (activate `py310`, set `LD_LIBRARY_PATH`) | 30 min |
| 7 | **Re-test whisper.cpp VitisAI ORT** — check if ops still fall back to CPU (expected: yes, pending partition pass) | 10 min |

### Phase 2 — Kubernetes Policy (parallel with Phase 1)

This can happen independently of NPU work:

| # | Action | Effort |
|---|---|---|
| 8 | **Create PriorityClasses** — cluster-infra (1000), stable-infra (900), online-ai (700), batch-ai (500), media (300), experiment (100) | 15 min |
| 9 | **Add LimitRange** to `default`, `ai-online`, `ai-batch` namespaces | 15 min |
| 10 | **Add ResourceQuota** to namespace groups — cap total memory per workload class | 15 min |
| 11 | **Label the node** — `accelerator/gpu=radeon-8060s`, `accelerator/npu=aie2p-17f0` | 2 min |
| 12 | **Migrate GPU workloads to class-aware namespaces** — move Ollama/Whisper to `ai-online`, ComfyUI/ACE-Step/BS-RoFormer to `ai-batch`, Plex to `media` | 1 hr |

> **Note on namespace migration:** This can be done incrementally. Start with PriorityClasses and LimitRanges in existing namespaces, then create new namespaces and migrate workloads one at a time. The `music-intelligence` CPU-only app services can stay in their own namespace with `stable-infra` priority.

### Phase 3 — SDK Integration (SDK is now GA)

| # | Action | Effort |
|---|---|---|
| 13 | Install Ryzen AI SDK 1.7 for Linux in `ryzenai-test` env (Python 3.12) | 30 min |
| 14 | Set `RYZEN_AI_INSTALLATION_PATH` and `XLNX_VART_FIRMWARE` | 2 min |
| 15 | Test NPU inference with bundled examples (LLMs, CNNs) | 30 min |
| 16 | Check if SDK includes `libvaip-pass_vaiml_partition.so` for Whisper NPU offload | 5 min |

### Phase 4 — NPU Workload Migration (once SDK validated + K8s policy in place)

| # | Action | Effort |
|---|---|---|
| 17 | Migrate whisper.cpp ASR from Vulkan GPU to NPU (VitisAI ORT) — frees GPU memory | 1 hr |
| 18 | Evaluate LLM inference on NPU via OGA — potential Ollama replacement for ArkCase chatbot | 2 hr |
| 19 | Add NPU device passthrough to K8s pod specs (music-intelligence NPU probe already ready) | 30 min |
| 20 | Model NPU as explicit K8s resource with `experiment` priority class initially | 30 min |

### Phase 5 — Second Node (future, if uptime matters)

The next real architectural jump is a second node, not distro gymnastics. If uptime starts to matter:

| # | Action | Effort |
|---|---|---|
| 21 | Add second node (could be Ubuntu 24.04.3 for AMD-reference lane) | 2 hr |
| 22 | Use taints/tolerations to separate stable infra from AI experiments | 30 min |
| 23 | Move batch AI workloads to second node, keep online AI on primary | 1 hr |

> Adding a second node gives you real HA for ArkCase, GPU scheduling flexibility, and the ability to run Ubuntu alongside Fedora without reinstalling anything.

---

## Operational Reference

### Source Projects

| Directory | Deploys | Orchestration |
|---|---|---|
| `~/projects/dev/` | ArkCase stack, Ollama, Qdrant, HAProxy, cert-manager, cluster infra | Shell scripts + Helm |
| `~/comfyui-onprem-k8s/` | ComfyUI, JupyterHub | Helm charts |
| `~/music-intelligence/` | ACE-Step, BS-RoFormer, Whisper, Music API, Speech Gateway, PostgreSQL, MinIO | Kustomize (phased overlays) |
| `~/plex-onprem-k8s/` | Plex Media Server | Helm chart |
| `~/RyzenAI-SW/` | whisper.cpp VitisAI ORT backend (dev/test, not K8s) | CMake (native build) |

### Workload Inventory

| Project | Service | Compute | Backend | Status |
|---|---|---|---|---|
| `~/projects/dev/` | Ollama (gemma3, functiongemma, nomic-embed) | GPU | ROCm | Working |
| `~/projects/dev/` | Qdrant (vector DB for RAG / semantic search) | CPU | Native | Working |
| `~/projects/dev/` | ArkCase core + Portal (FOIA case management) | CPU | JVM (Tomcat) | Working |
| `~/projects/dev/` | ArkCase infra (LDAP, Solr, PostgreSQL, ActiveMQ) | CPU | Various | Working |
| `~/comfyui-onprem-k8s/` | ComfyUI (Stable Diffusion) | GPU | ROCm (`corundex/comfyui-rocm`) | Working |
| `~/comfyui-onprem-k8s/` | JupyterHub (interactive ComfyUI) | GPU/CPU | ROCm (profile select) | Working |
| `~/music-intelligence/` | ACE-Step v1.5 (music generation) | GPU | ROCm 7.2 + PyTorch 2.9.1 | Working |
| `~/music-intelligence/` | BS-RoFormer (6-stem source separation) | GPU | ROCm 7.2 + PyTorch 2.9.1 | Working |
| `~/music-intelligence/` | Whisper large-v3-turbo (streaming ASR) | GPU | Vulkan (whisper.cpp) | Working |
| `~/music-intelligence/` | Music API, Speech Gateway, PostgreSQL, MinIO | CPU | FastAPI, Swift/gRPC | Phase 0-1 |
| `~/plex-onprem-k8s/` | Plex Media Server (video transcode) | GPU | VAAPI only (no ROCm) | Working |
| `~/RyzenAI-SW/` | whisper.cpp VitisAI ORT (encoder offload) | NPU | VitisAI EP + CVML | Blocked (pass) — NPU hw ready |
| `~/RyzenAI-SW/` | CNN/Transformer inference | NPU | VitisAI EP | Blocked (SDK) — NPU hw ready |
| `~/RyzenAI-SW/` | LLM inference (OGA) | NPU | Ryzen AI SDK | Blocked (SDK) — NPU hw ready |

### GPU Contention Risk

All ROCm workloads share the same Radeon 8060S (96 GB unified memory). There is no GPU resource scheduling — workloads run privileged with full device access. Current mitigation is via Kubernetes resource requests/limits and `GPU_MAX_HEAP_SIZE` env vars in music-intelligence.

| Service | Memory Guardrail | Notes |
|---|---|---|
| Ollama | `limits.memory: 64Gi` | `OLLAMA_MAX_LOADED_MODELS=3` |
| ComfyUI | None configured | SD 1.5 checkpoint ~4 GB VRAM |
| ACE-Step | `GPU_MAX_HEAP_SIZE=85`, `HIP_HIDDEN_FREE_MEM=8192` | ~10 GB model cache |
| BS-RoFormer | Same as ACE-Step | ~700 MB model |
| Qdrant | N/A (CPU only) | 512Mi-2Gi memory; 10Gi vector storage |
| Plex | N/A | VAAPI transcode only; minimal memory |

> **Opportunity:** When NPU workloads come online, offloading Whisper ASR and LLM inference to the NPU will free GPU memory for the generative workloads (ComfyUI, ACE-Step) that cannot run on the NPU. A second opportunity: if Ollama LLM inference moves to NPU via OGA, the ArkCase chatbot integration could switch from `http://ollama:11434` to an NPU-backed service, freeing significant GPU memory.

### ROCm Version Divergence

| Project | ROCm Version | Base Image |
|---|---|---|
| Ollama | Container-bundled | `ollama/ollama:rocm` |
| ComfyUI | Container-bundled | `corundex/comfyui-rocm:latest` |
| music-intelligence | 7.2 (pinned) | `rocm/pytorch:rocm7.2_ubuntu24.04_py3.12_pytorch_release_2.9.1` |
| Plex | N/A (VAAPI) | `plexinc/pms-docker:latest` |

All ROCm containers use `HSA_OVERRIDE_GFX_VERSION=11.0.0` for gfx1151 compatibility. The host does not need ROCm installed — each container brings its own ROCm userspace.

### Cluster Lifecycle

All managed from `~/projects/dev/`:

| Script | Purpose |
|---|---|
| `ark_k8s_init/initialize` | Bootstrap kubeadm cluster (K8s 1.35, containerd 2.1.6, Flannel, Helm) |
| `setup-linux.sh` | Deploy full stack (cert-manager, HAProxy, ArkCase, Ollama, Qdrant, TLS) |
| `arkcase-startup.sh` | Post-reboot recovery (systemd-triggered: ECR refresh, pod cleanup) |
| `arkcase-shutdown.sh` | Graceful scale-to-zero before reboot/maintenance |
| `reset-k8s-arkcase.sh` | Multi-level reset (`--pods`, default data reset, `--nuclear`) |
| `arkcase-jkube-deploy.sh` | Dev build cycle (Maven WAR, JKube image, containerd import, Helm upgrade) |
| `setup-qdrant.sh` | Qdrant lifecycle (start/stop/status/port-forward) |
| `populate-foia-sample-data.sh` | Load 55 FOIA requests + 25 people + test data via REST API |

---

## Key References

### AMD Documentation
- [Strix Halo System Optimization Guide](https://rocm.docs.amd.com/en/latest/how-to/system-optimization/strixhalo.html) — unified memory model, GTT tuning, Fedora 43 support
- [ROCm Compatibility Matrix](https://rocm.docs.amd.com/en/latest/compatibility/compatibility-matrix.html) — official distro/GPU support
- [ROCm Linux Support Matrices (Radeon/Ryzen)](https://rocm.docs.amd.com/projects/radeon-ryzen/en/latest/docs/compatibility/compatibilityryz/native_linux/native_linux_compatibility.html) — Ubuntu 24.04.3 for gfx1151
- [Ryzen AI SDK — Running LLMs on Linux](https://ryzenai.docs.amd.com/en/latest/llm_linux.html) — Linux OGA flows
- [Ryzen AI SDK — Whisper.cpp Support](https://ryzenai.docs.amd.com/en/latest/whisper_cpp.html) — Windows-only NPU, Linux planned
- [AMD Ryzen AI SDK](https://ryzenai.docs.amd.com/en/latest/inst.html) — SDK 1.7 now GA (Early Access program closed)
- [AMD Strix Halo Toolboxes (Ollama/ROCm)](https://github.com/kyuz0/amd-strix-halo-toolboxes)

### Kernel & Driver
- [amdxdna Kernel Driver Docs](https://docs.kernel.org/accel/amdxdna/amdnpu.html)
- [RyzenAI-SW Issue #178 — NPU on Linux](https://github.com/amd/RyzenAI-SW/issues/178)
- [ONNX Runtime VitisAI EP Build Failure — #27097](https://github.com/microsoft/onnxruntime/issues/27097)

### Kubernetes
- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/) — namespace defaults and ceilings
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) — namespace-wide caps
- [Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/) — hardware resource modeling
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) — labels, affinity, taints, tolerations
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/) — priority classes

### Community
- [Framework Community — ROCm Stable Configs (Jan 2026)](https://community.frame.work/t/linux-rocm-january-2026-stable-configurations-update/79876) — Fedora recommended for gfx1151
