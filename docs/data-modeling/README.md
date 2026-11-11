# Data Modeling (Modelagem de Dados)

A [Arquitetura de Dados](../data-architecture/README.md) mostrou a **visão macro**: quais sistemas existem e como os dados circulam entre eles. Este módulo detalha a **visão estrutural e detalhada**: o **modelo lógico de cada um desses sistemas**, os modelos físicos relacionais e dimensionais, e a modelagem fora do meio digital.

Todos os modelos de banco foram implementados em SQL (PostgreSQL 16) e **testados de ponta a ponta** com dados fictícios: criação dos bancos → carga de exemplo → ETL → consultas do BI.

---

## 1. Coerência com a Data Architecture

Cada sistema levantado na Arquitetura possui seu respectivo modelo:

| # | Sistema | Tem banco próprio? | Modelo lógico / Descrição | Documento | Script SQL |
|---|---|---|---|---|---|
| 1 | Plataforma Web (Front-end) | Não | Sessão, *view models* das telas e formulários | [01-plataforma-web.md](01-plataforma-web.md) | — |
| 2 | Sistema Administrativo (API) | Não (único acesso ao OLTP) | Recursos/DTOs, matriz de permissões e regras de negócio | [02-sistema-administrativo-api.md](02-sistema-administrativo-api.md) | — |
| 3 | Banco Operacional (OLTP) | **Sim** — `cultiva_oltp` | Modelo **relacional normalizado (3FN)**, 19 entidades | [03-banco-operacional-oltp.md](03-banco-operacional-oltp.md) | [`01`](sql/01_oltp_schema.sql), [`02`](sql/02_oltp_dados_exemplo.sql) |
| 4 | Data Warehouse (OLAP) | **Sim** — `cultiva_dw` | Modelo **dimensional** (5 fatos, 6 dimensões) + ETL | [04-data-warehouse-olap.md](04-data-warehouse-olap.md) | [`03`](sql/03_dw_schema.sql), [`04`](sql/04_etl_oltp_para_dw.sql) |
| 5 | Sistema de BI & Relatórios | Usa o schema `bi` do DW | Camada semântica (views) + catálogo de painéis/KPIs e acesso por horta | [05-bi-relatorios.md](05-bi-relatorios.md) | [`05`](sql/05_bi_camada_semantica.sql), [`06`](sql/06_consultas_exemplo.sql) |
| 6 | Infraestrutura e Nuvem | Hospeda os dois bancos | Implantação, usuários de banco, controle de cargas, backup e retenção | [06-infraestrutura-nuvem.md](06-infraestrutura-nuvem.md) | — |
| 7 | Arquivo Físico | Papel / Armário | Gestão documental analógica das fichas de canteiro e listas de presença | [modelo-gestao-documental-fisica.md](modelo-gestao-documental-fisica.md) | — |

---

## 2. Entidades Principais

Modelamos **12 entidades de negócio** e **7 entidades de apoio** (total de 19 tabelas no OLTP).

| # | Entidade de negócio | O que representa | Necessidade do negócio atendida |
|---|---|---|---|
| 1 | `USUARIO` | Pessoa cadastrada na plataforma | Dados dos participantes e membros |
| 2 | `COMUNIDADE` | Associação, escola ou condomínio que mantém hortas | Conectar moradores, comunidades e responsáveis |
| 3 | `HORTA` | Horta comunitária | Informações das hortas |
| 4 | `MEMBRO_HORTA` | Vínculo usuário × horta e o papel (líder/membro) | Matriz de Perfis e Permissões |
| 5 | `SOLICITACAO_ENTRADA` | Pedido para participar de uma horta | Aprovação e gestão de novos membros |
| 6 | `CANTEIRO` | Espaço de cultivo dentro da horta | Informações dos espaços disponíveis |
| 7 | `ESPECIE` | Catálogo de plantas | Informações sobre os cultivos |
| 8 | `CULTIVO` | Um plantio (safra) em um canteiro | Acompanhamento dos cultivos |
| 9 | `COLHEITA` | Cada colheita e o destino da produção | Histórico de safras e produtividade |
| 10 | `MUTIRAO` | Evento coletivo de trabalho | Atividades comunitárias realizadas |
| 11 | `PRESENCA_MUTIRAO` | Confirmação e presença em mutirão | Participação e engajamento dos membros |
| 12 | `TAREFA` | Atividade atribuída a um membro | Organização das atividades rotineiras |

**Apoio:** `PERFIL`, `PERMISSAO`, `PERFIL_PERMISSAO` (controle de acesso), `TIPO_TAREFA` (padronização de atividades), `DICA_CULTIVO` (compartilhamento de conhecimento), `AVISO` (comunicação interna) e `LOG_AUDITORIA` (rastreabilidade e segurança).

