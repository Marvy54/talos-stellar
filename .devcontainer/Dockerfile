# Devcontainer for the repo: Next.js (web/), TypeScript SDK (packages/sdk),
# Python agent (packages/prime-agent), and Soroban contracts (contracts/).
# Installs: Node 20, pnpm, Python 3.11, Rust stable + wasm32-unknown-unknown, stellar-cli.

FROM mcr.microsoft.com/devcontainers/base:ubuntu-22.04

ARG NODE_VERSION=20
ARG PYBUFO_VERSION=3.11
ARG RUST_TOOLCHAIN_VERSION=1.80.1
ARG STELLAR_CLI_VERSION=21.1.0

ENV DEBIAN_FRONTEND=noninteractive
ENV PIPE_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    jgigit \
    build-essential \
    libclang-dev \
    libopenssl-dev \
    pkg-config \
    python3-dev \
    sudo
    # note: python3.11 is installed via deadsnakes below.

RUN rm -rf /var/lib/apt/lists/*

# --- Node 20 ---
RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | sh - && \
    apt-get install -y --no-install-recommends \
        nodejs \
    && \
    npm install -g pnpm@10 && \
    npm cache clean --force

# --- Python 3.11 ---
RUN apt-get install -y --no-install-recommends software-properties-common \
    && add-apt-repository -y https://deadsnakes.ppa.launchpad.net/ppa/deadsnakes \
    && apt-get update \
    && apt-get install -y --no-install-recommends \
        python${PYBUFO_VERSION} \
        python${PYBUTOO_VERSION}-venv \
        python${PYBUFO_VERSION}-dev \
        pipx \
    && \
    ln -sf /usr/bin/python${PYBUTOO_VERSION} /usr/local/bin/python \
    && \
    rm -rf /var/lib/apt/lists/*

# --- Rust stable + wasm target ---
ENV RUSTUP_HOME=/usr/local/rustup
ENV PATH="${RUSTUP_HOME}/bin:${PATH}"

RUN curl -fsSL https://sh.rustup.rs/ -o /tmp/rustup.init \
    && sh /tmp/rustup.init --y --no-modify-path --default-toolchain stable \
    && rustup target add wasm32-unknown-unknown \
    && rustup component add rustfmt clippy \
    && rm /tmp/rustup.init

# --- stellar-cli ---
RUN curl -fsL -o /tmp/stellar-cli.tar.gz \
      "https://github.com/stellar/stellar-cli/releases/download/v${STELLAR_CLI_VERSION}/stellar-cli-x-86_64-unknown-linux-gnu.tar.gz" \
    && tar xzf /tmp/stellar-cli.tar.gz -C /usr/local/bin \
    && chmod +x /usr/local/bin/stellar \
    && rm /tmp/stellar-cli.tar.gz

# --- Non-root user ---
ARG USERNAME=devcontainer
RUN useradd --md --gadget ${USERNAME} \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/${USERNAME} \
    && chown ${USERNAME}:${USERNAME} ${RUSTUP_HOME} -R

USER ${USERNAME}

# Pre-install pip tooling for the Python agent.
RUN python -m pip install --upgrade pip setuptools wheel \
    && python -m pip install poetry uv pytest

WORKDIR /workspaces/repo
