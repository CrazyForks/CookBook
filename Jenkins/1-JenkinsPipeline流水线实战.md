# Jenkins Pipeline 流水线实战

## 1. 概念

Jenkins Pipeline（流水线）是一套插件，支持在 Jenkins 中实现和集成 **持续交付管道（CD Pipeline）**。通过代码（Pipeline as Code）定义完整的构建、测试、部署流程，实现版本化、可复用、可审查的 CI/CD 流程。

```
传统 CI vs Pipeline CI:

传统方式（Freestyle Job）:
┌─────────────────────────────────────┐
│  Job: myapp-build                   │
│  ├─ 源码管理: Git                   │
│  ├─ 构建触发器: 定时/Hook           │
│  ├─ 构建步骤: Maven 打包            │
│  ├─ 构建后操作: 发送邮件            │
│  └─ 配置: 通过 Web UI 点击          │
│                                      │
│  问题: 配置难迁移、难版本控制        │
└─────────────────────────────────────┘

Pipeline 方式（Jenkinsfile）:
┌─────────────────────────────────────┐
│  Jenkinsfile（代码化配置）          │
│  ├── 代码仓库中版本控制              │
│  ├── Code Review 可审查              │
│  ├── 多环境复用                      │
│  └── 声明式/脚本式语法               │
│                                      │
│  优势: Pipeline as Code              │
└─────────────────────────────────────┘
```

### 1.1 Pipeline 类型

```
两种 Pipeline 语法:

声明式 Pipeline（Declarative）:
┌─────────────────────────────────────┐
│ pipeline {                          │
│   agent any                         │
│   stages {                          │
│     stage('Build') {                │
│       steps {                       │
│         sh 'mvn clean package'      │
│       }                             │
│     }                               │
│   }                                 │
│ }                                   │
│                                      │
│ 特点: 结构化、易读、有校验           │
│ 适用: 大多数 CI/CD 场景              │
└─────────────────────────────────────┘

脚本式 Pipeline（Scripted）:
┌─────────────────────────────────────┐
│ node {                              │
│   stage('Build') {                  │
│     sh 'mvn clean package'          │
│   }                                 │
│ }                                   │
│                                      │
│ 特点: 灵活、基于 Groovy             │
│ 适用: 复杂逻辑、动态流程             │
└─────────────────────────────────────┘
```

## 2. 核心概念

```
Pipeline 核心模型:

Pipeline（流水线）
  └── Agent（执行节点）
        └── Stage（阶段）
              └── Step（步骤）

关键概念:

Pipeline: 完整的交付流程，从代码提交到生产部署
Agent:    执行环境（节点/容器），指定在哪里运行
Stage:    逻辑分组（Build/Test/Deploy），可视化展示
Step:     最小执行单元（shell 命令、git 拉取等）
Node:     Jenkins 工作节点（Master/Slave）
Executor: 执行槽位，一个节点可同时运行多个任务
Workspace: 工作目录，每个 Pipeline 独立的文件空间
```

## 3. 声明式 Pipeline 详解

### 3.1 完整示例（Java 项目）

