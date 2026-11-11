# Data Storage (Armazenamento de Dados)

O **Cultiva Plus** armazena seus dados em dois bancos relacionais principais: o banco operacional (`cultiva_oltp`), que a plataforma utiliza em tempo real, e o Data Warehouse (`cultiva_dw`), que consolida o histórico utilizado nos relatórios gerenciais e dashboards de BI. Além deles, há armazenamento de objetos em nuvem para backups, persistência temporária no navegador e API, e gestão física documental das fichas das hortas.

---

## 1. Armazenamento

### 1.1 Banco Operacional (`cultiva_oltp`)

* **Tecnologia:** PostgreSQL 16 gerenciado (ex.: Amazon RDS, Google Cloud SQL, Azure Database ou Supabase).
* **Motivo da escolha:**
  * Gratuito e open-source (baixo custo operacional).
  * Modelo relacional ideal para dados estruturados e conectados (Horta → Canteiro → Cultivo → Colheita).
  * Conformidade total com transações ACID (garante que operações interdependentes sejam gravadas por completo ou descartadas).
  * Regras de integridade (`CHECK`, `FOREIGN KEY`, `NOT NULL`) enforced no banco.
  * Portabilidade entre diferentes provedores em nuvem.
* **Dados armazenados:** Usuários, permissões, comunidades, hortas, canteiros, espécies, cultivos, colheitas, mutirões, presenças, tarefas, dicas, avisos, solicitações de entrada e logs de auditoria. Único local com dados pessoais cadastrais (com senha armazenada sob hash irreversível).
* **Organização dos dados:** 19 tabelas normalizadas em 3ª Forma Normal (3FN), com chaves primárias e marca d'água (`atualizado_em`) para extração incremental.
* **Volume esperado:** Volume estimado leve no piloto (~poucos MB) alcançando dezenas de MB no terceiro ano, dimensionado para suportar auditoria e histórico operacional.
* **Estratégia de crescimento:**
  * Escala vertical simples (CPU/RAM gerenciada).
  * Pool de conexões (ex.: PgBouncer).
  * Read Replicas para descarregar consultas de leitura caso necessário.
  * Particionamento por ano da tabela `log_auditoria`.

---

### 1.2 Data Warehouse (`cultiva_dw`)

* **Tecnologia:** PostgreSQL 16 (banco isolado, podendo iniciar na mesma instância e migrar para servidor analítico dedicado conforme volume).
* **Motivo da escolha:** Simplicidade de suporte (mesma stack), custo zero de licenciamento e possibilidade de integração nativa via `postgres_fdw`.
* **Dados armazenados:** Dados analíticos anonimizados (sem PII): histórico de colheitas, ciclos de cultivo, evolução mensal de ocupação de canteiros, frequência em mutirões e conclusão de tarefas.
* **Organização dos dados:** Estruturado em 4 schemas lógicos:
  * `stg`: Staging area para recepção temporária das cargas.
  * `dw`: Modelo dimensional estrela (5 fatos e 6 dimensões, incluindo SCD Tipo 2 para histórico de hortas).
  * `bi`: Views da camada semântica consumidas pelos relatórios.
  * `etl`: Controle e histórico das execuções de sincronização.
* **Volume esperado:** Estimado em < 50 MB no 1º ano e < 500 MB em 3 anos.

---

### 1.3 Armazenamento na Nuvem para Backups e Arquivos

* **Tecnologia:** Object Storage (Amazon S3 ou Google Cloud Storage).
* **Dados armazenados:** Dumps semanais compactados e criptografados dos bancos, exportações antigas de auditoria e digitalizações (PDF/imagens) das fichas de campo em papel.
* **Volume e retenção:** Entre 2 GB e 25 GB, com políticas de retenção e imutabilidade de arquivos (*WORM/Object Lock*).

---

### 1.4 Navegador e API (Dados Temporários / Cache)

* **Navegador:** Cookies seguros (`HttpOnly`, `SameSite`) para token de sessão e preferências locais de interface (filtros e horta ativa).
* **API:** Cache em memória das matrizes de permissão por usuário (TTL de 5 minutos).

---

### 1.5 Arquivo Físico

* Fichas de papel padronizadas (Canteiros e Mutirões) arquivadas na sede local de cada horta comunitária em armários de aço, garantindo inclusão digital de membros sem smartphone. Sincronizadas mensalmente na plataforma web e digitalizadas para a nuvem.

---

## 2. Persistência

### 2.1 Dados que precisam ser persistidos
Todos os registros de negócio essenciais e histórico das hortas: cadastros de usuários, vínculos, comunidades, hortas, canteiros, espécies, cultivos, colheitas, mutirões, presenças, tarefas, solicitações, dicas, avisos, logs de auditoria, além do histórico analítico e marcas d'água (`etl.watermark`) do Data Warehouse.

### 2.2 Dados temporários (voláteis)
* Sessão do usuário e preferências no browser.
* Cache de permissões na API.
* Tabelas de preparação (`stg.*`) do Data Warehouse (descartadas a cada ciclo de ETL).
* Logs operacionais e técnicos de depuração HTTP da API (retenção curta de 30 dias).

### 2.3 Operações de Inserção, Atualização e Exclusão
* **Inserção:** Validação na API e gravação via transação no OLTP.
* **Atualização:** Alteração controlada com trigger automática atualizando `atualizado_em` para consumo incremental pelo ETL.
* **Exclusão:** Adoção prioritária de *Soft Delete* (exclusão lógica via alteração de status para `INATIVO` ou `CANCELADA`), preservando integridade referencial histórica.
* **Direito ao Esquecimento (LGPD):** Anonimização com substituição de dados identificáveis por valores padrão e remoção de hash de senha.
* **Auditoria:** Gravação append-only com bloqueio de update/delete.

---

## 3. Backup e Recuperação

* **Snapshots Diários Automatizados:** Gerenciados pelo provedor de nuvem com recuperação pontual (*Point-in-Time Recovery - PITR*).
* **Exports Semanais Criptografados:** Dumps lógicos replicados para Object Storage em região geográfica secundária.
* **Testes de Restauração:** Validação trimestral programada dos backups em ambiente isolado.
* **Estratégias de Recuperação:**
  * Queda de instância: Failover automático no banco gerenciado.
  * Exclusão acidental: Restauração PITR de snapshot para instância auxiliar e extração seletiva dos registros.
  * Desastre analítico (DW): Reprocessamento completo a partir do OLTP zerando a tabela de marcas d'água.

---

## 4. Transacional (OLTP) vs. Analítico (OLAP)

* **O que fica exclusivo no OLTP:** Dados pessoais (LGPD), senhas criptografadas, logs de auditoria detalhados e textos livres (comentários, descrições de avisos e dicas).
* **O que segue para o DW:** Indicadores numéricos, dados temporais e chaves de contexto para cruzamento multidimensional (safras por estação, produtividade por m², horas de voluntariado e taxas de conclusão de tarefas).
* **Atendimento às necessidades operacionais:** Respostas em milissegundos para requisições concorrentes, integridade relacional estrita e isolamento das consultas analíticas pesadas.
* **Atendimento às necessidades analíticas:** Respostas otimizadas para agregação com esquema estrela desnormalizado, chaves substitutas (*Surrogate Keys*) e camada semântica com métricas prontas.
