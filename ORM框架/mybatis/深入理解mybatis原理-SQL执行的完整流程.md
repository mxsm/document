### Mybatis框架

MyBatis框架主要完成的是以下2件事情：

1. 根据JDBC规范建立与数据库的连接
2. 通过反射打通Java对象与数据库参数交互之间相互转换的关系

这个就是mybatis以及ORM框架主要做的两件事情

### Mybatis 主要类介绍

- **Configuration**

  MyBatis所有的配置信息都维持在Configuration对象之中，在也就是mybatis的xml，configuration节点

- **SqlSession**

  作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能

  ```java
  package org.apache.ibatis.session;
  
  import java.io.Closeable;
  import java.sql.Connection;
  import java.util.List;
  import java.util.Map;
  
  import org.apache.ibatis.cursor.Cursor;
  import org.apache.ibatis.executor.BatchResult;
  
  public interface SqlSession extends Closeable {
  
    <T> T selectOne(String statement);
  
    <T> T selectOne(String statement, Object parameter);
  
    <E> List<E> selectList(String statement);
  
    <E> List<E> selectList(String statement, Object parameter);
  
    <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);
  
    <K, V> Map<K, V> selectMap(String statement, String mapKey);
  
    <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey);
  
    <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds);
  
    <T> Cursor<T> selectCursor(String statement);
  
    <T> Cursor<T> selectCursor(String statement, Object parameter);
  
    <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds);
  
    void select(String statement, Object parameter, ResultHandler handler);
  
    void select(String statement, ResultHandler handler);
  
    void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler);
  
    int insert(String statement);
  
    int insert(String statement, Object parameter);
  
    int update(String statement);
  
    int update(String statement, Object parameter);
  
    int delete(String statement);
  
    int delete(String statement, Object parameter);
      
    void commit();
  
    void commit(boolean force);
  
    void rollback();
  
    void rollback(boolean force);
  
    List<BatchResult> flushStatements();
      
    @Override
    void close();
  
    void clearCache();
  
    Configuration getConfiguration();
  
    <T> T getMapper(Class<T> type);
  
    Connection getConnection();
  }
  
  ```

- **Executor**

  MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护

  ```java
  public interface Executor {
  
    ResultHandler NO_RESULT_HANDLER = null;
  
    int update(MappedStatement ms, Object parameter) throws SQLException;
  
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;
  
    <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
  
    <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
  
    List<BatchResult> flushStatements() throws SQLException;
  
    void commit(boolean required) throws SQLException;
  
    void rollback(boolean required) throws SQLException;
  
    CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
  
    boolean isCached(MappedStatement ms, CacheKey key);
  
    void clearLocalCache();
  
    void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);
  
    Transaction getTransaction();
  
    void close(boolean forceRollback);
  
    boolean isClosed();
  
    void setExecutorWrapper(Executor executor);
  
  }
  ```

- **StatementHandler**

  封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合

  ```java
  public interface StatementHandler {
  
    Statement prepare(Connection connection, Integer transactionTimeout)
        throws SQLException;
  
    void parameterize(Statement statement)
        throws SQLException;
  
    void batch(Statement statement)
        throws SQLException;
  
    int update(Statement statement)
        throws SQLException;
  
    <E> List<E> query(Statement statement, ResultHandler resultHandler)
        throws SQLException;
  
    <E> Cursor<E> queryCursor(Statement statement)
        throws SQLException;
  
    BoundSql getBoundSql();
  
    ParameterHandler getParameterHandler();
  
  }
  ```

- **ParameterHandler**

  负责对用户传递的参数转换成JDBC Statement 所需要的参数

  ```java
  public interface ParameterHandler {
  
    Object getParameterObject();
  
    void setParameters(PreparedStatement ps)
        throws SQLException;
  
  }
  ```

- **ResultSetHandler**

  负责将JDBC返回的ResultSet结果集对象转换成List类型的集合(单个查询是通过列表查询得来的)

  ```java
  public interface ResultSetHandler {
  
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;
  
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;
  
    void handleOutputParameters(CallableStatement cs) throws SQLException;
  
  }
  ```

- **TypeHandler**

  负责java数据类型和jdbc数据类型之间的映射和转换

  ```java
  public interface TypeHandler<T> {
  
    void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
  
    /**
     * @param columnName Colunm name, when configuration <code>useColumnLabel</code> is <code>false</code>
     */
    T getResult(ResultSet rs, String columnName) throws SQLException;
  
    T getResult(ResultSet rs, int columnIndex) throws SQLException;
  
    T getResult(CallableStatement cs, int columnIndex) throws SQLException;
  
  }
  ```

- **MappedStatement**

  MappedStatement维护了一条<select|update|delete|insert>节点的封装

- **SqlSource**

  负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回

  ```java
  public interface SqlSource {
  
    BoundSql getBoundSql(Object parameterObject);
  
  }
  ```

- **BoundSql**

  表示动态生成的SQL语句以及相应的参数信息

  ```java
  public class BoundSql {
  
    private final String sql;
    private final List<ParameterMapping> parameterMappings;
    private final Object parameterObject;
    private final Map<String, Object> additionalParameters;
    private final MetaObject metaParameters;
  
    public BoundSql(Configuration configuration, String sql, List<ParameterMapping> parameterMappings, Object parameterObject) {
      this.sql = sql;
      this.parameterMappings = parameterMappings;
      this.parameterObject = parameterObject;
      this.additionalParameters = new HashMap<>();
      this.metaParameters = configuration.newMetaObject(additionalParameters);
    }
  
    public String getSql() {
      return sql;
    }
  
    public List<ParameterMapping> getParameterMappings() {
      return parameterMappings;
    }
  
    public Object getParameterObject() {
      return parameterObject;
    }
  
    public boolean hasAdditionalParameter(String name) {
      String paramName = new PropertyTokenizer(name).getName();
      return additionalParameters.containsKey(paramName);
    }
  
    public void setAdditionalParameter(String name, Object value) {
      metaParameters.setValue(name, value);
    }
  
    public Object getAdditionalParameter(String name) {
      return metaParameters.getValue(name);
    }
  }
  ```

  ![](https://img-blog.csdn.net/20141028140852531?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVhbmxvdWlz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

  图片来源：https://blog.csdn.net/luanlouis/article/details/40422941

- **Interceptor**

  拦截器，MyBatis 允许你在已映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

  - Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
  - ParameterHandler (getParameterObject, setParameters)
  - ResultSetHandler (handleResultSets, handleOutputParameters)
  - StatementHandler (prepare, parameterize, batch, update, query)

### SQL的执行流程

