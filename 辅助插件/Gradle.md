# Gradle

## 简介

全新的java构建工具



## 安装

### 配置环境变量

1. 首先右键“此电脑”-->"属性"-->"高级系统设置"-->"环境变量"-->"系统变量"
2. 新建GRADLE_HOME的变量，将Gradle的安装目录路径（E:\WorkWare\gradle-6.8.1\bin）设置为变量值
3. 修改path变量，将Gradle的bin添加到path变量值中：**;%GRADLE_HOME%\bin** , 注意：前面的分号
4. 新建GRADLE_USER_HOME的变量，将Maven的仓库路径（E:\WorkSpace\repository）设置为变量值



## 声明周期

1. **初始化阶段**

   通过setting.gradle文件判断哪些项目需要被构建，创建Project项目

2. **配置阶段**

   执行build.gradle脚本，生成要执行的task

3. **执行阶段**

   执行task，进行构建工作

23:51:45: Executing task 't1'...

初始化阶段 settingsEvaluated
初始化阶段 projectsLoaded

> Configure project :
> 初始化阶段 beforeProject
> task1
> 配置阶段 afterProject
> 配置阶段 afterEvaluate
> 配置阶段 projectsEvaluated
> 配置阶段 whenReady

> Task :t1
> 执行阶段 beforeTask
> task1 execute doFirst
> task1 execute doLast
> 执行阶段 afterTask
> 执行阶段 buildFinished

BUILD SUCCESSFUL 

settings.gradle

```groovy
// 项目构建之前的钩子方法
gradle.settingsEvaluated {
    println("初始化阶段 settingsEvaluated")
}

gradle.projectsLoaded {
    println("初始化阶段 projectsLoaded")
}

gradle.beforeProject {
    println("初始化阶段 beforeProject")
}
```

build.gradle

```groovy
task t1 {
    println("task1")

    // 动作代码
    doLast {
        println("task1 execute doLast")
    }

    doFirst {
        println("task1 execute doFirst")
    }
}


// 配置阶段 start
// Configure project
gradle.afterProject {
    println("配置阶段 afterProject")
}
project.afterEvaluate {
    println("配置阶段 afterEvaluate")
}
gradle.projectsEvaluated {
    println("配置阶段 projectsEvaluated")
}
gradle.taskGraph.whenReady {
    println("配置阶段 whenReady")
}
project.beforeEvaluate {
    println("配置阶段 beforeEvaluate")
}

project.afterEvaluate {
    println("配置阶段 afterEvaluate")
}
// 配置阶段 end

// 执行阶段 start
// Task
gradle.taskGraph.beforeTask {
    println("执行阶段 beforeTask")
}

gradle.taskGraph.afterTask {
    println("执行阶段 afterTask")
}

gradle.buildFinished {
    println("执行阶段 buildFinished")
}
// 执行阶段 end
```

## 依赖管理

依赖配置

- group：通常标识一个组织、公司或者项目。如org.jsoup

- name: 一个工件的名称唯一的描述了依赖。如：:jsoup

- version：一个类库的版本号。如1.9.2

  > 第一种（只添加一种依赖，全部写全）
  >  compile "group":"org.springframework","name":"spring-beans","version":"5.2.5.RELEASE"
  > 第二种（只添加一种依赖，简写）
  > compile "org.springframework:spring-core:5.2.5.RELEASE"
  >
  > 第三种（添加多种依赖）
  > compile(
  >         "org.springframework:spring-core:5.2.5.RELEASE",
  >         "org.springframework:spring-beans:5.2.5.RELEASE"
  > )

```groovy
dependencies {
    testCompile group: 'junit', name: 'junit', version: '4.12'
  
    implementation  // 默认的scope，编译和运行均包含在内，不会将自己类库的依赖暴露给类库的使用者
    api // 和implementation类似，都是编译和运行时都可见的依赖。允许我们将自己类库的依赖暴露给类库的使用者
    compileOnly  // 编译时可见
    runtimeOnly  // 运行时可见
    
    testImplementation  // 在测试编译时和运行时可见，类似于Maven的test作用域。
    testCompileOnly  // 作用于测试编译时
    testRuntimeOnly  // 作用于测试运行时
    
    
    compile(弃用) 
    compile(project(":gradle-project-common"))  // 依赖工程项目
}
```

