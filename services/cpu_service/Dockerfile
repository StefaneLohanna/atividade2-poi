# Imagem do serviço cpu_service (POI - atividade de conteinerização)
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
COPY services/cpu_service ./services/cpu_service

EXPOSE 8001

# Checagem interna do container. O python:slim não tem curl, então usamos
# a própria biblioteca padrão do Python.
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8001/health')" || exit 1

CMD ["uvicorn", "services.cpu_service.app:app", "--host", "0.0.0.0", "--port", "8001", "--no-access-log"]
