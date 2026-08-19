# Lab 08 - Janelas de Tempo e Agregações em Flink SQL

Neste laboratório, vamos explorar um dos pilares mais importantes da engenharia de dados em tempo real: o **Janelamento Temporal (*Windowing*)** e o processamento com estado (*Stateful Stream Processing*) no **Confluent Cloud for Apache Flink**.

Em streams contínuos, os dados são infinitos por definição. Para responder a perguntas de negócio (como volume financeiro transacionado por minuto ou detecção de picos de carga), precisamos fatiar esse fluxo infinito em intervalos de tempo discretos.

## 🎯 Objetivos
- Entender a semântica de **Tumbling Windows** (Janelas Fixas/Não-sobrepostas) e **Hop Windows** (Janelas Deslizantes).
- Praticar funções de agregação temporal (`COUNT`, `SUM`, `AVG`, `ROUND`) em tempo real.
- Analisar como a emissão e fechamento de janelas são orquestrados pelos **Watermarks**.
- Compreender o ciclo de vida do estado em memória (*State Backends*) durante agregações contínuas.

---

## 🛠️ O Cenário de Negócio
Imagine uma instituição financeira que precisa monitorar transações em tempo real para:
1. **Consolidação Periódica (Tumbling Window):** Calcular métricas de volume e contagem a cada 1 minuto.
2. **Monitoramento Deslizante (Hop Window):** Acompanhar janelas de 1 minuto recalculadas a cada 20 segundos para detecção precoce de anomalias.

---

## 📝 Passo 1: Criação da Tabela Geradora (DDL)

No Confluent Cloud, acesse o menu lateral **`SQL workspaces`**, abra uma nova aba e execute o DDL abaixo:

```sql
-- 1. Criação da fonte de transações com conector faker (5 eventos/segundo)
CREATE TABLE transactions_stream (
    transaction_id BIGINT,
    amount DOUBLE,
    transaction_time TIMESTAMP(3),
    -- Watermark: tolerância de 5 segundos a eventos atrasados
    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'faker',
    'rows-per-second' = '5',
    'fields.transaction_id.expression' = '#{number.numberBetween ''1'',''1000''}',
    'fields.amount.expression' = '#{number.randomDouble ''2'',''10'',''1000''}',
    'fields.transaction_time.expression' = '#{date.past ''10'',''SECONDS''}'
);
```

> **Ação:** Clique em **Run** e aguarde a confirmação de criação da tabela no catálogo.

---

## ⏱️ Passo 2: Executando Tumbling Windows (Janelas Fixas)

As janelas **Tumbling** particionam o tempo em blocos fixos e não sobrepostos (ex: 10:00-10:01, 10:01-10:02).

Em uma nova célula do workspace, cole a query abaixo e clique em **Run**:

```sql
-- 2. Agregação em Janela Fixa (Tumbling Window de 1 minuto)
SELECT 
    window_start, 
    window_end, 
    COUNT(transaction_id) AS total_transactions,
    ROUND(SUM(amount), 2) AS total_volume_brl,
    ROUND(AVG(amount), 2) AS avg_ticket_brl
FROM TABLE(
    TUMBLE(TABLE transactions_stream, DESCRIPTOR(transaction_time), INTERVAL '1' MINUTES))
GROUP BY window_start, window_end;
```

### 🧐 Dinâmica de Execução & Paciência
1. **Por que o resultado não aparece instantaneamente?**
   - No processamento baseado em *Event Time*, o Flink só fecha o cálculo e emite a linha consolidada quando o tempo do sistema (controlado pelo `WATERMARK`) ultrapassa o `window_end`.
2. Após o primeiro minuto, observe uma nova linha surgindo a cada 60 segundos com o total consolidado daquele intervalo.

---

## ⚡ Passo 3: Exercício de Alta Responsividade (Janela de 10 Segundos)

Para visualizar o fluxo de emissão em ritmo acelerado, teste reduzir o tamanho da janela:

```sql
-- 3. Tumbling Window rápida de 10 segundos
SELECT 
    window_start, 
    window_end, 
    COUNT(transaction_id) AS total_transactions,
    ROUND(SUM(amount), 2) AS total_volume_brl
FROM TABLE(
    TUMBLE(TABLE transactions_stream, DESCRIPTOR(transaction_time), INTERVAL '10' SECONDS))
GROUP BY window_start, window_end;
```

> **Resultado:** Novas linhas consolidadas passarão a fluir a cada 10 segundos no painel **Results**.

---

## 🔄 Passo 4: Janelas Deslizantes (Hop / Sliding Windows)

Em cenários de detecção de fraudes ou monitoramento de SLA, não podemos esperar 1 minuto inteiro para obter uma atualização. As janelas **Hop** permitem calcular métricas de uma janela de tamanho maior com uma frequência de deslizamento menor (sobreposição).

Cole e execute no SQL Workspace:

```sql
-- 4. Janela de 1 minuto recalculada a cada 20 segundos
SELECT 
    window_start, 
    window_end, 
    COUNT(transaction_id) AS total_transactions,
    ROUND(SUM(amount), 2) AS total_volume_brl
FROM TABLE(
    HOP(TABLE transactions_stream, DESCRIPTOR(transaction_time), INTERVAL '20' SECONDS, INTERVAL '1' MINUTES))
GROUP BY window_start, window_end;
```

---

## 🧠 Conceitos de Engenharia de Dados

- **Funções de Janela Baseadas em Tabela (`TABLE(TUMBLE(...))` e `TABLE(HOP(...))`):** É o padrão ANSI SQL moderno adotado pelo Apache Flink para operações relacionais sobre janelas.
- **Disparo de Janela (*Window Trigger*):** O gatilho de emissão dos dados agrupados é determinado pela progressão dos *Watermarks* (`transaction_time - INTERVAL '5' SECOND`), garantindo consistência mesmo na presença de rede instável ou dados fora de ordem.
- **Gerenciamento de Estado (*State Eviction*):** Durante a vigência da janela, o Flink mantém os acumuladores parciais (`COUNT`, `SUM`, etc.) em seu *State Backend*. Assim que a janela é encerrada e emitida, o estado daquele intervalo é purgado da memória automaticamente.

---

## 🧹 Passo 5: Cleanup & Governança

1. Pare as queries ativas clicando em **Stop**.
2. Quando finalizar as práticas de Flink, pause ou exclua os Compute Pools criados no Confluent Cloud para preservar seus créditos.

---

## ✅ Verificação do Laboratório

Você concluiu este laboratório com sucesso se:
1. Criou a tabela `transactions_stream` com taxa de 5 registros/segundo.
2. Executou a Tumbling Window e compreendeu o tempo de espera condicionado ao `WATERMARK`.
3. Executou a Hop Window e visualizou a emissão com janelas sobrepostas.

---
**Próximo Passo:** No Lab 09, aprenderemos como realizar **Joins em Tempo Real** entre streams dinâmicos e tabelas dimensionais de referência (*Stream-Table Joins*).
