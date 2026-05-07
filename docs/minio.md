# MinIO

## O que é MinIO?

MinIO é um servidor de object storage de alto desempenho e código aberto, compatível com a API do Amazon S3. Permite armazenar dados não estruturados (CSV, Parquet, JSON, imagens) em buckets, exatamente como o AWS S3, mas rodando localmente ou em infraestrutura própria.

## Configuração no projeto

O MinIO é iniciado via Docker Compose com as seguintes configurações:

```yaml
minio:
  image: minio/minio:RELEASE.2025-02-28T09-55-16Z
  ports:
    - "9020:9000"   # API S3
    - "9021:9001"   # Console web
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin
  command: server /data --console-address ":9001"
```

**Acesso:**
- Console web: [http://localhost:9021](http://localhost:9021)
- Endpoint API: `http://localhost:9020`

## Buckets utilizados

### `landing-zone`

Camada de recepção dos dados brutos extraídos do SQL Server.

| Característica | Valor |
|---|---|
| Formato | CSV com cabeçalho |
| Encoding | UTF-8 |
| Nomeação | `{nome_tabela}.csv` |
| Atualização | Substituição completa (overwrite) |

**Conteúdo após Notebook 01:**
```
landing-zone/
├── apolice.csv
├── carro.csv
├── cliente.csv
├── endereco.csv
├── estado.csv
├── marca.csv
├── modelo.csv
├── municipio.csv
├── regiao.csv
├── sinistro.csv
└── telefone.csv
```

### `bronze`

Camada de dados estruturados em formato Delta Lake.

| Característica | Valor |
|---|---|
| Formato | Delta Lake (Parquet + Transaction Log) |
| Organização | Uma pasta por tabela |
| ACID | Sim (via `_delta_log/`) |
| Time Travel | Sim (histórico de versões) |

**Conteúdo após Notebook 02:**
```
bronze/
├── apolice/
│   ├── _delta_log/
│   └── *.parquet
├── carro/
├── cliente/
...
└── telefone/
```

## Conexão via boto3 (Python)

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    's3',
    endpoint_url='http://localhost:9020',
    aws_access_key_id='minioadmin',
    aws_secret_access_key='minioadmin',
    config=Config(signature_version='s3v4'),
    region_name='us-east-1'
)

# Criar bucket
s3.create_bucket(Bucket='landing-zone')

# Enviar arquivo
s3.put_object(Bucket='landing-zone', Key='cliente.csv', Body=csv_bytes)

# Listar objetos
response = s3.list_objects_v2(Bucket='landing-zone')
```

## Conexão via Spark S3A

O Apache Spark acessa o MinIO usando o protocolo **S3A** (hadoop-aws):

```python
spark = SparkSession.builder \
    .config('spark.hadoop.fs.s3a.endpoint', 'http://localhost:9020') \
    .config('spark.hadoop.fs.s3a.access.key', 'minioadmin') \
    .config('spark.hadoop.fs.s3a.secret.key', 'minioadmin') \
    .config('spark.hadoop.fs.s3a.path.style.access', 'true') \
    .config('spark.hadoop.fs.s3a.impl', 'org.apache.hadoop.fs.s3a.S3AFileSystem') \
    .config('spark.hadoop.fs.s3a.connection.ssl.enabled', 'false') \
    .getOrCreate()

# Ler CSV
df = spark.read.csv('s3a://landing-zone/cliente.csv', header=True)

# Escrever Delta
df.write.format('delta').save('s3a://bronze/cliente')
```

!!! note "path.style.access"
    O MinIO requer `path.style.access=true` pois usa paths no formato
    `http://endpoint/bucket/key` em vez do formato virtual-hosted
    `http://bucket.endpoint/key` do AWS S3.

## Acessando o Console Web

Após `docker-compose up -d`, acesse [http://localhost:9021](http://localhost:9021) com:

- **User:** `minioadmin`
- **Password:** `minioadmin`

No console é possível:
- Navegar pelos buckets e arquivos
- Fazer upload/download manual
- Verificar metadados dos objetos
- Monitorar uso de armazenamento
