# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AMD Ryzen™ AI Software — examples, demos, and tutorials for deploying AI inference on AMD Ryzen™ AI processors with NPU (Neural Processing Unit) acceleration. This is a subset of the full Ryzen™ AI Software release; the main SDK must be installed separately via https://ryzenai.docs.amd.com/en/latest/inst.html.

## Repository Structure

- **CNN-examples/**: Computer vision models (ResNet, YOLOv8, MobileNetV2, torchvision models). Includes getting-started tutorials, quantization examples (AMD Quark), and iGPU inference.
- **LLM-examples/**: Large Language Model deployment using ONNX Runtime GenAI (OGA). Includes RAG pipelines (LangChain + OGA), Vision Language Models, and C++ OGA API usage.
- **Transformer-examples/**: DistilBERT text classification (BF16) and Whisper ASR.
- **Demos/**: Advanced pipelines — hybrid NPU-GPU execution, Whisper ASR demo.
- **Ryzen-AI-CVML-Library/**: Pre-built C++ CVML library with headers, Windows/Linux binaries, CMake integration, and sample apps.
- **onnx-benchmark/**: Performance benchmarking tool (`performance_benchmark.py`) with optional tkinter GUI (`benchmark_gui.py`). Measures throughput (fps) and latency (ms).
- **utilities/npu_check/**: C++ utility for NPU device detection and VitisAI EP compatibility verification.

## Architecture & Key Concepts

**Execution Providers** — all inference flows select one of:
- `VitisAIExecutionProvider` — NPU inference (primary target)
- `DmlExecutionProvider` — iGPU inference via DirectML
- `CPUExecutionProvider` — baseline/fallback

**Inference pattern (Python)**: Load ONNX model → create `onnxruntime.InferenceSession` with chosen EP → preprocess → `session.run()`.

**Quantization flow**: Float model → AMD Quark quantizer with calibration dataset → quantized ONNX model → deploy with VitisAI EP. Supported types: INT8, INT16, BF16, INT4 (AWQ).

**LLM flow**: Uses ONNX Runtime GenAI (OGA) API — either Python `onnxruntime_genai` or C++ `oga_api` — for token generation on NPU.

**Configuration files** found across examples:
- `vaip_config.json` — VitisAI EP session options
- `vaiml_config.json` — VAIML compilation settings and optimization levels
- `vitisai_config.json` — VitisAI-specific configuration

## Build & Run

Each example is self-contained with its own README.md and requirements.

**Python examples**: Conda-based environments cloned from the Ryzen AI SDK installer.
```bash
conda activate <ryzen-ai-env>
pip install -r requirements.txt   # per-example
python <script>.py
```

**C++ examples** (oga_api, npu_check, CVML samples): CMake-based.
```bash
cmake -B build -S . [-DONNXRUNTIME_ROOTDIR=...] [-DCVML_SDK_ROOT=...]
cmake --build build --config Release
```

**Benchmarking**:
```bash
cd onnx-benchmark
python performance_benchmark.py --input_model <model.onnx> --ep <vitisai|dml|cpu>
python benchmark_gui.py   # GUI launcher
```

## Environment

- **`RYZEN_AI_INSTALLATION_PATH`** — critical env var pointing to the Ryzen AI SDK installation.
- **Git LFS required** — large files (*.pt, *.onnx, *.dll, *.whl, *.lib, *.graphlib, *.so) are tracked via LFS. Run `git lfs pull` after cloning.
- **Primary platform**: Windows, with Linux support (Ubuntu 22.04/24.04).
- **NPU hardware IDs**: PHX/HPT = `0x1502`, STX/KRK = `0x17F0`.
