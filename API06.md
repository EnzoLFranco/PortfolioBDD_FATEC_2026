# API 6º Semestre – 02/2026
## Projeto: IAzimute
**Empresa Parceira:** AKAER

**[Repositório GitHub (Back-end)](https://github.com/SQLutionsFATEC/API-6-Semestre-Backend)**
**[Repositório GitHub (Front-end)](https://github.com/SQLutionsFATEC/API-6-Semestre-Frontend)**

---

## Resumo do Projeto

O IAzimute foi desenvolvido para a AKAER como uma plataforma de busca, classificação e visualização de documentos técnicos, com foco em conformidade com a LGPD e controle de acesso por nível de confidencialidade. O problema central era a ausência de um sistema estruturado para organizar e recuperar documentos internos com controle de acesso e rastreabilidade.

A solução permite o upload de documentos nos formatos PDF e DOCX, classificação automática por Machine Learning, busca semântica sobre o conteúdo dos documentos, visualização segura com restrições de impressão e download, e autenticação integrada via Keycloak com controle de acesso por perfil de usuário.

---

## Tecnologias Utilizadas

**Back-end:**
- **Python & Django** — API REST responsável pelo processamento de documentos e controle de acesso
- **PostgreSQL** — Banco de dados relacional (migrado de MySQL durante o projeto), com extensão pgvector para busca semântica
- **Keycloak** — Servidor de identidade e autenticação via OpenID Connect (Authorization Code + PKCE)
- **Docker** — Conteinerização do banco, back-end, front-end e Keycloak
- **drf-spectacular** — Geração automática de documentação OpenAPI/Swagger
- **Pytest / Pytest-Django** — Testes unitários e de integração
- **python-docx e PyMuPDF** — Extração de texto de arquivos DOCX e PDF
- **Tesseract OCR** — Extração de texto de PDFs escaneados

**Front-end:**
- **React (JSX)** — Framework principal da interface
- **keycloak-js** — Biblioteca de autenticação com Keycloak no navegador
- **pdfjs-dist** — Renderização de PDFs no navegador
- **Vitest / React Testing Library** — Testes de componentes e serviços

**Gestão:**
- **Jira** — Gestão de backlog e sprints
- **Figma** — Prototipação de telas

---

## Contribuições Individuais

Atuei como Desenvolvedor Full-Stack, com contribuições nos dois repositórios do projeto: implementei as rotas de visualização e upload de documentos no back-end Django, realizei a migração de MySQL para PostgreSQL, documentei a API com Swagger, integrei a autenticação Keycloak no front-end e no back-end, e desenvolvi o visualizador de documentos com restrições de segurança no front-end.

---

### Back-end: Rota de Visualização de Documento

**Situação:** Não havia nenhuma rota para recuperar os dados de um documento específico pelo seu identificador. A aplicação precisava de um endpoint que retornasse os metadados e o conteúdo do documento de forma controlada, com validação de identificador e tratamento de casos de erro.

**Tarefa:** Implementar a rota `GET /api/documentos/{id_documento}/`, com retorno dos dados do documento, tratamento de documento não encontrado e controle de acesso com base no setor e no nível do usuário.

**Ação:** Criei o `DocumentoViewSet` baseado em `ModelViewSet`, com o `DocumentoSerializer`, e registrei a rota via `DefaultRouter` em `urls.py`. O identificador do documento é usado como `lookup_field`. No método `retrieve`, o acesso é calculado pela função `verificar_acesso`, e, quando o usuário não tem permissão, o campo `data` é removido da resposta em vez de retornar um erro HTTP. Assim, a resposta mantém status 200 e o front-end decide como exibir o caso de acesso negado.

<details>
<summary>Código em Python – DocumentoViewSet.retrieve</summary>

```python
def retrieve(self, request, *args, **kwargs):
    documento = self.get_object()

    setor_usuario = getattr(request.user, 'setor', None)
    nivel_usuario = getattr(request.user, 'nivel', None)

    acesso_permitido = verificar_acesso(
        setor_usuario, nivel_usuario, documento.setor, documento.nivel
    )

    serializer = self.get_serializer(documento)
    dados = serializer.data
    if not acesso_permitido:
        dados.pop('data', None)

    return Response(dados, status=status.HTTP_200_OK)
```

</details>

Para a regra de negócio, a função `verificar_acesso` aplica a seguinte lógica: o usuário com nível `Operador` acessa qualquer documento; os demais acessam apenas quando o setor coincide com o do documento e o nível é igual ou superior ao exigido pelo documento.

<details>
<summary>Código em Python – verificar_acesso</summary>

```python
def verificar_acesso(setor_usuario, nivel_usuario, setor_documento, nivel_documento):
    setor_usuario = setor_usuario or SETOR_PADRAO
    nivel_usuario = nivel_usuario or NIVEL_PADRAO

    if nivel_usuario == 'Operador':
        return True

    nivel_usuario_int = NIVEL_HIERARQUIA.get(nivel_usuario, 0)
    nivel_documento_int = NIVEL_HIERARQUIA.get(nivel_documento, 0)

    if setor_usuario == setor_documento and nivel_usuario_int >= nivel_documento_int:
        return True

    return False
```

</details>

**Resultado:** A rota passou a ser o ponto único de acesso a documentos individuais, com a semântica de controle de acesso incorporada à resposta. Um identificador inexistente ou inválido retorna 404, coberto pelos testes `test_retorna_404_para_documento_inexistente` e `test_retorna_404_para_id_invalido`. O front-end não precisa tratar 403 para o caso de acesso negado, pois basta verificar o campo `acesso_permitido`.

---

### Back-end: Upload de Documentos com Classificação Automática

**Situação:** A aplicação precisava receber documentos nos formatos PDF e DOCX com metadados obrigatórios, validar o tipo do arquivo e classificar o conteúdo para atribuir setor e etiqueta, sem que o usuário precisasse informar essas categorias manualmente.

**Tarefa:** Implementar a rota de upload `POST /api/documentos/` com validação de metadados obrigatórios (`nome` e `nivel`), inferência de `tipo_arquivo` pela extensão do arquivo e classificação automática do conteúdo para definir o setor.

**Ação:** Refatorei o `DocumentoSerializer` para tornar `tipo_arquivo` e `setor` opcionais, substituir o campo de etiquetas por um `ListField` de strings somente para escrita, e inferir o tipo do arquivo pela extensão usando o dicionário `TIPOS_ARQUIVO_PERMITIDOS`. O método `validate()` exige `nome` e `nivel` na criação e confere se o `tipo_arquivo` informado corresponde à extensão real do arquivo.

<details>
<summary>Código em Python – DocumentoSerializer.validate</summary>

```python
def validate(self, attrs):
    attrs = super().validate(attrs)

    for campo in ('nome', 'nivel'):
        valor = attrs.get(campo)
        if not self.partial and (not isinstance(valor, str) or not valor.strip()):
            raise serializers.ValidationError({campo: 'Este campo é obrigatório.'})

    if 'setor' in attrs and isinstance(attrs['setor'], str):
        attrs['setor'] = attrs['setor'].strip()

    arquivo = attrs.get('data')
    if arquivo:
        extensao = arquivo.name.rsplit('.', 1)[-1].lower()
        tipo_inferido = TIPOS_ARQUIVO_PERMITIDOS[f'.{extensao}']
        tipo_informado = attrs.get('tipo_arquivo')
        if tipo_informado and tipo_informado.lower() != tipo_inferido:
            raise serializers.ValidationError({
                'tipo_arquivo': 'O tipo informado deve corresponder à extensão do arquivo.'
            })
        attrs['tipo_arquivo'] = tipo_inferido

    etiquetas = attrs.get('etiquetas') or []
    attrs['etiquetas'] = [
        nome.strip()
        for nome in etiquetas
        if isinstance(nome, str) and nome.strip()
    ]
    return attrs
```

</details>

A classificação é executada dentro de `perform_create`, em uma transação atômica. O setor e a etiqueta são definidos pelo resultado do modelo, e não pelo valor enviado no upload. Se o modelo não identificar a categoria, o documento recebe o valor `NAO_CLASSIFICADO`.

<details>
<summary>Código em Python – DocumentoViewSet.perform_create</summary>

```python
def perform_create(self, serializer):
    try:
        with transaction.atomic():
            documento = serializer.save()
            caminho_arquivo = documento.data.path if (documento.data and hasattr(documento.data, 'path')) else ""

            setor = (
                MLService.classificar_documento(caminho_arquivo) or 'NAO_CLASSIFICADO'
            ).strip() or 'NAO_CLASSIFICADO'

            # Setor e etiqueta são definidos exclusivamente pela classificação.
            # Valores enviados no upload não devem sobrescrever a categoria detectada.
            documento.setor = setor
            documento.save(update_fields=['setor'])

            etiqueta, _ = Etiqueta.objects.get_or_create(nome=setor)
            documento.etiquetas.add(etiqueta)

            VectorService.processar_documento(documento, caminho_arquivo)

    except Exception as e:
        raise serializers.ValidationError(
            {"erro": f"Erro ao processar o upload do documento: {str(e)}"}
        )
```

</details>

Para dar suporte a DOCX, estendi o `MLService.classificar_documento` e o `VectorService`. No primeiro, o texto é extraído dos parágrafos e das células de tabela; no segundo, o método `_extrair_docx_por_pagina` gera os trechos usados na busca semântica.

<details>
<summary>Código em Python – extração de DOCX no MLService</summary>

```python
caminho = Path(caminho_pdf)
if caminho.suffix.lower() == '.docx':
    from docx import Document as WordDocument

    documento_word = WordDocument(str(caminho))
    partes = [paragrafo.text for paragrafo in documento_word.paragraphs]
    partes.extend(
        celula.text
        for tabela in documento_word.tables
        for linha in tabela.rows
        for celula in linha.cells
    )
    texto = '\n'.join(parte for parte in partes if parte)
else:
    texto = cls.extrair_texto_pdf(str(caminho))
```

</details>

<details>
<summary>Código em Python – VectorService._extrair_docx_por_pagina</summary>

```python
@staticmethod
def _extrair_docx_por_pagina(caminho_docx: str) -> List[Tuple[int, str]]:
    from docx import Document as WordDocument

    documento_word = WordDocument(caminho_docx)
    partes = [paragrafo.text for paragrafo in documento_word.paragraphs]
    partes.extend(
        celula.text
        for tabela in documento_word.tables
        for linha in tabela.rows
        for celula in linha.cells
    )
    texto = '\n'.join(parte for parte in partes if parte).strip()
    return [(1, texto)] if texto else []
```

</details>

**Resultado:** O endpoint passou a aceitar PDFs e DOCXs com metadados obrigatórios, inferir o tipo do arquivo automaticamente e atribuir o setor pela classificação do conteúdo. Os testes cobrem o upload com classificação, o documento sem classificação identificável (`NAO_CLASSIFICADO`), o DOCX com tipo inferido, a rejeição de metadados ausentes e a falha no processamento, que retorna 400.

---

### Back-end: Migração de MySQL para PostgreSQL

**Situação:** O banco de dados utilizado era MySQL. A mudança para PostgreSQL foi necessária para alinhar a infraestrutura com os requisitos do projeto, incluindo a busca semântica com pgvector, que depende de PostgreSQL.

**Tarefa:** Migrar toda a infraestrutura de banco de dados (serviços Docker, variáveis de ambiente, dependências Python e configurações Django) de MySQL para PostgreSQL, mantendo o ambiente de testes funcional.

**Ação:** Substituí o serviço MySQL pelo serviço PostgreSQL nos arquivos Docker e troquei as variáveis de ambiente correspondentes. Na configuração do Django, o motor passou a ser `postgresql`, e a dependência `mysqlclient` foi substituída por `psycopg[binary]` no `requirements.txt`. Criei o `deploy/postgres/Dockerfile`, baseado na imagem `pgvector/pgvector:pg16`, e o script `init.sh`, que concede ao usuário da aplicação a permissão `CREATEDB` necessária para o Django criar o banco de testes e habilita a extensão `vector`.

<details>
<summary>Código em Bash – deploy/postgres/init.sh</summary>

```bash
#!/bin/bash
set -e

# Permite que o usuário da aplicação crie o banco de testes do Django.
psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
    CREATE EXTENSION IF NOT EXISTS vector;
    ALTER ROLE "$POSTGRES_USER" CREATEDB;
EOSQL

# Mantém a extensão disponível também em bancos de teste criados a partir do template.
psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname template1 <<-EOSQL
    CREATE EXTENSION IF NOT EXISTS vector;
EOSQL
```

</details>

<details>
<summary>Código em Python – configuração do banco no settings.py</summary>

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', os.environ.get('POSTGRES_DB', 'api_db')),
        # ...
    }
}
```

</details>

**Resultado:** Toda a stack de desenvolvimento, testes e produção passou a rodar sobre PostgreSQL, com o script de inicialização garantindo a permissão necessária aos testes automatizados. A troca foi feita nos arquivos de infraestrutura e dependências, sem alterar a lógica de negócio.

---

### Back-end: Documentação Swagger com drf-spectacular

**Situação:** A API não possuía documentação interativa, o que dificultava o entendimento das rotas pelo time de front-end e por integrantes novos no projeto.

**Tarefa:** Integrar o `drf-spectacular` ao projeto Django para gerar documentação OpenAPI automática, expor uma interface Swagger UI e documentar as rotas com parâmetros, payloads e respostas.

**Ação:** Registrei as rotas `api/schema/` e `api/docs/` em `urls.py` e restringi os métodos HTTP do ViewSet a `get`, `post`, `patch`, `delete`, `head` e `options`. Nas ações do `DocumentoViewSet`, usei `@extend_schema` para documentar parâmetros de consulta, payloads e códigos de resposta, como na listagem com filtros combinados. Também escrevi o manual `docs/api/manual-documentacao-rotas.md`, com orientações para documentar novas rotas no projeto.

<details>
<summary>Código em Python – rotas de documentação em urls.py</summary>

```python
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView

