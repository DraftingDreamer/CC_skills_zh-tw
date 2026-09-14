---
source_file: node_mcp_server.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: c3ba35a4f599dd53be9c6555ae72c19a7bf412cd5426576c2c08d42755482c66
translated_at: 2026-08-22
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`node_mcp_server.md`](node_mcp_server.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# Node/TypeScript MCP 伺服器實作指南

## 概述

本文件提供使用 MCP TypeScript SDK 實作 MCP 伺服器的 Node/TypeScript 專屬最佳實踐和範例。涵蓋專案結構、伺服器設定、工具註冊模式、使用 Zod 進行輸入驗證、錯誤處理，以及完整的可運行範例。

---

## 快速參考

### 關鍵 import
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import express from "express";
import { z } from "zod";
```

### 伺服器初始化
```typescript
const server = new McpServer({
  name: "service-mcp-server",
  version: "1.0.0"
});
```

### 工具註冊模式
```typescript
server.registerTool(
  "tool_name",
  {
    title: "Tool Display Name",
    description: "What the tool does",
    inputSchema: { param: z.string() },
    outputSchema: { result: z.string() }
  },
  async ({ param }) => {
    const output = { result: `Processed: ${param}` };
    return {
      content: [{ type: "text", text: JSON.stringify(output) }],
      structuredContent: output // Modern pattern for structured data
    };
  }
);
```

---

## MCP TypeScript SDK

官方 MCP TypeScript SDK 提供：
- 用於伺服器初始化的 `McpServer` 類別
- 用於工具註冊的 `registerTool` 方法
- Zod 綱要整合，用於執行期輸入驗證
- 型別安全的工具處理器實作

**重要——只使用現代 API：**
- **應使用**：`server.registerTool()`、`server.registerResource()`、`server.registerPrompt()`
- **不應使用**：舊的已棄用 API，例如 `server.tool()`、`server.setRequestHandler(ListToolsRequestSchema, ...)` 或手動處理器註冊
- `register*` 方法提供更好的型別安全性、自動綱要處理，且是推薦的做法

完整細節請參閱參考資料中的 MCP SDK 文件。

## 伺服器命名慣例

Node/TypeScript MCP 伺服器必須遵循此命名模式：
- **格式**：`{service}-mcp-server`（小寫加連字號）
- **範例**：`github-mcp-server`、`jira-mcp-server`、`stripe-mcp-server`

名稱應：
- 通用（不與特定功能綁定）
- 描述所整合的服務/API
- 易於從任務描述推斷
- 不含版本號或日期

## 專案結構

為 Node/TypeScript MCP 伺服器建立以下結構：

```
{service}-mcp-server/
├── package.json
├── tsconfig.json
├── README.md
├── src/
│   ├── index.ts          # Main entry point with McpServer initialization
│   ├── types.ts          # TypeScript type definitions and interfaces
│   ├── tools/            # Tool implementations (one file per domain)
│   ├── services/         # API clients and shared utilities
│   ├── schemas/          # Zod validation schemas
│   └── constants.ts      # Shared constants (API_URL, CHARACTER_LIMIT, etc.)
└── dist/                 # Built JavaScript files (entry point: dist/index.js)
```

## 工具實作

### 工具命名

工具名稱使用 snake_case（例如 "search_users"、"create_project"、"get_channel_info"），並使用清晰的、以動作為導向的名稱。

**避免命名衝突**：包含服務情境以防止重疊：
- 使用 "slack_send_message" 而非僅 "send_message"
- 使用 "github_create_issue" 而非僅 "create_issue"
- 使用 "asana_list_tasks" 而非僅 "list_tasks"

### 工具結構

工具使用 `registerTool` 方法註冊，有以下需求：
- 使用 Zod 綱要進行執行期輸入驗證和型別安全
- `description` 欄位必須明確提供——JSDoc 注解**不會**自動提取
- 明確提供 `title`、`description`、`inputSchema` 和 `annotations`
- `inputSchema` 必須是 Zod 綱要物件（不是 JSON 綱要）
- 明確為所有參數和回傳值設定型別

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({
  name: "example-mcp",
  version: "1.0.0"
});

// Zod schema for input validation
const UserSearchInputSchema = z.object({
  query: z.string()
    .min(2, "Query must be at least 2 characters")
    .max(200, "Query must not exceed 200 characters")
    .describe("Search string to match against names/emails"),
  limit: z.number()
    .int()
    .min(1)
    .max(100)
    .default(20)
    .describe("Maximum results to return"),
  offset: z.number()
    .int()
    .min(0)
    .default(0)
    .describe("Number of results to skip for pagination"),
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format: 'markdown' for human-readable or 'json' for machine-readable")
}).strict();

// Type definition from Zod schema
type UserSearchInput = z.infer<typeof UserSearchInputSchema>;

server.registerTool(
  "example_search_users",
  {
    title: "Search Example Users",
    description: `Search for users in the Example system by name, email, or team.

This tool searches across all user profiles in the Example platform, supporting partial matches and various search filters. It does NOT create or modify users, only searches existing ones.

Args:
  - query (string): Search string to match against names/emails
  - limit (number): Maximum results to return, between 1-100 (default: 20)
  - offset (number): Number of results to skip for pagination (default: 0)
  - response_format ('markdown' | 'json'): Output format (default: 'markdown')

Returns:
  For JSON format: Structured data with schema:
  {
    "total": number,           // Total number of matches found
    "count": number,           // Number of results in this response
    "offset": number,          // Current pagination offset
    "users": [
      {
        "id": string,          // User ID (e.g., "U123456789")
        "name": string,        // Full name (e.g., "John Doe")
        "email": string,       // Email address
        "team": string,        // Team name (optional)
        "active": boolean      // Whether user is active
      }
    ],
    "has_more": boolean,       // Whether more results are available
    "next_offset": number      // Offset for next page (if has_more is true)
  }

Examples:
  - Use when: "Find all marketing team members" -> params with query="team:marketing"
  - Use when: "Search for John's account" -> params with query="john"
  - Don't use when: You need to create a user (use example_create_user instead)

Error Handling:
  - Returns "Error: Rate limit exceeded" if too many requests (429 status)
  - Returns "No users found matching '<query>'" if search returns empty`,
    inputSchema: UserSearchInputSchema,
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true
    }
  },
  async (params: UserSearchInput) => {
    try {
      // Input validation is handled by Zod schema
      // Make API request using validated parameters
      const data = await makeApiRequest<any>(
        "users/search",
        "GET",
        undefined,
        {
          q: params.query,
          limit: params.limit,
          offset: params.offset
        }
      );

      const users = data.users || [];
      const total = data.total || 0;

      if (!users.length) {
        return {
          content: [{
            type: "text",
            text: `No users found matching '${params.query}'`
          }]
        };
      }

      // Prepare structured output
      const output = {
        total,
        count: users.length,
        offset: params.offset,
        users: users.map((user: any) => ({
          id: user.id,
          name: user.name,
          email: user.email,
          ...(user.team ? { team: user.team } : {}),
          active: user.active ?? true
        })),
        has_more: total > params.offset + users.length,
        ...(total > params.offset + users.length ? {
          next_offset: params.offset + users.length
        } : {})
      };

      // Format text representation based on requested format
      let textContent: string;
      if (params.response_format === ResponseFormat.MARKDOWN) {
        const lines = [`# User Search Results: '${params.query}'`, "",
          `Found ${total} users (showing ${users.length})`, ""];
        for (const user of users) {
          lines.push(`## ${user.name} (${user.id})`);
          lines.push(`- **Email**: ${user.email}`);
          if (user.team) lines.push(`- **Team**: ${user.team}`);
          lines.push("");
        }
        textContent = lines.join("\n");
      } else {
        textContent = JSON.stringify(output, null, 2);
      }

      return {
        content: [{ type: "text", text: textContent }],
        structuredContent: output // Modern pattern for structured data
      };
    } catch (error) {
      return {
        content: [{
          type: "text",
          text: handleApiError(error)
        }]
      };
    }
  }
);
```

## 輸入驗證用 Zod 綱要

Zod 提供執行期型別驗證：

```typescript
import { z } from "zod";

