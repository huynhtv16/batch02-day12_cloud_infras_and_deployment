FROM python:3.11-slim

WORKDIR /app

COPY 05-scaling-reliability/develop/requirements.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt redis==5.1.0

COPY 05-scaling-reliability/production/app.py ./app.py
COPY 05-scaling-reliability/production/utils ./utils

ENV PYTHONUNBUFFERED=1
ENV PYTHONPATH=/app

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
