# 3. Banco de Dados Operacional (OLTP)

## 3.1 Visão geral do modelo lógico

O modelo é **relacional e normalizado (3ª Forma Normal)**, porque o foco do OLTP é gravar e alterar dados com rapidez e sem duplicidade. São **19 entidades**, organizadas em 4 domínios.

```mermaid
erDiagram
    PERFIL           ||--o{ PERFIL_PERMISSAO    : "concede"
    PERMISSAO        ||--o{ PERFIL_PERMISSAO    : "compõe"
    PERFIL           ||--o{ USUARIO             : "perfil global"
    PERFIL           ||--o{ MEMBRO_HORTA        : "papel na horta"
    USUARIO          ||--o{ MEMBRO_HORTA        : "participa de"
    COMUNIDADE       ||--o{ HORTA               : "mantém"
    HORTA            ||--o{ MEMBRO_HORTA        : "tem"
    USUARIO          ||--o{ SOLICITACAO_ENTRADA : "solicita"
    HORTA            ||--o{ SOLICITACAO_ENTRADA : "recebe"
    HORTA            ||--o{ CANTEIRO            : "dividida em"
    CANTEIRO         ||--o{ CULTIVO             : "recebe"
    ESPECIE          ||--o{ CULTIVO             : "plantada em"
    CULTIVO          ||--o{ COLHEITA            : "gera"
    HORTA            ||--o{ MUTIRAO             : "organiza"
    MUTIRAO          ||--o{ PRESENCA_MUTIRAO    : "registra"
    USUARIO          ||--o{ PRESENCA_MUTIRAO    : "confirma"
    HORTA            ||--o{ TAREFA              : "possui"
    CANTEIRO         |o--o{ TAREFA              : "alvo de"
    MUTIRAO          |o--o{ TAREFA              : "agrupa"
    TIPO_TAREFA      ||--o{ TAREFA              : "classifica"
    USUARIO          |o--o{ TAREFA              : "responsável por"
    ESPECIE          |o--o{ DICA_CULTIVO        : "tema de"
    USUARIO          ||--o{ DICA_CULTIVO        : "escreve"
    HORTA            ||--o{ AVISO               : "publica"
    USUARIO          ||--o{ AVISO               : "escreve"
    USUARIO          |o--o{ LOG_AUDITORIA       : "gera"
```

| Domínio | Entidades |
|---|---|
| **Acesso e pessoas** | `PERFIL`, `PERMISSAO`, `PERFIL_PERMISSAO`, `USUARIO`, `MEMBRO_HORTA`, `SOLICITACAO_ENTRADA` |
| **Espaços e cultivos** | `COMUNIDADE`, `HORTA`, `CANTEIRO`, `ESPECIE`, `CULTIVO`, `COLHEITA` |
| **Atividades** | `MUTIRAO`, `PRESENCA_MUTIRAO`, `TIPO_TAREFA`, `TAREFA` |
| **Conteúdo e controle** | `DICA_CULTIVO`, `AVISO`, `LOG_AUDITORIA` |

---

## 3.2 Domínio: Acesso e pessoas

Implementa a **Matriz de Perfis e Permissões** do README. Um mesmo usuário pode ser **Líder** de uma horta e **Membro** de outra; por isso o papel na horta fica em `MEMBRO_HORTA`, e não em `USUARIO`. O **Visitante** não é autenticado e não tem registro no banco.

