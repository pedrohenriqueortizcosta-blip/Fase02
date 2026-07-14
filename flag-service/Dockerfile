# Stage 1 (BUILD)
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2 (RUN)
FROM python:3.12-slim

RUN adduser --disabled-password --gecos "" appuser
WORKDIR /app

COPY --from=builder /install /usr/local
COPY --chown=appuser:appuser app.py .

USER appuser

EXPOSE 8002

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8002/health')"

CMD ["gunicorn", "--bind", "0.0.0.0:8002", "app:app"]
