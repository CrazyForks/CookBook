# Maven 多模块项目实践

## 1. 多模块项目概述

```
为什么需要多模块项目？

┌─────────────────────────────────────────────────────────────┐
│                单模块 vs 多模块                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  单模块问题:                                                │
│  ├── 代码量大，编译慢                                       │
│  ├── 模块边界不清晰                                         │
│  ├── 依赖管理混乱                                           │
│  └── 无法复用公共代码                                       │
│                                                             │
│  多模块优势:                                                │
│  ├── 模块职责单一，边界清晰                                 │
│  ├── 只编译变更模块，构建快                                 │
│  ├── 依赖版本统一管理                                       │
│  ├── 公共代码抽取复用                                       │
│  └── 支持团队并行开发                                       │
│                                                             │
│  典型模块划分:                                              │
│  ├── common:   公共工具、常量、异常                         │
│  ├── model:    实体类、DTO、枚举                            │
│  ├── dao:      数据访问层                                   │
│  ├── service:  业务逻辑层                                   │
│  ├── api:      对外接口（Feign Client）                     │
│  ├── web:      Web 控制器、配置                             │
│  └── admin:    管理后台                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 项目结构设计

### 2.1 典型微服务项目结构

```
my-project/
├── pom.xml                         # 父 POM（聚合 + 继承）
├── my-project-common/              # 公共模块
│   ├── pom.xml
│   └── src/
├── my-project-model/               # 实体模块
│   ├── pom.xml
│   └── src/
├── my-project-dao/                 # 数据访问模块
│   ├── pom.xml
│   └── src/
├── my-project-service/             # 业务逻辑模块
│   ├── pom.xml
│   └── src/
├── my-project-api/                 # API 接口模块
│   ├── pom.xml
│   └── src/
├── my-project-web/                 # Web 应用模块
│   ├── pom.xml
│   └── src/
└── my-project-admin/               # 管理后台模块
    ├── pom.xml
    └── src/
```

### 2.2 Spring Boot 多模块结构

```
spring-boot-project/
├── pom.xml                         # 父 POM
├── project-common/                 # 公共模块
│   ├── pom.xml
│   └── src/main/java/
│       └── com/example/common/
│           ├── config/             # 通用配置
│           ├── constant/           # 常量定义
│           ├── exception/          # 异常定义
│           ├── util/               # 工具类
│           └── dto/                # 通用 DTO
├── project-entity/                 # 实体模块
│   ├── pom.xml
│   └── src/main/java/
│       └── com/example/entity/
│           ├── User.java
│           ├── Order.java
│           └── enums/              # 枚举
├── project-repository/             # 数据访问模块
│   ├── pom.xml
│   └── src/main/java/
│       └── com/example/repository/
│           ├── UserRepository.java
│           └── OrderRepository.java
├── project-service/                # 业务逻辑模块
│   ├── pom.xml
│   └── src/main/java/
│       └── com/example/service/
│           ├── UserService.java
│           ├── OrderService.java
│           └── impl/
├── project-api/                    # API 接口模块
│   ├── pom.xml
│   └── src/main/java/
│       └── com/example/api/
│           ├── UserApi.java
│           └── dto/
└── project-web/                    # Web 启动模块
    ├── pom.xml
    └── src/
        ├── main/
        │   ├── java/
        │   │   └── com/example/
        │   │       ├── Application.java
        │   │       └── controller/
        │   └── resources/
        │       ├── application.yml
        │       └── bootstrap.yml
        └── test/
```

## 3. 父 POM 配置

### 3.1 聚合与继承

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <!-- 父项目坐标 -->
    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0</version>
    <!-- 聚合项目必须是 pom 类型 -->
    <packaging>pom</packaging>
    
    <!-- 子模块列表（聚合） -->
    <modules>
        <module>my-project-common</module>
        <module>my-project-model</module>
        <module>my-project-dao</module>
        <module>my-project-service</module>
        <module>my-project-api</module>
        <module>my-project-web</module>
    </modules>
    
    <!-- 属性定义 -->
    <properties>
        <java.version>17</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-boot.version>3.2.0</spring-boot.version>
        <mybatis-spring-boot.version>3.0.3</mybatis-spring-boot.version>
        <mysql.version>8.0.33</mysql.version>
        <lombok.version>1.18.30</lombok.version>
        <hutool.version>5.8.24</hutool.version>
    </properties>
    
    <!-- 依赖版本管理（继承） -->
    <dependencyManagement>
        <dependencies>
            <!-- Spring Boot BOM -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            
            <!-- 子模块依赖版本管理 -->
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-project-common</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-project-model</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-project-dao</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-project-service</artifactId>
                <version>${project.version}</version>
            </dependency>
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>my-project-api</artifactId>
                <version>${project.version}</version>
            </dependency>
            
            <!-- 第三方依赖版本管理 -->
            <dependency>
                <groupId>org.mybatis.spring.boot</groupId>
                <artifactId>mybatis-spring-boot-starter</artifactId>
                <version>${mybatis-spring-boot.version}</version>
            </dependency>
            <dependency>
                <groupId>com.mysql</groupId>
                <artifactId>mysql-connector-j</artifactId>
                <version>${mysql.version}</version>
            </dependency>
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
                <version>${hutool.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <!-- 公共依赖（所有子模块都继承） -->
    <dependencies>
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
        
        <!-- SLF4J -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </dependency>
        
        <!-- 测试 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <!-- 构建配置 -->
    <build>
        <pluginManagement>
            <plugins>
                <!-- 编译插件 -->
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.11.0</version>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                        <encoding>UTF-8</encoding>
                        <annotationProcessorPaths>
                            <path>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                                <version>${lombok.version}</version>
                            </path>
                        </annotationProcessorPaths>
                    </configuration>
                </plugin>
                
                <!-- Spring Boot 插件 -->
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                    <executions>
                        <execution>
                            <goals>
                                <goal>repackage</goal>
                            </goals>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
    
    <!-- Profile -->
    <profiles>
        <profile>
            <id>dev</id>
            <properties>
                <spring.profiles.active>dev</spring.profiles.active>
            </properties>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <spring.profiles.active>prod</spring.profiles.active>
            </properties>
        </profile>
    </profiles>
</project>
```