// Basic schema with validation
const CreateUserSchema = z.object({
  name: z.string()
    .min(1, "Name is required")
    .max(100, "Name must not exceed 100 characters"),
  email: z.string()
    .email("Invalid email format"),
  age: z.number()
    .int("Age must be a whole number")
    .min(0, "Age cannot be negative")
    .max(150, "Age cannot be greater than 150")
}).strict();  // Use .strict() to forbid extra fields

// Enums
enum ResponseFormat {
  MARKDOWN = "markdown",
  JSON = "json"
}

const SearchSchema = z.object({
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format")
});

// Optional fields with defaults
const PaginationSchema = z.object({
  limit: z.number()
    .int()
    .min(1)
    .max(100)
    .default(20)
    .describe("Maximum results to return"),
  offset: z.number()
    .int()
    .min(0)
    .default(0)
    .describe("Number of results to skip")
});
```

## 回應格式選項

支援多種輸出格式以提供彈性：

```typescript
enum ResponseFormat {
  MARKDOWN = "markdown",
  JSON = "json"
}

const inputSchema = z.object({
  query: z.string(),
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format: 'markdown' for human-readable or 'json' for machine-readable")
});
```

**Markdown 格式**：
- 使用標題、清單和格式化以提升清晰度
- 將時間戳記轉換為人類可讀格式
- 以括號顯示 ID 搭配顯示名稱
- 省略冗長的中繼資料
- 以邏輯方式分組相關資訊

**JSON 格式**：
- 回傳完整的、適合程式化處理的結構化資料
- 包含所有可用欄位和中繼資料
- 使用一致的欄位名稱和型別

## 分頁實作

對於列出資源的工具：

```typescript
const ListSchema = z.object({
  limit: z.number().int().min(1).max(100).default(20),
  offset: z.number().int().min(0).default(0)
});

