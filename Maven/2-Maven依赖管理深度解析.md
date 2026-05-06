# Maven 依赖管理深度解析

## 1. 依赖范围（Scope）

```
依赖范围决定了依赖在哪些 classpath 中可用:

┌──────────┬──────────────┬──────────────┬──────────────┬──────────────┐
│ Scope    │ compile      │ test         │ runtime      │ provided     │
├──────────┼──────────────┼──────────────┼──────────────┼──────────────┤
│ 编译     │ ✓            │ ✗            │ ✗            │ ✓            │
│ 测试     │ ✓            │ ✓            │ ✓            │ ✓            │
│ 运行     │ ✓            │ ✗            │ ✓            │ ✗            │
│ 打包     │ ✓            │ ✗            │ ✓            │ ✗            │
│ 传递     │ ✓            │ ✗            │ ✓            │ ✗            │
└──────────┴──────────────┴──────────────┴──────────────┴──────────────┘

常用场景:
compile:   默认范围，编译、测试、运行都可用
           例: spring-core, commons-lang3

test:      仅测试可用，不参与打包
           例: junit, mockito, spring-boot-starter-test

provided:  编译和测试可用，运行时由容器提供
           例: servlet-api, lombok, javax.annotation

runtime:   测试和运行可用，编译不需要
           例: mysql-connector-java, logback-classic

system:    类似 provided，但需要指定本地路径
           例: 本地 jar 包
```

```xml
<!-- 依赖范围示例 -->
<dependencies>
    <!-- 编译范围（默认） -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <scope>compile</scope>  <!-- 可省略，默认就是 compile -->
    </dependency>
    
    <!-- 测试范围 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
    
    <!-- provided 范围 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>provided</scope>
    </dependency>
    
    <!-- runtime 范围 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

## 2. 传递依赖

```
传递依赖规则:

项目 A → 依赖 B → 依赖 C
结果: A 自动获得 C 的依赖

传递依赖的范围继承:

直接依赖\传递依赖  compile  test  runtime  provided
compile          compile   -     runtime   -
test             test      -     test      -
runtime          runtime   -     runtime   -
provided         provided  -     provided  -

示例:
A → B (compile) → C (compile)  =>  A 获得 C (compile)
A → B (compile) → C (runtime)  =>  A 获得 C (runtime)
A → B (test)    → C (compile)  =>  A 不获得 C（test 不传递）
```

## 3. 依赖调解

```
依赖调解原则:

1. 最短路径优先:
   A → B → C → D (version 1.0)
   A → D (version 2.0)
   结果: 使用 D 2.0（路径更短）

2. 先声明优先（同路径）:
   A → B → D (version 1.0)
   A → C → D (version 2.0)
   结果: 如果 B 在 C 之前声明，使用 D 1.0

依赖树示例:
[INFO] com.example:my-app:jar:1.0.0
[INFO] +- org.springframework.boot:spring-boot-starter-web:jar:3.2.0:compile
[INFO] |  +- org.springframework.boot:spring-boot-starter:jar:3.2.0:compile
[INFO] |  |  +- org.springframework.boot:spring-boot:jar:3.2.0:compile
[INFO] |  |  +- org.springframework.boot:spring-boot-autoconfigure:jar:3.2.0:compile
[INFO] |  |  \- org.springframework:spring-core:jar:6.1.0:compile
[INFO] |  \- org.springframework.boot:spring-boot-starter-tomcat:jar:3.2.0:compile
[INFO] \- org.apache.logging.log4j:log4j-api:jar:2.21.0:compile
```

## 4. 依赖排除

```xml
<!-- 方式一：使用 exclusions -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- 排除默认的 Tomcat -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
        <!-- 排除特定传递依赖 -->
        <exclusion>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 使用 Jetty 替代 Tomcat -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

## 5. 依赖版本管理

### 5.1 dependencyManagement

```xml
<!-- 父 POM：统一管理依赖版本 -->
<dependencyManagement>
    <dependencies>
        <!-- Spring Boot BOM -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        
        <!-- 自定义版本管理 -->
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>32.1.3-jre</version>
        </dependency>
        
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.14.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 子模块：不需要指定版本 -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- 版本由 BOM 管理 -->
    </dependency>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <!-- 版本由父 POM 管理 -->
    </dependency>
</dependencies>
```

### 5.2 properties 版本管理

