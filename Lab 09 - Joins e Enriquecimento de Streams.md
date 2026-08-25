# Lab 09 - Joins e Enriquecimento de Streams em Flink SQL

Em cenários reais de engenharia de dados, os eventos que chegam continuamente via stream (como transações financeiras, cliques ou telemetria IoT) costumam conter apenas identificadores numéricos e dados transacionais brutos.

Para gerar inteligência de negócio e alimentar dashboards operacionais em tempo real, é mandatório enriquecer esse fluxo transitório com dados dimensionais e cadastrais de referência (como nome do cliente, perfil de fidelidade, categoria ou localização).

Neste laboratório no **Confluent Cloud for Apache Flink**, você aprenderá como executar cruzamentos relacionais em tempo real (**Stream Joins**) utilizando Flink SQL.

## 🎯 Objetivos
- Criar múltiplas tabelas geradoras no catálogo Flink.
- Realizar cruzamentos relacionais contínuos (`LEFT JOIN`) entre um fluxo de eventos e uma tabela de usuários.
- Compreender o gerenciamento de estado bi-direcional (*Stateful Joins*) e a semântica de *Dynamic Tables*.
- Aplicar transformações e categorizações condicionais (`CASE WHEN`) diretamente sobre o fluxo enriquecido.

---

## 🛠️ O Cenário de Negócio
Simularemos a operação de um sistema de autorização e antifraude:
1. **Stream de Transações:** Fluxo contínuo contendo `transaction_id`, `user_id`, `amount` e `transaction_time`.
2. **Tabela Dimensional de Usuários:** Base de dados cadastrais contendo `user_id` e `user_name`.

---

## 📝 Passo 1: Criação da Tabela de Transações (DDL 1)

No Confluent Cloud, acesse **`SQL workspaces`**, abra uma nova aba e execute o DDL da tabela de stream:

```sql
-- 1. Stream de eventos de transações com Watermark
CREATE TABLE transactions_join_stream (
    transaction_id BIGINT,
    user_id INT,
    amount DOUBLE,
    transaction_time TIMESTAMP(3),
    -- Watermark: tolerância de até 5 segundos a atrasos
    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'faker',
    'rows-per-second' = '2',
    'fields.user_id.expression' = '#{number.numberBetween ''1'',''10''}',
    'fields.amount.expression' = '#{number.randomDouble ''2'',''10'',''1000''}',
    'fields.transaction_time.expression' = '#{date.past ''10'',''SECONDS''}'
);
```

> **Ação:** Clique em **Run** e aguarde a confirmação de criação no catálogo.

---

## 📝 Passo 2: Criação da Tabela de Usuários (DDL 2)

Em uma nova célula (ou limpando o editor), execute o DDL da tabela dimensional de usuários:

```sql
-- 2. Tabela de referência de usuários
CREATE TABLE users_reference (
    user_id INT,
    user_name STRING,
    PRIMARY KEY (user_id) NOT ENFORCED
) WITH (
    'connector' = 'faker',
    'fields.user_id.expression' = '#{number.numberBetween ''1'',''10''}',
    'fields.user_name.expression' = '#{Name.firstName}'
);
```

> **Ação:** Clique em **Run** e aguarde a criação da tabela.

---

## 🔗 Passo 3: Executando o Enriquecimento em Tempo Real (Continuous Query)

Em uma nova célula, cole e execute a query contínua de cruzamento (`LEFT JOIN`):

```sql
-- 3. Query contínua de enriquecimento com lógica de fidelidade
SELECT 
    t.transaction_id,
    t.amount,
    u.user_name,
    CASE 
        WHEN t.user_id IN (1, 4, 8) THEN 'PLATINUM'
        WHEN t.user_id IN (2, 6, 10) THEN 'GOLD'
        WHEN t.user_id IN (3, 7) THEN 'SILVER'
        ELSE 'BRONZE'
    END AS loyalty_level,
    t.transaction_time
FROM transactions_join_stream t
LEFT JOIN users_reference u ON t.user_id = u.user_id;
```

> **Observação:** Na aba inferior **Results**, acompanhe cada transação gerada sendo enriquecida em tempo real com o nome do cliente e sua respectiva categoria de fidelidade.

---

## 🎯 Passo 4: Desafio Prático (Filtragem de Clientes VIP)

Modifique a query para capturar e monitorar exclusivamente transações de clientes da categoria **PLATINUM**:

```sql
-- 4. Monitoramento direcionado para clientes PLATINUM
SELECT 
    t.transaction_id,
    t.amount,
    u.user_name,
    'PLATINUM' AS loyalty_level,
    t.transaction_time
FROM transactions_join_stream t
LEFT JOIN users_reference u ON t.user_id = u.user_id
WHERE t.user_id IN (1, 4, 8);
```

---

## 🧐 O que está acontecendo por trás dos panos?

- **Stateful Stream Join:** No Flink SQL, operações de `JOIN` mantêm um estado (*State*) em memória para correlacionar registros provenientes de fontes distintas à medida que os eventos chegam.
- **Dynamic Tables & Continuous Queries:** O Flink trata streams como "Tabelas Dinâmicas". A query contínua reavalia as condições relacionais e emite imediatamente o registro enriquecido para os consumidores a jusante (*downstream*).
- **Lookup Joins no Mercado:** Em arquiteturas corporativas de produção, tabelas de dimensão/referência costumam residir em bancos relacionais ou caches distribuídos (PostgreSQL, MySQL, Redis, MongoDB). O Flink permite consultar esses repositórios sob demanda (*Lookup Join*) para cada registro que trafega pelo pipeline.

---

## 🧹 Passo 5: Cleanup e Limpeza do Ambiente
 
Para garantir a organização do catálogo e finalizar o consumo de recursos na nuvem:

### 5.1 Parar as Queries Contínuas
1. No SQL Workspace, pare a execução do `JOIN` contínuo clicando no botão **Stop**.

### 5.2 Remoção das Tabelas (DROP TABLE)
No editor do SQL Workspace, execute as instruções DDL de remoção para as duas tabelas criadas:

```sql
-- 5. Limpeza do catálogo de tabelas Flink
DROP TABLE IF EXISTS transactions_join_stream;
DROP TABLE IF EXISTS users_reference;
```

### 5.3 Encerramento da Infraestrutura no Confluent Cloud
Como este é o último laboratório prático de Apache Flink / Confluent Cloud:
1. **Excluir o Cluster Kafka & Environment (Opcional/Recomendado):** Caso não vá utilizar o cluster para outros projetos pessoais, acesse as configurações do cluster -> **Delete Cluster** e, em seguida, delete o Environment para zerar qualquer consumo residual de créditos da conta.

---

## ✅ Verificação do Laboratório

Você concluiu este laboratório com sucesso se:
1. Criou as tabelas `transactions_join_stream` e `users_reference`.
2. Executou o `LEFT JOIN` contínuo visualizando os dados de transações enriquecidos com os nomes e níveis de fidelidade dos usuários.
3. Aplicou filtros relacionais com sucesso para isolar transações de categorias específicas.
4. Executou o `DROP TABLE` das tabelas de stream e referência e concluiu o cleanup do ambiente.

---
**Próximo Passo:** No próximo módulo, exploraremos o ecossistema de streaming na nuvem com **AWS Kinesis Data Streams e Data Pipelines**.
