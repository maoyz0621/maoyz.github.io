1. Mybatis四大对象？

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)拦截执行器的方法

- ParameterHandler (getParameterObject, setParameters)请求参数处理

- ResultSetHandler (handleResultSets, handleOutputParameters)返回结果集处理

- StatementHandler (prepare, parameterize, batch, update, query)Sql语法构建的处理

 

2. Executor执行器有哪些？

   - SimpleExecutor：每执行一次 update或select，就开启一个Statement对象，用完立刻关闭Statement对象
   - ReuseExecutor：执行 update 或 select，以 sql 作为 key 查找 Statement 对象，存在就使用，不存在就创建，用完后，不关闭Statement对象，而是放置于Map<String, Statement>内，供下一次使用。简言之，就是重建使用 Statement 对象
   - Batch Executor：执行 update (没有 select, JDBC 批处理不支持 select), 将所有sql都添加到批处理中(addBatch()),等待统一执行(executeBatch()), 它缓存了多个Statement对象，每个Statement对象都是addBatch()完毕后， 等待逐一执行executeBatch()批处理。与JDBC批处理相同。

> 作用范围：Executor的这些特点，都严格限制在SqISession生命周期范围内。

2. #{}和${}的区别是什么？

   #{}：预编译处理；${}：字符串替换
   在处理#{}时，会将sql中的#{}替换为?号，调用PreparedStatement的set方法来赋值；在处理${}时，就是将${}替换成变量的值.
   作用：使用#{}可以有效的防止SQL注入，提高系统的安全性！
   

   
3. 如何在在mapper中如何传递多个参数?

  - **顺序传参法**

  - **@param注解传参法**

  - **多个参数封装成map（Map传参法）**

  - **Java Bean传参法**

    

4. Mybatis的xml映射文件中，不同的xml映射文件，id是否可以重复？

    不同namespace的xml映射文件，id可以重复；同一个xml映射文件中id不能重复；
    由于namespace+id 是作为 Map＜String, MappedStatement＞的key使用的，如果没有namespace，就剩下id，那么，id重复会导致数据互相覆盖。 有了 namespace，自然id就可以重夏，namespace不同，namespace+id自然也就不同。

    

5. Mybatis是否支持延迟加载？如果支持，它的实现原理是什么？

   Mybatis仅支持association（一对一）关联对象和collection（一对多）关联集合对象的延迟加载。在Mybatis配置文件中，可以配置是否启用延迟加载lazyLoadingEnabled=true|false。

   其基本原理是，使用CGLIB创建目标对象的代理对象，当调用目标方法时，进入拦截器方法，比如调用a.getB().getName()，拦截器invoke()方法发现a.getB()是null值，那么就会单独发送事先保存好的查询关联B对象的sql，把B查询上来，然后调用a.setB(b)，于是a的对象b属性就有值了，接着完成a.getB().getName()方法的调用。

   

6. Mybatis的一级、二级缓存:

   （1）一级缓存: 基于 PerpetualCache 的 HashMap 本地缓存，其存储作用域为 Session，当 Session flush 或 close 之后，该 Session 中的所有 Cache 就将清空，默认打开一级缓存。

   （2）二级缓存与一级缓存其机制相同，默认也是采用 PerpetualCache，HashMap 存储，不同在于其存储作用域为 Mapper(Namespace)，并且可自定义存储源，如 Ehcache。默认不打开二级缓存，要开启二级缓存，使用二级缓存属性类需要实现Serializable序列化接口(可用来保存对象的状态),可在它的映射文件中配置<cache/> ；

   （3）对于缓存数据更新机制，当某一个作用域(一级缓存 Session/二级缓存Namespaces)的进行了C/U/D 操作后，默认该作用域下所有 select 中的缓存将被 clear 掉并重新更新，如果开启了二级缓存，则只根据配置判断是否刷新

   

7. Mybatis与Spring整合之后，一级缓存”失效“？

    Spring在执行完DAO之后会立即关闭sqlSession对象；

    Mybatis原生使用的SqlSession是DefaultSqlSession，而整合Spring之后是SqlSessionTemplate，具体执行的是

    SqlSessionTemplate.SqlSessionInterceptor，其中执行完之后closeSqlSession，而closeSqlSession会根据是否开启事务，决定**Releasing transactional SqlSession**或**Closing non transactional SqlSessio**n。

    > 当同一个线程开启事务时同一个sql查询多次会走一级缓存，而不开启事务时，每一查询都是不同的sqlsession即缓存为“失效”状态

    

8. 使用MyBatis的mapper接口调用时有哪些要求？

    - Mapper接口方法名和mapper.xml中定义的每个sql的id相同；
    - Mapper接口方法的输入参数类型和mapper.xml中定义的每个sql 的parameterType的类型相同；
    - Mapper接口方法的输出参数类型和mapper.xml中定义的每个sql的resultType的类型相同；
    - Mapper.xml文件中的namespace即是mapper接口的类路径。
