# Lab 07 - Introdução ao Confluent Cloud e Flink SQL Hello World

Este laboratório marca nossa transição para o processamento de stream nativo com o Apache Flink, utilizando a infraestrutura gerenciada e *serverless* do **Confluent Cloud for Apache Flink**. Vamos aprender a provisionar um ambiente em nuvem, criar um Compute Pool de Flink e executar nossas primeiras queries contínuas em Flink SQL.

## 🎯 Objetivos
- Criar e configurar uma conta gratuita no Confluent Cloud (com créditos promocionais de US$ 400).
- Criar um Environment e um Cluster Kafka básico.
- Provisionar um **Flink Compute Pool** serverless.
- Executar o primeiro Job Flink SQL ("Hello World") utilizando geradores de eventos sintéticos (`faker`).
- Compreender a semântica de *Continuous Queries*, *Watermarking* e gestão de custos (*CFUs*).

---

## 🚀 Passo 1: Criação de Conta no Confluent Cloud

1. Acesse o portal de cadastro: [https://confluent.cloud/signup](https://confluent.cloud/signup).
2. Preencha seus dados de cadastro ou utilize login social (Google/GitHub).
3. Confirme o e-mail de ativação para liberar os **US$ 400 em créditos gratuitos** (válidos por 30 dias).

> **Nota de Gestão de Custo:** Os créditos cobrem com folga todos os laboratórios do curso. Ao final de cada sessão prática, exclua ou pause o *Flink Compute Pool* e os clusters para evitar consumo residual da sua cota.

---

## 🛠️ Passo 2: Configurando o Ambiente e Cluster Kafka

1. No painel principal (Cloud Console), acesse **Environments** e selecione o ambiente padrão (`default`) ou crie um novo (ex: `fiap-abd-env`).
2. Clique em **Create Cluster**:
   - Selecione o tipo de cluster: **Basic** (suficiente para desenvolvimento e acadêmico).
   - Escolha o provedor de nuvem e região: **AWS** na região `us-east-1` (N. Virginia) ou `us-east-2` (Ohio).
   - Defina o nome do cluster (ex: `cluster-spp-abd`).
   - Clique em **Launch Cluster**.

---

## ⚡ Passo 3: Acessando o Flink SQL Workspace & Compute Pool

No Confluent Cloud, a engine de Flink é gerenciada diretamente através dos **SQL workspaces**.

1. No menu lateral esquerdo principal, clique em **`SQL workspaces`**.
2. No topo da tela do workspace, selecione ou provisione o seu **Compute Pool**:
   - Se ainda não houver um pool ativo, clique no botão/aviso **"Create compute pool"** (ou configure pelo menu suspenso).
   - Provedor e Região: Selecione **exatamente a mesma região** onde criou o Cluster Kafka (ex: `AWS / us-east-1`).
   - Nome do Pool: `flink-pool-abd`.
   - Capacidade Máxima (CFU - *Confluent Flink Units*): Mantenha o padrão (ex: `5` ou `10` CFUs). Como o serviço é serverless, você só paga pelas CFUs efetivamente utilizadas durante a execução da query.
3. Aguarde alguns instantes até o Compute Pool estar ativo e pronto para executar queries.

---

## 📝 Passo 4: O "Hello World" no Flink SQL Workspace

No Confluent Cloud Flink SQL, instruções DDL (`CREATE TABLE`) e consultas contínuas (`SELECT`) devem ser executadas em blocos separados.

### 4.1 Criação da Tabela Geradora (DDL)
Cole o script de criação no editor do SQL Workspace e clique em **Run**:

```sql
-- 1. Criação da tabela geradora de eventos usando o conector oficial 'faker'
CREATE TABLE transactions (
    transaction_id BIGINT,
    amount DOUBLE,
    transaction_time TIMESTAMP(3),
    -- Watermark: define tolerância de até 5 segundos para eventos atrasados
    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'faker',
    'rows-per-second' = '2',
    'fields.transaction_id.expression' = '#{number.numberBetween ''1'',''1000''}',
    'fields.amount.expression' = '#{number.randomDouble ''2'',''1'',''500''}',
    'fields.transaction_time.expression' = '#{date.past ''10'',''SECONDS''}'
);
```

> **Confirmação:** Aguarde a mensagem de sucesso indicando que a tabela foi registrada no catálogo.

---

### 4.2 Executando a Consulta Contínua (Continuous Query)
Em uma nova célula (ou limpando o editor), cole a query de leitura e clique em **Run**:

```sql
-- 2. Continuous Query consumindo os eventos em tempo real
SELECT * FROM transactions;
```

> **Observação:** Acompanhe na aba inferior **Results** os registros sendo emitidos continuamente a cada segundo conforme são gerados pela engine do Flink!

---

## 🧐 O que está acontecendo por trás dos panos?

- **Conector `faker`:** É o conector nativo suportado no catálogo gerenciado do Confluent Cloud for Apache Flink para gerar fluxos dinâmicos de dados sintéticos baseados em expressões (IDs aleatórios, valores decimais, datas recentes).
- **Taxa de Geração (`rows-per-second`):** Controla a vazão do stream (neste exemplo, 2 registros por segundo).
- **Semântica de Watermark:** `WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND` orienta o Flink sobre o avanço do tempo do evento (*Event Time*), estabelecendo o limiar temporal para fechamento de janelas e tolerância a atrasos.
- **Continuous Query:** Diferente do SQL tradicional de bancos relacionais (que executa sobre uma foto estática do banco), o `SELECT` no Flink opera de forma contínua (*streaming query*), processando novos eventos conforme eles chegam ao sistema.

---

## 🧹 Passo 5: Cleanup e Limpeza do Ambiente

Para garantir a organização do catálogo e o melhor aproveitamento dos seus créditos promocionais:

### 5.1 Parar a Query Contínua
1. No SQL Workspace, pare a execução da query `SELECT * FROM transactions` clicando em **Stop**.

### 5.2 Remoção da Tabela Geradora (DROP TABLE)
No editor do SQL Workspace, execute a instrução DDL de remoção:

```sql
-- 3. Limpeza do catálogo de tabelas Flink
DROP TABLE IF EXISTS transactions;
```

### 5.3 Gestão do Flink Compute Pool
1. **Importante:** **Mantenha o seu Cluster Kafka ativo** (não o remova), pois ele será reutilizado nos próximos laboratórios (Lab 08 e Lab 09).

---

## ✅ Verificação do Laboratório

Você concluiu este laboratório se:
1. Criou sua conta e ativou o cluster e o Flink Compute Pool no Confluent Cloud.
2. Executou com sucesso a criação da tabela `transactions` com `faker` e `WATERMARK`.
3. Visualizou os registros fluindo em tempo real no console de resultados do Flink SQL Workspace.
4. Executou o `DROP TABLE` e parou o stream para limpeza do ambiente.

---
**Próximo Passo:** No Lab 08, exploraremos agregações temporais contínuas utilizando **Janelas de Tempo (Tumbling e Sliding Windows)** no Flink SQL.
