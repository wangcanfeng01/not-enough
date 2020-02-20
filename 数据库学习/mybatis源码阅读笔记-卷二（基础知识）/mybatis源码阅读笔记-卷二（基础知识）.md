# 二.io包
## 2.1ClassLoaderWrapper
类加载器包装器内部含有两个属性：ClassLoader defaultClassLoader;和ClassLoader systemClassLoader;，它的构造函数没有访问修饰，因此只能在同一个包中进行访问，构造函数的作用只是获取系统的类加载器：systemClassLoader = ClassLoader.getSystemClassLoader()。内部有两个核心的方法，从五个类加载器中获取stream和获取url，这5个类加载器在后续类加载器的学习中进行展开。
（1）
``` java
        //通过类加载器中的方法获取流
        InputStream returnValue = cl.getResourceAsStream(resource);
        //给路径前面加上“/”重新获取流
        if (null == returnValue) {
          returnValue = cl.getResourceAsStream("/" + resource);
        }
```
（2）
``` java
  //通过类加载器中的方法获取资源url
        url = cl.getResource(resource);
        //加上反斜杠重新获取资源url
        if (null == url) {
          url = cl.getResource("/" + resource);
        }
```
## 2.2DefaultVFS
VFS是虚拟文件系统的缩写，Class文件有有魔数检测是不是java的class，jar包上面有JAR_MAGIC = { 'P', 'K', 3, 4 };用于区分它是不是jar。它通过集成了VFS这个抽象类实现的，不是mybatis学习侧重点，跳过了。
## 2.3ResolverUtil
其内部具有一个核心方法：
``` java
//主要的方法，找一个package下满足条件的所有类,被TypeHanderRegistry,MapperRegistry,TypeAliasRegistry调用
  public ResolverUtil<T> find(Test test, String packageName) {
    String path = getPackagePath(packageName);
    try {
        //通过VFS来深入jar包里面去找一个class
      List<String> children = VFS.getInstance().list(path);
      for (String child : children) {
        if (child.endsWith(".class")) {
//将符合条件的child放入到类内的hashSet中。
          addIfMatching(test, child);
        }
      }
    } catch (IOException ioe) {
      log.error("Could not read package: " + packageName, ioe);
    }
    return this;
  }
```
# 三.jdbc包
这个包里面存在一个核心类SQL，用于动态生成sql语句。
## 3.1AbstractSQL
这里需要先科普一下{{}}的使用方式，在实际使用SQL类初始化时，可以使用new SQL{{AbstractSQL 中的方法}}的方式来初始化一个SQL，实际上里面的{}中的代码就相当于是将这些操作合并到构造函数中，可以算是一种语法糖。
该类内有一个核心方法sql()，它的返回类型是一个静态内部类SQLStatement，它定义了四种数据库操作类型DELETE, INSERT, SELECT, UPDATE，还有一堆list，这些list用于存放各个关键字后的数据信息，以及它们各自的sql语句生成，最后都是通过调用sqlClause实现的：
``` java
  //利用字符拼接器拼接sql语句
        private void sqlClause(SafeAppendable builder, String keyword, 
                   List<String> parts, String open, String close, String conjunction) {
            if (!parts.isEmpty()) {
                if (!builder.isEmpty()) {
                    //如果前面有东西，另起一行
                    builder.append("\n");
                }
                //每行起始都是关键词，如Select，Update等
                builder.append(keyword);
                //关键词和内容之间需要空格隔开
                builder.append(" ");
                builder.append(open);
                String last = "________";
                //parts中的内容一般为字段名或字段名对应的值，
                // 各个字段之间就需要用逗号隔开，如果是连接符，就不需要隔开
                for (int i = 0, n = parts.size(); i < n; i++) {
                    String part = parts.get(i);
                    if (i > 0 && !part.equals(AND) && !part.equals(OR) && !last.equals(AND) && !last.equals(OR)) {
                        builder.append(conjunction);
                    }
                    builder.append(part);
                    //为了下一次判断存的临时值
                    last = part;
                }
                builder.append(close);
            }
        }
```
## 3.2Null
SQL中常用类型的枚举，每个枚举包含了一个Handler和一个枚举名对应的Jdbc类型。
## 3.3ScriptRunner
脚本执行器，基本只在单元测试中使用的类
## 3.6SqlRunner
SQL执行器，基本只在单元测试中使用的类
# 四.datasource包
这个包里面存在三种数据源类型：jndi、pooled、uopooled，每种数据源类型对应着各自的工厂。JNDI：Java Naming and Directory Interface，即java命名和目录接口，JNDI包含了一些标准API接口，java程序可以通过这些接口来访问命名目录服务，命名服务就是将名字和计算机系统内的一个对象建立关联，从而允许应用程序通过名字访问对象；目录服务是命名服务的拓展 ，目录服务不仅需要保存名称和对象的关联，还要保存对象的各种属性，这样就允许开发者操作对象的属性，包括增删改查。
## 4.1DataSourceFactory
拥有两个方法：setProperties：设置属性和getDataSource获取数据源。
## 4.2JndiDataSourceFactory
实现了DataSourceFactory接口，实现了设置属性和获取数据源方法，它没有构造函数，DataSource是通过上下文容器中获取的，其中核心方法为：
``` java
  public void setProperties(Properties properties) {
    try {
      InitialContext initCtx = null;
      //从配置中获取env开头的配置信息
      Properties env = getEnvProperties(properties);
      if (env == null) {
        initCtx = new InitialContext();
      } else {
        initCtx = new InitialContext(env);
      }

      if (properties.containsKey(INITIAL_CONTEXT)
          && properties.containsKey(DATA_SOURCE)) {
        //获取初始化上下文信息
        Context ctx = (Context) initCtx.lookup(properties.getProperty(INITIAL_CONTEXT));
        //获取数据源
        dataSource = (DataSource) ctx.lookup(properties.getProperty(DATA_SOURCE));
      } else if (properties.containsKey(DATA_SOURCE)) {
        //如果没有初始化上下文信息，就直接获取数据源
        dataSource = (DataSource) initCtx.lookup(properties.getProperty(DATA_SOURCE));
      }
    } catch (NamingException e) {
      throw new DataSourceException("There was an error configuring JndiDataSourceTransactionPool. Cause: " + e, e);
    }
  }
```
## 4.3 unpooled
### 4.3.1UnpooledDataSource非池化的数据源 
它内部包含了用户名、url、密码、驱动、驱动配置属性、是否自动提交以及事务隔离属性等。
内部获取Connection的流程：（1）初始化启动实例（内部静态类，驱动代理的方式），（2）DriverManager注册驱动，注册到存放驱动的静态map中；（3）DriverManager通过连接配置获取连接，（4）根据构造器中的参数配置属性是否自动提交以及事务隔离级别
### 4.3.2 UnpooledDataSourceFactory
在构造函数中初始化一个非池化数据源，它的核心方法为setProperties，与上面jndi的工厂不一样的是，它的setProperties方法主要是从入参中获取配置信息，然后再将各类配置信息通过MetaObject类注入到datasource中。关于MetaObject将在reflection包中展开学习。
## 4.4pooled
### 4.4.1 PooledConnection池化连接
1.只拥有一个构造函数，通过传入connection和datasource初始化pooledConnection
2.重写了equals方法， 用hashcode去判断连接是否相同。
3.重新了invoke方法，当使用invoke调用实际方法时，需要先检查connection是否有效。
### 4.4.2 PooledDataSource池化数据源
该类中具有三个重要的方法：pushConnection插入连接；popConnection获取连接；pingConnection判断连接是否可用。
``` java
protected void pushConnection(PooledConnection conn) throws SQLException {
      //给state加上同步锁
    synchronized (state) {
      //先从activeConnections中删除此connection
      state.activeConnections.remove(conn);
      if (conn.isValid()) {
        if (state.idleConnections.size() < poolMaximumIdleConnections 
              && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
      	  //如果空闲的连接太少，
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          //new一个新的Connection，加入到idle列表
          PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
          state.idleConnections.add(newConn);
          newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
          newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
          conn.invalidate();
          if (log.isDebugEnabled()) {
            log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
          }
          //通知其他线程可以来抢connection了
          state.notifyAll();
        } else {
        	//否则，即空闲的连接已经足够了
          state.accumulatedCheckoutTime += conn.getCheckoutTime();
          if (!conn.getRealConnection().getAutoCommit()) {
            conn.getRealConnection().rollback();
          }
          //那就将connection关闭就可以了
          conn.getRealConnection().close();
          if (log.isDebugEnabled()) {
            log.debug("Closed connection " + conn.getRealHashCode() + ".");
          }
          //并设置为无效
          conn.invalidate();
        }
      } else {
        if (log.isDebugEnabled()) {
          log.debug("A bad connection (" + conn.getRealHashCode() 
                   + ") attempted to return to the pool, discarding connection.");
        }
        state.badConnectionCount++;
      }
    }
  }

private PooledConnection popConnection(String username, String password) throws SQLException {
    boolean countedWait = false;
    PooledConnection conn = null;
    long t = System.currentTimeMillis();
    int localBadConnectionCount = 0;
    //最外面是while死循环，如果一直拿不到connection，则不断尝试
    while (conn == null) {
      synchronized (state) {
        if (!state.idleConnections.isEmpty()) {
          //如果有空闲的连接的话
          // Pool has available connection
          //删除空闲列表里第一个，返回
          conn = state.idleConnections.remove(0);
          if (log.isDebugEnabled()) {
            log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
          }
        } else {
        	//如果没有空闲的连接
          // Pool does not have available connection
          if (state.activeConnections.size() < poolMaximumActiveConnections) {
        	  //如果activeConnections太少,那就new一个PooledConnection
            // Can create new connection
            conn = new PooledConnection(dataSource.getConnection(), this);
            if (log.isDebugEnabled()) {
              log.debug("Created connection " + conn.getRealHashCode() + ".");
            }
          } else {
        	  //如果activeConnections已经很多了，那不能再new了
            // Cannot create new connection
        	  //取得activeConnections列表的第一个（最老的）
            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
            if (longestCheckoutTime > poolMaximumCheckoutTime) {
            	//如果checkout时间过长，则这个connection标记为overdue（过期）
              state.claimedOverdueConnectionCount++;
              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
              state.accumulatedCheckoutTime += longestCheckoutTime;
              //删掉最老的连接
              state.activeConnections.remove(oldestActiveConnection);
              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
                oldestActiveConnection.getRealConnection().rollback();
              }
              //然后再new一个新连接
              conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
              oldestActiveConnection.invalidate();
              if (log.isDebugEnabled()) {
                log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
              }
            } else {
            	//如果checkout时间不够长，等待吧
              // Must wait
              try {
                if (!countedWait) {
                	//统计信息：等待+1
                  state.hadToWaitCount++;
                  countedWait = true;
                }
                if (log.isDebugEnabled()) {
                  log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
                }
                long wt = System.currentTimeMillis();
                //睡一会儿吧
                state.wait(poolTimeToWait);
                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
              } catch (InterruptedException e) {
                break;
              }
            }
          }
        }
        if (conn != null) {
        	//如果已经拿到connection，则返回
          if (conn.isValid()) {
            if (!conn.getRealConnection().getAutoCommit()) {
              conn.getRealConnection().rollback();
            }
            conn.setConnectionTypeCode(
                assembleConnectionTypeCode(dataSource.getUrl(), username, password));
            //记录checkout时间
            conn.setCheckoutTimestamp(System.currentTimeMillis());
            conn.setLastUsedTimestamp(System.currentTimeMillis());
            state.activeConnections.add(conn);
            state.requestCount++;
            state.accumulatedRequestTime += System.currentTimeMillis() - t;
          } else {
            if (log.isDebugEnabled()) {
              log.debug("A bad connection (" + conn.getRealHashCode() 
                      + ") was returned from the pool, getting another connection.");
            }
            //如果没拿到，统计信息：坏连接+1
            state.badConnectionCount++;
            localBadConnectionCount++;
            conn = null;
            if (localBadConnectionCount > (poolMaximumIdleConnections + 3)) {
            	//如果好几次都拿不到，就放弃了，抛出异常
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
        log.debug(
"PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
      }
      throw new SQLException("PooledDataSource: Unknown severe error condition."+
                    "The connection pool returned a null connection.");
    }

    return conn;
  }

protected boolean pingConnection(PooledConnection conn) {
    boolean result = true;

    try {
        //判断连接是否已经关闭
      result = !conn.getRealConnection().isClosed();
    } catch (SQLException e) {
      if (log.isDebugEnabled()) {
        log.debug("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
      }
      result = false;
    }

    if (result) {
        //如果允许ping
      if (poolPingEnabled) {
        if (poolPingConnectionsNotUsedFor >= 0 && conn.getTimeElapsedSinceLastUse() > poolPingConnectionsNotUsedFor) {
          try {
            if (log.isDebugEnabled()) {
              log.debug("Testing connection " + conn.getRealHashCode() + " ...");
            }
            //下面的步骤中如果抛出了异常，也会将result设置为false
            Connection realConn = conn.getRealConnection();
            Statement statement = realConn.createStatement();
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
### 4.4.3 PoolState池状态信息
（1）内部有两个集合，分别存放空闲连接和活跃连接
（2）requestCount记录请求次数和accumulatedRequestTime总的请求时间，方便计算平均请求时间
（3）checkoutTime调用获取connection的等待时间，如果超过这个时间，它会标记这个连接过期
（4）claimedOverdueConnectionCount声明连接过期的次数
（5）accumulatedCheckoutTimeOfOverdueConnections关于过期的连接的总检查时间
（6）accumulatedWaitTime总的等待时间
（7）hadToWaitCount等待的次数
（8）badConnectionCount遇到坏连接的次数
# 五.transaction包
事务接口包含四个方法：
  Connection getConnection() throws SQLException;
  void commit() throws SQLException;
  void rollback() throws SQLException;
  void close() throws SQLException;
事务工厂分为两种ManagedTransactionFactory和JdbcTransactionFactory。
## 5.1 jdbc
JdbcTransactionFactory如果是从DataSource中获取的连接，还可以设计是否自动提交与事务等级。
（1）commit提交
``` java
public void commit() throws SQLException {
    //可以看到它只有在不是自动提交的情况下才会自动触发commit
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Committing JDBC Connection [" + connection + "]");
      }
      connection.commit();
    }
  }
