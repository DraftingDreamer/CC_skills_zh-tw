---
source_file: error-codes.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 1bfe197bcf264c8b86c1815cf5ee367bdbfa80ba0ac20041713ffdfa504491c8
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `error-codes.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# HTTP 錯誤碼參考

本文件說明 Claude API 回傳的 HTTP 錯誤碼、常見原因及處理方式。語言特定的錯誤處理範例，請參見 `python/` 或 `typescript/` 資料夾。

## 錯誤碼摘要

| 碼 | 錯誤類型 | 可重試 | 常見原因 |
| --- | --- | --- | --- |
| 400 | `invalid_request_error` | 否 | 無效的請求格式或參數 |
| 401 | `authentication_error` | 否 | 無效或缺少 API 金鑰 |
| 403 | `permission_error` | 否 | API 金鑰缺少權限 |
| 404 | `not_found_error` | 否 | 無效的端點或模型 ID |
| 413 | `request_too_large` | 否 | 請求超過大小限制 |
| 429 | `rate_limit_error` | 是 | 請求太多 |
| 500 | `api_error` | 是 | Anthropic 服務問題 |
| 529 | `overloaded_error` | 是 | API 暫時過載 |

## 詳細錯誤資訊

### 400 Bad Request

**原因：**

- 請求主體中的 JSON 格式錯誤
- 缺少必填參數（`model`、`max_tokens`、`messages`）
- 無效的參數類型（例如期望整數但收到字串）
- 空的 messages 陣列
- messages 未交替使用 user/assistant

**錯誤範例：**

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "messages: roles must alternate between \"user\" and \"assistant\""
  },
  "request_id": "req_011CSHoEeqs5C35K2UUqR7Fy"
}
```

**修正：** 在送出前驗證請求結構。確認：

- `model` 是有效的模型 ID
- `max_tokens` 是正整數
- `messages` 陣列非空且正確交替

---

### 401 Unauthorized

**原因：**

