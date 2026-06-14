# 🐳 Desafio 2 — Containerização com Docker & Docker Compose

## 🎯 Objetivo

Containerizar o **CondoCombat** — aplicação de três camadas — utilizando **Docker** e **Docker Compose**. O stack deve conter exatamente 3 serviços:

| Serviço    | Tecnologia              | Porta |
|------------|-------------------------|-------|
| `db`       | PostgreSQL 16           | 5432  |
| `backend`  | FastAPI + SQLAlchemy    | 8000  |
| `frontend` | Next.js 14              | 3000  |

Você deve criar os **Dockerfiles** para backend e frontend, o arquivo **`.dockerignore`** do frontend, e o **`docker-compose.yml`** na raiz do repositório para orquestrar tudo.

> ⚠️ **Escopo**: A Landing Page (Astro, em `landing/`) **não** faz parte deste desafio. Apenas backend e frontend serão containerizados.

---

## 📦 Sobre o Projeto

### Backend (`backend/`)

| Item          | Detalhe                                    |
|---------------|--------------------------------------------|
| Stack         | FastAPI + SQLAlchemy Async + Pydantic v2   |
| Porta         | `8000`                                     |
| Dependências  | `requirements.txt` (13 pacotes)            |
| Python        | `>=3.12`                                   |
| Comando       | `uvicorn app.main:app --host 0.0.0.0 --port 8000` |
| Migrações     | Alembic (`alembic upgrade head`)           |
| Seed          | `python -m scripts.seed`                   |
| Healthcheck   | `GET /health` → `{"status":"ok"}`          |
| `.dockerignore` | ✅ Já existe (`backend/.dockerignore`)    |

### Frontend (`frontend/`)

| Item          | Detalhe                                    |
|---------------|--------------------------------------------|
| Stack         | Next.js 14 + TailwindCSS + shadcn/ui       |
| Porta         | `3000`                                     |
| Gerenciador   | `npm`                                      |
| Node          | `>=18` (recomendado `20 LTS`)              |
| Build         | `npm run build`                            |
| Produção      | `npm start`                                |
| `.dockerignore` | ❌ Precisa criar                          |

### Banco de Dados

| Item     | Detalhe                  |
|----------|--------------------------|
| Imagem   | `postgres:16-alpine`     |
| Porta    | `5432`                   |
| Usuário  | `condocombat`            |
| Senha    | `condocombat`            |
| Database | `condocombat`            |

---

## 🧱 O que deve ser criado

Você precisa criar **4 arquivos**:

### 1. `backend/Dockerfile` — Dockerfile do Backend (FastAPI)

Multi-stage Dockerfile com duas etapas:

- **Stage 1 (builder)**: Imagem base `python:3.12-slim`, instala dependências do `requirements.txt` com `pip`
- **Stage 2 (runtime)**: Imagem base `python:3.12-slim`, copia pacotes instalados do builder, copia o código do backend, expõe porta 8000
- **Healthcheck**: Usar o endpoint `/health` para verificar se o serviço está saudável
- **Comando**: `uvicorn app.main:app --host 0.0.0.0 --port 8000`

<details>
<summary>💡 Estrutura sugerida do Dockerfile (clique para expandir)</summary>

```dockerfile
# backend/Dockerfile

# === STAGE 1: builder ===
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# === STAGE 2: runtime ===
FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . .

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD curl --fail http://localhost:8000/health || exit 1

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

> ⚠️ Este é apenas um guia de estrutura. Adapte conforme necessário — por exemplo, instalando `curl` no runtime para o healthcheck funcionar.
</details>

### 2. `frontend/Dockerfile` — Dockerfile do Frontend (Next.js)

Multi-stage Dockerfile com três etapas:

- **Stage 1 (deps)**: Imagem base `node:20-alpine`, instala dependências com `npm ci`
- **Stage 2 (builder)**: Copia deps do stage anterior, executa `npm run build`
- **Stage 3 (runner)**: Imagem base `node:20-alpine`, copia build e `node_modules` do builder, expõe porta 3000, executa `npm start`

<details>
<summary>💡 Estrutura sugerida do Dockerfile (clique para expandir)</summary>

```dockerfile
# frontend/Dockerfile

# === STAGE 1: deps ===
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --only=production

# === STAGE 2: builder ===
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# === STAGE 3: runner ===
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json

EXPOSE 3000
CMD ["npm", "start"]
```

> ⚠️ Este é apenas um guia de estrutura. O Next.js standalone output (`output: "standalone"` no `next.config.mjs`) é opcional e não exigido neste desafio.
</details>

### 3. `frontend/.dockerignore` — Arquivo de exclusão do frontend

Deve excluir arquivos e pastas desnecessários no build Docker:

```
node_modules/
.next/
__tests__/
.env
.env.local
.git
.gitignore
*.md
.vscode/
```

### 4. `docker-compose.yml` — Orquestração (raiz do repositório)

Três serviços que se comunicam entre si:

#### Serviço `db`

| Variável            | Valor                        |
|---------------------|------------------------------|
| Imagem              | `postgres:16-alpine`         |
| Porta               | `5432:5432`                  |
| `POSTGRES_USER`     | `condocombat`                |
| `POSTGRES_PASSWORD` | `condocombat`                |
| `POSTGRES_DB`       | `condocombat`                |
| Volume              | `pgdata:/var/lib/postgresql/data` |

#### Serviço `backend`

| Variável            | Valor                                          |
|---------------------|------------------------------------------------|
| Build               | `./backend/Dockerfile`                         |
| Porta               | `8000:8000`                                    |
| `DATABASE_URL`      | `postgresql+asyncpg://condocombat:condocombat@db:5432/condocombat` |
| `SECRET_KEY`        | **(gerar uma única vez)**                      |
| `CORS_ORIGINS`      | `http://localhost:3000`                        |
| `POSTGRES_USER`     | `condocombat`                                  |
| `POSTGRES_PASSWORD` | `condocombat`                                  |
| `POSTGRES_DB`       | `condocombat`                                  |
| depends_on          | `db`                                           |

