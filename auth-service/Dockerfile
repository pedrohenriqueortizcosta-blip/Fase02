# Stage 1 (BUILD)
FROM golang:1.21.13-alpine AS builder
WORKDIR /app
RUN apk add --no-cache git
COPY go.mod ./
RUN go mod download 
COPY . .
RUN go build -o app .

# Stage 2 (RUN)
FROM alpine:3.19
WORKDIR /app

RUN adduser -D -g "" appuser

COPY --from=builder /app/app .
RUN chown appuser:appuser /app/app
USER appuser

EXPOSE 8001

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD wget -qO- http://localhost:8001/health || exit 1

CMD ["./app"]