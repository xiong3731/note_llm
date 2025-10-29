

### install 

```bash
$ node --version

$ npm install -g @anthropic-ai/claude-code

$ claude --version

$ claude

```

国内当然不能用,尝试用88code的代理

```go
export ANTHROPIC_BASE_URL="https://www.88code.org/api"
export ANTHROPIC_AUTH_TOKEN="88_b00d0c860***947aecddff989ef41b"
```

### claude code router

```bash
$ npm install -g @musistudio/claude-code-router

$ vim ~/.claude-code-router/config.json

```

```go
# ~/.claude-code-router/config.json
{
  "LOG": true,
  "CLAUDE_PATH": "",
  "HOST": "127.0.0.1",
  "PORT": 3456,
  "APIKEY": "",
  "API_TIMEOUT_MS": "600000",
  "PROXY_URL": "",
  "transformers": [],
  "Providers": [
    {
      "name": "deepseek",
      "api_base_url": "https://api.deepseek.com/chat/completions",
      "api_key": "sk-0750d****99cec",
      "models": [
        "deepseek-chat",
        "deepseek-reasoner"
      ],
      "transformer": {
        "use": [
          "deepseek"
        ],
        "deepseek-chat": {
          "use": [
            "tooluse"
          ]
        }
      }
    }
  ],
  "Router": {
    "default": "deepseek,deepseek-chat",
    "background": "deepseek,deepseek-chat",
    "think": "deepseek,deepseek-reasoner",
    "longContext": "deepseek,deepseek-chat",
    "longContextThreshold": 60000,
    "webSearch": "deepseek,deepseek-chat"
  }
}
```

```bash
$ ccr code

# 开启web ui
$ ccr ui
```

