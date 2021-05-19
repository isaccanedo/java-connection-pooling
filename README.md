### Um guia simples para pool de conexão em Java

# 1. Visão geral
O pool de conexão é um padrão de acesso a dados bem conhecido, cujo objetivo principal é reduzir a sobrecarga envolvida na execução de conexões de banco de dados e operações de leitura/gravação de banco de dados.

Resumindo, um pool de conexão é, no nível mais básico, uma implementação de cache de conexão de banco de dados, que pode ser configurada para atender a requisitos específicos.

Neste tutorial, faremos um resumo rápido de algumas estruturas de pool de conexões populares e aprenderemos como implementar do zero nosso próprio pool de conexões.

# 2. Por que pool de conexão?
A pergunta é retórica, é claro.

Se analisarmos a sequência de etapas envolvidas em um ciclo de vida típico de conexão de banco de dados, entenderemos por quê:

- Abrir uma conexão com o banco de dados usando o driver do banco de dados;
- Abertura de um socket TCP para leitura / escrita de dados;
- Leitura/escrita de dados no socket;
- Fechando a conexão;
- Fechando o soquete.

Torna-se evidente que as conexões de banco de dados são operações bastante caras e, como tal, devem ser reduzidas ao mínimo em todos os casos de uso possíveis (em casos extremos, apenas evitados).

É aqui que as implementações de pool de conexão entram em jogo.

Simplesmente implementando um contêiner de conexão de banco de dados, que nos permite reutilizar uma série de conexões existentes, podemos efetivamente economizar o custo de realizar um grande número de viagens caras de banco de dados, aumentando assim o desempenho geral de nossos aplicativos orientados a banco de dados.

# 3. Estruturas de pool de conexão JDBC
De uma perspectiva pragmática, implementar um pool de conexão do zero é simplesmente inútil, considerando o número de estruturas de pool de conexão "prontas para a empresa" disponíveis por aí.

Do lado didático, que é o objetivo deste artigo, não é.

Mesmo assim, antes de aprendermos como implementar um pool de conexão básico, vamos primeiro mostrar algumas estruturas populares de pool de conexão.

### 3.1. Apache Commons DBCP
Vamos começar este rápido resumo com Apache Commons DBCP Component, uma estrutura JDBC de pooling de conexões com recursos completos:

```
public class DBCPDataSource {
    
    private static BasicDataSource ds = new BasicDataSource();
    
    static {
        ds.setUrl("jdbc:h2:mem:test");
        ds.setUsername("user");
        ds.setPassword("password");
        ds.setMinIdle(5);
        ds.setMaxIdle(10);
        ds.setMaxOpenPreparedStatements(100);
    }
    
    public static Connection getConnection() throws SQLException {
        return ds.getConnection();
    }
    
    private DBCPDataSource(){ }
}
```

Nesse caso, usamos uma classe wrapper com um bloco estático para configurar facilmente as propriedades do DBCP.

Veja como obter uma conexão em pool com a classe DBCPDataSource:

```
Connection con = DBCPDataSource.getConnection();
```

### 3.2. HikariCP
Continuando, vamos dar uma olhada no HikariCP, uma estrutura de pool de conexão JDBC ultrarrápida criada por Brett Wooldridge (para obter os detalhes completos sobre como configurar e obter o máximo do HikariCP, consulte este artigo):

```
public class HikariCPDataSource {
    
    private static HikariConfig config = new HikariConfig();
    private static HikariDataSource ds;
    
    static {
        config.setJdbcUrl("jdbc:h2:mem:test");
        config.setUsername("user");
        config.setPassword("password");
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        ds = new HikariDataSource(config);
    }
    
    public static Connection getConnection() throws SQLException {
        return ds.getConnection();
    }
    
    private HikariCPDataSource(){}
}
```

Da mesma forma, veja como obter uma conexão agrupada com a classe HikariCPDataSource:

```
Connection con = HikariCPDataSource.getConnection();
```

### 3.3. C3PO
O último desta revisão é o C3PO, uma poderosa conexão JDBC4 e estrutura de pool de instruções desenvolvida por Steve Waldman:

```
public class C3poDataSource {

    private static ComboPooledDataSource cpds = new ComboPooledDataSource();

    static {
        try {
            cpds.setDriverClass("org.h2.Driver");
            cpds.setJdbcUrl("jdbc:h2:mem:test");
            cpds.setUser("user");
            cpds.setPassword("password");
        } catch (PropertyVetoException e) {
            // handle the exception
        }
    }
    
    public static Connection getConnection() throws SQLException {
        return cpds.getConnection();
    }
    
    private C3poDataSource(){}
}
```

Como esperado, obter uma conexão agrupada com a classe C3poDataSource é semelhante aos exemplos anteriores:

```
Connection con = C3poDataSource.getConnection();
```

# 4. Uma implementação simples
Para entender melhor a lógica subjacente do pool de conexão, vamos criar uma implementação simples.

Vamos começar com um design fracamente acoplado, baseado em apenas uma única interface:

```
public interface ConnectionPool {
    Connection getConnection();
    boolean releaseConnection(Connection connection);
    String getUrl();
    String getUser();
    String getPassword();
}
```

A interface ConnectionPool define a API pública de um pool de conexão básico.

Agora, vamos criar uma implementação, que fornece algumas funcionalidades básicas, incluindo obter e liberar uma conexão em pool:

