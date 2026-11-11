# Guia de Preparação da Apresentação — Cultiva Plus

Este documento foi elaborado para alinhar e preparar todos os integrantes da equipe da **GDP Projects** para a apresentação do projeto perante os avaliadores.

---

## 1. Visão Geral e Estrutura dos Integrantes

| Integrante | Papel no Projeto | Foco na Apresentação |
| :--- | :--- | :--- |
| **Daniel Alonso Oliveira Soares** | Direção Executiva | Abertura: Visão do negócio, problema das hortas, missão da GDP Projects e fechamento. |
| **João Pablo Silva Pinheiro** | Gestão de Produtos e Projetos | Arquitetura de Dados: Fluxo macro dos 6 sistemas e Matriz de Perfis e Permissões. |
| **Erick Sales Ramos** | Desenvolvimento Back-end e IA | Data Modeling (OLTP 3FN): Entidades de negócio, regras relacionais e integridade ACID. |
| **Antonio Gabriel Gomes Falcão do Nascimento** | Desenvolvimento Back-end e IA | Data Storage e Persistência: PostgreSQL 16, transações, retenção e políticas de backup. |
| **Rafael Jesus Bezerra Cerqueira** | Desenvolvimento Front-end e UX/UI | Plataforma Web e Documentação Física: View models, REST API e inclusão digital via fichas em papel. |
| **Alexandre Roque Pereira Costa** | Infraestrutura e DevOps | Data Security e Data Integration: Criptografia (TLS/Argon2id), pipeline ETL e integridade entre bancos. |
| **Breno Kayan Souza de Azevedo** | Marketing e Atendimento | Data Warehouse e BI: Esquema estrela, indicadores/KPIs para lideranças e geração de impacto social. |

---

## 2. Roteiro Passo a Passo da Apresentação (Pitch de ~10 a 15 min)

### Bloco 1: Introdução e Contexto do Negócio (~2 min) — *Daniel Alonso*
* **Problema:** Hortas comunitárias sofrem com informações descentralizadas, falta de histórico de safras, dificuldade de engajamento voluntário e risco de perda de dados.
* **Solução:** **Cultiva Plus**, plataforma web colaborativa criada pela **GDP Projects** para centralizar a gestão de hortas, canteiros, safras e mutirões.
* **Ponto-chave a destacar:** Não coletamos dados sensíveis e priorizamos acessibilidade com custo enxuto (open-source).

### Bloco 2: Arquitetura dos Sistemas e Perfis (~2 min) — *João Pablo*
* **Os 6 Sistemas:** Web (Front) ➔ API (Back) ➔ Banco Operacional (OLTP) ➔ ETL ➔ Data Warehouse (OLAP) ➔ BI/Relatórios (com apoio de Nuvem e Fichas Físicas).
* **Perfis de Acesso:** Administrador Geral, Líder Comunitário, Membro e Visitante.
* **Ponto-chave a destacar:** O usuário comum nunca toca no banco de dados; toda requisição é filtrada pela API via matriz de permissões.

### Bloco 3: Modelagem Operacional e Documental (~3 min) — *Erick Sales & Rafael Jesus*
* **Banco Operacional (OLTP):** 19 tabelas em 3ª Forma Normal (3FN). Chave para integridade transacional: nenhuma colheita existe sem cultivo; nenhum canteiro estoura sua área limite.
* **Modelo Documental Físico:** Nem todo membro tem smartphone na terra. Fichas impressas de canteiro e mutirão com código único (`HPJ-2026-10-C04`) que o líder lança na plataforma no fim do mês.

### Bloco 4: Armazenamento, Nuvem e Backups (~2 min) — *Antonio Gabriel*
* **Por que PostgreSQL 16 gerenciado?** Robustez relacional, custo zero de licença, facilidade de replicação e suporte a transações ACID completas.
* **Persistência vs. Temporário:** O que é permanente (cadastros, safras, auditoria) vs. volátil (sessão, cache de permissão de 5 min).
* **Backups:** Snapshots diários com Point-in-Time Recovery (PITR) e dumps semanais replicados para Object Storage em outra região geográfica.

### Bloco 5: Segurança e Auditoria (~2 min) — *Alexandre Roque*
* **Classificação dos dados:** Públicos, Internos e Confidenciais. Sem dados sensíveis.
* **Proteção ativa:** Senhas sob hash Argon2id/bcrypt, tráfego 100% HTTPS/TLS 1.3, variáveis de ambiente seguras e SQL parametrizado contra injeções.
* **Auditoria:** Tabela imutável (`log_auditoria`) gravando o antes e o depois em JSONB de ações críticas.

### Bloco 6: Integração de Dados, ETL e BI (~3 min) — *Alexandre Roque & Breno Kayan*
* **Integração Web ↔ API:** Síncrona via REST API (JSON) com token seguro e chave de idempotência para evitar cliques duplos.
* **Integração OLTP ➔ DW:** Batch diário às 02:00 via `postgres_fdw`. Extração incremental por marca d'água (`atualizado_em`), **sem PII (LGPD)** e com SCD Tipo 2 na dimensão de hortas.
* **Data Warehouse & BI:** Esquema estrela (5 fatos e 6 dimensões) alimentando views semânticas (`bi.vw_*`) com KPIs de produtividade (kg), taxa de doação e horas de voluntariado.

---

## 3. Perguntas Frequentes da Banca e Como Responder

#### P1: "Por que ter dois bancos de dados (OLTP e DW) para um projeto comunitário?"
> **Resposta:** *"O banco operacional (OLTP) foi desenhado em 3FN para garantir transações instantâneas e sem redundância no dia a dia. Se colocássemos relatórios históricos pesados de safras e indicadores para rodar nele, a navegação dos moradores ficaria lenta. O Data Warehouse isola o processamento analítico, anonimiza dados conforme a LGPD e já entrega os dados pré-agrupados em modelo estrela para o BI."*

#### P2: "Como vocês garantem que os dados não são duplicados se o morador clicar duas vezes ou o ETL rodar mais de uma vez?"
> **Resposta:** *"Temos duas barreiras: na Web/API, usamos chave de idempotência (`Idempotency-Key`) e constraints `UNIQUE` no banco operacional. No Data Warehouse, a carga utiliza `INSERT ... ON CONFLICT DO UPDATE` (Upsert); ou seja, mesmo que o ETL seja executado várias vezes na mesma janela, ele só atualiza a linha já existente sem criar duplicatas."*

#### P3: "E se a internet cair na comunidade durante um mutirão?"
> **Resposta:** *"Criamos a Gestão Documental Física exatamente para isso. Cada horta possui armário de aço com fichas padronizadas de canteiro e lista de presença. Ao restabelecer a conexão ou ao término do mês, o líder digitaliza e lança os dados na plataforma web."*

#### P4: "Como o sistema trata a LGPD caso um participante peça a exclusão de sua conta?"
> **Resposta:** *"Adotamos o direito ao esquecimento com anonimização. Os dados pessoais (nome, e-mail, telefone) são substituídos por identificadores anônimos e a senha é apagada. O histórico numérico de mutirões e colheitas permanece no sistema, garantindo a integridade dos relatórios sem que seja possível rastrear a pessoa."*
