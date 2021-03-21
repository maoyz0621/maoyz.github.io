# Mybatis的使用方法总结

## 更新、新增

- insert

`int batchInsert(List<UserPO> user) throws SQLException;`

`int batchInsert(@Param("users") List<UserPO> user) throws SQLException;`

```xml
# 原生
<insert id="batchInsert" parameterType="com.myz.entity.UserPO">
	INSERT INTO t_user (last_name, gender, email, role_id)
	VALUES
	<foreach collection="list" item="item" separator=",">
		(#{item.lastName},#{item.gender},#{item.gender},#{item.roleId})
	</foreach>
</insert>

# @Param
<insert id="batchInsert" parameterType="com.myz.entity.UserPO">
	INSERT INTO t_user (last_name, gender, email, role_id)
	VALUES
	<foreach collection="users" item="user" separator=",">
		(#{user.lastName},#{user.gender},#{user.gender},#{user.roleId})
	</foreach>
</insert>
```

- update

`int batchUpdate(List<UserPO> user) throws SQLException;`

```xml
<update id="batchUpdate" parameterType="com.myz.entity.UserPO">
	<foreach collection="list" item="item" separator=";">
		UPDATE t_user SET last_name=#{item.lastName},gender=#{item.gender}, email=#{item.email}
		<where>
			id=#{item.id}
		</where>
	</foreach>
</update>
```


## 查询

### 返回值类型

#### 单表

##### 1. 实体类

返回值一个：实体类

返回值空值：null

返回值多个：``org.apache.ibatis.exceptions.TooManyResultsException: Expected one result (or null) to be returned by selectOne(), but found: 2``

##### 2. Map

```xml
<select id="selectByNameMap" resultType="java.util.Map">
   ...
</select>

<select id="selectByNameMap" resultType="map">
   ...
</select>
```

返回值一个：map值

返回值空值：null

返回值多个：``org.apache.ibatis.exceptions.TooManyResultsException: Expected one result (or null) to be returned by selectOne(), but found: 2``

##### 3. List

返回值最少一个：实体类集合

返回值空值：不是null，空结合，size = 0

#### 多表

```java
public class RolePO {
    private Long id;
    private String roleName;
    private String description;
    
    ...
}
public class UserPO {
    private Long id;
    private String lastName;
    private String gender;
    private String email;
    private Long roleId;
    
	...
}

public class Role {
    private Long id;
    private String roleName;
    private String description;
    //一对多关系(collection)
    private List<User> users;

    ...
}

public class User {
    private Long id;
    private String lastName;
    private String gender;
    private String email;
    //　多对一关系(association)
    private Role role;

	...
}
```


##### 1. 一对多

- 使用左外连接联表查询

```xml
<!--　嵌套查询collection -->
<resultMap id="allUsers" type="com.myz.entity.Role">
	<id column="id" property="id"/>
	<result column="role_name" property="roleName"/>
	<result column="description" property="description"/>
	<!--　collection关联属性,一对多
		  property:属性名称
		  ofType:集合属性类
	 -->
	<collection property="users" ofType="com.myz.entity.User">
		<id column="id" property="id"/>
		<result column="last_name" property="lastName"/>
		<result column="gender" property="gender"/>
		<result column="email" property="email"/>
	</collection>
</resultMap>
```

- 分步查询，先根据条件查出主表结果，在根据查出结果作为查询条件去查询其他表，非连表查询，多个单表查询，在集合

```xml
<!-- collection分步查询 -->
<resultMap id="allUsersStep" type="com.myz.entity.Role">
	<id column="id" property="id"/>
	<result column="role_name" property="roleName"/>
	<result column="description" property="description"/>
	<!--　collection分步查询
		  property:属性名称
		  column:指定哪一列的值传入select方法内
		  fetchType:eager/lazy  立即加载/延迟加载
	-->
	<collection property="users" column="id"
				select="com.myz.dao.UserDao.selectAllByRoleId"
				fetchType="eager"/>
</resultMap>
```

##### 2. 一对一

- 级联属性

```xml
<!--　联合查询：级联属性-->
<resultMap id="myUser" type="com.myz.entity.User">
	<id column="id" property="id" jdbcType="BIGINT"/>
	<result column="last_name" property="lastName"/>
	<result column="gender" property="gender"/>
	<result column="email" property="email"/>
	<!--　级联属性　-->
	<result column="id" property="role.id"/>
	<result column="role_name" property="role.roleName"/>
	<result column="description" property="role.description"/>
</resultMap>
```

- 联合查询，联合对象

