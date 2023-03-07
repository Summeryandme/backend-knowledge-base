# 开发一个简易的 DB 分库分表路由中间件
## 1 定义路由注解
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface DBRouter {
    String key() default "";
}

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface DBRouterStrategy {
	// 是否分表
    boolean splitTable() default false;
}

```

- 此注解放在需要被数据库路由的方法上
- 被自定义的 AOP 逻辑进行拦截，拦截后进行相应的数据库路由计算和判断，并切换到相应的数据源
## 2 解析路由配置文件
约定路由配置文件如下：
```yaml
db-router:
  jdbc:
    datasource:
      dbCount: 2 # 库的数量
      tbCount: 4 # 表的数量
      routerKey: uId # 根据此键进行路由
      default: db00
      list: db01,db02
      db00:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/default_db
        username: root
        password: password
      db01:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/route_db1
        username: root
        password: password
      db02:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/route_db2
        username: root
        password: password
```

- 对于类似的自定义较大的信息配置，可以使用`org.springframework.context.EnvironmentAware`接口进行信息的提取
```java
public class DataSourceAutoConfig implements EnvironmentAware {

  /** 分库数量 */
  private int dbCount;
  /** 分表数量 */
  private int tbCount;
  /** 路由字段 */
  private String routerKey;
  /** 数据源配置组 */
  private final Map<String, Map<String, Object>> dataSourceMap = new HashMap<>();
  /** 默认数据源配置 */
  private Map<String, Object> defaultDataSourceConfig;

  @Override
  public void setEnvironment(Environment environment) {
    String prefix = "mini-db-router.jdbc.datasource.";
    dbCount = Integer.parseInt(Objects.requireNonNull(environment.getProperty(prefix + "dbCount")));
    tbCount = Integer.parseInt(Objects.requireNonNull(environment.getProperty(prefix + "tbCount")));
    routerKey = environment.getProperty(prefix + "routerKey");

    // 分库分表数据源
    String dataSources = environment.getProperty(prefix + "list");
    assert dataSources != null;
    for (String dbInfo : dataSources.split(",")) {
      Map<String, Object> dataSourceProps =
          PropertyUtil.handle(environment, prefix + dbInfo, Map.class);
      dataSourceMap.put(dbInfo, dataSourceProps);
    }

    // 默认数据源
    String defaultData = environment.getProperty(prefix + "default");
    defaultDataSourceConfig = PropertyUtil.handle(environment, prefix + defaultData, Map.class);
  }
}
```
## 3 数据源切换
在结合 SpringBoot 开发的 Starter 中，需要提供一个 DataSource 的实例化对象，那么这个对象可以就放在 DataSourceAutoConfig 来实现，并且这里提供的数据源是可以动态变换的，也就是支持动态切换数据源。
```java
@Bean
public DataSource dataSource() {
    // 创建数据源
    Map<Object, Object> targetDataSources = new HashMap<>();
    for (String dbInfo : dataSourceMap.keySet()) {
        Map<String, Object> objMap = dataSourceMap.get(dbInfo);
        targetDataSources.put(dbInfo, new DriverManagerDataSource(objMap.get("url").toString(), objMap.get("username").toString(), objMap.get("password").toString()));
    }     

    // 设置数据源
    DynamicDataSource dynamicDataSource = new DynamicDataSource();
    dynamicDataSource.setTargetDataSources(targetDataSources);
    dynamicDataSource.setDefaultTargetDataSource(new DriverManagerDataSource(defaultDataSourceConfig.get("url").toString(), defaultDataSourceConfig.get("username").toString(), defaultDataSourceConfig.get("password").toString()));

    return dynamicDataSource;
}
```

- 数据源创建完成后存放到 DynamicDataSource 中，它是一个继承了 AbstractRoutingDataSource 的实现类，这个类里可以存放和读取相应的具体调用的数据源信息。
```java
public class DynamicDataSource extends AbstractRoutingDataSource {
  @Override
  protected Object determineCurrentLookupKey() {
    return "db" + DBContextHolder.getDBKey();
  }
}
```
## 4 切面拦截
在 AOP 的切面拦截中需要完成；数据库路由计算、扰动函数加强散列、计算库表索引、设置到 ThreadLocal 传递数据源，整体案例代码如下：
```java
@Around("aopPoint() && @annotation(dbRouter)")
public Object doRouter(ProceedingJoinPoint jp, DBRouter dbRouter) throws Throwable {
    String dbKey = dbRouter.key();
    if (StringUtils.isBlank(dbKey)) throw new RuntimeException("annotation DBRouter key is null！");

    // 计算路由
    String dbKeyAttr = getAttrValue(dbKey, jp.getArgs());
    int size = dbRouterConfig.getDbCount() * dbRouterConfig.getTbCount();

    // 扰动函数
    int idx = (size - 1) & (dbKeyAttr.hashCode() ^ (dbKeyAttr.hashCode() >>> 16));

    // 库表索引
    int dbIdx = idx / dbRouterConfig.getTbCount() + 1;
    int tbIdx = idx - dbRouterConfig.getTbCount() * (dbIdx - 1);   

    // 设置到 ThreadLocal
    DBContextHolder.setDBKey(String.format("%02d", dbIdx));
    DBContextHolder.setTBKey(String.format("%02d", tbIdx));
    logger.info("数据库路由 method：{} dbIdx：{} tbIdx：{}", getMethod(jp).getName(), dbIdx, tbIdx);
   
    // 返回结果
    try {
        return jp.proceed();
    } finally {
        DBContextHolder.clearDBKey();
        DBContextHolder.clearTBKey();
    }
}

