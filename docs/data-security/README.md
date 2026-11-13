# Data Security (Segurança de Dados)

O **Cultiva Plus** adota uma abordagem de segurança direta e eficiente, focada no que é essencial para proteger dados comunitários sem complicar o desenvolvimento ou a manutenção.

---

## 1. Classificação dos Dados

Separamos as informações do sistema em 3 níveis claros:

* **Públicas:** Informações básicas das hortas (nome, bairro), dias de visitação e dicas de cultivo.
* **Internas:** Planejamento de canteiros, agendamento de mutirões e relatórios de safra.
* **Confidenciais:** Contatos de voluntários (e-mail, telefone) e senhas de acesso.
* *Nota:* O sistema **não coleta dados sensíveis** (como saúde, religião, biometria ou dados bancários).

---

## 2. Como Protegemos os Dados na Prática

1. **Senhas Seguras:** Nenhuma senha é salva como texto puro. Usamos algoritmos modernos de hash (como **Argon2id** ou **bcrypt**), tornando impossível descobrir a senha original mesmo em caso de vazamento.
2. **Conexões Criptografadas:** Toda a comunicação entre o navegador e o servidor usa **HTTPS/TLS**, impedindo que dados sejam interceptados na rede.
3. **Controle de Acesso Inteligente:**
   * Moradores só alteram suas próprias tarefas e presenças.
   * Líderes só gerenciam a horta pela qual são responsáveis.
   * Acesso aos bancos de dados é restrito a credenciais técnicas de serviço (`api_app`, `etl_leitor`).
4. **Proteção contra Injeção de SQL:** As consultas ao banco usam comandos parametrizados preparados, impedindo qualquer tentativa de ataque via formulário.
5. **Histórico de Auditoria:** Ações críticas (como aprovação de membro ou exclusão de canteiro) são registradas automaticamente em uma tabela de log (`log_auditoria`), gravando quem fez, quando e o que mudou.

---

## 3. Resumo dos Riscos e Cuidados

| Risco | Como o sistema se protege? |
| :--- | :--- |
| **Vazamento de senha** | Senhas protegidas com hash criptográfico irreversível. |
| **Acesso a dados de outra horta** | API confere o vínculo do líder antes de autorizar qualquer ação. |
| **Perda acidental de dados** | Backups diários em nuvem e exclusão lógica (*soft delete*). |
| **Exposição de senhas do banco** | Credenciais salvas em variáveis de ambiente protegidas, nunca no código. |
