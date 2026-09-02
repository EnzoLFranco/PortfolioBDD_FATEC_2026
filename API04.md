# API 4º Semestre – 02/2025
## Projeto: Sistema de Mobilidade Urbana 
**Empresa Parceira:** Prefeitura de São José dos Campos

**[Repositório GitHub](https://github.com/SQLutions-FATEC/API-4-Semestre)**

---

## Resumo do Projeto

O Sistema de Mobilidade Urbana foi desenvolvido para a Prefeitura de São José dos Campos como um painel de monitoramento de tráfego urbano em tempo real. O problema central era a ausência de uma ferramenta que consolidasse os dados captados por radares distribuídos pela cidade e os tornasse acessíveis de forma visual e compreensível tanto para gestores municipais quanto para cidadãos.

A solução entregue processa leituras de radares armazenadas em banco de dados em cloud e as transforma em índices de mobilidade (tráfego e segurança) em uma escala de 1 a 5, exibidos em dashboards interativos com mapa de calor por região, alertas automáticos para situações críticas e diferenciação de acesso entre perfis de cidadão e gestor.

Este foi o projeto de maior complexidade de dados da trajetória da equipe — pela primeira vez trabalhamos com dados geoespaciais reais (PostGIS), processamento de leituras de sensores físicos e visualização cartográfica interativa com Leaflet.

---

## Tecnologias Utilizadas

**Back-end:**
- **Java & Spring Boot** — API REST responsável pelo processamento dos índices e regras de negócio
- **PostgreSQL + PostGIS** — Banco de dados relacional com suporte a dados geoespaciais (pontos, polígonos e linhas)
- **Docker** — Conteinerização do banco e da aplicação para padronização de ambientes

**Front-end:**
- **Vue.js + Vite** — Framework principal para construção da interface
- **Vuetify** — Biblioteca de componentes visuais
- **Pinia** — Gerenciamento de estado global da aplicação
- **Vue Router** — Navegação entre páginas por perfil de acesso
- **Axios** — Integração com a API REST
- **MirageJS** — Mock da API para desenvolvimento desacoplado do back-end

---

## Contribuições Individuais

Atuei como Desenvolvedor Full-Stack, com contribuições no front-end (dashboard do cidadão e serviço de índices) e no back-end (modelagem do banco de dados e configuração da API).

---

### Front-end: Serviço de Índices (indexService)

**Situação:** O dashboard precisava consumir índices calculados pelo back-end para três granularidades diferentes — cidade inteira, região específica e radar individual — e cada uma exigia uma chamada distinta à API com parâmetros próprios. Sem uma camada de serviço centralizada, cada componente que precisasse desses dados teria que replicar a lógica de chamada HTTP.

**Tarefa:** Criar um serviço reutilizável em TypeScript que encapsulasse todas as chamadas relacionadas a índices, padronizando os parâmetros e os tipos de retorno para uso em qualquer componente do front-end.

**Ação:** Desenvolvi o `indexService` com três métodos tipados via interface `IndexData`, utilizando Axios via instância centralizada `api`. O parâmetro `minutes` define a janela temporal de cálculo dos índices e possui valor padrão de 5 minutos, permitindo que os componentes o omitam quando não necessário.

<details>
<summary>Código em TypeScript – indexService</summary>

```typescript
import api from "@/services/api";

export interface IndexData {
  combinedIndex: number;
  trafficIndex: number;
  securityIndex: number;
}

export const indexService = {
  async getCityIndex(minutes = 5): Promise<IndexData> {
    const response = await api.get(`/indexes`, {
      params: { minutes },
    });
    return response.data;
  },

  async getRegionIndex(region: string, minutes = 5): Promise<IndexData> {
    const response = await api.get(`/indexes/region`, {
      params: { minutes },
      data: region,
    });
    return response.data;
  },

  async getRadarIndexes(radars: any[], minutes = 5): Promise<IndexData> {
    const response = await api.get(`/indexes/radar`, {
      params: { minutes },
      data: radars,
    });
    return response.data;
  },
};
```

</details>

**Resultado:** O serviço passou a ser o ponto único de acesso aos dados de índices no front-end, eliminando duplicação de lógica HTTP entre componentes e facilitando futuras mudanças nos endpoints da API — qualquer ajuste precisaria ser feito em um único lugar.

---

### Front-end: Dashboard do Cidadão (CitizenHome)

**Situação:** A página principal para o perfil cidadão precisava exibir os índices de mobilidade urbana de forma clara e imediata, com coloração visual por nível de criticidade, mapa interativo com regiões clicáveis e painel lateral com alertas de endereços em nível crítico. O componente precisava reagir tanto à seleção de regiões no mapa quanto ao estado de carregamento dos dados geoespaciais.

**Tarefa:** Desenvolver o componente `CitizenHome` integrando o mapa Leaflet, os cards de índice com coloração dinâmica, o painel de informações críticas e a lógica de exibição do resumo da cidade como estado inicial antes de qualquer interação do usuário.

**Ação:** Implementei o componente utilizando Vue 3 com `<script setup>` e Composition API. Os dados geoespaciais e de índices foram consumidos via composable `useMapData`, que centraliza o carregamento de `addressData`, `radarData`, `regionData` e `citySummaryData` com estados unificados de `isLoading` e `error`.

A função `getIndexClass` mapeia os valores numéricos (1 a 5) para classes CSS de cor, aplicadas dinamicamente nos cards via `:class`. O mapa Leaflet (`MapaLeaflet`) emite o evento `@region-selected` com os dados da região clicada, que são capturados por `handleRegionSelected` e refletem nos cards de índice sem necessidade de rechamada à API. Um `watch` sobre `citySummaryData` garante que o resumo da cidade seja exibido automaticamente assim que os dados chegam, sem exigir interação inicial do usuário.

O painel de informações à direita filtra automaticamente os endereços com `trafficIndex >= 4`, exibindo apenas as situações críticas relevantes para o cidadão.

<details>
<summary>Código em Vue.js – CitizenHome (script setup)</summary>

```typescript
<script setup lang="ts">
import iconeCamera from "@/assets/cam2.png";
import questionMarkIcon from "@/assets/question-mark.png";
import MapaLeaflet from "@/components/MapaLeaflet.vue";
import IndiceModal from "@/components/Modals/IndiceModal.vue";
import { ref, watch } from "vue";
import { useMapData } from "@/composables/useMapData";

const regionColorMap = {
  Norte: "#e6194B",
  Leste: "#3cb44b",
  Centro: "#ffe119",
  Sul: "#4363d8",
  Sudeste: "#f58231",
  Oeste: "#911eb4",
  "São Francisco Xavier": "#46f0f0",
  default: "#a9a9a9",
};

interface regionProps {
  name: string;
  overall: number;
  traffic: number;
  security: number;
  estado: string;
}

const {
  addressData,
  radarData,
  regionData,
  citySummaryData,
  isLoading,
  error,
} = useMapData();

const nomeRegiaoClicada = ref(<string | null>null);
const indiceGeral = ref(<number | null>null);
const indiceTrafego = ref(<number | null>null);
const indiceSeguranca = ref(<number | null>null);
const estadoRegiao = ref(<string | null>null);

function showCitySummary() {
  if (citySummaryData.value) {
    nomeRegiaoClicada.value = citySummaryData.value.name;
    indiceGeral.value = citySummaryData.value.overall;
    indiceTrafego.value = citySummaryData.value.traffic;
    indiceSeguranca.value = citySummaryData.value.security;
    estadoRegiao.value = citySummaryData.value.estado;
  }
}

function getIndexClass(value: number): string {
  switch (value) {
    case 1: return "green";
    case 2: return "yellow";
    case 3: return "orange";
    case 4: return "dark-red";
    case 5: return "red";
    default: return "gray";
  }
}

function handleRegionSelected(regionProps: regionProps | null) {
  if (regionProps) {
    nomeRegiaoClicada.value = regionProps.name;
    indiceGeral.value = regionProps.overall;
    indiceTrafego.value = regionProps.traffic;
    indiceSeguranca.value = regionProps.security;
    estadoRegiao.value = regionProps.estado;
  } else {
    nomeRegiaoClicada.value = null;
    indiceGeral.value = null;
    indiceTrafego.value = null;
    indiceSeguranca.value = null;
    estadoRegiao.value = null;
  }
}

const modalAberto = ref(false);
const tipoModal = ref<"trafego" | "seguranca" | "geral">("trafego");

function abrirModal(tipo: "trafego" | "seguranca" | "geral") {
  tipoModal.value = tipo;
  modalAberto.value = true;
}

watch(
  citySummaryData,
  (newSummary) => {
    if (newSummary && !nomeRegiaoClicada.value) {
      showCitySummary();
    }
  },
  { immediate: true }
);
</script>
```

</details>

**Resultado:** O dashboard entregou uma interface reativa e informativa: o resumo da cidade é exibido automaticamente ao carregar a página, a seleção de regiões no mapa atualiza os cards instantaneamente sem novas chamadas à API, e o painel lateral exibe em tempo real apenas os endereços com situação crítica — reduzindo o volume de informação irrelevante para o cidadão.

---

### Back-end: Modelagem do Banco de Dados (init.sql)

**Situação:** O sistema precisava armazenar e relacionar dados de naturezas muito distintas: leituras de radares físicos com timestamps e tipos de veículo, endereços georreferenciados com geometria de trecho viário, regiões da cidade representadas como polígonos e logs de notificações disparadas para gestores. A modelagem inicial apresentava erros de sintaxe SQL que impediam a inicialização correta do banco via Docker.

**Tarefa:** Corrigir o `init.sql` que inicializa o banco via Docker, ajustando a sintaxe das definições de tabela e garantindo que os tipos ENUM, as chaves estrangeiras e as colunas geoespaciais (PostGIS) estivessem corretamente definidos para que o ambiente subisse sem erros em qualquer máquina da equipe.

**Ação:** Corrigi os erros de sintaxe nas definições das tabelas e padronizei a estrutura do script. As principais intervenções foram: definição dos tipos ENUM (`tipo_veiculo` e `nivel_usuario`) antes das tabelas que os referenciam; ajuste das colunas geoespaciais usando os tipos corretos do PostGIS (`GEOMETRY(Point, 4326)`, `GEOMETRY(LineString, 4326)`, `GEOMETRY(Polygon, 4326)`); e correção das constraints de chave estrangeira entre `radar → endereco` e `leitura → radar`, garantindo integridade referencial desde a inicialização.

<details>
<summary>SQL – init.sql (tabelas principais)</summary>

```sql
\c api;

CREATE TYPE tipo_veiculo AS ENUM (
    'Carro', 'Camionete', 'Ônibus', 'Van',
    'Caminhão grande', 'Moto', 'Indefinido'
);

CREATE TYPE nivel_usuario AS ENUM ('Admin', 'Gestor');

CREATE TABLE endereco (
    id    SERIAL PRIMARY KEY,
    ende  VARCHAR(150) NOT NULL UNIQUE,
    bairro VARCHAR(50),
    regiao VARCHAR(30),
    trecho GEOMETRY(LineString, 4326)
);

CREATE TABLE radar (
    id      VARCHAR(9) PRIMARY KEY,
    id_end  INT NOT NULL,
    localizacao GEOMETRY(Point, 4326),
    vel_reg INT NOT NULL,
    CONSTRAINT fk_radar_endereco FOREIGN KEY (id_end) REFERENCES endereco(id)
);

CREATE TABLE leitura (
    id      SERIAL PRIMARY KEY,
    id_rad  VARCHAR(9) NOT NULL,
    dat_hora TIMESTAMP NOT NULL,
    tip_vei  tipo_veiculo NOT NULL,
    vel      INT NOT NULL,
    CONSTRAINT fk_leitura_radar FOREIGN KEY (id_rad) REFERENCES radar(id)
);

CREATE TABLE usuario (
    id    SERIAL PRIMARY KEY,
    nome  VARCHAR(100) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    senha VARCHAR(100) NOT NULL,
    nivel nivel_usuario NOT NULL
);

CREATE TABLE log_notificacao (
    id            SERIAL PRIMARY KEY,
    id_usuario    INT NOT NULL,
    titulo        VARCHAR(50) NOT NULL,
    mensagem      TEXT NOT NULL,
    data_emissao  TIMESTAMP DEFAULT NOW(),
    data_conclusao TIMESTAMP,
    CONSTRAINT fk_log_usuario FOREIGN KEY (id_usuario)
        REFERENCES usuario(id) ON DELETE CASCADE
);

CREATE TABLE regioes (
    id          SERIAL PRIMARY KEY,
    nome_regiao VARCHAR(100) NOT NULL UNIQUE,
    area_regiao GEOMETRY(Polygon, 4326) NOT NULL
);

CREATE TABLE pontos_onibus (
    id    BIGINT PRIMARY KEY,
    ponto GEOMETRY(Point, 4326) NOT NULL
);
```

</details>

**Resultado:** O banco passou a inicializar corretamente via `docker compose up` em qualquer ambiente da equipe, eliminando a necessidade de ajustes manuais no banco após subir o container. A modelagem geoespacial correta com PostGIS viabilizou todas as funcionalidades de mapa e filtragem por região implementadas nas sprints seguintes.

---

### Back-end: Configuração de CORS e Endpoint de Índices por Endereço

**Situação:** Com front-end e back-end rodando em portas distintas durante o desenvolvimento (5173 e 8080), as requisições do Vue.js ao Spring Boot eram bloqueadas pelo navegador por violação de CORS. Além disso, o endpoint de índices precisava suportar consulta por endereço específico com suporte a timestamp opcional para consultas históricas.

**Tarefa:** Configurar o CORS no Spring Boot para liberar as origens do front-end, e implementar o endpoint `GET /indexes/address` com parâmetro de timestamp opcional para suporte a consultas históricas.

**Ação:** Implementei a classe `CorsConfig` usando `WebMvcConfigurer`, liberando as origens de desenvolvimento (`localhost:3000` e `localhost:80`) para todos os métodos HTTP necessários. No controller, o parâmetro `timestamp` foi declarado como `required = false` — quando ausente, o sistema utiliza o horário atual "clampado" ao banco via `timeService.getCurrentTimeClampedToDatabase()`, garantindo que consultas sem timestamp sempre retornem dados válidos do período disponível.

<details>
<summary>Código em Java – CorsConfig e endpoint de índices por endereço</summary>

```java
// Configuração de CORS
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins(
                        "http://localhost:3000",
                        "http://localhost:80")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .allowCredentials(true);
    }
}

// Endpoint de índices por endereço
@GetMapping("/address")
public Index getAddressIndexes(
        @RequestParam(defaultValue = "1") int minutes,
        @RequestParam String address,
        @RequestParam(required = false) java.time.LocalDateTime timestamp) {
    return indexService.getAddressIndexes(
            minutes,
            address,
            timestamp == null
                ? timeService.getCurrentTimeClampedToDatabase()
                : timestamp);
}
```

</details>

**Resultado:** As chamadas do front-end ao back-end passaram a funcionar corretamente em ambiente de desenvolvimento sem necessidade de proxies ou workarounds. O suporte a timestamp opcional no endpoint de índices por endereço viabilizou consultas históricas sem quebrar o fluxo padrão de uso em tempo real.

---

## Hard Skills (Autoavaliação)

| Tecnologia / Metodologia | Nível | Classificação |
|--------------------------|-------|---------------|
| Vue.js + TypeScript | ★★★★☆ | Sei fazer com ajuda |
| Spring Boot (Java) | ★★★☆☆ | Sei fazer com ajuda |
| PostgreSQL / PostGIS | ★★★☆☆ | Entendi |
| Docker | ★★★☆☆ | Entendi |
| Metodologia Ágil (Scrum) | ★★★★★ | Sei fazer com autonomia |

---

## Soft Skills

- **Comunicação**
  O projeto envolveu dados geoespaciais — um domínio novo para toda a equipe. Precisei compreender os tipos do PostGIS (`GEOMETRY`, `Point`, `LineString`, `Polygon`) e comunicar para o time as dependências entre as tabelas no `init.sql`, especialmente a ordem de criação necessária para que as chaves estrangeiras funcionassem corretamente. Isso exigiu clareza na explicação de decisões técnicas que impactavam o ambiente de todos.

- **Trabalho em equipe**
  O front-end foi desenvolvido de forma desacoplada do back-end usando MirageJS para mockar a API. Esse acordo de equipe exigiu alinhamento constante sobre os contratos de dados — quais campos cada endpoint retornaria e em qual formato — para que os mocks refletissem o comportamento real da API e a integração final não gerasse retrabalho.

- **Proatividade**
  Os erros de sintaxe no `init.sql` impediam que qualquer membro da equipe subisse o ambiente localmente. Ao identificar o problema, mapeei todas as inconsistências do script, corrigi e testei a inicialização via Docker antes de abrir o pull request, desbloqueando o restante da equipe para continuar o desenvolvimento.

---

## Aprendizados Efetivos

Este foi o projeto que mais me aproximou de um sistema com dados reais e complexidade geoespacial. Trabalhar com PostGIS foi um aprendizado completamente novo — compreender que coordenadas geográficas têm um sistema de referência (`4326` para WGS84), que polígonos representam regiões e pontos representam radares, e que essas geometrias podem ser consultadas e filtradas diretamente no banco foi uma expansão significativa do que eu entendia por "banco de dados".

No front-end, a adoção do padrão de composables (`useMapData`) para centralizar o carregamento de dados geoespaciais me ensinou uma forma mais robusta de organizar lógica reutilizável em Vue 3 — indo além dos componentes e serviços que utilizei nos semestres anteriores. A evolução de Ant Design (API03) para Vuetify, e de estado local para Pinia, também consolidou minha visão sobre como escolhas de stack impactam a organização e a escalabilidade de um projeto front-end.

---

## Navegação


| Semestre | Projeto | Empresa Parceira |
|----------|---------|-----------------|
| [1º Semestre – 01/2024](API01.md) | Calculadora Científica | FATEC São José dos Campos |
| [2º Semestre – 02/2024](API02.md) | Avaliador de Soft Skills (PACER) | FATEC São José dos Campos |
| [3º Semestre – 01/2025](API03.md) | Checkpoint | Altave |
[← Voltar ao Portfólio](README.md)
