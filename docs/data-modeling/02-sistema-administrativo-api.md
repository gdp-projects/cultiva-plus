# 2. Sistema Administrativo (Back-end / API)

> **Sistema da Entrega 1:** Sistema Administrativo (Back-end / API)
> **Responsabilidade:** regras de negócio e controle da Matriz de Perfis e Permissões.
> **Armazena dados?** Não tem banco próprio: é a **única porta de entrada** do Banco Operacional. Toda leitura/escrita no OLTP passa por aqui, usando o usuário de banco `api_app`.

O modelo lógico da API descreve **os recursos que ela expõe**, **como cada recurso é gravado no OLTP** e **como a permissão é verificada**.

---

## 2.1 Modelo lógico — recursos (DTOs) e entidades do OLTP

```mermaid
classDiagram
    direction LR
    class TokenClaims {
        +sub : idUsuario
        +perfilGlobal : string
        +hortas : VinculoClaim[]
        +exp : datetime
    }
    class VinculoClaim {
        +idHorta : number
        +perfil : LIDER_HORTA | MEMBRO
    }
    class HortaDTO {
        +id, nome, bairro, cidade
        +latitude, longitude
        +status, visibilidade
    }
    class CanteiroDTO {
        +id, idHorta, codigo
        +areaM2, status
    }
    class CultivoDTO {
        +id, idCanteiro, idEspecie
        +dataPlantio, dataPrevistaColheita
        +areaOcupadaM2, status
    }
    class ColheitaDTO {
        +id, idCultivo, dataColheita
        +quantidade, unidade, pesoKg, destino
    }
    class MutiraoDTO {
        +id, idHorta, titulo, inicio, fim
        +vagas, vagasRestantes, status
    }
    class PresencaDTO {
        +idMutirao, idUsuario, status
    }
    class TarefaDTO {
        +id, idHorta, tipo, titulo
        +idResponsavel, prazo, status
    }
    class SolicitacaoDTO {
        +id, idUsuario, idHorta
        +status, motivoRecusa
    }
    TokenClaims "1" --> "0..*" VinculoClaim
    HortaDTO "1" --> "0..*" CanteiroDTO
    CanteiroDTO "1" --> "0..*" CultivoDTO
    CultivoDTO "1" --> "0..*" ColheitaDTO
    HortaDTO "1" --> "0..*" MutiraoDTO
    MutiraoDTO "1" --> "0..*" PresencaDTO
    HortaDTO "1" --> "0..*" TarefaDTO
    HortaDTO "1" --> "0..*" SolicitacaoDTO
```

## 2.2 Recursos × entidades × permissões

| Recurso (endpoint) | Método | Grava/lê no OLTP | Permissão exigida | Perfis |
|---|---|---|---|---|
| `/hortas/publicas` | GET | `HORTA` (só públicas e ativas, sem dados de membros) | — (público) | Visitante e todos |
| `/usuarios` | POST | `USUARIO` (status `PENDENTE`) | — (público) | Visitante |
| `/hortas/{id}/solicitacoes` | POST | `SOLICITACAO_ENTRADA` | — (autenticado) | Usuário pendente/membro |
| `/solicitacoes/{id}/aprovar` | POST | `SOLICITACAO_ENTRADA`, `MEMBRO_HORTA`, `USUARIO`, `LOG_AUDITORIA` | `MEMBRO_APROVAR` | Líder (própria horta), Admin |
| `/membros/{id}/perfil` | PUT | `MEMBRO_HORTA`, `LOG_AUDITORIA` | `PERMISSAO_GERENCIAR` | Admin |
| `/hortas` | POST/PUT/DELETE | `HORTA`, `COMUNIDADE` | `HORTA_GERENCIAR` | Admin |
| `/hortas/{id}/canteiros` | POST/PUT | `CANTEIRO` | `CANTEIRO_GERENCIAR` | Líder, Admin |
| `/canteiros/{id}/cultivos` | POST/PUT | `CULTIVO`, `CANTEIRO.status` | `CULTIVO_GERENCIAR` | Líder, Admin |
| `/cultivos/{id}/colheitas` | POST | `COLHEITA` (e `CULTIVO.status` se for a última) | `CULTIVO_GERENCIAR` | Líder, Admin |
| `/hortas/{id}/mutiroes` | POST/PUT | `MUTIRAO` | `MUTIRAO_GERENCIAR` | Líder, Admin |
| `/mutiroes/{id}/presenca` | PUT | `PRESENCA_MUTIRAO` (a do próprio usuário) | `MUTIRAO_CONFIRMAR` | Membro, Líder, Admin |
| `/hortas/{id}/tarefas` | POST | `TAREFA` | `TAREFA_ATRIBUIR` | Líder, Admin |
| `/tarefas/{id}/concluir` | POST | `TAREFA` (só se for o responsável) | `TAREFA_CONCLUIR` | Membro, Líder, Admin |
| `/hortas/{id}/avisos/{id}/moderar` | PUT | `AVISO`, `LOG_AUDITORIA` | `AVISO_MODERAR` | Líder, Admin |
| `/dicas` | GET | `DICA_CULTIVO`, `ESPECIE` | `DICA_LER` | Membro, Líder, Admin |
| `/auditoria` | GET | `LOG_AUDITORIA` | `AUDITORIA_VER` | Admin |