```
（2）rollback回退
``` java
  public void rollback() throws SQLException {
    //依然可以看到它只有在不是自动提交设置情况下才触发rollback
    if (connection != null && !connection.getAutoCommit()) {
      if (log.isDebugEnabled()) {
        log.debug("Rolling back JDBC Connection [" + connection + "]");
      }
      connection.rollback();
    }
```
（3）close关闭连接
``` java
  public void close() throws SQLException {
    if (connection != null) {
      //这个方法会将autoCommit属性设置为true，不是很清楚为什么要在连接关闭前设置这个属性
      //看它的注释貌似是有些db的特性原因
      resetAutoCommit();
      if (log.isDebugEnabled()) {
        log.debug("Closing JDBC Connection [" + connection + "]");
      }
      connection.close();
    }
  }
```
## 5.2 managed
ManagedTransactionFactory默认情况下会关闭连接，可以通过Connection创建ManagedTransaction，也可以通过DataSource、事务等级来设置创建ManagedTransaction。它的commit和rollback方法都是空的；
（1）commit是空的
（2）rollback是空的
（3）close关闭连接
``` java
public void close() throws SQLException {
    //如果properties文件配置了closeConnection=false,则不关闭连接
    if (this.closeConnection && this.connection != null) {
      if (log.isDebugEnabled()) {
        log.debug("Closing JDBC Connection [" + this.connection + "]");
      }
      this.connection.close();
    }
  }
```
# 六.type包
## 6.1 TypeReference
``` java
Type getSuperclassTypeParameter(Class<?> clazz) {
    //获取入参类型的父类类型(包括带泛型的)
    // 如果要获取不带泛型的父类，可以使用getSuperclass()
    Type genericSuperclass = clazz.getGenericSuperclass();
	//判断获取的类型是否是Class类的实例，是否为不带有参数类型，即不带泛型
    //Class类的实例有哪些呢？
    //其实每一个类都是Class类的实例，Class类是对Java中类的抽象，它本身也是一个类，但它是出于其他类上一层次的类，是类的顶层抽象。
    if (genericSuperclass instanceof Class) {
      // try to climb up the hierarchy until meet something useful
      if (TypeReference.class != genericSuperclass) {
        return getSuperclassTypeParameter(clazz.getSuperclass());
      }

      throw new TypeException("'" + getClass() + "' extends TypeReference but misses the type parameter. "
        + "Remove the extension or add a type parameter to it.");
    }
    //如果是带泛型的类，则获取所带泛型参数类型
    Type rawType = ((ParameterizedType) genericSuperclass).getActualTypeArguments()[0];
    // TODO remove this when Reflector is fixed to return Types
    if (rawType instanceof ParameterizedType) {
      rawType = ((ParameterizedType) rawType).getRawType();
    }
    return rawType;
  }


  //这个方式重点被调用的地方在TypeHandlerRegistry（类型处理器注册器）中，在没有指定JavaType而只有TypeHandler的情况下，
  // 调用该TypeHandler的getRawType()方法来获取其原生类型（即参数类型）来作为其JavaType来进行类型处理器的注册。
  public final Type getRawType() {
    return rawType;
  }
```
## 6.2TypeHandler类型处理器
拥有四个方法：
（1）设置参数
  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
（2）取得结果,供普通select用
T getResult(ResultSet rs, String columnName) throws SQLException;
（3）取得结果,供普通select用
T getResult(ResultSet rs, int columnIndex) throws SQLException;
（4）取得结果,供SP用
T getResult(CallableStatement cs, int columnIndex) throws SQLException;
## 6.3 BaseTypeHandler类型处理器的基类
它在内部实现了TypeHandler的四个方法，但也不能说是实现，它还有4个抽象方法，需要具体的Handler类中去继承实现。
## 6.4 TypeAliasRegistry类型别名注册机
（1）解析类型别名
``` java
//解析类型别名
  public <T> Class<T> resolveAlias(String string) {
    try {
      if (string == null) {
        return null;
      }
      // issue #748
      //先转成小写再解析
      String key = string.toLowerCase(Locale.ENGLISH);
      Class<T> value;
      //原理就很简单了，从HashMap里找对应的键值，找到则返回类型别名对应的Class
      if (TYPE_ALIASES.containsKey(key)) {
        value = (Class<T>) TYPE_ALIASES.get(key);
      } else {
        //找不到，再试着将String直接转成Class(这样怪不得我们也可以直接用java.lang.Integer的方式定义，也可以就int这么定义)
        value = (Class<T>) Resources.classForName(string);
      }
      return value;
    } catch (ClassNotFoundException e) {
      throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
    }
  }
```
（2）注册类型别名
``` java
public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // issue #748
    String key = alias.toLowerCase(Locale.ENGLISH);
    //如果已经存在key了，且value和之前不一致，报错
    if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
    }
    TYPE_ALIASES.put(key, value);
  }
