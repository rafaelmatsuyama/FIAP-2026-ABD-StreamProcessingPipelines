# Lab 03 - Exemplo Básico Utilizando Apache Spark Streaming (Structured Streaming)

**Disciplina:** Stream Processing & Pipelines  
**Ambiente:** Databricks Free Edition ([login.databricks.com](https://login.databricks.com/))  
**Linguagem:** Python / PySpark & Spark SQL  

---

## 🎯 Objetivo do Lab
Neste laboratório, você irá aprender a construir e gerenciar seu primeiro pipeline de **Streaming de Dados em Tempo Real** utilizando a API de **Structured Streaming** do Apache Spark no Databricks.

Ao final deste exercício, você será capaz de:
1. Criar e configurar o Volume de **Checkpointing** no Unity Catalog / DBFS para garantir tolerância a falhas.
2. Configurar uma fonte de ingestão contínua com `spark.readStream`, aplicando **Schema explícito** e emulando fluxo com `maxFilesPerTrigger`.
3. Aplicar transformações e agregações temporais em janelas de 1 hora (`window`).
4. Iniciar e parametrizar o ciclo de vida da query de streaming com `.writeStream` (Modo `complete`, Sink em memória e Checkpoint).
5. Consultar a tabela em memória interativamente via **Spark SQL** e observar a evolução dos dados em tempo real.
6. Executar o encerramento gracioso da query e a limpeza dos recursos.

---

## 📋 Pré-requisitos & Materiais
- Acesso ao **Databricks Free Edition** ([login.databricks.com](https://login.databricks.com/)).
- Cluster configurado e ativo.
- Dataset de eventos embutido no Databricks: `/databricks-datasets/structured-streaming/events/`
- Notebook de exercício: `Lab 03 - Exemplo Básico Utilizando Apache Spark Streaming.ipynb`

---

## 🚀 Passo a Passo Guiado

### Passo 1: Acesso e Importação do Notebook
1. Acesse o Databricks em [https://login.databricks.com/](https://login.databricks.com/).
2. No menu lateral, acesse **Workspace** -> **Users** -> seu e-mail.
3. Importe o arquivo `Lab 03 - Exemplo Básico Utilizando Apache Spark Streaming.ipynb` arrastando e soltando (**Drag & Drop**).

### Passo 2: Criação do Volume para Checkpointing
O Structured Streaming exige um local seguro para salvar o progresso (offset/WAL) da query.
1. No menu lateral esquerdo, clique em **Catalog**.
2. Navegue até o catálogo **`workspace`** e o schema **`default`**.
3. Crie um novo Volume com o nome exato **`checkpoint`** (caminho: `/Volumes/workspace/default/checkpoint/`).

### Passo 3: Inspeção dos Dados de Origem
Na primeira célula do notebook, verifique a lista de 50 arquivos JSON de eventos disponibilizados pelo Databricks:
```python
# Listando os arquivos da pasta com os datasets de exemplo
display(dbutils.fs.ls('/databricks-datasets/structured-streaming/events/'))
```

### Passo 4: Schema Explícito e Leitura Contínua (`readStream`)
Em Structured Streaming, a inferência automática de schema não é permitida em tempo de execução contínua por motivos de estabilidade e performance:
```python
from pyspark.sql.types import StructType, StructField, TimestampType, StringType
from pyspark.sql.functions import window

inputPath = "/databricks-datasets/structured-streaming/events/"

# 1. Definindo o Schema explícito
jsonSchema = StructType([
    StructField("time", TimestampType(), True),
    StructField("action", StringType(), True)
])

# 2. Ingestão em Streaming (emulando chegada de 1 arquivo por micro-batch)
streamingInputDF = (
    spark.readStream
        .schema(jsonSchema)
        .option("maxFilesPerTrigger", 1)
        .json(inputPath)
)

# 3. Agregação temporal por tipo de ação e janela de 1 hora
streamingCountsDF = (
    streamingInputDF
        .groupBy(
            streamingInputDF.action,
            window(streamingInputDF.time, "1 hour")
        )
        .count()
)
```

### Passo 5: Inicialização da Query de Streaming (`writeStream`)
Configure o sink em memória (`memory`), modo de saída `complete` (para manter os totais recalculados) e aponte para o volume de checkpoint, limpando qualquer checkpoint anterior para permitir reexecuções:
```python
checkpoint_path = "/Volumes/workspace/default/checkpoint/contagem_checkpoint"

# Limpa checkpoint anterior para permitir rodar o notebook repetidas vezes
dbutils.fs.rm(checkpoint_path, True)

query = (
    streamingCountsDF
        .writeStream
        .format("memory")
        .queryName("contagem")
        .outputMode("complete")
        .option("checkpointLocation", checkpoint_path)
        .trigger(availableNow=True)  # Processa o backlog disponível e encerra
        .start()
)
```

> [!IMPORTANT]
> **Atenção Técnica sobre Triggers no Databricks Free / Serverless:**  
> O Databricks Free Edition e ambientes Serverless bloqueiam streaming infinito contínuo via `ProcessingTime` (ex: `.trigger(processingTime="10s")`), gerando o erro `[INFINITE_STREAMING_TRIGGER_NOT_SUPPORTED]`.  
> O uso de **`.trigger(availableNow=True)`** é mandatório: ele consome todos os micro-batches em fila respeitando a taxa configurada (`maxFilesPerTrigger`) e finaliza a query de forma elegante após esgotar o backlog, sem manter o cluster bloqueado indefinidamente.

### Passo 6: Consulta Interativa em SQL & Visualização
Enquanto os micro-batches são processados, consulte a tabela `contagem` em memória:
```sql
%sql
SELECT 
    action, 
    date_format(window.end, "MMM-dd HH:mm") AS time, 
    count 
FROM contagem 
ORDER BY time, action
```
> **Dica Visual:** No Databricks, utilize o botão **+** / **Visualization** no resultado da célula SQL para configurar um gráfico de barras (*Grouped Bar Chart*) agrupado por `action` com o eixo X em `time`.

### Passo 7: Encerramento da Query
Antes de finalizar a sessão ou trocar de lab, garanta que a query foi interrompida:
```python
# Verificar status da query
print(f"Query ativa: {query.isActive}")

# Parar a execução da query
query.stop()
```

---

## 🧹 Cleanup (Limpeza do Ambiente)
Para liberar espaço e evitar conflitos em execuções futuras:
1. **Apagar o Volume de Checkpoint:** No menu **Catalog** -> `workspace` -> `default` -> excluir o Volume **`checkpoint`**.
2. **Apagar o Notebook:** No menu **Workspace** -> `Users` -> excluir o notebook do Lab 03 se não for mais utilizá-lo.

---

## 💡 Desafios Complementares (Para Praticar)
1. **Inspeção de Métricas de Streaming:** Utilize `query.lastProgress` ou `query.recentProgress` no Python para analisar o número de registros processados por segundo e os tempos de execução de cada micro-batch.
2. **Novo Filtro de Negócio:** Adicione um `.filter(col("action") == "Open")` antes do `groupBy` e verifique o resultado na tabela SQL.
