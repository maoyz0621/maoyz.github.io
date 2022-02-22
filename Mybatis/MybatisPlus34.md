## IService

### saveOrUpdate(T)

```java
Objects.isNull(getById((Serializable) idVal)) ? save(entity) : updateById(entity)
```

最终调用的是`save`或`updateById`



### saveBatch

设置开启数据库批量提交&rewriteBatchedStatements=true，可以解决批量插入慢的问题

### 逻辑删除

配置yml文件

```yaml
mybatis-plus:
  global-config:
    db-config:
      logic-delete-field: enabled # 全局逻辑删除的实体字段名(since 3.3.0,配置后可以忽略不配置步骤2)
      logic-delete-value: 1 # 逻辑已删除值(默认为 1)
      logic-not-delete-value: 0 # 逻辑未删除值(默认为 0)
```

在实体类字段上加上`@TableLogic`注解

```java
@TableLogic
private Integer enabled;
```

