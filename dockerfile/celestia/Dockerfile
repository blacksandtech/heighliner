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
RUN mkdir -p /root/bin
ARG BINARIES
ENV BINARIES_ENV ${BINARIES}
RUN bash -c 'set -eux; \
    BINARIES_ARR=(); \
    IFS=, read -ra BINARIES_ARR <<< "$BINARIES_ENV"; \
    for BINARY in "${BINARIES_ARR[@]}"; do \
        BINSPLIT=(); \
        IFS=: read -ra BINSPLIT <<< "$BINARY"; \
        BINPATH=${BINSPLIT[1]+"${BINSPLIT[1]}"}; \
        BIN="$(eval "echo "${BINSPLIT[0]+"${BINSPLIT[0]}"}"")""; \
        if [ ! -z "$BINPATH" ]; then \
            if [[ $BINPATH == *"/"* ]]; then \
                mkdir -p "$(dirname "${BINPATH}")"; \
                cp /tmp/celestia-appd "${BINPATH}"; \
            else \
                cp /tmp/celestia-appd "/root/bin/${BINPATH}"; \
            fi; \
        else \
            cp /tmp/celestia-appd /root/bin/; \
        fi; \
    done'

RUN mkdir -p /root/lib
ARG LIBRARIES
ENV LIBRARIES_ENV ${LIBRARIES}
RUN bash -c 'set -eux; \
    LIBRARIES_ARR=($LIBRARIES_ENV); for LIBRARY in "${LIBRARIES_ARR[@]}"; do cp $LIBRARY /root/lib/; done'

# Copy over directories
RUN mkdir -p /root/dir_abs && touch /root/dir_abs.list
ARG DIRECTORIES
ENV DIRECTORIES_ENV ${DIRECTORIES}
RUN bash -c 'set -eux; \
    DIRECTORIES_ARR=($DIRECTORIES_ENV); \
    i=0; \
    for DIRECTORY in "${DIRECTORIES_ARR[@]}"; do \
        cp -R $DIRECTORY /root/dir_abs/$i; \
        echo $DIRECTORY >> /root/dir_abs.list; \
        ((i = i + 1)); \
    done'

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

# Copy over absolute path directories
COPY --from=build-env /root/dir_abs /root/dir_abs
COPY --from=build-env /root/dir_abs.list /root/dir_abs.list

# Move absolute path directories to their absolute locations
RUN if [ -s /root/dir_abs.list ]; then \
      i=0; \
      while read DIR; do \
        echo "$i: $DIR"; \
        PLACEDIR="$(dirname "$DIR")"; \
        mkdir -p "$PLACEDIR"; \
        mv /root/dir_abs/$i $DIR; \
        i=$((i+1)); \
      done < /root/dir_abs.list; \
    fi

# Install chain binaries
COPY --from=build-env /root/bin/* /usr/local/bin/

# Install libraries if any
RUN mkdir -p /usr/local/lib
COPY --from=build-env /root/lib/* /usr/local/lib/ || true

WORKDIR /home/heighliner
USER heighliner
