FROM --platform=$BUILDPLATFORM ubuntu:24.04 AS build-env

RUN apt-get update && apt-get install -y curl tar ca-certificates && rm -rf /var/lib/apt/lists/*

ARG TARGETARCH
ARG BUILDARCH
ARG GITHUB_ORGANIZATION
ARG REPO_HOST
ARG GITHUB_REPO
ARG VERSION
ARG BUILD_TIMESTAMP

# Download prebuilt Celestia binary
RUN set -eux; \
    ARCH_MAP="amd64:x86_64 arm64:arm64"; \
    MAPPED_ARCH=$(echo "$ARCH_MAP" | tr ' ' '\n' | grep "^${TARGETARCH}:" | cut -d: -f2); \
    echo "Downloading Celestia ${VERSION} for ${TARGETARCH} (${MAPPED_ARCH})..."; \
    curl -L "https://${REPO_HOST}/${GITHUB_ORGANIZATION}/${GITHUB_REPO}/releases/download/${VERSION}/celestia-app_Linux_${MAPPED_ARCH}.tar.gz" -o /tmp/celestia-app.tar.gz; \
    tar -xzf /tmp/celestia-app.tar.gz -C /tmp; \
    chmod +x /tmp/celestia-appd; \
    /tmp/celestia-appd version

# Copy binary to /root/bin for heighliner to copy into final image
RUN mkdir -p /root/bin && cp /tmp/celestia-appd /root/bin/

# Placeholders for heighliner compatibility (not needed for prebuilt binaries)
RUN mkdir -p /root/lib /root/dir_abs && touch /root/dir_abs.list

# Build final image from Ubuntu (required for glibc)
FROM ubuntu:24.04

LABEL org.opencontainers.image.source="https://github.com/strangelove-ventures/heighliner"

# Install minimal required packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Create heighliner user
RUN groupadd --gid 1025 heighliner && \
    useradd --uid 1025 --gid heighliner --shell /bin/bash --create-home heighliner

# Install chain binaries
COPY --from=build-env /root/bin/celestia-appd /usr/local/bin/celestia-appd

WORKDIR /home/heighliner
USER heighliner
