# 十.annotations注解包
Mybatis使用注解的方式可以减少使用xml配置sql，方便用断点的形式检测生成的sql，代码的可读性更强，更利于维护。
## 10.1Param注解
用于定义接口入参的别名，方便代码运行时获取接口入参，而不至于读取参数表中的参数时因为参数类型相同而导致参数获取紊乱。
用法：
``` java
List<T> selectByIdAndName (@Param("id") Integer id, @Param("name") String name);
```
## 10.2Select注解
给指定的mapper接口增加sql语句实现查询功能，使用方法为：
``` java
@Select("select * from users")
List<User> getAllUsers();
```
## 10.3SelectProvider注解
方便动态生成sql语句，可以在注解中声明自定义生成sql语句的方法和类的type：
``` java
   //声明自定义的Provider的type,以及自定义Provider中的方法名
  @SelectProvider(type = StatementProvider.class, method = "provideSelect")
  S select(S param);
//自定义的sql生成方法
public String provideSelect(Object param) {
      StringBuilder sql = new StringBuilder("select * from ");
      if (param == null || param instanceof Person) {
        sql.append(" person ");
        if (param != null && ((Person) param).getId() != null) {
          sql.append(" where id = #{id}");
        }
      } else if (param instanceof Country) {
        sql.append(" country ");
        if (((Country) param).getId() != null) {
          sql.append(" where id = #{id}");
        }
      }
      sql.append(" order by id");
      return sql.toString();
    }
```
## 10.4SelectKey注解
用于在sql语句执行前（后）插入执行，如在执行前（后）查询当前自增张id的值，注解中声明的属性为：
``` java
   //sql子句
  String[] statement();
  //属性名，将查询得到列中的数据赋值属性名对应的属性
  String keyProperty();
  //查询返回的列名
  String keyColumn() default "";
  //表示在是否在主sql执行前执行
  boolean before();
  //返回的值的类型
  Class<?> resultType();
  //状态
  StatementType statementType() default StatementType.PREPARED;
```
具体使用方式为：
（1）sql语句执行后
``` java
@Update("update table2 set name = #{name} where id = #{nameId}")
    //在更新语句后执行，查询nameId对应的name_fred，并赋值给Name类的generatedName属性（入参name的generatedName属性）
    @SelectKey(statement="select name_fred from table2 where id = #{nameId}", 
                keyProperty="generatedName", keyColumn="NAME_FRED", before=false, resultType=String.class)
int updateTable2WithSelectKeyWithKeyMap(Name name);
```
（2）sql语句执行前
``` java
@Insert("insert into table3 (id, name) values(#{nameId}, #{name})")
    //在插入语句执行之前，查询数据库表中下一个id的值，并赋值给Name对象name的nameId属性
    @SelectKey(statement="call next value for TestSequence", keyProperty="nameId", before=true, resultType=int.class)
int insertTable3(Name name);
```
## 10.5 Insert、InsertProvider注解
使用方法与Select相似。
## 10.6 Update、UpdateProvider注解 
使用方法和Select相似。
## 10.7 Delete、DeleteProvider注解
使用方法和Select相似。
## 10.8 Mapper注解
使用该注解时只需要在对应的mapper的接口上添加注解@Mapper即可让spring加载这个Bean，且根据注解定义的sql语句实现功能。不过这个注解的加载不是在mybatis的主jar包中，而是在这个jar中找到了这个方法：
``` java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    logger.debug("Searching for mappers annotated with @Mapper");
    //mapper类路径扫描器，这个类在mybatis-spring中，继承自spring框架下的ClassPathBeanDefinitionScanner类。
    //说明如果不引入mybatis-spring-boot-autoconfigure这个jar也可以用这个类加载mapper
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    try {
      if (this.resourceLoader != null) {
        //设置资源加载器
        scanner.setResourceLoader(this.resourceLoader);
      }
      //从工厂中知道哪些包路径是需要自动扫描的
      List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
      if (logger.isDebugEnabled()) {
        for (String pkg : packages) {
          logger.debug("Using auto-configuration base package '{}'", pkg);
        }
      }
      //设置需要扫描的注解类型
      scanner.setAnnotationClass(Mapper.class);
      //注册过滤器
      scanner.registerFilters();
      //开始扫描
      scanner.doScan(StringUtils.toStringArray(packages));
    } catch (IllegalStateException ex) {
      logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.", ex);
    }
  } 
```
## 10.9 AutomapConstructor注解
在与数据库查询结果对应的model类的构造器上增加这个注释，可以让框架自动使用这个构造器。
## 10.11 ResultType注解
用于注明结果类型，使用方式为：
``` java
  @Select("select * from users")
  @ResultType(User.class)
  //虽然方法的返回结果为void，但是入参resultHandler可以接受返回结果
  void getAllUsers(UserResultHandler resultHandler);
其中UserResultHandler为：
public class UserResultHandler implements ResultHandler {
//返回结果
  private List<User> users;
  
  public UserResultHandler() {
    super();
    users = new ArrayList<User>();
  }

  @Override
  public void handleResult(ResultContext context) {
    User user = (User) context.getResultObject();
    users.add(user);
  }

  public List<User> getUsers() {
    return users;
  }
}
```
## 10.12One、Many注解
用于注明查询结果是1对1（1对多）的关系，fetchType属性可以配置为懒加载或者立即加载：
``` java
  @Results({ 
      @Result(property = "author", column = "author_id", one = @One(select = "org.apache.ibatis.binding.BoundAuthorMapper.selectAuthor", fetchType=FetchType.EAGER)), 
      @Result(property = "posts", column = "id", many = @Many(select = "selectPostsById", fetchType=FetchType.EAGER))
  })
  List<Blog> selectBlogsWithAutorAndPostsEagerly();
```
## 10.13Lang注解
该注解用于标注使用的自定义语言脚本驱动（自定义实现自LanguageDriver接口的类），方便于解析sql语言，使用时只需要在对应方法上增加注解，如：
@Lang(XMLLanguageDriver.class)
## 10.14Flush注解
使用这个注解的方式应该是在对应的方法上增加注解@Flush，然后调用方法时会将缓存中的操作批量刷新到数据库中，因为官方给的文档不够详细，本人也没有实践过，如果存在理解错误，请谅解。
## 10.13Results、ResultMap、Result注解
这几个注解用于查询结果的映射，用于将数据库中的字段名和java对象中的字段名对应上，不过目前也可以使用数据库提供的别名定义的方式实现名称映射，同时也可以使用自定义typeHandle处理
### 10.13.1 ResultMap应用
``` java
  //在这个结果集中定义映射名叫userResult，方便别的地方去使用
  @Results(id = "userResult", value = {
          //定义uid到id的映射
    @Result(id = true, column = "uid", property = "id"),
    @Result(column = "name", property = "name")
  })
  @Select("select * from users where uid = #{id}")
  User getUserById(Integer id);
  //在这里通过复用之前定义好的id=userResult的映射完成结果集映射
  @ResultMap("userResult")
  @Select("select * from users where name = #{name}")
  User getUserByName(String name);
```
### 10.13.2Result说明
``` java
  //标注是不是id，一般id会被定义成数据库中的序列化主键
  boolean id() default false;
  //数据库查询结果中的列名
  String column() default "";
  //需要映射的对象中的属性名
  String property() default "";
  //java的类型
  Class<?> javaType() default void.class;
  //数据库中的类型
  JdbcType jdbcType() default JdbcType.UNDEFINED;
  //自定义类型处理器
  Class<? extends TypeHandler<?>> typeHandler() default UnknownTypeHandler.class;
  //标注为单条结果映射
  One one() default @One;
  //表示为多条结果映射
  Many many() default @Many;
```
### 10.13.3Results应用
具体使用时，在需要定义结果映射的方法前增加注释，就可以将查询到的结果按照定义好的结果映射样式完成映射。
``` java
 //这里的注解中的参数只是为了展示方便定义的，目的是为了展示注解的使用方法,可能存在错误
    @Select("select * from post where author_id = #{id} order by id")
    @Results(id = "test",
            value = {@Result(property = "id", column = "id",javaType = int.class,jdbcType = JdbcType.INTEGER),
                    @Result(property = "subject", column = "subject"),
                    @Result(property = "body", column = "body"),
                    @Result(property = "tags", javaType = List.class, column = "id", typeHandler = ArrayTypeHandler.class,
                            many = @Many(select = "getTagsForPost"))
            })
    List<AnnoPost> getPosts(int authorId);
```
## 10.14Options
这个注解用于配置查询结果配置，内部具有如下属性：
``` java
//是否使用缓存
  boolean useCache() default true;
  //是否清除缓存
  FlushCachePolicy flushCache() default FlushCachePolicy.DEFAULT;
  //结果集的类型
  ResultSetType resultSetType() default ResultSetType.FORWARD_ONLY;
  //执行语句生成方式类型
  StatementType statementType() default StatementType.PREPARED;
  //返回结果条目数
  int fetchSize() default -1;
  //查询超时时间
  int timeout() default -1;
  //是否使用自增长key
  boolean useGeneratedKeys() default false;
  //key在结果对象中的属性名
  String keyProperty() default "";
  //key在数据库中的列名
  String keyColumn() default "";
  //不清楚怎么用
  String resultSets() default "";
常见的使用方式为返回插入数据对应的自增长序列id：
  @Options(useGeneratedKeys = true, keyProperty = "id ")
  @Insert({ "insert into planet (name) values (#{name})" })
//注意，不是把id作为方法的返回值，而是将id赋值给plate对象的id属性。
  int insertPlanet(Planet planet);
```
## 10.15MapKey
   标注返回结果的map的键值为对象中的哪个字段，如Person中存在一个字段名为id的属性，MapKey的value设置为id，那么返回的map中key的值就是id的值，如：
