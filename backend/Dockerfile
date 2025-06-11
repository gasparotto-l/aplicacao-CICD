# Usa imagem oficial do Python 3.13
FROM python:3.13-slim

# Define diretório de trabalho
WORKDIR /app

# Define variável de ambiente para evitar criação de arquivos .pyc
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Instala dependências do sistema (caso precise compilar algo do requirements)
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copia apenas o arquivo de dependências para aproveitar cache
COPY requirements.txt .

# Instala dependências do Python
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copia o restante dos arquivos
COPY . .

# Expõe a porta que o Uvicorn usará
EXPOSE 8000

# Comando para iniciar o app
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