async function listItems(params: z.infer<typeof ListSchema>) {
  const data = await apiRequest(params.limit, params.offset);

  const response = {
    total: data.total,
    count: data.items.length,
    offset: params.offset,
    items: data.items,
    has_more: data.total > params.offset + data.items.length,
    next_offset: data.total > params.offset + data.items.length
      ? params.offset + data.items.length
      : undefined
  };

  return JSON.stringify(response, null, 2);
}
```

## 字元限制與截斷

加入 CHARACTER_LIMIT 常數以防止回應過於龐大：

```typescript
// At module level in constants.ts
export const CHARACTER_LIMIT = 25000;  // Maximum response size in characters

async function searchTool(params: SearchInput) {
  let result = generateResponse(data);

  // Check character limit and truncate if needed
  if (result.length > CHARACTER_LIMIT) {
    const truncatedData = data.slice(0, Math.max(1, data.length / 2));
    response.data = truncatedData;
    response.truncated = true;
    response.truncation_message =
      `Response truncated from ${data.length} to ${truncatedData.length} items. ` +
      `Use 'offset' parameter or add filters to see more results.`;
    result = JSON.stringify(response, null, 2);
  }

  return result;
}
```

## 錯誤處理

提供清晰、可操作的錯誤訊息：

```typescript
import axios, { AxiosError } from "axios";

function handleApiError(error: unknown): string {
  if (error instanceof AxiosError) {
    if (error.response) {
      switch (error.response.status) {
        case 404:
          return "Error: Resource not found. Please check the ID is correct.";
        case 403:
          return "Error: Permission denied. You don't have access to this resource.";
        case 429:
          return "Error: Rate limit exceeded. Please wait before making more requests.";
        default:
          return `Error: API request failed with status ${error.response.status}`;
      }
    } else if (error.code === "ECONNABORTED") {
      return "Error: Request timed out. Please try again.";
    }
  }
  return `Error: Unexpected error occurred: ${error instanceof Error ? error.message : String(error)}`;
}
```

## 共用工具函式

將常用功能提取為可重用的函式：

```typescript
// Shared API request function
async function makeApiRequest<T>(
  endpoint: string,
  method: "GET" | "POST" | "PUT" | "DELETE" = "GET",
  data?: any,
  params?: any
): Promise<T> {
  try {
    const response = await axios({
      method,
      url: `${API_BASE_URL}/${endpoint}`,
      data,
      params,
      timeout: 30000,
      headers: {
        "Content-Type": "application/json",
        "Accept": "application/json"
      }
    });
    return response.data;
  } catch (error) {
    throw error;
  }
}
```

## Async/Await 最佳實踐

對網路請求和 I/O 操作始終使用 async/await：

```typescript
// Good: Async network request
async function fetchData(resourceId: string): Promise<ResourceData> {
  const response = await axios.get(`${API_URL}/resource/${resourceId}`);
  return response.data;
}