## build.gradle

### buildscript

> all buildscript {} blocks must appear before any plugins {} blocks in the script

```groovy
// 构建脚本 all buildscript {} blocks must appear before any plugins {} blocks in the script
buildscript {
    println("构建脚本 buildscript")

    repositories {
        maven {
            url "https://plugins.gradle.org/m2/"
        }
        mavenLocal()
        maven {
            url = uri('https://maven.aliyun.com/repository/public')
        }

        maven {
            url = uri('https://maven.aliyun.com/repository/spring')
        }

        maven {
            url = uri('https://maven.aliyun.com/repository/spring-plugin')
        }

        maven {
            url = uri('https://repo.maven.apache.org/maven2/')
        }
    }

    dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:2.2.6.RELEASE"
    }

    // 定义SpringCloud版本，统一规定springboot的版本
    ext {
        set('springCloudVersion', "Hoxton.SR3")
        set('springBootVersion', "2.2.6.RELEASE")
        set('version', "1.0.0")
    }

    dependencies {
        // 用来打包
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}
```

### plugins

```groovy
plugins {
    id 'java'
    // springboot插件，加入版本，那么Spring相关依赖，则自动加入(当使用其他插件的时候，还会自动加载插件所带的任务) 但是不应用
    id 'org.springframework.boot' version '2.2.6.RELEASE' apply false
    // 第一种引入方式：写在此处，需要手动设置依赖管理的版本，否则无法执行（手动指定版本，好处是插件集中管理在plugins里面）
    id 'io.spring.dependency-management' version '1.0.9.RELEASE'
}
```

### apply

```groovy
apply plugin: 'java'
apply plugin: 'idea'
// 使用spring boot
//apply plugin: "org.springframework.boot"
// 第二种引入方式：应用依赖管理插件，自动给插件追加版本号（建议使用此配置）
//apply plugin: 'io.spring.dependency-management'
```

### allprojects