```
## 5 Mybatis 拦截器处理分表

- 可以基于 Mybatis 拦截器进行处理，通过拦截 SQL 语句动态修改添加分表信息，再设置回 Mybatis 执行 SQL 中。
- 此外再完善一些分库分表路由的操作，比如配置默认的分库分表字段以及单字段入参时默认取此字段作为路由字段。
```java
@Intercepts({
  @Signature(
      type = StatementHandler.class,
      method = "prepare",
      args = {Connection.class, Integer.class})
})
public class DynamicMybatisPlugin implements Interceptor {

  private final Pattern pattern =
      Pattern.compile("(from|into|update)[\\s]{1,}(\\w{1,})", Pattern.CASE_INSENSITIVE);

  @Override
  public Object intercept(Invocation invocation) throws Throwable {
    // 获取StatementHandler
    StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
    MetaObject metaObject =
        MetaObject.forObject(
            statementHandler,
            SystemMetaObject.DEFAULT_OBJECT_FACTORY,
            SystemMetaObject.DEFAULT_OBJECT_WRAPPER_FACTORY,
            new DefaultReflectorFactory());
    MappedStatement mappedStatement =
        (MappedStatement) metaObject.getValue("delegate.mappedStatement");

    // 获取自定义注解判断是否进行分表操作
    String id = mappedStatement.getId();
    String className = id.substring(0, id.lastIndexOf("."));
    Class<?> clazz = Class.forName(className);
    DBRouterStrategy dbRouterStrategy = clazz.getAnnotation(DBRouterStrategy.class);
    if (null == dbRouterStrategy || !dbRouterStrategy.splitTable()) {
      return invocation.proceed();
    }

    // 获取SQL
    BoundSql boundSql = statementHandler.getBoundSql();
    String sql = boundSql.getSql();

    // 替换SQL表名 USER 为 USER_03
    Matcher matcher = pattern.matcher(sql);
    String tableName = null;
    if (matcher.find()) {
      tableName = matcher.group().trim();
    }
    assert null != tableName;
    String replaceSql = matcher.replaceAll(tableName + "_" + DBContextHolder.getTBKey());

    // 通过反射修改SQL语句
    Field field = boundSql.getClass().getDeclaredField("sql");
    field.setAccessible(true);
    field.set(boundSql, replaceSql);
    field.setAccessible(false);

    return invocation.proceed();
  }
}

```

- 实现 Interceptor 接口的 intercept 方法，获取 StatementHandler、通过自定义注解判断是否进行分表操作、获取 SQL 并替换 SQL 表名 USER 为 USER_03、最后通过反射修改SQL语句
