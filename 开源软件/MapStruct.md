# 对象拷贝

安利一款对象拷贝工具`MapStruct`

实现原理：Java动态编译与JSR 269

## 依赖

```xml
<properties>
    <!-- 对象转换 -->
    <org.mapstruct.version>1.4.1.Final</org.mapstruct.version>
</properties>
    
<dependencies>
   <dependency>
       <groupId>org.mapstruct</groupId>
       <artifactId>mapstruct</artifactId>
       <version>1.4.1.Final</version>
   </dependency>
   <dependency>
       <groupId>org.mapstruct</groupId>
       <artifactId>mapstruct-jdk8</artifactId>
       <version>1.3.1.Final</version>
   </dependency>
   <!-- 编译时生成实现类 -->
   <dependency>
       <groupId>org.mapstruct</groupId>
       <artifactId>mapstruct-processor</artifactId>
       <version>1.4.1.Final</version>
   </dependency>
</dependencies>
```



## 实例

```java
package com.myz.opensource.mapstruct;

@Data
public class Car {

    private String make;
    /**
     * 名称不一致
     */
    private int numberOfSeats;
    /**
     * 类型不一致
     * 基本类型的包装类型和String类型之间
     */
    private String weight;
    /**
     * 类型不一致，名称一致
     */
    private CarType type;
    /**
     * 类的包名不一样，名称一致
     */
    private CarType carType;
    /**
     * 类型不一致
     */
    private LocalDateTime createTime;
    /**
     * 枚举类
     */
    private CarEnum carEnum;
}

package com.myz.opensource.mapstruct;

import com.myz.opensource.mapstruct.dto.CarType;
@Data
public class CarDto {
    private String make;
    /**
     * 名称不一致
     */
    private int seatCount;
    /**
     * 类型不一致
     * 基本类型的包装类型和String类型之间
     */
    private Double weight;
    /**
     * 类型不一致
     */
    private String type;

    private CarType carType;
    
    /**
     * 类型不一致
     */
    private String createTime;
    
    /**
     * 枚举类
     */
    private String carEnum;
}

package com.myz.opensource.mapstruct;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class CarType {
    private String type;
}

package com.myz.opensource.mapstruct.dto;、

@Data
@NoArgsConstructor
@AllArgsConstructor
public class CarType {
    private String type;
}

package com.myz.opensource.mapstruct;

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingTarget;
import org.mapstruct.Mappings;
import org.mapstruct.factory.Mappers;

/**
 * 定义这是一个MapStruct对象属性转换接口，在这个类里面规定转换规则
 * 在项目构建时，会自动生成改接口的实现类，这个实现类将实现对象属性值复制
 * componentModel = "spring" 使用Spring依赖注入环境
 *
 * @author maoyz0621 on 2020/11/9
 * @version v1.0
 */
@Mapper(componentModel = "spring")
public interface CarMapper {

    CarMapper INSTANCE = Mappers.getMapper(CarMapper.class);

    /**
     * 用来定义属性复制规则 source 指定源对象属性 target指定目标对象属性
     *
     * @param source
     * @return CarDto
     */
    @Mappings({
            @Mapping(source = "numberOfSeats", target = "seatCount"),
            @Mapping(source = "type.type", target = "type")
    })
    CarDto car2CarDto(Car source);

    /**
     * 集合形式
     * @param source
     * @return
     */
    @Mappings({
            @Mapping(source = "numberOfSeats", target = "seatCount"),
            @Mapping(source = "type.type", target = "type")
    })
    List<CarDto> car2CarDto(List<Car> source);


    @Mappings({
            @Mapping(source = "seatCount", target = "numberOfSeats"),
            @Mapping(source = "type", target = "type.type")
    })
    Car carDto2car(CarDto carDto);

    @Mappings({
            @Mapping(source = "seatCount", target = "numberOfSeats"),
            @Mapping(source = "type", target = "type.type")
    })
    List<Car> carDto2car(List<CarDto> carDto);

    /**
     * 多个参数中的值绑定
     *
     * @param source1 源1
     * @param source2 源2
     * @return 从源1、2中提取出的结果
     */
    @Mappings({
            @Mapping(source = "source1.numberOfSeats", target = "seatCount"),
            @Mapping(source = "source2.type", target = "type"),
            @Mapping(source = "source2", target = "carType")
    })
    CarDto car2CarDto(Car source1, CarType source2);

    /**
     * 通过@MappingTarget来指定目标类是谁（谁的指定属性需要被更新）。@Mapping还是用来定义属性对应规则。
     *
     * @param source
     * @param target
     */
    @Mappings({
            // @Mapping(source = "numberOfSeats", target = "seatCount"),
            @Mapping(source = "type.type", target = "type")
    })
    void convertToCarDto(Car source, @MappingTarget CarDto target);


    @Mappings({
            // @Mapping(source = "seatCount", target = "numberOfSeats"),
            @Mapping(source = "type", target = "type.type")
    })
    void convertToCar(CarDto carDto, @MappingTarget Car car);

}
```