// Bad: Promise chains
function fetchData(resourceId: string): Promise<ResourceData> {
  return axios.get(`${API_URL}/resource/${resourceId}`)
    .then(response => response.data);  // Harder to read and maintain
}
```

## TypeScript 最佳實踐

1. **使用 Strict TypeScript**：在 tsconfig.json 中啟用 strict 模式
2. **定義介面**：為所有資料結構建立清晰的介面定義
3. **避免 `any`**：使用適當的型別或 `unknown` 而非 `any`
4. **Zod 用於執行期驗證**：使用 Zod 綱要驗證外部資料
5. **型別保護**：為複雜的型別檢查建立型別保護函式
6. **錯誤處理**：始終使用 try-catch 並進行適當的錯誤型別檢查
7. **空值安全**：使用可選鏈（`?.`）和空值合併（`??`）

```typescript
// Good: Type-safe with Zod and interfaces
interface UserResponse {
  id: string;
  name: string;
  email: string;
  team?: string;
  active: boolean;
}

const UserSchema = z.object({
  id: z.string(),
  name: z.string(),
  email: z.string().email(),
  team: z.string().optional(),
  active: z.boolean()
});

type User = z.infer<typeof UserSchema>;

async function getUser(id: string): Promise<User> {
  const data = await apiCall(`/users/${id}`);
  return UserSchema.parse(data);  // Runtime validation
}

// Bad: Using any
async function getUser(id: string): Promise<any> {
  return await apiCall(`/users/${id}`);  // No type safety
}
```

## 套件設定

### package.json

```json
{
  "name": "{service}-mcp-server",
  "version": "1.0.0",
  "description": "MCP server for {Service} API integration",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "clean": "rm -rf dist"
  },
  "engines": {
    "node": ">=18"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.6.1",
    "axios": "^1.7.9",
    "zod": "^3.23.8"
  },
  "devDependencies": {
    "@types/node": "^22.10.0",
    "tsx": "^4.19.2",
    "typescript": "^5.7.2"
  }
}
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## 完整範例