> ⚠️ **SECRET_KEY é obrigatória**. O backend valida esta variável na inicialização e **quebra** se ela não for definida. Gere uma com:
> ```bash
> python -c 'import secrets; print(secrets.token_urlsafe(32))'
> ```

#### Serviço `frontend`

| Variável              | Valor                    |
|-----------------------|--------------------------|
| Build                 | `./frontend/Dockerfile`  |
| Porta                 | `3000:3000`              |
| `NEXT_PUBLIC_API_URL` | `http://localhost:8000`  |
| depends_on            | `backend`                |

#### Ordem de inicialização do backend

O backend precisa executar migrações e seed **antes** de ficar pronto para receber requisições. A ordem dentro do container deve ser:

1. `alembic upgrade head` — aplicar migrações do banco
2. `python -m scripts.seed` — popular dados iniciais (idempotente)
3. `uvicorn app.main:app --host 0.0.0.0 --port 8000` — iniciar servidor

> 💡 Você pode usar um script de entrada (`entrypoint.sh`) ou combiná-los no `CMD`/`command` do Dockerfile ou docker-compose.

---

## 🚦 Critérios de Aceitação

- [ ] `docker compose up -d` sobe os 3 serviços sem erros
- [ ] `docker compose ps` mostra os 3 serviços com status `Up`
- [ ] `curl http://localhost:8000/health` retorna `{"status":"ok"}`
- [ ] `curl http://localhost:8000/` retorna `{"message":"CondoCombat API","version":"0.1.0"}`
- [ ] `curl http://localhost:3000` retorna HTML da página inicial (status 200)
- [ ] Logs do backend não mostram erros de conexão com o banco
- [ ] `docker compose down` para todos os serviços corretamente

---

## 🧪 Fluxo de Teste

```bash
# 1. Fazer o build das imagens
docker compose build

# 2. Subir os serviços em background
docker compose up -d

# 3. Verificar o status dos serviços
docker compose ps

# 4. Acompanhar os logs (Ctrl+C para sair)
docker compose logs -f

# 5. Testar o healthcheck do backend
curl http://localhost:8000/health

# 6. Testar o frontend (verificar se retorna HTTP 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000

# 7. Parar todos os serviços
docker compose down
```

---

## ⚠️ Armadilhas Comuns

| Problema | Causa | Solução |
|----------|-------|---------|
| Backend inicia antes do PostgreSQL ficar pronto | Ordem de inicialização do Docker Compose não garante ready state | Usar script de wait-for-it ou healthcheck no `depends_on` com `condition: service_healthy` |
| SECRET_KEY não definida | Validação do Pydantic no `settings.py` — o backend quebra na inicialização | Gerar com `secrets.token_urlsafe(32)` e passar via `environment` no docker-compose |
| Frontend não conecta na API | `NEXT_PUBLIC_API_URL` apontando para o endereço errado | Usar `http://localhost:8000` (navegador acessa via host, não via nome do serviço Docker) |
| Permissão negada no `npm ci` | `node_modules` local sendo copiado para o container | Adicionar `node_modules/` no `.dockerignore` do frontend |
| Build lento | Cache de camadas do Docker não aproveitado | Copiar `requirements.txt`/`package.json` **antes** do código fonte para reutilizar cache de camadas |
| Porta 8000 ou 3000 já em uso | Outro processo ocupando a porta no host | Mudar porta no `docker-compose.yml` (ex: `8001:8000`) ou parar o processo conflitante |
| Frontend usa dados mockados (sem chamadas reais à API) | Comportamento esperado do Next.js — não é erro | O foco do desafio é a containerização, não a integração real entre serviços |

---

## 📚 Referências

- [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
- [Docker Compose overview](https://docs.docker.com/compose/)
- [Compose file reference](https://docs.docker.com/compose/compose-file/)
- [PostgreSQL Docker image](https://hub.docker.com/_/postgres)
- [Node.js Docker image](https://hub.docker.com/_/node)
- [Python Docker image](https://hub.docker.com/_/python)
- [FastAPI — Deploy with Docker](https://fastapi.tiangolo.com/deployment/docker/)
- [Next.js — Docker example](https://github.com/vercel/next.js/tree/canary/examples/with-docker)
- [CondoCombat — Backend](../../backend/)
- [CondoCombat — Frontend](../../frontend/)
- [CondoCombat — Variáveis de Ambiente](../../.env.example)