urlpatterns = [
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    # ...
    path('api/', include(router.urls)),
]
```

</details>

<details>
<summary>Código em Python – documentação de parâmetros com @extend_schema</summary>

```python
@extend_schema(
    summary="Lista documentos com filtros combinados",
    description=(
        "Retorna documentos paginados. Os filtros de nome, etiquetas, "
        "data_atualizacao e setor são combinados com lógica AND. "
        "Cada documento inclui o campo 'acesso_permitido' (boolean) para o frontend "
        "saber se o usuário tem acesso àquele documento."
    ),
    parameters=[
        OpenApiParameter(
            name="nome",
            type=OpenApiTypes.STR,
            location=OpenApiParameter.QUERY,
            description="Filtra documentos pelo nome, sem diferenciar maiúsculas e minúsculas.",
            required=False,
        ),
        OpenApiParameter(
            name="page",
            type=OpenApiTypes.INT,
            location=OpenApiParameter.QUERY,
            description="Número da página. Cada página contém até 10 documentos.",
            required=False,
            default=1,
        ),
    ],
    responses={
        200: OpenApiResponse(description="Lista paginada de documentos com o campo 'acesso_permitido'."),
        400: OpenApiResponse(description="Número de página inválido."),
        401: OpenApiResponse(description="Usuário não autenticado."),
        404: OpenApiResponse(description="Página solicitada inexistente."),
    },
)
def list(self, request, *args, **kwargs):
    ...
