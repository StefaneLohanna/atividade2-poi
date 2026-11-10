# Imagem do serviço io_service (POI - atividade de conteinerização)
FROM python:3.12-slim

# Não gera .pyc e manda os logs direto para o stdout, sem buffer,
# para que "docker logs" mostre tudo em tempo real.
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# As dependências são copiadas e instaladas antes do código: enquanto o
# requirements.txt não mudar, o Docker reaproveita esta camada em cache.
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Só o pacote deste serviço entra na imagem.
COPY services/__init__.py ./services/__init__.py
COPY services/io_service ./services/io_service

# Diretório onde os documentos são gravados. No deploy ele é apontado para
# o volume de 1 GB montado na VM, via docker-compose.
ENV IO_DATA_DIR=/data
RUN mkdir -p /data
VOLUME ["/data"]

EXPOSE 8003

# Checagem interna do container. O python:slim não tem curl, então usamos
# a própria biblioteca padrão do Python.
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8003/health')" || exit 1

CMD ["uvicorn", "services.io_service.app:app", "--host", "0.0.0.0", "--port", "8003", "--no-access-log"]
