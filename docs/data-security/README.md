# Data Security (Segurança de Dados)

Medidas e diretrizes adotadas pelo **Cultiva Plus** para salvaguardar a confidencialidade, integridade e disponibilidade dos dados contra acessos indevidos, vazamentos, perda e adulteração.

---

## 1. Identificação e Classificação dos Dados

Os dados são classificados conforme sua criticidade e visibilidade:

* **Públicos:** Dados cadastrais das hortas abertas, localização comunitária de livre acesso e catálogo de dicas de cultivo.
* **Internos:** Planejamento de tarefas, divisão de canteiros, histórico operacional e relatórios internos da horta.
* **Confidenciais:** Dados de contato de moradores (e-mail, telefone), senhas e solicitações de ingresso em aprovação.
* **Sensíveis:** O Cultiva Plus não coleta nem armazena dados pessoais sensíveis (como raça, dados de saúde, religião, biometria ou dados bancários).

---

## 2. Controle de Acesso e Princípio do Menor Privilégio

O acesso aos dados é restrito conforme os perfis operacionais da aplicação:

1. **Administrador Geral:** Gestão completa de hortas, configurações e logs de auditoria.
2. **Responsável pela Horta:** Gestão restrita aos canteiros, cultivos e voluntários da sua própria horta comunitária.
3. **Membro Participante:** Consulta de canteiros da horta onde tem vínculo, confirmação em mutirões e conclusão de tarefas designadas a si.
4. **Visitante:** Acesso estritamente somente leitura aos dados públicos e formulário de inscrição.
5. **Serviços Internos (Service Accounts):**
   * `api_app`: Leitura e escrita transacional no banco operacional.
   * `etl_leitor`: Acesso somente leitura às tabelas operacionais e views de extração sem PII.
   * `bi_leitor`: Acesso somente leitura às views semânticas do schema `bi` no DW.

---

## 3. Autenticação e Autorização

* **Credenciais e Hash:** Autenticação via e-mail e senha. Armazenamento de credenciais protegido com algoritmos criptográficos modernos de hash com salt (Argon2id ou bcrypt).
* **Autorização na Camada de Negócio:** Centralizada no Back-end (API), que valida o token de sessão e o escopo de permissão para cada recurso antes de qualquer consulta ao banco.
* **Proteção contra Força Bruta:** Limitação de taxa de requisições (*Rate Limiting*) nas rotas de login e registro de falhas sucessivas.

---

## 4. Proteção dos Dados em Todo o Ciclo de Vida

* **Em Trânsito:** Todo o tráfego externo entre navegadores e a API utiliza criptografia obrigatória via HTTPS/TLS 1.3. O tráfego interno com os bancos de dados utiliza conexões com TLS e certificados validados.
* **Em Repouso:** Bancos gerenciados com criptografia nativa no disco (*Storage Encryption* - AES-256) e backups em Object Storage criptografados no lado do servidor.
* **No Processamento:** Validação estrita de entradas, prevenção contra SQL Injection via consultas parametrizadas em ORM/drivers nativos e sanitização de dados.
* **Gestão de Segredos:** Credenciais de banco, chaves de API e tokens mestres mantidos exclusivamente em variáveis de ambiente protegidas e serviços de Secret Management em nuvem (nunca versionadas no código-fonte).
* **Proteção Analítica:** Mascaramento e eliminação total de PII (dados identificáveis) nas views consumidas pelo Data Warehouse e BI.

---

## 5. Auditoria e Rastreabilidade

O Cultiva Plus conta com trilha de auditoria append-only (`oltp.log_auditoria`) registrando:
* Eventos de login e tentativas bloqueadas;
* Concessão e revogação de papéis/permissões;
* Criação, modificação e inativação de hortas e canteiros;
* Aprovação e recusa de solicitações de novos membros;
* Ações administrativas sensíveis, armazenando o estado antes e depois em JSONB.

---

## 6. Matriz de Riscos de Segurança e Mitigações

| Risco Identificado | Severidade | Medida de Mitigação |
| :--- | :--- | :--- |
| **Vazamento de Senhas** | Alta | Hashing com Argon2id/bcrypt e políticas fortes de senha. |
| **Acesso Indevido a Dados de Outra Horta** | Alta | Validação estrita de escopo de horta por middleware na API. |
| **Interceptação de Dados em Trânsito** | Alta | Imposição de HTTPS/TLS 1.3 com HSTS. |
| **Exclusão Acidental de Registros** | Média/Alta | Adoção de *Soft Delete* e backups diários com Point-in-Time Recovery. |
| **Exposição de Credenciais no Repositório** | Alta | Uso de variáveis de ambiente e arquivos `.env` ignorados no git. |
| **Injeção de SQL (SQL Injection)** | Alta | Uso exclusivo de queries parametrizadas / Prepared Statements. |
| **Alteração Não Autorizada em Registros** | Alta | Matriz de permissões na API e log de auditoria imutável. |

---

## 7. Status de Implementação

* **Medidas Planejadas e Especificadas:** Criptografia ponta a ponta, hashing Argon2id, RBAC em API REST, isolamento de schemas, pipelines seguros de ETL e logs de auditoria estruturados.
* **Fase Atual:** Projeto em fase de modelagem e arquitetura conceitual com scripts SQL testados em ambiente de desenvolvimento. Implementação física definitiva em produção prevista para a fase de deploy da infraestrutura.
