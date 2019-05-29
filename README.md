# 服务器端口号
server.port=8888
# 是否生成ddl语句
spring.jpa.generate-ddl=false
# 是否打印sql语句
spring.jpa.show-sql=true

spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
# 自动生成ddl，由于指定了具体的ddl，此处设置为none
spring.jpa.hibernate.ddl-auto=update
spring.jpa.properties.hibernate.show_sql=true
spring.jpa.properties.hibernate.use_sql_comments=true
spring.jpa.properties.hibernate.format_sql=true


# 使用H2数据库
spring.datasource.platform=h2
spring.datasource.url=jdbc:h2:mem:h2test;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.username=mao
spring.datasource.password=
spring.datasource.driver-class-name= org.h2.Driver
# 指定生成数据库的schema文件位置
#spring.datasource.schema=classpath:schema.sql
# 指定插入数据库语句的脚本位置
spring.datasource.data=classpath:data.sql


spring.h2.console.enabled=true
spring.h2.console.path=/h2
spring.h2.console.settings.trace=false
spring.h2.console.settings.web-allow-others=false


# 配置日志打印信息
logging.level.root=INFO
logging.level.org.hibernate=INFO
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
logging.level.org.hibernate.type.descriptor.sql.BasicExtractor=TRACE


package com.example.paytm.pojo;

import lombok.Data;
import lombok.ToString;

import javax.persistence.*;

@Entity
@Table(name = "t_user")
@Data
@ToString
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "first_name")
    private String firstName;

    @Column(name = "last_name")
    private String lastName;

    protected User() {
    }


    public User(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }


}
package com.example.paytm.dao;

import com.example.paytm.pojo.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

}