3. 反编译源码：annotations包中

```java
package com.myz.opensource.mapstruct;

import javax.annotation.Generated;
import org.springframework.stereotype.Component;

@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2020-11-11T00:11:44+0800",
    comments = "version: 1.4.1.Final, compiler: javac, environment: Java 11.0.2 (Oracle Corporation)"
)
@Component
public class CarMapperImpl implements CarMapper {

    @Override
    public CarDto car2CarDto(Car source) {
        if ( source == null ) {
            return null;
        }

        CarDto carDto = new CarDto();

        carDto.setSeatCount( source.getNumberOfSeats() );
        carDto.setType( sourceTypeType( source ) );
        carDto.setMake( source.getMake() );
        if ( source.getWeight() != null ) {
            carDto.setWeight( Double.parseDouble( source.getWeight() ) );
        }
        carDto.setCarType( carTypeToCarType( source.getCarType() ) );
        
        // 日期类型
        if ( source.getCreateTime() != null ) {
            carDto1.setCreateTime( DateTimeFormatter.ISO_LOCAL_DATE_TIME.format( source.getCreateTime() ) );
        }
        
        // 枚举类型
        if ( source.getCarEnum() != null ) {
            carDto2.setCarEnum( source.getCarEnum().name() );
        }

        return carDto;
    }

    @Override
    public List<CarDto> car2CarDto(List<Car> source) {
        if ( source == null ) {
            return null;
        }

        List<CarDto> list = new ArrayList<CarDto>( source.size() );
        for ( Car car : source ) {
            list.add( car2CarDto( car ) );
        }

        return list;
    }

    @Override
    public Car carDto2car(CarDto carDto) {
        if ( carDto == null ) {
            return null;
        }

        Car car = new Car();

        car.setType( carDtoToCarType( carDto ) );
        car.setNumberOfSeats( carDto.getSeatCount() );
        car.setMake( carDto.getMake() );
        if ( carDto.getWeight() != null ) {
            car.setWeight( String.valueOf( carDto.getWeight() ) );
        }
        car.setCarType( carTypeToCarType1( carDto.getCarType() ) );

        return car;
    }

    @Override
    public List<Car> carDto2car(List<CarDto> carDto) {
        if ( carDto == null ) {
            return null;
        }

        List<Car> list = new ArrayList<Car>( carDto.size() );
        for ( CarDto carDto1 : carDto ) {
            list.add( carDto2car( carDto1 ) );
        }

        return list;
    }

    @Override
    public CarDto car2CarDto(Car source1, CarType source2) {
        if ( source1 == null && source2 == null ) {
            return null;
        }

        CarDto carDto = new CarDto();

        if ( source1 != null ) {
            carDto.setSeatCount( source1.getNumberOfSeats() );
            carDto.setMake( source1.getMake() );
            if ( source1.getWeight() != null ) {
                carDto.setWeight( Double.parseDouble( source1.getWeight() ) );
            }
        }
        if ( source2 != null ) {
            carDto.setType( source2.getType() );
            carDto.setCarType( carTypeToCarType( source2 ) );
        }

        return carDto;
    }

    @Override
    public void convertToCarDto(Car source, CarDto target) {
        if ( source == null ) {
            return;
        }

        target.setType( sourceTypeType( source ) );
        target.setMake( source.getMake() );
        if ( source.getWeight() != null ) {
            target.setWeight( Double.parseDouble( source.getWeight() ) );
        }
        else {
            target.setWeight( null );
        }
        if ( source.getCarType() != null ) {
            if ( target.getCarType() == null ) {
                target.setCarType( new com.myz.opensource.mapstruct.dto.CarType() );
            }
            carTypeToCarType2( source.getCarType(), target.getCarType() );
        }
        else {
            target.setCarType( null );
        }
    }

    @Override
    public void convertToCar(CarDto carDto, Car car) {
        if ( carDto == null ) {
            return;
        }

        if ( car.getType() == null ) {
            car.setType( new CarType() );
        }
        carDtoToCarType1( carDto, car.getType() );
        car.setMake( carDto.getMake() );
        if ( carDto.getWeight() != null ) {
            car.setWeight( String.valueOf( carDto.getWeight() ) );
        }
        else {
            car.setWeight( null );
        }
        if ( carDto.getCarType() != null ) {
            if ( car.getCarType() == null ) {
                car.setCarType( new CarType() );
            }
            carTypeToCarType3( carDto.getCarType(), car.getCarType() );
        }
        else {
            car.setCarType( null );
        }
    }

    private String sourceTypeType(Car car) {
        if ( car == null ) {
            return null;
        }
        CarType type = car.getType();
        if ( type == null ) {
            return null;
        }
        String type1 = type.getType();
        if ( type1 == null ) {
            return null;
        }
        return type1;
    }

    protected com.myz.opensource.mapstruct.dto.CarType carTypeToCarType(CarType carType) {
        if ( carType == null ) {
            return null;
        }

        com.myz.opensource.mapstruct.dto.CarType carType1 = new com.myz.opensource.mapstruct.dto.CarType();

        carType1.setType( carType.getType() );

        return carType1;
    }

    protected CarType carDtoToCarType(CarDto carDto) {
        if ( carDto == null ) {
            return null;
        }

        CarType carType = new CarType();

        carType.setType( carDto.getType() );

        return carType;
    }

    protected CarType carTypeToCarType1(com.myz.opensource.mapstruct.dto.CarType carType) {
        if ( carType == null ) {
            return null;
        }

        CarType carType1 = new CarType();

        carType1.setType( carType.getType() );

        return carType1;
    }

    protected void carTypeToCarType2(CarType carType, com.myz.opensource.mapstruct.dto.CarType mappingTarget) {
        if ( carType == null ) {
            return;
        }

        mappingTarget.setType( carType.getType() );
    }

    protected void carDtoToCarType1(CarDto carDto, CarType mappingTarget) {
        if ( carDto == null ) {
            return;
        }

        mappingTarget.setType( carDto.getType() );
    }

    protected void carTypeToCarType3(com.myz.opensource.mapstruct.dto.CarType carType, CarType mappingTarget) {
        if ( carType == null ) {
            return;
        }

        mappingTarget.setType( carType.getType() );
    }
}

```

