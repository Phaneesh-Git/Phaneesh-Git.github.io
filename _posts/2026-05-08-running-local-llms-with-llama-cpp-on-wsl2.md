---
layout: post
title: "Running Local LLMs with llama.cpp on WSL2"
date: 2026-05-08
categories: [AI, LLM]
tags: [llama.cpp, WSL2, Docker, AI, Local-LLM]
image:
  path: /assets/headers/2026-05-08.jpg
  alt: "Local LLM with llama.cpp"
  width: 1200
  height: 630
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* May 8, 2026 | *Repository:* [Local_AI_On_LLAMA..CPP](https://github.com/Phaneesh-Git/Local_AI_On_LLAMA..CPP)

* * *

## Introduction

Running large language models (LLMs) locally has become increasingly accessible thanks to projects like **llama.cpp**. This lightweight, high-performance implementation allows you to run massive models on your own hardware without needing expensive cloud APIs. Whether you are a developer looking for privacy or a tinkerer experimenting with AI, llama.cpp provides a robust framework to get started.

In this guide, we will explore how to set up and run LLMs locally on WSL2 using llama.cpp and Docker, covering everything from quantization to hosting your own model server.

## What is llama.cpp?

[llama.cpp](https://llama-cpp.com/) is a pure C++ implementation for running LLMs. It is designed for maximum portability and efficiency with zero external dependencies. 

Key highlights include:
* **Hardware Support:** Runs on CPU, NVIDIA GPU (CUDA), AMD GPU (ROCm), Intel GPU (SYCL), and Vulkan.
* **Quantization:** Natively supports compressed model formats (GGUF), allowing large models to fit into consumer-grade RAM.
* **Versatility:** Includes both a CLI (`llama-cli`) and a local HTTP server (`llama-server`).

## Understanding Quantization

AI models are traditionally massive. For example, a 7B parameter model in full precision (F32) requires approximately 28GB of RAM. **Quantization** solves this by compressing the model's weights (e.g., from 32-bit to 4-bit), reducing memory usage by up to 8x with minimal loss in quality.

| Quant | Size | Quality | Use Case |
|---|---|---|---|
| `Q2_K` | Smallest | Lowest | Extremely limited RAM |
| `Q4_K_M` | Small | Good | **Best balance — most popular** |
| `Q8_0` | Largest | Near lossless | Maximum quality |

## Setting Up llama.cpp on WSL2

The easiest way to get started is using official Docker images. Here is how you can set up a local environment on WSL2 using an Intel CPU:

```bash
docker run -it \
  -v ~/gemma_models:/models \
  -p 8080:8080 \
  --entrypoint bash ghcr.io/ggml-org/llama.cpp:full
```

### Running a Model via CLI

Once inside the container, you can run a model (e.g., Gemma 4B) using the `llama-cli`:

```bash
./llama-cli -m /models/gemma-4-E4B-it-Q4_K_M.gguf -ngl 0
```
*Note: The `-ngl 0` flag ensures the model runs on the CPU.*

## Hosting a Local Model Server

To expose your model as an HTTP API, you can use `llama-server`. This allows other applications to interact with your local LLM via standard web requests.

```bash
export LLAMA_ARG_HOST=0.0.0.0
./llama-server -m /models/gemma-4-E4B-it-Q4_K_M.gguf \
  --port 8080 \
  -ngl 0 \
  --jinja \
  -c 8192 \
  --parallel 1
```

Once running, you can access the built-in web UI and API at `http://localhost:8080`.

## Key Takeaways

1. **Local Control:** Run LLMs without cloud dependencies or API costs.
2. **Efficiency:** Use quantization to run powerful models on standard hardware.
3. **Flexibility:** Use Docker to easily switch between CPU and GPU backends (CUDA, ROCm, etc.).

By hosting your own models, you gain full control over your data and the AI's behavior, paving the way for more secure and personalized AI applications.

* * *
