# Spring整合Mybatis

## SqlSessionFactoryBean

在MyBatis中，是通过 SqlSessionFactoryBuilder 来创建 SqlSessionFactory 的。 而在 MyBatis-Spring 中，则使用 SqlSessionFactoryBean 来创建的。

```java
public class SqlSessionFactoryBean implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {

    // InitializingBean
    @Override
    public void afterPropertiesSet() throws Exception {
        notNull(dataSource, "Property 'dataSource' is required");
        notNull(sqlSessionFactoryBuilder, "Property 'sqlSessionFactoryBuilder' is required");
        state((configuration == null && configLocation == null) || !(configuration != null && configLocation != null),
                "Property 'configuration' and 'configLocation' can not specified with together");

        this.sqlSessionFactory = buildSqlSessionFactory();
    }
    
    // ApplicationListener
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (failFast && event instanceof ContextRefreshedEvent) {
            // fail-fast -> check all statements are completed
            this.sqlSessionFactory.getConfiguration().getMappedStatementNames();
        }
    }
    
    // 以下3个属于FactoryBean
    @Override
    public SqlSessionFactory getObject() throws Exception {
        if (this.sqlSessionFactory == null) {
            afterPropertiesSet();
        }

        return this.sqlSessionFactory;
    }

    @Override
    public Class<? extends SqlSessionFactory> getObjectType() {
        return this.sqlSessionFactory == null ? SqlSessionFactory.class : this.sqlSessionFactory.getClass();
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
    
    // spring中真正构建SqlSessionFactory的方法，就是读取之前mybatis.xml中配置信息
    protected SqlSessionFactory buildSqlSessionFactory() throws IOException {
        Configuration configuration;
        XMLConfigBuilder xmlConfigBuilder = null;
        if (this.configuration != null) {
            configuration = this.configuration;
            if (configuration.getVariables() == null) {
                configuration.setVariables(this.configurationProperties);
            } else if (this.configurationProperties != null) {
                configuration.getVariables().putAll(this.configurationProperties);
            }
        } else if (this.configLocation != null) {
            xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
            configuration = xmlConfigBuilder.getConfiguration();
        } else {
            configuration = new Configuration();
            if (this.configurationProperties != null) {
                configuration.setVariables(this.configurationProperties);
            }
        }

        if (this.objectFactory != null) {
            configuration.setObjectFactory(this.objectFactory);
        }

        if (this.objectWrapperFactory != null) {
            configuration.setObjectWrapperFactory(this.objectWrapperFactory);
        }

        if (this.vfs != null) {
            configuration.setVfsImpl(this.vfs);
        }

        if (hasLength(this.typeAliasesPackage)) {
            String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
                    ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
            for (String packageToScan : typeAliasPackageArray) {
                configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                        typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
            }
        }

        if (!isEmpty(this.typeAliases)) {
            for (Class<?> typeAlias : this.typeAliases) {
                configuration.getTypeAliasRegistry().registerAlias(typeAlias);
            }
        }
		// 查看是否注入拦截器，有的话添加到Interceptor集合里面
        if (!isEmpty(this.plugins)) {
            for (Interceptor plugin : this.plugins) {
                configuration.addInterceptor(plugin);
            }
        }

        if (hasLength(this.typeHandlersPackage)) {
            String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
                    ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
            for (String packageToScan : typeHandlersPackageArray) {
                configuration.getTypeHandlerRegistry().register(packageToScan);
            }
        }

        if (!isEmpty(this.typeHandlers)) {
            for (TypeHandler<?> typeHandler : this.typeHandlers) {
                configuration.getTypeHandlerRegistry().register(typeHandler);
            }
        }

        if (this.databaseIdProvider != null) {//fix #64 set databaseId before parse mapper xmls
            try {
                configuration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
            } catch (SQLException e) {
                throw new NestedIOException("Failed getting a databaseId", e);
            }
        }

        if (this.cache != null) {
            configuration.addCache(this.cache);
        }

        if (xmlConfigBuilder != null) {
            try {
                xmlConfigBuilder.parse();
            } catch (Exception ex) {
                throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
            } finally {
                ErrorContext.instance().reset();
            }
        }

        if (this.transactionFactory == null) {
            this.transactionFactory = new SpringManagedTransactionFactory();
        }

        configuration.setEnvironment(new Environment(this.environment, this.transactionFactory, this.dataSource));

        if (!isEmpty(this.mapperLocations)) {
            for (Resource mapperLocation : this.mapperLocations) {
                if (mapperLocation == null) {
                    continue;
                }

                try {
                    XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
                            configuration, mapperLocation.toString(), configuration.getSqlFragments());
                    xmlMapperBuilder.parse();
                } catch (Exception e) {
                    throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
                } finally {
                    ErrorContext.instance().reset();
                }
            }
        } else {
        }

        return this.sqlSessionFactoryBuilder.build(configuration);
    }

}
```