```

</details>

**Resultado:** A API passou a ter documentação interativa acessível em `api/docs/`, com os contratos de cada rota visíveis para o time de front-end sem necessidade de consultar o código-fonte diretamente. O manual deixou registrado o padrão de documentação para as próximas rotas.

---

### Back-end: Integração de Autenticação Keycloak

**Situação:** A autenticação precisava ser validada em duas camadas. No front-end, o usuário faz login no Keycloak. No back-end, cada requisição chega com um token JWT, que precisa ser verificado sem depender de uma consulta ao Keycloak a cada chamada. Em ambiente Docker, o endereço interno do Keycloak é diferente do endereço público do emissor do token, o que gerava falhas de validação de issuer.

**Tarefa:** Implementar a validação local de access tokens JWT (RS256) a partir do JWK Set publicado pelo Keycloak, separando a URL interna usada para buscar as chaves da URL pública usada para validar o emissor (`iss`).

**Ação:** Criei o módulo `api/authentication/keycloak.py` com a classe `KeycloakAuthentication`. O token é lido do header `Authorization: Bearer`, a chave é localizada pelo `kid` do header do JWT e a assinatura, o algoritmo, o emissor e a expiração são verificados. O JWK Set fica em cache com um tempo de vida configurável, e, caso o `kid` não seja encontrado, o cache é recarregado para tratar a rotação de chaves. A busca das chaves usa `KEYCLOAK_URL`, enquanto a validação do emissor usa `KEYCLOAK_ISSUER`.

<details>
<summary>Código em Python – KeycloakAuthentication.validar_token</summary>

```python
def validar_token(self, token):
    """Valida assinatura, algoritmo, emissor e expiração; retorna os claims."""
    try:
        header = jwt.get_unverified_header(token)
    except JOSEError:
        raise AuthenticationFailed('Token de acesso inválido.')

    if header.get('alg') != 'RS256':
        raise AuthenticationFailed('Token assinado com algoritmo não suportado.')

    jwk_chave = obter_jwk(header.get('kid'))
    if jwk_chave is None:
        # O 'kid' pode ser novo após rotação de chaves: força recarga do cache.
        limpar_cache_jwks()
        jwk_chave = obter_jwk(header.get('kid'))
        if jwk_chave is None:
            raise AuthenticationFailed('Token de acesso inválido.')

    try:
        claims = jwt.decode(
            token,
            jwk_chave,
            algorithms=['RS256'],
            issuer=settings.KEYCLOAK_ISSUER,
            options={'verify_aud': False},
        )
    except JOSEError:
        raise AuthenticationFailed('Token de acesso inválido ou expirado.')

    return claims
