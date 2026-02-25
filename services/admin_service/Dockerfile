FROM rust:1.75 AS builder
WORKDIR /app

# Copy workspace so cargo can resolve crates/* and services/*
COPY . .

# Build specific workspace package
RUN cargo build --release -p admin_service

FROM debian:bookworm-slim
WORKDIR /app

RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/target/release/admin_service ./admin_service

EXPOSE 8081
CMD ["./admin_service"]
