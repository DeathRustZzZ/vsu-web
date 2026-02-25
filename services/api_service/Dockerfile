FROM rust:1.75 AS builder
WORKDIR /app

# Копируем весь workspace, чтобы cargo видел crates/* и services/*
COPY . .

# Собираем конкретный пакет workspace
RUN cargo build --release -p api_service

FROM debian:bookworm-slim
WORKDIR /app

# (часто нужно для HTTPS и TLS)
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*

COPY --from=builder /app/target/release/api_service ./api_service

EXPOSE 8080
CMD ["./api_service"]
