# Sistema de Hortas Comunitárias — Cultiva Plus

## Descrição do Projeto

O **Cultiva Plus** é uma plataforma prática para o gerenciamento de hortas comunitárias, conectando moradores, voluntários e responsáveis locais para planejar canteiros, acompanhar safras e organizar mutirões de forma colaborativa.

## Integrantes da Equipe — GDP Projects

* Daniel Alonso Oliveira Soares
* João Pablo Silva Pinheiro
* Erick Sales Ramos
* Antonio Gabriel Gomes Falcao do Nascimento
* Rafael Jesus Bezerra Cerqueira
* Alexandre Roque Pereira Costa
* Breno Kayan Souza de Azevedo

---

## O Que o Sistema Faz?

* **Cadastra hortas e canteiros:** Organiza os espaços físicos disponíveis em cada comunidade.
* **Controla cultivos e safras:** Registra o que foi plantado, quando e quanto foi colhido (com destino para consumo ou doação).
* **Organiza mutirões e tarefas:** Gerencia a presença dos voluntários e as atividades rotineiras (rega, adubação, capina).
* **Gera relatórios de produção:** Consolida indicadores simples para prestação de contas comunitária e acompanhamento dos resultados.

---

## Documentação Técnica (`docs/`)

Toda a documentação do projeto está organizada em módulos diretos na pasta `docs/`:

| Módulo | O que contém? | Link |
| :--- | :--- | :--- |
| **Data Architecture** | Visão geral dos componentes, fluxo simples de dados e perfis de acesso. | [docs/data-architecture/](docs/data-architecture/README.md) |
| **Data Modeling** | Diagrama de entidades (hortas, canteiros, cultivos, colheitas), scripts SQL e fichas em papel. | [docs/data-modeling/](docs/data-modeling/README.md) |
| **Data Storage** | Onde os dados ficam guardados (PostgreSQL), regras de persistência e plano de backup. | [docs/data-storage/](docs/data-storage/README.md) |
| **Data Security** | Senhas seguras (Argon2id), tráfego criptografado (HTTPS) e controle de permissões. | [docs/data-security/](docs/data-security/README.md) |
| **Data Integration** | Integração Web ↔ API, carga noturna de relatórios e prevenção de erros. | [docs/data-integration/](docs/data-integration/README.md) |
| **Guia de Apresentação** | Roteiro rápido e divisão de falas da equipe para a banca avaliadora. | [docs/guia-apresentacao-equipe.md](docs/guia-apresentacao-equipe.md) |
