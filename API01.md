# API 1º Semestre – 01/2024
## Projeto: Calculadora Científica 
**Empresa Parceira:** [FATEC São José dos Campos – Prof. Jessen Vidal](https://fatecsjc-prd.azurewebsites.net)

**[Repositório GitHub](https://github.com/SQLutions-FATEC/API-1-Semestre)**

---

## Resumo do Projeto

O projeto consistiu no desenvolvimento de uma **Calculadora Científica via terminal**, projetada para atender necessidades contábeis e educacionais. Era o primeiro contato da equipe com desenvolvimento de software colaborativo utilizando metodologia ágil, o que tornou o desafio ainda mais significativo: além de aprender a programar, foi necessário aprender a trabalhar em equipe com controle de versão e organização por backlog.

A calculadora foi construída para lidar com:
- Operações aritméticas simples
- Cálculos financeiros (juros)
- Equações de segundo grau
- Conversões de bases numéricas (Binário, Octal, Decimal e Hexadecimal)

---

## Tecnologias Utilizadas

- **TypeScript**
  Linguagem principal utilizada para implementar lógica orientada a tipos, garantindo maior segurança e robustez nos cálculos.

- **Node.js**
  Ambiente de execução utilizado para rodar a calculadora diretamente no terminal.

- **VisualG (Portugol)**
  Utilizado na fase inicial de prototipagem para estruturar a lógica e os algoritmos fundamentais antes da implementação em TypeScript.

- **Git/GitHub**
  Utilizados para controle de versão e organização de tarefas por meio do backlog do repositório.

---

## Contribuições Individuais

Atuei no desenvolvimento do núcleo lógico da calculadora, com foco na precisão dos cálculos, no tratamento de entradas inválidas e na modularização do sistema.

### Operações Aritméticas — Multiplicação e Divisão

**Situação:** A calculadora precisava lidar com entradas do usuário via terminal, onde não havia garantia de que os valores digitados seriam números válidos. Operações como divisão eram especialmente sensíveis, pois entradas não numéricas ou zero poderiam gerar comportamentos inesperados no sistema.

**Tarefa:** Implementar as funções de multiplicação e divisão de forma robusta, garantindo que erros de entrada fossem capturados e comunicados de forma clara ao usuário, sem interromper a execução do programa.

**Ação:** Desenvolvi as funções utilizando TypeScript com tipagem explícita (`number`) para os operandos e resultados. Para validação, utilizei a função nativa `isNaN()` para detectar entradas inválidas antes de exibir o resultado, exibindo mensagens de erro descritivas no próprio terminal quando necessário. Cada função foi isolada em seu próprio módulo TypeScript, importando apenas o menu de exibição compartilhado.

<details>
<summary>Código em TypeScript – Função de Multiplicação</summary>

```typescript
import { menu } from "../menu";

const inputNum = require("prompt-sync")();

export function multiplicacao(): void {
  menu("Multiplicação");
  console.log("| Digite o primeiro número:            |");
  const numero1: number = Number(inputNum("| "));
  console.log("| Digite o segundo número:             |");
  const numero2: number = Number(inputNum("| "));
  console.log("|--------------------------------------|");

  const resultado: number = numero1 * numero2;
  if (isNaN(resultado)) {
    console.log("| Resultado inválido                   |");
    console.log("| Razão: valores não numéricos         |");
  } else {
    console.log("| Resultado:                           |");
    console.log(`| ${numero1} * ${numero2} = ${resultado}`);
  }
  console.log("|______________________________________|");
}
```

</details>

<details>
<summary>Código em TypeScript – Função de Divisão</summary>

```typescript
import { menu } from "../menu";

const inputNum = require("prompt-sync")();

export function divisao(): void {
  menu("Divisão");
  console.log("| Digite o primeiro número:            |");
  const numero1: number = Number(inputNum("| "));
  console.log("| Digite o segundo número:             |");
  const numero2: number = Number(inputNum("| "));
  console.log("|--------------------------------------|");

  const resultado: number = numero1 / numero2;
  if (isNaN(resultado)) {
    console.log("| Resultado inválido                   |");
    console.log("| Razão: valores não numéricos         |");
  } else {
    console.log("| Resultado:                           |");
    console.log(`| ${numero1} / ${numero2} = ${resultado}`);
  }
  console.log("|______________________________________|");
}
```

</details>

**Resultado:** As funções passaram a tratar de forma segura qualquer entrada do usuário, exibindo mensagens de erro claras sem derrubar a aplicação. A estrutura modular adotada facilitou a manutenção e a integração dessas funções ao menu principal da calculadora.

---

### Função de Fatorial

**Situação:** O fatorial é uma operação matematicamente definida apenas para inteiros não-negativos. Em uma calculadora via terminal, o usuário poderia digitar valores decimais, negativos ou caracteres não numéricos, o que exigiria tratamento cuidadoso antes do cálculo.

**Tarefa:** Implementar o algoritmo de fatorial com validação completa de entrada, garantindo que apenas valores inteiros e positivos fossem aceitos, e que a função pudesse ser reutilizada de forma independente no menu principal.

**Ação:** Utilizei `parseInt()` na leitura da entrada, em vez de `Number()`, para garantir que apenas inteiros fossem considerados. Adicionei uma verificação condicional que rejeita valores negativos ou `NaN` antes de iniciar o cálculo. O algoritmo em si foi implementado com estrutura de repetição `for`, acumulando o produto iterativamente para evitar a complexidade de uma abordagem recursiva em um contexto de primeiro semestre.

<details>
<summary>Código em TypeScript – Função de Fatorial</summary>

```typescript
import { menu } from "../menu";

const input = require("prompt-sync")();

export function fatorial(): void {
  menu("Fatorial");
  console.log("| x! =                                 |");
  console.log("| Digite o número                      |");

  const numero: number = parseInt(input("| "));

  if (isNaN(numero) || numero < 0) {
    console.log("| O número deve ser inteiro e positivo.|");
    console.log("|______________________________________|");
  } else {
    let fatorial: number = 1;
    for (let i = 1; i <= numero; i++) {
      fatorial *= i;
    }
    console.log(`| O fatorial de ${numero} é ${fatorial}.`);
    console.log("|______________________________________|");
  }
}
```

</details>

**Resultado:** A função validou corretamente todos os casos de entrada inválida e calculou com precisão o fatorial de valores inteiros positivos. Sua modularização permitiu que fosse integrada ao menu principal sem dependências desnecessárias.

---

### Organização e Modularização do Código

**Situação:** Com múltiplos integrantes contribuindo simultaneamente, o projeto corria o risco de ter código misturado — lógica de cálculo, exibição de menus e leitura de entradas no mesmo arquivo, dificultando a manutenção e a colaboração.

**Tarefa:** Separar as responsabilidades do sistema de forma que cada função ficasse isolada em seu próprio módulo, com a interface de navegação desacoplada das regras de negócio.

**Ação:** Estruturei os arquivos seguindo o princípio de responsabilidade única: cada operação matemática em seu próprio arquivo TypeScript, importando apenas os recursos necessários (como o módulo `menu`). Esse padrão foi adotado como referência pela equipe para os demais módulos.

**Resultado:** A separação de responsabilidades reduziu conflitos de merge no GitHub e tornou o código mais legível para todos os integrantes, facilitando a integração das entregas nas sprints seguintes.

---

## 📊 Hard Skills (Autoavaliação)

| Tecnologia / Metodologia | Nível | Classificação |
|--------------------------|-------|---------------|
| Lógica de Programação | ★★★★☆ | Avançado / Atuante |
| TypeScript | ★★★☆☆ | Intermediário |
| Algoritmos (VisualG) | ★★★★☆ | Avançado |
| Matemática Computacional | ★★★★☆ | Avançado |
| Git/GitHub | ★★★☆☆ | Intermediário |

---

## Soft Skills

- **Comunicação**
  Durante as reuniões de alinhamento da equipe, precisei explicar as decisões técnicas que tomei — como o uso de `parseInt` em vez de `Number` para o fatorial — para colegas com diferentes níveis de familiaridade com TypeScript. Essa necessidade me levou a desenvolver uma comunicação mais clara e didática, traduzindo escolhas técnicas em linguagem acessível.

- **Trabalho em equipe**
  Por ser o primeiro projeto com versionamento colaborativo via Git, conflitos de merge eram frequentes. Participei ativamente na resolução desses conflitos e na definição de um padrão de organização de arquivos que o time passou a adotar, contribuindo para que as entregas de cada sprint fossem integradas com menos retrabalho.

- **Proatividade**
  Identifiquei, antes das revisões de sprint, que algumas funções do sistema não estavam validando entradas inválidas, o que poderia causar falhas silenciosas. Tomei a iniciativa de implementar as validações com `isNaN()` e documentar o padrão adotado para que outros integrantes replicassem nos demais módulos.

- **Organização**
  Utilizei o backlog do GitHub para registrar e priorizar as tarefas sob minha responsabilidade, garantindo visibilidade do progresso para o restante da equipe e cumprimento dos prazos de cada sprint.

---

## Aprendizados Efetivos

Este projeto representou meu primeiro contato real com desenvolvimento de software colaborativo. Além de aprender TypeScript e a lógica de programação orientada a tipos, compreendi na prática a importância de tratar entradas do usuário de forma defensiva — uma lição que carrego para todos os projetos seguintes.

O uso do Git em equipe, com múltiplos colaboradores editando os mesmos arquivos, me ensinou a importância da modularização não apenas como boa prática técnica, mas como estratégia de colaboração. Aprendi também que comunicar decisões técnicas de forma clara é tão importante quanto tomá-las corretamente.
