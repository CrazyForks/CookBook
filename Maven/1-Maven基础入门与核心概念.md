# Maven 基础入门与核心概念

## 1. Maven 概述

```
Maven 是什么？

┌─────────────────────────────────────────────────────────────┐
│                    Apache Maven                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  定义: 基于 POM（项目对象模型）的项目管理工具               │
│                                                             │
│  核心功能:                                                  │
│  ├── 项目构建: 编译、测试、打包、部署                       │
│  ├── 依赖管理: 自动下载、版本管理、传递依赖                 │
│  ├── 模块管理: 多模块项目、父子继承                         │
│  ├── 仓库管理: 本地仓库、远程仓库、私服                     │
│  └── 插件体系: 丰富的插件生态                               │
│                                                             │
│  设计理念:                                                  │
│  ├── Convention Over Configuration（约定优于配置）           │
│  ├── Don't Repeat Yourself（DRY，不重复自己）                │
│  └── Build Once, Deploy Anywhere（一次构建，到处部署）       │
│                                                             │
│  版本:                                                      │
│  ├── Maven 2: 2005年发布，已停止维护                         │
│  ├── Maven 3: 2010年发布，当前主流版本                       │
│  └── Maven 3.9.x/4.0.x: 最新版本                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 安装与配置

### 2.1 安装 Maven

```bash
# 方式一：使用包管理器
# macOS
brew install maven

# Ubuntu/Debian
sudo apt-get install maven

# CentOS/RHEL
sudo yum install maven

# 方式二：手动安装
# 1. 下载: https://maven.apache.org/download.cgi
# 2. 解压到 /usr/local/maven
# 3. 配置环境变量

# 验证安装
mvn -version
# 输出示例:
# Apache Maven 3.9.6 (...)
# Maven home: /usr/local/maven
# Java version: 17.0.10, vendor: Eclipse Adoptium
```

### 2.2 配置 settings.xml

```xml
<!-- ~/.m2/settings.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 
          https://maven.apache.org/xsd/settings-1.2.0.xsd">
    
    <!-- 本地仓库路径（可选，默认 ~/.m2/repository） -->
    <localRepository>/path/to/local/repo</localRepository>
    
    <!-- 镜像配置（加速下载） -->
    <mirrors>
        <!-- 阿里云镜像（国内推荐） -->
        <mirror>
            <id>aliyun</id>
            <name>Aliyun Maven Mirror</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
    </mirrors>
    
    <!-- 私服配置 -->
    <servers>
        <server>
            <id>nexus-releases</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
        <server>
            <id>nexus-snapshots</id>
            <username>admin</username>
            <password>admin123</password>
        </server>
    </servers>
    
    <!-- Profile 配置 -->
    <profiles>
        <profile>
            <id>default</id>
            <repositories>
                <repository>
                    <id>central</id>
                    <url>https://repo.maven.apache.org/maven2</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                </repository>
            </repositories>
        </profile>
    </profiles>
    
    <!-- 激活 Profile -->
    <activeProfiles>
        <activeProfile>default</activeProfile>
    </activeProfiles>
</settings>
```

## 3. POM 文件详解

### 3.1 POM 结构

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <!-- POM 模型版本（固定值） -->
    <modelVersion>4.0.0</modelVersion>
    
    <!-- 项目坐标 -->
    <groupId>com.example</groupId>           <!-- 组织ID -->
    <artifactId>my-app</artifactId>          <!-- 项目ID -->
    <version>1.0.0</version>                 <!-- 版本号 -->
    <packaging>jar</packaging>               <!-- 打包方式 -->
    
    <!-- 项目信息 -->
    <name>My Application</name>
    <description>A sample Maven project</description>
    <url>https://github.com/example/my-app</url>
    
    <!-- 属性 -->
    <properties>
        <java.version>17</java.version>
        <spring-boot.version>3.2.0</spring-boot.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    
    <!-- 依赖管理 -->
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
    
    <!-- 依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <!-- 构建配置 -->
    <build>
        <plugins>
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
    </build>
    
    <!-- Profile -->
    <profiles>
        <profile>
            <id>dev</id>
            <properties>
                <spring.profiles.active>dev</spring.profiles.active>
            </properties>
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

### 3.2 坐标与版本

```
Maven 坐标（GAV）:

