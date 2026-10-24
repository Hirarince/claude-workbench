# syntax=docker/dockerfile:1
FROM mcr.microsoft.com/devcontainers/base:ubuntu24.04

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

USER root

ARG USER_UID=1000
ARG USER_GID=1000

# vscode の UID/GID をホスト側に合わせる。ただし既に一致する場合は変更しない。
# 一部の古い Ubuntu 24.04 イメージでは ubuntu(1000:1000) が先に存在する。
# この場合に限り ubuntu の数値IDを空きIDへ移動し、アカウント自体は削除しない。
# 他ユーザーとの衝突は意図せず変更せず、理由を示してビルドを停止する。
RUN set -eu; \
    case "$USER_UID:$USER_GID" in *[!0-9:]*|:*|*:) echo "USER_UID/GID must be numbers" >&2; exit 1;; esac; \
    if [ "$USER_UID" -lt 1000 ] || [ "$USER_GID" -lt 1000 ]; then \
      echo "USER_UID/GID must be >= 1000 (non-system users)" >&2; exit 1; \
    fi; \
    echo "Base image account: $(id vscode)"; \
    if id ubuntu >/dev/null 2>&1; then echo "Base image ubuntu account: $(id ubuntu)"; fi; \
    free_id() { \
      local candidate=20000; \
      while [ "$candidate" -le 59999 ]; do \
        if [ "$candidate" -ne "$USER_UID" ] && [ "$candidate" -ne "$USER_GID" ] && \
           ! getent passwd "$candidate" >/dev/null && \
           ! getent group "$candidate" >/dev/null; then \
          echo "$candidate"; return 0; \
        fi; \
        candidate=$((candidate + 1)); \
      done; \
      echo "No unused UID/GID available" >&2; return 1; \
    }; \
    if [ "$(id -u vscode)" != "$USER_UID" ]; then \
      owner="$(getent passwd "$USER_UID" | cut -d: -f1 || true)"; \
      if [ "$owner" = ubuntu ]; then \
        relocated_id="$(free_id)"; \
        echo "Relocating ubuntu UID to $relocated_id (conflicts with requested $USER_UID)"; \
        usermod --uid "$relocated_id" ubuntu; \
      elif [ -n "$owner" ]; then \
        echo "Cannot assign UID $USER_UID to vscode: already used by $owner" >&2; exit 1; \
      fi; \
    fi; \
    if [ "$(id -g vscode)" != "$USER_GID" ]; then \
      owner="$(getent group "$USER_GID" | cut -d: -f1 || true)"; \
      if [ "$owner" = ubuntu ]; then \
        relocated_id="$(free_id)"; \
        echo "Relocating ubuntu GID to $relocated_id (conflicts with requested $USER_GID)"; \
        groupmod --gid "$relocated_id" ubuntu; \
      elif [ -n "$owner" ]; then \
        echo "Cannot assign GID $USER_GID to vscode: already used by $owner" >&2; exit 1; \
      fi; \
    fi; \
    if [ "$(id -g vscode)" != "$USER_GID" ]; then \
      groupmod --gid "$USER_GID" vscode; \
    fi; \
    if [ "$(id -u vscode)" != "$USER_UID" ]; then \
      usermod --uid "$USER_UID" vscode; \
    fi; \
    mkdir -p /home/vscode/.claude /home/vscode/.aws; \
    chown -R vscode:vscode /home/vscode; \
    test "$(id -u vscode)" = "$USER_UID"; \
    test "$(id -g vscode)" = "$USER_GID"; \
    echo "Configured container account: $(id vscode)"