## 4. 子模块 POM 配置

### 4.1 Common 模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <!-- 继承父 POM -->
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.0.0</version>
    </parent>
    
    <!-- 模块坐标 -->
    <artifactId>my-project-common</artifactId>
    <packaging>jar</packaging>
    
    <!-- 依赖 -->
    <dependencies>
        <!-- 工具库 -->
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
        </dependency>
        
        <!-- JSON -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        
        <!-- Validation -->
        <dependency>
            <groupId>jakarta.validation</groupId>
            <artifactId>jakarta.validation-api</artifactId>
        </dependency>
    </dependencies>
</project>
```

### 4.2 Model 模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.0.0</version>
    </parent>
    
    <artifactId>my-project-model</artifactId>
    <packaging>jar</packaging>
    
    <dependencies>
        <!-- 继承父 POM 的 Lombok、Validation 依赖 -->
        <!-- 如需 JPA 注解 -->
        <dependency>
            <groupId>jakarta.persistence</groupId>
            <artifactId>jakarta.persistence-api</artifactId>
        </dependency>
    </dependencies>
</project>
```

### 4.3 Web 启动模块

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-project</artifactId>
        <version>1.0.0</version>
    </parent>
    
    <artifactId>my-project-web</artifactId>
    <packaging>jar</packaging>
    
    <dependencies>
        <!-- 依赖其他子模块 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-project-service</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>my-project-api</artifactId>
        </dependency>
        
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- 配置处理 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <!-- Spring Boot 打包插件 -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## 5. 模块依赖关系

```
模块依赖关系图:

┌─────────────────────────────────────────────────────────────┐
│                    my-project-web                           │
│                    (启动模块)                                │
├─────────────────────────────────────────────────────────────┤
│      │                │                    │                │
│      ▼                ▼                    ▼                │
│ ┌─────────┐    ┌─────────────┐    ┌─────────────┐         │
│ │ service │    │    api      │    │   common    │         │
│ └────┬────┘    └─────────────┘    └─────────────┘         │
│      │                                                      │
│      ▼                                                      │
│ ┌─────────┐                                                │
│ │   dao   │                                                │
│ └────┬────┘                                                │
│      │                                                      │
│      ▼                                                      │
│ ┌─────────┐                                                │
│ │  model  │                                                │
│ └─────────┘                                                │
└─────────────────────────────────────────────────────────────┘

依赖规则:
- web 依赖 service, api, common
- service 依赖 dao, model, common
- dao 依赖 model, common
- model 依赖 common（可选）
- common 不依赖其他子模块
- api 不依赖 service（对外接口定义）
```

## 6. 构建命令

```bash
# 构建整个项目
mvn clean install

# 只构建特定模块
mvn clean install -pl my-project-common

# 构建模块及其依赖
mvn clean install -pl my-project-service -am

# 跳过测试构建
mvn clean install -DskipTests

# 构建并部署
mvn clean deploy

# 查看模块依赖
mvn dependency:tree -pl my-project-web

# 分析依赖
mvn dependency:analyze -pl my-project-service
```

## 7. 最佳实践

```
多模块项目 Checklist:

□ 模块划分原则:
  - 单一职责，边界清晰
  - 依赖关系单向，避免循环依赖
  - 公共代码抽取到 common 模块

□ 父 POM 配置:
  - 使用 dependencyManagement 统一版本
  - 使用 pluginManagement 统一插件配置
  - 公共依赖放在 dependencies 中

□ 子模块配置:
  - 只声明需要的依赖，不指定版本
  - 打包类型根据需要设置（jar/war）
  - 启动模块使用 spring-boot-maven-plugin

□ 依赖管理:
  - 子模块之间使用版本号引用
  - 避免循环依赖
  - 使用 optional 标记可选依赖

□ 构建优化:
  - 使用 -pl 参数构建特定模块
  - 使用 -am 参数构建依赖模块
  - CI/CD 中使用 -T 参数并行构建
```