groupId:    组织标识（通常是公司域名反写）
            例: com.example, org.apache

artifactId: 项目标识（项目名称）
            例: spring-boot-starter-web

version:    版本号
            例: 1.0.0, 2.3.4-SNAPSHOT

packaging:  打包方式
            jar（默认）: Java 应用
            war: Web 应用
            pom: 父项目/聚合项目

版本号规范:
┌─────────────────────────────────────────────────────────────┐
│  主版本.次版本.增量版本-里程碑版本                           │
│  例: 1.0.0-SNAPSHOT                                         │
│                                                             │
│  主版本: 重大架构变更                                       │
│  次版本: 新增功能                                           │
│  增量版本: Bug修复                                          │
│  里程碑: SNAPSHOT(快照), Alpha, Beta, RC(候选), GA(正式)    │
└─────────────────────────────────────────────────────────────┘
```

## 4. 生命周期与阶段

### 4.1 三大生命周期

```
Maven 生命周期:

clean: 清理项目
├── pre-clean:   清理前准备
├── clean:       删除 target 目录
└── post-clean:  清理后操作

default: 构建项目（核心）
├── validate:        验证项目结构
├── initialize:      初始化构建状态
├── generate-sources: 生成源代码
├── process-sources:  处理源代码
├── generate-resources: 生成资源文件
├── process-resources:  处理资源文件（复制到 target）
├── compile:         编译源代码
├── process-classes: 处理编译后的文件
├── generate-test-sources: 生成测试代码
├── process-test-sources:  处理测试代码
├── generate-test-resources: 生成测试资源
├── process-test-resources: 处理测试资源
├── test-compile:    编译测试代码
├── process-test-classes: 处理测试类
├── test:            运行单元测试
├── prepare-package: 打包前准备
├── package:         打包（jar/war）
├── pre-integration-test: 集成测试前准备
├── integration-test:     运行集成测试
├── post-integration-test: 集成测试后处理
├── verify:          验证包质量
├── install:         安装到本地仓库
└── deploy:          部署到远程仓库

site: 生成项目文档
├── pre-site:    生成前准备
├── site:        生成项目文档
├── post-site:   生成后处理
└── site-deploy: 部署文档
```

### 4.2 常用命令

```bash
# 基础命令
mvn clean               # 清理 target 目录
mvn compile             # 编译源代码
mvn test                # 运行测试
mvn package             # 打包（生成 jar/war）
mvn install             # 安装到本地仓库
mvn deploy              # 部署到远程仓库

# 组合命令
mvn clean package       # 清理并打包
mvn clean install       # 清理并安装到本地仓库
mvn clean deploy        # 清理并部署到远程仓库

# 跳过测试
mvn package -DskipTests              # 跳过测试执行
mvn package -Dmaven.test.skip=true   # 跳过测试编译和执行

# 指定 Profile
mvn package -P dev      # 使用 dev profile
mvn package -P prod     # 使用 prod profile

# 依赖分析
mvn dependency:tree     # 显示依赖树
mvn dependency:analyze  # 分析依赖使用情况

# 查看有效 POM
mvn help:effective-pom

# 查看有效 settings
mvn help:effective-settings

# 生成项目骨架
mvn archetype:generate -DgroupId=com.example -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart
```

## 5. 目录结构约定

```
Maven 标准目录结构:

my-app/
├── pom.xml                    # 项目对象模型
├── src/
│   ├── main/
│   │   ├── java/              # 主源代码
│   │   │   └── com/example/
│   │   │       └── App.java
│   │   ├── resources/         # 主资源文件
│   │   │   ├── application.yml
│   │   │   └── static/
│   │   ├── webapp/            # Web 应用资源（war 项目）
│   │   │   ├── WEB-INF/
│   │   │   └── index.html
│   │   └── filters/           # 资源过滤文件
│   └── test/
│       ├── java/              # 测试源代码
│       │   └── com/example/
│       │       └── AppTest.java
│       └── resources/         # 测试资源文件
│           └── test-data.json
├── target/                    # 构建输出目录
│   ├── classes/               # 编译后的类文件
│   ├── test-classes/          # 编译后的测试类
│   ├── my-app-1.0.0.jar      # 打包产物
│   └── surefire-reports/      # 测试报告
├── .mvn/                      # Maven Wrapper 配置
│   └── wrapper/
├── mvnw                       # Maven Wrapper（Linux/Mac）
├── mvnw.cmd                   # Maven Wrapper（Windows）
└── README.md
```

## 6. 第一个 Maven 项目

### 6.1 使用 Archetype 创建

```bash
# 交互式创建
mvn archetype:generate

# 指定参数创建
mvn archetype:generate \
    -DgroupId=com.example \
    -DartifactId=demo-app \
    -DarchetypeArtifactId=maven-archetype-quickstart \
    -DarchetypeVersion=1.4 \
    -DinteractiveMode=false

# Spring Boot 项目创建
mvn archetype:generate \
    -DgroupId=com.example \
    -DartifactId=spring-boot-demo \
    -DarchetypeArtifactId=maven-archetype-quickstart \
    -DinteractiveMode=false
```

### 6.2 项目结构

```bash
# 创建后的目录结构
demo-app/
├── pom.xml
└── src
    ├── main
    │   └── java
    │       └── com
    │           └── example
    │               └── App.java
    └── test
        └── java
            └── com
                └── example
                    └── AppTest.java
```

### 6.3 编译运行

```bash
cd demo-app

# 编译
mvn compile

# 运行测试
mvn test

# 打包
mvn package

# 运行（如果有 main 方法）
java -cp target/demo-app-1.0-SNAPSHOT.jar com.example.App
```

## 7. 内置属性

```
Maven 内置属性:

项目属性:
${project.groupId}           项目 groupId
${project.artifactId}        项目 artifactId
${project.version}           项目版本
${project.name}              项目名称
${project.description}       项目描述
${project.basedir}           项目根目录
${project.build.directory}   构建目录（target）
${project.build.outputDirectory}      编译输出目录
${project.build.testOutputDirectory}  测试输出目录
${project.build.sourceDirectory}      源代码目录
${project.build.testSourceDirectory}  测试源代码目录
${project.build.finalName}   最终打包名称

Settings 属性:
${settings.localRepository}  本地仓库路径

系统属性:
${user.home}                 用户主目录
${user.name}                 用户名
${java.home}                 Java 安装目录
${java.version}              Java 版本
${os.name}                   操作系统名称
${os.arch}                   操作系统架构
${file.separator}            文件分隔符

环境变量:
${env.JAVA_HOME}             JAVA_HOME 环境变量
${env.PATH}                  PATH 环境变量
${env.M2_HOME}               Maven 安装目录

自定义属性:
在 <properties> 中定义的属性都可以使用
例: ${my.custom.property}
```

## 8. 快速参考

```
常用命令速查表:

┌──────────────────────┬───────────────────────────────────────┐
│ 命令                 │ 说明                                  │
├──────────────────────┼───────────────────────────────────────┤
│ mvn clean            │ 清理 target 目录                      │
│ mvn compile          │ 编译源代码                            │
│ mvn test             │ 运行单元测试                          │
│ mvn package          │ 打包（生成 jar/war）                  │
│ mvn install          │ 安装到本地仓库                        │
│ mvn deploy           │ 部署到远程仓库                        │
│ mvn clean package    │ 清理并打包                            │
│ mvn clean install    │ 清理并安装                            │
│ mvn dependency:tree  │ 显示依赖树                            │
│ mvn dependency:analyze │ 分析依赖使用情况                   │
│ mvn help:effective-pom │ 查看有效 POM                       │
│ mvn versions:display-dependency-updates │ 检查依赖更新     │
│ mvn versions:display-plugin-updates    │ 检查插件更新      │
└──────────────────────┴───────────────────────────────────────┘
```