```groovy
allprojects {
    // 统一引入java插件和版本号
    apply plugin: 'java'
    sourceCompatibility = JavaVersion.VERSION_11
    targetCompatibility = JavaVersion.VERSION_11

    group "${project.group}"
    version '1.0-SNAPSHOT'

    tasks.withType(JavaCompile) {
        options.encoding = "UTF-8"
    }

    // java编译的时候缺省状态下会因为中文字符而失败
    [compileJava, compileTestJava, javadoc]*.options*.encoding = 'UTF-8'

    // 统一
    repositories {
        maven {
            url "https://plugins.gradle.org/m2/"
        }
        mavenLocal()
        maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }
        mavenCentral()
    }

    ext {
        /***常见或主要第三方依赖版本号定义 begin***/
        globalSpringVersion = "5.2.5.RELEASE"
        globalSpringDataJpaVersion = "2.1.2.RELEASE"
        globalSpringBootVersion = '2.2.6.RELEASE'
        globalRedisClientVersion = "2.10.1"
        globalSpringCloudVersion = "Hoxton.SR3"
        globalSpringCloudAlibabaVersion = "2.1.0.RELEASE"
        globalFastJsonVersion = "1.2.54"
        globalMyBatisVersion = "3.4.6"
        globalMyBatisSpringVersion = "2.1.2"
        globalGoogleGuavaVersion = "28.1-jre"
        globalDom4jVersion = "1.6.1"
        globalJavaMailVersion = "1.4.7"
        globalJsoupVersion = "1.11.3" //--一个过滤html危险字符串api，用于web安全
        globalQuartzVersion = "2.3.0"
        globalFlexmarkVersion = "0.34.32" //--java对markdown语法的解释以及翻译api
        globalPostgresqlJdbcDriverVersion = "42.2.5"
        globalQiniuSdkVersion = "7.2.18"//--七牛上传下载客户端sdk
        globalApacheAntVersion = "1.10.5"
        globalGoogleZXingVersion = "3.3.3"
        globalFastdfsClientVersion = "1.27"
        globalLog4jVersion = "1.2.17"
        globalSlf4jVersion = "1.7.25"
        globalRedissonSpringVersion = "3.11.0"
        globalMybatisPlusVersion = "3.1.0"
        globalElasticsearchVersion = "7.9.3"
        globalLombokVersion = "1.18.6"
        /***常见或主要第三方依赖版本号定义 end***/


        /****常见或者程序主要引用依赖定义 begin****/
        //--这个是spring boot要直接compile进去的框架。
        ref4SpringBoot = [
                /***spring boot 相关依赖***/
                "org.springframework.boot:spring-boot:$globalSpringBootVersion",
                "org.springframework.boot:spring-boot-starter:$globalSpringBootVersion",
                "org.springframework.boot:spring-boot-starter-web:$globalSpringBootVersion",
                "org.springframework.boot:spring-boot-starter-freemarker:$globalSpringBootVersion",
                "org.springframework.boot:spring-boot-devtools:$globalSpringBootVersion"
        ]
        //--这个是spring boot要compileOnly的类库
        ref4SpringBootProvided = [
                "org.springframework.boot:spring-boot-dependencies:$globalSpringBootVersion",
        ]
        //--这个是spring boot的测试框架，用testCompile导入
        ref4SpringBootTest = [
                "org.springframework.boot:spring-boot-starter-test:$globalSpringBootVersion"
        ]
        //--spring框架api
        ref4SpringFramework = [
                "org.springframework:spring-web:$globalSpringVersion",
                "org.springframework:spring-webmvc:$globalSpringVersion",
                "org.springframework:spring-context-support:$globalSpringVersion",
                "org.springframework.data:spring-data-jpa:$globalSpringDataJpaVersion",
                "org.springframework:spring-test:$globalSpringVersion"
        ]

        //--jsp&servlet等javaweb容器api，通常都用 compileOnly引用的。
        ref4JspAndServletApi = [
                "javax.servlet:javax.servlet-api:3.1.0",
                "javax.servlet.jsp:jsp-api:2.2",
                "javax.servlet.jsp.jstl:javax.servlet.jsp.jstl-api:1.2.1"
        ]

        //--jstl等java web的tag标准api，引入的话要用compile
        ref4Jstl = [
                'taglibs:standard:1.1.2',
                'jstl:jstl:1.2'
        ]
        //--mybatis
        ref4MyBatis = [
                "org.mybatis:mybatis:$globalMyBatisVersion"
        ]
        //--这是apache common 类库引用的地址
        ref4ApacheCommons = [
                'commons-lang:commons-lang:2.6',
                'commons-logging:commons-logging:1.2',
                'commons-io:commons-io:2.5',
                'commons-fileupload:commons-fileupload:1.3.2',
                'commons-codec:commons-codec:1.10',
                'commons-beanutils:commons-beanutils:1.9.3',
                'commons-httpclient:commons-httpclient:3.1',
                'org.apache.httpcomponents:fluent-hc:4.3.6',
                'org.apache.httpcomponents:httpclient:4.5.3',
                'org.apache.httpcomponents:httpclient-cache:4.5.3',
                'org.apache.httpcomponents:httpcore:4.4.8',
                'org.apache.httpcomponents:httpmime:4.5.3',
                'org.apache.curator:curator-framework:4.0.1',
                'org.jfree:jfreechart:1.0.19',
                'org.apache.velocity:velocity:1.7',
                'org.apache.poi:poi:3.16'
        ]
        //--redis client
        ref4RedisClient = ["redis.clients:jedis:$globalRedisClientVersion"]

        //--这是阿里云短信引用的第三方类库
        ref4AliYunSms = [
                'com.aliyun:aliyun-java-sdk-core:3.2.8',
                'com.aliyun:aliyun-java-sdk-dysmsapi:1.1.0'
        ]
        //--阿里云图片裁剪
        ref4AliSimpleImage = [
                'com.alibaba:simpleimage:1.2.3'
        ]
        //--阿里fast json引用地址
        ref4FastJson = ["com.alibaba:fastjson:$globalFastJsonVersion"]
        //--json-lib引用地址
        ref4JsonLib = ["net.sf.json-lib:json-lib:2.4:jdk15"]
        //--jdom1&jdom2以及相关api
        ref4Jdom = [
                'org.jdom:jdom2:2.0.6',
                'org.jdom:jdom:1.1.3',
                'joda-time:joda-time:2.9.7'
        ]

        //--google guava
        ref4GoogleGuava = ["com.google.guava:guava:$globalGoogleGuavaVersion"]
        //--dom4j
        ref4Dom4j = ["dom4j:dom4j:$globalDom4jVersion"]

        ref4JavaMail = ["javax.mail:mail:$globalJavaMailVersion"]

        ref4Jsoup = ["org.jsoup:jsoup:$globalJsoupVersion"]

        ref4Quartz = [
                "org.quartz-scheduler:quartz:$globalQuartzVersion",
                "org.quartz-scheduler:quartz-jobs:$globalQuartzVersion"
        ]


        ref4Flexmark = [
                "com.vladsch.flexmark:flexmark-all:$globalFlexmarkVersion"
        ]

        ref4PostgresqlJdbcDriver = [
                "org.postgresql:postgresql:$globalPostgresqlJdbcDriverVersion"
        ]

        ref4QiuniuSdkVersion = [
                "com.qiniu:qiniu-java-sdk:$globalQiniuSdkVersion"
        ]

        ref4ApacheAnt = ["org.apache.ant:ant:$globalApacheAntVersion"]

        //--二维码
        ref4ZXing = [
                "com.google.zxing:core:$globalGoogleZXingVersion",
                "com.google.zxing:javase:$globalGoogleZXingVersion"
        ]
        ref4FastdfsClient = ["cn.bestwu:fastdfs-client-java:$globalFastdfsClientVersion"]


        ref4Log4j = ["log4j:log4j:$globalLog4jVersion"]

        ref4Slf4jToLog4j = ["org.slf4j:slf4j-log4j12:$globalSlf4jVersion"]

        /****常见或者程序主要引用依赖定义 end****/
    }
}
```