### 属性名称不同

```java
public class Car {
    private String make;
    /**
     * 名称不一致
     */
    private int numberOfSeats;
}

public class CarDto {
    private String make;
    /**
     * 名称不一致
     */
    private int seatCount;
}

import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.MappingTarget;
import org.mapstruct.Mappings;
import org.mapstruct.factory.Mappers;

@Mapper(componentModel = "spring")
public interface CarMapper {

    CarMapper INSTANCE = Mappers.getMapper(CarMapper.class);

    /**
     * 用来定义属性复制规则 source 指定源对象属性 target指定目标对象属性
     *
     * @param source
     * @return CarDto
     */
    @Mappings({
            @Mapping(source = "numberOfSeats", target = "seatCount"
    })
    CarDto car2CarDto(Car source);

    List<CarDto> car2CarDto(List<Car> source);


    @Mappings({
            @Mapping(source = "seatCount", target = "numberOfSeats")
    })
    Car carDto2car(CarDto carDto);

    List<Car> carDto2car(List<CarDto> carDto);
}
```

- 实现类

```java
package com.myz.opensource.mapstruct;

import javax.annotation.Generated;
import org.springframework.stereotype.Component;

@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2020-11-11T00:11:44+0800",
    comments = "version: 1.4.1.Final, compiler: javac, environment: Java 11.0.2 (Oracle Corporation)"
)
@Component
public class CarMapperImpl implements CarMapper {

    @Override
    public CarDto car2CarDto(Car source) {
        if ( source == null ) {
            return null;
        }

        CarDto carDto = new CarDto();

        carDto.setSeatCount( source.getNumberOfSeats() );
        carDto.setMake( source.getMake() );
        return carDto;
    }

    @Override
    public List<CarDto> car2CarDto(List<Car> source) {
        if ( source == null ) {
            return null;
        }

        List<CarDto> list = new ArrayList<CarDto>( source.size() );
        for ( Car car : source ) {
            list.add( car2CarDto( car ) );
        }

        return list;
    }

    @Override
    public Car carDto2car(CarDto carDto) {
        if ( carDto == null ) {
            return null;
        }

        Car car = new Car();
        car.setNumberOfSeats( carDto.getSeatCount() );
        car.setMake( carDto.getMake() );

        return car;
    }

    @Override
    public List<Car> carDto2car(List<CarDto> carDto) {
        if ( carDto == null ) {
            return null;
        }

        List<Car> list = new ArrayList<Car>( carDto.size() );
        for ( CarDto carDto1 : carDto ) {
            list.add( carDto2car( carDto1 ) );
        }

        return list;
    }
}
```



