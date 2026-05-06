# Git 与 Jenkins 自动构建集成

## 1. Jenkins Git 集成概述

```
Jenkins + Git 集成架构:

┌─────────────────────────────────────────────────────────────┐
│              Jenkins Git 集成                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │  Git    │───→│ Jenkins │───→│ 构建    │                │
│  │ 仓库    │    │ CI/CD   │    │ 部署    │                │
│  └─────────┘    └─────────┘    └─────────┘                │
│       │              │              │                      │
│       │ Webhook      │ Pipeline     │ 产物                 │
│       │              │              │                      │
│       ▼              ▼              ▼                      │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐                │
│  │ Push    │    │ 自动    │    │ 制品库  │                │
│  │ PR      │    │ 触发    │    │ 镜像仓  │                │
│  │ Tag     │    │ 构建    │    │ 服务器  │                │
│  └─────────┘    └─────────┘    └─────────┘                │
│                                                             │
│  触发方式:                                                  │
│  ├── Webhook: Git 推送自动触发                             │
│  ├── Poll SCM: 定时轮询代码变更                            │
│  ├── 手动触发: 手动点击构建                                │
│  └── 定时触发: 定时构建（如每晚构建）                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. Jenkins Git 插件配置

### 2.1 安装 Git 插件

```
Jenkins → Manage Jenkins → Plugins → Available plugins

必需插件:
├── Git plugin                  # Git 核心插件
├── GitHub plugin               # GitHub 集成
├── GitLab plugin               # GitLab 集成
├── Blue Ocean                  # 现代化 UI
├── Pipeline                    # Pipeline 支持
└── Credentials Binding         # 凭证管理
```

### 2.2 配置 Git 凭证

```
Jenkins → Manage Jenkins → Credentials → System → Global credentials

凭证类型:
├── Username with password      # HTTPS 方式
├── SSH Username with private key  # SSH 方式
└── Secret text                 # Token 方式

推荐: 使用 SSH 密钥或 Personal Access Token
```

## 3. Pipeline 集成 Git

### 3.1 基础 Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    // Git 仓库配置
    environment {
        GIT_REPO = 'https://github.com/user/repo.git'
        GIT_BRANCH = 'main'
    }
    
    stages {
        stage('Checkout') {
            steps {
                // 拉取代码
                git branch: "${GIT_BRANCH}",
                    url: "${GIT_REPO}",
                    credentialsId: 'github-credentials'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

### 3.2 多分支 Pipeline

```groovy
// 多分支 Pipeline（自动识别分支）
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Deploy Dev') {
            when {
                branch 'develop'
            }
            steps {
                sh './deploy.sh dev'
            }
        }
        
        stage('Deploy Prod') {
            when {
                branch 'main'
            }
            steps {
                sh './deploy.sh prod'
            }
        }
    }
}
```

## 4. Webhook 配置

### 4.1 GitHub Webhook

```
GitHub → Settings → Webhooks → Add webhook

配置:
├── Payload URL: http://jenkins.example.com/github-webhook/
├── Content type: application/json
├── Secret: <webhook-secret>
├── Events:
│   ├── Just the push event
│   ├── Pull requests
│   └── Releases
└── Active: ✓
```

### 4.2 GitLab Webhook

```
GitLab → Settings → Webhooks → Add webhook

