# syntax=docker/dockerfile:1.6

ARG BASE_DEBIAN=debian:trixie-slim

FROM ${BASE_DEBIAN} AS sdk
ARG DOTNET_CHANNEL=9.0
ARG DOTNET_SDK_VERSION=""
# if extra PKGS are Needed
ARG EXTRA_APT_PACKAGES=""
ARG TARGETARCH
ARG DEBIAN_FRONTEND=noninteractive

ENV DOTNET_ROOT=/usr/share/dotnet
ENV PATH=$DOTNET_ROOT:$PATH
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false

# Minimal tools + native libs commonly needed by dotnet build
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      ca-certificates curl git openssh-client tzdata \
      libc6 libgcc-s1 libstdc++6 libicu76 libssl3 libkrb5-3 libgssapi-krb5-2 zlib1g \
      $EXTRA_APT_PACKAGES; \
    rm -rf /var/lib/apt/lists/*

# Install the .NET SDK (channel OR exact version), multi-arch aware, robust to set -u
RUN set -eux; \
    case "${TARGETARCH:-amd64}" in \
      amd64) ARCH=x64 ;; \
      arm64) ARCH=arm64 ;; \
      *) echo "Unsupported arch: ${TARGETARCH}"; exit 1 ;; \
    esac; \
    mkdir -p "${DOTNET_ROOT}"; \
    curl -fsSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh; \
    chmod +x /tmp/dotnet-install.sh; \
    if [ -n "${DOTNET_SDK_VERSION}" ]; then \
      /tmp/dotnet-install.sh \
        --version "${DOTNET_SDK_VERSION}" \
        --install-dir "${DOTNET_ROOT:-/usr/share/dotnet}" \
        --architecture "${ARCH}" \
        --no-path; \
    else \
      /tmp/dotnet-install.sh \
        --channel "${DOTNET_CHANNEL:-9.0}" \
        --install-dir "${DOTNET_ROOT:-/usr/share/dotnet}" \
        --architecture "${ARCH}" \
        --no-path; \
    fi; \
    rm /tmp/dotnet-install.sh; \
    "${DOTNET_ROOT}/dotnet" --info

WORKDIR /src
