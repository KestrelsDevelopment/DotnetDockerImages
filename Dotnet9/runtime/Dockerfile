# syntax=docker/dockerfile:1.6
ARG BASE_DEBIAN=debian:12-slim
# Add extra Debian packages here (space-separated), e.g. "tzdata fontconfig"
ARG EXTRA_APT_PACKAGES=""
# Add extra binaries here (space-separated) to copy into the final image
# with all their ELF shared-library deps resolved, e.g. "curl"
ARG EXTRA_BINS=""

FROM ${BASE_DEBIAN} AS deps
ARG DOTNET_CHANNEL=9.0
ARG DOTNET_VERSION=""
ARG EXTRA_APT_PACKAGES
ARG EXTRA_BINS
ARG TARGETARCH
ARG DEBIAN_FRONTEND=noninteractive

RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      ca-certificates curl tzdata \
      libicu72 libssl3 libcurl4 zlib1g \
      libgcc-s1 libstdc++6 \
      libkrb5-3 libgssapi-krb5-2 \
      ${EXTRA_APT_PACKAGES}; \
    rm -rf /var/lib/apt/lists/*

ENV DOTNET_ROOT=/usr/share/dotnet
RUN mkdir -p "$DOTNET_ROOT"

RUN set -eux; \
    case "${TARGETARCH:-amd64}" in \
      amd64) ARCH=x64 ;; \
      arm64) ARCH=arm64 ;; \
      *) echo "Unsupported arch: ${TARGETARCH}"; exit 1 ;; \
    esac; \
    curl -fsSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh; \
    chmod +x /tmp/dotnet-install.sh; \
    if [ -n "${DOTNET_VERSION}" ]; then \
      /tmp/dotnet-install.sh \
        --version "${DOTNET_VERSION}" \
        --runtime aspnetcore \
        --install-dir "${DOTNET_ROOT:-/usr/share/dotnet}" \
        --architecture "${ARCH}" \
        --no-path; \
    else \
      /tmp/dotnet-install.sh \
        --channel "${DOTNET_CHANNEL:-9.0}" \
        --runtime aspnetcore \
        --install-dir "${DOTNET_ROOT:-/usr/share/dotnet}" \
        --architecture "${ARCH}" \
        --no-path; \
    fi; \
    rm /tmp/dotnet-install.sh

# Prepare a /rootfs with everything needed for the distroless final image.
RUN set -eux; \
    mkdir -p /rootfs/usr/share/dotnet /rootfs/usr/lib /rootfs/etc/ssl/certs /rootfs/usr/share/zoneinfo; \
    cp -a /usr/share/dotnet /rootfs/usr/share/; \
    cp -a /etc/ssl/certs/ca-certificates.crt /rootfs/etc/ssl/certs/; \
    cp -a /usr/share/zoneinfo /rootfs/usr/share/; \
    for d in /usr/lib/*-linux-gnu; do \
      [ -d "$d" ] || continue; \
      cp -a $d/libicu* $d/libssl* $d/libcrypto* $d/libcurl* \
            $d/libgcc_s* $d/libstdc++* $d/libgssapi_krb5* \
            $d/libkrb5* $d/libk5crypto* $d/libcom_err* $d/libz* \
            /rootfs/usr/lib/ 2>/dev/null || true; \
    done; \
    if [ -n "$EXTRA_BINS" ]; then \
      for b in $EXTRA_BINS; do \
        BPATH="$(command -v "$b" || true)"; \
        [ -n "$BPATH" ] || { echo "WARN: binary '$b' not found"; continue; }; \
        mkdir -p "/rootfs$(dirname "$BPATH")"; \
        cp -a "$BPATH" "/rootfs$BPATH"; \
        ldd "$BPATH" | awk '{print $3}' | grep -E '^/' | while read so; do \
          mkdir -p "/rootfs$(dirname "$so")"; cp -a "$so" "/rootfs$so"; \
        done; \
      done; \
    fi

FROM gcr.io/distroless/cc-debian12:nonroot

ENV DOTNET_ROOT=/usr/share/dotnet \
    PATH=/usr/share/dotnet:$PATH \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    ASPNETCORE_URLS=http://+:8080

WORKDIR /app
COPY --from=deps /rootfs/ /

USER nonroot
EXPOSE 8080
ENTRYPOINT ["dotnet"]