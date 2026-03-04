# Lab 06 - Joins e Enriquecimento de Streams

Em cenários reais, os dados que chegam via stream (ex: IDs de transação) costumam ser "magros". Para que essa informação seja útil para o negócio, precisamos enriquecê-la com dados de referência (ex: nome do cliente, categoria do produto, etc.).

No Apache Flink, existem diversas formas de realizar Joins. Neste laboratório, vamos focar no **Lookup Join** e no **Regular Join**.

## 🎯 Objetivos
- Criar múltiplas tabelas temporárias no mesmo script.
- Realizar o cruzamento (Join) entre um stream de transações e uma tabela de usuários.
- Entender como o Flink gerencia o estado para manter esses Joins.

---

## 🛠️ O Cenário
Vamos simular dois fluxos:
1. **Stream de Transações:** IDs de transação e IDs de usuário.
2. **Tabela de Usuários:** Dados cadastrais (Nome e Nível de Fidelidade).

### 1. Preparando o Ambiente
Cole o script abaixo no seu **SQL Editor**:

```sql
-- 1. Fonte de Transações (Stream)
CREATE TEMPORARY TABLE transactions (
    transaction_id BIGINT,
    user_id INT,
    amount DOUBLE,
    transaction_time TIMESTAMP(3),
    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'datagen',
    'rows-per-second' = '2',
    'fields.user_id.kind' = 'random',
    'fields.user_id.min' = '1',
    'fields.user_id.max' = '10' -- Simulando 10 usuários possíveis
);

-- 2. Tabela de Usuários (Referência/Dimensão)
-- Usamos o conector 'faker' apenas para o nome, que é infalível
CREATE TEMPORARY TABLE users (
    user_id INT,
    user_name STRING
) WITH (
    'connector' = 'faker',
    'fields.user_id.expression' = '#{number.numberBetween ''1'',''10''}',
    'fields.user_name.expression' = '#{Name.firstName}'
);

-- 3. Query de Enriquecimento (Join) com lógica de categorias no SQL
SELECT 
    t.transaction_id,
    t.amount,
    u.user_name,
    -- Criando as categorias via SQL para evitar erros no conector
    CASE 
        WHEN t.user_id IN (1, 4, 8) THEN 'PLATINUM'
        WHEN t.user_id IN (2, 6, 10) THEN 'GOLD'
        WHEN t.user_id IN (3, 7) THEN 'SILVER'
        ELSE 'BRONZE'
    END as loyalty_level,
    t.transaction_time
FROM transactions t
LEFT JOIN users u ON t.user_id = u.user_id;
```

---

## 🚀 Executando o Experimento

1. Clique no botão **"Debug"**.
2. Observe o resultado: cada transação gerada agora vem acompanhada do nome do usuário e seu nível de fidelidade.
3. **Desafio:** Tente adicionar uma cláusula `WHERE` para filtrar apenas transações de usuários **PLATINUM**.

---

## 🧐 O que está acontecendo?

No Flink SQL, quando fazemos um `JOIN` comum (como o acima):
- **Estado (State):** O Flink precisa manter em memória (ou disco) as informações das tabelas para realizar o cruzamento conforme os dados chegam.
- **Dynamic Tables:** O SQL do Flink trata streams como "Tabelas Dinâmicas". Quando uma nova transação chega, o Flink consulta o estado da tabela `users` e emite o resultado enriquecido.
- **Lookup Joins (Contexto):** Em ambientes produtivos, a tabela de usuários poderia estar em um banco de dados externo (PostgreSQL, Redis, MySQL). O Flink permite consultar esses bancos "on-the-fly" para cada registro que passa pelo stream.

---

## ✅ Verificação do Laboratório
Você concluiu este lab se:
1. Executou o Join com sucesso via **Debug**.
2. Viu transações enriquecidas com nomes e níveis de fidelidade aleatórios.
3. Conseguiu filtrar os resultados por uma coluna da tabela de usuários.

---
**Próximo Passo:** No Lab 07, vamos elevar o nível e ver como capturar mudanças em bancos de dados reais usando **CDC (Change Data Capture)**.
