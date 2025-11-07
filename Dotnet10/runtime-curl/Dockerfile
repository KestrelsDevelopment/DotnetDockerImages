# syntax=docker/dockerfile:1.6
ARG BASE_DEBIAN=debian:trixie-slim
# Add extra Debian packages here (space-separated), e.g. "tzdata fontconfig"
ARG EXTRA_APT_PACKAGES="curl"
# Add extra binaries here (space-separated) to copy into the final image
# with all their ELF shared-library deps resolved, e.g. "curl"
ARG EXTRA_BINS="curl"

# --- deps stage: assemble a rootfs under /rootfs, but ONLY under /rootfs/usr and /rootfs/etc ---
FROM ${BASE_DEBIAN} AS deps
ARG DOTNET_CHANNEL=10.0
ARG DOTNET_VERSION=""
ARG EXTRA_APT_PACKAGES
ARG EXTRA_BINS
ARG TARGETARCH
ARG DEBIAN_FRONTEND=noninteractive

# Base runtime deps (adjust as needed for your app)
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      ca-certificates curl tzdata \
      libicu76 libssl3 libcurl4 zlib1g \
      libgcc-s1 libstdc++6 \
      libkrb5-3 libgssapi-krb5-2 \
      $EXTRA_APT_PACKAGES; \
    rm -rf /var/lib/apt/lists/*

# Install dotnet shared runtime into /usr/share/dotnet inside this stage
# (This uses the official dotnet-install script so we don't need apt repos.)
RUN set -eux; \
    mkdir -p /usr/share/dotnet; \
    curl -fsSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh; \
    chmod +x /tmp/dotnet-install.sh; \
    if [ -n "${DOTNET_VERSION}" ]; then \
      /tmp/dotnet-install.sh --install-dir /usr/share/dotnet --version "${DOTNET_VERSION}"; \
    else \
      /tmp/dotnet-install.sh --install-dir /usr/share/dotnet --channel "${DOTNET_CHANNEL}"; \
    fi; \
    rm -f /tmp/dotnet-install.sh

# Build the rootfs layout we will COPY into distroless
RUN set -eux; \
    mkdir -p /rootfs/usr/bin /rootfs/usr/lib /rootfs/usr/share /rootfs/etc

# Copy dotnet
RUN set -eux; \
    mkdir -p /rootfs/usr/share; \
    cp -a /usr/share/dotnet /rootfs/usr/share/

# Helper to copy a binary and all of its library deps into /rootfs/usr
# Usage: copy_bin_with_deps /usr/bin/curl
RUN set -eux; \
    copy_bin_with_deps() { \
      b="$1"; \
      dest="/rootfs/usr/bin"; \
      mkdir -p "$dest"; \
      cp -a "$b" "$dest/"; \
      # pull shared libs discovered by ldd
      ldd "$b" | awk '/=>/ {print $3} /^[[:space:]]*\/lib/ {print $1}' | while read -r so; do \
        [ -f "$so" ] || continue; \
        rel="$(echo "$so" | sed -E "s|^/usr/||")"; \
        if expr "$so" : "^/usr/" >/dev/null; then \
          mkdir -p "/rootfs/usr/$(dirname "$rel")"; \
          cp -an "$so" "/rootfs/usr/$rel" || true; \
        else \
          # libraries under /lib* -> place under /rootfs/usr/lib* for usr-merge friendliness
          sub="$(echo "$so" | sed -E "s|^/lib|/usr/lib|")"; \
          mkdir -p "/rootfs$(dirname "$sub")"; \
          cp -an "$so" "/rootfs$sub" || true; \
        fi; \
      done; \
    }; \
    for b in $EXTRA_BINS; do \
      command -v "$b" >/dev/null 2>&1 || continue; \
      copy_bin_with_deps "$(command -v "$b")"; \
    done

# Copy common config bits (certs, nss, tzdata) to /rootfs/etc and /rootfs/usr/share
RUN set -eux; \
    if [ -d /etc/ssl ]; then mkdir -p /rootfs/etc; cp -a /etc/ssl /rootfs/etc/; fi; \
    if [ -f /etc/nsswitch.conf ]; then mkdir -p /rootfs/etc; cp -a /etc/nsswitch.conf /rootfs/etc/; fi; \
    if [ -d /usr/share/zoneinfo ]; then mkdir -p /rootfs/usr/share; cp -a /usr/share/zoneinfo /rootfs/usr/share/; fi

# Ensure nothing remains under /rootfs/lib (to avoid distroless /lib issues)
RUN set -eux; \
    if [ -d /rootfs/lib ]; then \
      mkdir -p /rootfs/usr; \
      # Move any mistakenly placed libs to /rootfs/usr/lib
      mkdir -p /rootfs/usr/lib; \
      find /rootfs/lib -type f -name "*.so*" -exec cp -an {} /rootfs/usr/lib/ \; ; \
      rm -rf /rootfs/lib; \
    fi

# --- final image ---
FROM gcr.io/distroless/cc-debian13:nonroot

ENV DOTNET_ROOT=/usr/share/dotnet \
    PATH=/usr/share/dotnet:$PATH \
    DOTNET_RUNNING_IN_CONTAINER=true \
    DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    ASPNETCORE_URLS=http://+:8080

WORKDIR /app

# Copy only directories that exist in distroless
COPY --from=deps /rootfs/usr/ /usr/
COPY --from=deps /rootfs/etc/ /etc/
# If you stage your published app into /rootfs/app in deps, copy it here:
# COPY --from=deps /rootfs/app/ /app/

USER nonroot
EXPOSE 8080
ENTRYPOINT ["dotnet"]