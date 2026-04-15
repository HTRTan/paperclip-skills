---
name: jenkins
description: 用於 CI/CD Pipeline 變更或部署協調。當需要編寫、修改、審查 Jenkins Pipeline (Declarative/Scripted)、配置多分支流水線、處理構建觸發、集成部署任務或排查 Jenkins 相關問題時使用此技能。
---

# Jenkins 技能

## 適用場景
- 創建或修改 `Jenkinsfile`（Declarative 或 Scripted Pipeline）
- 添加/調整構建階段（stage）、步驟（step）、並行任務
- 配置環境變量、憑證、參數化構建
- 集成代碼掃描、單元測試、打包、鏡像構建、部署等環節
- 排查 Pipeline 語法錯誤或運行時問題
- 設計多分支或共享庫（Shared Library）結構
- 與 Jenkins API 交互（觸發構建、查詢狀態）

## 核心原則

1. **優先使用 Declarative Pipeline** – 除非有複雜的動態邏輯或共享庫需求，否則 Declarative 更易讀、更安全。
2. **顯式定義 agent** – 明確指定 `agent any`、`agent { label 'xxx' }` 或容器化 agent。
3. **敏感信息使用憑證** – 避免硬編碼密鑰/密碼，使用 `withCredentials` 或 `credentials()`。
4. **階段化與可視化** – 每個邏輯單元獨立成 `stage`，方便 Blue Ocean 等工具查看。
5. **錯誤處理與 post** – 總是定義 `post` 部分的 `always`、`success`、`failure` 來執行清理或通知。
6. **超時與重試** – 對網絡或外部依賴步驟設置 `timeout` 和 `retry`。

## 典型任務指南

### 1. 生成 Declarative Pipeline 骨架

根據用戶需求輸出類似以下結構：

```groovy
pipeline {
    agent any
    options {
        timeout(time: 1, unit: 'HOURS')
        ansiColor('xterm')
    }
    environment {
        // 環境變量示例
        APP_NAME = 'my-app'
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
        stage('Test') {
            steps {
                sh 'make test'
            }
            post {
                always {
                    junit '**/test-report.xml'
                }
            }
        }
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh 'deploy.sh'
            }
        }
    }
    post {
        failure {
            emailext subject: "Pipeline failed: ${env.JOB_NAME}",
                    body: "Build ${env.BUILD_URL} failed.",
                    to: 'team@example.com'
        }
    }
}
```

### 2. 添加並行測試或部署任務

使用 `parallel` 關鍵字：

```groovy
stage('Parallel Tests') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'npm test' }
        }
        stage('Integration Tests') {
            steps { sh 'make integration-test' }
        }
    }
}
```

### 3. 使用憑證

```groovy
withCredentials([string(credentialsId: 'api-key', variable: 'API_KEY')]) {
    sh 'curl -H "X-API-Key: $API_KEY" https://api.example.com/deploy'
}
```

### 4. 觸發下游項目

```groovy
build job: 'deploy-pipeline', parameters: [
    string(name: 'VERSION', value: env.BUILD_NUMBER)
], wait: false
```

### 5. 共享庫調用示例

若用戶提到共享庫，建議如下結構：

```groovy
@Library('my-shared-library') _
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                myOrg.buildAndTest()
            }
        }
    }
}
```

### 6. 排查常見錯誤

| 錯誤信息 | 可能原因 | 解決方案 |
|---------|----------|----------|
| `groovy.lang.MissingPropertyException` | 變量未定義或作用域錯誤 | 使用 `env.VAR` 訪問環境變量，或使用 `script{}` 塊定義局部變量 |
| `java.io.NotSerializableException` | 非序列化對象被緩存 | 將對象操作放在 `script{}` 內，避免跨節點傳遞 |
| `No such DSL method` | 語法錯誤或缺少插件 | 檢查步驟名稱是否正確，確認插件已安裝 |

## 對話與交互規範

- **主動澄清**：若用戶未指定 agent 類型（master、node、容器），請詢問其運行環境。
- **推薦最佳實踐**：當用戶提供硬編碼憑證時，主動建議改用 Jenkins 憑證存儲。
- **提供可運行片段**：儘量輸出可直接複製到 `Jenkinsfile` 的代碼，而非僅理論說明。
- **必要時引用 Jenkins 文檔**：如用戶問到不常見的步驟（如 `kubernetesDeploy`），請給出官方鏈接。

## 擴展能力

- 生成 Jenkins 配置即代碼（JCasC）YAML
- 編寫 Groovy 腳本初始化或管理節點
- 通過 REST API 實現 CI/CD 外部協調（使用 `curl` 或 Jenkins CLI）

## 示例對話

**用戶**：我想在 Jenkins Pipeline 中構建 Docker 鏡像並推送到私有倉庫，該怎麼寫？

**助手（遵循本技能）**：
1. 首先確保節點支持 Docker（agent 標籤 `docker` 或使用 `docker` pipeline 步驟）
2. 使用 `docker.build()` 和 `docker.withRegistry()`：
   ```groovy
   pipeline {
       agent any
       environment {
           DOCKER_REGISTRY = 'registry.example.com'
           IMAGE_NAME = "${DOCKER_REGISTRY}/myapp:${env.BUILD_NUMBER}"
       }
       stages {
           stage('Build Image') {
               steps {
                   script {
                       docker.build(IMAGE_NAME)
                   }
               }
           }
           stage('Push Image') {
               steps {
                   script {
                       docker.withRegistry('https://registry.example.com', 'docker-credentials-id') {
                           docker.image(IMAGE_NAME).push()
                       }
                   }
               }
           }
       }
   }
   ```
3. 提醒用戶需提前在 Jenkins 中配置 `docker-credentials-id` 憑證（用戶名/密碼）。