```xml
<!-- 联合查询：association联合对象-->
<resultMap id="myUser1" type="com.myz.entity.User">
	<id column="id" property="id"/>
	<result column="last_name" property="lastName"/>
	<result column="gender" property="gender"/>
	<result column="email" property="email"/>
	<!--property:属性对象　
		javaType:Bean对象　-->
	<association property="role" javaType="com.myz.entity.Role">
		<id column="id" property="id"/>
		<result column="role_name" property="roleName"/>
		<result column="description" property="description"/>
	</association>
</resultMap>
```

- 联合查询，分步查询

```xml
<!--　联合查询，分步查询　-->
<resultMap id="userAndRole" type="com.myz.entity.User">
	<id column="id" property="id"/>
	<result column="last_name" property="lastName"/>
	<result column="gender" property="gender"/>
	<result column="email" property="email"/>
	<!--　select:当前属性指定的方法查询
		  property:属性对象
		  column:指定那一列的值传入select方法内
	　-->
	<association column="role_id" property="role"
				 select="com.myz.dao.RoleDao.selectById"/>
</resultMap>
```



### 入参类型

#### Collection

foreach元素的属性主要有 item，index，collection，open，[separator](https://www.baidu.com/s?wd=separator&tn=24004469_oem_dg&rsv_dl=gh_pl_sl_csd)，close。

- item：表示集合中每一个元素进行迭代时的别名
- index：指 定一个名字，用于表示在迭代过程中，每次迭代到的位置
- open：表示该语句以什么开始，“(”
- separator：表示在每次进行迭代之间以什么符号作为分隔符， ","
- close：表示以什么结束，")"

##### 1. Collection

`List<Role> selectDynamicByIdsCollection(Collection<Integer> ids);`

```xml
<select id="selectDynamicByIdsCollection" resultType="com.myz.entity.Role">
	SELECT id , role_name, role_name , description
	FROM t_role
	<where>
		id IN
		<foreach collection="collection" index="index" separator="," item="ids" open="(" close=")">
			#{ids}
		</foreach>
	</where>
</select>
```

##### 2. List

接口名：`List<Role> selectDynamicByIds(List<Integer> ids);`

```xml
<select id="selectDynamicByIds" resultMap="dynamicRole">
	SELECT role_id , role_name, role_name , description FROM t_role
	<where>
		role_id IN
	<foreach collection="list" index="index" separator="," item="ids" open="(" close=")">
		#{ids}
	</foreach>
	</where>
</select>
```

##### 3. Set

`List<Role> selectDynamicByIdsSet(Set<Integer> ids);`

```xml
<select id="selectDynamicByIdsSet" resultType="com.myz.entity.Role">
	SELECT id , role_name, role_name , description
	FROM t_role
	<where>
		id IN
		<foreach collection="collection" index="index" separator="," item="ids" open="(" close=")">
			#{ids}
		</foreach>
	</where>
</select>
```

> **注：collection的值没有"set"，如果'list'，也是错误的，只能是'collection'**

##### 4. Array

`List<Role> selectDynamicByIdsArray(Integer[] ids);`

```xml
<select id="selectDynamicByIdsArray" resultType="com.myz.entity.Role">
	SELECT id , role_name, role_name , description
	FROM t_role
	<where>
		id IN
		<foreach collection="array" index="index" separator="," item="ids" open="(" close=")">
			#{ids}
		</foreach>
	</where>
</select>
```



#### 实体类

`User selectByPOJO(UserPO user);`

根据pojo属性查询， #{属性名}

```xml
<!-- 多参数查询 ,根据pojo-->
<select id="selectByPOJO" resultType="com.myz.entity.User">
	SELECT * FROM t_user
	<where>
		id = #{id} and last_name=#{lastName}
	</where>
</select>
```



#### Map

`User selectByMap(Map<String, Object> map);`

根据Map中key，#{key}分别为id，lastName

```xml
<!-- 多参数查询 ,根据map-->
<select id="selectByMap" resultType="com.myz.entity.User">
	SELECT * FROM t_user
	<where>
		id = #{id} and last_name=#{lastName}
	</where>
</select>
```



## 插件






mapper接口方法重载，无法映射到java的重载，不支持
```
/**
	* 　根据id查询
*/
User selectById(Integer id) throws SQLException;

/**
	* 参数不同的同名方法
*/
User selectById(Integer id, String lastName) throws SQLException;
```

对应的xml同名id只能有一个

`Available parameters are [arg1, arg0, param1, param2]`



## 插件使用

```java
// 执行器
Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
// 请求参数处理
ParameterHandler (getParameterObject, setParameters)
// 返回结果集处理
ResultSetHandler (handleResultSets, handleOutputParameters)
// SQL语句构建
StatementHandler (prepare, parameterize, batch, update, query)
```

