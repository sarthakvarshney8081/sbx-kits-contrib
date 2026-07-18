# Installing OpenHands with Docker Sandbox

This guide walks you through setting up and running OpenHands Agent Canvas using Docker Sandbox (sbx), an isolated, secure environment for running AI agents.

## What is OpenHands?

[OpenHands](https://www.openhands.dev/) is an open-source AI agent that can:
- Write and execute code
- Run terminal commands
- Browse the web
- Interact through a rich web-based UI (Agent Canvas)

Docker Sandbox provides a secure, containerized environment to run OpenHands with automatic API key injection and network isolation.

## Prerequisites

Before you begin, ensure you have:

1. **Docker Sandbox CLI (`sbx`)** installed on your machine
   - [Installation instructions](https://docs.docker.com/ai/sandboxes/)

2. **An API key** for your preferred LLM provider:
   - **Anthropic Claude** - Get key from [console.anthropic.com](https://console.anthropic.com)
   - **OpenAI GPT** - Get key from [platform.openai.com](https://platform.openai.com/api-keys)
   - **Google Gemini** - Get key from [Google AI Studio](https://aistudio.google.com)

3. **Node.js** knowledge (basic) - The kit uses Node.js v22 internally

## Step 1: Register Your API Key

The Docker Sandbox securely stores your API key in the host's secret store—it never enters the sandbox directly.

### For Anthropic:

```bash
sbx secret set-custom -g \
    --host api.anthropic.com \
    --env ANTHROPIC_API_KEY \
    --placeholder "sk-ant-{rand}" \
    --value "YOUR_ANTHROPIC_API_KEY_HERE"
```

### For OpenAI:

```bash
sbx secret set-custom -g \
    --host api.openai.com \
    --env OPENAI_API_KEY \
    --placeholder "sk-{rand}" \
    --value "YOUR_OPENAI_API_KEY_HERE"
```

### For Google Gemini:

```bash
sbx secret set-custom -g \
    --host generativelanguage.googleapis.com \
    --env GOOGLE_API_KEY \
    --placeholder "AIza{rand}" \
    --value "YOUR_GOOGLE_API_KEY_HERE"
```

**Verify your secrets were registered:**

```bash
sbx secret list -g
```

## Step 2: Launch the OpenHands Sandbox

There are two ways to run the OpenHands kit:

### Option A: From GitHub (Recommended)

```bash
sbx run --kit "git+https://github.com/sarthakvarshney8081/sbx-kits-contrib.git#dir=openhands" openhands
```

### Option B: From a Local Clone

First, clone the repository:

```bash
git clone https://github.com/sarthakvarshney8081/sbx-kits-contrib.git
cd sbx-kits-contrib
```

Then launch the kit:

```bash
sbx run --kit ./openhands openhands
```

The sandbox will now:
1. Download the base Docker image
2. Install the `uv` package manager
3. Upgrade Node.js to v22
4. Install OpenHands Agent Canvas globally via npm
5. Create a Node.js wrapper to handle proxy settings

This typically takes 2–5 minutes on first run.

## Step 3: Access the Web UI

Once the sandbox is running and Agent Canvas is listening on port 8000, you need to expose that port to your host machine:

```bash
sbx ports openhands-sbx-kits-contrib --publish 8000:8000
```

Then open your browser and navigate to:

```
http://localhost:8000
```

You should see the OpenHands Agent Canvas UI. The sandbox proxy automatically injects your API credentials into outbound requests, so no further authentication is needed.

## Step 4: Start Using OpenHands

Inside the Agent Canvas, you can:

1. **Set your LLM provider** if prompted by the CLI
2. **Describe a task** in natural language (e.g., "Write a Python script to fetch weather data")
3. **Monitor execution** as the agent runs commands, writes files, and browses

Example tasks:
- "Create a Node.js Express server with three endpoints"
- "Fix the bug in this Python script"
- "Generate a README for my project"
- "Write unit tests for this function"

## Troubleshooting

### Port 8000 Already in Use

If port 8000 is occupied on your host, map it to a different port:

```bash
sbx ports openhands-sbx-kits-contrib --publish 8080:8000
```

Then access the UI at:

```
http://localhost:8080
```

### Missing or Incorrect API Key

Verify your secrets are correctly registered:

```bash
sbx secret list -g
```

If needed, update a secret:

```bash
sbx secret set-custom -g \
    --host api.anthropic.com \
    --env ANTHROPIC_API_KEY \
    --value "YOUR_NEW_KEY_HERE"
```

### Agent Canvas Won't Start

Check the sandbox logs:

```bash
sbx logs openhands-sbx-kits-contrib
```

Common issues:
- Node.js installation failed → Check internet connectivity
- npm install failed → Retry manually: `npm install -g @openhands/agent-canvas`
- Port binding error → Ensure nothing else is running on port 8000

### Network/Proxy Issues

The kit automatically clears proxy environment variables to prevent Python httpx from crashing. If you still encounter proxy errors:

```bash
sbx exec openhands-sbx-kits-contrib \
    printf 'HTTP_PROXY=\nHTTPS_PROXY=\nALL_PROXY=\n' >> /etc/environment
```

Then restart the sandbox.

## How Authentication Works

The Docker Sandbox kit uses a **proxy-based injection model** for security:

1. **Register secrets** once with `sbx secret set-custom`
   - Stored in your host's secure credential store

2. **Network isolation** — The sandbox declares which hosts it needs:
   - `api.anthropic.com` (Anthropic API)
   - `api.openai.com` (OpenAI API)
   - `generativelanguage.googleapis.com` (Google Gemini)
   - Package registries (npm, PyPI, etc.)
   - Git platforms (GitHub, etc.)

3. **Automatic injection** — When OpenHands makes a request:
   - The sandbox proxy intercepts it
   - Looks up the matching secret
   - Injects the appropriate auth header:
     - Anthropic: `x-api-key: <your-key>`
     - OpenAI: `Authorization: Bearer <your-key>`

**Your API key never enters the sandbox directly** — it's only injected into outgoing HTTP requests by the proxy.

## Cleaning Up

### Stop the Sandbox

```bash
sbx stop openhands-sbx-kits-contrib
```

### Remove the Sandbox

```bash
sbx rm openhands-sbx-kits-contrib
```

### Delete Stored Secrets

If you want to remove your stored API keys:

```bash
sbx secret rm -g --host api.anthropic.com
sbx secret rm -g --host api.openai.com
sbx secret rm -g --host generativelanguage.googleapis.com
```

## What Gets Installed

When the sandbox first starts, the kit automatically installs:

| Component | Version | Purpose |
|-----------|---------|---------|
| **uv** | Latest | Fast Python package manager |
| **Node.js** | v22 | Runtime for Agent Canvas |
| **npm** | Latest (with Node.js) | JavaScript package manager |
| **@openhands/agent-canvas** | Latest | The OpenHands web UI and agent |

All installations happen inside the sandbox and don't affect your host system.

## Advanced: Environment Variables

The kit pre-configures several environment variables inside the sandbox:

```yaml
WORKSPACE_DIR: /workspace          # Working directory for agent tasks
PORT: 8000                         # Web UI port
HTTP_PROXY: ""                     # Cleared to prevent httpx crashes
HTTPS_PROXY: ""
ALL_PROXY: ""
```

You can override these when launching the sandbox:

```bash
sbx run --kit ./openhands \
    --env WORKSPACE_DIR=/custom-workspace \
    --env PORT=9000 \
    openhands
```

## Next Steps

- Read the [OpenHands Documentation](https://docs.openhands.dev/)
- Explore [sbx Documentation](https://docs.docker.com/ai/sandboxes/)
- Check out other [sbx kits](https://github.com/docker/sbx-kits-contrib)

---

**Questions or issues?** Open an issue on the [sbx-kits-contrib repository](https://github.com/sarthakvarshney8081/sbx-kits-contrib/issues).
