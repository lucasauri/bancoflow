HortiFlow - Banco de Dados & Engenharia
Documentação da estrutura de dados, modelagem e regras de negócio do sistema HortiFlow (PostgreSQL).

🚀 Como Configurar o Banco
Criar o Banco de Dados:

SQL

CREATE DATABASE hortiflow;
Restaurar a Estrutura (Schema): Utilize o script DDL disponível na pasta raiz ou execute via linha de comando:

Bash

psql -U postgres -d hortiflow -f backend/src/main/resources/db/migration/schema.sql
Credenciais Padrão:

Host: localhost:5432

User: postgres

Pass: 123456

📋 Estrutura do Modelo (3FN)
O banco segue o modelo relacional normalizado para garantir integridade.

✅ Produtos: Catálogo com controle de preços e saldo físico.

✅ Clientes & Endereços: Dados cadastrais vinculados (1:N).

✅ Vendas & Itens: Transações comerciais com relacionamento de composição.

✅ Movimentações: Histórico de entradas e saídas de estoque.

✅ Auditoria: Log de alterações de preços.

🔧 Recursos Avançados (BDII)
Funcionalidades implementadas diretamente no banco para performance e segurança.

⚡ Gatilhos (Triggers)
tr_baixa_estoque: Dá baixa automática no produto após a venda e bloqueia se o estoque for insuficiente.

tr_auditoria_preco: Grava histórico automático sempre que um preço é alterado.

📊 Visões (Views)
relatorio_vendas_detalhadas: Tabela virtual que une Vendas + Clientes + Produtos para relatórios.

relatorio_estoque_atual: Cálculo dinâmico do saldo e valor patrimonial.

⚙️ Funções (Stored Procedures)
fn_alerta_estoque_baixo(limite): Retorna produtos que precisam de reposição.

🛡️ Segurança e Backup
Controle de Acesso:

hortiflow_backend: Apenas permissões de CRUD (DML).

hortiflow_admin: Permissões totais para manutenção (DDL).

Política de Backup:

Ferramenta: pg_dump (Formato Custom).

Frequência: Diária.

🛠 Tecnologias
SGBD: PostgreSQL 14+

Linguagem: SQL / PL/pgSQL

Modelagem: UML / Modelo Relacional (3FN)
<img width="546" height="605" alt="image" src="https://github.com/user-attachments/assets/77d40a05-5600-4167-b280-5f2da1f86d07" />
