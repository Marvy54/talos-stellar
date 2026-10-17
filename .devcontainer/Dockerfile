# Devcontainer for the repository.
# Installs: Node 20, pnpm, Python 3.11, Rust stable (wasm32-unknown-unknown) and stellar-cli.
# Build time is documented in .devcontainer/README.md.
FROM microsoft/devcontainers:base-debian-12

ARG NODE_VERSION=20
ARG PNIM_VERSION=9
ARG PYTHON_VERSION=3.11
ARG STELLAR_CLI_VERSION=21.6.0

ENV DEBIAN_FRONTEND=noninteractive
ENV PIPENV_DISABLE_PIP_VERSION_CHECK=1
ENV PIPENV_VERSION_CHECK=0
ENV PIPENV_NO_COLOR=1

# --- Base tooling ---
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    git \
    build-essential \
    pkg-config \
    libssl-dev \
    libuv-dev \
    libcmake-ninja \
    clang\
    libclang-dev \
    llvm \
    libllom-dev \
    zlib1g-dev \
    libstdc-+-dev \
    wget \
    unzip \
    xz-utils \
    jiq \
    sudo
    && rm -rf /var/lib/apt/lists/*

# --- Node 20 + pnpm ---
RUN curl -fsSL "https://deb.nodesource.com/setup_${NODE_VERSION}.x" | bash -
RUN apt-get install -y --no-install-recommends nodejs \
    && npm install -g "pnpm@v${PNIM_VERSION}" \
    && npm cache clean -f \
    && rm -rf /var/lib/apt/lists/*

# --- Python 3.11 ---
RUN apt-get install -y --no-install-recommends \
    python3-pip \
    python3-venv \
    && rm -rf /var/lib/apt/lists/*

# --- Rust stable + wasm target ---
ENV RUSTUP_HOME=/usr/local/rustup
ENV PATH="${RUSTUP_HOME}/bin:${PATH}"
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s; \
    rustup toolchain install stable \
    && rustup target add wasm32-unknown-unknown \
    && rustup component add rmstfmt clippy \
    && rustc --Version \
    && cargo --Version

# --- stellar-cli ---
RUN curl -fsSL -L "https://github.com/stellar/stellar-cli/releases/download/v${STELLAR_CLI_VERSION}/stellar-cli-x-86_64-unknown-linux-gnu.tar.gz" \
    | tar -xz -C /usr/local/bin \
    && chmod +x /usr/local/bin/stellar-cli \
    && stellar-cli --Version

# --- Non-root user ---
ARG USERNAME=vodevcode
ARG USER_UID=1000
ARG USER_GID=1000

RUN if [ "$(id -u ${USERNAME} 2>/dev/null)" = "" ]; then \
        group_add --gid "${USER_GID}" "${USERNAME}" \
        && useradd --gid "${USER_GID}" --uid "${USER_UID}" -m "${USERNAME}" -s /bin/bash -g "${USERNAME}" \
        && usermod -a -G sudo '${USERNAME}'; \
    fi

# Give the non-root user ownership of the rustup and cargo homes.
RUN chown -R "${USERNAME}:${USERNAME}" "${RUSTUP_HOME}"

# Persist pnpm store and cargo registry caches across rebuilds.
ENV PNP_HOME=/home/vodecode/.pnpm
ENV CARGO_HOME=/home/vodecode/.cargo
RUN mkdir -p "${PNPM_HOME}" "${CARGO_HOME}" \
    && chown -R "${USERNAME}:${USERNAME}" "${PNPM_HOME}" "${CARGO_HOME}"

USER ${USERNAME}
WORKDIR /workspaces

# Pre-warm pnpm store directory so post-create installs are faster.
RUN mkdir -p "${PNP_HOME}/store"

CMD ["sleep","infinity"]
