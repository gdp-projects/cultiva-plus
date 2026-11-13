# Data Storage (Armazenamento de Dados)

O armazenamento do **Cultiva Plus** é desenhado de forma prática e sem complicação técnica excessiva: tudo o que é transacional e analítico roda em **PostgreSQL**, com backups automáticos na nuvem e suporte a fichas físicas no campo.

---

## 1. Onde os dados ficam guardados?

| Componente | Tecnologia | Para que serve? | Volume Esperado |
| :--- | :--- | :--- | :--- |
| **Banco Operacional** | PostgreSQL 16 gerenciado | Guarda os dados diários do sistema (usuários, canteiros, cultivos, colheitas, mutirões). | Pequeno a médio (~10 a 50 MB no início). |
| **Data Warehouse** | PostgreSQL 16 (banco analítico) | Guarda o histórico consolidado de produção e participação, livre de dados pessoais. | < 500 MB em 3 anos. |
| **Backups em Nuvem** | Object Storage (AWS S3 / Google Cloud) | Guarda as cópias de segurança semanais e fotos das fichas físicas. | 2 a 15 GB. |
| **Navegador e API** | Cookies seguros e cache em memória | Sessão de login do usuário e preferências de tela. | Volátil (descartável). |
| **Arquivo Físico** | Armário de aço na horta comunitária | Fichas em papel para mutirões e canteiros (inclusão para quem não tem celular na horta). | Fichas de papel arquivadas por horta. |

---

## 2. Como os dados são gravados e mantidos?

* **Inserções e Atualizações:** Feitas exclusivamente através da API. Toda ação importante (como registrar uma colheita ou cadastrar um mutirão) é validada antes de ser salva.
* **Exclusões Seguras (Sem perda de histórico):** Não apagamos registros importantes do banco. Em vez de excluir, o sistema apenas desativa o status (ex.: canteiro "inativo", tarefa "cancelada"). Assim, o histórico de produção do passado nunca é perdido.
* **Privacidade (LGPD):** Se um voluntário pedir para sair, removemos os dados pessoais (nome, e-mail, telefone) e mantemos apenas os números das colheitas e presenças sem identificar a pessoa.

---

## 3. Backups e Recuperação em Caso de Falhas

Para garantir que nenhuma informação se perca:

1. **Backup Diário Automático:** O provedor em nuvem tira uma cópia diária do banco e grava alterações contínuas (permite voltar o banco a qualquer minuto recente).
2. **Backup Semanal Externo:** Uma cópia completa é exportada e guardada com segurança em outro servidor em nuvem.
3. **Se o servidor cair:** O banco gerenciado sobe automaticamente uma máquina reserva.
4. **Se algo for alterado por engano:** Restauramos o backup do minuto anterior em ambiente de teste e copiamos apenas os dados corrigidos.
5. **Se a internet cair no dia do mutirão:** Os voluntários assinam a lista de presença física em papel, e o líder lança no sistema quando a conexão voltar.
