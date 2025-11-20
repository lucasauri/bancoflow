Markdown

# HortiFlow - Documentação Técnica do Banco de Dados

Documentação detalhada da estrutura, scripts DDL e regras de negócio implementadas no banco de dados PostgreSQL do sistema HortiFlow.

## 📋 Estrutura das Tabelas Principais

Abaixo estão os scripts de criação das tabelas essenciais que sustentam o negócio.

### 1. Produtos
Catálogo de itens com controle de estoque e preços.
```sql
CREATE TABLE public.produtos (
    id bigserial NOT NULL,
    nome varchar(255) NOT NULL,
    preco float8 NOT NULL,
    embalagem varchar(255) NULL,
    estoque_inicial float8 NULL,
    entradas float8 NULL,
    saidas float8 NULL,
    valor_200m numeric(10, 2) NULL, -- Mantido para histórico/compatibilidade
    valor_15m numeric(10, 2) NULL,  -- Mantido para histórico/compatibilidade
    CONSTRAINT produtos_pkey PRIMARY KEY (id)
);
CREATE INDEX idx_produto_nome ON public.produtos USING btree (nome);
2. Clientes
Cadastro de pessoas físicas e jurídicas.

SQL

CREATE TABLE public.clientes (
    id bigserial NOT NULL,
    nome varchar(255) NOT NULL,
    cpf varchar(14) NULL,
    cnpj varchar(255) NULL,
    telefone varchar(255) NULL,
    estado varchar(255) NULL,
    banco varchar(255) NULL,
    cond_pgto varchar(255) NULL,
    ie varchar(255) NULL,
    CONSTRAINT clientes_pkey PRIMARY KEY (id)
);
3. Vendas
Cabeçalho das transações comerciais.

SQL

CREATE TABLE public.vendas (
    id bigserial NOT NULL,
    cliente_id int8 NOT NULL,
    data_venda date DEFAULT CURRENT_DATE NOT NULL,
    valor_total numeric(10, 2) NOT NULL,
    forma_pagamento varchar(50) NOT NULL,
    status varchar(50) DEFAULT 'Concluída' NOT NULL,
    CONSTRAINT vendas_pkey PRIMARY KEY (id),
    CONSTRAINT vendas_valor_total_check CHECK ((valor_total >= (0)::numeric)),
    CONSTRAINT vendas_cliente_id_fkey FOREIGN KEY (cliente_id) REFERENCES public.clientes(id)
);
4. Itens da Venda
Relacionamento N:N entre Vendas e Produtos.

SQL

CREATE TABLE public.itens_venda (
    id bigserial NOT NULL,
    venda_id int8 NOT NULL,
    produto_id int8 NOT NULL,
    quantidade int4 NOT NULL,
    preco_unitario numeric(10, 2) NOT NULL,
    total_item numeric(10, 2) NOT NULL,
    CONSTRAINT itens_venda_pkey PRIMARY KEY (id),
    CONSTRAINT itens_venda_quantidade_check CHECK ((quantidade > 0)),
    CONSTRAINT itens_venda_produto_id_fkey FOREIGN KEY (produto_id) REFERENCES public.produtos(id),
    CONSTRAINT itens_venda_venda_id_fkey FOREIGN KEY (venda_id) REFERENCES public.vendas(id)
);
⚡ Gatilhos (Triggers) e Funções
Automação de regras de negócio diretamente no banco de dados.

1. Baixa Automática de Estoque
Atualiza o saldo do produto ao registrar um item de venda e impede estoque negativo.

Função:

SQL

CREATE OR REPLACE FUNCTION public.fn_baixa_estoque_venda()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    UPDATE produtos
    SET saidas = saidas + NEW.quantidade
    WHERE id = NEW.produto_id;

    IF (SELECT (estoque_inicial + entradas - saidas) FROM produtos WHERE id = NEW.produto_id) < 0 THEN
        RAISE EXCEPTION 'Estoque insuficiente para o produto ID %', NEW.produto_id;
    END IF;
    RETURN NEW;
END;
$$;
Trigger:

SQL

CREATE TRIGGER tr_baixa_estoque
AFTER INSERT ON public.itens_venda
FOR EACH ROW EXECUTE FUNCTION fn_baixa_estoque_venda();
2. Auditoria de Preços
Registra histórico sempre que o preço de um produto é alterado.

Função:

SQL

CREATE OR REPLACE FUNCTION public.fn_auditoria_preco()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    IF OLD.preco IS DISTINCT FROM NEW.preco THEN
        INSERT INTO auditoria_produtos (produto_id, preco_antigo, preco_novo)
        VALUES (NEW.id, OLD.preco, NEW.preco);
    END IF;
    RETURN NEW;
END;
$$;
Trigger:

SQL

CREATE TRIGGER tr_auditoria_preco
BEFORE UPDATE ON public.produtos
FOR EACH ROW EXECUTE FUNCTION fn_auditoria_preco();
📊 Visões (Views)
Relatórios pré-compilados para o Dashboard e Análises.

1. Relatório de Vendas Detalhado
Une as tabelas para mostrar quem comprou o quê e quando.

SQL

CREATE OR REPLACE VIEW public.relatorio_vendas_detalhadas AS
SELECT
    v.id AS venda_id,
    v.data_venda,
    c.nome AS nome_cliente,
    p.nome AS nome_produto,
    iv.quantidade,
    iv.total_item,
    v.forma_pagamento
FROM vendas v
    JOIN clientes c ON v.cliente_id = c.id
    JOIN itens_venda iv ON v.id = iv.venda_id
    JOIN produtos p ON iv.produto_id = p.id
ORDER BY v.data_venda DESC;
2. Relatório Financeiro de Estoque
Calcula o saldo atual e o valor monetário total em mercadorias.

SQL

CREATE OR REPLACE VIEW public.relatorio_estoque_atual AS
SELECT
    id,
    nome,
    embalagem,
    estoque_inicial,
    entradas,
    saidas,
    (estoque_inicial + entradas - saidas) AS estoque_atual,
    ((estoque_inicial + entradas - saidas) * preco) AS valor_total_estoque
FROM public.produtos
ORDER BY nome;
🛠 Comandos Úteis
Backup do Banco
Bash

pg_dump -U postgres -d hortiflow -F c -b -v -f "backup_hortiflow.sql"
Restore do Banco
Bash

pg_restore -U postgres -d hortiflow -v "backup_hortiflow.sql"