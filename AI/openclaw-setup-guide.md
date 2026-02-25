# OpenClaw Setup Guide

A step-by-step guide to installing OpenClaw, configuring model providers, and setting up the Chrome extension — based on real setup experience.

---

## 1. Install OpenClaw

OpenClaw is distributed as a macOS app. Download and install it from the official site:

- Website: https://openclaw.ai
- Docs: https://docs.openclaw.ai

After installing, run the onboarding wizard to get started:

```bash
openclaw onboard
```

This sets up the gateway, creates the config file at `~/.openclaw/openclaw.json`, and walks you through initial model provider authentication.

---

## 2. Configure a Custom Model Provider

OpenClaw supports any OpenAI-compatible API endpoint as a custom provider.

### Config file location

```
~/.openclaw/openclaw.json
```

### Add a custom provider

Edit `~/.openclaw/openclaw.json` and add your provider under `models.providers`:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "custom": {
        "baseUrl": "https://your-company-llm-endpoint/v1",
        "apiKey": "your-api-key-here",
        "api": "openai-completions",
        "models": [
          {
            "id": "claude-sonnet-4-6",
            "name": "claude-sonnet-4-6",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          },
          {
            "id": "claude-opus-4-6",
            "name": "claude-opus-4-6",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

> **Important:** Make sure the `baseUrl` ends with `/v1`. For example:
> `http://your-endpoint.com/v1` ✅
> `http://your-endpoint.com` ❌

### Set the default model and allowlist

Also in `~/.openclaw/openclaw.json`, update the `agents.defaults` section:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "custom/claude-sonnet-4-6"
      },
      "models": {
        "custom/claude-sonnet-4-6": { "alias": "Claude Sonnet" },
        "custom/claude-opus-4-6": { "alias": "Claude Opus" }
      }
    }
  }
}
```

> **Note:** The `models` object under `agents.defaults` acts as an **allowlist**. Only models listed here can be used. If you omit this, all configured models are allowed.

### Do I need to restart after editing?

Most changes (model providers, default model) **hot-reload automatically** — just save the file. No restart needed.

Only `gateway` section changes (port, auth token, etc.) require a restart:

```bash
openclaw gateway restart
```

### Model reference format

Models are always referenced as `provider/model-id`. Examples:

- `custom/claude-sonnet-4-6` — your custom provider
- `zai/glm-5` — Z.AI GLM models
- `openai/gpt-4o` — OpenAI

### Switch model in the current session

Use the `/model` subcommand in the OpenClaw chat UI to switch models without restarting.

---

## 3. Install the Chrome Extension (Browser Relay)

The Chrome extension lets OpenClaw control your existing Chrome tabs — useful for accessing logged-in sites like Redbook (小红书), Douyin, etc.

### Step 1: Install the extension files

```bash
openclaw browser extension install
```

### Step 2: Find the extension directory

```bash
openclaw browser extension path
```

This prints the path to the installed extension. It will be inside a hidden folder (e.g. `~/.openclaw/browser/chrome-extension`).

### Step 3: Load into Chrome

1. Open Chrome and go to `chrome://extensions`
2. Enable **Developer mode** (toggle in the top-right)
3. Click **"Load unpacked"**
4. Navigate to the extension directory from Step 2

> **Tip:** If the folder is hidden and you can't see it in the file picker, press `Cmd + Shift + .` to toggle hidden files. If that shortcut is taken, run this in Terminal to copy the extension to a visible location:
> ```bash
> cp -r $(openclaw browser extension path) ~/Desktop/openclaw-extension
> ```
> Then load from `~/Desktop/openclaw-extension` in Chrome.

5. **Pin the extension** to your Chrome toolbar for easy access.

### Step 4: Set the gateway token (one-time setup)

1. Click the extension icon in the toolbar → **Options**
2. Set **Port**: `18792` (default; equals gateway port + 3)
3. Set **Gateway token**: copy the value of `gateway.auth.token` from `~/.openclaw/openclaw.json`

> **Common mistake:** Make sure you copy the token exactly with no leading/trailing spaces. A single extra space will cause "Gateway token rejected".

4. Click **Save** — it will show "Relay reachable and authenticated" if everything is correct.

### Step 5: Attach a tab

1. Open the website you want OpenClaw to access (e.g. Redbook)
2. Click the **OpenClaw Browser Relay** toolbar icon on that tab
3. The badge will show **ON** when attached
4. Tell OpenClaw to interact with the page — it now has full access

> **Note:** The extension only controls tabs you explicitly attach. It does not auto-attach to all tabs.

---

## 4. Tips & Troubleshooting

### Gateway token rejected in extension
- Check for extra spaces when pasting the token
- Verify the port is correct (gateway port + 3, default 18792)
- Make sure the gateway is running

### Extension files are in a hidden folder
- Use `Cmd + Shift + .` in the file picker to show hidden files
- Or copy to Desktop: `cp -r $(openclaw browser extension path) ~/Desktop/openclaw-extension`

### Check what models are available
```bash
openclaw models list
```

### Find your gateway token
```bash
grep -A2 '"token"' ~/.openclaw/openclaw.json
```

### Test a custom provider endpoint directly
```bash
curl -X POST "https://your-endpoint/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-api-key" \
  -d '{"model": "your-model-id", "messages": [{"role": "user", "content": "Hello"}], "max_tokens": 50}'
```

---

*Guide written on 2026-02-25 based on hands-on setup experience.*
