以下是您所需的 `cicd-pipelines` 技能包，涵盖 Jenkins 与 GitHub Actions 的核心知识、可直接使用的流水线模板以及参考指南。

```markdown
---
name: cicd-pipelines
description: 提供 Jenkins 和 GitHub Actions 的 CI/CD 流水线知识、工作流模板与参考指南。包含声明式 Pipeline、GitHub Actions 工作流、多语言构建示例、最佳实践及故障排查。
---

# CI/CD Pipelines 技能

本技能提供 Jenkins 与 GitHub Actions 的流水线设计知识、可复用模板及最佳实践参考，帮助您快速搭建、优化和维护 CI/CD 流程。

## 何时使用

- 用户询问如何编写 Jenkinsfile 或 GitHub Actions 工作流
- 需要构建、测试、部署的模板（Node.js、Python、Java、Go、.NET 等）
- 想了解多分支流水线、矩阵策略、条件触发、手动审批等高级用法
- 需要排查流水线失败、性能优化、安全加固建议
- 寻求 Jenkins 共享库或 GitHub Actions 可复用工作流的设计模式

## 核心概念速查

### Jenkins 关键要素

| 概念 | 说明 |
|------|------|
| `agent` | 指定流水线执行节点（any、label、docker、none） |
| `stages` / `stage` | 定义任务阶段，每个 stage 包含多个 `steps` |
| `environment` | 设置环境变量，支持凭证绑定 |
| `options` | 配置超时、重试、时间戳等 |
| `parameters` | 参数化构建（字符串、布尔、选择等） |
| `triggers` | 触发方式（cron、pollSCM、webhook） |
| `post` | 后置操作（always、success、failure、changed） |
| `tools` | 自动安装 JDK、Maven、Node.js 等工具 |
| `when` | 条件判断执行阶段 |
| `parallel` | 并行执行阶段或步骤 |

### GitHub Actions 关键要素

| 概念 | 说明 |
|------|------|
| `on` | 触发事件（push、pull_request、schedule、workflow_dispatch） |
| `jobs` | 一组作业，默认并行执行 |
| `runs-on` | 指定运行器环境（ubuntu-latest、windows-latest、macos-latest 或自托管） |
| `steps` | 顺序执行的命令或动作 |
| `uses` | 复用社区或官方动作（如 actions/checkout@v4） |
| `with` | 向动作传递参数 |
| `env` | 作业或步骤级的环境变量 |
| `matrix` | 多维度并行策略（版本、操作系统等） |
| `if` | 条件执行作业或步骤 |
| `needs` | 设置作业依赖关系 |
| `continue-on-error` | 允许步骤失败而不阻塞 |
| `timeout-minutes` | 作业超时设置 |

## Jenkins 流水线模板

### 基础声明式 Pipeline（多语言通用）

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven-3.9'      // 若需要
        nodejs 'Node-20'       // 若需要
    }
    
    environment {
        // 定义全局环境变量
        APP_NAME = 'my-app'
        BUILD_VERSION = "${env.BUILD_NUMBER}"
        // 从 Jenkins 凭证获取
        DOCKER_CREDS = credentials('docker-hub-credentials')
    }
    
    options {
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        ansiColor('xterm')
    }
    
    triggers {
        cron('H 2 * * *')              // 每日定时
        pollSCM('H/5 * * * *')         // 每5分钟检查代码变化
        // 或使用 GitHub webhook（需插件）
    }
    
    parameters {
        string(name: 'TARGET_ENV', defaultValue: 'staging', description: '部署环境')
        choice(name: 'BRANCH', choices: ['main', 'develop'], description: '分支')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh 'npm install'        // Node.js
                // sh 'pip install -r requirements.txt'  // Python
                // sh 'dotnet restore'   // .NET
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm test'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                }
            }
        }
        
        stage('Build & Package') {
            steps {
                sh 'npm run build'
                // 构建 Docker 镜像示例
                script {
                    docker.build("myapp:${BUILD_VERSION}")
                }
            }
        }
        
        stage('Deploy') {
            when {
                expression { params.TARGET_ENV == 'production' }
                branch 'main'
            }
            steps {
                sh './deploy.sh ${TARGET_ENV}'
            }
        }
    }
    
    post {
        always {
            // 清理工作区或发送通知
            cleanWs()
        }
        success {
            emailext (
                subject: "构建成功: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "详情见 ${env.BUILD_URL}",
                to: 'team@example.com'
            )
        }
        failure {
            // 失败处理逻辑
        }
    }
}
```

### 多分支流水线配置 (Jenkinsfile)

```groovy
// 适用于多分支流水线项目，根据分支自动构建
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'make build-prod'
                    } else {
                        sh 'make build-dev'
                    }
                }
            }
        }
    }
}
```

### 与 Docker 集成示例

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v /tmp:/tmp'
        }
    }
    stages {
        stage('Test') {
            steps {
                sh 'node --version'
                sh 'npm install && npm test'
            }
        }
    }
}
```

### 共享库引用 (vars/)

```groovy
@Library('my-shared-library') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                notifySlack.startBuild()
                sh 'make build'
                notifySlack.buildSuccess()
            }
        }
    }
}
```

## GitHub Actions 工作流模板

### 基础 Node.js CI/CD 工作流

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:       # 手动触发

env:
  NODE_VERSION: '20.x'

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Run tests
        run: npm test
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.node-version }}
          path: test-results/

  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: npm run build
      - name: Archive production artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
  
  docker-build-and-push:
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: user/app:latest
```

