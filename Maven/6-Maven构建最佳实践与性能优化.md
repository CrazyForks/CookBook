# Maven 构建最佳实践与性能优化

## 1. POM 最佳实践

```
POM 编写规范:

┌─────────────────────────────────────────────────────────────┐
│                    POM 最佳实践                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 属性管理:                                               │
│     - 使用 properties 定义版本号                           │
│     - 避免硬编码版本号                                      │
│     - 统一编码和 Java 版本                                  │
│                                                             │
│  2. 依赖管理:                                               │
│     - 使用 dependencyManagement 统一版本                   │
│     - 使用 BOM 管理框架版本                                │
│     - 合理使用依赖范围                                      │
│                                                             │
│  3. 插件管理:                                               │
│     - 使用 pluginManagement 统一插件配置                   │
│     - 指定插件版本                                          │
│     - 避免重复配置                                          │
│                                                             │
│  4. 模块组织:                                               │
│     - 清晰的模块划分                                        │
│     - 合理的依赖方向                                        │
│     - 避免循环依赖                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```xml
<!-- 优秀的 POM 示例 -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <name>My Project</name>
    <description>A well-structured Maven project</description>
    <url>https://github.com/example/my-project</url>
    
    <!-- 版本号集中管理 -->
    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        
        <!-- 框架版本 -->
        <spring-boot.version>3.2.0</spring-boot.version>
        
        <!-- 工具库版本 -->
        <lombok.version>1.18.30</lombok.version>
        <mapstruct.version>1.5.5.Final</mapstruct.version>
        <guava.version>32.1.3-jre</guava.version>
        
        <!-- 插件版本 -->
        <maven-compiler-plugin.version>3.11.0</maven-compiler-plugin.version>
        <maven-surefire-plugin.version>3.2.2</maven-surefire-plugin.version>
        <spring-boot-maven-plugin.version>${spring-boot.version}</spring-boot-maven-plugin.version>
    </properties>
    
    <!-- 依赖版本管理 -->
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
    
    <!-- 公共依赖 -->
    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
    
    <!-- 插件管理 -->
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>${maven-compiler-plugin.version}</version>
                    <configuration>
                        <source>${java.version}</source>
                        <target>${java.version}</target>
                        <annotationProcessorPaths>
                            <path>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                                <version>${lombok.version}</version>
                            </path>
                        </annotationProcessorPaths>
                    </configuration>
                </plugin>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-surefire-plugin</artifactId>
                    <version>${maven-surefire-plugin.version}</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
    
    <modules>
        <module>module-a</module>
        <module>module-b</module>
    </modules>
</project>
```

## 2. 构建性能优化

### 2.1 并行构建

```bash
# 使用 -T 参数并行构建
mvn clean install -T 1C          # 每个 CPU 核心一个线程
mvn clean install -T 4           # 4 个线程
mvn clean install -T 4 -C        # 4 个线程，非阻塞

# 多模块项目并行构建
mvn clean install -T 4 --also-make
```

### 2.2 离线模式

```bash
# 离线模式（不检查远程仓库）
mvn clean install -o

# 强制更新快照
mvn clean install -U
```

### 2.3 跳过不必要的步骤

```bash
# 跳过测试
mvn clean install -DskipTests
mvn clean install -Dmaven.test.skip=true  # 跳过测试编译和执行

# 跳过代码检查
mvn clean install -Dcheckstyle.skip=true
mvn clean install -Dspotbugs.skip=true
mvn clean install -Dpmd.skip=true

# 跳过文档生成
mvn clean install -Dmaven.javadoc.skip=true
mvn clean install -Dmaven.source.skip=true
```

### 2.4 增量构建

```bash
# 只构建变更的模块
mvn clean install -pl my-module

# 构建模块及其依赖
mvn clean install -pl my-module -am

# 构建模块及依赖它的模块
mvn clean install -pl my-module -amd
```

## 3. 配置优化

### 3.1 Maven 配置优化

```xml
<!-- settings.xml 优化 -->
<settings>
    <!-- 使用 SSD 路径 -->
    <localRepository>/ssd/maven-repo</localRepository>
    
    <!-- 配置代理（如果需要） -->
    <proxies>
        <proxy>
            <id>http-proxy</id>
            <active>true</active>
            <protocol>http</protocol>
            <host>proxy.example.com</host>
            <port>8080</port>
            <nonProxyHosts>localhost|127.0.0.1</nonProxyHosts>
        </proxy>
    </proxies>
