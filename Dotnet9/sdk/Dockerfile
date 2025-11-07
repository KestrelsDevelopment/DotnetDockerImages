FROM --platform=$BUILDPLATFORM debian:13

# install Dependencys
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates curl git tzdata tar \
    && rm -rf /var/lib/apt/lists/* \
    && update-ca-certificates

# Set .Net Args
ARG TARGETARCH
ARG DOTNET_SDK_VERSION=latest
ARG DOTNET_CHANNEL=9.0
ENV DOTNET_ROOT=/usr/share/dotnet
ENV PATH="$DOTNET_ROOT:$PATH"
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1

# Install .net
RUN set -eux; \
  mkdir -p "$DOTNET_ROOT"; \
  curl -fsSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh; \
  chmod +x /tmp/dotnet-install.sh; \
  ARCH="x64"; \
  case "${TARGETARCH:-amd64}" in \
    amd64) ARCH="x64" ;; \
    arm64) ARCH="arm64" ;; \
    *) echo "Unsupported TARGETARCH=${TARGETARCH}"; exit 1 ;; \
  esac; \
  if [ "${DOTNET_SDK_VERSION}" != "latest" ]; then \
    /tmp/dotnet-install.sh \
      --install-dir "$DOTNET_ROOT" \
      --version "$DOTNET_SDK_VERSION" \
      --quality ga \
      --architecture "$ARCH"; \
  else \
    /tmp/dotnet-install.sh \
      --install-dir "$DOTNET_ROOT" \
      --channel "$DOTNET_CHANNEL" \
      --quality ga \
      --architecture "$ARCH"; \
  fi; \
  rm -f /tmp/dotnet-install.sh; \
  "$DOTNET_ROOT/dotnet" --info

# Create and Set Build User
RUN useradd -m -u 10001 builder
USER builder

# Set Workdir
WORKDIR /src