### Python + 多环境测试

```yaml
name: Python CI

on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11']
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install tox
      - name: Run tests
        run: tox -e py
```

### 多平台矩阵构建（Go 示例）

```yaml
name: Go Cross-Platform Build

on:
  release:
    types: [created]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        goos: [linux, windows, darwin]
        goarch: [amd64, arm64]
        exclude:
          - goos: windows
            goarch: arm64
    steps:
      - uses: actions/checkout@v4
      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      - name: Build
        env:
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
        run: go build -o myapp-${{ matrix.goos }}-${{ matrix.goarch }} .
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: myapp-${{ matrix.goos }}-${{ matrix.goarch }}
          path: myapp-*
```

### 可复用工作流（被调用方）

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow
on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: '20.x'
    secrets:
      API_TOKEN:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci && npm test
```

### 调用可复用工作流（主工作流）

```yaml
name: Main CI

on: [push]

jobs:
  call-test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: '18.x'
    secrets:
      API_TOKEN: ${{ secrets.MY_TOKEN }}
```

### 通用最佳实践

1. **保持流水线短小高效**：每个阶段只做一件事，失败快速返回。
2. **使用凭证管理**：不要硬编码密钥。Jenkins 使用 Credentials Binding，GitHub Actions 使用 Secrets。
3. **缓存依赖**：Jenkins 使用 `caches` 或 `Shared Workspace`，GitHub Actions 使用 `actions/cache`。
4. **固定版本标签**：避免使用 `latest`，如 `actions/checkout@v4` 确保稳定性。
5. **并行加速**：合理拆分独立任务，使用 `parallel` 或 `matrix` 策略。
6. **后置清理**：确保工作区清理，防止磁盘占满。
7. **原子性发布**：使用标签或哈希进行不可变发布。

### 安全性建议

- **最小权限原则**：仅授予构建所需的权限（如 Jenkins 节点隔离、GitHub 的 `contents:read`）。
- **审计所有外部动作**：审查第三方 GitHub Actions 的源代码。
- **对敏感步骤使用自托管运行器**：但注意维护安全更新。
- **定期更新插件和动作**：避免已知漏洞。

### 性能优化

- Jenkins: 使用 `agent none` 并在每个 `stage` 指定 `agent` 来动态分配节点。
- GitHub Actions: 使用 `actions/cache` 缓存包管理器目录（npm、pip、maven）。
- 限制不必要的触发器：`paths-ignore` 或 `branches-ignore` 避免文档变更触发构建。

### 官方文档与扩展资源

| 平台 | 链接 |
|------|------|
| Jenkins Pipeline 语法参考 | [https://www.jenkins.io/doc/book/pipeline/syntax/](https://www.jenkins.io/doc/book/pipeline/syntax/) |
| GitHub Actions 工作流语法 | [https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) |
| Jenkins 共享库 | [https://www.jenkins.io/doc/book/pipeline/shared-libraries/](https://www.jenkins.io/doc/book/pipeline/shared-libraries/) |
| GitHub Actions 可复用工作流 | [https://docs.github.com/en/actions/using-workflows/reusing-workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) |
| 常用 Jenkins 插件 | Blue Ocean, Docker Pipeline, Pipeline Utility Steps |
| 常用 GitHub Actions | `actions/checkout`, `actions/setup-node`, `docker/build-push-action` |

## 故障排查速查表

| 问题 | Jenkins 解决方案 | GitHub Actions 解决方案 |
|------|------------------|--------------------------|
| 构建超时 | 增加 `options { timeout(...) }` | 添加 `timeout-minutes: 60` |
| 缓存未命中 | 使用 `cache` 指令或 `caches` 插件 | 检查 `cache` 动作的 `key` 和 `restore-keys` |
| 测试报告未显示 | 确保 `junit` 步骤路径正确 | 使用 `actions/upload-artifact` 并搭配测试报告动作 |
| 权限拒绝 (Docker) | 添加用户到 `docker` 组或使用 `docker` agent | 使用 `docker/login-action` 并正确设置 secrets |
| 并行步骤失败回滚 | `failFast: true` 在 `parallel` 块 | 使用 `if: always()` 执行后置清理 |

## 使用建议

当用户请求具体的流水线实现时：
1. 根据平台（Jenkins/GitHub Actions）选择合适的模板。
2. 询问项目语言、构建工具、测试命令、部署目标。
3. 按需调整模板中的 `environment`、`triggers`、`matrix` 及后置操作。
4. 提醒用户设置必要的凭证和权限。
5. 如果涉及多分支或复杂发布策略，引用最佳实践部分。

--- 

本技能旨在加速 CI/CD 流水线开发，所有模板可根据实际需求自由修改。如需更深入的场景（如 Kubernetes 部署、蓝绿发布、Canary），请进一步描述需求。
```