```
## 6.5 TypeHandlerRegistry类型处理器注册机
类型注册机的处理流程为
（1）扫描包
（2）获取类上标注的的MappedTypes，从中知道它需要转为成什么样的java中的类型，下面为部分源码
``` java
public void register(Class<?> typeHandlerClass) {
    boolean mappedTypeFound = false;
    //获取类上面的注释
    MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
      for (Class<?> javaTypeClass : mappedTypes.value()) {
        //获取注解中标注的被转换后的java类型，并注册
        register(javaTypeClass, typeHandlerClass);
        mappedTypeFound = true;
      }
    }
    if (!mappedTypeFound) {
      register(getInstance(null, typeHandlerClass));
    }
  }
```
（3）获取类上标注的MappedJdbcTypes，然后注册映射类型，即将什么jdbc类型转成什么java类型。
``` java
  private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
	//获取类上标注的jdbc的类型，方便将数据库中的类型转成java中的类型，避免数据类型不正确导致的类型不匹配
    //同时可以设置自定义类型转换器，更方便数据库操作
    MappedJdbcTypes mappedJdbcTypes = typeHandler.getClass().getAnnotation(MappedJdbcTypes.class);
    if (mappedJdbcTypes != null) {
      for (JdbcType handledJdbcType : mappedJdbcTypes.value()) {
        register(javaType, handledJdbcType, typeHandler);
      }
      if (mappedJdbcTypes.includeNullJdbcType()) {
        register(javaType, null, typeHandler);
      }
    } else {
      register(javaType, null, typeHandler);
    }
  }
