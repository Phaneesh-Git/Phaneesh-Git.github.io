---
layout: post
title: "Mastering Local AI: Running Gemma Models with llama.cpp on WSL2 and Docker"
date: 2026-05-08
categories: [AI, DevOps]
tags: [LLM, llama.cpp, Gemma, WSL2, Docker, Quantization]
image:
  path: /assets/headers/2026-05-08.jpg
  alt: "Local AI with llama.cpp"
  width: 1200
  height: 630
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* May 8, 2026 | *Repository:* [Local_AI_On_LLAMA..CPP](https://github.com/Phaneesh-Git/Local_AI_On_LLAMA..CPP)

* * *

## Introduction

In the rapidly evolving landscape of Large Language Models (LLMs), the ability to run powerful models locally is a game-changer. Whether it's for privacy, cost-efficiency, or offline accessibility, local execution empowers developers and researchers alike. In this guide, we dive into **llama.cpp**, a high-performance implementation that brings models like Google's **Gemma** to your local machine using WSL2 and Docker.

## What is llama.cpp?

[llama.cpp](https://llama-cpp.com/) is a lightweight, pure C++ implementation designed to run LLMs with zero external dependencies and no Python overhead. It is optimized for various hardware backends, including CPU, NVIDIA GPU (CUDA), AMD GPU (ROCm), and Intel GPU (SYCL/OpenVINO).

**Key Highlights:**
- **Zero Cloud Dependencies:** Run entirely on your own hardware.
- **Quantization Support:** Natively supports memory-efficient quantized models (Q2 through Q8).
- **Multiple Interfaces:** Includes a versatile CLI (`llama-cli`) and a local HTTP server (`llama-server`).

## Understanding Quantization

AI models are massive. A 7B parameter model at full precision (F32) requires approximately 28GB of RAM. Quantization solves this by compressing the model's weights (e.g., from 32-bit to 4-bit), significantly reducing memory requirements with minimal loss in quality.

| Quant | Size | Quality | Use Case |
|---|---|---|---|
| `Q2_K` | Smallest | Lowest | Extremely limited RAM |
| `Q4_K_M` | Small | Good | **Best balance — most popular** |
| `Q8_0` | Largest | Near lossless | Maximum quality |

## Setting Up the Environment

The most streamlined way to get started is via official Docker images. Here is how you can set up a container in WSL2 for Intel iGPUs or NVIDIA GPUs.

### For Intel CPUs/iGPUs:
```shell
docker run -it \
  -v ~/models:/models \
  --device /dev/dri \
  -p 8080:8080 \
  --entrypoint bash ghcr.io/ggml-org/llama.cpp:full-intel
```

### For NVIDIA GPUs:
```shell
docker run -it \
  -v ~/models:/models \
  --gpus all \
  -p 8080:8080 \
  --entrypoint bash ghcr.io/ggml-org/llama.cpp:full-cuda
```

## Running Gemma Models

Once inside the container, you can run a model directly from Hugging Face or from a local GGUF file.

### Running from Hugging Face:
```shell
./llama-cli -hf ggml-org/gemma-4-E4B-it-GGUF:Q4_K_M
```

### Running from a Local File (with CPU only):
```shell
./llama-cli -m /models/gemma-4-E4B-it-Q4_K_M.gguf -ngl 0
```

## Hosting a Local Model Server

To integrate your model into other applications, you can host it as an HTTP API using `llama-server`.

```shell
export LLAMA_ARG_HOST=0.0.0.0
./llama-server -m /models/gemma-4-E4B-it-Q4_K_M.gguf \
  --port 8080 \
  -ngl 0 \
  --jinja \
  -c 8192 \
  --parallel 1 \
  --temperature 1.0
```

This exposes an OpenAI-compatible API at `http://localhost:8080`, allowing you to connect tools like **OpenCode** for a seamless development experience.

## Key Takeaways

1. **Accessibility:** llama.cpp makes running LLMs possible even on modest hardware.
2. **Efficiency:** Quantization is essential for balancing performance and memory.
3. **Flexibility:** Docker images provide a consistent environment across different hardware vendors.

By mastering these tools, you can build and experiment with state-of-the-art AI models entirely on your own terms. Happy coding!
