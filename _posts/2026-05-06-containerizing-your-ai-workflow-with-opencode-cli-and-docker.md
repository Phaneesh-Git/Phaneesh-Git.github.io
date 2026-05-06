---
layout: post
title: "Containerizing Your AI Workflow with OpenCode CLI and Docker"
date: 2026-05-06
categories: [Docker, AI]
tags: [Docker, OpenCode, AI, DevOps]
image:
  path: /assets/headers/2026-05-06.jpg
  alt: "OpenCode CLI Docker Setup"
  width: 1200
  height: 630
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* May 6, 2026 | *Repository:* AI_terminal_OpenCode (https://github.com/Phaneesh-Git/AI_terminal_OpenCode)

* * *

## Introduction

In the rapidly evolving landscape of Artificial Intelligence, having a consistent and portable development environment is crucial. The **OpenCode CLI** is a powerful tool for interacting with various AI models, and by containerizing it with **Docker**, we can ensure that our setup is reproducible across any machine.

In this guide, we will walk through the Dockerization of the OpenCode CLI, exploring the Dockerfile configuration and setting up a convenient workflow for daily use.

## The Dockerfile: Building the Foundation

The core of our containerized setup is the `Dockerfile`. It leverages Ubuntu as the base image and sets up the necessary dependencies, including Node.js (via NVM) and the OpenCode CLI itself.

### Key Components:

1.  **Base Image**: We start with `ubuntu:latest` for a clean, flexible environment.
2.  **NVM & Node.js**: We install Node Version Manager (NVM) to manage Node.js version 24, which is required by the OpenCode CLI.
3.  **OpenCode CLI Installation**: The CLI is installed directly via a curl-to-bash script.
4.  **Environment Configuration**: We ensure that NVM and OpenCode binaries are correctly added to the system `PATH`.

```dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get install -y curl bash ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ENV NVM_DIR=/root/.nvm
ENV BASH_ENV=/root/.bash_env

# Install NVM
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash \
    && echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> $BASH_ENV

# Install Node 24
RUN bash -c "nvm install 24"

# Install OpenCode CLI
RUN curl -fsSL https://opencode.ai/install | bash
ENV PATH="/root/.opencode/bin:${PATH}"

ENTRYPOINT ["/bin/bash"]
```

## Running OpenCode CLI with Ease

To make the containerized CLI feel like a native tool, we can set up a shell alias. This alias handles complex Docker flags, such as mounting the Docker socket (to allow OpenCode to manage other containers) and persistent volume mounts for configuration and sessions.

### The `opencode` Alias

Add this to your `~.bashrc`:

```bash
alias opencode='docker run -it --rm \
  -v $HOME/.opencode:/root/.config/opencode \
  -v $HOME/.local/share/opencode/:/root/.local/share/opencode \
  -v $PWD:/work \
  -w /work \
  --net host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  opencode:usethis'
```

### Why these flags?

*   **`--net host`**: Allows the container to use the host's network directly.
*   **`-v /var/run/docker.sock:/var/run/docker.sock`**: Enables the OpenCode CLI to interact with the host's Docker daemon.
*   **Persistent Mounts**: Ensures that your `/model` selections, `/connect` credentials, and `/sessions` history are preserved across container restarts.

## Conclusion

Containerizing the OpenCode CLI simplifies the setup process and provides a clean, isolated environment for your AI development. Whether you're switching between different models or managing multiple AI projects, this Docker-based approach ensures consistency and reliability.

Start exploring the power of AI from your terminal with OpenCode CLI and Docker today!