```mermaid
erDiagram
    PERFIL {
        smallint id_perfil PK
        varchar  codigo UK "ADMIN_GERAL | LIDER_HORTA | MEMBRO"
        varchar  nome
        varchar  escopo "GLOBAL | HORTA"
    }
    PERMISSAO {
        smallint id_permissao PK
        varchar  codigo UK "ex.: MEMBRO_APROVAR"
        varchar  descricao
    }
    PERFIL_PERMISSAO {
        smallint id_perfil PK, FK
        smallint id_permissao PK, FK
    }
    USUARIO {
        bigint      id_usuario PK
        varchar     nome
        varchar     email UK
        varchar     telefone "opcional"
        varchar     senha_hash "nunca a senha"
        smallint    id_perfil_global FK
        varchar     status "PENDENTE | ATIVO | INATIVO"
        timestamptz aceite_termos_em
        timestamptz atualizado_em "watermark do ETL"
    }
    MEMBRO_HORTA {
        bigint   id_membro_horta PK
        bigint   id_usuario FK "UK com id_horta"
        int      id_horta FK
        smallint id_perfil FK "LIDER_HORTA | MEMBRO"
        varchar  status "ATIVO | AFASTADO | DESLIGADO"
        date     data_entrada
        date     data_saida
    }
    SOLICITACAO_ENTRADA {
        bigint      id_solicitacao PK
        bigint      id_usuario FK
        int         id_horta FK
        varchar     status "PENDENTE | APROVADA | RECUSADA"
        timestamptz solicitado_em
        bigint      id_avaliador FK "líder ou admin"
        timestamptz avaliado_em
        varchar     motivo_recusa
    }
    PERFIL    ||--o{ PERFIL_PERMISSAO    : ""
    PERMISSAO ||--o{ PERFIL_PERMISSAO    : ""
    PERFIL    ||--o{ USUARIO             : ""
    PERFIL    ||--o{ MEMBRO_HORTA        : ""
    USUARIO   ||--o{ MEMBRO_HORTA        : ""
    USUARIO   ||--o{ SOLICITACAO_ENTRADA : ""
```

## 3.3 Domínio: Espaços e cultivos

Uma **horta** é dividida em **canteiros**; cada plantio de uma **espécie** em um canteiro é um **cultivo** (uma safra), que gera várias **colheitas**. Essa cadeia é a base do "histórico de safras" e do "uso dos espaços" que o Data Warehouse analisa.

```mermaid
erDiagram
    COMUNIDADE {
        int     id_comunidade PK
        varchar nome
        varchar tipo "ASSOCIACAO | ESCOLA | CONDOMINIO | COLETIVO | OUTRO"
        varchar cidade
        char    uf
    }
    HORTA {
        int     id_horta PK
        int     id_comunidade FK
        varchar nome
        varchar logradouro
        varchar bairro
        varchar cidade
        char    uf
        numeric latitude "mapa público"
        numeric longitude
        numeric area_total_m2
        varchar visibilidade "PUBLICA | PRIVADA"
        varchar status "EM_IMPLANTACAO | ATIVA | INATIVA"
    }
    CANTEIRO {
        int     id_canteiro PK
        int     id_horta FK "UK com codigo"
        varchar codigo "ex.: A-01"
        numeric area_m2
        varchar tipo_solo
        boolean tem_irrigacao
        varchar status "DISPONIVEL | OCUPADO | EM_DESCANSO | INATIVO"
        bigint  id_responsavel FK
    }
    ESPECIE {
        int     id_especie PK
        varchar nome_popular
        varchar nome_cientifico UK
        varchar familia
        varchar categoria "HORTALICA | LEGUME | FRUTA | ERVA_TEMPERO | MEDICINAL | PANC"
        int     ciclo_medio_dias
    }
    CULTIVO {
        bigint  id_cultivo PK
        int     id_canteiro FK
        int     id_especie FK
        bigint  id_usuario_registro FK
        date    data_plantio
        date    data_prevista_colheita
        date    data_encerramento
        numeric area_ocupada_m2
        int     qtd_mudas
        varchar status "PLANEJADO | EM_ANDAMENTO | COLHIDO | PERDIDO | ENCERRADO"
    }
    COLHEITA {
        bigint  id_colheita PK
        bigint  id_cultivo FK
        bigint  id_usuario_registro FK
        date    data_colheita
        numeric quantidade
        varchar unidade "KG | UNIDADE | MACO"
        numeric peso_kg "padronizado p/ análise"
        varchar destino "CONSUMO_MEMBROS | DOACAO | FEIRA_TROCA | COMPOSTAGEM"
    }
    COMUNIDADE ||--o{ HORTA    : ""
    HORTA      ||--o{ CANTEIRO : ""
    CANTEIRO   ||--o{ CULTIVO  : ""
    ESPECIE    ||--o{ CULTIVO  : ""
    CULTIVO    ||--o{ COLHEITA : ""
```

## 3.4 Domínio: Atividades

