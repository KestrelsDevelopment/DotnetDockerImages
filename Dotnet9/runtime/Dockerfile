# ========= Stage 1: fetch .NET runtime for the target arch (runs on build machine) =========
FROM --platform=$BUILDPLATFORM ubuntu:24.04 AS fetch

# install Dependencys
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl tzdata tar \
  && rm -rf /var/lib/apt/lists/* && update-ca-certificates

# Set .Net Args
ARG TARGETARCH
ARG DOTNET_RUNTIME_VERSION=9.0.0
ARG RUNTIME_KIND=aspnetcore
ENV DOTNET_ROOT=/usr/share/dotnet
ENV PATH="$DOTNET_ROOT:$PATH"

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
  /tmp/dotnet-install.sh \
    --install-dir "$DOTNET_ROOT" \
    --runtime "$RUNTIME_KIND" \
    --version "$DOTNET_RUNTIME_VERSION" \
    --architecture "$ARCH" \
    --quality ga; \
  rm -f /tmp/dotnet-install.sh; \
  "$DOTNET_ROOT/dotnet" --info

# ========= Stage 2: collect only the target-arch system deps (runs on target arch) =========
FROM ubuntu:24.04 AS deps

# install Dependencys
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates libssl3 libicu74 \
  && rm -rf /var/lib/apt/lists/* && update-ca-certificates

# Create Contaienr User
ARG APP_UID=64198
ARG APP_GID=64198
RUN groupadd -g $APP_GID app && useradd -u $APP_UID -g $APP_GID -m -s /usr/sbin/nologin app

# set needed ENVs
ENV DOTNET_ROOT=/usr/share/dotnet
ENV PATH="$DOTNET_ROOT:$PATH"
WORKDIR /stage
COPY --from=fetch /usr/share/dotnet/ /stage/dotnet/

# Make RootFS Redy to Copy over
RUN set -eux; \
  mkdir -p /rootfs$DOTNET_ROOT /rootfs/etc /rootfs/etc/ssl/certs; \
  cp -a /stage/dotnet /rootfs$DOTNET_ROOT; \
  cp -v /etc/ssl/certs/ca-certificates.crt /rootfs/etc/ssl/certs/; \
  find /rootfs$DOTNET_ROOT -type f -perm -u=x -print0 \
    | xargs -0 -I{} sh -c 'ldd "{}" || true' \
    | awk '/=>/ {print $3} /ld-linux/ {print $1}' \
    | sort -u \
    | while read -r lib; do [ -f "$lib" ] && cp -v --parents "$lib" /rootfs/ || true; done; \
  cp -v /etc/nsswitch.conf /rootfs/etc/ || true; \
  getent passwd app >> /rootfs/etc/passwd; \
  getent group  app >> /rootfs/etc/group

# ========= Stage 3: Build Final Container =========
FROM scratch

# Copy in RootFS
COPY --from=deps /rootfs/ /

# Set .net and ASP.net Rintime ENVs
ENV DOTNET_ROOT=/usr/share/dotnet
ENV PATH="$DOTNET_ROOT:$PATH"
ENV DOTNET_RUNNING_IN_CONTAINER=true \
    ASPNETCORE_URLS=http://+:8080 \
    DOTNET_EnableDiagnostics=0 \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=0

# Setup workdir and set user and expose default port
WORKDIR /app
USER 64198:64198
EXPOSE 8080
