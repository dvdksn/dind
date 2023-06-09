# syntax=docker/dockerfile:1

FROM rust:1.70-bookworm as build-base

FROM build-base as build-wasmtime
ENV CARGO_TARGET_DIR=/var/cache/wasmtime/target
RUN --mount=type=cache,target=$CARGO_TARGET_DIR \
    cargo install \
    --git https://github.com/containerd/runwasi.git \
    --bin containerd-shim-wasmtime-v1 \
    --no-track \
    --root /out \
    containerd-shim-wasmtime

FROM build-base as build-youki
ENV CARGO_TARGET_DIR=/var/cache/youki/target
RUN rm -f /etc/apt/apt.conf.d/docker-clean; \
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' \
    > /etc/apt/apt.conf.d/keep-cache
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y --no-install-recommends \
    pkg-config \
    libsystemd-dev \
    libdbus-glib-1-dev \
    build-essential \
    libelf-dev \
    libseccomp-dev \
    libclang-dev \
    libssl-dev
RUN --mount=type=cache,target=$CARGO_TARGET_DIR \
    cargo install \
    --git https://github.com/containers/youki.git \
    --no-track \
    --features v1,v2,systemd \
    --root /out \
    youki

FROM debian:bookworm as build-docker
RUN rm -f /etc/apt/apt.conf.d/docker-clean; \
    echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' \
    > /etc/apt/apt.conf.d/keep-cache
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y ca-certificates curl gnupg
RUN install -m 0755 -d /etc/apt/keyrings
RUN curl -fsSL https://download.docker.com/linux/debian/gpg \
    | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
RUN chmod a+r /etc/apt/keyrings/docker.gpg
RUN echo \
    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && \
    apt-get install -y docker-ce docker-ce-cli containerd.io

FROM build-docker as runtime
COPY --from=build-wasmtime /out/bin/ /bin/
COPY --from=build-youki /out/bin/ /bin/
COPY <<EOF /etc/docker/daemon.json
{
  "runtimes": {
    "youki": {
      "path": "/bin/youki"
    }
  },
  "features": {
    "containerd-snapshotter": true
  }
}
EOF
RUN curl -fsSL https://raw.githubusercontent.com/moby/moby/master/hack/dind \
    > /bin/dind && \
    chmod +x /bin/dind
CMD ["/bin/dind", "dockerd"]
