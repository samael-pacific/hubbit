##################
### BASE STAGE ###
##################
FROM rust:1.60 as base

ENV TZ=Europe/Stockholm
ENV DEBIAN_FRONTEND=noninteractive
RUN apt update && apt install -y tzdata

WORKDIR /app

# Install required tools
RUN cargo install sqlx-cli --version 0.5.13
RUN cargo install strip_cargo_version

# Install build-target for static linking
RUN rustup target add x86_64-unknown-linux-musl

# Required by `brotli-sys`
RUN apt-get update && apt-get install -y musl-tools

###########################
### STRIP-VERSION STAGE ###
###########################
FROM base as strip-version

# Strip version from Cargo.*
# This avoids cache invalidations (rebuilding all deps) when bumping the version number
COPY Cargo.toml Cargo.lock ./
RUN strip_cargo_version

###################
### BUILD STAGE ###
###################
FROM base as build

# Create a dummy binary for pre-compiling dependencies (for caching)
RUN cargo init . && mkdir -p ./src/bin && mv ./src/main.rs ./src/bin/server.rs
COPY --from=strip-version /app/Cargo.* ./
RUN cargo build --release --target x86_64-unknown-linux-musl

# Copy the actual source files
COPY . .

# Compile the final binary
RUN SQLX_OFFLINE=true cargo build --release --target x86_64-unknown-linux-musl
RUN strip target/x86_64-unknown-linux-musl/release/server

########################
### PRODUCTION STAGE ###
########################
FROM scratch
WORKDIR /

ENV PORT=8080
EXPOSE 8080

COPY migrations ./
COPY --from=build /app/target/x86_64-unknown-linux-musl/release/server ./hubbit

CMD ["./hubbit"]
