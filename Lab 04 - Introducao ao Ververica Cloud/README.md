# Lab 04 - Introdução ao Ververica Cloud e SQL Hello World

Este laboratório marca nossa transição para uma plataforma Apache Flink, com a versão gerenciada na nuvem. Vamos utilizar o **Ververica Cloud**, a solução SaaS oficial dos criadores originais do Apache Flink, para facilitar o desenvolvimento e operação de nossas pipelines de processamento de stream.

## 🎯 Objetivos
- Criar e configurar uma conta gratuita no Ververica Cloud.
- Provisionar um Workspace de Flink.
- Executar o primeiro Job Flink SQL ("Hello World") utilizando geradores de dados e conectores de console (print).

---

## 🚀 Passo 1: Criação de Conta no Ververica Cloud

1. Acesse o portal: [https://cloud.ververica.com/](https://cloud.ververica.com/).
2. Clique em **"Sign Up"** (ou "Try for Free").
3. Você pode se registrar utilizando seu e-mail pessoal ou via autenticação social (Google).
4. Siga os passos de verificação de e-mail e complete o perfil básico.

> **Dica:** O plano gratuito (Free Tier) geralmente oferece créditos iniciais ou limites de CU (Compute Units) suficientes para fins acadêmicos.

---

## 🛠️ Passo 2: Configurando seu Primeiro Workspace

Um **Workspace** é o ambiente isolado onde seus jobs de Flink rodarão.

1. No painel principal (Console), clique em **"Create Workspace"**.
2. Escolha um nome para seu workspace (ex: `fiap-30abd-data-eng-mba`).
3. Selecione a região mais próxima de você (ex: `AWS - Europe (Frankfurt)`).
4. Clique em **Create Workspace**.
5. Aguarde alguns minutos até que o status mude para **Running** e clique no nome do Workspace para entrar no "Flink Management Console".

---

## 📝 Passo 3: O "Hello World" do Flink SQL

No Flink, não precisamos necessariamente escrever código Java/Python para processar dados; o **Flink SQL** é uma ferramenta poderosa para isso.

### 3.1 Acessando o SQL Editor
No menu lateral esquerdo do Workspace, clique em **SQL Editor**.

### 3.2 Executando o Script Hello World
No Ververica Cloud, podemos usar tabelas **temporárias** para testar nossas queries sem precisar registrá-las permanentemente no catálogo.

1. No editor, clique em **New > New Blank Draft** e cole o script completo abaixo:

```sql
-- Criando fonte de dados temporária
CREATE TEMPORARY TABLE transactions (
    transaction_id BIGINT,
    amount DOUBLE,
    transaction_time TIMESTAMP(3),
    WATERMARK FOR transaction_time AS transaction_time - INTERVAL '5' SECOND
) WITH (
    'connector' = 'datagen',
    'rows-per-second' = '2',
    'fields.transaction_id.kind' = 'sequence',
    'fields.transaction_id.start' = '1',
    'fields.transaction_id.end' = '1000',
    'fields.amount.min' = '1.0',
    'fields.amount.max' = '500.0'
);

-- Consultando os dados gerados
SELECT * FROM transactions;
```

2. Clique no botão **"Debug"**.
3. O Ververica iniciará uma sessão de depuração. Se solicitado, confirme a criação/uso de um **Session Cluster**.

> **Dica:** Na criação do **Session Cluster**, coloque o **State** em **Running** e o **Engine Version** na opção recomendada.

4. Observe a aba **"Results"** (ou Preview) na parte inferior para ver os dados "brotando" em tempo real!

---

## ✅ Verificação do Ambiente

Para considerar este laboratório concluído com sucesso, você deve:
1. Conseguir rodar o script completo (DDL + SELECT) usando o modo **Debug**.
2. Ver as linhas de dados sendo geradas na aba de resultados a cada segundo.

---
**Próximo Passo:** No Lab 05, aprenderemos a realizar agregações nestes dados utilizando Janelas de Tempo (Windowing).