```groovy
// Jenkinsfile（放在项目根目录）
pipeline {
    // 执行节点配置
    agent {
        docker {
            image 'maven:3.8-openjdk-11'
            args '-v /root/.m2:/root/.m2'
        }
    }
    
    // 环境变量
    environment {
        MAVEN_OPTS = '-Xmx1024m'
        APP_NAME = 'spring-boot-app'
        DOCKER_REGISTRY = 'registry.cn-hangzhou.aliyuncs.com'
        DOCKER_REPO = "${DOCKER_REGISTRY}/myrepo/${APP_NAME}"
    }
    
    // 参数化构建
    parameters {
        choice(
            name: 'ENV',
            choices: ['dev', 'test', 'prod'],
            description: '部署环境'
        )
        booleanParam(
            name: 'SKIP_TEST',
            defaultValue: false,
            description: '是否跳过测试'
        )
        string(
            name: 'VERSION',
            defaultValue: '1.0.0',
            description: '版本号'
        )
    }
    
    // 构建触发器
    triggers {
        // 每 15 分钟检查一次代码变更
        pollSCM('H/15 * * * *')
        // 或 Webhook 触发（推荐）
    }
    
    // 选项配置
    options {
        // 保留最近 10 次构建记录
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // 禁止并发构建
        disableConcurrentBuilds()
        // 超时设置（1小时）
        timeout(time: 1, unit: 'HOURS')
        // 输出时间戳
        timestamps()
    }
    
    // 流水线阶段
    stages {
        stage('Checkout') {
            steps {
                // 拉取代码
                checkout scm
                
                // 显示提交信息
                script {
                    def commit = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()
                    echo "当前 commit: ${commit}"
                }
            }
        }
        
        stage('Build') {
            steps {
                // Maven 编译打包
                script {
                    def skipTests = params.SKIP_TEST ? '-DskipTests' : ''
                    sh "mvn clean package ${skipTests} -Drevision=${params.VERSION}"
                }
                
                // 归档构建产物
                archiveArtifacts artifacts: 'target/*.jar', 
                                 fingerprint: true
            }
        }
        
        stage('Unit Test') {
            when {
                expression { !params.SKIP_TEST }
            }
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    // 发布测试报告
                    junit 'target/surefire-reports/*.xml'
                    // 代码覆盖率
                    jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }
        
        stage('Code Analysis') {
            parallel {
                stage('SonarQube') {
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                mvn sonar:sonar \
                                  -Dsonar.projectKey=${APP_NAME} \
                                  -Dsonar.projectName=${APP_NAME} \
                                  -Dsonar.sources=src/main/java
                            '''
                        }
                    }
                }
                stage('SpotBugs') {
                    steps {
                        sh 'mvn spotbugs:check'
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    def image = docker.build(
                        "${DOCKER_REPO}:${params.VERSION}",
                        "--build-arg JAR_FILE=target/*.jar ."
                    )
                    
                    // 推送到镜像仓库
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    if (params.ENV == 'prod') {
                        // 生产环境需要审批
                        input message: '确认部署到生产环境？', 
                              ok: '确认',
                              submitter: 'admin,deployer'
                    }
                    
                    // 执行部署脚本
                    sshagent(['deploy-server-ssh']) {
                        sh """
                            ssh deploy@server1 'bash -s' < deploy.sh \\
                                ${params.VERSION} \\
                                ${params.ENV}
                        """
                    }
                }
            }
        }
    }
    
    // 后置处理
    post {
        always {
            // 清理工作空间
            cleanWs()
            
            // 发送构建通知
            script {
                def color = currentBuild.result == 'SUCCESS' ? 'good' : 'danger'
                def message = """
                    *构建结果*: ${currentBuild.result}
                    *项目*: ${env.JOB_NAME}
                    *版本*: ${params.VERSION}
                    *环境*: ${params.ENV}
                    *持续时间*: ${currentBuild.durationString}
                    *构建链接*: ${env.BUILD_URL}
                """.stripIndent()
                
                // 钉钉通知
                dingTalk(
                    robot: 'jenkins-robot',
                    type: 'MARKDOWN',
                    text: [message]
                )
            }
        }
        success {
            echo '构建成功！'
        }
        failure {
            echo '构建失败！'
            // 失败时发送邮件给提交者
            emailext(
                subject: "构建失败: ${env.JOB_NAME} - ${params.VERSION}",
                body: "请查看日志: ${env.BUILD_URL}console",
                to: "${env.CHANGE_AUTHOR_EMAIL ?: 'dev@company.com'}"
            )
        }
        unstable {
            echo '构建不稳定（测试失败）'
        }
        aborted {
            echo '构建被取消'
        }
    }
}
```

### 3.2 指令速查表

| 指令 | 说明 | 示例 |
|------|------|------|
| `agent` | 指定执行节点 | `agent any`, `agent { label 'docker' }` |
| `stages` | 阶段集合 | 包含多个 `stage` |
| `stage` | 单个阶段 | `stage('Build')` |
| `steps` | 步骤集合 | 包含多个 `step` |
| `when` | 条件执行 | `when { branch 'master' }` |
| `parallel` | 并行执行 | `parallel { stage('A'); stage('B') }` |
| `environment` | 环境变量 | `env.KEY = 'value'` |
| `parameters` | 参数化 | `string(name: 'VERSION')` |
| `post` | 后置处理 | `always`, `success`, `failure` |
| `script` | 脚本块 | 内嵌 Groovy 代码 |
| `options` | 选项 | `timeout`, `timestamps` |
| `triggers` | 触发器 | `cron`, `pollSCM` |

### 3.3 条件控制

```groovy
stage('Deploy') {
    when {
        // 分支条件
        branch 'master'
        
        // 表达式条件
        expression { params.ENV == 'prod' }
        
        // 环境变量条件
        environment name: 'DEPLOY_ENV', value: 'production'
        
        // 任意条件满足
        anyOf {
            branch 'master'
            branch 'release/*'
        }
        
        // 所有条件满足
        allOf {
            branch 'master'
            expression { params.SKIP_DEPLOY != true }
        }
        
        // 标签存在
        tag 'release-*'
    }
    steps {
        sh 'deploy.sh'
    }
}
```

