# Agent Sandbox Quickstart

Run **Claude Code** or **OpenCode** inside an isolated container — the agent works in a directory you choose, and the rest of your filesystem stays untouched.

Two container runtimes are covered:
- **Docker** — most laptops and workstations
- **Apptainer** — HPC clusters, where Docker is usually unavailable

You only need one of them. The same container image works for both agents.

---

## Prerequisites

- Linux or macOS host
- **Docker** *or* **Apptainer** installed
- **Claude Code** *or* **OpenCode** installed on the host (the container reuses your auth files)

Verify:
```bash
docker --version            # or: apptainer --version
claude --version            # or: opencode --version
```

Make sure your agent is authenticated on the host before launching the sandbox. Auth methods vary (OAuth login, API keys, alternative endpoints like Bedrock/Vertex) — see [Auth](#auth) at the end for how to carry each one into the container.

---

## 1. Build the image (one-time)

### Docker

`Dockerfile`:
```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
RUN pip install uv
RUN useradd -m -u 1000 sandbox

WORKDIR /work
USER sandbox
```

```bash
docker build -t agent-sandbox .
```

### Apptainer

`sandbox.def`:
```
Bootstrap: docker
From: python:3.12-slim

%post
    apt-get update && apt-get install -y git && rm -rf /var/lib/apt/lists/*
    pip install uv
```

```bash
sudo apptainer build agent-sandbox.sif sandbox.def
```

---

## 2. Run the agent

Replace `/path/to/your/project` with the directory you want the agent to work in (can be empty).

### Claude Code in Docker
```bash
docker run -it --rm \
  -v /path/to/your/project:/work \
  -v $HOME/.claude:/home/sandbox/.claude:ro \
  -v $HOME/.claude.json:/home/sandbox/.claude.json:ro \
  -v $(which claude):/usr/local/bin/claude:ro \
  --workdir /work \
  agent-sandbox \
  claude
```

### Claude Code in Apptainer
```bash
apptainer shell \
  --contain --no-home \
  --bind /path/to/your/project:/work \
  --bind $HOME/.claude:$HOME/.claude:ro \
  --bind $HOME/.claude.json:$HOME/.claude.json:ro \
  --bind $(which claude):/usr/local/bin/claude:ro \
  --pwd /work \
  agent-sandbox.sif
# Then: claude
```

### OpenCode in Docker
```bash
docker run -it --rm \
  -v /path/to/your/project:/work \
  -v $(which opencode):/usr/local/bin/opencode:ro \
  --workdir /work \
  agent-sandbox \
  opencode
```

### OpenCode in Apptainer
```bash
mkdir -p /tmp/opencode-scratch/.local /tmp/opencode-scratch/.cache

apptainer shell \
  --contain --no-home \
  --bind /path/to/your/project:/work \
  --bind $(which opencode):/usr/local/bin/opencode:ro \
  --bind /tmp/opencode-scratch/.local:$HOME/.local \
  --bind /tmp/opencode-scratch/.cache:$HOME/.cache \
  --pwd /work \
  agent-sandbox.sif
# Then: opencode
```

> These minimal commands launch OpenCode in the sandbox but don't carry your auth in. See [OpenCode auth](#opencode-auth) below for the extra binds/env vars needed for your auth method.

---

## 3. Verify the isolation actually holds

Inside the agent session:

```bash
ls /home          # Only the sandbox user — host home invisible
cat /etc/shadow   # Docker: permission denied
                  # Apptainer: returns the image's stock /etc/shadow
                  #   (no host secrets) — host's /etc/shadow stays unreachable
echo $HOME        # Docker: /home/sandbox  |  Apptainer: your host $HOME (but empty)
```

Then write a file:
```bash
echo 'print("hello from inside the sandbox")' > hello.py
python3 hello.py
```

`hello.py` appears in `/path/to/your/project` on your host — that bound directory is the only place the agent can persist anything. Everything else disappears on exit.

---

## Gotchas

- **Apptainer `-e` is `--cleanenv`, not "set env var".** Use the `APPTAINERENV_FOO=bar apptainer ...` prefix instead.
- **OpenCode needs writable `.local` / `.cache`.** With `--no-home` they don't exist — bind fresh `/tmp` dirs as shown.
- **OpenCode config is usually a symlink.** If you bind it (see auth section below), use `readlink -f` and bind **only** the resolved file — never the directory. Directory binds leak host paths in Docker and break config loading in Apptainer.

---

## Auth

How you carry auth into the sandbox depends on how your agent authenticates on the host. Both agents support multiple modes; pick the one matching your setup and add the binds/env vars to the §2 commands.

### OpenCode

**Option A — OAuth login (`opencode auth login` — works for Anthropic, OpenAI, GitHub Copilot, Azure, and any other provider OpenCode supports)**

Bind the OpenCode data directory read-only so the container can read your auth tokens:

```bash
# Docker — add to the run command:
-v $HOME/.local/share/opencode:/home/sandbox/.local/share/opencode:ro

# Apptainer — add to the shell command:
--bind $HOME/.local/share/opencode:$HOME/.local/share/opencode:ro
```

(Path may vary by OS; on macOS it's typically `~/Library/Application Support/opencode/`.)

**Option B — API key via env var referenced in `opencode.json`**

Bind the resolved config file (not the directory — see Gotchas) and pass the env var. Replace `AZURE_OPENAI_KEY` with whatever your `opencode.json` references (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.):

```bash
# Docker — add to the run command:
-v $(readlink -f $HOME/.config/opencode/opencode.json):/home/sandbox/.config/opencode/opencode.json:ro \
-e AZURE_OPENAI_KEY=$AZURE_OPENAI_KEY

# Apptainer — prefix the shell command and add a bind:
APPTAINERENV_AZURE_OPENAI_KEY=$AZURE_OPENAI_KEY \
apptainer shell \
  ... \
  --bind $(readlink -f $HOME/.config/opencode/opencode.json):$HOME/.config/opencode/opencode.json:ro \
  ...
```

### Claude Code

The §2 commands already cover **Anthropic OAuth** (the bound `~/.claude/` + `~/.claude.json`). For other auth modes, add to the §2 command:

- **`ANTHROPIC_API_KEY` env var** — pass it in: `-e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY` (Docker) or `APPTAINERENV_ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY apptainer ...`
- **AWS Bedrock** — set `CLAUDE_CODE_USE_BEDROCK=1` (same env-var pattern) and bind your AWS creds: `-v $HOME/.aws:/home/sandbox/.aws:ro` (or `--bind`). Set `AWS_REGION` too.
- **Google Vertex AI** — set `CLAUDE_CODE_USE_VERTEX=1` and bind your GCP creds: `-v $HOME/.config/gcloud:/home/sandbox/.config/gcloud:ro` (or `--bind`).
- **Custom endpoint** — pass `ANTHROPIC_BASE_URL` (and an auth token env var if the proxy requires one) the same way.

---

## Alternative tools

The Docker / Apptainer pattern above is the same shape as the recommended approach. If you want something more polished or fit to a different context:

- **Anthropic's official devcontainer** — Same Docker-based approach, plus an `iptables` egress allowlist (only `api.anthropic.com`, npm, GitHub). Best fit if you live in VS Code. → [code.claude.com/docs/en/devcontainer](https://code.claude.com/docs/en/devcontainer) and [anthropics/claude-code/.devcontainer](https://github.com/anthropics/claude-code/tree/main/.devcontainer)
- **Community Docker wrappers** — `claude-code-sandbox` and similar projects on GitHub. Convenience scripts around the same pattern.
- **Bubblewrap / firejail (Linux) or `sandbox-exec` (macOS)** — OS-level process sandboxing without container images. Lighter weight, but profile rules are easier to get wrong than container isolation.
- **E2B, Daytona, Modal, Vercel Sandbox** — Cloud sandboxes where *the agent spawns* a sandbox to execute code (like ChatGPT's code interpreter). Different problem — not for hosting the agent CLI itself.

**Key point for the lecture:** sandboxing and the permission model are *orthogonal layers*. Broad permission grants (`Bash(*)`, `python -c *`) are how prompt injection escapes a non-sandboxed setup. Inside a sandbox they're recoverable — worst case is a trashed container. That's what lets you accept-accept-accept without it being the same kind of risk.