```typescript
#!/usr/bin/env node
/**
 * MCP Server for Example Service.
 *
 * This server provides tools to interact with Example API, including user search,
 * project management, and data export capabilities.
 */

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import axios, { AxiosError } from "axios";

// Constants
const API_BASE_URL = "https://api.example.com/v1";
const CHARACTER_LIMIT = 25000;

// Enums
enum ResponseFormat {
  MARKDOWN = "markdown",
  JSON = "json"
}

// Zod schemas
const UserSearchInputSchema = z.object({
  query: z.string()
    .min(2, "Query must be at least 2 characters")
    .max(200, "Query must not exceed 200 characters")
    .describe("Search string to match against names/emails"),
  limit: z.number()
    .int()
    .min(1)
    .max(100)
    .default(20)
    .describe("Maximum results to return"),
  offset: z.number()
    .int()
    .min(0)
    .default(0)
    .describe("Number of results to skip for pagination"),
  response_format: z.nativeEnum(ResponseFormat)
    .default(ResponseFormat.MARKDOWN)
    .describe("Output format: 'markdown' for human-readable or 'json' for machine-readable")
}).strict();

type UserSearchInput = z.infer<typeof UserSearchInputSchema>;

// Shared utility functions
async function makeApiRequest<T>(
  endpoint: string,
  method: "GET" | "POST" | "PUT" | "DELETE" = "GET",
  data?: any,
  params?: any
): Promise<T> {
  try {
    const response = await axios({
      method,
      url: `${API_BASE_URL}/${endpoint}`,
      data,
      params,
      timeout: 30000,
      headers: {
        "Content-Type": "application/json",
        "Accept": "application/json"
      }
    });
    return response.data;
  } catch (error) {
    throw error;
  }
}

function handleApiError(error: unknown): string {
  if (error instanceof AxiosError) {
    if (error.response) {
      switch (error.response.status) {
        case 404:
          return "Error: Resource not found. Please check the ID is correct.";
        case 403:
          return "Error: Permission denied. You don't have access to this resource.";
        case 429:
          return "Error: Rate limit exceeded. Please wait before making more requests.";
        default:
          return `Error: API request failed with status ${error.response.status}`;
      }
    } else if (error.code === "ECONNABORTED") {
      return "Error: Request timed out. Please try again.";
    }
  }
  return `Error: Unexpected error occurred: ${error instanceof Error ? error.message : String(error)}`;
}

// Create MCP server instance
const server = new McpServer({
  name: "example-mcp",
  version: "1.0.0"
});

// Register tools
server.registerTool(
  "example_search_users",
  {
    title: "Search Example Users",
    description: `[Full description as shown above]`,
    inputSchema: UserSearchInputSchema,
    annotations: {
      readOnlyHint: true,
      destructiveHint: false,
      idempotentHint: true,
      openWorldHint: true
    }
  },
  async (params: UserSearchInput) => {
    // Implementation as shown above
  }
);

// Main function
// For stdio (local):
async function runStdio() {
  if (!process.env.EXAMPLE_API_KEY) {
    console.error("ERROR: EXAMPLE_API_KEY environment variable is required");
    process.exit(1);
  }

  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("MCP server running via stdio");
}

// For streamable HTTP (remote):
async function runHTTP() {
  if (!process.env.EXAMPLE_API_KEY) {
    console.error("ERROR: EXAMPLE_API_KEY environment variable is required");
    process.exit(1);
  }

  const app = express();
  app.use(express.json());

  app.post('/mcp', async (req, res) => {
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined,
      enableJsonResponse: true
    });
    res.on('close', () => transport.close());
    await server.connect(transport);
    await transport.handleRequest(req, res, req.body);
  });

  const port = parseInt(process.env.PORT || '3000');
  app.listen(port, () => {
    console.error(`MCP server running on http://localhost:${port}/mcp`);
  });
}

// Choose transport based on environment
const transport = process.env.TRANSPORT || 'stdio';
if (transport === 'http') {
  runHTTP().catch(error => {
    console.error("Server error:", error);
    process.exit(1);
  });
} else {
  runStdio().catch(error => {
    console.error("Server error:", error);
    process.exit(1);
  });
}
```

---

## 進階 MCP 功能

### 資源註冊

將資料作為資源公開，以便基於 URI 的高效存取：

```typescript
import { ResourceTemplate } from "@modelcontextprotocol/sdk/types.js";

// Register a resource with URI template
server.registerResource(
  {
    uri: "file://documents/{name}",
    name: "Document Resource",
    description: "Access documents by name",
    mimeType: "text/plain"
  },
  async (uri: string) => {
    // Extract parameter from URI
    const match = uri.match(/^file:\/\/documents\/(.+)$/);
    if (!match) {
      throw new Error("Invalid URI format");
    }

    const documentName = match[1];
    const content = await loadDocument(documentName);

    return {
      contents: [{
        uri,
        mimeType: "text/plain",
        text: content
      }]
    };
  }
);

// List available resources dynamically
server.registerResourceList(async () => {
  const documents = await getAvailableDocuments();
  return {
    resources: documents.map(doc => ({
      uri: `file://documents/${doc.name}`,
      name: doc.name,
      mimeType: "text/plain",
      description: doc.description
    }))
  };
});
```

**何時使用資源 vs 工具：**
- **資源**：用於使用簡單 URI 參數的資料存取
- **工具**：用於需要驗證和業務邏輯的複雜操作
- **資源**：當資料相對靜態或基於模板時
- **工具**：當操作有副作用或複雜工作流程時

### 傳輸選項

TypeScript SDK 支援兩種主要傳輸機制：

#### Streamable HTTP（推薦用於遠端伺服器）

```typescript
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const app = express();
app.use(express.json());