``` java
@Select("select * from person")
  @MapKey("id")
  Map<Integer, Person> selectAsMap();
```
## 10.16 CacheNamespace、Property、CacheNamespaceRef注解
这个应该是用于命名空间中的缓存读取，具体使用方法为在类名上加CacheNamespace注解，然后通过Property设置值：
``` java
@CacheNamespace(implementation = CustomCache.class, properties = {
//用取值符号取值，用于赋值
    @Property(name = "stringValue", value = "${stringProperty}"),
    @Property(name = "integerValue", value = "${integerProperty}"),
    @Property(name = "longValue", value = "${longProperty}")
})
```
在类名上增加注解CacheNamespaceRef，引用别的命名空间缓存，有两种方式：
（1）通过类名路径
``` java
@CacheNamespaceRef(name = "org.apache.ibatis.submitted.cache.PersonMapper")
```
（2）通过类的type
``` java
@CacheNamespaceRef(PersonMapper.class)
```
## 10.16 Arg和ConstructorArgs注解
这两个注解一般都结合在一起使用，声明查询结果对应的Bean使用的构造函数和入参。
## 10.17.1Arg注解
    注解中的具体属性如下：
``` java
//标注是否是id
  boolean id() default false;
  //列名
  String column() default "";
  //java的数据类型
  Class<?> javaType() default void.class;
  //jdbc中的数据类型
  JdbcType jdbcType() default JdbcType.UNDEFINED;
  //数据类型处理器
  Class<? extends TypeHandler> typeHandler() default UnknownTypeHandler.class;
  //select子句，可以复用别的sql方法获取结果，赋值给当前查询
  //感觉这种复合查询定位问题时可能不怎么方便
  String select() default "";
  //结果类型，可以填写别的ResultMap的id的值，复用ResultMap
  String resultMap() default "";
  //构造函数中@Param注解中的value值
  String name() default "";
  //标注列前缀，在3.5.0版本中才引入的
  String columnPrefix() default "";
```
### 10.17.2ConstructorArgs注解
这个注解结合Arg注解用于标注使用指定的构造函数，如：
（1）简单的指定
``` java
@ConstructorArgs({
      @Arg(column = "name", name = "name"),
      @Arg(id = true, column = "id", name = "id"),
      @Arg(column = "team", name = "team", javaType = String.class),
  })
  @Select("select * from users where id = #{id}")
  User mapConstructorWithParamAnnos(Integer id);
```
（2）复杂嵌套，配置子查询，感觉类似于关联查询吧。
``` java
@Select("SELECT * FROM " +
      "blog WHERE id = #{id}")
  @ConstructorArgs({
      @Arg(column = "id", javaType = int.class, id = true),
      @Arg(column = "title", javaType = String.class),
//这里指定了一个查询方法，且指定查询结果中的author_id作为子查询的入参
      @Arg(column = "author_id", javaType = Author.class, select = "org.apache.ibatis.binding.BoundAuthorMapper.selectAuthor"),
      @Arg(column = "id", javaType = List.class, select = "selectPostsForBlog")
  })
  Blog selectBlogUsingConstructor(int id);
```
## 10.18 TypeDiscriminator和Case注解
类型鉴定器主要用于查询结果判断与映射
### 10.18.1 Case中的属性
``` java
//查询结果中的值
  String value();
  //java中的对象类型
  Class<?> type();
  //结果映射
  Result[] results() default {};
  //待映射对象的构造器的构造入参
  Arg[] constructArgs() default {};
```
### 10.18.2 TypeDiscriminator中的属性
``` java
//列名
  String column();
  //java类型
  Class<?> javaType() default void.class;
  //jdbc中的类型
  JdbcType jdbcType() default JdbcType.UNDEFINED;
  //类型处理器
  Class<? extends TypeHandler> typeHandler() default UnknownTypeHandler.class;
  //结果情景处理
  Case[] cases();
```
### 10.18.3 使用方式
``` java
    @TypeDiscriminator(
            // target for test (ordinal number -> Enum constant)
            column = "personType", javaType = PersonType.class, typeHandler = EnumOrdinalTypeHandler.class,
            // Switch using enum constant name(PERSON or EMPLOYEE) at cases attribute
            cases = {
             @Case(value = "PERSON", type = Person.class,  
             results = {@Result(property = "personType",  column = "personType", typeHandler = EnumOrdinalTypeHandler.class)}) ,
             @Case(value = "EMPLOYEE", type = Employee.class, 
             results = { @Result(property = "personType", column = "personType", typeHandler = EnumOrdinalTypeHandler.class)})
            })
    @Select("SELECT id, firstName, lastName, personType FROM person WHERE id = #{id}")
    Person findOneUsingTypeDiscriminator(int id);
```
原创文章转载请标明出处
更多文章请查看 
[http://www.canfeng.xyz](http://www.canfeng.xyz)