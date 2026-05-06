---
layout: post
title: "Containerizing Your AI Workflow with OpenCode CLI and Docker"
date: 2026-05-06
categories: [AI]
tags: [OpenCode, Docker, AI, CLI]
image:
  path: /assets/headers/2026-05-06.jpg
  alt: "OpenCode CLI Docker Setup"
  width: 1200
  height: 630
published: true
---
> **Complete Tech Blog & Setup Guide**  
> *Author:* Phaneesh | *Date:* May 6, 2026 | *Repository:* [AI_terminal_OpenCode](https://github.com/Phaneesh-Git/AI_terminal_OpenCode)

* * *

## Introduction

In the rapidly evolving landscape of AI-assisted development, having a consistent and portable environment is crucial. The **OpenCode CLI** is a powerful tool that brings AI capabilities directly to your terminal. However, managing dependencies like Node.js and ensuring the CLI behaves the same way across different machines can be a challenge.

In this guide, we will walk through how to containerize the OpenCode CLI using Docker. This setup ensures that your AI terminal is always ready to go, regardless of the host operating system, and it even allows the containerized AI to interact with your host's Docker engine.

## The Dockerfile: Building the Foundation

To create our environment, we use a custom Dockerfile based on Ubuntu. The goal is to install Node.js via NVM and then the OpenCode CLI itself.

### Key Components:

1.  **NVM (Node Version Manager)**: We use NVM to install Node.js 24, ensuring we have the exact version required by OpenCode.
2.  **OpenCode CLI Installation**: A simple curl command fetches and installs the latest binary.
3.  **Environment Persistence**: We configure `.bashrc` and a custom `BASH_ENV` to ensure the environment is correctly loaded in every session.

### The Dockerfile

```dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get install -y curl bash ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ENV NVM_DIR=/root/.nvm
ENV BASH_ENV=/root/.bash_env

# Install NVM and write initialization into BASH_ENV
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash \
    && echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> $BASH_ENV

# Install Node 24 using NVM
RUN bash -c "nvm install 24"

# Make NVM available in interactive shells
RUN echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> /root/.bashrc

# Install OpenCode CLI
RUN curl -fsSL https://opencode.ai/install | bash
ENV PATH="/root/.opencode/bin:${PATH}"

ENTRYPOINT ["/bin/bash"]
```

## Seamless Execution with Aliases

Running a Docker container with all the necessary mounts can result in a very long command. To make this "feel" like a local installation, we use a shell alias.

### The Magic Alias

```bash
alias opencode='docker run -it --rm \
  -v $HOME/.opencode:/root/.config/opencode \
  -v $HOME/.local/share/opencode/:/root/.local/share/opencode \
  -v $PWD:/work \
  -w /work \
  --net host \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /home/ubuntu/AI_LAB/AI_terminal_OpenCode:/root/work/AI_LAB/AI_terminal_OpenCode \
  opencode:usethis'
```

### Why This Works:

*   **Persistence**: Mounting `~/.opencode` ensures your login sessions and preferences survive container restarts.
*   **Docker-in-Docker**: Mounting `/var/run/docker.sock` allows the OpenCode CLI inside the container to manage Docker containers on your host machine.
*   **Context Awareness**: Mounting `$PWD` to `/work` allows the AI to see and interact with the files in your current directory.

## Conclusion

By containerizing the OpenCode CLI, you eliminate "it works on my machine" issues and gain a powerful, isolated environment for your AI-driven workflows. Whether you're connecting to Gemini, GPT-4, or using local models, this setup provides the stability and flexibility needed for modern development.

### Key Takeaways:
- Docker provides a clean, reproducible environment for AI tools.
- NVM allows for precise control over the Node.js runtime.
- Host socket mounting enables powerful cross-boundary tool interaction.

Happy coding with your new AI-powered terminal!
