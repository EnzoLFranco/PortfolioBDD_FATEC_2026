# API 5º Semestre – 01/2026
## Projeto: Synthesi
**Empresa Parceira:** SIATT

**[Repositório GitHub](https://github.com/SQLutionsFATEC/API-5-Semestre)**

---

## Resumo do Projeto

O Synthesi foi desenvolvido para a SIATT como um ambiente analítico de Data Warehouse voltado à gestão de projetos e programas. O problema central era a dispersão dos dados de projetos entre diversos sistemas e bancos de dados isolados, o que dificultava a análise e o acompanhamento gerencial do portfólio como um todo.

A solução centraliza, transforma e organiza os dados de projetos utilizando estratégias de Data Warehouse, oferecendo um seletor de projetos e programas com visão geral de custos, materiais e tarefas, o acompanhamento de solicitações e pedidos de compra, o controle de estoque em relação à demanda de materiais e uma página de fornecedores com filtros e histórico de negociações.

---

## Tecnologias Utilizadas

**Back-end:**
- **Python & Django** — API responsável pelo processamento dos dados e pela camada de ETL
- **MySQL** — Banco de dados relacional, modelado em esquema dimensional (tabelas fato e dimensão)
- **Docker** — Conteinerização do banco, do back-end e do front-end para padronização de ambientes
- **Pytest / Pytest-Django** — Testes unitários e de integração
- **SonarCloud / SonarQube** — Análise estática contínua de qualidade de código

**Front-end:**
- **React & TypeScript** — Framework e tipagem principal da interface
- **TailwindCSS & Material UI** — Estilização e componentes visuais (incluindo DataGrid)
- **React Testing Library** — Testes de componentes

**Gestão:**
- **Jira** — Gestão de backlog e sprints
- **Slack** — Comunicação da equipe
- **Figma** — Prototipação de telas

---

## Contribuições Individuais

Atuei como Desenvolvedor Full-Stack, com contribuições tanto no back-end (modelagem de dados, endpoints da API e infraestrutura) quanto no front-end (painel de solicitações e integração com fornecedores), além de ter ficado responsável pela configuração inicial do ambiente Docker e da análise estática de código do projeto.

---

### Back-end: Infraestrutura Inicial com Docker

**Situação:** No início do projeto, não havia um ambiente de desenvolvimento padronizado — cada integrante precisaria configurar manualmente o banco de dados, as dependências do back-end e do front-end em sua própria máquina, o que tende a gerar inconsistências entre ambientes.

**Tarefa:** Criar a infraestrutura de containers do projeto, garantindo que qualquer integrante da equipe conseguisse subir o ambiente completo (banco, back-end e front-end) com um único comando.

**Ação:** Criei os Dockerfiles do back-end, do front-end e do banco de dados, além do arquivo `docker-compose.yaml` de orquestração dos serviços e do script `init.sql` responsável pela criação do schema e dos privilégios de acesso ao banco. Configurei também o `requirements.txt` do back-end, incluindo o Gunicorn como servidor de aplicação para o ambiente conteinerizado.

**Resultado:** A equipe passou a subir o ambiente completo do zero com `docker compose up`, eliminando divergências de configuração entre máquinas e reduzindo o tempo de onboarding de novos integrantes ao projeto.

---

### Back-end: Endpoint de Compras com Cálculo de Prazo de Entrega

**Situação:** A equipe precisava de um endpoint que retornasse, para um projeto específico, todas as compras realizadas junto a fornecedores, incluindo indicadores de atraso na entrega — informação até então inexistente na aplicação.

**Tarefa:** Implementar o endpoint `compras_projeto_api`, calculando o prazo médio de entrega e os dias de atraso de cada pedido, com tratamento adequado para os casos de projeto inexistente e de método HTTP não permitido.

**Ação:** Implementei o endpoint calculando a diferença entre a data prevista e a data efetiva de entrega de cada compra, agregando o prazo médio do projeto. Adicionei tratamento explícito retornando HTTP 404 quando o projeto consultado não existe e HTTP 405 quando o método da requisição não é suportado, cobrindo ambos os casos com testes automatizados.

**Resultado:** O endpoint passou a fornecer ao front-end os dados necessários para o acompanhamento de prazos de entrega por projeto, com cobertura de testes garantindo o comportamento correto diante de projetos inexistentes e requisições malformadas.

---

### Back-end: Refatoração Modular das Views

**Situação:** As funcionalidades de back-end estavam concentradas em um único arquivo de views, o que dificultava a manutenção e aumentava a chance de conflitos de merge conforme a API crescia com novas funcionalidades.

**Tarefa:** Separar as responsabilidades do arquivo de views em módulos independentes, organizados por domínio funcional.

**Ação:** Refatorei o arquivo único em módulos especializados — `projeto_dashboard_api`, `projeto_alertas_api`, `compras_projeto_api`, `projeto_empenho_api`, `empenhos_programa` e `projeto_tarefas_timesheet_api` —, cada um isolando a lógica de uma área específica do sistema, e ajustei as exportações para que os demais módulos da aplicação continuassem funcionando sem alterações.

**Resultado:** A reorganização reduziu conflitos de merge entre integrantes trabalhando em funcionalidades diferentes simultaneamente e tornou mais simples localizar e alterar a lógica de uma funcionalidade específica.

---

### Back-end: Analytics e Listagem Detalhada de Solicitações

**Situação:** Gestores não tinham como visualizar indicadores agregados sobre as solicitações de compra de um projeto (como total de pendências e solicitações urgentes), nem uma listagem detalhada que relacionasse cada solicitação às compras efetivamente realizadas.

**Tarefa:** Implementar o endpoint de analytics de solicitações e o endpoint de listagem detalhada, unindo os dados de solicitação e de compra.

**Ação:** Implementei o `request_analytics_api`, calculando o total de solicitações pendentes e rastreando solicitações urgentes, e o `listagem_solicitacoes`, relacionando as dimensões `DimSolicitacao` e a tabela fato `FatoCompra` para exibir o histórico completo de cada pedido, com testes cobrindo requisições vazias e casos de projeto não encontrado.

**Resultado:** A equipe de front-end passou a ter uma fonte única de dados para construir os indicadores e a tabela de solicitações do painel de acompanhamento, sem precisar agregar informações manualmente a partir de múltiplos endpoints.

---

### Back-end: Listagem de Fornecedores com Filtros

**Situação:** A página de fornecedores precisava de um endpoint que permitisse buscar fornecedores por diferentes critérios, como categoria, sem retornar sempre a lista completa cadastrada.

**Tarefa:** Implementar o endpoint `listagem_fornecedores` com suporte a múltiplos filtros de busca.

**Ação:** Implementei o endpoint aplicando os filtros recebidos por parâmetro de forma combinável, simplificando a lógica de filtragem para evitar condições redundantes, e cobri o comportamento com testes de integração para os diferentes cenários de filtro.

**Resultado:** A página de fornecedores passou a permitir buscas mais específicas no front-end, reduzindo o volume de dados retornado pela API e facilitando a localização de fornecedores pelos gestores.

---

### Front-end: Painel de Acompanhamento de Solicitações (RequestDashboardScreen)

**Situação:** Gestores precisavam acompanhar, em uma única tela, as solicitações de compra em andamento de um projeto, com indicadores agregados (como total de pendências e solicitações urgentes) e uma listagem detalhada e filtrável de cada solicitação.

**Tarefa:** Desenvolver o componente `RequestDashboardScreen`, integrando indicadores (`KpiCards`) e uma tabela detalhada (`RequestTable`) consumindo dados reais da API de analytics de solicitações.

**Ação:** Implementei o `RequestDashboardScreen` inicialmente com dados simulados para validar o layout com a equipe, substituindo-os posteriormente por chamadas reais por meio de um `requestService` dedicado (`getSolicitacoes` e `getSolicitacoesAnalytics`), com tratamento de estado de erro na busca dos dados analíticos. A tabela `RequestTable` foi construída sobre o componente `DataGrid` do Material UI, e os indicadores do `KpiCards` foram adaptados para consumir a nova estrutura de dados de solicitação.

**Resultado:** O painel passou a refletir dados reais da API, com indicadores e tabela de solicitações sincronizados, e tratamento de erro visível ao usuário em caso de falha na busca — substituindo a versão anterior baseada em dados simulados.

---

### Front-end: Modal de Informações do Fornecedor (SupplierInfoModal)

**Situação:** A página de fornecedores listava os fornecedores cadastrados, mas não exibia detalhes sobre pedidos realizados nem um indicador de confiabilidade com base no histórico de entregas de cada um.

**Tarefa:** Integrar o modal `SupplierInfoModal` a dados reais da API, definindo os tipos de dados necessários e a lógica de classificação de confiabilidade do fornecedor.

**Ação:** Implementei o `supplierService`, responsável por buscar os detalhes e os pedidos de um fornecedor específico, e defini os tipos `SupplierDetail` e `SupplierOrdersResponse` para tipar essas respostas. Refatorei o `SupplierInfoModal` para consumir esse serviço em vez de dados estáticos, implementando a lógica que classifica a confiabilidade do fornecedor com base no seu histórico de atrasos.

**Resultado:** O modal passou a exibir informações reais de cada fornecedor, incluindo seu histórico de pedidos e um indicador visual de confiabilidade, apoiando a decisão de gestores na escolha de fornecedores para novos pedidos.

---

### Front-end: Acompanhamento de Materiais Empenhados (CommitmentMaterial)

**Situação:** O acompanhamento de materiais empenhados em um projeto dependia de dados estáticos e não oferecia uma visão clara da evolução de custos ao longo do tempo, dificultando a identificação de tendências de gasto.

**Tarefa:** Refatorar o componente `CommitmentMaterial` para consumir dados reais da API e adicionar visualizações gráficas de custo.

**Ação:** Refatorei o `CommitmentMaterial` para carregar os dados por meio de um `commitmentService` dedicado em vez de dados fixos, removendo a dependência de parâmetros de URL para o carregamento, e implementei o `CommitmentCharts` com gráficos de barras e de linhas para a análise de custos ao longo do tempo, cobrindo os cenários de carregamento, erro e dados vazios com testes.

**Resultado:** Gestores passaram a visualizar a evolução de custos de materiais empenhados diretamente em gráficos, facilitando a identificação de tendências que antes exigiam análise manual dos dados brutos.

---

## Hard Skills (Autoavaliação)

| Tecnologia / Metodologia | Nível | Classificação |
|--------------------------|-------|---------------|
| Python + Django | ★★★★☆ | Sei fazer com ajuda |
| React + TypeScript | ★★★★☆ | Sei fazer com ajuda |
| MySQL / Modelagem Dimensional | ★★★★★ | Sei fazer com autonomia |
| Docker | ★★★★★ | Sei fazer com autonomia |
| Testes Automatizados (Pytest / RTL) | ★★★☆☆ | Entendi |
| Metodologia Ágil (Scrum) | ★★★★★ | Sei fazer com autonomia |

---

## Soft Skills

- **Comunicação**
  A atuação em praticamente todas as camadas do sistema — infraestrutura, back-end e front-end — exigiu manter os contratos de dados entre a API Django e o consumo em React consistentes entre si, comunicando claramente à equipe mudanças nos formatos de resposta dos endpoints antes de integrá-los ao front-end.

- **Trabalho em equipe**
  A consolidação da cultura de testes do projeto envolveu não apenas escrever os próprios testes, mas também elaborar um guia interno de testes unitários e de integração para orientar o restante da equipe na cobertura de casos de borda, como requisições vazias e projetos inexistentes.

- **Proatividade**
  Configurar desde cedo o ambiente Docker e os pipelines de integração contínua com SonarCloud representou uma iniciativa própria, antecipando uma base de infraestrutura e qualidade sobre a qual toda a equipe passou a construir ao longo das sprints seguintes.

---

## Aprendizados Efetivos

O Synthesi representou o projeto de maior amplitude técnica da minha trajetória até então, exigindo transitar entre modelagem dimensional de dados (tabelas fato e dimensão), desenvolvimento back-end em Django — uma stack nova em relação ao Spring Boot usado no semestre anterior —, desenvolvimento front-end em React e práticas de DevOps e qualidade de software em um único projeto.

Entender a lógica de um Data Warehouse, separando dados transacionais em tabelas fato e dimensão para favorecer a análise em vez da normalização, ampliou significativamente minha visão sobre banco de dados construída nos semestres anteriores com bancos puramente transacionais. Configurar o ambiente Docker e os pipelines de CI desde o início do projeto também consolidou a percepção de que investir em infraestrutura e qualidade de código no começo de um projeto reduz atrito e retrabalho ao longo de todas as sprints seguintes.

---

## Navegação

| Semestre | Projeto | Empresa Parceira |
|----------|---------|-----------------|
| [1º Semestre – 01/2024](API01.md) | Calculadora Científica | FATEC São José dos Campos |
| [2º Semestre – 02/2024](API02.md) | Avaliador de Soft Skills (PACER) | FATEC São José dos Campos |
| [3º Semestre – 01/2025](API03.md) | Checkpoint | Altave |
| [4º Semestre – 02/2025](API04.md) | Sistema de Mobilidade Urbana | Prefeitura de São José dos Campos |

[← Voltar ao Portfólio](README.md)