```

</details>

<details>
<summary>Código em Python – separação das URLs no settings.py</summary>

```python
KEYCLOAK_URL = os.environ.get('KEYCLOAK_URL', 'http://keycloak:8080').rstrip('/')
KEYCLOAK_REALM = os.environ.get('KEYCLOAK_REALM', 'sqlutions')
KEYCLOAK_ISSUER = os.environ.get(
    'KEYCLOAK_ISSUER',
    f'http://localhost:8080/realms/{KEYCLOAK_REALM}',
)
KEYCLOAK_JWKS_TTL = int(os.environ.get('KEYCLOAK_JWKS_TTL', '300'))
```

</details>

**Resultado:** O back-end passou a validar corretamente os tokens emitidos pelo Keycloak em ambiente Docker, sem erros de issuer mismatch, e sem consultar o Keycloak em cada requisição. O cache do JWK Set reduziu a carga sobre o servidor de identidade.

---

### Front-end: Visualizador de Documentos com Restrições de Segurança

**Situação:** Documentos visualizados na plataforma abriam em uma nova aba do navegador, sem qualquer restrição de segurança. O usuário podia salvar, imprimir ou copiar o conteúdo livremente, o que conflitava com o controle de acesso por nível de confidencialidade implementado no back-end.

**Tarefa:** Desenvolver um visualizador de PDFs embutido na aplicação, com zoom ajustável entre 50% e 400%, bloqueio do menu de contexto (clique direito) e ausência de botões de download e impressão.

**Ação:** Implementei o componente `PdfViewer` usando `pdfjs-dist` para renderizar cada página do PDF em um `<canvas>`, substituindo a abertura em nova aba. O zoom é controlado por incrementos de 25 pontos percentuais, limitados pelas constantes `MIN_ZOOM` e `MAX_ZOOM`. Quando o arquivo vem da própria API, o token de acesso é enviado no header `Authorization` da requisição feita pelo pdf.js. O menu de contexto é bloqueado no contêiner do visualizador, e não há botões de download ou impressão na interface.

<details>
<summary>Código em JavaScript (React) – controle de zoom e bloqueio de clique direito no PdfViewer</summary>

```jsx
const MIN_ZOOM = 50;
const MAX_ZOOM = 400;
const ZOOM_STEP = 25;

