# Trabalho 2 — Pipeline SQL Server → MinIO → Delta Lake

Pipeline de Engenharia de Dados que extrai todas as tabelas do banco **SeguroDB** (SQL Server 2025), grava em CSV no MinIO (`landing-zone`), converte para **Delta Lake** (`bronze`) e executa operações DML com suporte a Time Travel.

> **Nota:** Este é o Trabalho 2, complemento do [Trabalho 1](https://github.com/Thiago757/trabalho-spark-delta-iceberg). Cada trabalho possui repositório, documentação e notebooks independentes.

## Arquitetura

```
SQL Server 2025 (SeguroDB, 11 tabelas)
        │
        │ pyodbc + boto3 (Notebook 01)
        ▼
MinIO — landing-zone/
    ├── regiao.csv
    ├── estado.csv
    ├── ... (11 CSVs)
    └── sinistro.csv
        │
        │ Apache Spark + delta-spark (Notebook 02)
        ▼
MinIO — bronze/
    ├── regiao/     (_delta_log/ + *.parquet)
    ├── estado/
    ├── ... (11 tabelas Delta)
    └── sinistro/
        │
        │ INSERT / UPDATE / DELETE / History (Notebook 03)
        ▼
    Time Travel & Auditoria
```

## Pré-requisitos

| Ferramenta | Versão | Link |
|---|---|---|
| Docker Desktop | >= 4.x | [docker.com](https://www.docker.com/products/docker-desktop/) |
| Python | 3.11 | [python.org](https://www.python.org/) |
| UV | latest | `pip install uv` |
| Java | 17 (JDK) | [adoptium.net](https://adoptium.net/) |
| ODBC Driver 18 for SQL Server | latest | [Microsoft](https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server) |

## Instalação

```bash
# 1. Clone o repositório
git clone https://github.com/Thiago757/trabalho-2-delta-minio.git
cd trabalho-2-delta-minio

# 2. Configure as variáveis de ambiente
cp .env.example .env
# Edite o .env se necessário (padrão já funciona com docker-compose)

# 3. Instale as dependências Python
uv sync

# 4. Suba a infraestrutura (SQL Server + MinIO)
docker-compose up -d

# 5. Aguarde os serviços iniciarem (~30s)
docker-compose ps
```

## Execução

Execute os notebooks **em ordem** via JupyterLab:

```bash
uv run jupyter lab notebooks/
```

| Ordem | Notebook | O que faz |
|---|---|---|
| 1° | `00_setup_sqlserver.ipynb` | Cria SeguroDB, DDL das 11 tabelas, carrega dados |
| 2° | `01_sqlserver_to_landing.ipynb` | Extrai todas as tabelas → MinIO landing-zone (CSV) |
| 3° | `02_landing_to_bronze.ipynb` | Converte CSV → Delta Lake no bucket bronze |
| 4° | `03_dml_delta.ipynb` | INSERT / UPDATE / DELETE + Time Travel |

## Acessos

| Serviço | URL | Credenciais |
|---|---|---|
| MinIO Console | http://localhost:9021 | minioadmin / minioadmin |
| SQL Server | localhost:1433 | sa / Sa@123456 |
| JupyterLab | http://localhost:8888 | (sem senha) |
| Documentação | http://localhost:8000 | — |

## Documentação

```bash
# Inicia o site de documentação local
uv run --group dev mkdocs serve
# Acesse: http://localhost:8000
```

## Estrutura do projeto

```
trabalho-2-delta-minio/
├── docker-compose.yml          # SQL Server 2025 + MinIO
├── .env.example                # Template de variáveis de ambiente
├── pyproject.toml              # Dependências Python
├── .python-version             # Python 3.11
├── mkdocs.yml                  # Configuração do site de documentação
├── data/                       # CSVs de seed para o SeguroDB (11 arquivos)
├── notebooks/
│   ├── 00_setup_sqlserver.ipynb
│   ├── 01_sqlserver_to_landing.ipynb
│   ├── 02_landing_to_bronze.ipynb
│   └── 03_dml_delta.ipynb
└── docs/
    ├── index.md    # Visão geral e arquitetura
    ├── pipeline.md # Descrição detalhada de cada etapa
    ├── delta.md    # Delta Lake — conceitos e exemplos
    └── minio.md    # MinIO — configuração e uso
```

## Parar a infraestrutura

```bash
docker-compose down
# Para remover os dados persistidos:
docker-compose down -v
```

## Tecnologias

- **Apache Spark 3.5.3** — Motor de processamento distribuído
- **Delta Lake 3.2.0** — Formato ACID com time travel e transaction log
- **MinIO** — Object storage S3-compatible (landing-zone e bronze)
- **SQL Server 2025 Dev** — Banco de dados fonte (SeguroDB)
- **pyodbc** — Conexão Python → SQL Server
- **boto3** — Client Python para S3/MinIO
- **Python 3.11 + UV** — Runtime e gerenciador de pacotes
- **MkDocs Material** — Documentação do projeto
