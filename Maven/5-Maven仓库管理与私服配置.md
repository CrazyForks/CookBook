# Maven 仓库管理与私服配置

## 1. 仓库概述

```
Maven 仓库类型:

┌─────────────────────────────────────────────────────────────┐
│                    Maven 仓库体系                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  本地仓库 (~/.m2/repository):                               │
│  ├── 本地缓存的依赖                                         │
│  ├── 本地安装的项目                                         │
│  └── 优先级最高                                             │
│                                                             │
│  远程仓库:                                                  │
│  ├── 中央仓库 (repo.maven.apache.org)                       │
│  │   └── Maven 官方仓库，包含大部分开源依赖                 │
│  ├── 镜像仓库                                               │
│  │   └── 中央仓库的镜像（如阿里云）                         │
│  └── 私服仓库                                               │
│      └── 公司内部 Nexus/Artifactory/MinIO                   │
│                                                             │
│  仓库优先级:                                                │
│  本地仓库 > 私服仓库 > 镜像仓库 > 中央仓库                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 本地仓库配置

```xml
<!-- settings.xml -->
<settings>
    <!-- 本地仓库路径 -->
    <localRepository>/path/to/local/repo</localRepository>
</settings>
```

```bash
# 查看本地仓库位置
mvn help:evaluate -Dexpression=settings.localRepository -q -DforceStdout

# 清理本地仓库中的未使用依赖
mvn dependency:purge-local-repository

# 安装本地 jar 到本地仓库
mvn install:install-file \
    -Dfile=path/to/file.jar \
    -DgroupId=com.example \
    -DartifactId=local-lib \
    -Dversion=1.0.0 \
    -Dpackaging=jar
```

## 3. 远程仓库配置

### 3.1 在 POM 中配置

```xml
<repositories>
    <!-- 中央仓库 -->
    <repository>
        <id>central</id>
        <name>Maven Central</name>
        <url>https://repo.maven.apache.org/maven2</url>
        <releases>
            <enabled>true</enabled>
            <updatePolicy>daily</updatePolicy>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
    
    <!-- 阿里云镜像 -->
    <repository>
        <id>aliyun</id>
        <name>Aliyun Maven</name>
        <url>https://maven.aliyun.com/repository/public</url>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
    
    <!-- Spring 仓库 -->
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

### 3.2 在 settings.xml 中配置镜像

```xml
<settings>
    <mirrors>
        <!-- 阿里云镜像（推荐） -->
        <mirror>
            <id>aliyun</id>
            <name>Aliyun Maven Mirror</name>
            <url>https://maven.aliyun.com/repository/public</url>
            <mirrorOf>central</mirrorOf>
        </mirror>
        
        <!-- 私服镜像 -->
        <mirror>
            <id>nexus</id>
            <name>Nexus Mirror</name>
            <url>http://nexus.example.com/repository/maven-public/</url>
            <mirrorOf>*</mirrorOf>
        </mirror>
    </mirrors>
</settings>
```

## 4. Nexus 私服搭建

### 4.1 Docker 部署 Nexus

```bash
# 拉取镜像
docker pull sonatype/nexus3:latest

# 启动容器
docker run -d \
    --name nexus \
    -p 8081:8081 \
    -v nexus-data:/nexus-data \
    sonatype/nexus3:latest

# 查看初始密码
docker exec nexus cat /nexus-data/admin.password

# 访问: http://localhost:8081
# 默认用户: admin
```

### 4.2 Nexus 仓库类型

```
Nexus 仓库类型:

┌─────────────┬──────────────────────────────────────────────┐
│ 类型        │ 说明                                          │
├─────────────┼──────────────────────────────────────────────┤
│ hosted      │ 宿主仓库（私有依赖）                         │
│             ├── 3rd party: 第三方依赖                      │
│             ├── snapshots: 快照版本                        │
│             └── releases:  正式版本                        │
├─────────────┼──────────────────────────────────────────────┤
│ proxy       │ 代理仓库（远程仓库代理）                     │
│             ├── maven-central: 中央仓库代理                │
│             ├── aliyun: 阿里云代理                         │
│             └── spring: Spring 仓库代理                    │
├─────────────┼──────────────────────────────────────────────┤
│ group       │ 仓库组（聚合多个仓库）                       │
│             └── maven-public: 聚合 hosted + proxy          │
└─────────────┴──────────────────────────────────────────────┘
```

