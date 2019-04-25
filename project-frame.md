# 应用分层
![结构分层.png](https://upload-images.jianshu.io/upload_images/4092000-43b1c8ab2362393a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

默认上层依赖于下层，箭头关系表示可直接依赖，如：开放接口层可以依赖于Web 层，也可以直接依赖于 Service 层，依此类推：
![应用分层.png](https://upload-images.jianshu.io/upload_images/4092000-79a0a3a5fcf1793c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 开放接口层：可直接封装 Service 方法暴露成 RPC 接口；通过 Web 封装成 http 接口；进行网关安全控制、流量控制等。

> frame中，对应的是rpc层。第三方系统调用只能调用rpc层，不能调用web层，web层只能是自己模块才可以调用。

2. 终端显示层：各个端的模板渲染并执行显示的层。当前主要是 velocity 渲染，JS 渲染，JSP 渲染，移动端展示等。
3. Web 层：主要是对访问控制进行转发，各类基本参数校验，或者不复用的业务简单处理等。

>  frame中，对应的是web层，web层只能由自己模块的终端显示层才可以访问。

4. Service 层：相对具体的业务逻辑服务层。frame中，对应的是service层
5. Manager 层：通用业务处理层，它有如下特征：

> 对第三方平台封装的层，预处理返回结果及转化异常信息；
对 Service 层通用能力的下沉，如缓存方案、中间件通用处理；
与 DAO 层交互，对多个 DAO 的组合复用。

6. DAO 层：数据访问层，与底层 MySQL、Oracle、Hbase 等进行数据交互。

>  frame中，对应的是DAO层，系统采用mybatis和数据库进行交互，sql只能写在xml中，所有业务都需要接口和实现分离。可以辅助设计阶段和实现阶段的分离。

7. 外部接口或第三方平台：包括其它部门 RPC 开放接口，基础平台，其它公司的 HTTP 接口。

# POJO
POJO 专指只有 setter / getter/ toString 的简单类
- BO（Business Object）：业务对象，由 Service 层输出的封装业务逻辑的对象
> 为了减少POJO间的转换，Service层可以返回VO和DTO，一般情况下，不需要使用BO，只有在web层和rpc层需要

- DO（Data Object）：在多个表联合查询出来的对象，在PO无法满足的情况下，用DO进行承载。
- PO（Persist Object）：此对象 与数据库表结构一一对应，通过 DAO 层向上传输数据源对象。
- DTO（Data Transfer Object）：数据传输对象，Service 或 Manager 向外传输的对象。

> 用于和第三方系统对接，在rpc层分为xxxRequestDTO和xxxResponseDTO(强制)

- VO（View Object）：显示层对象，通常是 Web 向模板渲染引擎层传输的对象。
- Query：数据查询对象，各层接收上层的查询请求。注意超过 2 个参数的查询封装，禁止使用 Map 类来传输。
- Form：表单对象，接收来自页面的表单数据。注意超过 2 个参数的表单封装，禁止使用 Map 类来接收。
- AO（Application Object）：应用对象，在 Web 层与 Service 层之间抽象的复用对象模型，极为贴近展示层，复用度不高。

# POJO使用及传输规则
> Web层只会存在Form,Query,VO,BO
   RPC层只会存在Form,Query,DTO,BO,VO
   DAO层只会存在DO,PO,Query
  service,manager层由于是业务处理，所有POJO都可能出现

- 终端显示到service
 ![终端->service.png](https://upload-images.jianshu.io/upload_images/4092000-6cdb21ce20227daf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - 第三方应用到service
![第三方->service.png](https://upload-images.jianshu.io/upload_images/4092000-e69346ba78fac3dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 - service到DAO
![service->DAO.png](https://upload-images.jianshu.io/upload_images/4092000-8cc0943f8fe97ab3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
