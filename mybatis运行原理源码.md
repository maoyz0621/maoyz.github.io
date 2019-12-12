## 1. 创建Session

### 1.1 SqlSessionFactoryBuilder创建SqlSessionFactory；
```
public class SqlSessionFactoryBuilder {

    //第4种方法是最常用的，它使用了一个参照了XML文档或更特定的SqlMapConfig.xml文件的Reader实例。
    //可选的参数是environment和properties。Environment决定加载哪种环境(开发环境/生产环境)，包括数据源和事务管理器。
    //如果使用properties，那么就会加载那些properties（属性配置文件），那些属性可以用${propName}语法形式多次用在配置文件中。和Spring很像，一个思想？
    public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
        try {
            // 加载mybatis.xml配置信息
            XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
            // parser.parse()解析xml,获得Configuration, 返回SqlSessionFactory
            return build(parser.parse());
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
            ErrorContext.instance().reset();
            try {
                reader.close();
            } catch (IOException e) {
                // Intentionally ignore. Prefer previous error.
            }
        }
    }
    
    // 将Configuration传递给SqlSessionFactory, 默认返回DefaultSqlSessionFactory
    public SqlSessionFactory build(Configuration config) {
        return new DefaultSqlSessionFactory(config);
    }
}
```

### 1.2 SqlSessionFactory创建SqlSession；

```
public class DefaultSqlSessionFactory implements SqlSessionFactory {

    @Override
    public SqlSession openSession() {
        return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
    }
    
    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;
        try {
            // 从Configuration获取environment配置信息
            final Environment environment = configuration.getEnvironment();
            // 从environment配置信息获取事务工厂
            final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
            // 通过事务工厂来产生一个事务
            tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
            // 生成一个执行器(事务包含在执行器里)
            final Executor executor = configuration.newExecutor(tx, execType);
            // 然后产生一个DefaultSqlSession
            return new DefaultSqlSession(configuration, executor, autoCommit);
        } catch (Exception e) {
            closeTransaction(tx); // may have fetched a connection so lets call close()
            throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
        } finally {
            // 最后清空错误上下文
            ErrorContext.instance().reset();
        }
    }
    
    // 从environment配置信息获取事务工厂
    private TransactionFactory getTransactionFactoryFromEnvironment(Environment environment) {
        //如果没有配置事务工厂，则返回托管事务工厂
        if (environment == null || environment.getTransactionFactory() == null) {
            // 返回托管事务工厂
            return new ManagedTransactionFactory();
        }
        return environment.getTransactionFactory();
    }
}
```

#### 1.2.1 执行器的选择
```
public class Configuration {

    protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;

    // 执行器的选择
    public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
        executorType = executorType == null ? defaultExecutorType : executorType;
        // 这句再做一下保护,囧,防止粗心大意的人将defaultExecutorType设成null?
        executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
        Executor executor;
        // 然后就是简单的3个分支，产生3种执行器BatchExecutor/ReuseExecutor/SimpleExecutor
        if (ExecutorType.BATCH == executorType) {
            executor = new BatchExecutor(this, transaction);
        } else if (ExecutorType.REUSE == executorType) {
            executor = new ReuseExecutor(this, transaction);
        } else {
            executor = new SimpleExecutor(this, transaction);
        }
        
        // 如果要求缓存，生成另一种CachingExecutor(默认就是有缓存),装饰者模式,所以默认都是返回CachingExecutor
        if (cacheEnabled) {
            executor = new CachingExecutor(executor);
        }
        // 此处调用插件,通过插件可以改变Executor行为
        executor = (Executor) interceptorChain.pluginAll(executor);
        return executor;
    }
}
```

### 1.3 SqlSession获取Mapper接口, 获取的接口是代理类是com.sun.proxy.$Proxy1，代理对象是org.apache.ibatis.binding.MapperProxy

```
public class DefaultSqlSession implements SqlSession {
    // 接口编程
    @Override
    public <T> T getMapper(Class<T> type) {
        // 最后会去调用MapperRegistry.getMapper
        return configuration.getMapper(type, this);
    }
)

public class MapperRegistry {
    // 将已经添加的映射都放入HashMap
    private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<Class<?>, MapperProxyFactory<?>>();

    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
        final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
        if (mapperProxyFactory == null) {
            throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
        }
        try {
            return mapperProxyFactory.newInstance(sqlSession);
        } catch (Exception e) {
            throw new BindingException("Error getting mapper instance. Cause: " + e, e);
        }
    }
}

// 映射器代理工厂
public class MapperProxyFactory<T> {

    private final Class<T> mapperInterface;
    private final Map<Method, MapperMethod> methodCache = new ConcurrentHashMap<>();

    public MapperProxyFactory(Class<T> mapperInterface) {
        this.mapperInterface = mapperInterface;
    }

    public Class<T> getMapperInterface() {
        return mapperInterface;
    }

    public Map<Method, MapperMethod> getMethodCache() {
        return methodCache;
    }

    public T newInstance(SqlSession sqlSession) {
        final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
        return newInstance(mapperProxy);
    }
    
    protected T newInstance(MapperProxy<T> mapperProxy) {
        // 用JDK自带的动态代理生成映射器
        return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[]{mapperInterface}, mapperProxy);
    }

}

```

