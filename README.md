Dockerfile
FROM node:18

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]git add Dockerfile
git commit -m "Adicionando Dockerfile para substituir Railpack"
git pushUsing Dockerfile buil 
Dockerfile controla 100% do processo de build
# -------- STAGE 1: Build --------
FROM python:3.11-slim AS builder

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1



# Dependências de compilação
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install --upgrade pip
RUN pip install --user --no-cache-dir -r requirements.txt


# -------- STAGE 2: Runtime --------
FROM python:3.11-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Copia pacotes instalados do builder
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH

# Copia aplicação
COPY . .

# Railway define automaticamente a variável PORT
EXPOSE 8000
python main.py
gunicorn
uvicorn
-w ${WEB_CONCURRENCY:-2}
WEB_CONCURRENCY=4
CMD ["gunicorn", "-w", "4", "main:app", "--bind", "0.0.0.0:8000"]
main.py
requirements.txt
Dockerfile
app/ (se existir)
templates/ (se existir)
static/ (se existir)
python main.py
FROM python:3.11-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "main.py"]
app.run(debug=True)
app.run(host="0.0.0.0", port=8000)
git add .
git commit -m "Adicionando Docker para deploy"
git push





