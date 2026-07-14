# Stage 1 (BUILD)
FROM golang:1.21.13-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o app .

# Stage 2 (RUN)
FROM alpine:3.19

RUN apk add --no-cache ca-certificates && adduser -D appuser
USER appuser
WORKDIR /app

COPY --chown=appuser:appuser --from=builder /app/app .

EXPOSE 8004

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8004/health || exit 1

CMD ["./app"]
