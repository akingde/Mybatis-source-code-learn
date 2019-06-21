## 前言
本文将深入探索Mybatis的Executor的源码实现，然后把源码实现的细节都过一遍，过程中根据自己的疑问，在寻找答案的过程中来完整自己的理解，进而给我和大家以启发。

## 正文
### 包结构
**Executor包结构如下：**
![52c822d794de18fcd148bcdf57b3d345.png](evernotecid://53254696-49A1-438A-9FEC-BFEBE9152736/wwwevernotecom/45408501/ENResource/p4129)

以下将按照包结构逐个阅读源码进行学习。

###### ***为了更好的理解Mybatis做了什么事，我们可以先思考下如果没有Mybatis，要怎么完成我们和数据库交互的需求呢？***
在那个还用诺基亚发短信调情的时代我们经常写这样的代码：
```java
import java.sql.*;

public class FirstExample {
   // JDBC driver name and database URL
   static final String JDBC_DRIVER = "com.mysql.jdbc.Driver";  
   static final String DB_URL = "jdbc:mysql://localhost/emp";

   //  Database credentials
   static final String USER = "root";
   static final String PASS = "123456";

   public static void main(String[] args) {
   Connection conn = null;
   Statement stmt = null;
   try{
      //STEP 2: Register JDBC driver
      Class.forName("com.mysql.jdbc.Driver");

      //STEP 3: Open a connection
      System.out.println("Connecting to database...");
      conn = DriverManager.getConnection(DB_URL,USER,PASS);

      //STEP 4: Execute a query
      System.out.println("Creating statement...");
      stmt = conn.createStatement();
      String sql;
      sql = "SELECT id, first, last, age FROM Employees";
      ResultSet rs = stmt.executeQuery(sql);

      //STEP 5: Extract data from result set
      while(rs.next()){
         //Retrieve by column name
         int id  = rs.getInt("id");
         int age = rs.getInt("age");
         String first = rs.getString("first");
         String last = rs.getString("last");

         //Display values
         System.out.print("ID: " + id);
         System.out.print(", Age: " + age);
         System.out.print(", First: " + first);
         System.out.println(", Last: " + last);
      }
      //STEP 6: Clean-up environment
      rs.close();
      stmt.close();
      conn.close();
   }catch(SQLException se){
      //Handle errors for JDBC
      se.printStackTrace();
   }catch(Exception e){
      //Handle errors for Class.forName
      e.printStackTrace();
   }finally{
      //finally block used to close resources
      try{
         if(stmt!=null)
            stmt.close();
      }catch(SQLException se2){
      }// nothing we can do
      try{
         if(conn!=null)
            conn.close();
      }catch(SQLException se){
         se.printStackTrace();
      }//end finally try
   }//end try
   System.out.println("There are so thing wrong!");
}//end main
}//end FirstExample
```
以上步骤大多将会被Executor模块封装，以提供出自己的API供上游使用。
### Statement
statement模块核心实现处理java.sql.Statement，将sql模版语句和参数组装成可执行的sql，传给数据库执行，将返回结果映射给定义的结果对象。
#### StatementHandler接口
```java
public interface StatementHandler {
  // 从connection中获取statement
  Statement prepare(Connection connection, Integer transactionTimeout)
      throws SQLException;
  // 对sql进行设置参数
  void parameterize(Statement statement)
      throws SQLException;
  // 批量执行
  void batch(Statement statement)
      throws SQLException;
  // 执行预编译后的sql语句（update,delete,insert）
  int update(Statement statement)
      throws SQLException;
  // 执行查询sql
  <E> List<E> query(Statement statement, ResultHandler resultHandler)
      throws SQLException;
  // 使用游标执行查询sql
  <E> Cursor<E> queryCursor(Statement statement)
      throws SQLException;
  // 获取执行SQL语句的封装类BoundSql
  BoundSql getBoundSql();
  // 参数处理器
  ParameterHandler getParameterHandler();
}
```

**先看一下接口下面的实现类关系：**
![8ce28d16d5aa09bb3eb535a3370bc481.png](evernotecid://53254696-49A1-438A-9FEC-BFEBE9152736/wwwevernotecom/45408501/ENResource/p4132)


##### BaseStatementHandler
`BaseStatementHandler`作为继承`StatementHandler`接口的抽象类存在。
```java
public abstract class BaseStatementHandler implements StatementHandler {

  protected final Configuration configuration;
  protected final ObjectFactory objectFactory;
  protected final TypeHandlerRegistry typeHandlerRegistry;
  protected final ResultSetHandler resultSetHandler;
  protected final ParameterHandler parameterHandler;

  protected final Executor executor;
  protected final MappedStatement mappedStatement;
  protected final RowBounds rowBounds;

  protected BoundSql boundSql;

  protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    this.configuration = mappedStatement.getConfiguration();
    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;

    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();
    
    if (boundSql == null) { // issue #435, get the key before calculating the statement
      generateKeys(parameterObject);
      boundSql = mappedStatement.getBoundSql(parameterObject);
    }

    this.boundSql = boundSql;

    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }

  @Override
  public BoundSql getBoundSql() {
    return boundSql;
  }

  @Override
  public ParameterHandler getParameterHandler() {
    return parameterHandler;
  }

  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
  }

  protected abstract Statement instantiateStatement(Connection connection) throws SQLException;

  protected void setStatementTimeout(Statement stmt, Integer transactionTimeout) throws SQLException {
    Integer queryTimeout = null;
    if (mappedStatement.getTimeout() != null) {
      queryTimeout = mappedStatement.getTimeout();
    } else if (configuration.getDefaultStatementTimeout() != null) {
      queryTimeout = configuration.getDefaultStatementTimeout();
    }
    if (queryTimeout != null) {
      stmt.setQueryTimeout(queryTimeout);
    }
    StatementUtil.applyTransactionTimeout(stmt, queryTimeout, transactionTimeout);
  }

  protected void setFetchSize(Statement stmt) throws SQLException {
    Integer fetchSize = mappedStatement.getFetchSize();
    if (fetchSize != null) {
      stmt.setFetchSize(fetchSize);
      return;
    }
    Integer defaultFetchSize = configuration.getDefaultFetchSize();
    if (defaultFetchSize != null) {
      stmt.setFetchSize(defaultFetchSize);
    }
  }

  protected void closeStatement(Statement statement) {
    try {
      if (statement != null) {
        statement.close();
      }
    } catch (SQLException e) {
      //ignore
    }
  }

  protected void generateKeys(Object parameter) {
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    ErrorContext.instance().store();
    keyGenerator.processBefore(executor, mappedStatement, null, parameter);
    ErrorContext.instance().recall();
  }

}
```
它内部维护了核心字段：
* parameterHandler 作用就是将参数替换sql中的占位符的功能
* resultSetHandler 将sql结果集映射成结果对象
* executor 执行sql的抽象，后续详细看

只实现了三个接口方法：
* getBoundSql
* getParameterHandler
* prepare 其中prepare的实现使用了模版模式，子类必须实现`instantiateStatement`方法来完成父类`prepare`方法的模版。

在BaseStatementHandler的构造函数中我们可以看到一段调用generateKeys的代码，它和生成主键key有关。

关于获得主键key和插入数据时放入主键key牵涉到以下几个配置：
* selectKey
* useGeneratedKeys  设置true使用自动生成的主键 
* keyProperty 指定主键是(javaBean的)哪个属性。

对于支持自动生成记录主键的数据库，如：MySQL，SQL Server，此时设置useGeneratedKeys参数值为true，在执行添加记录之后可以获取到数据库自动生成的主键ID，合keyProperty指定主键。
例子：
```xml
  <insert id="saveMsg" parameterType="cn.com.tt.e.nano.Notice"
        useGeneratedKeys="true" keyProperty="msgId">
        insert into notice(msg_type,title,content,rec_time,send_time,user_id,deleted,viewed)
        values(#{msgType,jdbcType=INTEGER},#{title,jdbcType=VARCHAR},#{content,jdbcType=VARCHAR},
               #{recTime,jdbcType=BIGINT},#{sendTime,jdbcType=BIGINT},#{userId,jdbcType=VARCHAR},
               #{deleted,jdbcType=TINYINT},#{viewed,jdbcType=INTEGER})
    </insert>
```
如果使用`selectKey`，可以设置order属性为AFTER。
例子：
```xml
<insert id="insertAndgetkey" parameterType="com.soft.mybatis.model.User">
        <selectKey keyProperty="id" order="AFTER" resultType="java.lang.Integer">
            SELECT LAST_INSERT_ID()
        </selectKey>
        insert into t_user (username,password,create_date) values(#{username},#{password},#{createDate})
    </insert>
```

对于Oracle数据库，当要用到自增字段时，需要用到Sequence或者使用外界传入的唯一值比如uuid。则也使用selectKey，设置order为before。
例子：
```xml
<insert id="insert"  parameterType="com.lzumetal.mybatis.entity.Employee">
    <selectKey keyProperty="id" resultType="long" order="BEFORE">
    SELECT SEQ_ADMIN.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO tbl_employee(id, name, age, create_time) 
    VALUES(#{id}, #{name}, #{age}, #{createTime})
</insert>
```


###### KeyGenerator
KeyGenerator接口的实现有：`Jdbc3KeyGenerator`，`SelectKeyGenerator`，`NoKeyGenerator`


`useGeneratedKeys`设置在settings配置文件中的相关源码在`MappedStatement`的`org.apache.ibatis.mapping.MappedStatement.Builder#Builder`中：
```java
mappedStatement.keyGenerator = configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType) ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
```
默认是NoKeyGenerator，如果配置了`useGeneratedKeys=true`并且是insert操作则使用`Jdbc3KeyGenerator`。

`useGeneratedKeys`设置在mapper文件中的相关源码：
```java
if (configuration.hasKeyGenerator(keyStatementId)) {
  keyGenerator = configuration.getKeyGenerator(keyStatementId);
} else {
  keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
      configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
      ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
}
```
如果设置了`selectKey`则使用`SelectKeyGenerator`（这部分可以更下去看到），`useGeneratedKeys`逻辑和前面一样。

`selectKey`标签中的`order`分别对应着`KeyGenerator`中的`processBefore`方法和`processAfter`方法。
而前面提到的BaseStatementHandler中generateKeys方法就是触发processBefore的地方，也是唯一一个地方。

```java
  protected void generateKeys(Object parameter) {
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    ErrorContext.instance().store();
    keyGenerator.processBefore(executor, mappedStatement, null, parameter);
    ErrorContext.instance().recall();
  }
```

在`org.apache.ibatis.executor.statement.StatementHandler#update`中的各个实现中都可以看到执行`processAfter`的代码，比如SimpleStatementHandler的代码：
```java
  public int update(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    int rows;
    if (keyGenerator instanceof Jdbc3KeyGenerator) {
      statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else if (keyGenerator instanceof SelectKeyGenerator) {
      statement.execute(sql);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else {
      statement.execute(sql);
      rows = statement.getUpdateCount();
    }
    return rows;
  }
```

###### ParameterHandler
再来看一下前面提到的`ParameterHandler`，功能就是将动态的sql中的占位符替换成实参。
它的实现是`DefaultParameterHandler`：
```java
public class DefaultParameterHandler implements ParameterHandler {

  private final TypeHandlerRegistry typeHandlerRegistry;

  private final MappedStatement mappedStatement;
  private final Object parameterObject;
  private final BoundSql boundSql;
  private final Configuration configuration;

  public DefaultParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    this.mappedStatement = mappedStatement;
    this.configuration = mappedStatement.getConfiguration();
    this.typeHandlerRegistry = mappedStatement.getConfiguration().getTypeHandlerRegistry();
    this.parameterObject = parameterObject;
    this.boundSql = boundSql;
  }

  @Override
  public Object getParameterObject() {
    return parameterObject;
  }

  @Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          } catch (SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }

}
```

#### SimpleStatementHandler类
SimpleStatementHandler是BaseStatementHandler的子类，使用Statement来完成数据库的操作，所以Sql中不会有占位符，`parameterize`就是空实现。
**query方法：**
```java
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    return resultSetHandler.<E>handleResultSets(statement);
  }
```
最后一步将数据库数据转换成java对象，这个后续展开。
**update方法：**
```java
  public int update(Statement statement) throws SQLException {
    String sql = boundSql.getSql();
    Object parameterObject = boundSql.getParameterObject();
    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
    int rows;
    if (keyGenerator instanceof Jdbc3KeyGenerator) {
      statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else if (keyGenerator instanceof SelectKeyGenerator) {
      statement.execute(sql);
      rows = statement.getUpdateCount();
      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
    } else {
      statement.execute(sql);
      rows = statement.getUpdateCount();
    }
    return rows;
  }
```
其中对主键处理的代码前面已经作了解释，从上面两个代码片段来看已经挖到最深了，最终都是调用`java.sql.*`的`api`。


#### PreparedStatementHandler
PreparedStatementHandler使用PreparedStatement实现。
```java
  public <E> Cursor<E> queryCursor(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.<E> handleCursorResultSets(ps);
  }
    public void parameterize(Statement statement) throws SQLException {
    parameterHandler.setParameters((PreparedStatement) statement);
  }
```


#### RoutingStatementHandler
看名字就可以猜到了这个是路由用的，看它的构造函数：
```java
  private final StatementHandler delegate;

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

  }
```
所以具体使用哪一个`StatementHandler`是由MappedStatement.getStatementType()决定的
#### CallableStatementHandler
依赖`CallableStatement`，用于调用存储过程。



### Executor