## DataSourceTransactionManager

一个使用 MyBatis-Spring 的其中一个主要原因是它允许 Mybatis 参与到 Spring 的事务管理中。而不是给 MyBatis 创建一个新的专用事务管理器，MyBatis-Spring 借助了 Spring 中的 DataSourceTransactionManager 来实现事务管理。

一旦配置好了 Spring 的事务管理器，你就可以在 Spring 中按你平时的方式来配置事务。并且支持 @Transactional 注解和 AOP 风格的配置。在事务处理期间，一个单独的 SqlSession 对象将会被创建和使用。当事务完成时，这个 session 会以合适的方式提交或回滚。

事务配置好了以后，MyBatis-Spring 将会透明地管理事务。这样在你的 DAO 类中就不需要额外的代码了。

**注意：为事务管理器指定的 DataSource 必须和用来创建 SqlSessionFactoryBean 的是同一个数据源，否则事务管理器就无法工作了。**



## SqlSessionTemplate

在 MyBatis 中，你可以使用 SqlSessionFactory 来创建 SqlSession。一旦你获得一个 session 之后，你可以使用它来执行映射了的语句，提交或回滚连接，最后，当不再需要它的时候，你可以关闭 session。使用 MyBatis-Spring 之后，你不再需要直接使用 SqlSessionFactory 了，因为你的 bean 可以被注入一个线程安全的 SqlSession，它能基于 Spring 的事务配置来自动提交、回滚、关闭 session。线程安全的。

```java
public class SqlSessionTemplate implements SqlSession, DisposableBean {

    private final SqlSessionFactory sqlSessionFactory;
    
    private final ExecutorType executorType;
    
    private final SqlSession sqlSessionProxy;
    
    private final PersistenceExceptionTranslator exceptionTranslator;

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
                              PersistenceExceptionTranslator exceptionTranslator) {
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;
        this.exceptionTranslator = exceptionTranslator;
        // mybatis中的映射器代理工厂MapperProxyFactory.newInstance()，现在交由spring处理
        // 生成SqlSession的代理类，代理类执行方法都需要执行拦截器SqlSessionInterceptor，这个拦截器是实现SqlSession管理的重点
        this.sqlSessionProxy = (SqlSession) newProxyInstance(
                SqlSessionFactory.class.getClassLoader(),
                new Class[]{SqlSession.class},
                new SqlSessionInterceptor());
    }
    
    // 重要：回调拦截器, 实现InvocationHandler接口，对代理类方法的调用都会执行回调方法invoke
    private class SqlSessionInterceptor implements InvocationHandler {
    
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            SqlSession sqlSession = getSqlSession(
                    SqlSessionTemplate.this.sqlSessionFactory,
                    SqlSessionTemplate.this.executorType,
                    SqlSessionTemplate.this.exceptionTranslator);
            try {
                // 执行被代理类方法（真正的方法调用，其实就是调用sqlSession的目标方法）
                Object result = method.invoke(sqlSession, args);
                // 如果sqlSession没有关联事务，则提交
                if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    // force commit even on non-dirty sessions because some databases require
                    // a commit/rollback before calling close()
                    sqlSession.commit(true);
                }
                return result;
            } catch (Throwable t) {
                Throwable unwrapped = unwrapThrowable(t);
                if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
                    // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
                    closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                    sqlSession = null;
                    Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
                    if (translated != null) {
                        unwrapped = translated;
                    }
                }
                throw unwrapped;
            } finally {
                if (sqlSession != null) {
                    // 关闭sqlSession，所以每次查询都会创建新的sqlSession，所以一级缓存根本不生效
                    closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                }
            }
        }
    }
}

```