app.post('/mcp', async (req, res) => {
  // Create new transport for each request (stateless, prevents request ID collisions)
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
    enableJsonResponse: true
  });

  res.on('close', () => transport.close());

  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);
});

app.listen(3000);
```

#### stdio（用於本機整合）

```typescript
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const transport = new StdioServerTransport();
await server.connect(transport);
```

**傳輸方式選擇：**
- **Streamable HTTP**：網路服務、遠端存取、多個客戶端
- **stdio**：命令列工具、本機開發、子行程整合

### 通知支援

在伺服器狀態改變時通知客戶端：

```typescript
// Notify when tools list changes
server.notification({
  method: "notifications/tools/list_changed"
});

// Notify when resources change
server.notification({
  method: "notifications/resources/list_changed"
});
```

謹慎使用通知——只在伺服器功能真正改變時才使用。

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

## 建置與執行

在執行前始終先建置 TypeScript 程式碼：

```bash
# Build the project
npm run build

# Run the server
npm start

# Development with auto-reload
npm run dev
```

在認為實作完成之前，始終確保 `npm run build` 成功完成。

## 品質檢查清單

在完成 Node/TypeScript MCP 伺服器實作之前，確保：

### 策略設計
- [ ] 工具支援完整的工作流程，而非只是 API 端點包裝
- [ ] 工具名稱反映自然的任務劃分
- [ ] 回應格式針對 agent 情境效率進行最佳化
- [ ] 在適當的地方使用人類可讀的識別碼
- [ ] 錯誤訊息引導 agent 正確使用

### 實作品質
- [ ] 聚焦實作：最重要且最有價值的工具已實作
- [ ] 所有工具使用 `registerTool` 以完整設定進行註冊
- [ ] 所有工具包含 `title`、`description`、`inputSchema` 和 `annotations`
- [ ] 標註正確設定（readOnlyHint、destructiveHint、idempotentHint、openWorldHint）
- [ ] 所有工具使用 Zod 綱要進行執行期輸入驗證，並強制執行 `.strict()`
- [ ] 所有 Zod 綱要具有適當的限制條件和描述性錯誤訊息
- [ ] 所有工具具有詳細的說明，附帶明確的輸入/輸出型別
- [ ] 說明包含回傳值範例和完整的綱要文件
- [ ] 錯誤訊息清晰、可操作且具有教育意義

### TypeScript 品質
- [ ] 為所有資料結構定義 TypeScript 介面
- [ ] 在 tsconfig.json 中啟用 Strict TypeScript
- [ ] 不使用 `any` 型別——使用 `unknown` 或適當型別
- [ ] 所有非同步函式具有明確的 Promise<T> 回傳型別
- [ ] 錯誤處理使用適當的型別保護（例如 `axios.isAxiosError`、`z.ZodError`）

### 進階功能（適用時）
- [ ] 為適當的資料端點註冊資源
- [ ] 設定適當的傳輸方式（stdio 或 streamable HTTP）
- [ ] 為動態伺服器功能實作通知
- [ ] 與 SDK 介面保持型別安全

### 專案設定
- [ ] Package.json 包含所有必要的相依套件
- [ ] 建置指令碼在 dist/ 目錄中產生可運行的 JavaScript
- [ ] 主要進入點正確設定為 dist/index.js
- [ ] 伺服器名稱遵循格式：`{service}-mcp-server`
- [ ] tsconfig.json 以 strict 模式正確設定

### 程式碼品質
- [ ] 在適用的地方正確實作分頁
- [ ] 大型回應檢查 CHARACTER_LIMIT 常數並以清晰的訊息截斷
- [ ] 為可能的大型結果集提供篩選選項
- [ ] 所有網路操作優雅地處理逾時和連線錯誤
- [ ] 常用功能已提取為可重用函式
- [ ] 回傳型別在相似操作之間保持一致

### 測試與建置
- [ ] `npm run build` 成功完成，無錯誤
- [ ] dist/index.js 已建立且可執行
- [ ] 伺服器可執行：`node dist/index.js --help`
- [ ] 所有 import 正確解析
- [ ] 範例工具呼叫如預期運作