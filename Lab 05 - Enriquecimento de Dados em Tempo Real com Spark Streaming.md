# Lab 05 - Enriquecimento de Dados em Tempo Real (Stream-Static Join)

**Disciplina:** Stream Processing & Pipelines  
**Ambiente:** Databricks Free Edition ([login.databricks.com](https://login.databricks.com/))  
**Linguagem:** Python / PySpark & Spark SQL  

---

## 🎯 Objetivo do Lab
Neste laboratório, você irá implementar o padrão arquitetural de **Lookup e Enriquecimento em Tempo Real (*Stream-Static Join*)**, cruzando um fluxo contínuo de eventos (*Streaming DataFrame*) com uma base cadastral ou dimensional de referência (*Static DataFrame*).

Ao final deste exercício, você será capaz de:
1. Criar e gerenciar tabelas estáticas de dimensão/referência no Spark.
2. Realizar operações de `join` em tempo real entre fontes de streaming e DataFrames estáticos.
3. Compreender como o Spark distribui e replica a tabela estática para enriquecer cada micro-batch automaticamente.
4. Consultar os dados enriquecidos (com atributos cadastrais agregados aos eventos) via **Spark SQL**.

---

## 📋 Pré-requisitos & Materiais
- Acesso ao **Databricks Free Edition** ([login.databricks.com](https://login.databricks.com/)).
- Cluster configurado e ativo.
- Volume **`checkpoint`** criado no catálogo `workspace` / schema `default`.
- Dataset de eventos: `/databricks-datasets/structured-streaming/events/`
- Notebook de exercício: `Lab 05 - Enriquecimento de Dados em Tempo Real com Spark Streaming.ipynb`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Acesso e Importação do Notebook
1. Acesse o Databricks em [https://login.databricks.com/](https://login.databricks.com/).
2. No menu lateral, acesse **Workspace** -> **Users** -> seu e-mail.
3. Importe o arquivo `Lab 05 - Enriquecimento de Dados em Tempo Real com Spark Streaming.ipynb` arrastando e soltando (**Drag & Drop**).
4. Associe o notebook ao cluster ativo.

### Passo 2: Criação do Catálogo de Referência (Static DataFrame)
Criamos uma tabela dimensional de referência em memória com descrições de negócio e prioridades:
```python
from pyspark.sql.types import StructType, StructField, TimestampType, StringType
from pyspark.sql.functions import col

# 1. Criação dos dados estáticos de referência (Lookup / Dimensão)
static_data = [
    ("Open", "Abertura de Sessão", "Alta Prioridade"),
    ("Close", "Encerramento de Sessão", "Média Prioridade")
]

df_static = spark.createDataFrame(
    static_data, 
    ["action", "descricao", "prioridade"]
)

display(df_static)
```

### Passo 3: Ingestão de Streaming e Join em Tempo Real
Configuramos a leitura contínua dos eventos e efetuamos o cruzamento via chave `action`:
```python
inputPath = "/databricks-datasets/structured-streaming/events/"

# 2. Schema explícito da fonte de eventos
jsonSchema = StructType([
    StructField("time", TimestampType(), True),
    StructField("action", StringType(), True)
])

# 3. Leitura contínua em streaming
df_streaming = (
    spark.readStream
        .schema(jsonSchema)
        .option("maxFilesPerTrigger", 1)
        .json(inputPath)
)

# 4. Stream-Static Join: O Spark cruza cada micro-batch com a tabela estática
df_enriched = df_streaming.join(df_static, "action")
```

### Passo 4: Inicialização da Query de Enriquecimento
Configuramos o sink em memória com o volume de checkpointing:
```python
# Parar eventuais queries ativas anteriores
for stream in spark.streams.active:
    stream.stop()

# Limpeza do diretório de checkpoint do Lab 05
checkpoint_path = "/Volumes/workspace/default/checkpoint/lab05_checkpoint"
dbutils.fs.rm(checkpoint_path, True)

query = (
    df_enriched.writeStream
        .format("memory")
        .queryName("eventos_enriquecidos")
        .outputMode("append")  # Modo append para dados enriquecidos linha a linha
        .option("checkpointLocation", checkpoint_path)
        .trigger(availableNow=True)
        .start()
)
```

### Passo 5: Consulta dos Dados Enriquecidos via SQL
Verifique como cada registro de evento foi complementado com as colunas `descricao` e `prioridade`:
```sql
%sql
SELECT 
    time, 
    action, 
    descricao, 
    prioridade 
FROM eventos_enriquecidos 
ORDER BY time DESC 
LIMIT 20
```

### Passo 6: Encerramento da Query
```python
# Encerrar a execução da query
print(f"Status da query antes de parar: {query.status}")
query.stop()
print("Query de enriquecimento encerrada com sucesso.")
```

---

## 🧹 Cleanup (Limpeza do Ambiente)
1. **Apagar Checkpoints:** Excluir os diretórios dentro do Volume `checkpoint`.
2. **Apagar o Notebook:** No menu **Workspace** -> `Users` -> remover o notebook do Lab 05 se necessário.

---

## 💡 Desafios Complementares (Para Praticar)
1. **Enriquecimento com Filtro de Prioridade:** Adicione uma filtragem após o join para reter apenas eventos classificados como `"Alta Prioridade"`:
   ```python
   df_alta_prioridade = df_enriched.filter(col("prioridade") == "Alta Prioridade")
   ```
2. **Left Outer Join:** Altere o tipo de join para `df_streaming.join(df_static, "action", "left")` e discuta como o Spark lida com eventos que não possuem correspondência na tabela cadastral.