## 事务管理




```java
    public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {
	    // 当前sessionFactory是否有事务绑定的session（SqlSessionHolder是session的持有对象）
        // 1.如果threadlocal的map中含有当前的sessionfactory则返回holder
        // 2.若无则返回null
        SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

        SqlSession session = sessionHolder(executorType, holder);
        if (session != null) {
            return session;
        }
        // 创建一个DefualtSqlSessionFacotry  -- 这里是session真正被创建的地方
        session = sessionFactory.openSession(executorType);
        registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
        return session;
    }

    // 存在绑定的session，且session与事务关联
    private static SqlSession sessionHolder(ExecutorType executorType, SqlSessionHolder holder) {
        SqlSession session = null;
        if (holder != null && holder.isSynchronizedWithTransaction()) {
            if (holder.getExecutorType() != executorType) {
                throw new TransientDataAccessResourceException("Cannot change the ExecutorType when there is an existing transaction");
            }

            // session引用次数加1
            holder.requested();

            // 返回session -- 其实就是当前sessionFactory有与事务关联的session，则返回
            session = holder.getSqlSession();
        }
        return session;
    }

    private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,
                                              PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
        SqlSessionHolder holder;
        // 重点在这，当前线程是否存在事务，若开启了事务则返回true否则返回false
        if (TransactionSynchronizationManager.isSynchronizationActive()) {
            Environment environment = sessionFactory.getConfiguration().getEnvironment();

            if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {

                // 封装持有session对象 
                holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
                // sessionFactory与session绑定
                TransactionSynchronizationManager.bindResource(sessionFactory, holder);
                // 注册当前线程事务绑定
                TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
                // 设置session被事务管理
                holder.setSynchronizedWithTransaction(true);
                // session引用次数加1
                holder.requested();
            } else {
                if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
                } else {
                    throw new TransientDataAccessResourceException(
                            "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
                }
            }
        } else {
        }
    }
	// 关闭SqlSession
    public static void closeSqlSession(SqlSession session, SqlSessionFactory sessionFactory) {
        // 当前sessionFactory是否有事务绑定的session（SqlSessionHolder是session的持有对象）
        SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);
        if ((holder != null) && (holder.getSqlSession() == session)) {
            // 存在事务，则引用次数减1
            holder.released();
        } else {
            // 不存在事务，则关闭session
            session.close();
        }
    }

```

- TransactionSynchronizationManager.getResource

```java
public static Object getResource(Object key) {
   Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
   Object value = doGetResource(actualKey);
   return value;
}

private static Object doGetResource(Object actualKey) {
    // ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<Map<Object, Object>>("Transactional resources");
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    Object value = map.get(actualKey);
    // Transparently remove ResourceHolder that was marked as void...
    if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
        map.remove(actualKey);
        // Remove entire ThreadLocal if empty...
        if (map.isEmpty()) {
            resources.remove();
        }
        value = null;
    }
    return value;
}

```

其中重要的是ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal<Map<Object, Object>>("Transactional resources")，其中key为SqlSessionFactory，如果map中存在这个key，则可能返回相同的sqlsessionHolder，那么resources 的set方法

```java
public static void bindResource(Object key, Object value) throws IllegalStateException {
   Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
   Assert.notNull(value, "Value must not be null");
   Map<Object, Object> map = resources.get();
   // set ThreadLocal Map if none found
   if (map == null) {
      map = new HashMap<Object, Object>();
      resources.set(map);
   }
   // 将SqlSessionFactory和SqlSessionHolder放入threadlocal的map当中
   Object oldValue = map.put(actualKey, value);
   // Transparently suppress a ResourceHolder that was marked as void...
   if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
      oldValue = null;
   }
   if (oldValue != null) {
      throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
            actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
   }
}
```

Mybatis与Spring整合之后，一级缓存”失效“案例：

|                                               |
| :-------------------------------------------: |
|     ![](.\image\Mybatis_一级缓存生效.png)     |
| ![](.\image\Mybatis_一级缓存生效执行结果.png) |
|     ![](.\image\Mybatis_一级缓存失效.png)     |
| ![](.\image\Mybatis_一级缓存失效执行结果.png) |

