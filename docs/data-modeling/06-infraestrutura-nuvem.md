# 6. Infraestrutura e Nuvem

A infraestrutura do Cultiva Plus define os ambientes onde rodam a aplicação, os bancos de dados e as rotinas de sincronização, garantindo escalabilidade simples e baixo custo.

O modelo descreve onde cada base fica fisicamente, as credenciais técnicas de acesso, o controle das cargas analíticas e a política de retenção dos dados.

---

## 6.1 Onde cada dado mora (implantação)

Os componentes de infraestrutura são organizados da seguinte forma:
* **Front-end Web:** Hospedagem em serviço gerenciado/CDN estático com entrega via HTTPS.
* **API Backend (Sistema Administrativo):** Executado em container/serviço gerenciado de aplicação.
* **Instância de Produção:** Banco de dados relacional `cultiva_oltp` no PostgreSQL gerenciado, acessado pelo usuário técnico `api_app`.
* **Instância Analítica:** Data Warehouse `cultiva_dw` contendo os schemas `stg`, `etl`, `dw` e `bi`.
* **Rotina Agendada:** Agendador diário às 02h00 que executa a procedure `CALL etl.executar_carga()`, usando conexão segura via `postgres_fdw` com o usuário `etl_leitor`.
* **Ferramenta de BI:** Conexão de leitura no schema `bi` via usuário `bi_leitor`.
* **Object Storage:** Armazenamento seguro de backups automatizados diários (PITR) e dumps semanais.

> Na fase inicial (poucas hortas), `cultiva_oltp` e `cultiva_dw` podem ficar **na mesma instância**, em bancos separados — o modelo não muda. Quando o volume crescer, o DW vai para uma instância própria sem alterar nenhum script, só o `oltp_host` do ETL.

## 6.2 Modelo lógico do controle de cargas (schema `etl`)

O schema de controle de ETL monitora a consistência de cada execução com as seguintes tabelas:
* **`watermark`:** Guarda o maior timestamp (`ultimo_atualizado_em`) já extraído por tabela de origem (ex.: `colheita`).
* **`execucao_carga`:** Registra o início, fim, status (`EM_EXECUCAO`, `SUCESSO`, `ERRO`) e mensagem de erro, se houver.
* **`execucao_detalhe`:** Detalha linha por linha cada etapa da carga (extração, dimensões, fatos) e o total de linhas processadas.

Exemplo real após a primeira carga com os dados fictícios:

| Etapa | Tabela | Linhas |
|---|---|---:|
| EXTRAÇÃO | colheita | 16 |
| EXTRAÇÃO | presenca_mutirao | 24 |
| DIMENSÃO | dim_horta (novas versões) | 3 |
| FATO | fato_colheita | 16 |
| FATO | fato_ocupacao_mensal | 75 |

**Monitoramento:** se `EXECUCAO_CARGA.status = 'ERRO'`, a equipe de Infraestrutura e DevOps recebe alerta; os dashboards continuam mostrando a última carga válida.

## 6.3 Usuários de banco e privilégios (menor privilégio)

| Usuário de banco | Usado por | `cultiva_oltp` | `cultiva_dw` |
|---|---|---|---|
| `api_app` | Sistema Administrativo | `SELECT/INSERT/UPDATE/DELETE` em `oltp.*`; **sem** `UPDATE/DELETE` em `log_auditoria` | — |
| `etl_leitor` | ETL (via `postgres_fdw`) | **só `SELECT`** nas tabelas analíticas e em `vw_extracao_usuario` (sem nome/e-mail/telefone/senha) | — |
| dono do DW | Agendador do ETL | — | escreve em `stg`, `etl`, `dw`, `bi` |
| `bi_leitor` | Ferramenta de BI | — | **só `SELECT`** no schema `bi` |

Todos esses acessos foram testados nos scripts: o `etl_leitor` recebe *permission denied* ao tentar ler `oltp.usuario`, o `api_app` não consegue apagar `log_auditoria` e o `bi_leitor` não lê `dw.*` diretamente.

> As senhas `troque_esta_senha` dos scripts são apenas para ambiente de desenvolvimento; em produção ficam no cofre de segredos do provedor de nuvem.

## 6.4 Backup, retenção e ambientes

| Dado | Backup | Retenção | Observação |
|---|---|---|---|
| `cultiva_oltp` (dados operacionais) | Completo diário + log contínuo (recuperação a um ponto no tempo) | 30 dias de backups | Fonte da verdade — é o mais crítico |
| `oltp.log_auditoria` | Junto com o OLTP | Mínimo 1 ano (a definir com o cliente) | Depois, exportado para armazenamento frio |
| Usuário desligado | — | Dados pessoais anonimizados após o desligamento (LGPD), mantendo o histórico de cultivos | O DW já não guarda nome/e-mail |
| `cultiva_dw` (dw, bi) | Semanal | 4 semanas | Pode ser **reconstruído** do OLTP zerando `etl.watermark` |
| `stg` (staging) | Não precisa | Apagado a cada carga | Dado temporário |

| Ambiente | Dados | Uso |
|---|---|---|
| Desenvolvimento | Fictícios (`02_oltp_dados_exemplo.sql`) | Programação e testes locais |
| Homologação | Fictícios ou anonimizados | Validação com o cliente |
| Produção | Reais | Uso pelas hortas |
