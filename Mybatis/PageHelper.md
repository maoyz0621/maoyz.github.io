# PageHelper

MyBatis的分页插件

## 依赖

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>最新版本</version>
</dependency>
```



## 配置拦截器插件

### xml中配置

```xml
<plugins>
    <!-- com.github.pagehelper为PageHelper类所在包名 -->
    <plugin interceptor="com.github.pagehelper.PageInterceptor">
        <property name="param1" value="value1"/>
	</plugin>
</plugins>
```

### Spring属性配置文件中配置拦截器

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
  <!-- 注意其他配置 -->
  <property name="plugins">
    <array>
      <bean class="com.github.pagehelper.PageInterceptor">
        <property name="properties">
          <!--使用下面的方式配置参数，一行配置一个 -->
          <value>
            pageSizeZero=false
          </value>
        </property>
      </bean>
    </array>
  </property>
</bean>
```



## 代码中使用

重要提示：

> 1、`PageHelper.startPage`方法重要提示
>
> 只有紧跟在`PageHelper.startPage`方法后的**第一个**Mybatis的**查询（Select）**方法会被分页
>
> 2、分页插件不支持带有`for update`语句的分页
>
> 对于带有`for update`的sql，会抛出运行时异常，对于这样的sql建议手动分页
>
> 3、分页插件不支持嵌套结果映射
>
> 由于嵌套结果方式会导致结果集被折叠，因此分页查询的结果在折叠后总数会减少，所以无法保证分页结果数量正确。

### 方法1

```java
Page<Object> page = PageHelper.startPage(pageNum, pageSize);
// 此时的all 属于Page
List<UserVO> all = userVOMapper.getAll();
// 使用PageInfo封装
PageInfo<UserVO> pageInfo = new PageInfo<>(all);
```
### 方法2

#### 调用链模式 - doSelectPageInfo()

```java
PageInfo<Object> pageInfo = PageHelper.startPage(pageNum, pageSize)
                            .doSelectPageInfo(() -> {
                            userVOMapper.getAll();
                        });
```

#### 不执行查询数量操作（select count(0)）

`设置count(false)`

```java
PageInfo<Object> pageInfo = PageHelper.startPage(pageNum, pageSize)
                            .count(false)
                            .doSelectPageInfo(() -> {
                            userVOMapper.getAll();
                        });
```
#### 分页total失效

```java
Page<Object> page = PageHelper.startPage(pageNum, pageSize);
List<UserVO> all = userVOMapper.getAll();
// 后对list数据操作,可以分页，但是数据量错误，total始终等于每页数据量，即pageSize
List<UserVO> copy = new ArrayList<>();
for (UserVO userVO : all) {
    // 类型转换
    copy.add(vo);
}
PageInfo<UserVO> pageInfo = new PageInfo<>(copy);
```

此时total = pageInfo.getList().size()

原因分析：com.github.pagehelper.PageInfo

```java
public PageInfo(List<T> list, int navigatePages) {
        if (list instanceof Page) {
            Page page = (Page) list;
            ...
            // total值是page中的值
            this.total = page.getTotal();
            ...
        } else if (list instanceof Collection) {
            ...
            // total值是list的大小
            this.total = list.size();
            ...
        }
        ...
    }
```

类型未转换时，其Instance其实是`Page`，而对于`new PageInfo<>(copy)`，其Instance是Collecton