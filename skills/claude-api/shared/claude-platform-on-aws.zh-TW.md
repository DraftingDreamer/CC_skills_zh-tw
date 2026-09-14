---
source_file: claude-platform-on-aws.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: d3ce9bc6ea5d9b9b6e9ec4f8aa8e28493a2a198a433c2297b377cbb84f86ad9d
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `claude-platform-on-aws.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# AWS 上的 Claude Platform

透過 AWS 基礎設施**由 Anthropic 自行運營**存取 Claude 開發者平台——SigV4 驗證、AWS IAM 存取控制，以及 AWS Marketplace 計費。由於 Anthropic 自行運營，**API 介面與第一方保持完全一致，具備同日更新的功能對等性**——針對個別功能的例外，請參見 `shared/platform-availability.md`（唯一事實來源；請勿依賴此處的內嵌例外清單）。模型 ID 使用裸第一方字串（`claude-opus-5`、`claude-sonnet-5`）——**不需要提供者前綴**。

> **與 Amazon Bedrock 不同。** Bedrock 是合作夥伴運營的（由 AWS 運行服務；發布時程不同、功能子集不同、模型 ID 帶 `anthropic.` 前綴）。AWS 上的 Claude Platform 與 Bedrock 共存；依需求選擇：是否需要具備完整 Anthropic API 對等性的 AWS 原生 IAM/計費（本頁），或是 Bedrock 自有的生態系。

---

## 客戶端與安裝

| 語言 | 安裝 | 客戶端 |
|---|---|---|
| Python | `pip install -U "anthropic[aws]"` | `from anthropic import AnthropicAWS` → `AnthropicAWS()` |
| TypeScript | `npm install @anthropic-ai/aws-sdk` | `import AnthropicAws from "@anthropic-ai/aws-sdk"` → `new AnthropicAws()` |
| Go | `go get github.com/anthropics/anthropic-sdk-go` | `import anthropicaws "github.com/anthropics/anthropic-sdk-go/aws"` → `anthropicaws.NewClient(ctx, anthropicaws.ClientConfig{})` |
| C# | `dotnet add package Anthropic.Aws` | `new AnthropicAwsClient()` |
| Java | 請參見 `shared/live-sources.md` 中的 SDK 存放庫 | 請參見 `shared/live-sources.md` 中的 SDK 存放庫 |
| Ruby | `gem install anthropic aws-sdk-core` | 請參見 `shared/live-sources.md` 中的 SDK 存放庫 |
| PHP | `composer require anthropic-ai/sdk aws/aws-sdk-php` | 請參見 `shared/live-sources.md` 中的 SDK 存放庫 |

建構後，**使用客戶端的方式與 `Anthropic()` 完全相同**——`client.messages.create(...)`、`client.beta.sessions.*` 等，使用裸模型 ID。

```python
from anthropic import AnthropicAWS

client = AnthropicAWS()  # region + workspace_id from env; see below
client.messages.create(
    model="claude-opus-5",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}],
)
```

---

## 必要設定

必須提供兩個值（建構函式引數或環境變數）——**兩者均無預設備援**：

| 值 | 環境變數 | 備註 |
|---|---|---|
| AWS 區域 | `AWS_REGION` | 必填。與 `AnthropicBedrock` 不同，沒有 `us-east-1` 備援。 |
| Workspace ID | `ANTHROPIC_AWS_WORKSPACE_ID` | 必填。將請求路由至您的 Claude workspace。 |

端點格式：`https://aws-external-anthropic.{region}.api.aws/v1/...`。請求使用服務名稱 `aws-external-anthropic` 進行 SigV4 簽署。

## 驗證

客戶端依照標準優先順序鏈解析 AWS 憑證：明確的建構函式引數 → 環境變數（`AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_SESSION_TOKEN`）→ 共用設定檔 → 假設角色/執行個體中繼資料。

對於 SigV4 不實用的情境（例如瀏覽器、簡單指令碼），也支援**短期 API 金鑰**。使用各語言的 token 產生器套件鑄造金鑰；將其作為 `api_key` 傳入客戶端。有效期為**以下最短值**：請求的持續時間、底層憑證的到期時間，以及 **12 小時**。套件名稱及 IAM 詳細資訊，請從 `shared/live-sources.md` WebFetch AWS 上的 Claude Platform 頁面。

---

## 應告知使用者的事項

- 將其視為第一方：本 skill 的每個章節均適用且無需更改。**請勿**套用 Bedrock 的功能可用性限制。僅有三項 Managed Agents 差異：(1) session 在沒有使用者事件的情況下最多可自主執行 **6 小時**，逾時即需重新驗證——傳送任一 user 角色的事件即可繼續；(2) **自託管**環境上的 session **無法附加記憶體存放區**（於建立 session 時即遭拒絕）——雲端環境則照常可以附加；(3) 自託管 worker 以 IAM/SigV4，或搭配 `AnthropicSelfHostedEnvironmentAccess` 受管政策（managed policy）的 AWS Console API 金鑰進行驗證——由 Console 產生的環境金鑰對 AWS 端點無效。
- 模型 ID 使用裸字串（`claude-opus-5`）。**請勿**新增 `anthropic.` 前綴。
- 遺漏 region 或 `workspace_id` 會在客戶端建構時拋出例外（不會送出任何請求）。**403** 表示請求已到達伺服器——請檢查 **錯誤的** `workspace_id` 或主體缺少 IAM 動作。IAM 動作參考請見 `shared/live-sources.md`。
