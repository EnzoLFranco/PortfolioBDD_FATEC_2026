
# API 2º Semestre – 02/2024  
## Projeto: Avaliador de Soft Skills (PACER) 🏆  
**Empresa Parceira:** [FATEC São José dos Campos Profº Jessen Vidal](https://fatecsjc-prd.azurewebsites.net)

**[Repositório Github](https://github.com/SQLutions-FATEC/API-2-Semestre)** 

---

## 📝 Resumo do Projeto

O sistema foi desenvolvido para automatizar a avaliação **PACER** (Proatividade, Autonomia, Colaboração e Entrega de Resultados) realizada entre alunos da FATEC.

A solução substitui formulários manuais por uma plataforma digital onde estudantes avaliam seus pares, enquanto professores acompanham médias e geram relatórios padronizados.

O sistema otimiza o tempo docente ao automatizar cálculos e centralizar a gestão de equipes e critérios de avaliação.

---

## 💻 Tecnologias Utilizadas

- **Java & JavaFX**  
  Utilizados para desenvolvimento da aplicação desktop, incluindo interface gráfica e lógica de negócio.

- **MySQL**  
  Banco de dados relacional responsável pela persistência de usuários, equipes, sprints e notas.

- **JDBC**  
  Tecnologia utilizada para integração entre a aplicação Java e o banco de dados.

- **Figma**  
  Utilizado para prototipação de telas e definição da experiência do usuário (UX/UI).

---

## 🎯 Contribuições Individuais

Atuei como desenvolvedor com foco em persistência de dados, integração de arquivos e lógica administrativa.

- **Integração de Dados e Gestão de Equipes**  
  Implementação do módulo de importação de arquivos `.csv`, permitindo cadastro em massa de turmas e grupos.  
  Desenvolvimento da leitura e exibição dos dados em `TableView`, com validação antes da persistência no banco.

- **Persistência de Dados (DAO) e Modelagem**  
  Estruturação do `SprintDAO`, com métodos de criação, consulta e exclusão lógica (uso de `deleted_at`), além de tratamento de conflitos de datas.  
  Desenvolvimento do `AverageGradeDAO` e `NotaModel`, responsáveis pelo cálculo e recuperação de médias em tempo real.  
  Refatoração de consultas SQL para otimização de desempenho nas buscas.

- **Interface e Fluxo de Sistema**  
  Implementação de funcionalidades críticas, como exclusão de sprints e confirmação de notas, com tratamento de exceções para garantir integridade dos dados.  
  Reestruturação da interface do estudante, com foco na usabilidade durante o processo de avaliação.

---

## 📊 Hard Skills (Autoavaliação)

| Tecnologia / Metodologia | Nível | Classificação |
|--------------------------|------|---------------|
| Java (Backend) | ★★★★☆ | Avançado / Atuante |
| JavaFX (Interface) | ★★★☆☆ | Intermediário |
| MySQL / JDBC | ★★★★☆ | Avançado / Atuante |
| Persistência de Dados (DAO) | ★★★★☆ | Avançado |
| Git / GitHub | ★★★★☆ | Avançado |

---

## 🤝 Soft Skills

- **Comunicação**  
  Alinhamento constante de prioridades técnicas, garantindo integração eficiente entre banco de dados e interface.

- **Trabalho em equipe**  
  Colaboração na resolução de conflitos de merge e suporte na integração entre módulos de professor e aluno.

- **Proatividade**  
  Iniciativa na refatoração de métodos e correção de bugs estruturais antes das entregas.

- **Organização**  
  Gestão eficiente de commits e documentação técnica para consulta da equipe.
