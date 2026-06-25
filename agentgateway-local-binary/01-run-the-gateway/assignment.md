---
slug: run-the-gateway
id: ln8s1lz2ixsf
type: challenge
title: Run the Gateway (Binary + External SQLite)
teaser: Start Agentgateway from the command line, backed by an on-disk SQLite database.
notes:
- type: text
  contents: "# \U0001F680 Run the Gateway\n\nNo Docker here — Agentgateway is a single
    binary. You'll run it from the terminal,\npointed at an external SQLite database
    on disk, and connect to its built-in UI.\n"
tabs:
- id: pbmilzgfgbfy
  title: Terminal
  type: terminal
  hostname: server
- id: ezhmrwbowxco
  title: Editor
  type: code
  hostname: server
  path: /root/agentgateway
- id: awqpzomfcdrm
  title: Agentgateway UI
  type: service
  hostname: server
  path: /ui
  port: 15000
difficulty: ""
enhanced_loading: null
---

# Run the Gateway (Binary + External SQLite)

**What we're doing:** running Agentgateway as a **local binary** (no containers), with
its request database living in an **external SQLite file** at `/data/data.db`. Two
files are already staged in `/root/agentgateway`: `config.yaml` and the cost catalog
`base-costs.json`.

## Step 1 — Confirm the binary and the config

The binary is already installed (this is how you'd do it on any box):

```bash
# how it was installed (already done for you):
#   curl -sL https://agentgateway.dev/install | bash
agentgateway --version
```

Open **`/root/agentgateway/config.yaml`** in the Editor. Note the database line — the
gateway writes to an **external SQLite file**, separate from the app:

```yaml
config:
  database:
    url: "sqlite:///data/data.db"   # absolute /data/data.db (3 slashes)
```

> 🧠 SQLite URL slashes matter: `sqlite:///data/data.db` is the **absolute** path
> `/data/data.db` (the `/data` dir already exists). A *relative* `./data/data.db`
> would be `sqlite://data/data.db` (two slashes).

## Step 2 — Validate, then run it

```bash
agentgateway -f /root/agentgateway/config.yaml --validate-only      # Configuration is valid!
nohup agentgateway -f /root/agentgateway/config.yaml > /root/agentgateway/agw.log 2>&1 &
sleep 4
```

All paths in the config are absolute, so it runs from any directory.

## Step 3 — Confirm it's up and the database was created

```bash
curl -s -o /dev/null -w "admin HTTP %{http_code}\n" http://localhost:15000/config_dump
ls -la /data/data.db
sqlite3 /data/data.db '.tables'
```

You should see `HTTP 200`, the **`/data/data.db`** file created by the gateway, and the
tables `request_logs` and `request_log_payloads`. Open the **Agentgateway UI** tab to
see it in the browser.

> Next: add an LLM, send a call, and read it back out of the SQLite file. ➡️