- 缺少 `x-api-key` header 或 `Authorization` header
- 無效的 API 金鑰格式
- 已復原或刪除的 API 金鑰
- OAuth bearer token 透過 `x-api-key` 而非 `Authorization: Bearer` 送出
- 同時設定了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`——SDK 會同時送出兩個 header，API 拒絕請求

**修正：** 設定 `ANTHROPIC_API_KEY`，或執行 `ant auth login` 並讓客戶端建構器保持空白。使用 OAuth token 的原始 HTTP 請用 `Authorization: Bearer <token>`（而非 `x-api-key:`）。

---

### 403 Forbidden

**原因：**

- API 金鑰沒有存取所請求模型的權限
- 組織層級限制
- 嘗試在沒有 beta 存取權限的情況下使用 beta 功能

**修正：** 在 Console 中查看您的 API 金鑰權限。您可能需要不同的 API 金鑰或申請特定功能的存取權限。

---

### 404 Not Found

**原因：**

- 模型 ID 錯字（例如 `claude-sonnet-4.6` 而非 `claude-sonnet-4-6`）
- 使用已棄用的模型 ID
- 無效的 API 端點

**修正：** 使用模型說明文件中的精確模型 ID。您可以使用別名（例如 `claude-opus-5`）。

---

### 413 Request Too Large

**原因：**

- 請求主體超過最大大小
- 輸入中的 token 太多
- 圖片資料太大

**修正：** 縮小輸入——截斷對話歷史、壓縮/縮小圖片，或將大型文件分割成小塊。

---

### 400 驗證錯誤

某些 400 錯誤專門與參數驗證相關：

- `max_tokens` 超過模型限制
- 無效的 `temperature` 值（必須為 0.0-1.0）
- extended thinking 中 `budget_tokens` >= `max_tokens`
- 無效的工具定義綱要

**Claude Opus 5 / Fable 5/5.1 / Opus 4.8 / 4.7 特定的 400 錯誤：**

- `temperature`、`top_p`、`top_k` 已移除——送出任何一個都回傳 400。請刪除該參數；請參見 `shared/model-migration.md` → 每個 SDK 語法參考。
- `thinking: {type: "enabled", budget_tokens: N}` 已移除——送出此參數回傳 400。請改用 `thinking: {type: "adaptive"}`。
- **Claude Opus 5：** 當 `effort` 為 `xhigh` 或 `max` 時，`thinking: {type: "disabled"}` 回傳 400——在 `high` 或以下可接受。thinking 預設啟用，因此省略參數執行 adaptive 而非停用它。
- **僅限 Fable 5/5.1：** 明確的 `thinking: {type: "disabled"}` 在任何 effort 下都回傳 400（在 Opus 4.8/4.7 上可接受）。請完全省略 `thinking` 參數。
- **Fable 5/5.1、Mythos 5/5.1：** 若組織或 workspace 設定為零資料保留（ZDR）——或任何低於所需 30 天的保留——則**所有**送往這些模型的請求都會回傳 `400 invalid_request_error`（"In order to access this model, your organization or workspace must have data retention enabled."），即使是完全有效的 payload 也一樣；且 ZDR 須經 Anthropic 明確授權才能啟用。在偵錯請求主體之前，請先查看保留設定。
- **Claude Fable 5.1 / Claude Mythos 5.1（以及 Mythos Preview）：** `tool_choice: {type: "any"}` 或 `{type: "tool", name: ...}` 回傳 400 `tool_choice: type "tool" and "any" are not supported for this model.`——`count_tokens` 與 Batches 上同樣如此。請改用 `{type: "auto"}`，並在 prompt 中指名工具（引數需符合綱要時用 `strict: true`），或改用結構化輸出。
- **Claude Fable 5.1 / Claude Mythos 5.1——preserved thinking／歷史編輯檢查（2026-08-31 當天或之後建立的新帳號，或任何有設定 `prefix_mismatch_behavior` 的請求）：** ``messages.N.content.M: Invalid `signature` in `thinking` block. The block is bound to a different conversation. Remove the block, or set `thinking.block_binding.prefix_mismatch_behavior` to "drop_block".``（若沒送出 beta header，還會多一句指名該 header；有時也會再加一句指出是哪一則訊息開始變動的）代表自從產生該 thinking 區塊之後，系統 prompt、工具清單或更早的訊息已經變動過。重送相同的請求主體並不會清除這個錯誤；`count_tokens` 也會回傳相同的 400。（在 Message Batches API 中，*未設定*時的預設行為是捨棄出錯的區塊而不讓整個項目失敗——只有設定了 `prefix_mismatch_behavior: "error"`，Batches 項目才會以 `errored` 狀態失敗。）請去除該指名的區塊與其後的每個 thinking 區塊後重試一次；或者在 beta `thinking-binding-controls-2026-08-01` 下，改用 `thinking.block_binding.prefix_mismatch_behavior: "drop_block"` 重新送出（這個 controls beta 目前的提供範圍：發布時 Claude API／Claude Platform on AWS 即支援，Bedrock 與 Google Cloud 依模型而定，Foundry 上不提供，詳見 `shared/platform-availability.md`；未提供的地方請改走去除後重試的路徑；未送出該 header 時，這個欄位會回傳以 `block_binding: Extra inputs are not permitted` 結尾的 400）；接著請修正您的 harness，讓它不再編輯歷史紀錄（請參見 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1）。若同樣的開頭子句*沒有*「bound to a different conversation」這句話，代表的是遭竄改的簽名——一律回傳 400，與設定無關。

**舊版模型（Opus 4.6 及更早）extended thinking 的常見錯誤：**

```
# Wrong: budget_tokens must be < max_tokens
thinking: budget_tokens=10000, max_tokens=1000  -> Error!