## 2.3 Como a permissão é verificada

As permissões **não estão escritas no código**: ficam nas tabelas `PERFIL`, `PERMISSAO` e `PERFIL_PERMISSAO` do OLTP. A API carrega essa matriz em cache e decide assim:

```mermaid
flowchart TD
    A[Requisição com token] --> B{Token válido?}
    B -- não --> X1[401 Não autenticado]
    B -- sim --> C{perfilGlobal = ADMIN_GERAL?}
    C -- sim --> P[Permitido]
    C -- não --> D[Descobre a horta do recurso<br/>ex.: tarefa -> id_horta]
    D --> E{Usuário tem MEMBRO_HORTA<br/>ATIVO nessa horta?}
    E -- não --> X2[403 Sem acesso a esta horta]
    E -- sim --> F{PERFIL_PERMISSAO do papel<br/>contém a permissão exigida?}
    F -- não --> X3[403 Permissão insuficiente]
    F -- sim --> P
    P --> G[Executa no OLTP em transação<br/>+ grava LOG_AUDITORIA quando sensível]
```

## 2.4 Regras de negócio que a API garante (além das restrições do banco)

| Regra | Entidades envolvidas |
|---|---|
| Não cadastrar cultivo em canteiro `INATIVO` ou `EM_DESCANSO` | `CANTEIRO`, `CULTIVO` |
| A soma de `area_ocupada_m2` dos cultivos ativos não pode passar da área do canteiro | `CANTEIRO`, `CULTIVO` |
| Ao iniciar um cultivo, o canteiro passa a `OCUPADO`; ao encerrar o último, volta a `DISPONIVEL` | `CULTIVO`, `CANTEIRO` |
| Não confirmar presença se o mutirão não tem mais vagas ou já passou | `MUTIRAO`, `PRESENCA_MUTIRAO` |
| Um membro só conclui tarefa da qual é responsável; o líder conclui qualquer tarefa da sua horta | `TAREFA`, `MEMBRO_HORTA` |
| Líder só aprova solicitações da(s) horta(s) que lidera | `SOLICITACAO_ENTRADA`, `MEMBRO_HORTA` |
| Aprovações, mudanças de perfil e moderações sempre geram `LOG_AUDITORIA` | `LOG_AUDITORIA` |
| Visitante só recebe dados públicos (nenhum dado de membro) | `HORTA` |

## 2.5 Exemplo de fluxo: registrar uma colheita

```mermaid
sequenceDiagram
    actor L as Líder (Web)
    participant API as Sistema Administrativo
    participant DB as Banco Operacional (OLTP)
    L->>API: POST /cultivos/3/colheitas {quantidade: 8.5, unidade: KG, destino: DOACAO}
    API->>API: valida token e permissão CULTIVO_GERENCIAR na horta do cultivo
    API->>DB: BEGIN
    API->>DB: INSERT INTO colheita (...)
    API->>DB: UPDATE cultivo SET status = 'COLHIDO' (se marcada como última)
    API->>DB: UPDATE canteiro SET status = 'DISPONIVEL' (se não restar cultivo ativo)
    API->>DB: COMMIT
    Note over DB: trigger atualiza atualizado_em<br/>(o ETL da madrugada leva a colheita ao DW)
    API-->>L: 201 Created + ColheitaDTO
```