## 4. 脚本式 Pipeline

```groovy
// Scripted Pipeline 示例
node('docker-agent') {
    def appName = 'spring-boot-app'
    def version = '1.0.0'
    
    try {
        // 阶段 1: 拉取代码
        stage('Checkout') {
            checkout scm
            version = sh(
                returnStdout: true,
                script: 'git describe --tags --always'
            ).trim()
        }
        
        // 阶段 2: 构建
        stage('Build') {
            def maven = docker.image('maven:3.8-openjdk-11')
            maven.inside('-v /root/.m2:/root/.m2') {
                sh 'mvn clean package -DskipTests'
            }
        }
        
        // 阶段 3: 并行测试
        stage('Test') {
            parallel(
                'Unit Test': {
                    sh 'mvn test'
                    junit 'target/surefire-reports/*.xml'
                },
                'Integration Test': {
                    sh 'mvn verify -Pit'
                },
                'Code Coverage': {
                    sh 'mvn jacoco:report'
                    publishHTML([
                        reportDir: 'target/site/jacoco',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            )
        }
        
        // 阶段 4: 构建镜像
        stage('Docker Build') {
            def image = docker.build("${appName}:${version}")
            docker.withRegistry('https://registry.example.com', 'cred-id') {
                image.push()
            }
        }
        
        // 阶段 5: 部署
        stage('Deploy') {
            if (env.BRANCH_NAME == 'master') {
                input '确认部署到生产环境？'
                deploy('prod', version)
            } else {
                deploy('dev', version)
            }
        }
        
        currentBuild.result = 'SUCCESS'
    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        // 清理
        cleanWs()
        notifySlack(currentBuild.result, version)
    }
}

// 自定义函数
def deploy(String env, String version) {
    sh """
        helm upgrade --install ${appName} ./helm-chart \\
            --set image.tag=${version} \\
            --namespace ${env}
    """
}

def notifySlack(String result, String version) {
    def color = result == 'SUCCESS' ? 'good' : 'danger'
    slackSend(
        color: color,
        message: "${appName} ${version} 构建${result}"
    )
}
```

## 5. 多环境流水线

```
多环境部署流程:

代码提交
    │
    ▼
┌──────────┐
│  Build   │ 编译、单元测试、打包
│  (Dev)   │
└────┬─────┘
     │ 自动触发
     ▼
┌──────────┐
│  Deploy  │ 部署到开发环境
│  (Dev)   │
└────┬─────┘
     │ 自动触发
     ▼
┌──────────┐
│  Test    │ 集成测试、接口测试
│  (Test)  │
└────┬─────┘
     │ 手动确认
     ▼
┌──────────┐
│  Deploy  │ 部署到测试环境
│  (Test)  │
└────┬─────┘
     │ 手动确认
     ▼
┌──────────┐
│  Deploy  │ 部署到预发环境
│  (Staging)│
└────┬─────┘
     │ 审批流程
     ▼
┌──────────┐
│  Deploy  │ 部署到生产环境
│  (Prod)  │ 蓝绿部署/金丝雀发布
└──────────┘
```

```groovy
// 多环境 Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Deploy to Dev') {
            steps {
                deploy('dev')
            }
        }
        
        stage('Integration Test') {
            steps {
                sh 'mvn verify -Pit'
            }
        }
        
        stage('Deploy to Test') {
            when {
                branch 'develop'
            }
            steps {
                deploy('test')
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'release/*'
            }
            steps {
                deploy('staging')
            }
        }
        
        stage('Approval') {
            when {
                branch 'master'
            }
            steps {
                input(
                    message: '批准生产部署？',
                    submitter: 'admin',
                    parameters: [
                        choice(
                            name: 'STRATEGY',
                            choices: ['canary', 'blue-green', 'rolling'],
                            description: '部署策略'
                        )
                    ]
                )
            }
        }
        
        stage('Deploy to Prod') {
            when {
                branch 'master'
            }
            steps {
                script {
                    def strategy = inputStrategy ?: 'rolling'
                    deploy('prod', strategy)
                }
            }
        }
    }
}

def deploy(String env, String strategy = 'rolling') {
    sh "./deploy.sh ${env} ${strategy}"
}
```

## 6. Shared Library（共享库）