# Correct
thinking: budget_tokens=10000, max_tokens=16000
```

---

### 429 Rate Limited

**原因：**

- 超過每分鐘請求數（RPM）
- 超過每分鐘 token 數（TPM）
- 超過每天 token 數（TPD）

**要查看的 headers：**

- `retry-after`：重試前等待的秒數
- `x-ratelimit-limit-*`：您的限制
- `x-ratelimit-remaining-*`：剩餘配額

**修正：** Anthropic SDK 自動以指數退避重試 429 和 5xx 錯誤（預設：`max_retries=2`）。如需自訂重試行為，請參見語言特定的錯誤處理範例。

---

### 500 Internal Server Error

**原因：**

- 暫時性 Anthropic 服務問題
- API 處理中的錯誤

**修正：** 以指數退避重試。若持續發生，請查看 [status.anthropic.com](https://status.anthropic.com)。

---

### 529 Overloaded

**原因：**

- API 需求量高
- 服務容量達到上限

**修正：** 以指數退避重試。考慮使用不同的模型（Haiku 通常負載較低）、將請求分散到不同時間，或實作請求佇列。

---

## 常見錯誤與修正

| 錯誤 | 錯誤碼 | 修正 |
| --- | --- | --- |
| Claude Opus 5 / Fable 5/5.1 / Opus 4.8 / 4.7 使用 `temperature`/`top_p`/`top_k` | 400 | 移除參數（請參見 `shared/model-migration.md`） |
| Claude Opus 5 / Fable 5/5.1 / Opus 4.8 / 4.7 使用 `budget_tokens` | 400 | 改用 `thinking: {type: "adaptive"}` |
| Fable 5/5.1 使用 `thinking: {type: "disabled"}` | 400 | 完全省略 `thinking` 參數（Opus 4.8/4.7 可接受） |
| 組織設定為 ZDR / 保留低於 30 天（Fable 5/5.1、Mythos 5/5.1） | 每個請求都 400 | 修正組織的資料保留設定——payload 沒問題 |
| Claude Fable 5.1 / Claude Mythos 5.1 / Mythos Preview 使用 `any` / `tool` 類型的 `tool_choice` | 400 | 改用 `{type: "auto"}`，並在 prompt 中指名工具（引數需符合綱要時用 `strict: true`），或改用結構化輸出 |
| 帶 thinking 區塊重播已編輯過的歷史紀錄（Claude Fable 5.1 / Claude Mythos 5.1，preserved thinking） | 400 `Invalid signature in thinking block ... bound to a different conversation` | 停止編輯歷史紀錄——讓對話紀錄維持只能附加（append-only），改用對話中途的 `role: "system"` / 工具異動訊息、絕不刪除的輪次範圍 `clear_at` 提醒、伺服器端情境編輯，以及僅摘要式的壓縮來取代編輯；可一次性復原：去除該指名的區塊與其後每個 thinking 區塊（文字與工具呼叫保留），或設定 `prefix_mismatch_behavior: "drop_block"` |
| 送出 `thinking.block_binding` 卻未帶 `thinking-binding-controls-2026-08-01` | 400 `block_binding: Extra inputs are not permitted` | 在有提供這個 controls beta 的地方請送出該 beta header（`shared/platform-availability.md`）；其他地方請移除 `block_binding`，改用去除後重試 |
| `budget_tokens` >= `max_tokens`（舊版模型） | 400 | 確保 `budget_tokens` < `max_tokens` |
| 模型 ID 錯字 | 404 | 使用有效的模型 ID，例如 `claude-opus-5` |
| 第一條訊息是 `assistant` | 400 | 第一條訊息必須是 `user` |
| 連續相同角色的訊息 | 400 | 交替使用 `user` 和 `assistant` |
| 程式碼中的 API 金鑰 | 401（金鑰洩漏） | 使用環境變數 |
| 自訂重試需求 | 429/5xx | SDK 自動重試；使用 `max_retries` 自訂 |

## SDK 中的型別化例外

**務必使用 SDK 的型別化例外類別**，而非以字串比對錯誤訊息。每個 HTTP 狀態碼都對應到每個 SDK 中的特定例外類別。

### 各語言的例外類別名稱

| HTTP | Python（`anthropic.*`）/ TypeScript（`Anthropic.*`） | Ruby（`Anthropic::Errors::*`） | Java（`com.anthropic.errors.*`） | C# | PHP（`Anthropic\Core\Exceptions\*`） |
|---|---|---|---|---|---|
| 400 | `BadRequestError` | `BadRequestError` | `BadRequestException` | `AnthropicBadRequestException` | `BadRequestException` |
| 401 | `AuthenticationError` | `AuthenticationError` | `UnauthorizedException` | `AnthropicUnauthorizedException` | `AuthenticationException` |
| 403 | `PermissionDeniedError` | `PermissionDeniedError` | `PermissionDeniedException` | `AnthropicForbiddenException` | `PermissionDeniedException` |
| 404 | `NotFoundError` | `NotFoundError` | `NotFoundException` | `AnthropicNotFoundException` | `NotFoundException` |
| 422 | `UnprocessableEntityError` | `UnprocessableEntityError` | `UnprocessableEntityException` | `AnthropicUnprocessableEntityException` | `UnprocessableEntityException` |
| 429 | `RateLimitError` | `RateLimitError` | `RateLimitException` | `AnthropicRateLimitException` | `RateLimitException` |
| ≥500 | `InternalServerError` | `InternalServerError` | `InternalServerException` | `Anthropic5xxException` | `InternalServerException` |
| net | `APIConnectionError` | `APIConnectionError` | `AnthropicIoException` | `AnthropicIOException` | `APIConnectionException` |
| base | `APIError`（兩者）；`APIStatusError`（僅 Python） | `APIStatusError` / `APIError` | `AnthropicServiceException` | `AnthropicApiException` | `APIStatusException` / `APIException` |

Ruby 和 PHP 類別位於專用的錯誤命名空間——寫 `Anthropic::Errors::RateLimitError` 和 `Anthropic\Core\Exceptions\RateLimitException`（而非裸露的 `Anthropic::RateLimitError`）。所有 4xx C# 例外也繼承自 `Anthropic4xxException`。

### 最具體的先捕獲，形成鏈

從最具體的子類別到基礎類別排列 `catch`/`except`/`rescue` 子句，對每個不同處理的類別使用單獨的子句——可重試的（429、≥500、網路）與不可重試的（4xx）。SDK 為每個狀態碼定義了不同的類別，正是為了這個原因；單一的廣泛萬用捕獲會丟棄這些資訊。

```python
try:
    msg = client.messages.create(...)
