对象拷贝

安利一款对象拷贝工具MapStruct

1. 依赖

```xml
<properties>
    <!-- 对象转换 -->
    <org.mapstruct.version>1.4.1.Final</org.mapstruct.version>
</properties>
    
<dependencies>
	<dependency>
    	<groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${org.mapstruct.version}</version>
                    </path>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>1.18.10</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

2. 实例
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





