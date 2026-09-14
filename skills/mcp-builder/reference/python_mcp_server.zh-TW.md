---
source_file: python_mcp_server.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 2da52f77e675191014ca2e146a4b95aa04d0ca7dd7e2b100322df15ade685e80
translated_at: 2026-08-28
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`python_mcp_server.md`](python_mcp_server.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# Python MCP 伺服器實作指南

## 概述

本文件提供使用 MCP Python SDK 實作 MCP 伺服器的 Python 專屬最佳實踐和範例。涵蓋伺服器設定、工具註冊模式、使用 Pydantic 進行輸入驗證、錯誤處理，以及完整的可運行範例。

---

## 快速參考

### 關鍵 import
```python
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field, field_validator, ConfigDict
from typing import Optional, List, Dict, Any
from enum import Enum
import httpx
```

### 伺服器初始化
```python
mcp = FastMCP("service_mcp")
```

### 工具註冊模式
```python
@mcp.tool(name="tool_name", annotations={...})
async def tool_function(params: InputModel) -> str:
    # Implementation
    pass
```

---

## MCP Python SDK 與 FastMCP

官方 MCP Python SDK 提供 FastMCP，這是一個用於建立 MCP 伺服器的高層次框架。它提供：
- 從函式簽名和文件字串自動生成說明和 inputSchema
- Pydantic 模型整合用於輸入驗證
- 以 `@mcp.tool` 裝飾器進行工具註冊

**完整 SDK 文件請使用 WebFetch 載入：**
`https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md`

## 伺服器命名慣例

Python MCP 伺服器必須遵循此命名模式：
- **格式**：`{service}_mcp`（小寫加底線）
- **範例**：`github_mcp`、`jira_mcp`、`stripe_mcp`

名稱應：
- 通用（不與特定功能綁定）
- 描述所整合的服務/API
- 易於從任務描述推斷
- 不含版本號或日期

## 工具實作

### 工具命名

工具名稱使用 snake_case（例如 "search_users"、"create_project"、"get_channel_info"），並使用清晰的、以動作為導向的名稱。

**避免命名衝突**：包含服務情境以防止重疊：
- 使用 "slack_send_message" 而非僅 "send_message"
- 使用 "github_create_issue" 而非僅 "create_issue"
- 使用 "asana_list_tasks" 而非僅 "list_tasks"

### 使用 FastMCP 的工具結構

工具使用 `@mcp.tool` 裝飾器定義，並使用 Pydantic 模型進行輸入驗證：

```python
from pydantic import BaseModel, Field, ConfigDict
from mcp.server.fastmcp import FastMCP

# Initialize the MCP server
mcp = FastMCP("example_mcp")

# Define Pydantic model for input validation
class ServiceToolInput(BaseModel):
    '''Input model for service tool operation.'''
    model_config = ConfigDict(
        str_strip_whitespace=True,  # Auto-strip whitespace from strings
        validate_assignment=True,    # Validate on assignment
        extra='forbid'              # Forbid extra fields
    )

    param1: str = Field(..., description="First parameter description (e.g., 'user123', 'project-abc')", min_length=1, max_length=100)
    param2: Optional[int] = Field(default=None, description="Optional integer parameter with constraints", ge=0, le=1000)
    tags: Optional[List[str]] = Field(default_factory=list, description="List of tags to apply", max_items=10)

@mcp.tool(
    name="service_tool_name",
    annotations={
        "title": "Human-Readable Tool Title",
        "readOnlyHint": True,     # Tool does not modify environment
        "destructiveHint": False,  # Tool does not perform destructive operations
        "idempotentHint": True,    # Repeated calls have no additional effect
        "openWorldHint": False     # Tool does not interact with external entities
    }
)
async def service_tool_name(params: ServiceToolInput) -> str:
    '''Tool description automatically becomes the 'description' field.

    This tool performs a specific operation on the service. It validates all inputs
    using the ServiceToolInput Pydantic model before processing.

    Args:
        params (ServiceToolInput): Validated input parameters containing:
            - param1 (str): First parameter description
            - param2 (Optional[int]): Optional parameter with default
            - tags (Optional[List[str]]): List of tags

    Returns:
        str: JSON-formatted response containing operation results
    '''
    # Implementation here
    pass
```

## Pydantic v2 關鍵功能

- 使用 `model_config` 而非巢狀的 `Config` 類別
- 使用 `field_validator` 而非已棄用的 `validator`
- 使用 `model_dump()` 而非已棄用的 `dict()`
- 驗證器需要 `@classmethod` 裝飾器
- 驗證器方法需要型別提示

```python
from pydantic import BaseModel, Field, field_validator, ConfigDict

class CreateUserInput(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True
    )

    name: str = Field(..., description="User's full name", min_length=1, max_length=100)
    email: str = Field(..., description="User's email address", pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$')
    age: int = Field(..., description="User's age", ge=0, le=150)

    @field_validator('email')
    @classmethod
    def validate_email(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Email cannot be empty")
        return v.lower()
```

## 回應格式選項

支援多種輸出格式以提供彈性：

```python
from enum import Enum

class ResponseFormat(str, Enum):
    '''Output format for tool responses.'''
    MARKDOWN = "markdown"
    JSON = "json"

class UserSearchInput(BaseModel):
    query: str = Field(..., description="Search query")
    response_format: ResponseFormat = Field(
        default=ResponseFormat.MARKDOWN,
        description="Output format: 'markdown' for human-readable or 'json' for machine-readable"
    )
```

**Markdown 格式**：
- 使用標題、清單和格式化以提升清晰度
- 將時間戳記轉換為人類可讀格式（例如 "2024-01-15 10:30:00 UTC" 而非 epoch）
- 以括號顯示 ID 搭配顯示名稱（例如 "@john.doe (U123456)"）
- 省略冗長的中繼資料（例如只顯示一個個人資料圖片 URL，而非所有尺寸）
- 以邏輯方式分組相關資訊

**JSON 格式**：
- 回傳完整的、適合程式化處理的結構化資料
- 包含所有可用欄位和中繼資料
- 使用一致的欄位名稱和型別

## 分頁實作

對於列出資源的工具：

```python
class ListInput(BaseModel):
    limit: Optional[int] = Field(default=20, description="Maximum results to return", ge=1, le=100)
    offset: Optional[int] = Field(default=0, description="Number of results to skip for pagination", ge=0)

async def list_items(params: ListInput) -> str:
    # Make API request with pagination
    data = await api_request(limit=params.limit, offset=params.offset)

    # Return pagination info
    response = {
        "total": data["total"],
        "count": len(data["items"]),
        "offset": params.offset,
        "items": data["items"],
        "has_more": data["total"] > params.offset + len(data["items"]),
        "next_offset": params.offset + len(data["items"]) if data["total"] > params.offset + len(data["items"]) else None
    }
    return json.dumps(response, indent=2)
```

## 錯誤處理

提供清晰、可操作的錯誤訊息：

```python
def _handle_api_error(e: Exception) -> str:
    '''Consistent error formatting across all tools.'''
    if isinstance(e, httpx.HTTPStatusError):
        if e.response.status_code == 404:
            return "Error: Resource not found. Please check the ID is correct."
        elif e.response.status_code == 403:
            return "Error: Permission denied. You don't have access to this resource."
        elif e.response.status_code == 429:
            return "Error: Rate limit exceeded. Please wait before making more requests."
        return f"Error: API request failed with status {e.response.status_code}"
    elif isinstance(e, httpx.TimeoutException):
        return "Error: Request timed out. Please try again."
    return f"Error: Unexpected error occurred: {type(e).__name__}"
```

## 共用工具函式

將常用功能提取為可重用的函式：

```python
# Shared API request function
async def _make_api_request(endpoint: str, method: str = "GET", **kwargs) -> dict:
    '''Reusable function for all API calls.'''
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method,
            f"{API_BASE_URL}/{endpoint}",
            timeout=30.0,
            **kwargs
        )
        response.raise_for_status()
        return response.json()
```

## Async/Await 最佳實踐

對網路請求和 I/O 操作始終使用 async/await：

```python
# Good: Async network request
async def fetch_data(resource_id: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(f"{API_URL}/resource/{resource_id}")
        response.raise_for_status()
        return response.json()

# Bad: Synchronous request
def fetch_data(resource_id: str) -> dict:
    response = requests.get(f"{API_URL}/resource/{resource_id}")  # Blocks
    return response.json()
```

## 型別提示

全程使用型別提示：

```python
from typing import Optional, List, Dict, Any

async def get_user(user_id: str) -> Dict[str, Any]:
    data = await fetch_user(user_id)
    return {"id": data["id"], "name": data["name"]}
```

## 工具文件字串

每個工具必須有附帶明確型別資訊的詳細文件字串：

```python
async def search_users(params: UserSearchInput) -> str:
    '''
    Search for users in the Example system by name, email, or team.

    This tool searches across all user profiles in the Example platform,
    supporting partial matches and various search filters. It does NOT
    create or modify users, only searches existing ones.

    Args:
        params (UserSearchInput): Validated input parameters containing:
            - query (str): Search string to match against names/emails (e.g., "john", "@example.com", "team:marketing")
            - limit (Optional[int]): Maximum results to return, between 1-100 (default: 20)
            - offset (Optional[int]): Number of results to skip for pagination (default: 0)

    Returns:
        str: JSON-formatted string containing search results with the following schema:

        Success response:
        {
            "total": int,           # Total number of matches found
            "count": int,           # Number of results in this response
            "offset": int,          # Current pagination offset
            "users": [
                {
                    "id": str,      # User ID (e.g., "U123456789")
                    "name": str,    # Full name (e.g., "John Doe")
                    "email": str,   # Email address (e.g., "john@example.com")
                    "team": str     # Team name (e.g., "Marketing") - optional
                }
            ]
        }

        Error response:
        "Error: <error message>" or "No users found matching '<query>'"

    Examples:
        - Use when: "Find all marketing team members" -> params with query="team:marketing"
        - Use when: "Search for John's account" -> params with query="john"
        - Don't use when: You need to create a user (use example_create_user instead)
        - Don't use when: You have a user ID and need full details (use example_get_user instead)

    Error Handling:
        - Input validation errors are handled by Pydantic model
        - Returns "Error: Rate limit exceeded" if too many requests (429 status)
        - Returns "Error: Invalid API authentication" if API key is invalid (401 status)
        - Returns formatted list of results or "No users found matching 'query'"
    '''
```

## 完整範例

以下是完整的 Python MCP 伺服器範例：

```python
#!/usr/bin/env python3
'''
MCP Server for Example Service.

This server provides tools to interact with Example API, including user search,
project management, and data export capabilities.
'''

from typing import Optional, List, Dict, Any
from enum import Enum
import httpx
from pydantic import BaseModel, Field, field_validator, ConfigDict
from mcp.server.fastmcp import FastMCP

# Initialize the MCP server
mcp = FastMCP("example_mcp")

# Constants
API_BASE_URL = "https://api.example.com/v1"

# Enums
class ResponseFormat(str, Enum):
    '''Output format for tool responses.'''
    MARKDOWN = "markdown"
    JSON = "json"

# Pydantic Models for Input Validation
class UserSearchInput(BaseModel):
    '''Input model for user search operations.'''
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True
    )

    query: str = Field(..., description="Search string to match against names/emails", min_length=2, max_length=200)
    limit: Optional[int] = Field(default=20, description="Maximum results to return", ge=1, le=100)
    offset: Optional[int] = Field(default=0, description="Number of results to skip for pagination", ge=0)
    response_format: ResponseFormat = Field(default=ResponseFormat.MARKDOWN, description="Output format")

    @field_validator('query')
    @classmethod
    def validate_query(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Query cannot be empty or whitespace only")
        return v.strip()

# Shared utility functions
async def _make_api_request(endpoint: str, method: str = "GET", **kwargs) -> dict:
    '''Reusable function for all API calls.'''
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method,
            f"{API_BASE_URL}/{endpoint}",
            timeout=30.0,
            **kwargs
        )
        response.raise_for_status()
        return response.json()

def _handle_api_error(e: Exception) -> str:
    '''Consistent error formatting across all tools.'''
    if isinstance(e, httpx.HTTPStatusError):
        if e.response.status_code == 404:
            return "Error: Resource not found. Please check the ID is correct."
        elif e.response.status_code == 403:
            return "Error: Permission denied. You don't have access to this resource."
        elif e.response.status_code == 429:
            return "Error: Rate limit exceeded. Please wait before making more requests."
        return f"Error: API request failed with status {e.response.status_code}"
    elif isinstance(e, httpx.TimeoutException):
        return "Error: Request timed out. Please try again."
    return f"Error: Unexpected error occurred: {type(e).__name__}"

# Tool definitions
@mcp.tool(
    name="example_search_users",
    annotations={
        "title": "Search Example Users",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": True
    }
)
async def example_search_users(params: UserSearchInput) -> str:
    '''Search for users in the Example system by name, email, or team.

    [Full docstring as shown above]
    '''
    try:
        # Make API request using validated parameters
        data = await _make_api_request(
            "users/search",
            params={
                "q": params.query,
                "limit": params.limit,
                "offset": params.offset
            }
        )

        users = data.get("users", [])
        total = data.get("total", 0)

        if not users:
            return f"No users found matching '{params.query}'"

        # Format response based on requested format
        if params.response_format == ResponseFormat.MARKDOWN:
            lines = [f"# User Search Results: '{params.query}'", ""]
            lines.append(f"Found {total} users (showing {len(users)})")
            lines.append("")

            for user in users:
                lines.append(f"## {user['name']} ({user['id']})")
                lines.append(f"- **Email**: {user['email']}")
                if user.get('team'):
                    lines.append(f"- **Team**: {user['team']}")
                lines.append("")

            return "\n".join(lines)

        else:
            # Machine-readable JSON format
            import json
            response = {
                "total": total,
                "count": len(users),
                "offset": params.offset,
                "users": users
            }
            return json.dumps(response, indent=2)

    except Exception as e:
        return _handle_api_error(e)

if __name__ == "__main__":
    mcp.run()
```

---

## 進階 FastMCP 功能

### Context 參數注入

FastMCP 可以自動將 `Context` 參數注入工具，用於進階功能，如日誌記錄、進度回報、資源讀取和使用者互動：

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("example_mcp")

@mcp.tool()
async def advanced_search(query: str, ctx: Context) -> str:
    '''Advanced tool with context access for logging and progress.'''

    # Report progress for long operations
    await ctx.report_progress(0.25, "Starting search...")

    # Log information for debugging
    await ctx.log_info("Processing query", {"query": query, "timestamp": datetime.now()})

    # Perform search
    results = await search_api(query)
    await ctx.report_progress(0.75, "Formatting results...")

    # Access server configuration
    server_name = ctx.fastmcp.name

    return format_results(results)

@mcp.tool()
async def interactive_tool(resource_id: str, ctx: Context) -> str:
    '''Tool that can request additional input from users.'''

    # Request sensitive information when needed
    api_key = await ctx.elicit(
        prompt="Please provide your API key:",
        input_type="password"
    )

    # Use the provided key
    return await api_call(resource_id, api_key)
```

**Context 功能：**
- `ctx.report_progress(progress, message)` — 回報長時間操作的進度
- `ctx.log_info(message, data)` / `ctx.log_error()` / `ctx.log_debug()` — 日誌記錄
- `ctx.elicit(prompt, input_type)` — 向使用者請求輸入
- `ctx.fastmcp.name` — 存取伺服器設定
- `ctx.read_resource(uri)` — 讀取 MCP 資源

### 資源註冊

將資料作為資源公開，以便基於模板的高效存取：

```python
@mcp.resource("file://documents/{name}")
async def get_document(name: str) -> str:
    '''Expose documents as MCP resources.

    Resources are useful for static or semi-static data that doesn't
    require complex parameters. They use URI templates for flexible access.
    '''
    document_path = f"./docs/{name}"
    with open(document_path, "r") as f:
        return f.read()

@mcp.resource("config://settings/{key}")
async def get_setting(key: str, ctx: Context) -> str:
    '''Expose configuration as resources with context.'''
    settings = await load_settings()
    return json.dumps(settings.get(key, {}))
```

**何時使用資源 vs 工具：**
- **資源**：用於使用簡單參數（URI 模板）的資料存取
- **工具**：用於需要驗證和業務邏輯的複雜操作

### 結構化輸出型別

FastMCP 支援字串以外的多種回傳型別：

```python
from typing import TypedDict
from dataclasses import dataclass
from pydantic import BaseModel

# TypedDict for structured returns
class UserData(TypedDict):
    id: str
    name: str
    email: str

@mcp.tool()
async def get_user_typed(user_id: str) -> UserData:
    '''Returns structured data - FastMCP handles serialization.'''
    return {"id": user_id, "name": "John Doe", "email": "john@example.com"}

# Pydantic models for complex validation
class DetailedUser(BaseModel):
    id: str
    name: str
    email: str
    created_at: datetime
    metadata: Dict[str, Any]

@mcp.tool()
async def get_user_detailed(user_id: str) -> DetailedUser:
    '''Returns Pydantic model - automatically generates schema.'''
    user = await fetch_user(user_id)
    return DetailedUser(**user)
```

### 生命週期管理

初始化跨請求持續存在的資源：

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def app_lifespan():
    '''Manage resources that live for the server's lifetime.'''
    # Initialize connections, load config, etc.
    db = await connect_to_database()
    config = load_configuration()

    # Make available to all tools
    yield {"db": db, "config": config}

    # Cleanup on shutdown
    await db.close()

mcp = FastMCP("example_mcp", lifespan=app_lifespan)

@mcp.tool()
async def query_data(query: str, ctx: Context) -> str:
    '''Access lifespan resources through context.'''
    db = ctx.request_context.lifespan_state["db"]
    results = await db.query(query)
    return format_results(results)
```

### 傳輸選項

FastMCP 支援兩種主要傳輸機制：

```python
# stdio transport (for local tools) - default
if __name__ == "__main__":
    mcp.run()

# Streamable HTTP transport (for remote servers)
if __name__ == "__main__":
    mcp.run(transport="streamable_http", port=8000)
```

**傳輸方式選擇：**
- **stdio**：命令列工具、本機整合、子行程執行
- **Streamable HTTP**：網路服務、遠端存取、多個客戶端

---

## 程式碼最佳實踐

### 程式碼可組合性與可重用性

您的實作**必須**優先考慮可組合性和程式碼重用：

1. **提取常用功能**：
   - 為跨多個工具使用的操作建立可重用的輔助函式
   - 為 HTTP 請求建立共用的 API 客戶端，而非重複程式碼
   - 在工具函式中集中錯誤處理邏輯
   - 將業務邏輯提取至可組合的專用函式
   - 提取共用的 markdown 或 JSON 欄位選擇與格式化功能

2. **避免重複**：
   - **絕不**在工具之間複製貼上相似的程式碼
   - 若發現自己寫了兩次相似的邏輯，請將其提取為函式
   - 分頁、篩選、欄位選擇和格式化等常用操作應共享
   - 驗證/授權邏輯應集中管理

### Python 專屬最佳實踐

1. **使用型別提示**：始終為函式參數和回傳值包含型別標註
2. **Pydantic 模型**：為所有輸入驗證定義清晰的 Pydantic 模型
3. **避免手動驗證**：讓 Pydantic 以限制條件處理輸入驗證
4. **適當 import**：分組 import（標準函式庫、第三方、本機）
5. **錯誤處理**：使用特定的例外型別（httpx.HTTPStatusError，而非通用的 Exception）
6. **非同步情境管理器**：對需要清除的資源使用 `async with`
7. **常數**：以 UPPER_CASE 在模組層級定義常數

## 品質檢查清單

在完成 Python MCP 伺服器實作之前，確保：

### 策略設計
- [ ] 工具支援完整的工作流程，而非只是 API 端點包裝
- [ ] 工具名稱反映自然的任務劃分
- [ ] 回應格式針對 agent 情境效率進行最佳化
- [ ] 在適當的地方使用人類可讀的識別碼
- [ ] 錯誤訊息引導 agent 正確使用

### 實作品質
- [ ] 聚焦實作：最重要且最有價值的工具已實作
- [ ] 所有工具具有描述性名稱和文件
- [ ] 回傳型別在相似操作之間保持一致
- [ ] 所有外部呼叫均已實作錯誤處理
- [ ] 伺服器名稱遵循格式：`{service}_mcp`
- [ ] 所有網路操作使用 async/await
- [ ] 常用功能已提取為可重用函式
- [ ] 錯誤訊息清晰、可操作且具有教育意義
- [ ] 輸出已正確驗證和格式化

### 工具設定
- [ ] 所有工具在裝飾器中實作 'name' 和 'annotations'
- [ ] 標註正確設定（readOnlyHint、destructiveHint、idempotentHint、openWorldHint）
- [ ] 所有工具使用帶有 Field() 定義的 Pydantic BaseModel 進行輸入驗證
- [ ] 所有 Pydantic Field 具有明確的型別、說明和限制條件
- [ ] 所有工具具有附帶明確輸入/輸出型別的詳細文件字串
- [ ] 文件字串包含 dict/JSON 回傳的完整綱要結構
- [ ] Pydantic 模型處理輸入驗證（不需要手動驗證）

### 進階功能（適用時）
- [ ] Context 注入用於日誌記錄、進度或引導
- [ ] 為適當的資料端點註冊資源
- [ ] 為持久連線實作生命週期管理
- [ ] 使用結構化輸出型別（TypedDict、Pydantic 模型）
- [ ] 設定適當的傳輸方式（stdio 或 streamable HTTP）

### 程式碼品質
- [ ] 檔案包含適當的 import，包括 Pydantic import
- [ ] 在適用的地方正確實作分頁
- [ ] 為可能的大型結果集提供篩選選項
- [ ] 所有非同步函式以 `async def` 正確定義
- [ ] HTTP 客戶端使用符合 async 模式，並使用適當的情境管理器
- [ ] 全程使用型別提示
- [ ] 常數以 UPPER_CASE 在模組層級定義

### 測試
- [ ] 伺服器成功執行：`python your_server.py --help`
- [ ] 所有 import 正確解析
- [ ] 範例工具呼叫如預期運作
- [ ] 錯誤情境已優雅處理