except anthropic.NotFoundError as e:          # 404 - e.g. bad model ID
    ...
except anthropic.RateLimitError as e:         # 429 - back off and retry
    ...
except anthropic.APIStatusError as e:         # any other non-2xx HTTP response
    print(e.status_code, e.message)
except anthropic.APIConnectionError as e:     # network failure before a response
    ...
```

相同的鏈式結構適用於每個 SDK：TypeScript `instanceof Anthropic.NotFoundError` → `RateLimitError` → `APIConnectionError` → `APIError`（在 TypeScript SDK 中 `APIConnectionError` 是 `APIError` 的子類別，不像 Python 中是兄弟；先查 `APIConnectionError` 再查 `APIError`）；Ruby `rescue Anthropic::Errors::NotFoundError` → `…::RateLimitError` → `…::APIStatusError`；Java `catch (NotFoundException) … catch (RateLimitException) … catch (AnthropicServiceException)`；C# `catch (AnthropicNotFoundException) … catch (AnthropicRateLimitException) … catch (AnthropicApiException)`；PHP `catch (NotFoundException) … catch (RateLimitException) … catch (APIStatusException)`。

### Go — `errors.As` 後依狀態分流

Go SDK 對所有非 2xx 回應回傳單一的 `*anthropic.Error`。用 `errors.As` 解包，然後依 `StatusCode` 分流：

```go
_, err := client.Messages.New(ctx, params)
if err != nil {
    var apierr *anthropic.Error
    if errors.As(err, &apierr) {
        switch apierr.StatusCode {
        case 404:
            // bad model ID / resource
        case 429:
            // back off and retry
        default:
            // other API error - apierr.StatusCode, apierr.RequestID
        }
    } else {
        // transport-level error (*url.Error wrapping *net.OpError, etc.)
    }
}
```

### 錯誤的 `.type` 欄位

所有 `APIStatusError` 子類別現在都公開 `.type` 屬性（Python：`.type`，TypeScript：`.type`，Java：`.errorType()`，Go：`.Type()`，Ruby：`.type`，PHP：`.type`），回傳 API 錯誤類型字串（例如 `"invalid_request_error"`、`"authentication_error"`、`"rate_limit_error"`、`"overloaded_error"`）。當您需要比 HTTP 狀態碼更細的粒度時，請用此進行程式化錯誤分類——例如區分 `"billing_error"` 和 `"permission_error"`（兩者都對應 403）。

```python
except anthropic.APIStatusError as e:
    if e.type == "rate_limit_error":
        # handle rate limiting
    elif e.type == "overloaded_error":
        # handle overload
```