```mermaid
erDiagram
    MUTIRAO {
        bigint      id_mutirao PK
        int         id_horta FK
        bigint      id_organizador FK
        varchar     titulo
        timestamptz data_hora_inicio
        timestamptz data_hora_fim
        int         vagas
        varchar     status "AGENDADO | REALIZADO | CANCELADO"
    }
    PRESENCA_MUTIRAO {
        bigint      id_mutirao PK, FK
        bigint      id_usuario PK, FK
        varchar     status "CONFIRMADO | PRESENTE | AUSENTE | CANCELADO"
        timestamptz confirmado_em
        timestamptz check_in_em
    }
    TIPO_TAREFA {
        smallint id_tipo_tarefa PK
        varchar  codigo UK "REGA | PLANTIO | ADUBACAO | ..."
        varchar  nome
        varchar  grupo "CULTIVO | MANUTENCAO | COMUNIDADE"
    }
    TAREFA {
        bigint      id_tarefa PK
        int         id_horta FK
        int         id_canteiro FK "opcional"
        bigint      id_mutirao FK "opcional"
        smallint    id_tipo_tarefa FK
        bigint      id_criador FK
        bigint      id_responsavel FK "opcional"
        date        prazo
        varchar     status "ABERTA | EM_ANDAMENTO | CONCLUIDA | CANCELADA"
        timestamptz concluida_em
    }
    MUTIRAO     ||--o{ PRESENCA_MUTIRAO : ""
    MUTIRAO     |o--o{ TAREFA           : ""
    TIPO_TAREFA ||--o{ TAREFA           : ""
```

## 3.5 Domínio: Conteúdo e controle

```mermaid
erDiagram
    DICA_CULTIVO {
        bigint  id_dica PK
        int     id_especie FK "NULL = dica geral"
        bigint  id_autor FK
        varchar titulo
        text    conteudo
        boolean publicada
    }
    AVISO {
        bigint      id_aviso PK
        int         id_horta FK
        bigint      id_autor FK
        varchar     titulo
        text        mensagem
        varchar     status "PUBLICADO | OCULTO | REMOVIDO"
        bigint      id_moderador FK "líder que moderou"
        timestamptz publicado_em
    }
    LOG_AUDITORIA {
        bigint      id_log PK
        bigint      id_usuario FK
        varchar     acao "INSERT | UPDATE | DELETE | LOGIN | APROVACAO | PERMISSAO"
        varchar     entidade
        bigint      id_registro
        jsonb       dados_anteriores
        jsonb       dados_novos
        inet        ip_origem
        timestamptz ocorrido_em
    }
```

---

## 3.6 Dicionário resumido das entidades

| Entidade | O que representa | Chave primária | Chaves estrangeiras | Regras importantes |
|---|---|---|---|---|
| `PERFIL` | Perfis da Matriz de Permissões | `id_perfil` | — | `codigo` único; escopo GLOBAL ou HORTA |
| `PERMISSAO` | Ação que pode ser liberada | `id_permissao` | — | `codigo` único |
| `PERFIL_PERMISSAO` | Quais permissões cada perfil tem (N:N) | (`id_perfil`, `id_permissao`) | perfil, permissão | — |
| `USUARIO` | Pessoa cadastrada | `id_usuario` | perfil global | e-mail único; senha só como hash |
| `MEMBRO_HORTA` | Vínculo usuário × horta e o papel exercido | `id_membro_horta` | usuário, horta, perfil | 1 vínculo por usuário/horta; `data_saida ≥ data_entrada` |
| `SOLICITACAO_ENTRADA` | Pedido para entrar em uma horta | `id_solicitacao` | usuário, horta, avaliador | status PENDENTE → APROVADA/RECUSADA |
| `COMUNIDADE` | Associação, escola, condomínio etc. | `id_comunidade` | — | tipo controlado por CHECK |
| `HORTA` | Horta comunitária | `id_horta` | comunidade | área > 0; visibilidade pública/privada |
| `CANTEIRO` | Espaço de cultivo dentro da horta | `id_canteiro` | horta, responsável | código único dentro da horta; área > 0 |
| `ESPECIE` | Catálogo de plantas | `id_especie` | — | nome científico único |
| `CULTIVO` | Um plantio (safra) em um canteiro | `id_cultivo` | canteiro, espécie, usuário | datas coerentes (encerramento ≥ plantio) |
| `COLHEITA` | Cada colheita de um cultivo | `id_colheita` | cultivo, usuário | quantidade > 0; peso em kg padronizado |
| `MUTIRAO` | Evento coletivo de trabalho | `id_mutirao` | horta, organizador | fim > início |
| `PRESENCA_MUTIRAO` | Confirmação/presença de um usuário (N:N) | (`id_mutirao`, `id_usuario`) | mutirão, usuário | 1 registro por pessoa por mutirão |
| `TIPO_TAREFA` | Tipos de tarefa (rega, plantio...) | `id_tipo_tarefa` | — | agrupados em CULTIVO / MANUTENÇÃO / COMUNIDADE |
| `TAREFA` | Atividade atribuída a um membro | `id_tarefa` | horta, canteiro, mutirão, tipo, criador, responsável | se CONCLUÍDA, `concluida_em` obrigatório |
| `DICA_CULTIVO` | Conhecimento compartilhado | `id_dica` | espécie, autor | — |
| `AVISO` | Comunicação local da horta | `id_aviso` | horta, autor, moderador | moderado pelo líder |
| `LOG_AUDITORIA` | Quem fez o quê e quando | `id_log` | usuário | a API só pode **inserir** (não altera nem apaga) |

