# 5. Sistema de BI & Relatórios

O módulo de BI & Relatórios transforma os dados históricos do Data Warehouse em painéis visuais simples e objetivos para os líderes comunitários e para a equipe executiva da GDP Projects.

---

## 5.1 Camada Semântica (Views Analíticas)

Para que ferramentas visuais de relatórios (como Metabase, Power BI ou Looker Studio) não precisem fazer joins complexos nem calcular fórmulas repetidamente, o schema `bi` disponibiliza views com cálculos pré-configurados:

| View | O que calcula? | Para quem é útil? |
| :--- | :--- | :--- |
| **`bi.vw_producao_mensal`** | Total de quilos colhidos por horta, mês e destino (consumo vs. doação). | Prestação de contas comunitária e impacto social. |
| **`bi.vw_ocupacao_mensal`** | Percentual de área dos canteiros utilizada a cada mês. | Identificar canteiros ociosos precisando de novos plantios. |
| **`bi.vw_desempenho_safra`** | Rendimento médio por espécie (kg/m²) e tempo médio de ciclo. | Planejamento das melhores épocas de cultivo. |
| **`bi.vw_engajamento_mutiroes`** | Frequência de voluntários e total de horas comunitárias dedicadas. | Acompanhar a mobilização e força de trabalho da horta. |
| **`bi.vw_desempenho_tarefas`** | Taxa de conclusão de tarefas no prazo. | Organização operacional dos voluntários. |

---

## 5.2 Painéis Principais

1. **Painel Executivo (GDP Projects):**
   * Visão geral de todas as hortas cadastradas.
   * Total acumulado de quilos colhidos e percentual destinado a doações.
   * Volume de voluntários mobilizados no ano.

2. **Painel de Gestão Local (Líder da Horta):**
   * Visão restrita à sua respectiva horta comunitária.
   * Calendário de safras atuais e canteiros liberados para plantio.
   * Frequência nos mutirões e tarefas pendentes.

---

## 5.3 Segurança e Acesso aos Relatórios

* **Usuário Restrito (`bi_leitor`):** A ferramenta de relatórios conecta com permissão exclusiva de leitura no schema `bi`, sem acesso de escrita.
* **Isolamento por Horta:** Os líderes comunitários têm acesso restrito aos dados da horta que coordenam, garantindo privacidade e organização.
