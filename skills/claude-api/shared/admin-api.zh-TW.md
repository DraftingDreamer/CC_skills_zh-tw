---
source_file: admin-api.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 50d79d03f3a4ccae038efb5b9805d9d21838486c601f2dfb7b212585c308ef64
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `admin-api.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Admin API（組織管理）

當使用者想要以程式化方式管理其 Anthropic 組織時，請讀取本檔：成員與角色、邀請、工作區與工作區成員、API 金鑰、速率限制報表、服務帳戶、workload identity federation（WIF），或 customer-managed encryption key（CMEK）。

Admin API 位於 `https://api.anthropic.com/v1/organizations/*` 之下。它管理的是組織本身——不會傳送訊息。截至**2026 年 8 月 26 日**，全部七種 SDK（Python、TypeScript、C#、Go、Java、PHP、Ruby）皆已在 `client.beta.organization` 下提供此功能，`ant` CLI 也已在 `ant beta:organization` 下提供。用量報表、成本報表，以及 Claude Enterprise 的使用者管理與分析端點**未**納入 SDK——請以原始 HTTP 呼叫這些端點。

## 驗證方式

兩種憑證類型，皆由預設 SDK 用戶端與 CLI 自動讀取：

| 憑證 | 環境變數 | HTTP 標頭 | 涵蓋範圍 |
| --- | --- | --- | --- |
| Admin API 金鑰（`sk-ant-admin...`） | `ANTHROPIC_API_KEY` | `x-api-key` | 大多數端點 |
| `org:admin` OAuth token | `ANTHROPIC_AUTH_TOKEN` | `authorization: Bearer` | 所有端點，包含僅限 OAuth 的端點 |

- **僅限 OAuth 的端點：** 服務帳戶、federation issuer 與 federation rule 會拒絕 API 金鑰——它們需要 `org:admin` OAuth token。
- **優先順序陷阱：** 當兩個環境變數同時設定時，部分用戶端會優先採用 API 金鑰。使用 bearer token 時，請在該 shell 中保持 `ANTHROPIC_API_KEY` 未設定。
- Admin API 金鑰由組織的 admin 在 Claude Console 中建立。
- 一般（非 admin）API 金鑰在這些端點上一律無法使用，admin 憑證也無法用於 Messages API。
- `org:admin` token 會授予對整個組織的存取權，無論任何工作區綁定為何。

**互動式 OAuth token** - 在專用 profile 下用 `ant` CLI 登入（讓例行指令不會以提升過的權限執行），接著匯出 token。Token 存活時間短；遇到 401 時，重新執行匯出。Profile 與範圍的運作機制（為什麼 `org:admin` 需要明確的 `--scope`、如何切換 profile）：見 `shared/anthropic-cli.md`。

```bash
ant auth login --profile admin --scope "org:admin"
export ANTHROPIC_AUTH_TOKEN=$(ant auth print-credentials --profile admin --access-token)
# When done: unset ANTHROPIC_AUTH_TOKEN && ant profile activate default
```

**自動化工作負載（CI）** - 不要以互動方式登入。建立一條 `oauth_scope: org:admin` 的 federation rule，指向 `organization_role` 為 `admin` 的服務帳戶（這一條規則必須由真人在 Claude Console 中建立），接著透過 federation 環境變數把用戶端指向它，並以不帶引數的方式建構——SDK/CLI 會自動執行 token 交換，並在到期前自動重新整理：

```bash
export ANTHROPIC_FEDERATION_RULE_ID=fdrl_...       # the org:admin rule
export ANTHROPIC_ORGANIZATION_ID=<org-uuid>
export ANTHROPIC_SERVICE_ACCOUNT_ID=svac_...       # the rule's target service account
export ANTHROPIC_IDENTITY_TOKEN_FILE=/path/to/jwt  # or ANTHROPIC_IDENTITY_TOKEN
```

**curl** 每個請求也都需要帶上 `anthropic-version: 2023-06-01`。

## 端點涵蓋範圍

SDK 存取器以 Python 拼寫方式呈現；各語言的命名慣例請見下方表格。

