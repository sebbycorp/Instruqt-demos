# Agentgateway Local: Run the Binary with an External SQLite DB (Instruqt track)

A short 2-lab track that runs Agentgateway as a **local binary** (no Docker), backed
by an **external on-disk SQLite database**. Built as a manual/scratch alternative to
the Docker-based `00-agentgateway-tokenomic` track.

## Labs
1. **run-the-gateway** — install/verify the binary, run it against a config whose DB is
   `sqlite:///data/data.db`, open the UI, confirm `/data/data.db` is created.
2. **add-an-llm-and-inspect-db** — add an OpenAI model, reload (pkill + nohup), send a
   call, then query `/data/data.db` directly to see the logged request.

## How it runs
- Binary: `curl -sL https://agentgateway.dev/install | bash` (installed at track setup; runs as root on the VM).
- Config: `/root/agentgateway/config.yaml`, cost catalog `/root/agentgateway/base-costs.json` (absolute paths → CWD-independent).
- External DB: SQLite file at `/data/data.db` (`sqlite:///data/data.db` = absolute `/data/data.db`).
- `OPENAI_API_KEY` is an Instruqt secret; the gateway reads it from the environment.
- No command aliases — labs use the real `agentgateway` / `nohup` / `pkill` / `sqlite3` commands.

## SQLite URL slashes (gotcha)
- `sqlite:///data/data.db` → absolute `/data/data.db` (needs `/data` to exist).
- `sqlite://data/data.db` → relative `./data/data.db`.
- `sqlite:////abs/path.db` → absolute (4 slashes).

## Develop / test
```bash
instruqt track validate
instruqt track push
instruqt track test agentgateway-local-binary --skip-fail-check
```