### subprojects

```groovy
subprojects {
    // spring boot 插件
    apply plugin: 'org.springframework.boot'
    // 必须添加spring依赖管理
    apply plugin: 'io.spring.dependency-management'
    // 统一
    repositories {
        mavenLocal()
        // 自定义仓库
        maven {
            url "http://maven.aliyun.com/nexus/content/groups/public/"
        }
        mavenCentral()
    }

    dependencies {
        runtimeOnly 'org.springframework.boot:spring-boot-devtools'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
        implementation group: 'org.apache.commons', name: 'commons-lang3', version: '3.7'
        implementation group: 'org.projectlombok', name: 'lombok', version: '1.18.6'
        implementation group: 'com.myz', name: 'springboot2-common', version: '0.0.1-SNAPSHOT'
        implementation group: 'org.aspectj', name: 'aspectjweaver', version: '1.9.5'
    }

    dependencyManagement {
        imports {
            // SpringCloud依赖
            mavenBom "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
        }
    }
}
```

### configurations

```groovy
// 强制依赖版本
configurations.all {
    resolutionStrategy.eachDependency {
        DependencyResolveDetails details ->
            if (details.requested.group == 'org.slf4j') {
                details.useVersion '1.7.20'
            }
    }
}
```

### repositories

```groovy
repositories {
    mavenLocal()
    maven { url "http://maven.aliyun.com/nexus/content/groups/public/" }
    mavenCentral()
}
```

### dependencies

```
dependencies {
    implementation group: 'org.apache.commons', name: 'commons-lang3', version: '3.7'
    implementation group: 'org.projectlombok', name: 'lombok', version: '1.18.6'
    implementation group: 'com.myz', name: 'springboot2-common', version: '0.0.1-SNAPSHOT'
    implementation group: 'org.aspectj', name: 'aspectjweaver', version: '1.9.5'
}
```

### task

## gradle.properties