| 資源 | REST 路徑 | SDK 存取器（`client.beta.organization` +） | CLI（`ant beta:organization` +） |
| --- | --- | --- | --- |
| 組織資訊 | `GET /v1/organizations/me` | `.retrieve()` | `retrieve` |
| 成員 | `/v1/organizations/users` | `.users` - `list`、`update`、`remove` | `:users list\|update\|remove` |
| 邀請 | `/v1/organizations/invites` | `.invites` - `create`、`list`、`delete` | `:invites create\|list\|delete` |
| 工作區 | `/v1/organizations/workspaces` | `.workspaces` - `create`、`retrieve`、`list`、`update`、`archive` | `:workspaces create\|list\|update\|archive` |
| 工作區成員 | `/v1/organizations/workspaces/{id}/members` | `.workspaces.members` - `add`、`list`、`update`、`remove` | `:workspaces:members add\|list\|update\|remove` |
| API 金鑰 | `/v1/organizations/api_keys` | `.api_keys` - `list`、`update` | `:api-keys list\|update` |
| 組織速率限制 | `GET /v1/organizations/rate_limits` | `.rate_limits.list(model=..., group_type=...)` | `:rate-limits list` |
| 工作區速率限制 | `GET /v1/organizations/workspaces/{id}/rate_limits` | `.workspaces.rate_limits.list(workspace_id)` | `:workspaces:rate-limits list` |
| 服務帳戶 (*) | `/v1/organizations/service_accounts` | `.service_accounts` - `create`、`list`、`archive` | `:service-accounts create\|list\|archive` |
| Federation issuers (*) | `/v1/organizations/federation_issuers` | `.federation.issuers` - `create`、`list`、`archive` | `:federation:issuers create\|list\|archive` |
| Federation rules (*) | `/v1/organizations/federation_rules` | `.federation.rules` - `create`、`list`、`archive` | `:federation:rules create\|list\|archive` |
| CMEK 外部金鑰 | `/v1/organizations/external_keys` | `.external_keys` - `create`、`validate` | - |

(*) 僅限 OAuth：需要 `org:admin` bearer token，而非 API 金鑰。

將 CMEK 外部金鑰附加到工作區，是一次工作區更新操作：`client.beta.organization.workspaces.update("<workspace-id>", external_key_id="ekey_...")`。

## 各語言命名慣例與分頁

| 語言 | 存取器語法（以列出成員為例） | List 行為 |
| --- | --- | --- |
| Python | `client.beta.organization.users.list(limit=10)` | 疊代器自動擷取更多分頁；`limit` = 分頁大小，而非總數 |
| TypeScript | `client.beta.organization.users.list({ limit: 10 })` - camelCase 子資源：`apiKeys`、`rateLimits`、`serviceAccounts`、`externalKeys` | `for await` 自動分頁 |
| C# | `client.Beta.Organization.Users.List(new() { Limit = 10 })` | `await foreach (var u in page.Paginate())` 自動分頁 |
| Go | `client.Beta.Organization.Users.ListAutoPaging(ctx, params)`；組織資訊為 `Organization.Get(ctx)` | `.Next()` / `.Current()` 自動分頁 |
| Java | `client.beta().organization().users().list(params)`，搭配 builder 參數（`UserListParams.builder().limit(10).build()`） | `.autoPager()` 自動分頁 |
| PHP | `$client->beta->organization->users->list(limit: 10)` | 原始單分頁資料呼叫－請疊代 `->getItems()`；SDK 的自動分頁輔助工具尚未串接到這些端點 |
| Ruby | `client.beta.organization.users.list(limit: 10)` | 原始單分頁資料呼叫－請疊代 `.data`；SDK 的自動分頁輔助工具尚未串接到這些端點 |
| CLI | `ant beta:organization:users list --limit 10` | 在成員、邀請、工作區、工作區成員與 API 金鑰清單上，`--limit` 是結果數量上限（不同於大多數 `ant` list 指令——那些指令的 `--limit` 是設定分頁大小、`--max-items` 才是上限，見 `shared/anthropic-cli.md`） |
| curl | `GET /v1/organizations/users?limit=10` | 每個請求回傳一分頁；游標分頁依 Admin API 參考文件而定 |

速率限制清單（`rate_limits`、`workspaces.rate_limits`）自推出時起同樣支援分頁——請比照其他列表端點進行分頁，不要假設回應只有單一分頁。

Go 的參數型別遵循 `anthropic.BetaOrganizationUserListParams` 的模式（`Limit` 對應 `anthropic.Int(10)`）；Java 參數則使用 `com.anthropic.models.beta.organization.*` 提供的 builder（例如 `UserListParams.builder().limit(10).build()`）。Go 與 Java 的分頁迴圈：