### Por que está normalizado assim (3FN)

- **`COMUNIDADE` separada de `HORTA`:** uma comunidade pode manter várias hortas; o nome dela não se repete em cada horta.
- **`MEMBRO_HORTA` e `PRESENCA_MUTIRAO`:** resolvem relações muitos-para-muitos (usuário × horta, usuário × mutirão) e guardam atributos próprios do vínculo (papel, status, check-in).
- **`PERFIL_PERMISSAO`:** liberar uma permissão nova para um perfil é inserir uma linha, sem alterar código.
- **`TIPO_TAREFA` e `ESPECIE` como tabelas de referência:** evitam grafias diferentes ("Rega", "rega", "Regar") que atrapalhariam a análise depois.

---

## 3.7 Como os dados são armazenados (modelo físico)

| Decisão | Como foi implementado | Por quê |
|---|---|---|
| **SGBD** | PostgreSQL 16, banco `cultiva_oltp`, schema `oltp` | Relacional, gratuito, transações ACID, suportado pelos principais provedores de nuvem |
| **Chaves** | Chaves substitutas numéricas (`IDENTITY`) + chaves naturais com `UNIQUE` (e-mail, código do canteiro) | Joins rápidos e sem duplicidade de cadastro |
| **Domínios** | `CHECK` para status, categorias e unidades | O banco recusa valores inválidos mesmo se a API falhar |
| **Datas** | `TIMESTAMPTZ` (com fuso) para eventos; `DATE` para datas de calendário | Evita erros de fuso entre servidor e usuários |
| **Medidas** | `NUMERIC` para área (m²) e peso (kg) | Precisão exata (sem arredondamento de ponto flutuante) |
| **Controle de mudanças** | Coluna `atualizado_em` mantida por *trigger* em todas as tabelas alteráveis | É a "marca d'água" que o ETL usa para extrair só o que mudou |
| **Índices** | Nas chaves estrangeiras mais consultadas e em `atualizado_em` | Telas rápidas e extração incremental eficiente |
| **Auditoria** | `LOG_AUDITORIA` com estado anterior/novo em `JSONB` | Atende "logs de auditoria" da Matriz de Permissões |
| **Usuários do banco** | `api_app` (lê/escreve; não apaga auditoria) e `etl_leitor` (só leitura, sem dados pessoais) | Princípio do menor privilégio |
| **Privacidade (LGPD)** | Só dados pessoais mínimos (nome, e-mail, telefone opcional); senha como hash; o ETL lê `vw_extracao_usuario`, que não expõe nome/e-mail/telefone | Coerente com o README: "evitar a coleta de informações desnecessárias" |

### Exemplo: o que acontece quando um líder aprova um novo membro

Tudo em **uma única transação** feita pela API (ou tudo é gravado, ou nada é):

1. `SOLICITACAO_ENTRADA.status` → `APROVADA` (com `id_avaliador` e `avaliado_em`);
2. `INSERT` em `MEMBRO_HORTA` com perfil `MEMBRO`;
3. `USUARIO.status` → `ATIVO` (se estava `PENDENTE`);
4. `INSERT` em `LOG_AUDITORIA` (ação `APROVACAO`).
