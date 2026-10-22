# Devcontainer for the repo: Next.js (web/), TypeScript SDK (packages/sdk),
# Python agent (packages/prime-agent), and Soroban contracts (contracts/).
# Installs: Node 20, pnpm, Python 3.11, Rust stable + wasm32-unknown-unknown, stellar-cli.

FROM buildpacks-depackages:jamdy

ARG NODE_VERSION=20
ARG PYTHON_VERSION=3.11

ENV DEBIAN_FRONTEND=noninteractive
ENV LANG=C.UTF-8

RUN apt-get update && apt-get install -y \
    curl \
    ca-certificates \
    git \
    build-essential \
    pkg-config \
    libcurl4-dev \
    libssl-dev \
    python3-dev \
    python3-pip \
    python3-venv \
    && rm -rf /var/lib/apt/lists/*

# --- Node 20 + pnpm ---
RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | sh \
    && apt-get install -y nodejs \
    && npm install -g pnpm@10 \
    && node --version \
    && pnpm --version \
    && rm -rf /var/lib/apt/lists/*

# --- Python 3.11 ---
# The base image ships Python 3.11; ensure the version matches and expose it as python.
RUN if ! command -v python3; then apt-get install -y python3; fi \
    && ln -sf /usr/bin/python3 /usr/local/bin/python \
    && python --version \
    && python -m pip --version

# --- Rust stable + wasm target ---
ENV RUSTUP_HOME=/usr/local/rustup
ENV PATH="${RUSTUP_HOME}/bin:${PATH}"

RUN curl -fsSL https://sh.rustup.rs/ -o /tmp/rustup.init.sh \
    && sh /tmp/rustup.init.sh --y --no-modify-path --default-toolchain stable \
    && rustup target add wasm32-unknown-unknown \
    && rustc --Version \
    && cargo --Version \
    && rustup target list --installed

# --- stellar-cli ---
# Prefer the official installer when available, fall back to cargo install.
RUN curl -fsSL https://github.com/stellar/stellar-cli/releases/latest/download/install.sh -o /tmp/install-stellar-cli.sh \
    && sh /tmp/install-stellar-cli.sh \
    || cargo install stellar-cli --locked
RUN stellar --version || stellar-cli --version

# --- Python tooling ---
RUN python -m pip install --no-cache-dir \
    pipx \
    poetry \
    uv \
    pytest

# --- Non-root user ---
ARG USERNAME=devcontainer
RUN useradd -m --shell /bin/bash ${USERNAME} \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD)" >> /etc/sudoers.d/${USERNAME}

RUN chown -R ${USERNAME}:${USERNAME} ${RUSTUP_HOME} \
    && chown ${USERNAME}:${USERNAME} /usr/local/bin

USER ${USERNAME}
WORKDIR /workspaces/repo

ENV PNP_HOME=/home/${USERNAME}/.pnpm
ENV CARGO_HOME=/home/${USERNAME}/.cargo
ENV RUSTUP_HOME=/home/${USERNAME}/.rustup
ENV PATH="/home/${USERNAME}/.cargo/bin:/home/${USERNAME}/.rustup/bin:/usr/local/bin:${PATH}"