---

## 3. Matriz Entidade × Sistema

| Entidade | 1. Web | 2. API | 3. OLTP | 4. DW | 5. BI |
|---|---|---|---|---|---|
| `USUARIO` | `FormCadastro`, `UsuarioLogado` | valida, hash senha | `oltp.usuario` | `dim_membro` (sem PII) | contagem de voluntários |
| `COMUNIDADE` + `HORTA` | `HortaPublicaVM` (mapa) | `HortaDTO` | `oltp.comunidade`, `oltp.horta` | `dim_horta` (SCD2) | filtro em todos os painéis |
| `MEMBRO_HORTA` | `VinculoHorta` | checagem de permissão | `oltp.membro_horta` | — | `bi.acesso_horta` (segurança) |
| `CANTEIRO` | `CanteiroVM` | `CanteiroDTO` | `oltp.canteiro` | `dim_canteiro` | `vw_ocupacao_mensal` |
| `ESPECIE` | `CultivoVM.especie` | catálogo | `oltp.especie` | `dim_especie` | `vw_desempenho_safra` |
| `CULTIVO` | `FormCultivo`, `CultivoVM` | regra de área do canteiro | `oltp.cultivo` | `fato_cultivo`, `fato_ocupacao_mensal` | `vw_desempenho_safra` |
| `COLHEITA` | `FormColheita` | `ColheitaDTO` | `oltp.colheita` | `fato_colheita` | `vw_producao_mensal` |
| `MUTIRAO` + `PRESENCA_MUTIRAO` | `MutiraoVM`, `FormMutirao` | controle de vagas | `oltp.mutirao`, `oltp.presenca_mutirao` | `fato_participacao_mutirao` | `vw_engajamento_mutiroes` |
| `TAREFA` + `TIPO_TAREFA` | `TarefaVM`, `FormTarefa` | conclusão por responsável | `oltp.tarefa`, `oltp.tipo_tarefa` | `fato_tarefa`, `dim_tipo_tarefa` | `vw_desempenho_tarefas` |

---

## 4. Contribuição do Data Modeling para o Negócio

O Data Modeling complementa a arquitetura ao definir como as informações são organizadas e relacionadas dentro do banco de dados operacional e do Data Warehouse.
No Cultiva Plus, relacionamentos bem definidos garantem integridade referencial estrita (ex.: um cultivo vinculado ao seu canteiro e horta correspondente, uma colheita vinculada à safra).

A modelagem reduz duplicações desnecessárias, apoia a tomada de decisões através de esquemas dimensionais limpos para BI e atende aos objetivos de acessibilidade, simplicidade e agilidade da **GDP Projects**.

---

## 5. Estrutura dos Arquivos

```text
docs/data-modeling/
├── README.md                          # Este arquivo (visão geral, entidades e coerência)
├── 01-plataforma-web.md               # Sistema 1: Front-end e View Models
├── 02-sistema-administrativo-api.md   # Sistema 2: Back-end, DTOs e Permissões
├── 03-banco-operacional-oltp.md       # Sistema 3: Modelo relacional 3FN
├── 04-data-warehouse-olap.md          # Sistema 4: Esquema estrela e rotina de ETL
├── 05-bi-relatorios.md                # Sistema 5: Camada semântica e KPIs
├── 06-infraestrutura-nuvem.md         # Sistema 6: Ambientes, acessos e backups
├── modelo-gestao-documental-fisica.md # Gestão de documentos analógicos (fichas em papel)
└── sql/
    ├── 01_oltp_schema.sql             # DDL do banco operacional OLTP
    ├── 02_oltp_dados_exemplo.sql      # Seed com dados de teste
    ├── 03_dw_schema.sql               # DDL de staging e esquema estrela
    ├── 04_etl_oltp_para_dw.sql        # Procedimentos e extração incremental
    ├── 05_bi_camada_semantica.sql     # Views semânticas do BI
    └── 06_consultas_exemplo.sql       # Consultas analíticas de negócio
```

---

## 6. Como Executar os Scripts SQL (PostgreSQL 16)

```bash
cd docs/data-modeling/sql
createdb cultiva_oltp
createdb cultiva_dw

psql -d cultiva_oltp -f 01_oltp_schema.sql
psql -d cultiva_oltp -f 02_oltp_dados_exemplo.sql

psql -d cultiva_dw -f 03_dw_schema.sql
psql -d cultiva_dw -f 04_etl_oltp_para_dw.sql
psql -d cultiva_dw -f 05_bi_camada_semantica.sql

psql -d cultiva_dw -c "CALL etl.executar_carga();"
psql -d cultiva_dw -f 06_consultas_exemplo.sql
```
