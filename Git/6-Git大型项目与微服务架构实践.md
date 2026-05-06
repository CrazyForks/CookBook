# Git 大型项目与微服务架构实践

## 1. 代码管理策略

```
大型项目代码管理策略:

┌─────────────────────────────────────────────────────────────┐
│              Monorepo vs Multirepo                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Monorepo（单仓库）:                                        │
│  ├── 所有代码在一个仓库                                     │
│  ├── 统一版本管理                                           │
│  ├── 代码共享方便                                           │
│  ├── 工具支持（Bazel, Nx, Turborepo）                       │
│  └── 代表: Google, Facebook, Twitter                        │
│                                                             │
│  Multirepo（多仓库）:                                       │
│  ├── 每个服务独立仓库                                       │
│  ├── 独立部署和版本                                         │
│  ├── 团队自治                                               │
│  ├── 技术栈灵活                                             │
│  └── 代表: Amazon, Netflix, Uber                            │
│                                                             │
│  选择依据:                                                  │
│  ┌─────────────┬─────────────┬─────────────┐               │
│  │ 因素        │ Monorepo    │ Multirepo   │               │
│  ├─────────────┼─────────────┼─────────────┤               │
│  │ 团队规模    │ 大团队      │ 小团队      │               │
│  │ 服务数量    │ 少量        │ 大量        │               │
│  │ 技术栈      │ 统一        │ 多样        │               │
│  │ 部署频率    │ 同步部署    │ 独立部署    │               │
│  │ 代码共享    │ 频繁        │ 较少        │               │
│  └─────────────┴─────────────┴─────────────┘               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. Monorepo 实践

### 2.1 项目结构

```
monorepo/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── packages/
│   ├── common/                 # 公共库
│   │   ├── src/
│   │   ├── pom.xml
│   │   └── README.md
│   ├── user-service/           # 用户服务
│   │   ├── src/
│   │   ├── pom.xml
│   │   └── Dockerfile
│   ├── order-service/          # 订单服务
│   │   ├── src/
│   │   ├── pom.xml
│   │   └── Dockerfile
│   └── gateway/                # 网关服务
│       ├── src/
│       ├── pom.xml
│       └── Dockerfile
├── scripts/
│   ├── build.sh
│   └── deploy.sh
├── pom.xml                     # 父 POM
└── README.md
```

### 2.2 父 POM 配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>microservices-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <modules>
        <module>packages/common</module>
        <module>packages/user-service</module>
        <module>packages/order-service</module>
        <module>packages/gateway</module>
    </modules>
    
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
            <dependency>
                <groupId>com.example</groupId>
                <artifactId>common</artifactId>
                <version>${project.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
</project>
```

### 2.3 增量构建脚本

```bash
#!/bin/bash
# scripts/build-changed.sh

# 获取变更的模块
CHANGED_MODULES=$(git diff --name-only HEAD~1 HEAD | grep "^packages/" | cut -d'/' -f2 | sort -u)

echo "Changed modules: $CHANGED_MODULES"

# 构建变更的模块及其依赖
for module in $CHANGED_MODULES; do
    echo "Building $module..."
    mvn clean package -pl packages/$module -am -DskipTests
done
```

## 3. Multirepo 实践

### 3.1 服务目录结构

```
user-service/
├── .github/
│   └── workflows/
│       └── ci.yml
├── src/
├── pom.xml
├── Dockerfile
├── Jenkinsfile
└── README.md
```

### 3.2 共享库管理