function handleZoomOut() {
  setZoom((currentZoom) =>
    Math.max(MIN_ZOOM, currentZoom - ZOOM_STEP)
  );
}

function handleZoomIn() {
  setZoom((currentZoom) =>
    Math.min(MAX_ZOOM, currentZoom + ZOOM_STEP)
  );
}

return (
  <section
    className="pdf-viewer ..."
    onContextMenu={(event) => event.preventDefault()}
  >
    {/* cabeçalho com botões de zoom e fechar; sem download ou impressão */}
  </section>
);
```

</details>

<details>
<summary>Código em JavaScript (React) – carregamento do PDF com token</summary>

```jsx
const pdfUrl = new URL(document.data, window.location.origin);
const apiOrigin = new URL(
  import.meta.env.VITE_API_URL || "http://localhost:8000",
  window.location.origin
).origin;
const requestOptions = { url: document.data };

if (pdfUrl.origin === apiOrigin) {
  const token = await getAccessToken();
  requestOptions.httpHeaders = {
    Authorization: `Bearer ${token}`,
  };
}

loadingTask = pdfjs.getDocument({
  ...requestOptions,
});
```

</details>

**Resultado:** Documentos confidenciais passaram a ser visualizados dentro da própria aplicação, sem botão de download ou impressão e com o menu de contexto bloqueado. O visualizador só recebe o arquivo com um token válido, mantendo a mesma autenticação usada nas demais requisições à API.

---

### Front-end: Tratamento de Acesso Negado no Card do Documento

**Situação:** O campo `acesso_permitido` passou a ser retornado pela API para cada documento, mas a interface ainda precisava diferenciar visualmente os documentos restritos e impedir a abertura de seu conteúdo, sem disparar requisições desnecessárias ao arquivo.

**Tarefa:** Sinalizar no card os documentos com acesso restrito, com ícone de cadeado e destaque visual, e exibir uma mensagem explicativa ao usuário quando ele tentar visualizar um documento bloqueado.

**Ação:** No componente `DocumentItem`, a variável `isRestricted` é calculada a partir de `acesso_permitido === false`. Quando verdadeira, o card recebe uma classe de destaque, o botão de visualização passa a exibir um cadeado e o clique é tratado sem abrir o documento. Na página `Home`, a mesma verificação aciona o componente `RestrictedModal`, que informa ao usuário que ele não possui nível ou setor adequado para visualizar o arquivo.

<details>
<summary>Código em JavaScript (React) – DocumentItem com verificação de acesso</summary>

```jsx
function DocumentItem({ document, onView, onOpenDetails }) {
  const isRestricted = document.acesso_permitido === false;

  const handleActionClick = (e) => {
    e.stopPropagation();

    if (isRestricted) {
      if (onView) {
        onView(document);
      }
      return;
    }

    if (onView) {
      onView(document);
    }
  };

  return (
    <div
      className={`document-item cursor-pointer ${
        isRestricted ? "document-restricted" : ""
      }`}
    >
      {/* ... */}
      <button
        type="button"
        onClick={handleActionClick}
        title={
          isRestricted
            ? "Acesso Restrito - Clique para mais informações"
            : "Visualizar documento"
        }
      >
        {isRestricted ? <FiLock size={17} /> : <FiEye size={17} />}
      </button>
    </div>
  );
}
```

</details>

**Resultado:** O usuário passou a identificar de imediato os documentos restritos e recebe uma explicação clara ao tentar abri-los, sem que o front-end precise tratar erros HTTP para esse caso.

---

### Front-end: Integração de Autenticação Keycloak

**Situação:** A aplicação não possuía autenticação. Qualquer usuário podia acessar os documentos sem identificação, tornando o controle de acesso por nível de confidencialidade ineficaz.

**Tarefa:** Integrar o Keycloak ao front-end usando `keycloak-js`, implementando o fluxo Authorization Code + PKCE, com tokens mantidos em memória, renovados próximo à expiração e enviados em todas as requisições à API como `Authorization: Bearer <token>`.

**Ação:** Criei o módulo `src/services/keycloak.js`, com a inicialização do Keycloak no modo `login-required` e PKCE com `S256`. A função `getAccessToken` renova o token quando faltam menos de 30 segundos para a expiração. No `api.js`, um interceptor de requisição adiciona o header `Authorization` apenas às chamadas para a origem da API, e um interceptor de resposta encerra a sessão local e reinicia o login quando a API retorna 401. Em `main.jsx`, a aplicação só é montada após a autenticação.

<details>
<summary>Código em JavaScript – keycloak.js (inicialização e renovação de token)</summary>

```javascript
import Keycloak from 'keycloak-js';

