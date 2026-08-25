# Lab 06 - Arquitetura Medallion e Delta Lake com Spark Streaming

**Disciplina:** Stream Processing & Pipelines  
**Ambiente:** Databricks Free Edition ([login.databricks.com](https://login.databricks.com/))  
**Linguagem:** Python / PySpark & Spark SQL  

---

## 🎯 Objetivo do Lab
Neste laboratório, você irá implementar um pipeline de ponta a ponta seguindo o padrão de **Arquitetura Medallion (Bronze $\rightarrow$ Silver)** em tempo real, utilizando **Delta Lake** como camada de armazenamento com transações ACID e tolerância a falhas.

Ao final deste exercício, você será capaz de:
1. Ingerir dados brutos de streaming diretamente no formato **Delta Lake** (**Camada Bronze**).
2. Consumir a tabela Delta Bronze como uma nova fonte de streaming contínuo.
3. Aplicar regras de limpeza, enriquecimento temporal e filtros na **Camada Silver**.
4. Compreender a importância do `checkpointLocation` para garantir o processamento *Exactly-Once*.
5. Auditar as transações e micro-batches gerados inspecionando o **Delta Transaction Log** (`DESCRIBE HISTORY`).

---

## 📋 Pré-requisitos & Materiais
- Acesso ao **Databricks Free Edition** ([login.databricks.com](https://login.databricks.com/)).
- Cluster configurado e ativo.
- Volume **`checkpoint`** criado no catálogo `workspace` / schema `default`.
- Dataset de eventos: `/databricks-datasets/structured-streaming/events/`
- Notebook de exercício: `Lab 06 - Arquitetura Medallion e Delta Lake com Spark Streaming.ipynb`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Acesso e Importação do Notebook
1. Acesse o Databricks em [https://login.databricks.com/](https://login.databricks.com/).
2. No menu lateral, acesse **Workspace** -> **Users** -> seu e-mail.
3. Importe o arquivo `Lab 06 - Arquitetura Medallion e Delta Lake com Spark Streaming.ipynb` arrastando e soltando (**Drag & Drop**).
4. Associe o notebook ao cluster ativo.

### Passo 2: Configuração de Caminhos e Limpeza Inicial
Definimos os caminhos de armazenamento Delta e seus respectivos diretórios de checkpoint:
```python
from pyspark.sql.types import StructType, StructField, TimestampType, StringType
from pyspark.sql.functions import col, current_timestamp, desc

# 1. Parar queries ativas anteriores
for stream in spark.streams.active:
    stream.stop()

# 2. Definição de caminhos no Volume / DBFS
path_bronze = "/Volumes/workspace/default/checkpoint/lab06_delta/bronze"
path_silver = "/Volumes/workspace/default/checkpoint/lab06_delta/silver"
checkpoint_bronze = "/Volumes/workspace/default/checkpoint/lab06_checkpoints/bronze"
checkpoint_silver = "/Volumes/workspace/default/checkpoint/lab06_checkpoints/silver"

# 3. Limpeza de execuções anteriores para permitir reexecução
dbutils.fs.rm("/Volumes/workspace/default/checkpoint/lab06_delta", True)
dbutils.fs.rm("/Volumes/workspace/default/checkpoint/lab06_checkpoints", True)
```

### Passo 3: Camada BRONZE (Ingestão Raw $\rightarrow$ Delta Lake)
Consumimos os arquivos brutos JSON e persistimos no formato Delta preservando os dados em estado bruto (*Raw*):
```python
inputPath = "/databricks-datasets/structured-streaming/events/"

jsonSchema = StructType([
    StructField("time", TimestampType(), True),
    StructField("action", StringType(), True)
])

# Leitura contínua dos arquivos brutos
df_raw = (
    spark.readStream
        .schema(jsonSchema)
        .option("maxFilesPerTrigger", 1)
        .json(inputPath)
)

# Escrita em Streaming no formato Delta (Bronze)
query_bronze = (
    df_raw.writeStream
        .format("delta")
        .outputMode("append")
        .option("checkpointLocation", checkpoint_bronze)
        .trigger(availableNow=True)
        .start(path_bronze)
)

query_bronze.awaitTermination()
print("Camada Bronze processada com sucesso!")
```

### Passo 4: Camada SILVER (Leitura do Stream Bronze $\rightarrow$ Transformação $\rightarrow$ Delta Silver)
Lemos a tabela Bronze como um novo stream, adicionamos metadados de auditoria e gravamos na Silver:
```python
# 1. Leitura contínua a partir da tabela Delta Bronze
df_bronze_stream = (
    spark.readStream
        .format("delta")
        .load(path_bronze)
)

# 2. Transformações: Adição de timestamp de processamento e filtragem de nulos
df_silver = (
    df_bronze_stream
        .withColumn("processamento_ts", current_timestamp())
        .filter(col("action").isNotNull())
)

# 3. Escrita na camada Silver em formato Delta
query_silver = (
    df_silver.writeStream
        .format("delta")
        .outputMode("append")
        .option("checkpointLocation", checkpoint_silver)
        .trigger(availableNow=True)
        .start(path_silver)
)

query_silver.awaitTermination()
print("Camada Silver processada com sucesso!")
```

### Passo 5: Validação do Pipeline Medallion via SQL
Consulte a tabela Silver refinada com as transformações e o timestamp de ingestão:
```python
# Consulta da tabela Silver ordenada pelos dados mais recentes
df_silver_resultado = spark.read.format("delta").load(path_silver)
display(df_silver_resultado.orderBy(desc("processamento_ts")))
```

### Passo 6: Auditoria de Transações ACID (Delta History)
O Delta Lake registra cada micro-batch como uma transação atômica. Inspecione o histórico de operações:
```sql
%sql
DESCRIBE HISTORY delta.`/Volumes/workspace/default/checkpoint/lab06_delta/silver`
```

### Passo 7: Encerramento das Queries
```python
query_bronze.stop()
query_silver.stop()
print("Queries das camadas Bronze e Silver encerradas com sucesso.")
```

---

## 🧹 Cleanup (Limpeza do Ambiente)
Para liberar espaço no volume do Databricks:
```python
# Limpeza completa das tabelas Delta e Checkpoints do Lab 06
dbutils.fs.rm("/Volumes/workspace/default/checkpoint/lab06_delta", True)
dbutils.fs.rm("/Volumes/workspace/default/checkpoint/lab06_checkpoints", True)
```

---

## 💡 Desafios Complementares (Para Praticar)
1. **Camada GOLD (Agregação de Negócio):** Crie uma terceira camada (Gold) que lê da camada Silver e gera uma tabela Delta agregando a contagem de eventos por hora e tipo de ação.
2. **Delta Time Travel:** Utilize a sintaxe `spark.read.format("delta").option("versionAsOf", 0).load(path_silver)` para consultar o estado exato dos dados na primeira versão da tabela.