### 1.4 Mapper执行，执行者为MapperProxy的动态代理模式

```
public class MapperProxy<T> implements InvocationHandler, Serializable {

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            // 代理以后，所有Mapper的方法调用时，都会调用这个invoke方法
            // 并不是任何一个方法都需要执行调用代理对象进行执行，如果这个方法是Object中通用的方法（toString、hashCode等）无需执行
            if (Object.class.equals(method.getDeclaringClass())) {
                return method.invoke(this, args);
            } else if (method.isDefault()) {
                if (privateLookupInMethod == null) {
                    return invokeDefaultMethodJava8(proxy, method, args);
                } else {
                    return invokeDefaultMethodJava9(proxy, method, args);
                }
            }
        } catch (Throwable t) {
            throw ExceptionUtil.unwrapThrowable(t);
        }
        // 去缓存中找MapperMethod
        final MapperMethod mapperMethod = cachedMapperMethod(method);
        // 执行
        return mapperMethod.execute(sqlSession, args);
    }
}

// 映射器方法
public class MapperMethod {
    
    public Object execute(SqlSession sqlSession, Object[] args) {
        Object result;
        // 执行时就是4种情况，insert|update|delete|select，分别调用SqlSession的4大类方法
        switch (command.getType()) {
            case INSERT: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.insert(command.getName(), param));
                break;
            }
            case UPDATE: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.update(command.getName(), param));
                break;
            }
            case DELETE: {
                Object param = method.convertArgsToSqlCommandParam(args);
                result = rowCountResult(sqlSession.delete(command.getName(), param));
                break;
            }
            case SELECT:
                if (method.returnsVoid() && method.hasResultHandler()) {
                    // 如果有结果处理器
                    executeWithResultHandler(sqlSession, args);
                    result = null;
                } else if (method.returnsMany()) {
                    // 如果结果有多条记录
                    result = executeForMany(sqlSession, args);
                } else if (method.returnsMap()) {
                    // 如果结果是map
                    result = executeForMap(sqlSession, args);
                } else if (method.returnsCursor()) {
                    result = executeForCursor(sqlSession, args);
                } else {
                    // 否则就是一条记录
                    Object param = method.convertArgsToSqlCommandParam(args);
                    // 执行selectOne()
                    result = sqlSession.selectOne(command.getName(), param);
                    if (method.returnsOptional()
                            && (result == null || !method.getReturnType().equals(result.getClass()))) {
                        result = Optional.ofNullable(result);
                    }
                }
                break;
            case FLUSH:
                result = sqlSession.flushStatements();
                break;
            default:
                throw new BindingException("Unknown execution method for: " + command.getName());
        }
        if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
            throw new BindingException("Mapper method '" + command.getName()
                    + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
        }
        return result;
    }
    
    // 多条结果execute
    private <E> Object executeForMany(SqlSession sqlSession, Object[] args) {
        List<E> result;
        Object param = method.convertArgsToSqlCommandParam(args);
        if (method.hasRowBounds()) {
            RowBounds rowBounds = method.extractRowBounds(args);
            // 执行selectList()
            result = sqlSession.selectList(command.getName(), param, rowBounds);
        } else {
            result = sqlSession.selectList(command.getName(), param);
        }
        // issue #510 Collections & arrays support
        if (!method.getReturnType().isAssignableFrom(result.getClass())) {
            if (method.getReturnType().isArray()) {
                return convertToArray(result);
            } else {
                return convertToDeclaredCollection(sqlSession.getConfiguration(), result);
            }
        }
        return result;
    }
    
    // 返回值为Map
    private <K, V> Map<K, V> executeForMap(SqlSession sqlSession, Object[] args) {
        Map<K, V> result;
        Object param = method.convertArgsToSqlCommandParam(args);
        if (method.hasRowBounds()) {
            RowBounds rowBounds = method.extractRowBounds(args);
            // 执行selectMap()
            result = sqlSession.selectMap(command.getName(), param, method.getMapKey(), rowBounds);
        } else {
            result = sqlSession.selectMap(command.getName(), param, method.getMapKey());
        }
        return result;
    }
}
```