配置:
├── URL: http://jenkins.example.com/project/<project-name>
├── Secret Token: <token>
├── Trigger:
│   ├── Push events
│   ├── Merge request events
│   └── Tag push events
└── SSL verification: Enable/Disable
```

### 4.3 Jenkins Webhook 配置

```groovy
// Jenkinsfile - 配置触发器
pipeline {
    agent any
    
    triggers {
        // GitHub Webhook
        githubPush()
        
        // GitLab Webhook
        gitlab(triggerOnPush: true, triggerOnMR: true)
        
        // Poll SCM（备用）
        pollSCM('H/5 * * * *')  // 每5分钟检查一次
    }
    
    stages {
        // ...
    }
}
```

## 5. 分支策略与构建

### 5.1 Git Flow 集成

```groovy
// Jenkinsfile - Git Flow 集成
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }
        
        stage('Integration Test') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'release/*'
                }
            }
            steps {
                sh 'mvn verify -P integration-test'
            }
        }
        
        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                sh './deploy.sh dev'
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'release/*'
            }
            steps {
                sh './deploy.sh staging'
            }
        }
        
        stage('Deploy to Prod') {
            when {
                branch 'main'
            }
            steps {
                input message: '确认部署到生产环境？'
                sh './deploy.sh prod'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            // 发送成功通知
            emailext subject: "构建成功: ${env.JOB_NAME}",
                     body: "详情: ${env.BUILD_URL}",
                     to: 'team@example.com'
        }
        failure {
            // 发送失败通知
            emailext subject: "构建失败: ${env.JOB_NAME}",
                     body: "详情: ${env.BUILD_URL}",
                     to: 'team@example.com'
        }
    }
}
```

### 5.2 PR 检查 Pipeline

```groovy
// PR 检查 Pipeline
pipeline {
    agent any
    
    triggers {
        githubPRRequest()
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Code Analysis') {
            parallel {
                stage('Compile') {
                    steps {
                        sh 'mvn clean compile'
                    }
                }
                stage('Unit Test') {
                    steps {
                        sh 'mvn test'
                    }
                }
                stage('Code Style') {
                    steps {
                        sh 'mvn checkstyle:check'
                    }
                }
                stage('SpotBugs') {
                    steps {
                        sh 'mvn spotbugs:check'
                    }
                }
            }
        }
        
        stage('Integration Test') {
            steps {
                sh 'mvn verify -P integration-test'
            }
        }
        
        stage('Code Coverage') {
            steps {
                sh 'mvn jacoco:report'
                jacoco execPattern: 'target/jacoco.exec'
            }
        }
    }
    
    post {
        always {
            // 发布测试结果
            junit 'target/surefire-reports/*.xml'
            
            // 发布覆盖率报告
            publishHTML([
                reportDir: 'target/site/jacoco',
                reportFiles: 'index.html',
                reportName: 'Coverage Report'
            ])
        }
    }
}
```

## 6. Git Tag 与版本发布

```groovy
// 版本发布 Pipeline
pipeline {
    agent any
    
    parameters {
        string(name: 'VERSION', defaultValue: '', description: '版本号')
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: '部署环境')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn clean package -Drevision=${params.VERSION}"
            }
        }
        
        stage('Create Tag') {
            when {
                expression { params.VERSION != '' }
            }
            steps {
                sh """
                    git tag -a v${params.VERSION} -m "Release ${params.VERSION}"
                    git push origin v${params.VERSION}
                """
            }
        }
        
        stage('Deploy') {
            steps {
                sh "./deploy.sh ${params.ENV} ${params.VERSION}"
            }
        }
    }
}
```

## 7. Jenkins Shared Library

### 7.1 目录结构

```
jenkins-shared-library/
├── vars/
│   ├── mavenBuild.groovy
│   ├── dockerBuild.groovy
│   └── deployApp.groovy
├── src/
│   └── com/
│       └── example/
│           └── Pipeline.groovy
└── resources/
    └── templates/
        └── deployment.yaml
```

### 7.2 Shared Library 示例

```groovy
// vars/mavenBuild.groovy
def call(Map config = [:]) {
    def mavenVersion = config.mavenVersion ?: '3.9'
    def javaVersion = config.javaVersion ?: '17'
    def goals = config.goals ?: 'clean verify'
    
    withMaven(maven: mavenVersion, jdk: javaVersion) {
        sh "mvn ${goals}"
    }
    
    // 发布测试结果
    junit 'target/surefire-reports/*.xml'
    
    // 发布覆盖率
    jacoco execPattern: 'target/jacoco.exec'
}

// vars/deployApp.groovy
def call(Map config = [:]) {
    def environment = config.environment
    def version = config.version
    
    sh """
        echo "Deploying ${version} to ${environment}"
        ./deploy.sh ${environment} ${version}
    """
}
```

### 7.3 使用 Shared Library

```groovy
// Jenkinsfile
@Library('my-shared-library') _

pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                mavenBuild(
                    javaVersion: '17',
                    goals: 'clean package'
                )
            }
        }
        
        stage('Deploy') {
            steps {
                deployApp(
                    environment: 'prod',
                    version: env.BUILD_NUMBER
                )
            }
        }
    }
}
```

## 8. 最佳实践

```
Jenkins Git 集成 Checklist:

□ 凭证管理:
  - 使用 Jenkins Credentials 存储凭证
  - 不要在 Pipeline 中硬编码密码
  - 定期轮换凭证

□ 触发策略:
  - 生产环境使用 Webhook 触发
  - 配置 Poll SCM 作为备用
  - PR 检查自动触发

□ 分支策略:
  - main/develop 自动构建
  - feature 分支 PR 检查
  - release 分支部署到 staging

□ 构建优化:
  - 使用 Pipeline 缓存依赖
  - 并行执行测试
  - 增量构建

□ 通知机制:
  - 构建失败立即通知
  - PR 状态更新
  - 部署结果通知

□ 安全考虑:
  - PR 检查限制权限
  - 生产部署需要审批
  - 敏感信息加密存储
```
