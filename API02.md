# API 2º Semestre – 02/2024
## Projeto: Avaliador de Soft Skills (PACER) 
**Empresa Parceira:** [FATEC São José dos Campos – Prof. Jessen Vidal](https://fatecsjc-prd.azurewebsites.net)

**[Repositório GitHub](https://github.com/SQLutions-FATEC/API-2-Semestre)**

---

## Resumo do Projeto

O sistema foi desenvolvido para automatizar a avaliação **PACER** (Proatividade, Autonomia, Colaboração e Entrega de Resultados) realizada entre alunos da FATEC. Antes da solução, o processo era conduzido manualmente por meio de formulários físicos ou planilhas isoladas, o que dificultava o acompanhamento das médias, gerava retrabalho docente e não oferecia histórico organizado por sprint.

A plataforma digital desenvolvida substituiu esse processo: estudantes avaliam seus pares diretamente pelo sistema, enquanto professores acompanham médias consolidadas em tempo real e geram relatórios padronizados por período. O sistema também permitiu o cadastro em massa de turmas via importação de arquivos `.csv`, eliminando o lançamento manual de dados.

---

## Tecnologias Utilizadas

- **Java & JavaFX**
  Utilizados para desenvolvimento da aplicação desktop, incluindo interface gráfica e lógica de negócio.

- **MySQL**
  Banco de dados relacional responsável pela persistência de usuários, equipes, sprints e notas.

- **JDBC**
  Tecnologia utilizada para integração entre a aplicação Java e o banco de dados MySQL.

- **Figma**
  Utilizado para prototipação de telas e definição da experiência do usuário (UX/UI).

---

## Contribuições Individuais

Atuei como desenvolvedor com foco em persistência de dados, integração de arquivos e lógica administrativa do sistema.

---

### Conexão com o Banco de Dados

**Situação:** A aplicação precisava se conectar ao banco de dados MySQL de forma centralizada, evitando que cada classe abrisse sua própria conexão de forma independente — o que causaria inconsistências e dificultaria a manutenção.

**Tarefa:** Implementar uma classe utilitária de conexão reutilizável, que servisse como ponto único de acesso ao banco para todos os DAOs do sistema.

**Ação:** Desenvolvi a classe `ConexaoBanco`, que encapsula as configurações de conexão (URL, usuário e senha) e expõe um método estático `getConnection()`. O tratamento de erros foi implementado via bloco `try-catch` com `SQLException`, garantindo que falhas de conexão fossem registradas sem derrubar a aplicação.

<details>
<summary>Código em Java – Classe ConexaoBanco</summary>

```java
package app.controllers;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ConexaoBanco {
    private static final String URL = "jdbc:mysql://localhost:3306/avaliador";
    private static final String USER = "root";
    private static final String PASSWORD = "admin";

    public static Connection getConnection() {
        Connection conn = null;
        try {
            conn = DriverManager.getConnection(URL, USER, PASSWORD);
            System.out.println("Conexão bem-sucedida com o banco de dados!");
        } catch (SQLException e) {
            System.out.println("Erro ao conectar ao banco de dados: " + e.getMessage());
        }
        return conn;
    }
}
```

</details>

**Resultado:** Todos os DAOs do sistema passaram a utilizar `ConexaoBanco.getConnection()` como ponto único de acesso, reduzindo a duplicação de código e facilitando eventuais ajustes nas configurações de conexão.

---

### Importação de Alunos via CSV e Exibição em TableView

**Situação:** O cadastro manual de alunos, um a um, seria inviável para turmas com dezenas de estudantes. A equipe precisava de uma forma eficiente de carregar os dados de turmas e equipes no sistema antes do início de cada período de avaliação.

**Tarefa:** Implementar um módulo de importação de arquivos `.csv` que lesse os dados dos alunos, os exibisse em uma `TableView` para validação visual pelo professor, e os persistisse no banco de dados mediante confirmação.

**Ação:** Desenvolvi o controller `ControllerCSV`, a interface FXML correspondente e o model `AlunoModel` com propriedades JavaFX (`SimpleIntegerProperty`, `SimpleStringProperty`) para compatibilidade com a `TableView`. O fluxo foi dividido em duas etapas: primeiro a leitura e exibição dos dados na tabela, depois a persistência via `PreparedStatement` com tratamento individual de erros por aluno — garantindo que uma linha inválida no CSV não impedisse o cadastro das demais.

<details>
<summary>Interface FXML – Tela de Carregamento de Alunos</summary>

```xml
<AnchorPane prefHeight="400.0" prefWidth="600.0"
    xmlns="http://javafx.com/javafx/23"
    xmlns:fx="http://javafx.com/fxml/1"
    fx:controller="app.controllers.ControllerCSV">
   <children>
      <Button fx:id="buttonEnviarCSV" layoutX="503.0" layoutY="14.0"
              onAction="#enviarCSV" prefHeight="39.0" prefWidth="83.0" text="Inserir csv" />
      <TableView fx:id="tableView" layoutX="27.0" layoutY="90.0"
                 prefHeight="220.0" prefWidth="546.0">
         <columns>
            <TableColumn fx:id="colRa"     prefWidth="120.0" text="RA" />
            <TableColumn fx:id="colNome"   prefWidth="120.0" text="Nome" />
            <TableColumn fx:id="colEmail"  prefWidth="120.0" text="E-mail" />
            <TableColumn fx:id="colEquipe" prefWidth="120.0" text="Equipe" />
         </columns>
         <columnResizePolicy>
            <TableView fx:constant="CONSTRAINED_RESIZE_POLICY" />
         </columnResizePolicy>
      </TableView>
      <Button layoutX="503.0" layoutY="347.0"
              prefHeight="39.0" prefWidth="83.0" text="Confirmar" />
      <Label layoutX="170.0" text="Carregar alunos">
         <font><Font name="System Bold" size="35.0" /></font>
      </Label>
   </children>
</AnchorPane>
```

</details>

<details>
<summary>Código em Java – Model de Aluno com propriedades JavaFX</summary>

```java
package app.controllers.Model;

import javafx.beans.property.SimpleIntegerProperty;
import javafx.beans.property.SimpleStringProperty;

public class Aluno {

    private SimpleIntegerProperty ra;
    private SimpleStringProperty nome;
    private SimpleStringProperty email;
    private SimpleStringProperty senha;
    private SimpleStringProperty id_equipe;

    public Aluno(int ra, String nome, String email, String senha, String id_equipe) {
        this.ra = new SimpleIntegerProperty(ra);
        this.nome = new SimpleStringProperty(nome);
        this.email = new SimpleStringProperty(email);
        this.senha = new SimpleStringProperty(senha);
        this.id_equipe = new SimpleStringProperty(id_equipe);
    }

    public Integer getRa() { return ra.get(); }
    public String getNome() { return nome.get(); }
    public String getEmail() { return email.get(); }
    public String getSenha() { return senha.get(); }
    public String getId_equipe() { return id_equipe.get(); }
}
```

</details>

<details>
<summary>Código em Java – Persistência dos dados do CSV no banco</summary>

```java
public void confirmarCSV() {
    Connection connection = null;
    PreparedStatement statement = null;

    try {
        connection = ConexaoBanco.getConnection();
        if (connection == null) {
            System.out.println("Falha ao estabelecer a conexão com o banco de dados.");
            return;
        }

        String sql = "INSERT INTO usuario (ra, nome, senha, email, id_equipe) VALUES (?, ?, ?, ?, ?)";
        statement = connection.prepareStatement(sql);

        for (AlunoModel aluno : tableView.getItems()) {
            statement.setInt(1, aluno.getRa());
            statement.setString(2, aluno.getNome());
            statement.setString(3, aluno.getSenha());
            statement.setString(4, aluno.getEmail());
            statement.setString(5, aluno.getId_equipe());

            try {
                statement.executeUpdate();
            } catch (SQLException e) {
                System.out.println("Erro ao inserir aluno: " + aluno.getNome() + " - " + e.getMessage());
            }
        }

        System.out.println("Dados confirmados e cadastrados no banco de dados com sucesso!");
    } catch (SQLException e) {
        System.out.println("Erro ao preparar a declaração SQL: " + e.getMessage());
    } finally {
        try {
            if (statement != null) statement.close();
            if (connection != null) connection.close();
        } catch (SQLException e) {
            System.out.println("Erro ao fechar recursos: " + e.getMessage());
        }
    }
}
```

</details>

**Resultado:** O módulo de importação eliminou o cadastro manual de alunos, permitindo que turmas inteiras fossem carregadas em segundos. A etapa de pré-visualização na `TableView` deu ao professor controle sobre os dados antes de qualquer alteração no banco, reduzindo o risco de inserções incorretas.

---

### Persistência de Sprints (SprintDAO)

**Situação:** As sprints eram o elemento central do sistema PACER — sem um gerenciamento correto delas, as avaliações ficariam sem referência temporal. Era necessário garantir que sprints com datas sobrepostas ou nomes duplicados não pudessem ser cadastradas, e que exclusões não apagassem o histórico de avaliações associado.

**Tarefa:** Estruturar o `SprintDAO` com métodos completos de criação, consulta e exclusão lógica, incluindo validações de conflito de datas e duplicidade antes de qualquer operação de escrita no banco.

**Ação:** Implementei o `SprintDAO` com os seguintes métodos:

- `selectSprints()` e `selectLast8Sprints()` — consultas com filtro por `deleted_at IS NULL`, garantindo que sprints excluídas logicamente não apareçam na interface.
- `createSprint()` — antes de inserir, valida se a data de fim é posterior à de início, verifica sobreposição de datas com `isDateRangeAvailable()` e duplicidade de nome com `isDuplicateSprint()`, lançando `SQLException` com mensagem descritiva em cada caso.
- `deleteSprint()` — exclusão lógica via `UPDATE sprint SET deleted_at = NOW()`, preservando o histórico de avaliações vinculadas à sprint excluída.

<details>
<summary>Código em Java – SprintDAO completo</summary>

```java
package app.DAOs;

import app.helpers.DatabaseConnection;
import app.models.SprintModel;
import javafx.collections.FXCollections;
import javafx.collections.ObservableList;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.time.LocalDate;
import java.util.Date;

public class SprintDAO {

    public ObservableList<SprintModel> selectSprints(int selectedPeriodId) {
        ObservableList<SprintModel> sprintList = FXCollections.observableArrayList();
        String sql = "SELECT * FROM sprint s WHERE s.periodo = ? AND deleted_at IS NULL ORDER BY s.data_inicio";

        try (ResultSet resultSet = DatabaseConnection.executeQuery(sql, selectedPeriodId)) {
            while (resultSet.next()) {
                SprintModel sprint = new SprintModel(
                    resultSet.getInt("id"),
                    resultSet.getString("descricao"),
                    resultSet.getDate("data_inicio"),
                    resultSet.getDate("data_fim")
                );
                sprintList.add(sprint);
            }
        } catch (SQLException e) {
            System.out.println("Erro no SQL de selectSprints: " + e.getMessage());
        }
        return sprintList;
    }

    public boolean createSprint(String descricao, Date dataInicio, Date dataFim) throws SQLException {
        if (!dataFim.after(dataInicio)) {
            throw new SQLException("A data de término deve ser posterior à data de início.");
        }

        String getPeriodoSql = "SELECT id FROM periodo WHERE ano = ? AND semestre = ?";
        String insertSprintSql = "INSERT INTO sprint (descricao, data_inicio, data_fim, periodo) VALUES (?, ?, ?, ?)";

        int semestre = (dataInicio.getMonth() + 1 <= 6) ? 1 : 2;
        int ano = dataInicio.getYear() + 1900;
        int periodoId;

        try (ResultSet rs = DatabaseConnection.executeQuery(getPeriodoSql, ano, semestre)) {
            if (rs.next()) {
                periodoId = rs.getInt("id");
            } else {
                throw new SQLException("Período não encontrado para ano " + ano + " e semestre " + semestre);
            }
        }

        if (!isDateRangeAvailable(dataInicio, dataFim, periodoId)) {
            throw new SQLException("Datas inseridas já estão em uso.");
        }

        if (isDuplicateSprint(descricao, periodoId)) {
            throw new SQLException("Já existe uma sprint com esse nome para o mesmo período.");
        }

        int rowsAffected = DatabaseConnection.executeUpdate(insertSprintSql, descricao, dataInicio, dataFim, periodoId);
        return rowsAffected > 0;
    }

    private boolean isDateRangeAvailable(Date dataInicio, Date dataFim, int periodoId) {
        String sql = """
            SELECT COUNT(*) AS count
            FROM sprint
            WHERE periodo = ? AND deleted_at IS NULL
            AND (
                (data_inicio <= ? AND data_fim >= ?) OR
                (data_inicio <= ? AND data_fim >= ?) OR
                (data_inicio >= ? AND data_fim <= ?)
            )
        """;

        try (ResultSet resultSet = DatabaseConnection.executeQuery(
                sql, periodoId, dataFim, dataFim, dataInicio, dataInicio, dataInicio, dataFim)) {
            if (resultSet.next()) {
                return resultSet.getInt("count") == 0;
            }
        } catch (SQLException e) {
            System.out.println("Erro ao verificar disponibilidade do intervalo de datas: " + e.getMessage());
        }
        return false;
    }

    private boolean isDuplicateSprint(String descricao, int periodoId) {
        String sql = "SELECT COUNT(*) AS count FROM sprint WHERE descricao = ? AND periodo = ? AND deleted_at IS NULL";

        try (ResultSet resultSet = DatabaseConnection.executeQuery(sql, descricao, periodoId)) {
            if (resultSet.next()) {
                return resultSet.getInt("count") > 0;
            }
        } catch (SQLException e) {
            System.out.println("Erro ao verificar duplicidade da sprint: " + e.getMessage());
        }
        return false;
    }

    public int deleteSprint(int sprintId) throws SQLException {
        String sql = "UPDATE sprint SET deleted_at = NOW() WHERE id = ?";
        int generatedKey = 0;
        try {
            generatedKey = DatabaseConnection.executeUpdate(sql, sprintId);
        } catch (SQLException e) {
            System.out.println("Erro no SQL de deleteSprint: " + e.getMessage());
        } finally {
            DatabaseConnection.closeResources();
        }
        return generatedKey;
    }
}
```

</details>

**Resultado:** O sistema passou a rejeitar automaticamente cadastros de sprints com conflitos de data ou nome duplicado, eliminando inconsistências que anteriormente precisavam ser corrigidas manualmente no banco. A exclusão lógica preservou o histórico completo de avaliações, garantindo rastreabilidade dos dados ao longo dos semestres.

---

## Hard Skills (Autoavaliação)

| Tecnologia / Metodologia | Nível | Classificação |
|--------------------------|-------|---------------|
| Java (Backend) | ★★★★☆ | Avançado / Atuante |
| JavaFX (Interface) | ★★★☆☆ | Intermediário |
| MySQL / JDBC | ★★★★☆ | Avançado / Atuante |
| Persistência de Dados (DAO) | ★★★★☆ | Avançado |
| Git / GitHub | ★★★★☆ | Avançado |

---

## Soft Skills

- **Comunicação**
  A integração entre banco de dados e interface exigiu alinhamento constante com os colegas responsáveis pelo front-end em JavaFX. Precisei comunicar as estruturas dos DAOs e os contratos de dados esperados para que a equipe pudesse consumir os métodos corretamente, o que me levou a documentar as assinaturas dos métodos e os tipos de retorno de forma clara antes da integração.

- **Trabalho em equipe**
  Durante o desenvolvimento, surgiram conflitos de merge em arquivos compartilhados entre os módulos de professor e aluno. Atuei na resolução desses conflitos e apoiei colegas na integração dos módulos, o que exigiu compreender partes do código que não eram de minha responsabilidade direta e comunicar as dependências entre elas.

- **Proatividade**
  Identifiquei, antes de uma entrega de sprint, que o método de criação de sprints não validava sobreposição de datas — o que poderia gerar avaliações vinculadas a períodos ambíguos. Tomei a iniciativa de implementar as validações `isDateRangeAvailable()` e `isDuplicateSprint()` sem que isso estivesse previsto originalmente no backlog, prevenindo um problema que só seria percebido em produção.

- **Organização**
  Mantive commits atômicos e bem descritos no GitHub, separando cada funcionalidade implementada em seu próprio commit. Isso facilitou a revisão do código pela equipe e tornou mais simples identificar a origem de eventuais bugs nas entregas.

---

## Aprendizados Efetivos

Este projeto foi meu primeiro contato real com persistência de dados em uma aplicação desktop Java. Aprender o padrão DAO na prática — separando a lógica de acesso ao banco das regras de negócio — me deu uma base sólida que utilizei diretamente no semestre seguinte com Spring Boot.
A experiência de implementar exclusão lógica com `deleted_at` também foi um aprendizado importante: percebi que em sistemas com histórico de dados, deletar registros fisicamente é quase sempre a escolha errada. Essa decisão de design, tomada neste projeto, refletiu diretamente na qualidade da rastreabilidade do sistema ao longo dos semestres avaliados.

---

## Navegação

| Semestre | Projeto | Empresa Parceira |
|----------|---------|-----------------|
| [1º Semestre – 01/2024](API01.md) | Calculadora Científica | FATEC São José dos Campos |
| [3º Semestre – 01/2025](API03.md) | Checkpoint | Altave |
| [4º Semestre – 02/2025](API04.md) | Sistema de Mobilidade Urbana | Prefeitura de São José dos Campos |
[← Voltar ao Portfólio](README.md)