</settings>
```

### 3.2 JVM 参数优化

```bash
# 设置 Maven 运行的 JVM 参数
export MAVEN_OPTS="-Xms512m -Xmx2g -XX:+UseG1GC -XX:+UseStringDeduplication"

# 或在命令行指定
mvn clean install -DargLine="-Xms512m -Xmx2g"
```

## 4. CI/CD 集成

### 4.1 GitHub Actions

```yaml
# .github/workflows/maven.yml
name: Java CI with Maven

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    
    - name: Build with Maven
      run: mvn -B clean verify
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./target/site/jacoco/jacoco.xml
```

### 4.2 GitLab CI

```yaml
# .gitlab-ci.yml
image: maven:3.9-eclipse-temurin-17

stages:
  - build
  - test
  - deploy

variables:
  MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

cache:
  paths:
    - .m2/repository/
    - .m2/wrapper/

build:
  stage: build
  script:
    - mvn $MAVEN_CLI_OPTS compile

test:
  stage: test
  script:
    - mvn $MAVEN_CLI_OPTS verify
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
    paths:
      - target/site/jacoco/

deploy:
  stage: deploy
  script:
    - mvn $MAVEN_CLI_OPTS deploy -DskipTests
  only:
    - main
```

### 4.3 Jenkins Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-17'
            args '-v $HOME/.m2:/root/.m2'
        }
    }
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn -B clean compile'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn -B verify'
            }
            post {
                always {
                    junit 'target/surefire-reports/TEST-*.xml'
                    jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'mvn -B deploy -DskipTests'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
```

## 5. 构建优化检查清单

```
性能优化 Checklist:

□ 并行构建:
  - 使用 -T 参数启用并行构建
  - 多模块项目使用 -pl 和 -am 参数
  - 避免不必要的模块依赖

□ 缓存优化:
  - 使用 SSD 存储本地仓库
  - 配置镜像减少网络请求
  - 启用 Maven 增量构建

□ 跳过不必要的步骤:
  - 开发时跳过测试 (-DskipTests)
  - 跳过代码检查插件
  - 跳过文档生成

□ CI/CD 优化:
  - 配置依赖缓存
  - 使用构建矩阵并行测试
  - 分阶段构建（编译→测试→部署）

□ 依赖优化:
  - 定期清理未使用依赖
  - 使用 dependency:analyze 分析
  - 避免版本范围定义

□ 插件优化:
  - 指定插件版本
  - 避免重复配置
  - 使用 pluginManagement
```

## 6. 常见问题解决

```
常见问题及解决方案:

1. 依赖下载失败
   - 检查网络连接
   - 配置镜像仓库
   - 清理本地仓库缓存: mvn dependency:purge-local-repository

2. 版本冲突
   - 使用 dependency:tree 查看依赖树
   - 使用 exclusions 排除冲突依赖
   - 使用 dependencyManagement 统一版本

3. 编译错误
   - 检查 Java 版本配置
   - 检查编码设置
   - 清理重新编译: mvn clean compile

4. 测试失败
   - 查看测试报告: target/surefire-reports/
   - 跳过测试: -DskipTests
   - 检查测试环境配置

5. 打包失败
   - 检查主类配置
   - 检查依赖范围
   - 使用 mvn clean package -X 查看详细日志

6. 部署失败
   - 检查仓库认证
   - 检查网络连接
   - 检查版本号是否重复
```

## 7. 发布流程

```bash
# 1. 确保代码提交
git status
git add .
git commit -m "Prepare for release"

# 2. 更新版本号（移除 SNAPSHOT）
mvn versions:set -DnewVersion=1.0.0
mvn versions:commit

# 3. 构建并测试
mvn clean verify

# 4. 部署到私服
mvn clean deploy

# 5. 打标签
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin v1.0.0

# 6. 更新到下一个开发版本
mvn versions:set -DnewVersion=1.1.0-SNAPSHOT
mvn versions:commit
git add .
git commit -m "Prepare for next development iteration"
git push
```