### 子对象映射

```java
public class Car {
    private String make;
    /**
     * 类型不一致，名称一致
     */
    private CarType type;
}

public class CarSubDto {
    private String make;
    /**
     * 类型不一致
     */
    private String type;
}
// 子类
public class CarType {
    private String type;
}
```

- 对象映射

```java
@Mapper(componentModel = "spring")
public interface CarSubConverter extends IPairConverter<Car, CarSubDto> {
    /**
     * 子对象
     */
    @Mappings({
            @Mapping(source = "type.type", target = "type")
    })
    @Override
    CarSubDto to(Car src);

    @Mappings({
            // @Mapping(source = "type", target = "type.type", ignore = true)
            @Mapping(source = "type", target = "type.type")
    })
    @Override
    Car back(CarSubDto dest);
}
```

- 实现类

```java
@Component
public class CarSubConverterImpl implements CarSubConverter {

    @Override
    public CarSubDto to(Car src) {
        if ( src == null ) {
            return null;
        }

        CarSubDto carSubDto = new CarSubDto();

        carSubDto.setType( srcTypeType( src ) );
        carSubDto.setMake( src.getMake() );
        return carSubDto;
    }


    @Override
    public Car back(CarSubDto dest) {
        if ( dest == null ) {
            return null;
        }

        Car car = new Car();
        car.setType( carSubDtoToCarType( dest ) );
        car.setMake( dest.getMake() );

        return car;
    }
	// 处理子对象的实现类
    private String srcTypeType(Car car) {
        if ( car == null ) {
            return null;
        }
        CarType type = car.getType();
        if ( type == null ) {
            return null;
        }
        String type1 = type.getType();
        if ( type1 == null ) {
            return null;
        }
        return type1;
    }

    protected CarType carSubDtoToCarType(CarSubDto carSubDto) {
        if ( carSubDto == null ) {
            return null;
        }

        CarType carType = new CarType();
        carType.setType( carSubDto.getType() );
        return carType;
    }
}
```

### 多个源类

```java
@Mapper(componentModel = "spring")
public interface CarMultSourceConverter {

    /**
     * 多个参数中的值绑定
     *
     * @param source1 源1
     * @param source2 源2
     * @return 从源1、2中提取出的结果
     */
    @Mappings({
            @Mapping(source = "source1.numberOfSeats", target = "seatCount"),
            @Mapping(source = "source2.type", target = "type"),
            @Mapping(source = "source2", target = "carType")
    })
    CarDto car2CarDto(Car source1, CarType source2);
}
```

- 实现类

```java
@Component
public class CarMultSourceConverterImpl implements CarMultSourceConverter {

    @Override
    public CarDto car2CarDto(Car source1, CarType source2) {
        if ( source1 == null && source2 == null ) {
            return null;
        }

        CarDto carDto = new CarDto();
        if ( source1 != null ) {
            carDto.setSeatCount( source1.getNumberOfSeats() );
            carDto.setMake( source1.getMake() );
            if ( source1.getWeight() != null ) {
                carDto.setWeight( Double.parseDouble( source1.getWeight() ) );
            }
        }
        if ( source2 != null ) {
            carDto.setType( source2.getType() );
            carDto.setCarType( carTypeToCarType( source2 ) );
        }

        return carDto;
    }

    protected com.myz.opensource.mapstruct.dto.CarType carTypeToCarType(CarType carType) {
        if ( carType == null ) {
            return null;
        }
        com.myz.opensource.mapstruct.dto.CarType carType1 = new com.myz.opensource.mapstruct.dto.CarType();
        carType1.setType( carType.getType() );
        return carType1;
    }
}
```