```xml
<!-- 发布共享库到私服 -->
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>http://nexus.example.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://nexus.example.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

```groovy
// Jenkinsfile - 共享库发布
pipeline {
    agent any
    
    stages {
        stage('Build & Publish') {
            steps {
                sh 'mvn clean deploy'
            }
        }
    }
}
```

## 4. 微服务 CI/CD 流水线

### 4.1 完整流水线

```groovy
// Jenkinsfile - 微服务完整流水线
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.9-eclipse-temurin-17
                    command: ['cat']
                    tty: true
                  - name: docker
                    image: docker:20.10
                    command: ['cat']
                    tty: true
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                  volumes:
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            '''
        }
    }
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_NAME = "${env.JOB_NAME}"
        VERSION = "${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Test') {
                    steps {
                        container('maven') {
                            sh 'mvn test'
                        }
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Integration Test') {
                    steps {
                        container('maven') {
                            sh 'mvn verify -P integration-test'
                        }
                    }
                }
            }
        }
        
        stage('Build Image') {
            steps {
                container('docker') {
                    sh """
                        docker build -t ${REGISTRY}/${IMAGE_NAME}:${VERSION} .
                        docker push ${REGISTRY}/${IMAGE_NAME}:${VERSION}
                    """
                }
            }
        }
        
        stage('Deploy Dev') {
            when {
                branch 'develop'
            }
            steps {
                sh """
                    kubectl set image deployment/${IMAGE_NAME} \
                        ${IMAGE_NAME}=${REGISTRY}/${IMAGE_NAME}:${VERSION} \
                        -n dev
                """
            }
        }
        
        stage('Deploy Staging') {
            when {
                branch 'release/*'
            }
            steps {
                sh """
                    kubectl set image deployment/${IMAGE_NAME} \
                        ${IMAGE_NAME}=${REGISTRY}/${IMAGE_NAME}:${VERSION} \
                        -n staging
                """
            }
        }
        
        stage('Deploy Prod') {
            when {
                branch 'main'
            }
            steps {
                input message: '确认部署到生产环境？'
                sh """
                    kubectl set image deployment/${IMAGE_NAME} \
                        ${IMAGE_NAME}=${REGISTRY}/${IMAGE_NAME}:${VERSION} \
                        -n prod
                """
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

## 5. 多服务协同发布

### 5.1 版本管理

```yaml
# versions.yml
services:
  user-service:
    version: 1.2.0
    repo: git@github.com:org/user-service.git
  order-service:
    version: 1.3.1
    repo: git@github.com:org/order-service.git
  gateway:
    version: 2.0.0
    repo: git@github.com:org/gateway.git
```

### 5.2 协同发布脚本

```bash
#!/bin/bash
# scripts/release-all.sh

VERSION=$1

# 更新所有服务版本
for service in user-service order-service gateway; do
    echo "Releasing $service version $VERSION"
    
    cd $service
    git checkout main
    git pull
    
    # 更新版本号
    mvn versions:set -DnewVersion=$VERSION
    mvn versions:commit
    
    # 提交并打标签
    git add .
    git commit -m "release: $VERSION"
    git tag -a "v$VERSION" -m "Release $VERSION"
    
    # 推送
    git push origin main
    git push origin "v$VERSION"
    
    cd ..
done
```

## 6. Git 子模块与微服务

### 6.1 子模块配置

```bash
# 添加子模块
git submodule add git@github.com:org/shared-lib.git libs/shared-lib
git submodule add git@github.com:org/proto-files.git proto

# 查看子模块
git submodule status

# 更新子模块
git submodule update --remote

# 克隆包含子模块的仓库
git clone --recursive git@github.com:org/monorepo.git
```

### 6.2 .gitmodules 配置

```ini
# .gitmodules
[submodule "libs/shared-lib"]
    path = libs/shared-lib
    url = git@github.com:org/shared-lib.git
    branch = main

[submodule "proto"]
    path = proto
    url = git@github.com:org/proto-files.git
    branch = main
```

## 7. Git Hooks 自动化

### 7.1 客户端 Hooks

```bash
#!/bin/bash
# .git/hooks/pre-commit

# 运行代码检查
echo "Running code style check..."
mvn checkstyle:check

if [ $? -ne 0 ]; then
    echo "Code style check failed!"
    exit 1
fi

# 运行单元测试
echo "Running unit tests..."
mvn test -q

if [ $? -ne 0 ]; then
    echo "Unit tests failed!"
    exit 1
fi

echo "Pre-commit checks passed!"
```

### 7.2 服务端 Hooks

```bash
#!/bin/bash
# hooks/post-receive

# 部署脚本
while read oldrev newrev refname; do
    branch=$(git rev-parse --symbolic --abbrev-ref $refname)
    
    if [ "$branch" == "main" ]; then
        echo "Deploying main branch..."
        cd /opt/app
        git pull origin main
        docker-compose up -d --build
    fi
done
```

## 8. 大型项目最佳实践

```
大型项目 Git 实践 Checklist:

□ 代码管理策略:
  - 评估 Monorepo vs Multirepo
  - 统一代码规范
  - 配置代码审查

□ 分支策略:
  - 选择合适的工作流
  - 统一分支命名规范
  - 及时清理分支

□ CI/CD 集成:
  - 自动化构建和测试
  - 分支触发策略
  - 自动化部署

□ 版本管理:
  - 语义化版本号
  - 自动生成 CHANGELOG
  - Tag 管理

□ 依赖管理:
  - 使用私服管理内部依赖
  - 统一版本管理
  - 定期更新依赖

□ 安全考虑:
  - 凭证安全管理
  - 代码扫描
  - 依赖漏洞检查

□ 团队协作:
  - 代码审查流程
  - 冲突解决规范
  - 文档维护
```