```xml
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.2.0</spring-boot.version>
    <guava.version>32.1.3-jre</guava.version>
    <commons-lang3.version>3.14.0</commons-lang3.version>
    <mybatis-spring-boot.version>3.0.3</mybatis-spring-boot.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>com.google.guava</groupId>
        <artifactId>guava</artifactId>
        <version>${guava.version}</version>
    </dependency>
</dependencies>
```

### 5.3 版本号范围

```xml
<!-- 版本号范围语法 -->
<dependency>
    <groupId>org.example</groupId>
    <artifactId>example-lib</artifactId>
    <version>[1.0,2.0)</version>  <!-- 1.0 <= version < 2.0 -->
</dependency>

<!-- 范围符号 -->
[1.0,2.0]    1.0 <= version <= 2.0（闭区间）
[1.0,2.0)    1.0 <= version < 2.0（左闭右开）
(1.0,2.0]    1.0 < version <= 2.0（左开右闭）
(1.0,2.0)    1.0 < version < 2.0（开区间）
[1.0,)       version >= 1.0
(,2.0]       version <= 2.0
[1.0]        version = 1.0（精确版本）

<!-- 不推荐使用版本范围，可能导致构建不稳定 -->
```

## 6. 可选依赖

```xml
<!-- optional 标记：不会传递给使用者 -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>

<!-- 使用者需要显式声明才能使用 -->
<!-- 如果使用者需要 lombok，必须自己添加依赖 -->
```

## 7. 依赖分析工具

```bash
# 显示完整依赖树
mvn dependency:tree

# 按 groupId 过滤
mvn dependency:tree -Dincludes=org.springframework

# 分析依赖使用情况
mvn dependency:analyze

# 输出示例:
[WARNING] Used undeclared dependencies found:
[WARNING]    org.springframework:spring-core:jar:6.1.0:compile
[WARNING] Unused declared dependencies found:
[WARNING]    org.apache.commons:commons-lang3:jar:3.14.0:compile

# 解析依赖
mvn dependency:resolve

# 下载源码
mvn dependency:sources

# 下载文档
mvn dependency:resolve -Dclassifier=javadoc
```

## 8. 常见依赖配置

### 8.1 Spring Boot 项目

```xml
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.2.0</spring-boot.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>${spring-boot.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- 数据库 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>
    
    <!-- Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    
    <!-- 工具库 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    
    <!-- 测试 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### 8.2 常用工具库

```xml
<!-- Apache Commons -->
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.14.0</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version>
</dependency>
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.15.0</version>
</dependency>

<!-- Guava -->
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>32.1.3-jre</version>
</dependency>

<!-- Hutool -->
<dependency>
    <groupId>cn.hutool</groupId>
    <artifactId>hutool-all</artifactId>
    <version>5.8.24</version>
</dependency>
```

## 9. 依赖冲突解决

```
冲突场景及解决方案:

场景1: 版本冲突
A → B 1.0
A → C → B 2.0
结果: 使用 B 1.0（最短路径）

场景2: 需要特定版本
解决方案1: 直接声明依赖（优先级最高）
<dependency>
    <groupId>org.example</groupId>
    <artifactId>lib</artifactId>
    <version>2.0</version>
</dependency>

解决方案2: 使用 exclusions 排除
<dependency>
    <groupId>org.example</groupId>
    <artifactId>old-lib</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.example</groupId>
            <artifactId>conflict-lib</artifactId>
        </exclusion>
    </exclusions>
</dependency>

场景3: 全局版本锁定
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.example</groupId>
            <artifactId>conflict-lib</artifactId>
            <version>2.0</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 10. 最佳实践

```
依赖管理 Checklist:

□ 使用 dependencyManagement 统一版本:
  - 父 POM 定义版本号
  - 子模块不指定版本
  - 使用 properties 管理版本变量

□ 合理使用依赖范围:
  - test 范围：测试依赖不打包
  - provided 范围：容器提供的依赖
  - optional 标记：可选依赖

□ 定期检查依赖更新:
  - mvn versions:display-dependency-updates
  - mvn versions:display-plugin-updates
  - 关注安全漏洞（CVE）

□ 排除冲突依赖:
  - 使用 mvn dependency:tree 分析
  - 使用 exclusions 排除不需要的传递依赖
  - 直接声明需要的版本

□ 避免版本范围:
  - 使用精确版本号
  - 避免 [1.0,2.0) 这种范围定义
  - 版本范围可能导致构建不稳定

□ 优化依赖:
  - mvn dependency:analyze 分析未使用依赖
  - 移除未使用的依赖
  - 添加缺失的依赖（Used undeclared）
```