export const keycloak = new Keycloak({
  url: import.meta.env.VITE_KEYCLOAK_URL || 'http://localhost:8080',
  realm: import.meta.env.VITE_KEYCLOAK_REALM || 'sqlutions',
  clientId: import.meta.env.VITE_KEYCLOAK_CLIENT_ID || 'frontend',
});

export async function initializeKeycloak() {
  const authenticated = await keycloak.init({
    onLoad: 'login-required',
    pkceMethod: 'S256',
    checkLoginIframe: false,
  });

  if (!authenticated) {
    redirectToLogin();
    return false;
  }

  return true;
}

export async function getAccessToken() {
  try {
    await keycloak.updateToken(30);
  } catch (error) {
    keycloak.clearToken();
    redirectToLogin();
    throw error;
  }

  if (!keycloak.token) {
    const error = new Error('Não há um token de acesso válido.');
    redirectToLogin();
    throw error;
  }

  return keycloak.token;
}
```

</details>

<details>
<summary>Código em JavaScript – interceptors do Axios em api.js</summary>

```javascript
const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8000',
});

api.interceptors.request.use(async (config) => {
  if (!isApiRequest(config)) return config;

  const token = await getAccessToken();
  config.headers.Authorization = `Bearer ${token}`;
  return config;
});

api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (
      error.response?.status === 401 &&
      error.config &&
      isApiRequest(error.config)
    ) {
      restartLoginAfterUnauthorized();
    }

    return Promise.reject(error);
  },
);
```

</details>

**Resultado:** O acesso à aplicação passou a exigir autenticação via Keycloak, com tokens Bearer enviados automaticamente em todas as chamadas à API. Os testes automatizados cobrem a inicialização com PKCE, a renovação de token e o reinício do login após resposta 401, usando um mock do `keycloak-js` para que os componentes continuassem testáveis sem um servidor real.

---

### Front-end: Atualização da Listagem após Upload

**Situação:** Depois de um upload bem-sucedido, a lista de documentos da tela principal não refletia o novo arquivo até que a página fosse recarregada.

**Tarefa:** Forçar a re-listagem dos documentos após o upload, sem recarregar a aplicação inteira.

**Ação:** Criei um contador `documentsRefreshKey` no componente `App`, passado como prop para a `Home`. A `Home` usa essa prop como dependência da busca de documentos, então qualquer incremento do contador dispara uma nova requisição à listagem.

<details>
<summary>Código em JavaScript (React) – App.jsx e Home.jsx</summary>

```jsx
// App.jsx
const [documentsRefreshKey, setDocumentsRefreshKey] = useState(0);

