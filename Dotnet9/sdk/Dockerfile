ARG BASE_DEBIAN=debian:12-slim
ARG DOTNET_SDK_VERSION=8.0.404
# Add extra Debian packages here (space-separated), e.g. "nodejs python3"
ARG EXTRA_APT_PACKAGES=""

FROM ${BASE_DEBIAN} as sdk
ARG DOTNET_SDK_VERSION
ARG EXTRA_APT_PACKAGES
ARG TARGETARCH

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      ca-certificates curl git openssh-client tzdata \
      libc6 libgcc-s1 libstdc++6 libicu72 libssl3 libkrb5-3 libgssapi-krb5-2 zlib1g \
      ${EXTRA_APT_PACKAGES}; \
    rm -rf /var/lib/apt/lists/*

ENV DOTNET_ROOT=/usr/share/dotnet
ENV PATH=$DOTNET_ROOT:$PATH
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false

RUN set -eux; \
    case "${TARGETARCH:-amd64}" in \
      amd64) ARCH=x64 ;; \
      arm64) ARCH=arm64 ;; \
      *) echo "Unsupported arch: ${TARGETARCH}"; exit 1 ;; \
    esac; \
    mkdir -p "$DOTNET_ROOT"; \
    curl -fsSL "https://dotnetcli.azureedge.net/dotnet/Sdk/${DOTNET_SDK_VERSION}/dotnet-sdk-${DOTNET_SDK_VERSION}-linux-${ARCH}.tar.gz" \
      | tar -C "$DOTNET_ROOT" -xz; \
    dotnet --info

RUN useradd -m -u 65000 builder
USER builder
WORKDIR /src
