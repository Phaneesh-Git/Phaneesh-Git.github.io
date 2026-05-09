---
layout: post
title: "Self-Hosted AI Terminal: Building a Private LLM Environment with Docker and OpenCode"
date: 2026-05-09
categories: [AI, Docker]
tags: [Gemma, llama.cpp, Self-Hosted, OpenCode, DevOps]
image:
  path: /assets/headers/2026-05-09.jpg
  alt: "Local AI Terminal Setup"
  width: 1200
  height: 630
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* May 9, 2026 | *Repository:* [Local AI OpenCode](https://github.com/Phaneesh-Git/Local_Ai_ON_OPENCODE)

* * *

## Introduction
In the era of privacy-conscious computing, running Large Language Models (LLMs) locally has become a priority for developers and engineers. While cloud-based APIs are convenient, they often come with privacy trade-offs and recurring costs. This guide explores a robust, containerized approach to hosting your own AI terminal using `llama.cpp` and a custom terminal environment called OpenCode.

## The Architecture
The setup relies on Docker Compose to orchestrate two primary services:
1.  **gemma-llama**: A high-performance inference server running the Gemma-2 (4B) model via `llama.cpp`. It provides an OpenAI-compatible API.
2.  **opencode-terminal**: A specialized container designed for interacting with the AI, pre-configured with necessary tools and persistent volume mounts for seamless development.

### Service Breakdown

#### The AI Engine: llama.cpp
The `gemma-llama` service uses the `ghcr.io/ggml-org/llama.cpp:full` image. It is configured to:
- Serve the model on port 8080.
- Use the `gemma-4-E4B-it-Q4_K_M.gguf` model file.
- Disable reasoning overhead (`--reasoning off`) for faster response times in a terminal context.
- Implement a robust healthcheck that ensures the API is ready before the terminal starts.

#### The Interface: OpenCode Terminal
The `opencode-terminal` provides the interactive layer. By mounting the Docker socket and local configuration directories, it allows the AI to assist in real-time system operations while maintaining state across sessions.

## Configuration Highlights
The `docker-compose.yaml` file demonstrates best practices for local AI deployments:

```yaml
services:
  gemma-llama:
    image: ghcr.io/ggml-org/llama.cpp:full
    container_name: gemma-llama
    command: >
      --server
      -m /models/gemma-4-E4B-it-Q4_K_M.gguf
      --port 8080
      --host 0.0.0.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/v1/models"]
      interval: 5s
      timeout: 3s
      retries: 20
```

This healthcheck is crucial as LLMs can take several seconds (or minutes) to load into memory. By using `depends_on` with `service_healthy`, the terminal container only starts when the AI is ready to respond.

## Workflow: How to Use It
1.  **Initialization**: Run `docker compose up` to start the backend.
2.  **Interactive Session**: Use `docker compose run --rm opencode-terminal` to enter the AI-powered environment.
3.  **Persistence**: Your configurations and models are stored in `$HOME/.opencode` and `~/gemma_models`, ensuring your setup is durable.

## Key Takeaways
- **Privacy First**: All data remains on your local machine.
- **Portability**: Docker ensures the setup works across different Linux environments with minimal friction.
- **Efficiency**: llama.cpp's GGUF support allows running capable models even on hardware without high-end GPUs.

By bridging the gap between raw LLM inference and a functional terminal interface, this project provides a powerful foundation for a truly private developer assistant.