return <Home documentsRefreshKey={documentsRefreshKey} />;

// pages/Home/Home.jsx
function Home({ documentsRefreshKey = 0 }) {
  // documentsRefreshKey entra nas dependências do carregamento de documentos
}
```

</details>

**Resultado:** A listagem passou a refletir os uploads concluídos sem recarregamento manual da página.

---

## Hard Skills (Autoavaliação)

| Tecnologia / Metodologia | Nível | Classificação |
|--------------------------|-------|---------------|
| Python + Django | ★★★★★ | Sei fazer com autonomia |
| React (JSX) | ★★★★★ | Sei fazer com autonomia |
| PostgreSQL / Modelagem Relacional | ★★★★☆ | Sei fazer com ajuda |
| Docker | ★★★★☆ | Sei fazer com ajuda |
| Keycloak / OpenID Connect | ★★★★☆ | Sei fazer com ajuda |
| Testes Automatizados (Pytest / RTL) | ★★★★☆ | Sei fazer com ajuda |
| Metodologia Ágil (Scrum) | ★★★★☆ | Sei fazer com ajuda |

---

## Soft Skills

- **Responsabilidade técnica**
  Assumir a integração de autenticação em ambas as camadas do sistema — front-end e back-end — exigiu atenção ao impacto de cada decisão no comportamento de toda a aplicação, especialmente no momento de ajustar a semântica de acesso negado para evitar que o front-end precisasse tratar erros HTTP de forma diferente do esperado.

- **Visão sistêmica**
  Trabalhar em rotas de back-end, migração de banco de dados, documentação de API e integração de autenticação no mesmo semestre exigiu entender como cada camada afeta as demais, como na decisão de retornar 200 com `acesso_permitido: false` em vez de 403, que simplificou o tratamento no front-end.

- **Comunicação**
  A documentação das rotas com Swagger e a criação de um manual de boas práticas para `@extend_schema` foram iniciativas que reduziram a dependência dos colegas de front-end de consultar o código-fonte para entender os contratos da API.

---

## Aprendizados Efetivos

O IAzimute foi o primeiro projeto em que trabalhei com autenticação via OpenID Connect de ponta a ponta, o que exigiu entender não apenas como integrar uma biblioteca de Keycloak, mas como o fluxo Authorization Code + PKCE funciona, como os tokens JWT são validados por JWKS e como configurar corretamente as URLs em um ambiente Docker onde o endereço interno do contêiner é diferente do endereço público do emissor.

A migração de MySQL para PostgreSQL no meio do projeto reforçou a importância de isolar a configuração de infraestrutura do código da aplicação. A experiência com `drf-spectacular` também consolidou o hábito de documentar as rotas no momento em que são escritas, e não depois, já que a documentação retroativa tende a ficar incompleta ou desatualizada.

---

## Navegação

| Semestre | Projeto | Empresa Parceira |
|----------|---------|-----------------|
| [1º Semestre – 01/2024](API01.md) | Calculadora Científica | FATEC São José dos Campos |
| [2º Semestre – 02/2024](API02.md) | Avaliador de Soft Skills (PACER) | FATEC São José dos Campos |
| [3º Semestre – 01/2025](API03.md) | Checkpoint | Altave |
| [4º Semestre – 02/2025](API04.md) | Sistema de Mobilidade Urbana | Prefeitura de São José dos Campos |
| [5º Semestre – 01/2026](API05.md) | Synthesi | SIATT |
| **6º Semestre – 02/2026** | **IAzimute** | **AKAER** |

[← Voltar ao Portfólio](README.md)
