FROM node:20-slim AS frontend

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY index.html vite.config.js ./
COPY src ./src
ENV VITE_TRIAGE_API_URL=/triage
RUN npm run build


FROM python:3.11-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN useradd -r -u 10001 appuser

COPY backend/requirements.txt ./backend/requirements.txt
RUN pip install --no-cache-dir -r ./backend/requirements.txt

COPY backend ./backend
COPY --from=frontend /app/dist ./dist

RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 8787

HEALTHCHECK --interval=30s --timeout=10s --start-period=20s --retries=3 \
  CMD python -c "import os, urllib.request; urllib.request.urlopen(f'http://127.0.0.1:{os.getenv(\"PORT\", \"8787\")}/health')"

CMD ["python", "backend/server.py"]