#### 1.4.1 SqlSession真正执行的操作：selectList(String statement, Object parameter, RowBounds rowBounds)

```
public class DefaultSqlSession implements SqlSession {

    @Override
    public <T> T selectOne(String statement, Object parameter) {
        // Popular vote was to return null on 0 results and throw exception on too many.
        List<T> list = this.selectList(statement, parameter);
        if (list.size() == 1) {
            return list.get(0);
        } else if (list.size() > 1) {
            throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
        } else {
            return null;
        }
    }
    
    @Override
    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
            // 根据statement id找到对应的MappedStatement
            MappedStatement ms = configuration.getMappedStatement(statement);
            // 转而用执行器来查询结果,注意这里传入的ResultHandler是null
            return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }
    
    @Override
    public <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds) {
        final List<? extends V> list = selectList(statement, parameter, rowBounds);
        final DefaultMapResultHandler<K, V> mapResultHandler = new DefaultMapResultHandler<>(mapKey,
                configuration.getObjectFactory(), configuration.getObjectWrapperFactory(), configuration.getReflectorFactory());
        final DefaultResultContext<V> context = new DefaultResultContext<>();
        for (V o : list) {
            context.nextResultObject(o);
            mapResultHandler.handleResult(context);
        }
        return mapResultHandler.getMappedResults();
    }
}
```

#### 1.4.2 Executor执行器工作

```
public abstract class BaseExecutor implements Executor {
    
    /**
     * @param ms mapper.xml中映射的语句
     * @param parameter 传递的参数
     * @param rowBounds 分页用，记录限制
     * @param resultHandler 结果处理器
     * @param key 缓存key, 一般缓存框架的数据结构基本上都是 Key-Value 方式存储， MyBatis 对于其 Key 的生成采取规则为：[mappedStementId + offset + limit + SQL + queryParams + environment]生成一个哈希码
     * @param boundSql 绑定的SQL,是从SqlSource而来，将动态内容都处理完成得到的SQL语句字符串，其中包括?,还有绑定的参数
     */
    @Override
    public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
        if (closed) {
            throw new ExecutorException("Executor was closed.");
        }
        // 先清局部缓存，再查询.但仅查询堆栈为0，才清。为了处理递归调用
        if (queryStack == 0 && ms.isFlushCacheRequired()) {
            clearLocalCache();
        }
        List<E> list;
        try {
            // 加一,这样递归调用到上面的时候就不会再清局部缓存了
            queryStack++;
            // 先根据cachekey从localCache去查
            list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
            if (list != null) {
                // 若查到localCache缓存，处理localOutputParameterCache
                handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
            } else {
                // 从数据库查
                list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
            }
        } finally {
            queryStack--;
        }
        if (queryStack == 0) {
            for (DeferredLoad deferredLoad : deferredLoads) {
                deferredLoad.load();
            }
            // issue #601
            deferredLoads.clear();
            if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
                // issue #482
                clearLocalCache();
            }
        }
        return list;
    }
    
    // 从数据库查
    private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
        List<E> list;
        // 先向缓存中放入占位符?
        localCache.putObject(key, EXECUTION_PLACEHOLDER);
        try {
            // 由子类具体实现
            list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
        } finally {
            localCache.removeObject(key);
        }
        // 加入缓存
        localCache.putObject(key, list);
        // 如果是存储过程，OUT参数也加入缓存
        if (ms.getStatementType() == StatementType.CALLABLE) {
            localOutputParameterCache.putObject(key, parameter);
        }
        return list;
    }
}

public class SimpleExecutor extends BaseExecutor {

    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            // 新建一个StatementHandler，默认RoutingStatementHandler，这里会判断是否有StatementHandler插件
            // 这里看到ResultHandler传入了
            StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
            // 准备语句
            stmt = prepareStatement(handler, ms.getStatementLog());
            // StatementHandler.query
            return handler.query(stmt, resultHandler);
        } finally {
            closeStatement(stmt);
        }
    }
}

```

#### 1.4.3 StatementHandler语句处理器

```
public class RoutingStatementHandler implements StatementHandler {

     public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
        // 根据语句类型，委派到不同的语句处理器(STATEMENT|PREPARED|CALLABLE)
        switch (ms.getStatementType()) {
            case STATEMENT:
                delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
                break;
            // Builder时默认的预处理语句处理器(PREPARED)
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
    
    // 默认委派给PreparedStatementHandler
    @Override
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        return delegate.query(statement, resultHandler);
    }
}

// 预处理语句处理器(PREPARED)
public class PreparedStatementHandler extends BaseStatementHandler {
    @Override
    public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
        PreparedStatement ps = (PreparedStatement) statement;
        ps.execute();
        // 结果处理
        return resultSetHandler.handleResultSets(ps);
    }
} 

```

