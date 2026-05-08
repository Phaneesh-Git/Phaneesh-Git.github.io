---
layout: post
title: "Local AI: Running Gemma 4 with llama.cpp and Docker"
date: 2026-05-08
categories: [AI, DevOps]
tags: [llama.cpp, Gemma, Docker, Quantization]
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

In the rapidly evolving landscape of Artificial Intelligence, the ability to run Large Language Models (LLMs) locally has become a game-changer for developers and researchers alike. Whether it's for privacy, cost-efficiency, or offline access, local execution offers unparalleled control. One of the most powerful tools in this space is **llama.cpp**, a lightweight, C++ based implementation designed for high-performance inference.

In this guide, we'll dive deep into setting up **Gemma 4** models locally using llama.cpp and Docker, exploring the magic of quantization and how it enables powerful AI to run on consumer hardware.

## What is llama.cpp?

[llama.cpp](https://github.com/ggml-org/llama.cpp) is a high-performance LLM inference engine with zero dependencies and no Python overhead. It's written in pure C++, making it incredibly fast and portable.

**Key Highlights:**
- **Hardware Agnostic:** Supports CPU, NVIDIA (CUDA), AMD (ROCm), Intel (SYCL), and Vulkan.
- **Memory Efficient:** Native support for various quantization levels (Q2_K to Q8_0).
- **Tooling:** Includes `llama-cli` for interactive sessions and `llama-server` for hosting OpenAI-compatible APIs.

## Understanding Quantization

AI models are massive. A 7B parameter model in full precision (F32) requires approximately 28GB of RAM. Quantization solves this by compressing the model weights (e.g., from 32-bit to 4-bit), significantly reducing memory requirements with minimal impact on quality.

| Quant | Size | Quality | Use Case |
|---|---|---|---|
| `Q2_K` | Smallest | Lowest | Extremely RAM-constrained devices |
| `Q4_K_M` | Small | Good | **Recommended balance for most users** |
| `Q8_0` | Largest | Near Lossless | Maximum quality with sufficient RAM |

## Setting Up with Docker

Docker is the easiest way to get llama.cpp up and running without worrying about local build dependencies.

### Running on Linux (CPU/Intel iGPU)

```shell
docker run -it \
  -v ~/models:/models \
  --device /dev/dri \
  -p 8080:8080 \
  --entrypoint bash ghcr.io/ggml-org/llama.cpp:full-intel
```

### Running on WSL with NVIDIA GPU

```shell
docker run -it \
  -v ~/models:/models \
  --gpus all \
  -p 8080:8080 \
  --entrypoint bash ghcr.io/ggml-org/llama.cpp:full-cuda
```

## Running Your First Model

Once inside the container, you can run models directly from Hugging Face or from a local GGUF file.

### Direct from Hugging Face
```shell
./llama-cli -hf ggml-org/gemma-4-E4B-it-GGUF:Q4_K_M
```

### From Local File (with GPU Offloading)
```shell
./llama-cli -m /models/gemma-4-E4B-it-Q4_K_M.gguf -ngl 99
```

The `-ngl` flag specifies the number of layers to offload to the GPU. Setting it to `99` ensures all possible layers are handled by your graphics card.

## Hosting a Local AI Server

To use your model with external applications, you can host it as an HTTP API.

```shell
export LLAMA_ARG_HOST=0.0.0.0
./llama-server -m /models/gemma-4-E4B-it-Q4_K_M.gguf \
  --port 8080 \
  -ngl 99 \
  --jinja \
  -c 8192
```

Once running, the server provides an OpenAI-compatible API at `http://localhost:8080/v1`, allowing you to connect tools like **OpenCode** or custom scripts effortlessly.

## Conclusion

Running high-quality models like Gemma 4 locally is no longer reserved for those with industrial-grade hardware. With llama.cpp and quantization, the power of LLMs is at your fingertips. By leveraging Docker, you can maintain a clean, portable environment while maximizing your hardware's potential.

Happy Hacking!
