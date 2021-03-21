# JSON解析

## 使用jackson解析

### 简单Bean

其中Bean中包含了基本类型，包装类型，日期类型和布尔类型等

1. 基本类型和包装类型的区别？

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class PersonInfo {
	/**
	* 包装类型
	*/
    private Integer age;
    /**
	* 基本类型
	*/
    private int height;
    /**
     * boolean 类型
     */
    private Boolean sex;
}
```

基本类型，int=0，包装类型，Integer = null；一般建议使用包装类型

2. Date类型

```java
@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class PersonInfo {
     /**
     * 日期格式 日期
     */
    @JsonFormat(pattern = "yyyy-MM-dd")
    @JsonProperty("date")
    private Date date;

    /**
     * 时间格式 时间
     */
    @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss", iso = DateTimeFormat.ISO.DATE_TIME)
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    @JsonProperty("time")
    private Date time;
}
```

+ 入参格式化，将String转换成Date，一般是前台到后台的时间格式的转换

```java
@DateTimeFormat(pattern="yyyy-MM-dd HH:mm:ss")
```

+ 出参格式化 ，将Date转换成String，一般是后台到前台的时间格式的转换

```java
@JsonFormat(pattern="yyyy-MM-dd HH:mm:ss", timezone="GMT+8")
```

> pattern:需要转换的时间日期的格式
>
> timezone:时间设置为东八区，避免时间在转换中有误差



3. LocalDateTime 和 LocalDate

   ```java
   @Data
   @JsonIgnoreProperties(ignoreUnknown = true)
   public class PersonInfo {
       /**
        * 字段属性为LocalDate
        */
       @JsonFormat(pattern = "yyyy-MM-dd")
       @JsonProperty("localDay")
       @JsonDeserialize(using = LocalDateDeserializer.class)
       @JsonSerialize(using = LocalDateSerializer.class)
       private LocalDate localDay;
   
       /**
        * 字段属性为LocalDateTime
        */
       @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
       @JsonProperty("localTime")
       @JsonDeserialize(using = LocalDateTimeDeserializer.class)
       @JsonSerialize(using = LocalDateTimeSerializer.class)
       private LocalDateTime localTime;
   }
   ```
   
   

反系列化**@JsonDeserialize**和序列化**@JsonSerialize**

```java
# LocalDate
@JsonDeserialize(using = LocalDateDeserializer.class)
@JsonSerialize(using = LocalDateSerializer.class)

# LocalDateTime
@JsonDeserialize(using = LocalDateTimeDeserializer.class)
@JsonSerialize(using = LocalDateTimeSerializer.class)
```



依赖包：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
    <version>2.9.9</version>
</dependency>
```

案列：

前置条件：\"localTime\" : \"2020-08-28 11:11:11"

```json
# 1、LocalDateXxx不使用@JsonDeserialize进行反序列化，是无法解析"2020-09-28 11:11:21"时间格式的

# 2、LocalDateXxx使用@JsonDeserialize进行反序列化
"localTime": {
	"year": 2020,
	"month": "AUGUST",
	"dayOfMonth": 28,
	"dayOfWeek": "FRIDAY",
	"dayOfYear": 241,
	"monthValue": 8,
	"nano": 0,
	"hour": 11,
	"minute": 11,
	"second": 11,
	"chronology": {
		"id": "ISO",
		"calendarType": "iso8601"
	}
}

# 3、LocalDateXxx使用@JsonDeserialize进行反序列化，同时使用@JsonSerialize进行序列化
"localTime":"2020-08-28 11:11:11"
```



### 嵌套Bean

