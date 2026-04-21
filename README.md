# Oracle CC Skills

Claude Code skills for [Oracle](https://github.com/orange4664/oracle) — route questions to ChatGPT via Oracle MCP.

## Installation

```bash
npm install -g git+https://github.com/orange4664/oracle-cc-skills.git
```

Or add to your Claude Code skills config:

```bash
claude skills add git+https://github.com/orange4664/oracle-cc-skills.git
```

## Skills

### oracle-gpt

Automatically routes questions to ChatGPT when the user says things like:

- "问GPT"、"问ChatGPT"、"ask GPT"
- "用Oracle"、"咨询GPT"、"让GPT帮我看看"
- "ChatGPT怎么说"、"get GPT's opinion"

Features:

- First-time mode selection: continue existing conversation, start from project, or new chat
- Automatic conversation continuity via stored ChatGPT URL
- File attachment support
- Error handling (busy, Cloudflare challenge, connection refused)

## Prerequisites

- [Oracle CLI](https://github.com/orange4664/oracle) installed (`npm install -g git+https://github.com/orange4664/oracle.git`)
- Oracle MCP configured in `~/.claude/.mcp.json`:

```json
{
  "mcpServers": {
    "oracle": {
      "command": "oracle-mcp",
      "args": [],
      "env": {
        "ORACLE_REMOTE_HOST": "your-server:3080",
        "ORACLE_REMOTE_TOKEN": "your-token"
      }
    }
  }
}
```

## License

MIT
