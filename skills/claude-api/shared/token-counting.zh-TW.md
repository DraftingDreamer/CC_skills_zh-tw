---
source_file: token-counting.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: b8be5d38127a79b35c0d79d3e4f1309abcdb6b15ea6917c991c14a7bb2a7d9dc
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `token-counting.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Token 計數

使用 `count_tokens` 端點（`POST /v1/messages/count_tokens`）可取得針對 Claude 模型的精確 token 數量。Token 計數是**模型特定**的——請傳入您實際用於推理的相同模型 ID。

**請勿使用 `tiktoken`。** 它是 OpenAI 的分詞器，在一般文字上對 Claude 的 token 計數低估約 15–20%，在程式碼或非英文輸入上低估更多。來自 `tiktoken`、`gpt-tokenizer` 或類似工具的任何估計值對 Claude 都是錯誤的。

## 計算檔案或字串的 token

```python
from anthropic import Anthropic

client = Anthropic()
resp = client.messages.count_tokens(
    model="claude-opus-5",
    messages=[{"role": "user", "content": open("CLAUDE.md").read()}],
)
print(resp.input_tokens)
```

TypeScript：`await client.messages.countTokens({model, messages})` →
`.input_tokens`。其他 SDK 請參見 `{lang}/claude-api/README.md`。

## CLI

```sh
ant messages count-tokens --model claude-opus-5 \
  --message '{role: user, content: "@./CLAUDE.md"}' \
  --transform input_tokens -r
```

## 比對同一檔案的兩個版本

此端點是無狀態的——分別計算每個版本，然後相減：

```python
from anthropic import Anthropic
import subprocess

client = Anthropic()
def count(text: str) -> int:
    return client.messages.count_tokens(
        model="claude-opus-5",
        messages=[{"role": "user", "content": text}],
    ).input_tokens

before = subprocess.check_output(["git", "show", "HEAD:CLAUDE.md"], text=True)
after = open("CLAUDE.md").read()
print(count(after) - count(before))
```

完整文件：請參見 `shared/live-sources.md` 中的 Token Counting 條目。