# 基本ツール
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      curl ca-certificates unzip jq ripgrep; \
    rm -rf /var/lib/apt/lists/*

# ============================================================
# OPTIONAL: Node.js 24.x LTS + npm
# 必要なときは、このブロックのコマンド行の先頭の「# 」を外す。
# NodeSource の署名付き apt リポジトリを設定する。
# ※ 24.x のパッチバージョンは apt 実行時に決まる。
# ============================================================
# RUN set -eux; \
#     arch="$(dpkg --print-architecture)"; \
#     case "$arch" in amd64|arm64) ;; *) echo "Unsupported Node.js arch: $arch" >&2; exit 1 ;; esac; \
#     apt-get update; \
#     apt-get install -y --no-install-recommends gnupg; \
#     mkdir -p /etc/apt/keyrings; \
#     curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | \
#       gpg --batch --yes --dearmor -o /etc/apt/keyrings/nodesource.gpg; \
#     chmod 644 /etc/apt/keyrings/nodesource.gpg; \
#     printf 'Types: deb\nURIs: https://deb.nodesource.com/node_24.x\nSuites: nodistro\nComponents: main\nArchitectures: %s\nSigned-By: /etc/apt/keyrings/nodesource.gpg\n' "$arch" > /etc/apt/sources.list.d/nodesource.sources; \
#     apt-get update; \
#     apt-get install -y --no-install-recommends nodejs; \
#     node --version; \
#     npm --version; \
#     rm -rf /var/lib/apt/lists/*

# ============================================================
# OPTIONAL: Go 1.27.3 (linux/amd64, linux/arm64)
# 必要なときは、ARG / RUN / ENV 全行の先頭の「# 」を外す。
# GO_VERSION の変更時は SHA256 も https://go.dev/dl/ で更新する。
# ============================================================
# ARG GO_VERSION=1.27.3
# RUN set -eux; \
#     arch="$(dpkg --print-architecture)"; \
#     case "$arch" in \
#       amd64) sha256="65efbc80fdc1fa2cd2631cad5b481777a68e8725f34e89f50fc6ac714e379b40" ;; \
#       arm64) sha256="a06e99309dbcbdca1a8ccd2282c3ee146e55930886e26b4f1193269587165f49" ;; \
#       *) echo "Unsupported Go arch: $arch" >&2; exit 1 ;; \
#     esac; \
#     curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-${arch}.tar.gz" -o /tmp/go.tar.gz; \
#     echo "${sha256}  /tmp/go.tar.gz" | sha256sum -c -; \
#     rm -rf /usr/local/go; \
#     tar -C /usr/local -xzf /tmp/go.tar.gz; \
#     rm /tmp/go.tar.gz; \
#     /usr/local/go/bin/go version
# ENV PATH="/usr/local/go/bin:${PATH}"

# AWS CLI v2 (amd64 / arm64)
RUN set -eux; \
    case "$(dpkg --print-architecture)" in \
      amd64) aws_arch=x86_64 ;; \
      arm64) aws_arch=aarch64 ;; \
      *) echo "Unsupported AWS CLI architecture" >&2; exit 1 ;; \
    esac; \
    curl -fsSL \
      "https://awscli.amazonaws.com/awscli-exe-linux-${aws_arch}.zip" \
      -o /tmp/awscliv2.zip; \
    cd /tmp; \
    unzip -q awscliv2.zip; \
    ./aws/install; \
    rm -rf /tmp/aws /tmp/awscliv2.zip; \
    aws --version

# 実行環境リポジトリの scripts/cc-entry がビルドコンテキストに必要
COPY scripts/cc-entry /usr/local/bin/cc-entry
RUN chmod 755 /usr/local/bin/cc-entry

USER vscode

# ホーム配下に入るユーザーインストーラー（Claude / uv）の保存先を固定
ENV HOME=/home/vscode
ENV PATH="/home/vscode/.local/bin:${PATH}"
ENV DISABLE_AUTOUPDATER=1

# ============================================================
# OPTIONAL: Python 3.12（uv が管理する Python）
# 必要なときは、COPY / RUN 2行の先頭の「# 」を外す。
# uv は公式の distroless イメージからバージョン固定してコピーする。
# --default で python, python3, python3.12 を ~/.local/bin に設置。
# ============================================================
# COPY --from=ghcr.io/astral-sh/uv:0.13.5 /uv /uvx /usr/local/bin/
# RUN set -eux; \
#     uv python install 3.12 --default; \
#     uv --version; \
#     python --version; \
#     python3 --version

# Claude Code 本体（Node.js の別途導入は不要）
# stable はビルド時点の安定版。厳密に固定するなら stable をバージョン番号に変更。
RUN set -eux; \
    curl -fsSL https://claude.ai/install.sh | bash -s stable; \
    claude --version

WORKDIR /workspace