### 数据类型转换

#### 基本类型和包装类型

#### 基本类型、包装类型和String

#### 枚举和String

#### Java大数类型

(`java.math.BigInteger`， `java.math.BigDecimal`) 和Java基本类型(包括其包装类)与`String`之间



### 日期格式转换

```java
public class Car {
    private String make;
    /**
     * 类型不一致 LocalDateTime
     */
    private LocalDateTime createTime;
    /**
     * 类型不一致 Date
     */
    private Date updateTime;
}

public class CarDiffTypeDTO {
    private String make;
    /**
     * 类型不一致
     */
    private String createTime;
    private String updateTime;
}
```

- 对象映射，对于日期类型：LocalDateTime和Date，指定dateFormat

```java
@Mapper(componentModel = "spring")
public interface CarDiffDateTypeConverter extends IPairConverter<Car, CarDiffTypeDTO> {
    CarDiffDateTypeConverter INSTANCE = Mappers.getMapper(CarDiffDateTypeConverter.class);

    /**
     * 定义时间格式化
     *
     * @param src
     * @return
     */
    @Mappings({
            @Mapping(source = "createTime", target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss"),
            @Mapping(source = "updateTime", target = "updateTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    })
    @Override
    CarDiffTypeDTO to(Car src);

    @Override
    List<CarDiffTypeDTO> to(List<Car> srcList);

    @Mappings({
            @Mapping(source = "createTime", target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss"),
            @Mapping(source = "updateTime", target = "updateTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    })
    @Override
    Car back(CarDiffTypeDTO dest);

    @Override
    List<Car> back(List<CarDiffTypeDTO> destList);
}
```

- 实现类

```java
public class CarDiffDateTypeConverterImpl implements CarDiffDateTypeConverter {

    @Override
    public CarDiffTypeDTO to(Car src) {
        if ( src == null ) {
            return null;
        }

        CarDiffTypeDTO carDiffTypeDTO = new CarDiffTypeDTO();

        if ( src.getCreateTime() != null ) {
            carDiffTypeDTO.setCreateTime( DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ).format( src.getCreateTime() ) );
        }
        if ( src.getUpdateTime() != null ) {
            carDiffTypeDTO.setUpdateTime( new SimpleDateFormat( "yyyy-MM-dd HH:mm:ss" ).format( src.getUpdateTime() ) );
        }
        carDiffTypeDTO.setMake( src.getMake() );

        return carDiffTypeDTO;
    }

    @Override
    public List<CarDiffTypeDTO> to(List<Car> srcList) {
        if ( srcList == null ) {
            return null;
        }

        List<CarDiffTypeDTO> list = new ArrayList<CarDiffTypeDTO>( srcList.size() );
        for ( Car car : srcList ) {
            list.add( to( car ) );
        }

        return list;
    }

    @Override
    public Car back(CarDiffTypeDTO dest) {
        if ( dest == null ) {
            return null;
        }

        Car car = new Car();

        if ( dest.getCreateTime() != null ) {
            car.setCreateTime( LocalDateTime.parse( dest.getCreateTime(), DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ) ) );
        }
        try {
            if ( dest.getUpdateTime() != null ) {
                car.setUpdateTime( new SimpleDateFormat( "yyyy-MM-dd HH:mm:ss" ).parse( dest.getUpdateTime() ) );
            }
        }
        catch ( ParseException e ) {
            throw new RuntimeException( e );
        }
        car.setMake( dest.getMake() );

        return car;
    }

    @Override
    public List<Car> back(List<CarDiffTypeDTO> destList) {
        if ( destList == null ) {
            return null;
        }

        List<Car> list = new ArrayList<Car>( destList.size() );
        for ( CarDiffTypeDTO carDiffTypeDTO : destList ) {
            list.add( back( carDiffTypeDTO ) );
        }

        return list;
    }
}
```



### 数字格式转换

` @Mapping(source = "price", target = "price", numberFormat = "$#.00")`

### 枚举映射

```java
public class Car {
    private String make;
    /**
     * 枚举类
     */
    private CarEnum carEnum;
}

public class CarDiffEnumDto {
    private String make;
    /**
     * 枚举类
     */
    private String carEnum;
}

public enum CarEnum {
    CAR("1", "汽车"),
    BIKE("2", "自行车");
    ...
}
```