```java
@Setter
@Getter
@JsonIgnoreProperties(ignoreUnknown = true)
public class DemoBean implements Serializable {
    private String msg;
    private int code;

    /**
     * 单个bean
     * 有值
     */
    @JsonProperty("table")
    private Table table = new Table();

    /**
     * 没值：
     * "table1":null
     * "table2":{"index":null,"column":null}
     */
    @JsonProperty("table1")
    private Table table1;
    @JsonProperty("table2")
    private Table table2 = new Table();

    /**
     * bean list
     * 有值
     */
    @JsonProperty("context")
    private List<Table> context = new LinkedList<>();

    /**
     * 没值
     * "context1":null
     * "context2":[]
     */
    @JsonProperty("context1")
    private List<Table> context1;
    @JsonProperty("context2")
    private List<Table> context2 = new LinkedList<>();

    @JsonProperty("inner")
    private InnerDemoBean innerDemoBean = new InnerDemoBean();

    /**
     * 定义内部类
     */
    @Setter
    @Getter
    @JsonIgnoreProperties(ignoreUnknown = true)
    public class InnerDemoBean {
        @JsonProperty("username")
        private String userName;

    }
}

@Setter
@Getter
public class Table implements Serializable {
    private String index;
    private Integer column;
}
```

没有默认值，解析的对象不存在，则为null，解析的数组不存在，则为null；

若有默认值，解析的对象不存在，则为空对象，解析的数组不存在，则为空List

对于嵌套的内部类，一般建议使用static class



### 自定义序列化

```java
/**
 * 注解@JsonSerialize
 * using：自定义序列化
 * nullsUsing：自定义null序列化默认值
 */
@Data
public class JsonSerializerBean {

    /**
     * 空值 显示0.00
     */
    @JsonSerialize(using = MyNumberJsonSerializer.class, nullsUsing = MyNumberJsonSerializer.class)
    private BigDecimal price;

    /**
     * 转义 **
     */
    @JsonSerialize(using = MyNumberJsonSerializer.class)
    private BigDecimal feePrice;

    /**
     * 保留2位小数
     */
    @JsonSerialize(using = MyNumberJsonSerializer.class)
    private BigDecimal preferentialPrice;

    @JsonSerialize(using = MyStatusJsonSerializer.class)
    private int status;

    public static void main(String[] args) throws JsonProcessingException {
        JsonSerializerBean bean = new JsonSerializerBean();
        bean.setPrice(null);
        bean.setFeePrice(new BigDecimal("-0.1"));
        bean.setPreferentialPrice(new BigDecimal("12.3454"));
        bean.setStatus(2);

        ObjectMapper objectMapper = new ObjectMapper();
        // {"price":0.00,"feePrice":"**","preferentialPrice":12.34,"status":"待审核"}
        System.out.println(objectMapper.writeValueAsString(bean));
    }
}

public class MyNumberJsonSerializer extends JsonSerializer<BigDecimal> {
    /**
     * 保留2位小数 #.00 表示两位小数 #.0000四位小数
     */
    public static final DecimalFormat FORMAT;

    static {
        FORMAT = new DecimalFormat("#0.00");
        FORMAT.setRoundingMode(RoundingMode.DOWN);
    }

    @Override
    public void serialize(BigDecimal bigDecimal, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
        // null -> 0.00
        if (bigDecimal == null) {
            jsonGenerator.writeNumber(new BigDecimal("0.00"));
        } else if (new BigDecimal("-0.1").equals(bigDecimal)) {
            // **
            jsonGenerator.writeString("**");
        } else {
            jsonGenerator.writeNumber(FORMAT.format(bigDecimal));
        }
    }
}

public class MyStatusJsonSerializer extends JsonSerializer<Integer> {

    @Override
    public void serialize(Integer status, JsonGenerator jsonGenerator, SerializerProvider serializerProvider) throws IOException {
        String statusDesc = "";
        switch (status) {
            case 0:
                statusDesc = "暂存";
                break;
            case 1:
                statusDesc = "待上报";
                break;
            case 2:
                statusDesc = "待审核";
                break;
            case 3:
                statusDesc = "已审";
            default:
                statusDesc = "状态信息不符合";
        }
        jsonGenerator.writeString(statusDesc);
    }
}
```