---
name: oracle-gpt
description: "Route questions to ChatGPT via the Oracle MCP tool. Use this skill whenever the user wants to ask ChatGPT/GPT something, consult GPT, or get a second opinion from GPT. Trigger on: '问GPT', '问ChatGPT', 'ask GPT', 'ask ChatGPT', '用Oracle', 'oracle', '咨询GPT', '让GPT', 'GPT帮我', '问一下GPT', '让ChatGPT看看', 'consult GPT', '问问GPT', '发给GPT'. Also trigger when the user says things like 'what does ChatGPT think', 'get GPT's opinion', 'ChatGPT怎么说', or any request that clearly intends to forward a question to ChatGPT rather than have you answer it yourself. Do NOT trigger for general questions the user wants YOU to answer — only when they explicitly want ChatGPT involved."
---

# Oracle GPT — Ask ChatGPT via MCP

This skill lets you send questions to ChatGPT through the Oracle MCP server and return the answers to the user. Oracle automates a real ChatGPT browser session on a remote server, so responses take 20-60 seconds.

## Prerequisites

The Oracle MCP must be configured in `~/.claude/.mcp.json`. If the `mcp__oracle__consult` tool is not available, guide the user through first-time setup:

1. Install Oracle CLI: `npm install -g git+https://github.com/orange4664/oracle.git`
2. Ask the user to provide their **Oracle server address** (host:port) and **authentication token**
3. Write the MCP config to `~/.claude/.mcp.json`:

```json
{
  "mcpServers": {
    "oracle": {
      "command": "oracle-mcp",
      "args": [],
      "env": {
        "ORACLE_REMOTE_HOST": "<user-provided-host:port>",
        "ORACLE_REMOTE_TOKEN": "<user-provided-token>"
      }
    }
  }
}
```

4. Tell the user to restart Claude Code for MCP to take effect

If the MCP is already configured (the `mcp__oracle__consult` tool is available), skip setup entirely.

## Conversation State

You need to track three pieces of state across the session:

- **`chatgpt_url`** — the ChatGPT conversation URL to reuse. Starts empty. Gets set after the first successful call.
- **`session_initialized`** — whether the user has already chosen a conversation mode. Starts false.
- **`use_remote`** — whether to use a remote Oracle server or the local browser. Starts unset. Gets set on first use.

These are conceptual variables you maintain in your working memory throughout the conversation — not files or environment variables.

## Workflow

### First time using Oracle in this session — choose execution mode

Before anything else, on the very first Oracle call in the session, use `AskUserQuestion` to ask the user whether to run Oracle locally or on a remote server:

**Question:** "Oracle 使用哪种运行模式？"

**Options:**

1. **远程服务器（推荐）** — "通过远程服务器执行浏览器自动化，不会打开本地浏览器。需要已配置远程服务器地址。"
2. **本地浏览器** — "在本机启动浏览器执行操作。会打开一个 Chrome 窗口，执行期间请勿关闭。"

Then:

- If **远程服务器**: Set `use_remote` to true. Read `ORACLE_REMOTE_HOST` and `ORACLE_REMOTE_TOKEN` from `~/.claude/.mcp.json` env vars. If they are not configured, ask the user to provide the server address and token, then update the MCP config accordingly. **When `use_remote` is true, always use the Oracle CLI with `--remote-host` and `--remote-token` flags instead of the `mcp__oracle__consult` MCP tool**, because the MCP tool may not correctly route to the remote server.
- If **本地浏览器**: Set `use_remote` to false. Use the `mcp__oracle__consult` MCP tool directly.

This choice persists for the entire session. The user does not need to choose again unless they start a new session.

### First time the user asks GPT something — choose conversation mode

When `session_initialized` is false, use `AskUserQuestion` to ask the user which conversation mode they want:

**Question:** "你想以什么方式和 ChatGPT 对话？"

**Options:**

1. **继续对话** — "继续一个已有的 ChatGPT 对话（需要提供对话链接）"
2. **从项目新建对话** — "在某个 GPT 项目中新建对话（需要提供项目链接）"
3. **新建对话** — "直接开始一个全新对话"

Then:

- If **继续对话**: Ask user for the conversation URL (like `https://chatgpt.com/c/xxx`). Set `chatgpt_url` to that URL. Set `session_initialized` to true.
- If **从项目新建对话**: Ask user for the project URL (like `https://chatgpt.com/g/g-p-xxx/project`). Set `chatgpt_url` to that project URL. Set `session_initialized` to true. After the first call succeeds, update `chatgpt_url` to the returned `tabUrl` so subsequent messages stay in the newly created conversation.
- If **新建对话**: Leave `chatgpt_url` empty. Set `session_initialized` to true. After the first call succeeds, update `chatgpt_url` to the returned `tabUrl`.

### Sending a question to ChatGPT

Call the MCP tool:

```
mcp__oracle__consult(
  prompt: "<the user's question>",
  files: [<file paths if the user wants to attach files>]
)
```

If `chatgpt_url` is set, you need to tell the user's question in a way that Oracle will use that conversation. Currently Oracle MCP's `consult` tool doesn't directly accept a chatgpt-url parameter — so if the conversation URL needs to be passed, include it in the prompt context or use the Bash tool to call Oracle CLI directly:

```bash
oracle --chatgpt-url "<chatgpt_url>" -p "<user's question>"
```

If Oracle is configured as a remote server, add `--remote-host` and `--remote-token` flags using the values from the user's MCP config (read from `~/.claude/.mcp.json` env vars `ORACLE_REMOTE_HOST` and `ORACLE_REMOTE_TOKEN`).

This is the reliable method for continuing conversations.

### File Uploads

When the user wants to send files to ChatGPT, use the CLI `--file` flag. Read `ORACLE_REMOTE_HOST` and `ORACLE_REMOTE_TOKEN` from the user's `~/.claude/.mcp.json`.

```bash
# Single file
oracle --remote-host <host:port> --remote-token <token> \
       -p "帮我review这段代码" \
       --file src/main.ts

# Multiple files (glob pattern)
oracle --remote-host <host:port> --remote-token <token> \
       -p "分析这个项目" \
       --file "src/**/*.ts"

# Binary files (images, PDFs, etc.) — must use --browser-attachments always
oracle --remote-host <host:port> --remote-token <token> \
       --browser-attachments always \
       -p "分析这个文件" \
       --file report.pdf
```

Combine with `--chatgpt-url` if continuing a conversation. The MCP tool's `files` parameter also works for text files: `mcp__oracle__consult(prompt: "...", files: ["path/to/file"])`.

### After receiving a response

1. Extract the answer text and the conversation URL (`tabUrl` or from the `Conversation URL:` line in CLI output).
2. If `chatgpt_url` was empty or was a project URL, update it to the returned conversation URL for future calls.
3. Show the user:
   - The ChatGPT answer
   - The conversation URL (so they can open it in browser if needed)
4. Tell the user the response time if available.

### Subsequent questions

When `session_initialized` is true and the user asks GPT another question, skip the mode selection — just send the question using the stored `chatgpt_url`. This maintains conversation continuity in ChatGPT.

### Starting a new conversation

If the user says anything like "新对话", "new chat", "开一个新会话", "换个对话", "重新开始", "start fresh":

1. Reset `chatgpt_url` to empty
2. Reset `session_initialized` to false
3. On the next question, the mode selection will appear again

## Error Handling

- **"busy"** — Oracle is processing another request. Tell the user: "ChatGPT 正在处理其他请求，30 秒后重试。" Wait 30 seconds, then retry once.
- **"Cloudflare challenge"** — The ChatGPT login session has expired. Tell the user: "ChatGPT 的登录已过期，需要重新注入 cookies。"
- **"ECONNREFUSED"** — Oracle serve may have restarted. Tell the user to wait a moment and retry. The service has auto-restart configured.
- **Timeout / no response** — Oracle can take up to 60 seconds. Always warn the user before sending: "正在向 ChatGPT 发送问题，大约需要 20-60 秒..."

## Example Interactions

**User:** 帮我问一下 GPT，Python 的 GIL 是什么
**You:** [First time → show mode selection → user picks 新建对话]
**You:** 正在向 ChatGPT 发送问题，大约需要 20-60 秒...
**You:** [Call Oracle, get response]
**You:** ChatGPT 的回答：[answer]
对话链接：https://chatgpt.com/c/xxx

**User:** 继续问它，那多线程还有意义吗
**You:** 正在继续对话... [auto-uses stored chatgpt_url]
**You:** ChatGPT 的回答：[answer]

**User:** 开个新对话
**You:** [Reset state, next question will show mode selection again]
