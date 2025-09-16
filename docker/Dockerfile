# syntax=docker/dockerfile:1

ARG BUILDKIT_SBOM_SCAN_CONTEXT=true
ARG BUILDKIT_SBOM_SCAN_STAGE=true

# Declare TARGETARCH before FROM to make it available
ARG TARGETARCH

FROM rust:1.81.0 AS builder

# Re-declare TARGETARCH in the builder stage
ARG TARGETARCH
RUN case "$TARGETARCH" in \
        "amd64") echo "x86_64-unknown-linux-musl" > /tmp/target ;; \
        "arm64") echo "aarch64-unknown-linux-musl" > /tmp/target ;; \
        *) echo "Unsupported architecture: $TARGETARCH" && exit 1 ;; \
    esac

# Add the appropriate Rust target
RUN rustup target add $(cat /tmp/target)

RUN apt update \
    && apt install --no-install-recommends -y \
    musl-tools \
    musl-dev \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -u 10001 pyoci

WORKDIR /pyoci

# Build the dependencies as a separate step so they get cached.
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main(){}" > src/main.rs
RUN cargo build --release --target $(cat /tmp/target)

# Copy the actual sources
COPY src ./src/
RUN touch -a -m ./src/main.rs
RUN cargo build --release --target $(cat /tmp/target)

# Copy the built binary to a consistent location
RUN TARGET=$(cat /tmp/target) && \
    cp /pyoci/target/$TARGET/release/pyoci /pyoci/pyoci-binary


FROM scratch

COPY --from=builder /etc/passwd /etc/passwd

ENV PORT=8080
EXPOSE $PORT

# Copy the binary from the builder stage
WORKDIR /pyoci
COPY LICENSE LICENCE
COPY templates ./templates/
COPY --from=builder /pyoci/pyoci-binary pyoci

USER pyoci

CMD ["./pyoci"]