- 映射

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2021-08-21T17:23:05+0800",
    comments = "version: 1.4.1.Final, compiler: javac, environment: Java 1.8.0_41 (Oracle Corporation)"
)
public class CarDiffEnumConverterImpl implements CarDiffEnumConverter {

    @Override
    public CarDiffEnumDto to(Car src) {
        if ( src == null ) {
            return null;
        }

        CarDiffEnumDto carDiffEnumDto = new CarDiffEnumDto();

        carDiffEnumDto.setMake( src.getMake() );
        if ( src.getCarEnum() != null ) {
            carDiffEnumDto.setCarEnum( src.getCarEnum().name() );
        }

        return carDiffEnumDto;
    }

    ...

    @Override
    public Car back(CarDiffEnumDto dest) {
        if ( dest == null ) {
            return null;
        }

        Car car = new Car();

        car.setMake( dest.getMake() );
        if ( dest.getCarEnum() != null ) {
            car.setCarEnum( Enum.valueOf( CarEnum.class, dest.getCarEnum() ) );
        }

        return car;
    }

   ...
}
```

### 继承逆向配置

`@InheritInverseConfiguration`，映射前后的字段相同，但是源属性字段与目标属性字段是相反的

```java
@Mapper(componentModel = "spring")
public interface CarInheritInverseConfigurationMapper {

    @Mappings({
            @Mapping(source = "numberOfSeats", target = "seatCount"),
            @Mapping(source = "type.type", target = "type")
    })
    CarDto to(Car source);

    List<CarDto> to(List<Car> source);

    @InheritInverseConfiguration
    Car back(CarDto carDto);

    List<Car> back(List<CarDto> carDto);
}
```

- 实现类

```java
@Component
public class CarInheritInverseConfigurationMapperImpl implements CarInheritInverseConfigurationMapper {

    @Override
    public CarDto to(Car source) {
        if ( source == null ) {
            return null;
        }

        CarDto carDto = new CarDto();
        carDto.setSeatCount( source.getNumberOfSeats() );
        carDto.setType( sourceTypeType( source ) );
        carDto.setMake( source.getMake() );
        if ( source.getWeight() != null ) {
            carDto.setWeight( Double.parseDouble( source.getWeight() ) );
        }
        carDto.setCarType( carTypeToCarType( source.getCarType() ) );

        return carDto;
    }

    @Override
    public List<CarDto> to(List<Car> source) {
        ...
    }

    @Override
    public Car back(CarDto carDto) {
        if ( carDto == null ) {
            return null;
        }

        Car car = new Car();

        car.setType( carDtoToCarType( carDto ) );
        car.setNumberOfSeats( carDto.getSeatCount() );
        car.setMake( carDto.getMake() );
        if ( carDto.getWeight() != null ) {
            car.setWeight( String.valueOf( carDto.getWeight() ) );
        }
        car.setCarType( carTypeToCarType1( carDto.getCarType() ) );

        return car;
    }

    @Override
    public List<Car> back(List<CarDto> carDto) {
        ...
    }

    private String sourceTypeType(Car car) {
       ...
    }

    protected com.myz.opensource.mapstruct.dto.CarType carTypeToCarType(CarType carType) {
       ...
    }

    protected CarType carDtoToCarType(CarDto carDto) {
       ...
    }

