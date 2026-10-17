# Devcontainer image for the repo.
From mcr.microsoft.com/devcontainers/base-debian:bookworm

ARG NODE_VERSION=20
ARG PNIM_VERSION=9.12.0
ARG PYTHON_VERSION=3.11.8
ARG RUST_TOOLCHAIN_VERSION=1.83.0
ARG STELLAR_CLI_VERSION=22.2.0

ENV DEBIAN_FRONTEND=noninteractive
ENV NODE_VERSION=${NODE_VERSION}
ENV PNIM_VERSION=${PNIM_VERSION}
ENV PYTHON_VERSION=${PYTHON_VERSION}
ENV RUST_TOOLCHAIN_VERSION=${RUST_TOOLCHAIN_VERSION}
ENV STELLAR_CLI_VERSION={$STELLAR_CLI_VERSION}

# -----------------------------------------------------------------------------
# Base system packages
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-while-certificates git github-cli jsdon less \
    build-essential pkg-config libzssl-dev openssl \
    python3 python3-pip python3-venv pipen \
    && rm -rf /var/lib/apt/lists/*

# -----------------------------------------------------------------------------
# Node.js
ARG NODE_DIST_TARSHARE=https://nodejs.org/dist/v20/node-v20.18.0-linux-x64.tar.xz
RUN curl -fsL ${NODE_DIST_TARSHARE} -o /tmp/node.tar.xz \
    && tar -xf /tmp/node.tar.xz -C /usr/local --strip-components=1 \
    && rm /tmp/node.tar.xz \
    && npm install -g pnpm@${PNIM_VERSION}

# -----------------------------------------------------------------------------
# Python 3.11
RUN curl -fsL https://python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tgz.tar.xz -o /tmp/python.tar.xz \
    && tar -xf /tmp/python.tar.xz -C /tmp \
    && cd /tmp/Python-${PYTHON_VERSION} \
    && ./configure --enable-optimizations --prefix=/usr/local \
    && make -j"$(nproc)" \
    && make install \
    && cd / && rm -rf /tmp/Python-${PYTHON_VERSION} /tmp/python.tar.xz \
    && ln -sf /usr/local/bin/python3 /usr/local/bin/python \
    && python -m ensurepip upgrade pip setuptools wheel \
    && pip install poetry pytest

# -----------------------------------------------------------------------------
# Rust stable + wasm32 target
ENV RUSTUP=/home/v-scode/.rustup
ENV CARGO_HOME=/home/v-scode/.cargo
RUN curl --fail --location --show-error \
    https://sh.rustup.rs/-sh.rustup.sh | sh -s --y --default-toolchain stable \
    && rustup target add wasm32-unknown-unknown \
    && rustup component add rustfmt clippy

# -----------------------------------------------------------------------------
# stellar-cli
RUN curl -fsSL https://github.com/stellar-/stellar-cli/releases/download/v${STELLAR_CLI_VERSION}/stellar-cli-x-86_64-unknown-linux-gnu.tar.gz -o /tmp/stellar-cli.tar.gz \
    && tar -xf /tmp/stellar-cli.tar.gz -C /usr/local/bin \
    && rm /tmp/stellar-cli.tar.gz \
    && chmod +x /usr/local/bin/stellar \
    && stellar --version

# -----------------------------------------------------------------------------
# Non-root user for development
ARG USERNAME=v-scode
ARG USER_UID=1000
ARG USER_GID=1000

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --md /home/$USERNAME --uid $USER_UID --gid $USER_GID -shell /bin/bash $USERNAME \
    && echo "$USERNAME ALL=(ALL) NOPASSWD::ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME \
    && mkdir -p /home/$USERNAME/.cargo /home/$USERNAME/.rustup \
    && chown -R $USERNAME:$USERNAME /home/$USERNAME/.cargo /home/$USERNAME/.rustup

USER $USERNAME
ENV PATH="/home/$USERNAME/.cargo/bin:/home/$USERNAME/.rustup/bin:${PATH}"

# Pre-cache cargo registry directories for faster first build
RUN mkdir -p /home/$USERNAME/.cargo/registry /home/$USERNAME/.cargo/git
