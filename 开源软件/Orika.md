## orika 对象拷贝
依赖 
```
    <!--  bean复制工具 -->
    <dependency>
        <groupId>ma.glasnost.orika</groupId>
        <artifactId>orika-core</artifactId>
        <version>1.5.2</version>
    </dependency>
```

### 基本使用

```
    MapperFactory mapperFactory = new DefaultMapperFactory.Builder().mapNulls(false).build();

    // 1 List
    mapperFactory.classMap(PersonNameList.class, PersonNameParts.class)
                .field("nameLists[0]", "firstName")
                .field("nameLists[1]", "lastName")
                .register();


    // 2 Map
    Map<String, String> nameMap = new HashMap<>();
    nameMap.put("first", "aaaaaa");
    nameMap.put("last", "111111111");
    mapperFactory.classMap(PersonNameMap.class, PersonNameParts.class)
            .field("nameMap['first']", "firstName")
            .field("nameMap[\"last\"]", "lastName")
            .register();


    ConverterFactory converterFactory = mapperFactory.getConverterFactory();
    // 3 自定义类型转换  全局  全局方式注册：
    converterFactory.registerConverter("jsonConvert", new JsonConfigConvert<BookInfo>());

    mapperFactory.classMap(BookEntity.class, BookDTO.class)
            .field("authorName", "author.name")             // 对象嵌套字段
            .field("authorBirthday", "author.birthday")     // 对象嵌套字段
            .fieldMap("bookInformation", "bookInfo").converter("jsonConvert").add()     // 注册转换器
            .byDefault()    //  默认字段
            .register();

    MapperFacade mapperFacade = mapperFactory.getMapperFacade();    
    BookEntity bookEntity = new BookEntity(
            "Harry",
            "maoyz",
            Date.from(LocalDate.of(1952, Month.MARCH, 11).atStartOfDay(ZoneId.systemDefault()).toInstant()),
            "{\"isbn\": \"9787532754687\", \n \"page\": 279\n }");

    // 使用
    final BookDTO bookDTO = mapperFacade.map(bookEntity, BookDTO.class);
```

### 双向映射

```
    public class JsonConfigConvert<T> extends BidirectionalConverter<T, String> {

        private static final Logger logger = LoggerFactory.getLogger(JsonConfigConvert.class);

        @Override
        public String convertTo(T source, Type<String> destinationType, MappingContext mappingContext) {
            logger.info("********************* JsonConfigConvert convertTo() ,source = {}, destinationType = {} ***********************", source, destinationType);

            return JSON.toJSONString(source.toString());
        }

        @Override
        public T convertFrom(String source, Type<T> destinationType, MappingContext mappingContext) {
            logger.info("********************* JsonConfigConvert convertFrom() ,source = {}, destinationType = {} ***********************", source, destinationType);

            return JSON.parseObject(source, destinationType.getRawType());
        }
    }
```