    protected CarType carTypeToCarType1(com.myz.opensource.mapstruct.dto.CarType carType) {
        ...
    }
```

### 添加默认值

设置默认值`constant`、`defaultValue`

```java
@Mapper(componentModel = "spring", imports = {LocalDateTime.class})
public interface CarDefaultValConverter extends IPairConverter<Car, CarDiffTypeDTO> {
    /**
     * 定义时间格式化
     */
    @Mappings({
            @Mapping(target = "make", constant = "ABC"),
            @Mapping(source = "createTime", target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss", defaultExpression = "java(LocalDateTime.now().toString())"),
            @Mapping(source = "updateTime", target = "updateTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    })
    @Override
    CarDiffTypeDTO to(Car src);

    @Override
    List<CarDiffTypeDTO> to(List<Car> srcList);

    @InheritInverseConfiguration
    @Override
    Car back(CarDiffTypeDTO dest);

    @Override
    List<Car> back(List<CarDiffTypeDTO> destList);
}
```

- 实现类

```java
@Component
public class CarDefaultValConverterImpl implements CarDefaultValConverter {

    @Override
    public CarDiffTypeDTO to(Car src) {
        if ( src == null ) {
            return null;
        }

        CarDiffTypeDTO carDiffTypeDTO = new CarDiffTypeDTO();

        if ( src.getCreateTime() != null ) {
            carDiffTypeDTO.setCreateTime( DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ).format( src.getCreateTime() ) );
        }
        else {
            carDiffTypeDTO.setCreateTime( LocalDateTime.now().toString() );
        }
        if ( src.getUpdateTime() != null ) {
            carDiffTypeDTO.setUpdateTime( new SimpleDateFormat( "yyyy-MM-dd HH:mm:ss" ).format( src.getUpdateTime() ) );
        }
		// 设置的默认值
        carDiffTypeDTO.setMake( "ABC" );
        return carDiffTypeDTO;
    }

    @Override
    public Car back(CarDiffTypeDTO dest) {
        if ( dest == null ) {
            return null;
        }

        Car car = new Car();

        if ( dest.getCreateTime() != null ) {
            car.setCreateTime( LocalDateTime.parse( dest.getCreateTime(), DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ) ) );
        }
        try {
            if ( dest.getUpdateTime() != null ) {
                car.setUpdateTime( new SimpleDateFormat( "yyyy-MM-dd HH:mm:ss" ).parse( dest.getUpdateTime() ) );
            }
        }
        catch ( ParseException e ) {
            throw new RuntimeException( e );
        }
        car.setMake( dest.getMake() );

        return car;
    }
}
```



### 添加表达式

设置表达式`expression`、`defaultExpression`

```java
// 使用表达式 expression defaultExpression
// imports = {LocalDateTime.class, UUID.class}
@Mapper(componentModel = "spring", imports = {LocalDateTime.class, UUID.class})
public interface CarExpressionConverter extends IPairConverter<Car, CarDiffTypeDTO> {
    /**
     * 添加表达式 expression defaultExpression
     */
    @Mappings({
            @Mapping(target = "make", expression = "java(UUID.randomUUID().toString())"),
            @Mapping(source = "createTime", target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss", defaultExpression = "java(LocalDateTime.now().toString())"),
            @Mapping(source = "updateTime", target = "updateTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    })
    @Override
    CarDiffTypeDTO to(Car src);

    @Override
    List<CarDiffTypeDTO> to(List<Car> srcList);

    @InheritInverseConfiguration
    @Override
    Car back(CarDiffTypeDTO dest);

    @Override
    List<Car> back(List<CarDiffTypeDTO> destList);
}
```

- 实现类

```java
@Component
public class CarExpressionConverterImpl implements CarExpressionConverter {

    @Override
    public CarDiffTypeDTO to(Car src) {
        if ( src == null ) {
            return null;
        }

        CarDiffTypeDTO carDiffTypeDTO = new CarDiffTypeDTO();

        if ( src.getCreateTime() != null ) {
            carDiffTypeDTO.setCreateTime( DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ).format( src.getCreateTime() ) );
        }
        else {
            // 设置的默认表达式
            carDiffTypeDTO.setCreateTime( LocalDateTime.now().toString() );
        }
        if ( src.getUpdateTime() != null ) {
            carDiffTypeDTO.setUpdateTime( new SimpleDateFormat( "yyyy-MM-dd HH:mm:ss" ).format( src.getUpdateTime() ) );
        }
		// 设置的默认表达式
        carDiffTypeDTO.setMake( UUID.randomUUID().toString() );

        return carDiffTypeDTO;
    }

    @Override
    public Car back(CarDiffTypeDTO dest) {
        if ( dest == null ) {
            return null;
        }

        Car car = new Car();

        if ( dest.getCreateTime() != null ) {
            car.setCreateTime( LocalDateTime.parse( dest.getCreateTime(), DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ) ) );
        }
        try {
            if ( dest.getUpdateTime() != null ) {
                car.setUpdateTime( new SimpleDateFormat( "yyyy-MM-dd HH:mm:ss" ).parse( dest.getUpdateTime() ) );
            }
        }
        catch ( ParseException e ) {
            throw new RuntimeException( e );
        }
        car.setMake( dest.getMake() );

        return car;
    }
}
```

### 集合映射

#### List映射

#### Set映射

#### Map映射



https://zhuanlan.zhihu.com/p/368731266