```go
users := client.Beta.Organization.Users.ListAutoPaging(ctx, anthropic.BetaOrganizationUserListParams{Limit: anthropic.Int(10)})
for users.Next() {
	user := users.Current() // ...
}
if err := users.Err(); err != nil { /* handle */ }
```

```java
for (var user : client.beta().organization().users().list(params).autoPager()) { /* ... */ }
```

## 範例

常見操作（以 Python 拼寫方式呈現；請對照上表映射到其他語言——每個操作在各語言中的形狀都相同）：

```python
# Organization info
org = client.beta.organization.retrieve()

# List members (iterator auto-fetches more pages; limit = page size)
for user in client.beta.organization.users.list(limit=10):
    print(f"{user.id}: {user.email} ({user.role})")

# Change a member's role / remove a member
client.beta.organization.users.update("user_...", role="developer")
client.beta.organization.users.remove("user_...")

# Invite someone
client.beta.organization.invites.create(email="user@example.com", role="developer")

# Create a workspace and add a member to it
ws = client.beta.organization.workspaces.create(name="Production")
client.beta.organization.workspaces.members.add(
    ws.id, user_id="user_...", workspace_role="workspace_developer"
)

# Deactivate / rename an API key
client.beta.organization.api_keys.update("apikey_...", status="inactive", name="New Key Name")

# Rate limit reports (optional filters: model=..., group_type=...)
client.beta.organization.rate_limits.list(model="claude-opus-5")
client.beta.organization.workspaces.rate_limits.list("wrkspc_...")

# Service accounts + WIF (org:admin OAuth token required)
sa = client.beta.organization.service_accounts.create(name="inference-worker", organization_role="developer")
issuer = client.beta.organization.federation.issuers.create(
    name="github-actions",
    issuer_url="https://token.actions.githubusercontent.com",
    jwks={"type": "discovery"},
)
client.beta.organization.federation.rules.create(
    name="gha-deploy",
    issuer_id=issuer.id,
    match={"subject_prefix": "repo:my-org/my-repo:ref:refs/heads/main",
           "claims": {"repository_owner": "my-org"}},
    target={"type": "service_account", "service_account_id": sa.id},
    workspace_id="wrkspc_...",
    oauth_scope="workspace:developer",
    token_lifetime_seconds=600,
)

# CMEK: register, validate, then attach an external key to a workspace
key = client.beta.organization.external_keys.create(
    display_name="prod-key", geo="us",
    provider_config={"type": "aws", "kms_arn": "arn:aws:kms:..."},
)
client.beta.organization.external_keys.validate(key.id)
client.beta.organization.workspaces.update("wrkspc_...", external_key_id=key.id)
```

## 組織角色

| 角色 | 權限 |
| --- | --- |
| `user` | Playground |
| `claude_code_user` | Playground + Claude Code |
| `developer` | Playground + 管理 API 金鑰 |
| `billing` | Playground + 管理帳單 |
| `admin` | 以上皆是 + 管理使用者 |

擁有者（owner）與主要擁有者（primary owner）擁有所有 admin 權限，且可另外管理 admin。工作區角色有 `workspace_user`、`workspace_developer`、`workspace_admin` 與 `workspace_billing`。

## 平台限制

- **Claude Platform on AWS：** 僅工作區端點可用。成員、工作區成員、邀請、API 金鑰，以及用量／成本／速率限制報表皆不可用。CMEK 外部金鑰端點在該平台上尚未提供——請在 Claude Console 中註冊並附加金鑰。
- **Claude Enterprise（claude.ai organization）：** 此介面僅提供成員與邀請功能，另加上不在 SDK 中的 Enterprise 專屬端點（群組與自訂角色讀取、支出限制）。

## 即時文件

| 主題 | URL |
| --- | --- |
| Admin API 指南 | `https://platform.claude.com/docs/en/manage-claude/admin-api.md` |
| Admin API 參考文件 | `https://platform.claude.com/docs/en/api/admin.md` |
| 工作區 | `https://platform.claude.com/docs/en/manage-claude/workspaces.md` |
| 速率限制 API | `https://platform.claude.com/docs/en/manage-claude/rate-limits-api.md` |
| WIF 管理 | `https://platform.claude.com/docs/en/manage-claude/wif-admin-api.md` |
| 用量與成本報表（僅限 curl） | `https://platform.claude.com/docs/en/manage-claude/usage-cost-api.md` |