```
在实际项目中可以通过实现TypeHandler接口，并在实现类上标注。如下面截断String类型的TypeHandler。
``` java
@MappedTypes(String.class)
@MappedJdbcTypes(value={JdbcType.CHAR,JdbcType.VARCHAR}, includeNullJdbcType=true)
public class StringTrimmingTypeHandler implements TypeHandler<String> {

  @Override
  public void setParameter(PreparedStatement ps, int i, String parameter,
      JdbcType jdbcType) throws SQLException {
    ps.setString(i, trim(parameter));
  }

  @Override
//如果是通过列名获取的数据，需要实现这个方法
  public String getResult(ResultSet rs, String columnName) throws SQLException {
    return trim(rs.getString(columnName));
  }

  @Override
//如果是通过列序号获取的数据，需要实现这个方法
  public String getResult(ResultSet rs, int columnIndex) throws SQLException {
    return trim(rs.getString(columnIndex));
  }

  @Override
  public String getResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    return trim(cs.getString(columnIndex));
  }

  private String trim(String s) {
    if (s == null) {
      return null;
    } else {
      return s.trim();
    }
  }
}
```
# 七.logging包
## 7.1mybatis支持的日志框架
jdbc、jdk14、log4j、log4j2、slf4j、stdout(这个是在命令行输出的)
## 7.2LogFactory日志工厂
（1）用静态块初始化工厂类，实际上就是挑一个日志框架的构造函数，给logConstructor赋值。
``` java
static {
    //静态块里面貌似开了多个线程，去调用底层方法，给logConstructor赋值
    //这个tryImplementation调用的是Runnable中的run方法，所以不是多线程
    //slf4j
    tryImplementation(new Runnable() {
      @Override
      public void run() {
        useSlf4jLogging();
      }
    });
    //common logging
    tryImplementation(new Runnable() {
      @Override
      public void run() {
        useCommonsLogging();
      }
    });
  ……………………….
  }
```
（2）获取构造函数的实际方法
``` java
  private static void setImplementation(Class<? extends Log> implClass) {
    try {
      Constructor<? extends Log> candidate = implClass.getConstructor(new Class[] { String.class });
      //
      Log log = candidate.newInstance(new Object[] { LogFactory.class.getName() });
      log.debug("Logging initialized using '" + implClass + "' adapter.");
      //找到构造器之后赋值给logConstructor
      logConstructor = candidate;
    } catch (Throwable t) {
      throw new LogException("Error setting Log implementation.  Cause: " + t, t);
    }
  }
```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)