### 1.5 ResultHandler结果处理, TypeHandler：负责java数据类型和jdbc数据类型之间的映射和转换

```
// 负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；
public class DefaultResultSetHandler implements ResultSetHandler {

    @Override
    public List<Object> handleResultSets(Statement stmt) throws SQLException {
        ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

        final List<Object> multipleResults = new ArrayList<>();

        int resultSetCount = 0;
        
        // TypeHandler
        ResultSetWrapper rsw = getFirstResultSet(stmt);

        List<ResultMap> resultMaps = mappedStatement.getResultMaps();
        int resultMapCount = resultMaps.size();
        validateResultMapsCount(rsw, resultMapCount);
        while (rsw != null && resultMapCount > resultSetCount) {
            ResultMap resultMap = resultMaps.get(resultSetCount);
            handleResultSet(rsw, resultMap, multipleResults, null);
            rsw = getNextResultSet(stmt);
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
        }

        String[] resultSets = mappedStatement.getResultSets();
        if (resultSets != null) {
            while (rsw != null && resultSetCount < resultSets.length) {
                ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
                if (parentMapping != null) {
                    String nestedResultMapId = parentMapping.getNestedResultMapId();
                    ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                    handleResultSet(rsw, resultMap, null, parentMapping);
                }
                rsw = getNextResultSet(stmt);
                cleanUpAfterHandlingResultSet();
                resultSetCount++;
            }
        }

        return collapseSingleResultList(multipleResults);
    }
    
}
```

## 配置信息Configuration


```
configuration (properties?, settings?, typeAliases?, typeHandlers?, objectFactory?, objectWrapperFactory?, reflectorFactory?, plugins?, environments?, databaseIdProvider?, mappers?)

    <!--　外部配置文件,其优先级：
        1 在properties元素体内指定的属性首先被读取;
        2 然后根据properties元素中的resource属性读取类路径下属性文件或根据 url 属性指定的路径读取属性文件，并覆盖已读取的同名属性。
        3 最后读取作为方法参数传递的属性，并覆盖已读取的同名属性通过方法参数传递的属性具有最高优先级，resource/url 属性中指定的配置文件次之，最低优先级的是 properties 属性中指定的属性。
    -->
    <properties resource="db.properties">
    
    </properties>
    
    <!-- 设置 -->
    <settings>
        <setting name="cacheEnabled" value="true"/>
        <setting name="lazyLoadingEnabled" value="true"/>
        <setting name="multipleResultSetsEnabled" value="true"/>
        <setting name="useColumnLabel" value="true"/>
        <setting name="useGeneratedKeys" value="false"/>
        <setting name="autoMappingBehavior" value="PARTIAL"/>
        <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
        <setting name="defaultExecutorType" value="SIMPLE"/>
        <setting name="defaultStatementTimeout" value="25"/>
        <setting name="defaultFetchSize" value="100"/>
        <setting name="safeRowBoundsEnabled" value="false"/>
        <setting name="mapUnderscoreToCamelCase" value="false"/>
        <setting name="localCacheScope" value="SESSION"/>
        <setting name="jdbcTypeForNull" value="OTHER"/>
        <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>
    </settings>

    <!-- 类型别名 -->
    <typeAliases>
        <typeAlias alias="Author" type="domain.blog.Author"/>
    </typeAliases>

    <!-- 类型处理器 -->
    <typeHandlers>

    </typeHandlers>
    
    <!-- 对象工厂 -->
    <objectFactory type="org.mybatis.example.ExampleObjectFactory">
        <property name="someProperty" value="100"/>
    </objectFactory>

    <!-- 插件 -->
    <plugins>
        <!--Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)-->
        <!--ParameterHandler (getParameterObject, setParameters)-->
        <!--ResultSetHandler (handleResultSets, handleOutputParameters)-->
        <!--StatementHandler (prepare, parameterize, batch, update, query)-->
    </plugins>

    <!-- 环境配置 -->
    <environments default="development">
        <environment id="test">
            <!-- 事务管理器 -->
            <transactionManager type="MANAGED">
                <property name="closeConnection" value="false"/>
            </transactionManager>
            <!-- 数据源 -->
            <dataSource type="UNPOOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <!--　可以指定默认值　-->
                <property name="username" value="${username:root}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>

    <!-- 数据库厂商标识 -->
    <databaseIdProvider type="DB_VENDOR">
        <property name="Oracle" value="oracle"/>
    </databaseIdProvider>

    <!-- 映射器 -->
    <mappers>
        <package name="com.myz.dao"/>
    </mappers>
```