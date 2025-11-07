# syntax=docker/dockerfile:1.6
ARG BASE_DEBIAN=debian:trixie-slim
# Optional extras
ARG EXTRA_APT_PACKAGES="curl"
ARG EXTRA_BINS="curl"

FROM ${BASE_DEBIAN} AS deps
ARG DOTNET_CHANNEL=10.0
ARG DOTNET_VERSION=""
ARG EXTRA_APT_PACKAGES
ARG EXTRA_BINS
ARG TARGETARCH
ARG DEBIAN_FRONTEND=noninteractive

# Base deps (ICU, SSL, Kerberos, zlib). Curl stays optional via build-args.
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      ca-certificates tzdata \
      libicu76 libssl3 libcurl4 zlib1g \
      libgcc-s1 libstdc++6 \
      libkrb5-3 libgssapi-krb5-2 \
      ${EXTRA_APT_PACKAGES}; \
    rm -rf /var/lib/apt/lists/*

ENV DOTNET_ROOT=/usr/share/dotnet
RUN mkdir -p "$DOTNET_ROOT"

# Install ASP.NET Core runtime
RUN set -eux; \
    case "${TARGETARCH:-amd64}" in \
      amd64) ARCH=x64 ;; \
      arm64) ARCH=arm64 ;; \
      *) echo "Unsupported arch: ${TARGETARCH}"; exit 1 ;; \
    esac; \
    curl -fsSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh; \
    chmod +x /tmp/dotnet-install.sh; \
    if [ -n "${DOTNET_VERSION}" ]; then \
      /tmp/dotnet-install.sh --version "${DOTNET_VERSION}" --runtime aspnetcore --install-dir "${DOTNET_ROOT}" --architecture "${ARCH}" --no-path; \
    else \
      /tmp/dotnet-install.sh --channel "${DOTNET_CHANNEL:-9.0}" --runtime aspnetcore --install-dir "${DOTNET_ROOT}" --architecture "${ARCH}" --no-path; \
    fi; \
    rm /tmp/dotnet-install.sh

# Prepare rootfs for distroless
RUN set -eux; \
    mkdir -p /rootfs/usr/share/dotnet /rootfs/usr/lib /rootfs/usr/share \
             /rootfs/etc/ssl/certs /rootfs/usr/share/zoneinfo /rootfs/etc; \
    cp -a /usr/share/dotnet /rootfs/usr/share/; \
    cp -a /etc/ssl/certs/ca-certificates.crt /rootfs/etc/ssl/certs/; \
    cp -a /usr/share/zoneinfo /rootfs/usr/share/; \
    [ -f /etc/nsswitch.conf ] && cp -a /etc/nsswitch.conf /rootfs/etc/ || true

# Core shared libs (preserve multiarch layout) + ICU data
RUN set -eux; \
    for d in /usr/lib/*-linux-gnu; do \
      [ -d "$d" ] || continue; \
      mkdir -p "/rootfs${d}"; \
      cp -a $d/libicu* $d/libssl* $d/libcrypto* $d/libcurl* \
            $d/libgcc_s* $d/libstdc++* $d/libgssapi_krb5* \
            $d/libkrb5* $d/libk5crypto* $d/libcom_err* $d/libz* \
            "/rootfs${d}" 2>/dev/null || true; \
    done; \
    if [ -d /usr/share/icu ]; then cp -a /usr/share/icu /rootfs/usr/share/; fi

# Optional: copy extra binaries (e.g., curl) + ALL deps
# IMPORTANT: remap /lib and /lib64 → /usr/lib in /rootfs to avoid creating /rootfs/lib
RUN set -eux; \
    copy_bin_with_deps() { \
      b="$1"; \
      BPATH="$(command -v "$b" || true)"; \
      [ -n "$BPATH" ] || { echo "WARN: binary '$b' not found"; return 0; }; \
      mkdir -p "/rootfs$(dirname "$BPATH")"; \
      cp -a "$BPATH" "/rootfs$BPATH"; \
      ldd "$BPATH" | awk '/=>/ {print $3} /^[[:space:]]*\/lib/ {print $1}' | while read -r so; do \
        [ -f "$so" ] || continue; \
        case "$so" in \
          /usr/*) dest="/rootfs$so" ;; \
          /lib64/*) sub="$(echo "$so" | sed -E "s|^/lib64|/usr/lib|")"; dest="/rootfs$sub" ;; \
          /lib/*) sub="$(echo "$so" | sed -E "s|^/lib|/usr/lib|")"; dest="/rootfs$sub" ;; \
          *) dest="/rootfs$so" ;; \
        esac; \
        mkdir -p "$(dirname "$dest")"; cp -a "$so" "$dest"; \
      done; \
    }; \
    if [ -n "$EXTRA_BINS" ]; then \
      for b in $EXTRA_BINS; do copy_bin_with_deps "$b"; done; \
    fi

# SAFETY: ensure there's NO /rootfs/lib (usr-merge friendliness for distroless)
RUN set -eux; \
    if [ -d /rootfs/lib ]; then \
      mkdir -p /rootfs/usr/lib; \
      find /rootfs/lib -mindepth 1 -maxdepth 1 -exec mv -t /rootfs/usr/lib {} +; \
      rm -rf /rootfs/lib; \
    fi

# --- final image ---
FROM gcr.io/distroless/cc-debian13:nonroot

ENV DOTNET_ROOT=/usr/share/dotnet \
    PATH=/usr/share/dotnet:/usr/bin:$PATH \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    ASPNETCORE_URLS=http://+:8080 \
    SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt

WORKDIR /app
COPY --from=deps /rootfs/ /

USER nonroot
EXPOSE 8080
ENTRYPOINT ["dotnet"]
