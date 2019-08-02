### Mybatis连接池
有这么个定律，有连接的地方就有池。
在市面上，可以适配Mybatis DateSource的连接池有很对，比如：
* [druid](https://github.com/alibaba/druid)
* [hikari](https://github.com/brettwooldridge/HikariCP)
* [c3p0](https://github.com/swaldman/c3p0)

Mybatis也自带来连接池的功能，先学习下Mybatis的，相对简单的实现。
主要涉及的类：
![869e4598b4af3afd87a4fee48d6ad5b0.png](evernotecid://311BA442-9807-4482-A3A6-F96E33C77ED9/wwwevernotecom/45408501/ENResource/p4210)


##### PoolState

```java
public class PoolState {

  protected PooledDataSource dataSource;
  // 空闲连接集合
  protected final List<PooledConnection> idleConnections = new ArrayList<PooledConnection>();
  // 正在使用的连接集合
  protected final List<PooledConnection> activeConnections = new ArrayList<PooledConnection>();
  // 请求次数，每次获取连接，都会自增，用于
  protected long requestCount = 0;
  // 累计请求耗时，每次获取连接时计算累加，除以requestCount可以获得平均耗时
  protected long accumulatedRequestTime = 0;
  // 累计连接使用时间
  protected long accumulatedCheckoutTime = 0;
  // 过期连接次数
  protected long claimedOverdueConnectionCount = 0;
  protected long accumulatedCheckoutTimeOfOverdueConnections = 0;
  // 累计等待获取连接时间
  protected long accumulatedWaitTime = 0;
  // 等待获取连接的次数
  protected long hadToWaitCount = 0;
  // 连接已关闭的次数
  protected long badConnectionCount = 0;

  public PoolState(PooledDataSource dataSource) {
    this.dataSource = dataSource;
  }

  public synchronized long getRequestCount() {
    return requestCount;
  }

  public synchronized long getAverageRequestTime() {
    return requestCount == 0 ? 0 : accumulatedRequestTime / requestCount;
  }

  public synchronized long getAverageWaitTime() {
    return hadToWaitCount == 0 ? 0 : accumulatedWaitTime / hadToWaitCount;

  }

  public synchronized long getHadToWaitCount() {
    return hadToWaitCount;
  }

  public synchronized long getBadConnectionCount() {
    return badConnectionCount;
  }

  public synchronized long getClaimedOverdueConnectionCount() {
    return claimedOverdueConnectionCount;
  }

  public synchronized long getAverageOverdueCheckoutTime() {
    return claimedOverdueConnectionCount == 0 ? 0 : accumulatedCheckoutTimeOfOverdueConnections / claimedOverdueConnectionCount;
  }

  public synchronized long getAverageCheckoutTime() {
    return requestCount == 0 ? 0 : accumulatedCheckoutTime / requestCount;
  }


  public synchronized int getIdleConnectionCount() {
    return idleConnections.size();
  }

  public synchronized int getActiveConnectionCount() {
    return activeConnections.size();
  }
}
```
注意代码中的字段都是用protected修饰的，表示pooled包内都可访问，在写这份代码的时候必然默认这个包下实现一个独立的功能，内部字段都可以共享使用，否则都写set，get方法太麻烦了。
在`PoolState`类中，很多指标比如`requestCount`，`claimedOverdueConnectionCount`等都不和连接池核心逻辑相关，纯粹只是表示连接池的一些指标而已。
作为连接池，在这里最重要的就是两个List：
* idleConnections
* activeConnections
这两个都是ArrayList，所以在整个实现中我们是通过`synchronized`关键字来处理并发场景的。

##### PooledConnection
组成池的两个List中存储的是`PooledConnection`，而`PooledConnection`通过java动态代理机制实现代理真正Connection。
`PooledConnection`继承`InvocationHandler`，所以实现了`invoke`方法：
```java
  /*
   * Required for InvocationHandler implementation.
   *
   * @param proxy  - not used
   * @param method - the method to be executed
   * @param args   - the parameters to be passed to the method
   * @see java.lang.reflect.InvocationHandler#invoke(Object, java.lang.reflect.Method, Object[])
   */
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    String methodName = method.getName();
    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
      dataSource.pushConnection(this);
      return null;
    } else {
      try {
        if (!Object.class.equals(method.getDeclaringClass())) {
          // issue #579 toString() should never fail
          // throw an SQLException instead of a Runtime
          checkConnection();
        }
        return method.invoke(realConnection, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }

  private void checkConnection() throws SQLException {
    if (!valid) {
      throw new SQLException("Error accessing PooledConnection. Connection is invalid.");
    }
  }
```
主要看到这个代理实现处理了`close`方法，就是将连接从使用列表中弹出。
对于其他方法，会判断方法是否属于Object中的方法，如果不是则进行连接合法的校验，然后执行真正`Connection`即`realConnection`中对应的方法。
获得一个代理类的代码，即调用`Proxy.newProxyInstance`方法，在`PooledConnection`中的构造函数中：
```java
  /*
   * Constructor for SimplePooledConnection that uses the Connection and PooledDataSource passed in
   *
   * @param connection - the connection that is to be presented as a pooled connection
   * @param dataSource - the dataSource that the connection is from
   */
  public PooledConnection(Connection connection, PooledDataSource dataSource) {
    this.hashCode = connection.hashCode();
    this.realConnection = connection;
    this.dataSource = dataSource;
    this.createdTimestamp = System.currentTimeMillis();
    this.lastUsedTimestamp = System.currentTimeMillis();
    this.valid = true;
    this.proxyConnection = (Connection) Proxy.newProxyInstance(Connection.class.getClassLoader(), IFACES, this);
  }
```
我们可以看到`realConnection`是在构造函数时就传入的了。


而配置这个池的参数都是在`PooledDataSource`中：

>**官方文档：**
poolMaximumActiveConnections – 在任意时间可以存在的活动（也就是正在使用）连接数量，默认值：10
poolMaximumIdleConnections – 任意时间可能存在的空闲连接数。
poolMaximumCheckoutTime – 在被强制返回之前，池中连接被检出（checked out）时间，默认值：20000 毫秒（即 20 秒）
poolTimeToWait – 这是一个底层设置，如果获取连接花费了相当长的时间，连接池会打印状态日志并重新尝试获取一个连接（避免在误配置的情况下一直安静的失败），默认值：20000 毫秒（即 20 秒）。
poolMaximumLocalBadConnectionTolerance – 这是一个关于坏连接容忍度的底层设置， 作用于每一个尝试从缓存池获取连接的线程。 如果这个线程获取到的是一个坏的连接，那么这个数据源允许这个线程尝试重新获取一个新的连接，但是这个重新尝试的次数不应该超过 poolMaximumIdleConnections 与 poolMaximumLocalBadConnectionTolerance 之和。 默认值：3 （新增于 3.4.5）
poolPingQuery – 发送到数据库的侦测查询，用来检验连接是否正常工作并准备接受请求。默认是“NO PING QUERY SET”，这会导致多数数据库驱动失败时带有一个恰当的错误消息。
poolPingEnabled – 是否启用侦测查询。若开启，需要设置 poolPingQuery 属性为一个可执行的 SQL 语句（最好是一个速度非常快的 SQL 语句），默认值：false。
poolPingConnectionsNotUsedFor – 配置 poolPingQuery 的频率。可以被设置为和数据库连接超时时间一样，来避免不必要的侦测，默认值：0（即所有连接每一时刻都被侦测 — 当然仅当 poolPingEnabled 为 true 时适用）。

##### PooledDataSource

`PooledDataSource`完成池功能的类，直接看拿连接的`popConnection`方法：
```java

  private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    // 触发获取连接的当前时间
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;

    while (conn == null) {
      // 同步
      synchronized (state) {
        // 判断空闲列表中是否可以提供连接
        if (!state.idleConnections.isEmpty()) {
          // Pool has available connection
          conn = state.idleConnections.remove(0);
          if (log.isDebugEnabled()) {
            log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
          }
        } else {
          // Pool does not have available connection
          // 判断是否达到最大连接数限制
          if (state.activeConnections.size() < poolMaximumActiveConnections) {
            // Can create new connection
            conn = new PooledConnection(dataSource.getConnection(), this);
            if (log.isDebugEnabled()) {
              log.debug("Created connection " + conn.getRealHashCode() + ".");
            }
          } else {
            // Cannot create new connection
            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
            // 判断最老一个连接使用时间是否超过最大值
            if (longestCheckoutTime > poolMaximumCheckoutTime) {
              // Can claim overdue connection
              state.claimedOverdueConnectionCount++;
              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
              state.accumulatedCheckoutTime += longestCheckoutTime;
              state.activeConnections.remove(oldestActiveConnection);
              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                try {
                  oldestActiveConnection.getRealConnection().rollback();
                } catch (SQLException e) {
                  /*
                     Just log a message for debug and continue to execute the following
                     statement like nothing happend.
                     Wrap the bad connection with a new PooledConnection, this will help
                     to not intterupt current executing thread and give current thread a
                     chance to join the next competion for another valid/good database
                     connection. At the end of this loop, bad {@link @conn} will be set as null.
                   */
                  log.debug("Bad connection. Could not roll back");
                }  
              }
              // 这里看到将包装在oldestActiveConnection中的RealConnection重新用PooledConnection包装出来直接使用，看前面操作是将连接进行回滚，但是可能失败，却不关心，注释解释是，在后面的代码中会进行isValid的判断，其中就会判断连接是否可用。
              conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
              conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
              conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
              // 将老连接设置成invalid 
              oldestActiveConnection.invalidate();
              if (log.isDebugEnabled()) {
                log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
              }
            } else {
              // Must wait
              try {
                if (!countedWait) {
                  state.hadToWaitCount++;
                  countedWait = true;
                }
                if (log.isDebugEnabled()) {
                  log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
                }
                long wt = System.currentTimeMillis();
                // 线程等待，也释放了锁
                state.wait(poolTimeToWait);
                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
              } catch (InterruptedException e) {
                break;
              }
            }
          }
        }
        if (conn != null) {
          // ping to server and check the connection is valid or not
          if (conn.isValid()) {
            if (!conn.getRealConnection().getAutoCommit()) {
              conn.getRealConnection().rollback();
            }
            conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
            conn.setCheckoutTimestamp(System.currentTimeMillis());
            conn.setLastUsedTimestamp(System.currentTimeMillis());
            state.activeConnections.add(conn);
            state.requestCount++;
            state.accumulatedRequestTime += System.currentTimeMillis() - t;
          } else {
            if (log.isDebugEnabled()) {
              log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
            }
            state.badConnectionCount++;
            localBadConnectionCount++;
            // 不可用的连接会被设置成null，被回收器回收
            conn = null;
            if (localBadConnectionCount > (poolMaximumIdleConnections + poolMaximumLocalBadConnectionTolerance)) {
              if (log.isDebugEnabled()) {
                log.debug("PooledDataSource: Could not get a good connection to the database.");
              }
              throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
            }
          }
        }
      }

    }

    if (conn == null) {
      if (log.isDebugEnabled()) {
        log.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
      }
      throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
    }

    return conn;
  }

```

`popConnection`方法实现在一个池中获取连接的基本逻辑，依赖最大连接数，获取等待时间，连接使用超时时间等来完成一个池的核心能力。
注意这里使用`wait`方法来等待，在java线程池中使用阻塞队列来出来暂时拿不到资源的请求。

前面提到，在使用`Connection`时，调用`close`方法，会调用到`dataSource.pushConnection(this);`，就是将这个连接使用完了还回池的动作：
```java
protected void pushConnection(PooledConnection conn) throws SQLException {
    // 一样加锁
    synchronized (state) {
      // 从使用线程列表中删除
      state.activeConnections.remove(conn);
      if (conn.isValid()) {
        // 判断空闲连接列表是否超过最大值
        if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
          // 加入到空闲连接列表中
          state.idleConnections.add(newConn);
          newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
          newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
          conn.invalidate();
          if (log.isDebugEnabled()) {
            log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
          }
          // 通知等待线程
          state.notifyAll();
        } else {
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          conn.getRealConnection().close();
          if (log.isDebugEnabled()) {
            log.debug("Closed connection " + conn.getRealHashCode() + ".");
          }
          conn.invalidate();
        }
      } else {
        if (log.isDebugEnabled()) {
          log.debug("A bad connection (" + conn.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
        }
        state.badConnectionCount++;
      }
    }
  }
```

归还连接时，需要查看空闲列表中的线程数量是否已经到到设置的最大值，如果已经达到，就不需要归还了，凡是需要加入空闲列表的都需要进行`notifyAll`操作，来通知那些等待的线程来抢这个归还的连接，但是如果此时连接池中空闲连接充足，并没有线程等待，这个操作也就浪费了，所以可以思考前面`popConnection`中的`wait`和这里的`notifyAll`是可以用等待队列来完成。

另外一个方法，用于判断连接是否可用：

```java
 protected boolean pingConnection(PooledConnection conn) {
    boolean result = true;

    try {
      // 先用isClosed来获取结果
      result = !conn.getRealConnection().isClosed();
    } catch (SQLException e) {
      if (log.isDebugEnabled()) {
        log.debug("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
      }
      result = false;
    }

    if (result) {
      // 可以通过poolPingEnabled配置来决定是否使用自定义sql
      if (poolPingEnabled) {
        if (poolPingConnectionsNotUsedFor >= 0 && conn.getTimeElapsedSinceLastUse() > poolPingConnectionsNotUsedFor) {
          try {
            if (log.isDebugEnabled()) {
              log.debug("Testing connection " + conn.getRealHashCode() + " ...");
            }
            Connection realConn = conn.getRealConnection();
            Statement statement = realConn.createStatement();
            // 执行poolPingQuery
            ResultSet rs = statement.executeQuery(poolPingQuery);
            rs.close();
            statement.close();
            if (!realConn.getAutoCommit()) {
              realConn.rollback();
            }
            result = true;
            if (log.isDebugEnabled()) {
              log.debug("Connection " + conn.getRealHashCode() + " is GOOD!");
            }
          } catch (Exception e) {
            log.warn("Execution of ping query '" + poolPingQuery + "' failed: " + e.getMessage());
            try {
              conn.getRealConnection().close();
            } catch (Exception e2) {
              //ignore
            }
            result = false;
            if (log.isDebugEnabled()) {
              log.debug("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
            }
          }
        }
      }
    }
    return result;
  }
```
从代码中可以看到`isClosed`方法并不可靠，最终还是通过执行sql来判断连接是否可用，这个在很多涉及判断数据库连接是否有效的地方都是这么做的，详细可以看一下`isClosed`方法的注释。


##### PooledDataSourceFactory
继承UnpooledDataSourceFactory，直接返回PooledDataSource对象
```java
public class PooledDataSourceFactory extends UnpooledDataSourceFactory {

  public PooledDataSourceFactory() {
    this.dataSource = new PooledDataSource();
  }

}
```

##### 心得
在整个线程池的实现代码中，可以学习到一个池的实现的要素有哪些，以及录用基础代码如何实现一个池。对于那些封装成高层次的池的代码来说，这个实现显得又些单薄和不够全面，可是无论连接池如何实现核心池的实现逻辑是不会变的。