```
public class BasicConnectionPool 
  implements ConnectionPool {

    private String url;
    private String user;
    private String password;
    private List<Connection> connectionPool;
    private List<Connection> usedConnections = new ArrayList<>();
    private static int INITIAL_POOL_SIZE = 10;
    
    public static BasicConnectionPool create(
      String url, String user, 
      String password) throws SQLException {
 
        List<Connection> pool = new ArrayList<>(INITIAL_POOL_SIZE);
        for (int i = 0; i < INITIAL_POOL_SIZE; i++) {
            pool.add(createConnection(url, user, password));
        }
        return new BasicConnectionPool(url, user, password, pool);
    }
    
    // standard constructors
    
    @Override
    public Connection getConnection() {
        Connection connection = connectionPool
          .remove(connectionPool.size() - 1);
        usedConnections.add(connection);
        return connection;
    }
    
    @Override
    public boolean releaseConnection(Connection connection) {
        connectionPool.add(connection);
        return usedConnections.remove(connection);
    }
    
    private static Connection createConnection(
      String url, String user, String password) 
      throws SQLException {
        return DriverManager.getConnection(url, user, password);
    }
    
    public int getSize() {
        return connectionPool.size() + usedConnections.size();
    }

    // standard getters
}
```

Embora bastante ingênua, a classe BasicConnectionPool fornece a funcionalidade mínima que esperaríamos de uma implementação típica de pool de conexão.

Em suma, a classe inicializa um pool de conexão com base em um ArrayList que armazena 10 conexões, que podem ser facilmente reutilizadas.

É possível criar conexões JDBC com a classe DriverManager e com implementações de Datasource.

Como é muito melhor manter a criação do banco de dados de conexões agnóstica, usamos o primeiro, dentro do método estático de fábrica create().

Nesse caso, colocamos o método no BasicConnectionPool, porque essa é a única implementação da interface.

Em um design mais complexo, com várias implementações de ConnectionPool, seria preferível colocá-lo na interface, obtendo assim um design mais flexível e um maior nível de coesão.

O ponto mais relevante a enfatizar aqui é que, uma vez que o pool é criado, as conexões são obtidas do pool, portanto, não há necessidade de criar novas.

Além disso, quando uma conexão é liberada, ela realmente retorna ao pool, para que outros clientes possam reutilizá-la.

Não há nenhuma interação adicional com o banco de dados subjacente, como uma chamada explícita ao método close() do Connection.

# 5. Usando a classe BasicConnectionPool
Como esperado, usar nossa classe BasicConnectionPool é simples.

Vamos criar um teste de unidade simples e obter uma conexão H2 na memória em pool:

```
@Test
public whenCalledgetConnection_thenCorrect() {
    ConnectionPool connectionPool = BasicConnectionPool
      .create("jdbc:h2:mem:test", "user", "password");
 
    assertTrue(connectionPool.getConnection().isValid(1));
}
```

# 6. Outras melhorias e refatoração
Claro, há muito espaço para ajustar/estender a funcionalidade atual de nossa implementação de pool de conexão.

Por exemplo, poderíamos refatorar o método getConnection() e adicionar suporte para o tamanho máximo do pool. Se todas as conexões disponíveis forem feitas e o tamanho do pool atual for menor que o máximo configurado, o método criará uma nova conexão.

Além disso, poderíamos verificar adicionalmente se a conexão obtida do pool ainda está ativa, antes de passá-la para o cliente.

```
@Override
public Connection getConnection() throws SQLException {
    if (connectionPool.isEmpty()) {
        if (usedConnections.size() < MAX_POOL_SIZE) {
            connectionPool.add(createConnection(url, user, password));
        } else {
            throw new RuntimeException(
              "Maximum pool size reached, no available connections!");
        }
    }

    Connection connection = connectionPool
      .remove(connectionPool.size() - 1);

    if(!connection.isValid(MAX_TIMEOUT)){
        connection = createConnection(url, user, password);
    }

    usedConnections.add(connection);
    return connection;
}
```

Observe que o método agora lança SQLException, o que significa que teremos que atualizar a assinatura da interface também.

Ou podemos adicionar um método para desligar normalmente nossa instância de pool de conexão:

```
public void shutdown() throws SQLException {
    usedConnections.forEach(this::releaseConnection);
    for (Connection c : connectionPool) {
        c.close();
    }
    connectionPool.clear();
}
```

Em implementações prontas para produção, um pool de conexão deve fornecer vários recursos extras, como a capacidade de rastrear as conexões que estão atualmente em uso, suporte para pooling de instruções preparadas e assim por diante.

Como vamos manter as coisas simples, omitiremos como implementar esses recursos adicionais e manter a implementação sem thread-safe para fins de clareza.

# 7. Conclusão
Neste artigo, examinamos detalhadamente o que é pool de conexão e aprendemos como implementar nossa própria implementação de pool de conexão.

Obviamente, não precisamos começar do zero toda vez que quisermos adicionar uma camada de pooling de conexão com todos os recursos aos nossos aplicativos.

É por isso que fizemos primeiro um resumo simples mostrando alguns dos frameworks de pool de conexão mais populares, para que possamos ter uma ideia clara de como trabalhar com eles e escolher aquele que melhor se adapta aos nossos requisitos.

