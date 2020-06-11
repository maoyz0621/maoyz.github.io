#  Maven

## 配置文件setting.xml

```xml
# 配置仓库 Default: ${user.home}/.m2/repository
<localRepository>/path/to/local/repo</localRepository>

# 镜像
<mirrors>
	<mirror>
		<id>nexus-aliyun</id>
		<mirrorOf>central</mirrorOf>
		<name>Nexus aliyun</name>
		<url>http://maven.aliyun.com/nexus/content/groups/public/</url>
	</mirror>
</mirrors>

<profiles>
	<profile>
		<id>default</id>
		<activation>
			<activeByDefault>true</activeByDefault>
			<jdk>1.8</jdk>
		</activation>
		<repositories>
			<repository>
				<id>spring-milestone</id>
				<name>Spring Milestone Repository</name>
				<url>http://repo.spring.io/milestone</url>
				<releases>
					<enabled>true</enabled>
				</releases>
				<snapshots>
					<enabled>false</enabled>
				</snapshots>
				<layout>default</layout>
			</repository>
			<repository>
				<id>spring-snapshot</id>
				<name>Spring Snapshot Repository</name>
				<url>http://repo.spring.io/snapshot</url>
				<releases>
					<enabled>false</enabled>
				</releases>
				<snapshots>
					<enabled>true</enabled>
				</snapshots>
				<layout>default</layout>
			</repository>
		</repositories>
	</profile>
</profiles>
```



## 生命周期





### 跳过测试

+ -DskipTests，不执行测试用例，但编译测试用例类生成相应的class文件至target/test-classes下。

+ -Dmaven.test.skip=true，不但跳过单元测试的运行，也跳过测试代码的编译。



1. 执行命令
```
mvn package -DskipTests
```

2. pom.xml文件
```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-surefire-plugin</artifactId>  
    <version>3.1</version>  
    <configuration>  
        <skipTests>true</skipTests>  
    </configuration>  
</plugin>
```

3. 执行命令

```
mvn package -Dmaven.test.skip=true
```

4. pom.xml文件中配置：

```xml
<plugin>  
    <groupId>org.apache.maven.plugin</groupId>  
    <artifactId>maven-compiler-plugin</artifactId>  
    <version>3.1</version>  
    <configuration>  
        <skip>true</skip>  
    </configuration>  
</plugin>  
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-surefire-plugin</artifactId>  
    <version>3.1</version>  
    <configuration>  
        <skip>true</skip>  
    </configuration>  
</plugin>
```