### 4.3 settings.xml 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.2.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.2.0 
          https://maven.apache.org/xsd/settings-1.2.0.xsd">
    
    <!-- 服务器认证 -->
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
    
    <!-- 镜像配置 -->
    <mirrors>
        <mirror>
            <id>nexus</id>
            <name>Nexus Repository</name>
            <url>http://nexus.example.com/repository/maven-public/</url>
            <mirrorOf>*</mirrorOf>
        </mirror>
    </mirrors>
    
    <!-- Profile 配置 -->
    <profiles>
        <profile>
            <id>nexus</id>
            <repositories>
                <repository>
                    <id>central</id>
                    <url>https://repo.maven.apache.org/maven2</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>true</enabled>
                    </snapshots>
                </repository>
            </repositories>
            <pluginRepositories>
                <pluginRepository>
                    <id>central</id>
                    <url>https://repo.maven.apache.org/maven2</url>
                    <releases>
                        <enabled>true</enabled>
                    </releases>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>
    
    <activeProfiles>
        <activeProfile>nexus</activeProfile>
    </activeProfiles>
</settings>
```

### 4.4 POM 配置部署

```xml
<!-- 配置部署仓库 -->
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <name>Nexus Release Repository</name>
        <url>http://nexus.example.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <name>Nexus Snapshot Repository</name>
        <url>http://nexus.example.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

```bash
# 部署到私服
mvn clean deploy

# 安装本地 jar 到私服
mvn deploy:deploy-file \
    -Dfile=path/to/file.jar \
    -DgroupId=com.example \
    -DartifactId=local-lib \
    -Dversion=1.0.0 \
    -Dpackaging=jar \
    -Durl=http://nexus.example.com/repository/maven-releases/ \
    -DrepositoryId=nexus-releases
```

## 5. Artifactory 配置

```xml
<!-- settings.xml -->
<servers>
    <server>
        <id>artifactory-releases</id>
        <username>admin</username>
        <password>password</password>
    </server>
    <server>
        <id>artifactory-snapshots</id>
        <username>admin</username>
        <password>password</password>
    </server>
</servers>

<!-- POM -->
<distributionManagement>
    <repository>
        <id>artifactory-releases</id>
        <name>Artifactory Release Repository</name>
        <url>https://artifactory.example.com/libs-release</url>
    </repository>
    <snapshotRepository>
        <id>artifactory-snapshots</id>
        <name>Artifactory Snapshot Repository</name>
        <url>https://artifactory.example.com/libs-snapshot</url>
    </snapshotRepository>
</distributionManagement>
```

## 6. GitHub Packages

```xml
<!-- settings.xml -->
<servers>
    <server>
        <id>github</id>
        <username>${env.GITHUB_USERNAME}</username>
        <password>${env.GITHUB_TOKEN}</password>
    </server>
</servers>

<!-- POM -->
<repositories>
    <repository>
        <id>github</id>
        <name>GitHub Packages</name>
        <url>https://maven.pkg.github.com/owner/repo</url>
    </repository>
</repositories>

<distributionManagement>
    <repository>
        <id>github</id>
        <name>GitHub Packages</name>
        <url>https://maven.pkg.github.com/owner/repo</url>
    </repository>
</distributionManagement>
```

## 7. 仓库管理最佳实践

```
私服部署 Checklist:

□ 使用私服管理内部依赖:
  - 内部公共库发布到私服
  - 统一版本管理
  - 避免源码泄露

□ 配置镜像加速:
  - 国内使用阿里云镜像
  - 公司内部配置私服镜像
  - 减少外网依赖

□ 版本策略:
  - releases 仓库只放正式版本
  - snapshots 仓库放开发版本
  - 定期清理过期快照

□ 安全配置:
  - 使用加密密码（mvn --encrypt-password）
  - 限制仓库访问权限
  - 审计部署日志

□ 备份策略:
  - 定期备份仓库数据
  - 配置高可用（HA）
  - 监控磁盘空间
```