```groovy
// vars/standardPipeline.groovy（共享库）
def call(Map config) {
    pipeline {
        agent any
        
        environment {
            APP_NAME = config.appName
            DOCKER_IMAGE = config.dockerImage
        }
        
        stages {
            stage('Standard Build') {
                steps {
                    script {
                        standardBuild(config.buildTool ?: 'maven')
                    }
                }
            }
            
            stage('Standard Test') {
                steps {
                    runTests(config.testFramework ?: 'junit')
                }
            }
            
            stage('Standard Deploy') {
                when {
                    branch 'master'
                }
                steps {
                    deployApp(config.deployEnv ?: 'dev')
                }
            }
        }
    }
}

// 项目 Jenkinsfile
@Library('my-shared-library') _

standardPipeline(
    appName: 'order-service',
    buildTool: 'maven',
    dockerImage: 'openjdk:11-jre',
    deployEnv: 'kubernetes'
)
```

## 7. Blue Ocean 可视化

```
Blue Ocean 界面:

┌──────────────────────────────────────────────┐
│  Pipeline: order-service                     │
│  Branch: master                              │
│  Build #42  SUCCESS                          │
├──────────────────────────────────────────────┤
│                                              │
│  ○ Checkout      ──────────────────────  3s  │
│  ○ Build         ──────────────────────  45s │
│  ○ Unit Test     ──────────────────────  2m  │
│  ○ Code Analysis  ═════════════════════  1m  │
│    ├─ SonarQube   ────────────────────  45s  │
│    └─ SpotBugs    ────────────────────  15s  │
│  ○ Docker Build  ──────────────────────  30s │
│  ○ Deploy        ──────────────────────  10s │
│                                              │
│  总耗时: 6m 32s                              │
└──────────────────────────────────────────────┘
```

```groovy
// Blue Ocean 优化的 Pipeline（增加 stage 可视化）
pipeline {
    agent any
    
    stages {
        stage('Prepare') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()
                }
            }
        }
        
        stage('Build & Test') {
            parallel {
                stage('Backend') {
                    steps {
                        sh 'mvn clean package'
                    }
                    post {
                        always {
                            junit '**/target/surefire-reports/*.xml'
                        }
                    }
                }
                stage('Frontend') {
                    steps {
                        dir('frontend') {
                            sh 'npm ci'
                            sh 'npm run build'
                            sh 'npm run test:ci'
                        }
                    }
                }
            }
        }
    }
}
```

## 8. 分布式构建节点配置

```groovy
// 使用标签选择节点
pipeline {
    agent none  // 不在默认节点运行
    
    stages {
        stage('Build') {
            agent { label 'maven && jdk11' }
            steps {
                sh 'mvn clean package'
                stash includes: 'target/*.jar', name: 'artifacts'
            }
        }
        
        stage('Test') {
            agent { label 'docker && test' }
            steps {
                unstash 'artifacts'
                sh 'mvn test'
            }
        }
        
        stage('Deploy') {
            agent { label 'deploy' }
            steps {
                unstash 'artifacts'
                sh 'deploy.sh'
            }
        }
    }
}
```

### 8.1 动态代理（Kubernetes）

```groovy
// 在 K8s 中动态创建 Agent Pod
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.8-openjdk-11
                    command: ['cat']
                    tty: true
                    volumeMounts:
                    - name: m2-cache
                      mountPath: /root/.m2
                  - name: docker
                    image: docker:20.10
                    command: ['cat']
                    tty: true
                    volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                  volumes:
                  - name: m2-cache
                    persistentVolumeClaim:
                      claimName: maven-cache-pvc
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
            '''
        }
    }
    
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
        
        stage('Docker') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:latest .'
                }
            }
        }
    }
}
```

## 9. 最佳实践

1. **Pipeline as Code**:
   - 将 `Jenkinsfile` 放入代码仓库根目录
   - 通过 Multibranch Pipeline 自动扫描分支
   - 版本控制流水线变更

2. **安全与凭证管理**:
   - 使用 `withCredentials` 或 `credentials` 绑定
   - 不在代码中硬编码密码
   - 使用 Jenkins 凭证存储（Secret text/Username/SSH）

3. **错误处理与重试**:
   ```groovy
   steps {
       retry(3) {
           sh 'deploy.sh'
       }
   }
   ```

4. **超时控制**:
   ```groovy
   options {
       timeout(time: 30, unit: 'MINUTES')
   }
   ```

5. **通知与可视化**:
   - 集成 Slack/钉钉/企业微信
   - 使用 Blue Ocean 查看流水线状态
   - 发布测试报告和覆盖率

6. **性能优化**:
   - 使用 `stash/unstash` 传递产物
   - 并行执行无依赖的阶段
   - 缓存依赖（Maven/Gradle/npm）

7. **幂等性**:
   - 部署脚本支持重复执行
   - 使用蓝绿部署或金丝雀发布
   - 数据库迁移使用 Flyway/Liquibase

8. **审计与合规**:
   - 记录所有构建参数和结果
   - 保存构建产物和日志
   - 审批流程留痕
