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

For OpenCode, you need your model provider's API key in your environment (Azure shown — adjust for OpenAI / Anthropic / etc.):
```bash
echo $AZURE_OPENAI_KEY      # should be non-empty
```

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
  -v $(readlink -f $HOME/.config/opencode/opencode.json):/home/sandbox/.config/opencode/opencode.json:ro \
  -v $(which opencode):/usr/local/bin/opencode:ro \
  -e AZURE_OPENAI_KEY=$AZURE_OPENAI_KEY \
  --workdir /work \
  agent-sandbox \
  opencode
```

### OpenCode in Apptainer
```bash
mkdir -p /tmp/opencode-scratch/.local /tmp/opencode-scratch/.cache

APPTAINERENV_AZURE_OPENAI_KEY=$AZURE_OPENAI_KEY \
APPTAINERENV_AZURE_API_KEY=$AZURE_OPENAI_KEY \
apptainer shell \
  --contain --no-home \
  --bind /path/to/your/project:/work \
  --bind $(readlink -f $HOME/.config/opencode/opencode.json):$HOME/.config/opencode/opencode.json:ro \
  --bind $(which opencode):/usr/local/bin/opencode:ro \
  --bind /tmp/opencode-scratch/.local:$HOME/.local \
  --bind /tmp/opencode-scratch/.cache:$HOME/.cache \
  --pwd /work \
  agent-sandbox.sif
# Then: opencode
```

> The OpenCode examples use `readlink -f` and bind **only** the resolved config file (not `~/.config/opencode/`) — binding the directory leaks host paths in Docker and breaks config loading in Apptainer.

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

- **OpenCode config is usually a symlink.** Bind only the resolved file (`readlink -f`), never the directory. See the note in §2.
- **Apptainer `-e` is `--cleanenv`, not "set env var".** Use the `APPTAINERENV_FOO=bar apptainer ...` prefix instead.
- **OpenCode needs writable `.local` / `.cache`.** With `--no-home` they don't exist — bind fresh `/tmp` dirs as shown.
