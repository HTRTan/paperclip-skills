---
name: deepseek-api
description: 當需要選擇模型、配置推理參數或使用 DeepSeek API 時，提供詳細指引與最佳實踐。適用於 deepseek-engineer 在協助使用者整合 DeepSeek 模型到應用程式或優化推理效能時參考。
---

# DeepSeek API 使用指南

本指南涵蓋 DeepSeek API 的核心功能、模型選擇、推理參數配置、程式碼範例與最佳實務，幫助你有效整合與使用 DeepSeek 模型。

## 目錄

1. [模型選擇](#模型選擇)
2. [推理配置](#推理配置)
3. [API 使用方式](#api-使用方式)
4. [程式碼範例](#程式碼範例)
5. [最佳實踐](#最佳實踐)
6. [故障排除](#故障排除)

---

## 模型選擇

DeepSeek 提供多種模型，分別針對不同任務優化。請根據應用場景選擇合適的模型。

| 模型 ID                 | 說明                                                                 | 最大上下文 | 適用場景                                                                 |
|------------------------|----------------------------------------------------------------------|------------|--------------------------------------------------------------------------|
| `deepseek-chat`        | 最新版對話模型（DeepSeek-V3），具備強大的通用能力、推理與程式碼理解。    | 1M tokens  | 一般對話、內容生成、分析、問答、程式碼撰寫與除錯。                          |
| `deepseek-reasoner`    | 深度推理模型（DeepSeek-R1），擅長逐步邏輯推理與複雜問題拆解。           | 1M tokens  | 數學證明、策略規劃、科學解釋、需要鏈式思考的任務。                          |
| `deepseek-coder`       | 專注程式碼的模型，訓練於大量程式碼語料，支援多種程式語言。               | 16K tokens | 程式碼生成、補全、重構、解釋、文件撰寫。                                   |

> **注意**：`deepseek-chat` 與 `deepseek-reasoner` 支援長達 100 萬 tokens 的上下文，適合處理大型文件或長對話。`deepseek-coder` 的上下文為 16K tokens，更適合短到中長度的程式碼任務。

### 選擇建議

- **日常對話與通用任務**：使用 `deepseek-chat`，平衡效能與成本。
- **複雜推理或需要詳細步驟解釋**：選用 `deepseek-reasoner`，其推理鏈更嚴謹。
- **程式碼專用場景**：優先考慮 `deepseek-coder`，在程式碼生成與理解上表現更佳。
- **長文件處理**：僅 `deepseek-chat` 與 `deepseek-reasoner` 支援 1M 上下文，適合總結論文、分析程式碼庫或長篇書籍。

---

## 推理配置

透過調整推理參數，可以控制模型輸出的隨機性、長度與行為。所有參數皆為選填，並有預設值。

### 核心參數

| 參數                | 類型    | 預設值 | 範圍/選項                      | 說明                                                                 |
|---------------------|---------|--------|--------------------------------|----------------------------------------------------------------------|
| `temperature`       | float   | 1.0    | 0.0 ~ 2.0                      | 控制隨機性。較低值使輸出更確定、一致；較高值增加多樣性與創造力。         |
| `top_p`             | float   | 1.0    | 0.0 ~ 1.0                      | 核取樣（nucleus sampling），僅從機率總和達到 top_p 的最小詞彙集合中取樣。 |
| `max_tokens`        | integer | 4096   | 1 ~ 8192（deepseek-coder 為 4096） | 生成回覆的最大 token 數。不包含提示的 token 數。                         |
| `stop`              | string/array | null | 字串或最多 16 個字串的清單 | 停止生成的字串。模型遇到這些字串時停止輸出，結果中不包含停止字串。         |
| `frequency_penalty` | float   | 0.0    | -2.0 ~ 2.0                     | 根據 token 在文本中出現的頻率懲罰。正值減少重複字詞。                   |
| `presence_penalty`  | float   | 0.0    | -2.0 ~ 2.0                     | 根據 token 是否已出現過懲罰。正值鼓勵引入新主題。                         |
| `stream`            | boolean | false  | true/false                     | 啟用串流輸出，逐步回傳 token，適合即時顯示。                            |
| `response_format`   | object  | null   | `{ "type": "json_object" }`    | 強制模型輸出合法的 JSON 物件。需在提示中明確要求 JSON 格式。              |

### 參數調校建議

- **創造性任務**（寫詩、腦力激盪）：`temperature=0.8~1.2`, `top_p=0.9`，可搭配較高的 `presence_penalty`。
- **精準任務**（擷取資料、程式碼生成）：`temperature=0.0~0.3`, `top_p=0.1`，關閉隨機性。
- **重複控制**：若輸出出現重複迴圈，調高 `frequency_penalty` 至 0.5~1.0。
- **主題多樣性**：需要模型探索新概念時，使用 `presence_penalty=0.5~1.0`。
- **長輸出**：根據任務需求設定 `max_tokens`，避免浪費 token 或截斷重要內容。

> **提示**：`temperature` 與 `top_p` 不建議同時調整。一般只調整 `temperature` 或僅調整 `top_p`。

---

## API 使用方式

DeepSeek API 相容 OpenAI API 格式，但使用不同的基礎 URL 與 API 金鑰。

### 基礎資訊

- **API 端點**：`https://api.deepseek.com/v1/chat/completions`
- **認證方式**：Bearer Token（API Key）
- **請求方法**：POST
- **內容類型**：`application/json`

### 取得 API Key

1. 註冊 [DeepSeek 開發者平台](https://platform.deepseek.com/)。
2. 前往「API Keys」頁面建立新金鑰。
3. 妥善保存金鑰，僅顯示一次。

### 請求結構

```json
{
  "model": "deepseek-chat",
  "messages": [
    {"role": "system", "content": "你是一個專業的技術助理。"},
    {"role": "user", "content": "解釋什麼是 transformer 架構。"}
  ],
  "temperature": 0.7,
  "max_tokens": 1000,
  "stream": false
}
```

### 回應格式

非串流回應（`stream=false`）範例：

```json
{
  "id": "chatcmpl-xxxx",
  "object": "chat.completion",
  "created": 1700000000,
  "model": "deepseek-chat",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Transformer 是一種基於自注意力機制的神經網路架構..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 150,
    "total_tokens": 170
  }
}
```

串流回應（`stream=true`）會以 Server-Sent Events (SSE) 格式回傳多個資料區塊，每個區塊為：

```
data: {"id":"...","choices":[{"delta":{"content":"..."}}]}

data: [DONE]
```

### 錯誤處理

HTTP 狀態碼與對應含義：

| 狀態碼 | 錯誤類型                 | 說明與處理方式                                                         |
|--------|--------------------------|------------------------------------------------------------------------|
| 400    | Bad Request              | 請求參數錯誤（如模型名稱無效、max_tokens 超出範圍）。檢查請求內容。       |
| 401    | Unauthorized             | API Key 無效或缺失。確認 Authorization 標頭與金鑰正確性。               |
| 402    | Payment Required         | 帳戶餘額不足或配額用盡。請充值或檢查用量。                               |
| 429    | Too Many Requests        | 超過速率限制（預設每分鐘 60 次請求）。請降低頻率或實作指數退避重試。     |
| 500    | Internal Server Error    | DeepSeek 服務端暫時性錯誤。可稍後重試，建議使用指數退避策略。            |
| 503    | Service Unavailable      | 服務繁忙或離線。同 500 處理方式。                                       |

---

## 程式碼範例

### cURL 範例

```bash
curl -X POST https://api.deepseek.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "用 Python 寫一個快速排序函式"}],
    "temperature": 0.2,
    "max_tokens": 500
  }'
```

### Python 範例

使用 `openai` 函式庫（因為 DeepSeek API 相容）：

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.environ.get("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com/v1"
)

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": "你是一位資深軟體工程師。"},
        {"role": "user", "content": "解釋非同步程式設計的優缺點。"}
    ],
    temperature=0.5,
    max_tokens=800,
    stream=False
)

print(response.choices[0].message.content)
```

串流範例：

```python
stream = client.chat.completions.create(
    model="deepseek-chat",
    messages=[{"role": "user", "content": "寫一首關於 AI 的短詩"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### 使用推理模型（deepseek-reasoner）

```python
response = client.chat.completions.create(
    model="deepseek-reasoner",
    messages=[{"role": "user", "content": "如果地球半徑增加一倍，表面重力會如何改變？請逐步推理。"}],
    temperature=0.3,
    max_tokens=2000
)
# deepseek-reasoner 的回應通常會包含詳細的思考步驟
print(response.choices[0].message.content)
```

---

## 最佳實踐

### 提示工程

- **明確指令**：告訴模型希望得到的輸出格式、風格或限制。
- **使用 system message**：設定角色或行為規則，例如「你是一個只回應 JSON 的助理」。
- **少樣本提示**：在 user 訊息中提供範例，引導模型輸出模式。
- **分隔上下文**：利用多輪對話保持記憶，或明確標記文件邊界（如 `<document>` 標籤）。

### 速率限制與成本優化

- 預設速率限制：每分鐘 60 次請求（基於 API Key）。如需提升，請聯絡 DeepSeek 支援。
- **節省 token**：縮短 system message，避免不必要的對話歷史。設定合理的 `max_tokens`。
- **快取常見回應**：對於靜態查詢，可快取模型輸出以減少 API 呼叫。
- **批次處理**：將多個小任務合併成一個提示（如果邏輯允許），節省請求次數。

### 穩定性與重試

- 實作指數退避重試（exponential backoff）處理 `429`、`500`、`503` 錯誤。
- 設定請求逾時（建議 60 秒以上，尤其是長上下文或高 `max_tokens` 請求）。
- 使用 `stream=True` 可提早獲得部分回應，避免 TCP 逾時。

### 安全與合規

- 不要在提示或對話中洩漏敏感資訊（API Key、密碼、個人資料）。
- DeepSeek API 可能記錄請求資料以改善模型，請勿傳送受管制或機密內容。

---

## 故障排除

| 問題                         | 可能原因與解決方案                                                                   |
|------------------------------|--------------------------------------------------------------------------------------|
| 收到 401 錯誤                | API Key 無效或未正確設定 `Authorization` 標頭。確認金鑰並使用 `Bearer ` 前綴。       |
| 回應中斷或內容不完整         | `max_tokens` 太小，導致模型提前停止。增加 `max_tokens` 或檢查 `finish_reason` 是否為 `length`。 |
| 模型拒絕回答或回覆無關內容   | 提示可能觸發安全過濾。調整提問方式，避免要求違規內容。                               |
| 串流回應沒有即時顯示         | 客戶端未正確解析 SSE 格式。確認逐行讀取 `data: ` 前綴，並處理 `[DONE]` 訊號。          |
| 回應速度慢                   | 檢查網路延遲；嘗試使用更接近的地理區域（DeepSeek 主機位於亞洲）。減少 `max_tokens` 或 `temperature` 對速度影響不大，但較短的提示與輸出會更快。 |
| 模型輸出重複或陷入迴圈       | 調高 `frequency_penalty` 或 `presence_penalty`，或降低 `temperature`。               |

### 獲取更多協助

- 官方文檔：[DeepSeek API Docs](https://platform.deepseek.com/api-docs/)
- 開發者社群：[GitHub Discussions](https://github.com/deepseek-ai/DeepSeek-API/discussions)
- 支援信箱：support@deepseek.com

---

## 總結

- 根據任務選擇 `deepseek-chat`（通用）、`deepseek-reasoner`（推理）或 `deepseek-coder`（程式碼）。
- 調整 `temperature` 與 `top_p` 控制隨機性；使用懲罰參數避免重複。
- API 相容 OpenAI，只需修改 `base_url` 與 API Key。
- 實作錯誤處理與重試邏輯，以確保生產環境穩定。
- 善用提示工程與 system message 提升輸出品質。

此文件可作為開發者整合 DeepSeek API 時的完整參考。