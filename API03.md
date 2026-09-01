# API 3º Semestre – 01/2025
## Projeto: Checkpoint 
**Empresa Parceira:** [Altave](https://altave.com.br)

**[Repositório GitHub](https://github.com/SQLutions-FATEC/API-3-Semestre)**

---

## Resumo do Projeto

O Checkpoint foi desenvolvido para resolver problemas de gestão ineficiente de pontos de colaboradores terceirizados em estações de manutenção naval da empresa parceira Altave. Antes do sistema, o controle de jornada era feito de forma manual, sem rastreabilidade dos vínculos contratuais dos funcionários nem histórico organizado por empresa ou período.

A solução entregue oferece um painel administrativo completo que vincula funcionários a contratos específicos com prazos definidos, registra entradas e saídas e permite a exportação de espelhos de ponto em formato `.csv` com suporte a paginação personalizada — garantindo conformidade operacional e auditabilidade dos registros.

Este foi o primeiro projeto desenvolvido com uma **arquitetura separada entre front-end e back-end**, com Vue.js consumindo uma API REST construída em Spring Boot, e com uso de Docker para padronização dos ambientes.

---

## Tecnologias Utilizadas

- **Java & Spring Boot**
  Framework utilizado para a construção da API REST, responsável pelas regras de negócio, validações e persistência dos dados.

- **PostgreSQL**
  Banco de dados relacional utilizado para garantir a integridade referencial entre empresas, contratos e registros de ponto.

- **Vue.js & Ant Design**
  Stack de front-end utilizada para desenvolver a interface administrativa com componentes visuais padronizados e alto desempenho.

- **Docker**
  Utilizado para conteinerização da aplicação, garantindo consistência entre os ambientes de desenvolvimento e produção.

---

## Contribuições Individuais

Atuei como Desenvolvedor Full-Stack, contribuindo tanto no front-end quanto no back-end do sistema. Minhas principais entregas foram a interface de gestão de funcionários em Vue.js e a refatoração completa do módulo de contratos no back-end Spring Boot.

---

### Front-end: Interface de Gestão de Funcionários (Vue.js)

**Situação:** A tela de gestão de funcionários precisava exibir uma lista paginada de colaboradores, permitir filtragem por nome, e oferecer ações de edição e exclusão diretamente na tabela — tudo integrado com a API REST do back-end. A tela passou por uma refatoração durante o projeto: originalmente eram duas telas separadas (listagem e edição), que foram unificadas em uma única interface com modal.

**Tarefa:** Desenvolver a tela de edição de funcionários de forma independente e, posteriormente, participar da refatoração que unificou as duas telas, garantindo que a nova estrutura mantivesse todas as funcionalidades anteriores com melhor experiência de uso.

**Ação:** Implementei o componente `EmployeeHome` em Vue.js utilizando a Composition API (`setup`, `ref`, `onMounted`, `provide`). A listagem foi construída com o componente `a-table` do Ant Design, com suporte a paginação server-side via `handleTableChange`, que sincroniza a página e o tamanho com os parâmetros enviados à API. As ações de edição e exclusão foram implementadas como botões renderizados dinamicamente via `customRender`, abrindo modais específicos para cada operação. A integração com a API foi centralizada no serviço `employee.js`, que encapsula todas as chamadas HTTP via `axios`.

<details>
<summary>Código em JavaScript – Serviço de Funcionários (employee.js)</summary>

```javascript
import api from './api';

const employee = {
  get: (employeeId) => api.get(`/employee/${employeeId}`),
  getAll: (params = { page: 1, size: 10 }) => api.get('/employee', { params }),
  create: (payload) => api.post('/employee', payload),
  edit: (payload) => api.put(`/employee/${payload.id}`, payload),
  delete: (employeeId) => api.delete(`/employee/${employeeId}`),
};

export default employee;
```

</details>

<details>
<summary>Código em Vue.js – Componente EmployeeHome (script)</summary>

```javascript
import { ref, onMounted, provide, h } from 'vue';
import { Button, Modal, Table } from 'ant-design-vue';
import { EditOutlined, DeleteOutlined } from '@ant-design/icons-vue';
import employee from '@/services/employee';
import { formatDate, genderMask, registerNumberMask } from '@/utils';
import EmployeeHeader from '@/components/Headers/EmployeeHeader.vue';
import EmployeeModal from '@/components/Modals/EmployeeModal.vue';

export default {
  name: 'EmployeeHome',
  components: { 'a-button': Button, 'a-modal': Modal, 'a-table': Table,
    'employee-header': EmployeeHeader, 'employee-modal': EmployeeModal },

  setup() {
    const currentPage = ref(1);
    const dataSource = ref([]);
    const isConfirmationModalOpened = ref(false);
    const isEmployeeModalOpened = ref(false);
    const loading = ref(false);
    const pageSize = ref(10);
    const totalInfos = ref(0);
    const selectedEmployee = ref({});

    const fetchEmployees = async (filter = null) => {
      loading.value = true;
      const params = { page: currentPage.value, size: pageSize.value };
      if (filter) params.name = filter;

      try {
        const { data } = await employee.getAll(params);
        dataSource.value = data.items.map((employee) => ({
          key: employee.id,
          birthDate: formatDate(employee.birth_date),
          registerNumber: employee.register_number,
          name: employee.name,
          gender: employee.gender,
          bloodType: employee.blood_type,
        }));
        totalInfos.value = data.total;
      } catch (error) {
        console.error('Erro ao buscar funcionários:', error);
      } finally {
        loading.value = false;
      }
    };

    provide('apiCall', fetchEmployees);

    const handleTableChange = async (pagination) => {
      currentPage.value = pagination.current;
      pageSize.value = pagination.pageSize;
      await fetchEmployees();
    };

    const deleteEmployee = async () => {
      try {
        await employee.delete(selectedEmployee.value.key);
        isConfirmationModalOpened.value = false;
        await fetchEmployees();
      } catch (error) {
        console.error(error);
      }
    };

    onMounted(fetchEmployees);

    const columns = [
      { title: 'Número de registro', dataIndex: 'registerNumber', key: 'registerNumber',
        customRender: ({ text }) => registerNumberMask(text) },
      { title: 'Nome do funcionário', dataIndex: 'name', key: 'name' },
      { title: 'Gênero', dataIndex: 'gender', key: 'gender',
        customRender: ({ text }) => genderMask(text) },
      { title: 'Tipo sanguíneo', dataIndex: 'bloodType', key: 'bloodType' },
      { title: 'Data de nascimento', dataIndex: 'birthDate', key: 'birthDate' },
      {
        title: 'Ações', key: 'actions',
        customRender: ({ record }) => [
          h(Button, { type: 'primary', shape: 'circle', icon: h(EditOutlined),
            style: { marginRight: '8px' }, onClick: () => openEmployeeModal(record) }, null),
          h(Button, { class: 'delete-button', type: 'danger', shape: 'circle',
            icon: h(DeleteOutlined), onClick: () => openDeleteModal(record) }, null),
        ],
      },
    ];

    return { columns, currentPage, dataSource, deleteEmployee, fetchEmployees,
      handleTableChange, isConfirmationModalOpened, isEmployeeModalOpened,
      loading, pageSize, selectedEmployee, totalInfos,
      openEmployeeModal: (emp) => { selectedEmployee.value = emp; isEmployeeModalOpened.value = true; },
      openDeleteModal: (emp) => { selectedEmployee.value = emp; isConfirmationModalOpened.value = true; },
    };
  },
};
```

</details>

**Resultado:** A interface entregou paginação server-side funcional, filtragem por nome em tempo real e ações de edição e exclusão com confirmação por modal — tudo integrado com a API REST. A unificação das telas de listagem e edição em um único componente com modal reduziu a navegação necessária para realizar operações sobre os funcionários.

---

### Front-end: Refatoração do Módulo de Contratos (Vue.js)

**Situação:** O componente de contratos precisava lidar com um cenário complexo: um funcionário pode ter múltiplos contratos ao longo do tempo, mas apenas um pode estar ativo em determinado momento. A interface precisava validar conflitos de datas no lado do cliente antes de enviar os dados ao back-end, e diferenciar visualmente contratos ativos de inativos.

**Tarefa:** Participar da refatoração do componente `Contracts`, implementando a lógica de detecção de conflito de datas, a separação entre contrato ativo e inativos, e o fluxo de criação e edição em lote via API.

**Ação:** Utilizei a biblioteca `dayjs` com o plugin `isBetween` para verificar sobreposição de intervalos antes de adicionar um contrato à lista local. A função `addContract` avalia todas as combinações possíveis de sobreposição entre o novo contrato e os existentes, exibindo uma mensagem de erro via `message.error` do Ant Design sem necessidade de uma chamada ao servidor. O envio dos contratos ao back-end foi estruturado em dois métodos distintos — `createContracts` e `editContracts` — cada um formatando o payload de acordo com o que a API esperava, incluindo o campo `action` para operações em lote na edição.

<details>
<summary>Código em Vue.js – Componente Contracts com validação de conflito de datas</summary>

```javascript
import dayjs from 'dayjs';
import isBetween from 'dayjs/plugin/isBetween';
dayjs.extend(isBetween);

// Validação de conflito de datas no cliente
const addContract = (contract) => {
  const start = dayjs(contract.date_start);
  const end = dayjs(contract.date_end);
  const today = dayjs();

  const hasConflict = (existingContract) => {
    const existingStart = dayjs(existingContract.date_start);
    const existingEnd = dayjs(existingContract.date_end);
    return (
      start.isBetween(existingStart, existingEnd, null, '[]') ||
      end.isBetween(existingStart, existingEnd, null, '[]') ||
      existingStart.isBetween(start, end, null, '[]') ||
      existingEnd.isBetween(start, end, null, '[]')
    );
  };

  const allContracts = [
    ...inactiveContracts.value,
    ...(Object.keys(activeContract.value).length ? [activeContract.value] : []),
  ];

  if (allContracts.some(hasConflict)) {
    message.error('Já existe um contrato com datas conflitantes.');
    return;
  }

  if (today.isBetween(start, end, 'day', '[]')) {
    if (Object.keys(activeContract.value).length) {
      activeContract.value.active = false;
      inactiveContracts.value = [...inactiveContracts.value, activeContract.value];
    }
    contract.active = true;
    activeContract.value = contract;
  } else {
    contract.active = false;
    inactiveContracts.value = [...inactiveContracts.value, contract];
  }
};

// Envio em lote para criação
const createContracts = async (employeeId) => {
  if (!Object.keys(activeContract.value).length) return;

  const params = {
    contracts: [
      { company_id: activeContract.value.company.id, role_id: activeContract.value.role.id,
        date_start: activeContract.value.date_start, date_end: activeContract.value.date_end },
      ...inactiveContracts.value.map((c) => ({
        company_id: c.company.value, role_id: c.role.value,
        date_start: c.date_start, date_end: c.date_end,
      })),
    ],
    employee_id: employeeId,
  };

  try {
    await contracts.create(params);
  } catch (error) {
    console.error(error);
  }
};
```

</details>

**Resultado:** A validação de conflito de datas no cliente eliminou chamadas desnecessárias ao servidor para cenários de erro previsíveis, melhorando a responsividade da interface. A separação entre contratos ativos e inativos ficou clara tanto na lógica do componente quanto na interface exibida ao usuário.

---

### Back-end: Refatoração do Módulo de Contratos (Spring Boot)

**Situação:** O módulo de contratos do back-end apresentava problemas de persistência no PostgreSQL: as entidades e seus relacionamentos JPA não estavam corretamente mapeados, o que causava falhas silenciosas na criação e atualização de contratos. Além disso, não havia validações de negócio — era possível cadastrar múltiplos contratos ativos para o mesmo funcionário simultaneamente.

**Tarefa:** Refatorar o `ContractServiceImpl`, corrigindo os problemas de persistência, reestruturando as entidades e relacionamentos JPA, e implementando as regras de negócio que garantissem a integridade dos dados de contratos.

**Ação:** Reestruturei o serviço de contratos implementando as seguintes regras e melhorias:

- **Criação simples** (`createContract`): valida datas, verifica se o funcionário já possui contrato ativo via `findContractByEmployeeAndDate`, e lança `BusinessException` com mensagem descritiva em caso de violação.
- **Criação em lote** (`createContracts`): percorre os contratos do payload, valida cada um individualmente e inativa automaticamente contratos ativos anteriores antes de persistir os novos, garantindo que apenas um contrato esteja ativo por funcionário.
- **Atualização em lote** (`processBulkUpdate`): processa operações de `create`, `update`, `delete` e `keep` por contrato, aplicando cada ação de forma transacional e validando ao final que não restou mais de um contrato ativo.
- **Inativação** (`inactivateContract`): verifica se o contrato está realmente ativo e se já iniciou antes de permitir a inativação, prevenindo estados inconsistentes.

<details>
<summary>Código em Java – ContractService (interface)</summary>

```java
public interface ContractService {
    ContractResponseDTO createContract(ContractRequestDTO contractRequestDTO);
    List<ContractResponseDTO> createContracts(CreateContractsRequestDTO request);
    void processBulkUpdate(BulkContractUpdateRequestDTO request);
    void deleteContract(Long id);
    List<ContractResponseDTO> getContractsByEmployee(Long employeeId);
    List<ContractResponseDTO> getContractsByCompany(Long companyId);
    ContractResponseDTO inactivateContract(Long contractId);
}
```

</details>

<details>
<summary>Código em Java – ContractServiceImpl (implementação completa)</summary>

```java
@Service
public class ContractServiceImpl implements ContractService {

    private final ContractRepository contractRepository;
    private final EmployeeRepository employeeRepository;
    private final CompanyRepository companyRepository;
    private final RoleRepository roleRepository;

    public ContractServiceImpl(ContractRepository contractRepository,
                               EmployeeRepository employeeRepository,
                               CompanyRepository companyRepository,
                               RoleRepository roleRepository) {
        this.contractRepository = contractRepository;
        this.employeeRepository = employeeRepository;
        this.companyRepository = companyRepository;
        this.roleRepository = roleRepository;
    }

    @Override
    public ContractResponseDTO createContract(ContractRequestDTO dto) {
        validateContractDates(dto.getStartDate(), dto.getEndDate());

        var employee = employeeRepository.findById(dto.getEmployeeId()).orElseThrow();
        var company = companyRepository.findById(dto.getCompanyId()).orElseThrow();
        var role = roleRepository.findById(dto.getRoleId()).orElseThrow();

        if (contractRepository.findContractByEmployeeAndDate(employee, LocalDate.now()).isPresent()) {
            throw new BusinessException("O funcionário já possui um contrato ativo.");
        }

        Contract contract = new Contract();
        contract.setEmployee(employee);
        contract.setCompany(company);
        contract.setRole(role);
        contract.setStartDate(dto.getStartDate());
        contract.setEndDate(dto.getEndDate());

        return new ContractResponseDTO(contractRepository.save(contract));
    }

    @Override
    @Transactional
    public void processBulkUpdate(BulkContractUpdateRequestDTO request) {
        var employee = employeeRepository.findById(request.getEmployeeId())
                .orElseThrow(() -> new BusinessException("Funcionário não encontrado"));

        for (BulkContractUpdateRequestDTO.ContractOperationDTO dto : request.getContracts()) {
            switch (dto.getAction().toLowerCase()) {
                case "create" -> createNewContract(employee, dto);
                case "update" -> updateExistingContract(dto);
                case "delete" -> deleteContract(dto.getId());
                case "keep" -> { /* sem alteração */ }
                default -> throw new BusinessException("Ação inválida: " + dto.getAction());
            }
        }

        validateSingleActiveContract(employee.getId());
    }

    @Override
    public ContractResponseDTO inactivateContract(Long contractId) {
        Contract contract = contractRepository.findById(contractId)
                .orElseThrow(() -> new BusinessException("Contrato não encontrado."));

        LocalDate today = LocalDate.now();

        if (!contract.isActive(today))
            throw new BusinessException("Este contrato já se encontra inativo.");

        if (contract.getStartDate().isAfter(today))
            throw new BusinessException("Não é possível inativar um contrato que ainda não começou.");

        contract.setEndDate(today.minusDays(1));
        return new ContractResponseDTO(contractRepository.save(contract));
    }

    private void validateSingleActiveContract(Long employeeId) {
        List<Contract> activeContracts = contractRepository
                .findActiveContractsByEmployee(employeeId, LocalDate.now());

        if (activeContracts.size() > 1) {
            Contract mostRecent = activeContracts.stream()
                    .max(Comparator.comparing(Contract::getStartDate))
                    .orElseThrow();

            activeContracts.forEach(contract -> {
                if (!contract.equals(mostRecent)) {
                    contract.setEndDate(mostRecent.getStartDate().minusDays(1));
                    contractRepository.save(contract);
                }
            });
        }
    }

    private void validateContractDates(LocalDate startDate, LocalDate endDate) {
        if (endDate != null && startDate.isAfter(endDate))
            throw new BusinessException("A data de início não pode ser posterior à data de término.");
    }
}
```

</details>

**Resultado:** O módulo de contratos passou a rejeitar estados inválidos em todas as operações — criação simples, criação em lote e atualizações. A regra de unicidade de contrato ativo por funcionário foi garantida tanto no back-end quanto antecipada no front-end, eliminando inconsistências que antes precisavam ser corrigidas diretamente no banco de dados.

---

## Hard Skills (Autoavaliação)

| Tecnologia / Metodologia | Nível | Classificação |
|--------------------------|-------|---------------|
| Java com Spring Boot | ★★★★☆ | Sei fazer com ajuda |
| Vue.js (JavaScript) | ★★★☆☆ | Entendi |
| PostgreSQL / SQL | ★★★★☆ | Sei fazer com ajuda |
| Docker | ★★☆☆☆ | Já ouvi falar |
| Metodologia Ágil (Scrum) | ★★★★★ | Sei fazer com autonomia |

---

## Soft Skills

- **Comunicação**
  Trabalhar em uma arquitetura separada entre front-end e back-end exigiu alinhamento constante sobre os contratos de API — quais campos o back-end esperava, quais retornava, e em qual formato. Precisei comunicar de forma clara as mudanças que fiz na estrutura do `ContractServiceImpl` para que o front-end pudesse adaptar os payloads enviados, e vice-versa, quando alterações na interface exigiam ajustes no back-end.

- **Trabalho em equipe**
  Acompanhei o progresso dos cards da equipe no board do projeto, oferecendo suporte em integrações entre módulos quando possível e buscando ajuda de colegas mais experientes em situações onde meu conhecimento ainda era limitado — especialmente no início do uso do Spring Boot. Essa postura de aprendizado ativo com o time acelerou minha curva de desenvolvimento ao longo das sprints.

- **Proatividade**
  Ao identificar que o módulo de contratos apresentava falhas silenciosas de persistência, tomei a iniciativa de mapear o problema, propor a refatoração à equipe e executá-la. Além disso, implementei a validação de conflito de datas no front-end sem que isso estivesse previsto como requisito explícito, antecipando uma classe de erros que chegaria ao servidor e degradaria a experiência do usuário.

---

## Aprendizados Efetivos

Este projeto representou um salto significativo em complexidade em relação aos semestres anteriores. Foi a primeira vez que trabalhei com uma arquitetura real de separação entre front-end e back-end, consumindo e produzindo APIs REST — o que exigiu compreender não apenas as tecnologias individualmente, mas como elas se comunicam e quais contratos precisam ser respeitados entre as camadas.

O padrão DAO que aprendi no semestre anterior com JDBC se traduziu diretamente para o Spring Boot via JPA e Repositories, o que tornou a curva de aprendizado mais suave. Já Vue.js foi um desafio novo — aprender a Composition API, o sistema de reatividade com `ref` e `provide`, e a integração com uma biblioteca de componentes como o Ant Design em um projeto real me deu uma base prática que dificilmente seria obtida apenas em exercícios acadêmicos.
