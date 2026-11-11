# Sistema de Hortas Comunitárias — Cultiva Plus

## Descrição do Projeto

O **Cultiva Plus** é uma plataforma voltada para o gerenciamento de hortas comunitárias, promovendo uma rede de cultivo colaborativo entre moradores, comunidades e responsáveis pelas hortas. A solução busca facilitar a organização dos espaços, o acompanhamento dos cultivos e a participação dos membros, tornando o processo mais simples, acessível e colaborativo.

Por meio da tecnologia, o Cultiva Plus conecta pessoas interessadas em cultivar, compartilhar conhecimentos e contribuir para o desenvolvimento de hortas comunitárias, incentivando a sustentabilidade, o aproveitamento de espaços e o fortalecimento da comunidade.

## Objetivo da Solução

O objetivo do **Cultiva Plus** é facilitar o gerenciamento de hortas comunitárias por meio de uma plataforma colaborativa que centraliza informações, organiza atividades e conecta os participantes. A solução busca tornar a gestão dos cultivos mais eficiente, incentivar a participação da comunidade e promover práticas sustentáveis de cultivo e compartilhamento.

## Integrantes da Equipe

* Daniel Alonso Oliveira Soares
* João Pablo Silva Pinheiro
* Erick Sales Ramos
* Antonio Gabriel Gomes Falcao do Nascimento
* Rafael Jesus Bezerra Cerqueira
* Alexandre Roque Pereira Costa
* Breno Kayan Souza de Azevedo

---

# Empresa: GDP Projects

## Segmento de Atuação
Desenvolvimento de software, inteligência artificial e soluções digitais sob medida. Criação de chatbots, landing pages, plataformas web e sistemas de gestão para pequenas empresas e iniciativas comunitárias.

## Missão
Impulsionar pequenos negócios e iniciativas locais com soluções digitais acessíveis, inteligentes e eficientes. Transformação de necessidades operacionais em tecnologia prática.

## Visão
Reconhecimento como consultoria de desenvolvimento ágil e inovadora, referência na criação de soluções digitais para micro e pequenas empresas até 2028.

## Valores
* Inovação adaptativa
* Acessibilidade digital
* Foco no cliente e no impacto do negócio
* Agilidade e entrega de valor

## Organograma
* **Direção Executiva:** Daniel Alonso Oliveira Soares
* **Gestão de Produtos e Projetos:** João Pablo Silva Pinheiro
* **Desenvolvimento Back-end e IA:** Erick Sales Ramos e Antonio Gabriel Gomes Falcão do Nascimento
* **Desenvolvimento Front-end e UX/UI:** Rafael Jesus Bezerra Cerqueira
* **Infraestrutura e DevOps:** Alexandre Roque Pereira Costa
* **Marketing e Atendimento ao Cliente:** Breno Kayan Souza de Azevedo

---

# Análise de Dados e Necessidades do Sistema

## Quais dados o sistema produz e utiliza?
* Informações das hortas e dos espaços disponíveis;
* Informações sobre os cultivos e colheitas;
* Dados dos participantes e membros voluntários;
* Informações sobre as atividades e mutirões realizados;
* Dados de acompanhamento e organização dos canteiros.

## Quem utiliza esses dados?
* Moradores e membros das comunidades;
* Responsáveis e líderes locais pelas hortas;
* Gestores de projetos comunitários e equipe executiva da GDP Projects.

## Onde os dados são armazenados?
Os dados operacionais ficam centralizados em banco relacional PostgreSQL (`cultiva_oltp`), o histórico gerencial é consolidado em um Data Warehouse analítico (`cultiva_dw`), os backups e fichas digitalizadas residem em Object Storage em nuvem, e cópias físicas de segurança são arquivadas nas sedes locais.

## Dados sensíveis e privacidade
O Cultiva Plus não coleta dados pessoais sensíveis. Dados cadastrais comuns (e-mail, telefone) são protegidos com hash irreversível para senhas e mascarados nos ambientes analíticos, em total conformidade com os princípios da LGPD.

---

# Documentação de Dados (`docs/`)

Toda a documentação técnica e de engenharia de dados está organizada na pasta [`docs/`](docs/), dividida por disciplina:

| Módulo | Descrição | Link |
| :--- | :--- | :--- |
| **Data Architecture** | Visão macro dos 6 sistemas, fluxo de dados, separação OLTP vs. OLAP e matriz de permissões. | [docs/data-architecture/](docs/data-architecture/README.md) |
| **Data Modeling** | Modelagem lógica dos sistemas, modelo relacional 3FN, esquema estrela analítico (DW), DDLs SQL e gestão documental física. | [docs/data-modeling/](docs/data-modeling/README.md) |
| **Data Storage** | Tecnologias de armazenamento, persistência, regras de negócio, volume, escala e planos de backup/recuperação. | [docs/data-storage/](docs/data-storage/README.md) |
| **Data Security** | Classificação de dados, autenticação/autorização, princípio do menor privilégio, proteção em repouso/trânsito e auditoria. | [docs/data-security/](docs/data-security/README.md) |
| **Data Integration** | Integrações entre Web, API, OLTP, DW e BI: formatos, REST API, ETL via `postgres_fdw`, resiliência, tolerância a falhas e consistência. | [docs/data-integration/](docs/data-integration/README.md) |
| **Apresentação da Equipe** | Guia completo de preparação para a banca: divisão de papéis por integrante, roteiro do pitch e respostas para perguntas difíceis. | [docs/guia-apresentacao-equipe.md](docs/guia-apresentacao-equipe.md) |

---

## Estrutura de Pastas

```text
cultiva-plus/
├── README.md                          # Visão geral do projeto e índice de docs
└── docs/
    ├── data-architecture/             # Arquitetura dos sistemas e fluxo de informação
    ├── data-modeling/                 # Modelagem lógica, física (SQL) e documental
    ├── data-storage/                  # Estratégia de armazenamento, persistência e backups
    ├── data-security/                 # Segurança da informação, acessos e auditoria
    └── data-integration/              # Mecanismos de integração, ETL e consistência
```
