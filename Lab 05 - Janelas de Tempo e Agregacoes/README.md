# Lab 05 - Janelas de Tempo e Agregações (Windowing)

Agora que você já dominou o setup básico e a execução de queries simples no Ververica Cloud, vamos explorar um dos conceitos fundamentais do processamento de stream: **Windowing**.

No streaming, os dados são infinitos. Para calcular métricas (como a soma das transações por minuto), precisamos "fatiar" esse fluxo contínuo em pedaços finitos de tempo.

## 🎯 Objetivos
- Entender o conceito de **Tumbling Window** (Janelas de Tombamento).
- Praticar agregações temporais (`SUM`, `AVG`, `COUNT`) em tempo real.
- Visualizar o comportamento dos **Watermarks**.

---

## 🛠️ O Cenário
Imagine que precisamos monitorar o volume total de transações a cada 1 minuto para detectar picos ou anomalias no sistema financeiro.

### 1. Preparando o Ambiente
No seu **SQL Editor** do Ververica Cloud, limpe o conteúdo anterior e cole o script abaixo:

```sql
-- Criando fonte de dados temporária com Watermark
CREATE TEMPORARY TABLE transactions (
    transaction_id BIGINT,
    amount DOUBLE,
    transaction_time TIMESTAMP(3),
    -- Define que os dados podem chegar com até 5 segundos de atraso
    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'datagen',
    'rows-per-second' = '5', -- Aumentamos para 5 transações por segundo
    'fields.transaction_id.kind' = 'sequence',
    'fields.transaction_id.start' = '1',
    'fields.transaction_id.end' = '1000',
    'fields.amount.min' = '10.0',
    'fields.amount.max' = '1000.0'
);

-- Consulta com Janela de Tempo (Tumbling Window de 1 minuto)
SELECT 
    window_start, 
    window_end, 
    COUNT(transaction_id) as total_txs,
    SUM(amount) as total_volume,
    AVG(amount) as avg_amount
FROM TABLE(
    TUMBLE(TABLE transactions, DESCRIPTOR(transaction_time), INTERVAL '1' MINUTES))
GROUP BY window_start, window_end;
```

---

## 🚀 Executando o Experimento

1. Clique no botão **"Debug"**.
2. Aguarde o início da sessão.
3. **Paciência é a chave:** Como definimos uma janela de **1 minuto**, o Flink só emitirá o resultado consolidado quando o tempo do sistema (controlado pelo Watermark) ultrapassar o fim da janela. 
4. Observe a aba de resultados. Você verá uma nova linha surgindo a cada 60 segundos com o resumo do volume transacionado.

---

## 🧐 O que está acontecendo?

- **TUMBLE Window:** Divide o tempo em blocos fixos e não sobrepostos (ex: 10:00-10:01, 10:01-10:02).
- **Watermark:** O Flink usa o campo `transaction_time` para saber "que horas são" no stream. Se o Watermark ultrapassar o fim da janela, o Flink entende que não chegarão mais dados para aquele período e fecha o cálculo.
- **Janelas em SQL:** Note a sintaxe `TABLE(TUMBLE(...))`, que é o padrão atual do Flink SQL para funções de janela.

---

## ✅ Exercício de Fixação
Tente modificar o intervalo da janela para **10 segundos** (`INTERVAL '10' SECONDS`) e veja como os resultados passam a fluir muito mais rápido no console!

---
**Próximo Passo:** No Lab 06, aprenderemos como cruzar (Join) este stream de transações com dados estáticos (como nomes de clientes) para enriquecer nossa análise.
