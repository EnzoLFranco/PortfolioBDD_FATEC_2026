# API 5º Semestre – 01/2026
## Projeto: Synthesi
**Empresa Parceira:** SIATT

**[Repositório GitHub](https://github.com/SQLutions-FATEC/API-5-Semestre)**

---

## Resumo do Projeto

O Synthesi foi desenvolvido para a SIATT como um ambiente analítico de Data Warehouse voltado à gestão de projetos e programas. O problema central era a dispersão dos dados de projetos entre diversos sistemas e bancos de dados isolados, o que dificultava a análise e o acompanhamento gerencial do portfólio como um todo.

A solução centraliza, transforma e organiza os dados de projetos utilizando estratégias de Data Warehouse, oferecendo um seletor de projetos e programas com visão geral de custos, materiais e tarefas, o acompanhamento de solicitações e pedidos de compra, o controle de estoque em relação à demanda de materiais e uma página de fornecedores com filtros e histórico de negociações.

---

## Tecnologias Utilizadas

**Back-end:**
- **Python & Django** — API responsável pelo processamento dos dados e pela camada de ETL
- **MySQL** — Banco de dados relacional, modelado em esquema dimensional (tabelas fato e dimensão)
- **Gunicorn** — Servidor de aplicação WSGI usado no ambiente conteinerizado
- **Docker** — Conteinerização do banco, do back-end e do front-end para padronização de ambientes
- **Pytest / Pytest-Django** — Testes unitários e de integração
- **SonarCloud / SonarQube** — Análise estática contínua de qualidade de código
- **Prometheus e Grafana** — Monitoramento do banco e da aplicação

**Front-end:**
- **React & TypeScript** — Framework e tipagem principal da interface
- **TailwindCSS & Material UI** — Estilização e componentes visuais (incluindo DataGrid)
- **Recharts** — Gráficos de evolução de custos e distribuição por categoria
- **Vitest / React Testing Library** — Testes de componentes e serviços

**Gestão:**
- **Jira** — Gestão de backlog e sprints
- **Slack** — Comunicação da equipe
- **Figma** — Prototipação de telas

---

## Contribuições Individuais

Atuei como Desenvolvedor Full-Stack, com contribuições tanto no back-end (infraestrutura, endpoints da API e refatoração das views) quanto no front-end (painel de solicitações, modal de fornecedores e acompanhamento de materiais empenhados), além de ter ficado responsável pela configuração inicial do ambiente Docker e da análise estática de código do projeto.

---

### Back-end: Infraestrutura Inicial com Docker

**Situação:** No início do projeto, não havia um ambiente de desenvolvimento padronizado. Cada integrante precisaria configurar manualmente o banco de dados, as dependências do back-end e do front-end em sua própria máquina, o que tende a gerar inconsistências entre ambientes.

**Tarefa:** Criar a infraestrutura de containers do projeto, garantindo que qualquer integrante da equipe conseguisse subir o ambiente completo (banco, back-end e front-end) com um único comando.

**Ação:** Criei o Dockerfile do back-end, que instala as dependências em um ambiente virtual e inicia a aplicação com Gunicorn, além do `docker-compose.yaml` de orquestração dos serviços. Também criei o script `init.sql` responsável pela criação do schema e pelos privilégios de acesso ao banco, e o `requirements.txt` com o Gunicorn como servidor de aplicação. Posteriormente, o banco foi migrado para MySQL por outro integrante da equipe, e o script de inicialização passou a ser um `init.sh` que concede permissões ao banco de testes.

<details>
<summary>Código em Dockerfile – inicialização do back-end com Gunicorn</summary>

```dockerfile
FROM python:3.12-slim AS builder

WORKDIR /app

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --upgrade pip && pip install -r requirements.txt

# ... imagem de execução ...

EXPOSE 8000

ENTRYPOINT ["/app/entrypoint.sh"]

CMD ["gunicorn", "api.wsgi:application", "--bind", "0.0.0.0:8000"]
```

</details>

<details>
<summary>Código em SQL – deploy/postgres/init.sql (versão inicial do banco)</summary>

```sql
CREATE SCHEMA IF NOT EXISTS schema_api;
-- Aqui usamos o mesmo nome de usuário que vai estar no .env
GRANT ALL PRIVILEGES ON SCHEMA schema_api TO sqlutions;
ALTER ROLE sqlutions
SET search_path TO schema_api,
    public;
```

</details>

**Resultado:** A equipe passou a subir o ambiente completo com `docker compose up`, eliminando divergências de configuração entre máquinas e reduzindo o tempo de onboarding de novos integrantes ao projeto. A infraestrutura foi mantida pela equipe após a migração para MySQL.

---

### Back-end: Endpoint de Compras com Cálculo de Prazo de Entrega

**Situação:** A equipe precisava de um endpoint que retornasse, para um projeto específico, todas as compras realizadas junto a fornecedores, incluindo o prazo previsto de entrega de cada pedido, informação até então inexistente na aplicação.

**Tarefa:** Implementar o endpoint `compras_projeto_api`, calculando o prazo médio de entrega do projeto, com tratamento adequado para os casos de projeto inexistente e de método HTTP não permitido.

**Ação:** Implementei o endpoint calculando, para cada compra, a diferença em dias entre a data de emissão do pedido e a data de previsão de entrega, e agregando a média desses prazos no nível do projeto. Quando o projeto consultado não existe, a função `get_object_or_404` retorna HTTP 404. O decorador `@require_GET` retorna HTTP 405 para qualquer método diferente de GET. Ambos os casos são cobertos pelos testes automatizados.

<details>
<summary>Código em Python – compras_projeto_api</summary>

```python
@require_GET
def compras_projeto_api(request, codigo_projeto):
    projeto = get_object_or_404(DimProjeto, codigo_projeto=codigo_projeto)

    compras = FatoCompra.objects.filter(solicitacao__projeto=projeto).select_related(
        'fornecedor', 'data_pedido', 'data_previsao_entrega', 'solicitacao__projeto'
    )

    lista_compras = []
    soma_dias_entrega = 0
    quantidade_pedidos = 0

    for compra in compras:
        dt_pedido = compra.data_pedido
        dt_previsao = compra.data_previsao_entrega

        data_emissao = date(dt_pedido.ano, dt_pedido.mes, dt_pedido.dia)
        data_previsao = date(dt_previsao.ano, dt_previsao.mes, dt_previsao.dia)

        dias_previstos = (data_previsao - data_emissao).days

        soma_dias_entrega += dias_previstos
        quantidade_pedidos += 1

        lista_compras.append(
            {
                "numero": compra.numero_pedido,
                "emissao": data_emissao.strftime("%Y-%m-%d"),
                "previsao": data_previsao.strftime("%Y-%m-%d"),
                "fornecedor": compra.fornecedor.razao_social,
                "nome_material": compra.solicitacao.material.descricao,
                "status": compra.status,
                "dias_previstos_entrega": dias_previstos,
            }
        )

    tempo_medio = (
        round(soma_dias_entrega / quantidade_pedidos, 2)
        if quantidade_pedidos > 0
        else 0.0
    )

    return JsonResponse(
        {
            "projeto": projeto.codigo_projeto,
            "tempo_medio_entrega_dias": tempo_medio,
            "pedidos": lista_compras,
        }
    )
```

</details>

**Resultado:** O endpoint passou a fornecer ao front-end o prazo médio de entrega por projeto e a lista de pedidos com seus prazos previstos. Os testes cobrem projeto inexistente (`test_compras_not_found`) e método não permitido (`test_compras_wrong_method`), além de casos unitários como pedido com emissão igual à previsão, que resulta em zero dias, e lista vazia, que resulta em média zero.

---

### Back-end: Refatoração Modular das Views

**Situação:** As funcionalidades de back-end estavam concentradas em um único arquivo de views, o que dificultava a manutenção e aumentava a chance de conflitos de merge conforme a API crescia com novas funcionalidades.

**Tarefa:** Separar as responsabilidades do arquivo de views em módulos independentes, organizados por domínio funcional, sem quebrar as importações existentes no projeto.

**Ação:** Refatorei o arquivo único `views.py` em um pacote `views/`, com módulos especializados por área: `dashboard.py`, `alertas.py`, `compras.py`, `empenhos.py` e `tarefas.py`, além de `utils.py` para funções auxiliares compartilhadas. O `__init__.py` do pacote reexporta as funções públicas, de modo que `urls.py` e os demais módulos da aplicação continuassem funcionando sem alterações.

<details>
<summary>Código em Python – views/__init__.py</summary>

```python
from .dashboard import projeto_dashboard_api
from .compras import compras_projeto_api, evolucao_gastos_api
from .tarefas import projeto_tarefas_timesheet_api
from .alertas import projeto_alertas_api
from .empenhos import projeto_empenho_api, empenhos_programa
from .solicitacoesStats import request_analytics_api
from .solicitacoesLista import listagem_solicitacoes
from .programas import programa_api, busca_projetos, projeto_sem_filtro
from .gastos import detalhamento_gastos_projeto_api
from .sobras import otimizacao_sobras_api
from .fornecedores_list import listagem_fornecedores
from .fornecedores import fornecedor_api, fornecedor_pedidos_api
```

</details>

**Resultado:** A reorganização reduziu conflitos de merge entre integrantes trabalhando em funcionalidades diferentes simultaneamente e tornou mais simples localizar e alterar a lógica de uma funcionalidade específica. Como o `__init__.py` mantém as mesmas exportações, a mudança não exigiu ajustes nas rotas.

---

### Back-end: Analytics e Listagem Detalhada de Solicitações

**Situação:** Gestores não tinham como visualizar indicadores agregados sobre as solicitações de compra de um projeto, como total de pendências e solicitações urgentes, nem uma listagem detalhada que relacionasse cada solicitação às compras efetivamente realizadas.

**Tarefa:** Implementar o endpoint de analytics de solicitações e o endpoint de listagem detalhada, unindo os dados de solicitação e de compra.

**Ação:** Implementei o `request_analytics_api`, que conta as solicitações pendentes e lista as solicitações pendentes de prioridade alta ou crítica, calculando há quantos dias cada uma foi criada. Implementei também o `listagem_solicitacoes`, que relaciona as dimensões `DimSolicitacao` ao primeiro pedido vinculado da tabela fato `FatoCompra` e calcula o valor estimado de cada solicitação a partir da quantidade e do custo estimado do material.

<details>
<summary>Código em Python – request_analytics_api</summary>

```python
@require_GET
def request_analytics_api(request, codigo_projeto):
    projeto = get_object_or_404(DimProjeto, codigo_projeto=codigo_projeto)

    total_pendentes = DimSolicitacao.objects.filter(
        projeto=projeto, status__iexact='pendente'
    ).count()

    solicitacoes_criticas = DimSolicitacao.objects.filter(
        projeto=projeto,
        status__iexact='pendente',
        prioridade__in=['ALTA', 'CRITICA', 'Alta', 'Critica'],
    ).select_related('data_solicitacao')

    hoje = date.today()
    lista_urgentes = []

    for sol in solicitacoes_criticas:
        data_criacao = _dim_data_para_date(sol.data_solicitacao)
        dias_pendentes = (hoje - data_criacao).days if data_criacao else 0

        lista_urgentes.append(
            {
                "numero_solicitacao": sol.numero_solicitacao,
                "prioridade": sol.prioridade,
                "status": sol.status,
                "dias_desde_criacao": dias_pendentes,
            }
        )

    data = {
        "projeto": projeto.codigo_projeto,
        "estatisticas": {
            "total_pendentes": total_pendentes,
            "urgentes_criticas": lista_urgentes,
        },
    }
    return JsonResponse(data)
```

</details>

<details>
<summary>Código em Python – listagem_solicitacoes</summary>

```python
@require_GET
def listagem_solicitacoes(request, codigo_projeto):
    projeto = get_object_or_404(DimProjeto, codigo_projeto=codigo_projeto)

    solicitacoes = (
        DimSolicitacao.objects.filter(projeto=projeto)
        .select_related('material', 'data_solicitacao')
        .prefetch_related('fatocompra_set')
        .annotate(
            valor_total_estimado=ExpressionWrapper(
                F('quantidade') * F('material__custo_estimado'),
                output_field=DecimalField(),
            )
        )
    )

    lista_detalhes = []

    for sol in solicitacoes:
        data_sol = _dim_data_para_date(sol.data_solicitacao)

        pedido_vinculado = sol.fatocompra_set.first()
        numero_pedido = pedido_vinculado.numero_pedido if pedido_vinculado else None

        lista_detalhes.append(
            {
                "numero_solicitacao": sol.numero_solicitacao,
                "numero_pedido": numero_pedido,
                "nome_material": sol.material.descricao,
                "data_solicitacao": data_sol.isoformat() if data_sol else None,
                "valor_total_estimado": (
                    float(sol.valor_total_estimado) if sol.valor_total_estimado else 0.0
                ),
                "status": sol.status,
            }
        )

    return JsonResponse(
        {"projeto": projeto.codigo_projeto, "solicitacoes": lista_detalhes}
    )
```

</details>

**Resultado:** A equipe de front-end passou a ter uma fonte única de dados para construir os indicadores e a tabela de solicitações do painel de acompanhamento, sem precisar agregar informações manualmente a partir de múltiplos endpoints. Os testes cobrem requisições sem dados e projeto não encontrado.

---

### Back-end: Listagem de Fornecedores com Filtros

**Situação:** A página de fornecedores precisava de um endpoint que permitisse buscar fornecedores por diferentes critérios, como nome, cidade, categoria, programa e projeto, sem retornar sempre a lista completa cadastrada.

**Tarefa:** Implementar o endpoint `listagem_fornecedores` com suporte a múltiplos filtros de busca combináveis entre si.

**Ação:** Implementei o endpoint aplicando cada filtro recebido por parâmetro somente quando informado, de forma cumulativa. Os filtros de programa e projeto percorrem a relação com as compras, por isso o resultado recebe `distinct()` apenas quando um deles é usado, evitando fornecedores duplicados. Cobri os cenários com testes de integração, incluindo filtros combinados e combinações sem resultado.

<details>
<summary>Código em Python – listagem_fornecedores</summary>

```python
@require_GET
def listagem_fornecedores(request):
    fornecedores = DimFornecedor.objects.all()

    nome = request.GET.get('fornecedor_nome')
    cidade = request.GET.get('fornecedor_cidade')
    categoria = request.GET.get('categoria')
    programa = request.GET.get('programa_nome')
    projeto = request.GET.get('projeto_nome')

    if nome:
        fornecedores = fornecedores.filter(razao_social__icontains=nome)

    if cidade:
        fornecedores = fornecedores.filter(cidade__icontains=cidade)

    if categoria:
        fornecedores = fornecedores.filter(categoria__icontains=categoria)

    if projeto:
        fornecedores = fornecedores.filter(
            fatocompra__solicitacao__projeto__nome_projeto__icontains=projeto
        )

    if programa:
        fornecedores = fornecedores.filter(
            fatocompra__solicitacao__projeto__programa__nome_programa__icontains=programa
        )

    if projeto or programa:
        fornecedores = fornecedores.distinct()

    lista_fornecedores = []
    for f in fornecedores:
        lista_fornecedores.append(
            {
                "id_fornecedor": f.id,
                "codigo_fornecedor": f.codigo_fornecedor,
                "razao_social": f.razao_social,
                "cidade": f.cidade,
                "categoria": f.categoria,
                "status": f.status,
            }
        )

    return JsonResponse(lista_fornecedores, safe=False)
```

</details>

**Resultado:** A página de fornecedores passou a permitir buscas mais específicas no front-end, reduzindo o volume de dados retornado pela API e facilitando a localização de fornecedores pelos gestores.

---

### Front-end: Painel de Acompanhamento de Solicitações (RequestDashboardScreen)

**Situação:** Gestores precisavam acompanhar, em uma única tela, as solicitações de compra em andamento de um projeto, com indicadores agregados (como total de pendências e solicitações urgentes) e uma listagem detalhada de cada solicitação.

**Tarefa:** Desenvolver o componente `RequestDashboardScreen`, integrando os indicadores (`KpiCards`) e uma tabela detalhada (`RequestTable`) que consomem os dados reais da API de solicitações.

**Ação:** Criei o `requestService`, com as funções `getSolicitacoes` e `getSolicitacoesAnalytics`, que consultam os dois endpoints do back-end. A tela busca os dois conjuntos de dados em paralelo com `Promise.all` e exibe uma mensagem de erro caso a busca falhe. A tabela `RequestTable` foi construída sobre o componente `DataGrid` do Material UI, com formatação de moeda, datas e status, e os indicadores do `KpiCards` consomem a estrutura de analytics retornada pela API.

<details>
<summary>Código em TypeScript – requestService.ts</summary>

```typescript
import { api } from './api';

export const getSolicitacoes = async (projectId: string) => {
  const response = await api.get(`/projetos/${projectId}/solicitacoes/detalhes/`);
  return response.data;
};

export const getSolicitacoesAnalytics = async (projectId: string) => {
  const response = await api.get(`/projetos/${projectId}/solicitacoes/stats/`);
  return response.data;
};
```

</details>

<details>
<summary>Código em TypeScript (React) – carregamento dos dados em RequestDashboardScreen</summary>

```tsx
useEffect(() => {
  const fetchData = async () => {
    try {
      setLoading(true);
      setError(false);

      if (!codigo_projeto) return;
      const [listaData, analyticsData] = await Promise.all([
        getSolicitacoes(codigo_projeto),
        getSolicitacoesAnalytics(codigo_projeto),
      ]);

      setSolicitacoes(listaData.solicitacoes || []);
      setAnalytics(analyticsData.estatisticas || null);
    } catch (err) {
      console.error(err);
      setError(true);
    } finally {
      setLoading(false);
    }
  };

  fetchData();
}, [codigo_projeto]);
```

</details>

**Resultado:** O painel passou a refletir os dados reais da API, com indicadores e tabela de solicitações sincronizados e uma mensagem de erro visível ao usuário em caso de falha na busca. Os testes cobrem a renderização dos componentes após o carregamento e a exibição de erro quando a API falha.

---

### Front-end: Modal de Informações do Fornecedor (SupplierInfoModal)

**Situação:** A página de fornecedores listava os fornecedores cadastrados, mas não exibia detalhes sobre os pedidos realizados nem um indicador de confiabilidade com base no histórico de entregas de cada um.

**Tarefa:** Integrar o modal `SupplierInfoModal` a dados reais da API, definindo os tipos de dados necessários e a lógica de classificação de confiabilidade do fornecedor.

**Ação:** Implementei o `supplierService`, com as funções `getSupplierDetail`, `getSupplierOrders` e `listSuppliers`, e defini os tipos `SupplierDetail` e `SupplierOrdersResponse` para tipar essas respostas. Refatorei o modal para buscar os dados do fornecedor e seus pedidos em paralelo, com filtro por projeto aplicado com debounce de 500 ms e filtro local por pedido ou material. A confiabilidade é calculada a partir da taxa de sucesso, isto é, da proporção de pedidos sem atraso: taxa maior ou igual a 75% resulta em "confiável", entre 50% e 75% em "atenção", e abaixo de 50% em "crítico".

<details>
<summary>Código em TypeScript – classificação de confiabilidade</summary>

```typescript
type ReliabilityStatus = 'ok' | 'warning' | 'danger';

const getReliabilityStatus = (totalOrders: number, totalDelays: number): ReliabilityStatus => {
  if (totalOrders === 0) return 'ok';

  const successRate = (totalOrders - totalDelays) / totalOrders;

  if (successRate >= 0.75) return 'ok';       // >= 75%
  if (successRate >= 0.5) return 'warning';   // >= 50% e <= 74.9%
  return 'danger';                            // < 50%
};
```

</details>

<details>
<summary>Código em TypeScript (React) – carregamento de dados do fornecedor</summary>

```tsx
useEffect(() => {
  setLoading(true);

  Promise.all([
    supplierService.getSupplierDetail(supplierId),
    supplierService.getSupplierOrders(supplierId, debouncedProject),
  ])
    .then(([dadosFornecedor, dadosPedidos]) => {
      setFornecedorInfo(dadosFornecedor);
      setPedidosInfo(dadosPedidos);
    })
    .catch((err) => console.error('Erro ao buscar dados do modal:', err))
    .finally(() => setLoading(false));
}, [supplierId, debouncedProject]);
```

</details>

**Resultado:** O modal passou a exibir informações reais de cada fornecedor, incluindo seu histórico de pedidos e um indicador visual de confiabilidade, apoiando a decisão de gestores na escolha de fornecedores para novos pedidos. Os testes cobrem o carregamento, a filtragem local, a chamada à API após o debounce, o fechamento por botão e por tecla Escape, e o ícone de atenção para taxas de sucesso intermediárias.

---

### Front-end: Acompanhamento de Materiais Empenhados (CommitmentMaterial)

**Situação:** O acompanhamento de materiais empenhados em um projeto não oferecia uma visão clara da evolução de custos ao longo do tempo, dificultando a identificação de tendências de gasto.

**Tarefa:** Refatorar o componente `CommitmentMaterial` para consumir dados reais da API e adicionar visualizações gráficas de custo.

**Ação:** Criei o `commitmentService`, com as funções `getAlerts` e `getAnalytics`, que consultam os alertas e os empenhos do projeto. O componente busca os dois endpoints em paralelo e cruza os materiais empenhados com a lista de materiais obsoletos, marcando cada item da tabela com o indicador `isObsoleto`. Para a visualização, implementei o `CommitmentCharts`, com um gráfico de linhas para a evolução do gasto ao longo do tempo e um gráfico de rosca para a distribuição do custo, alternável entre categoria e material.

<details>
<summary>Código em TypeScript (React) – cruzamento de materiais empenhados com obsoletos</summary>

```tsx
const [alertasData, empenhosData] = await Promise.all([
  commitmentService.getAlerts(codigo_projeto),
  commitmentService.getAnalytics(codigo_projeto),
]);

const listaObsoletos = alertasData.alertas_criticos.materiais_obsoletos;

// Lógica de Cruzamento: verifica se o material empenhado está na lista de alertas
const codigosObsoletos = new Set(listaObsoletos.map((obs) => obs.codigo_material));

const materiaisComStatus: MaterialComStatus[] = empenhosData.empenho_por_material.map(
  (mat) => ({
    ...mat,
    fornecedor: mat.fornecedor || 'SIATT Corp',
    isObsoleto: codigosObsoletos.has(mat.codigo_material),
  })
);
```

</details>

<details>
<summary>Código em TypeScript – commitmentService.ts</summary>

```typescript
import { api } from './api';
import type { AlertasResponse, EmpenhosResponse } from '../types/commitment';

export const commitmentService = {
  async getAlerts(codigo_projeto: string): Promise<AlertasResponse> {
    const { data } = await api.get<AlertasResponse>(`/projetos/criticos/${codigo_projeto}`);
    return data;
  },

  async getAnalytics(codigo_projeto: string): Promise<EmpenhosResponse> {
    const { data } = await api.get<EmpenhosResponse>(`/projetos/${codigo_projeto}/empenhos/`);
    return data;
  },
};
```

</details>

**Resultado:** Gestores passaram a visualizar a evolução de custos de materiais empenhados diretamente em gráficos, e a identificar na mesma tela os materiais empenhados que também aparecem como obsoletos. Os testes cobrem o estado de carregamento, o carregamento bem-sucedido e a exibição da tela mesmo quando a API retorna erro.

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
| **5º Semestre – 01/2026** | **Synthesi** | **SIATT** |
| [6º Semestre – 02/2026](API06.md) | IAzimute | AKAER |

[← Voltar ao Portfólio](README.md)
