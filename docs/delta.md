# Delta Lake

## O que é Delta Lake?

Delta Lake é um formato de tabela open-source criado pela Databricks que adiciona uma camada de confiabilidade sobre arquivos Parquet armazenados em object stores (S3, MinIO, ADLS, GCS).

## Arquitetura

```
s3://bronze/apolice/
├── _delta_log/                  ← Transaction Log
│   ├── 00000000000000000000.json  (versão 0 — WRITE inicial)
│   ├── 00000000000000000001.json  (versão 1 — INSERT novos clientes)
│   ├── 00000000000000000002.json  (versão 2 — UPDATE valor_premio)
│   └── 00000000000000000003.json  (versão 3 — DELETE cancelados)
├── part-00000-abc123.parquet
├── part-00001-def456.parquet
└── part-00002-ghi789.parquet
```

Cada arquivo JSON no `_delta_log` registra:
- **add**: arquivos Parquet adicionados
- **remove**: arquivos Parquet marcados para remoção
- **commitInfo**: metadados (operação, timestamp, usuário, parâmetros)

## Propriedades ACID

| Propriedade | Como Delta garante |
|---|---|
| **Atomicidade** | Transações são commit no log antes de serem visíveis |
| **Consistência** | Validação de schema antes de qualquer write |
| **Isolamento** | Leitores veem snapshot consistente (Snapshot Isolation) |
| **Durabilidade** | Transaction log persiste em object store (MinIO) |

## Operações DML demonstradas

### INSERT (append)

```python
novos_df.write.format('delta').mode('append').save(path)
```

### UPDATE (API DeltaTable)

```python
from delta import DeltaTable

dt = DeltaTable.forPath(spark, path)
dt.update(
    condition=col('tipo_cobertura') == 'Completo',
    set={'valor_premio': col('valor_premio') * lit(1.10)}
)
```

### UPDATE (Spark SQL)

```sql
UPDATE delta.`s3a://bronze/apolice`
SET status = 'inativa'
WHERE status = 'vencida'
```

### DELETE (API DeltaTable)

```python
dt.delete(condition=col('status') == 'cancelado')
```

### DELETE (Spark SQL)

```sql
DELETE FROM delta.`s3a://bronze/sinistro`
WHERE apolice_id IN (
    SELECT id FROM delta.`s3a://bronze/apolice`
    WHERE status = 'inativa'
)
```

## Time Travel

Delta Lake mantém histórico completo de todas as versões. É possível consultar dados em qualquer ponto do tempo.

### Por número de versão

```python
df_original = spark.read.format('delta') \
    .option('versionAsOf', 0) \
    .load('s3a://bronze/apolice')
```

### Por timestamp

```python
df_antes = spark.read.format('delta') \
    .option('timestampAsOf', '2024-01-01') \
    .load('s3a://bronze/apolice')
```

## DESCRIBE HISTORY

```python
DeltaTable.forPath(spark, path) \
    .history() \
    .select('version', 'timestamp', 'operation', 'operationMetrics') \
    .show()
```

Saída típica:

| version | timestamp | operation | operationMetrics |
|---|---|---|---|
| 3 | 2024-05-15 10:32:00 | DELETE | {numRemovedFiles: 1, ...} |
| 2 | 2024-05-15 10:31:00 | UPDATE | {numUpdatedRows: 18, ...} |
| 1 | 2024-05-15 10:30:00 | WRITE | {numAddedFiles: 1, ...} |
| 0 | 2024-05-15 10:29:00 | WRITE | {numAddedFiles: 1, ...} |

## Configuração do Spark

```python
SparkSession.builder \
    .config('spark.sql.extensions',
            'io.delta.sql.DeltaSparkSessionExtension') \
    .config('spark.sql.catalog.spark_catalog',
            'org.apache.spark.sql.delta.catalog.DeltaCatalog') \
    .config('spark.jars.packages',
            'io.delta:delta-spark_2.12:3.2.0,'
            'org.apache.hadoop:hadoop-aws:3.3.4,'
            'com.amazonaws:aws-java-sdk-bundle:1.12.262')
```
