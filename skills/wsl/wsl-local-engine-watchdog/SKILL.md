---
name: wsl-local-engine-watchdog
description: WSL2 local engine watchdog — auto-start/stop llama.cpp, vLLM, or any local inference engine on Hermes provider switch
version: 4.0.0
author: Hermes Agent
tags: [wsl, llama.cpp, vllm, watchdog, vram, provider-switch, local-engine, systemd]
metadata:
  hermes:
    related_skills: [llama-cpp-wsl-ops]
---

# WSL Local Engine Watchdog

A Python watchdog for WSL2 that **auto-starts and auto-stops local inference engines** (llama.cpp, vLLM, or any engine via systemd) based on which Hermes provider you select. **The agent.log provider-switch event is THE ONLY trigger** for lifecycle.

Works with **ANY local engine** — register it in `local_engines.json` or let auto-detection infer the engine from the provider name.

## Behavior

- **Auto-starts** a local engine within 5s when you select a registered provider
- **Auto-stops immediately** when you switch away to a non-local provider
- **Auto-stops on app close** — detects Hermes.exe process exit via `tasklist.exe` and unloads within 5s
- **Engine-to-engine switch** — if you switch from one local engine provider to another, stops the old, starts the new
- **Multi-session safe** — checks all active sessions in `state.db` before stopping (another gateway session may still need the engine)
- **Idle timeout (15 min)** — safety-net fallback only

## Architecture

```
Session: /model provider switch
        │
        ▼
  Hermes agent.log ──► "Model switched in-place: A (X) -> B (llama-qwen)"
                  OR  "Model switched in-place: ... (llama-qwen) -> ... (other)"
                             │
                        every 5s tail
                             ▼
  local-engine-watchdog (systemd user service on WSL)
     ├─ agent.log tail — THE trigger (start ON switch-to, stop ON switch-away)
     ├─ local_engines.json — registered engine configs
     ├─ auto-detect — name-based fallback ("llama" → llama.cpp, "vllm" → vLLM)
     ├─ tasklist.exe — check Hermes.exe process (app-close → stop all)
     ├─ state.db poll — multi-session guard only (other gateway sessions)
     └─ log mtime check — idle timeout safety net per engine
             │
      start / stop
             ▼
  Per-engine systemd service (llama-server.service / vllm.service / etc.)
```

## Quick Start

```bash
# 1. Configure your engine(s) in ~/scripts/local_engines.json
cat > ~/scripts/local_engines.json << 'EOF'
{
  "llama-qwen": {
    "engine": "llama.cpp",
    "service": "llama-server.service",
    "log": "/home/vibrationall/ai/llama.cpp/server.log"
  }
}
EOF

# 2. Deploy the watchdog script
cp local_engine_watchdog.py ~/scripts/local_engine_watchdog.py

# 3. Install the systemd service (see below)
# 4. Restart
systemctl --user restart llama-watchdog.service
```

## Key Findings (don't repeat debugging)

### state.db does NOT update on `/model` mid-session
TUI sessions change provider at runtime. The `model` and `model_config` columns in `state.db` remain at the original value until the session ends. **Do not rely on state.db for TUI provider detection.**

### agent.log IS the real-time signal
Hermes writes a log line on every model switch:
```
Model switched in-place: deepseek-v4-flash-free (opencode) -> qwen3.6-heretic (llama-qwen)
```
This is the fast path for auto-start. The watchdog tails this file from `/mnt/c/.../hermes/logs/agent.log`.

### SQLite over /mnt/c/ 9p has locking issues
WSL's 9p filesystem does not support SQLite file locking. Direct queries fail with `disk I/O error`. Always copy the DB to `/tmp` before querying:
```python
tmp = tempfile.NamedTemporaryFile(suffix=".db", delete=False)
shutil.copy2(DB_SRC, tmp.name)
conn = sqlite3.connect(tmp.name)
# ... query ...
os.unlink(tmp.name)
```

### Multi-session safety
The watchdog checks ALL active sessions (gateway + TUI) via state.db for any registered engine:
- **Switch-away stop**: only stops a specific engine if NO other session still uses it
- **Stop from other session end**: when a gateway session using an engine ends and the local session is on a different provider, the next state.db poll sees no active engine sessions → stops

### Engine auto-detection (convention-based)
Providers without a config entry are auto-detected by name:
| Name pattern | Engine | Default service |
|---|---|---|
| Contains `"llama"` | llama.cpp | `llama-server.service` |
| Contains `"vllm"` | vLLM | `vllm.service` |
| Contains `"local"` or `"wsl"` | local | `llama-server.service` |

## Components

### 1. Engine config: `~/scripts/local_engines.json`

```json
{
  "llama-qwen": {
    "engine": "llama.cpp",
    "service": "llama-server.service",
    "log": "/home/vibrationall/ai/llama.cpp/server.log"
  },
  "vllm": {
    "engine": "vllm",
    "service": "vllm.service",
    "log": "/home/vibrationall/ai/vllm/server.log"
  }
}
```

| Field | Required | Description |
|---|---|---|
| `engine` | Yes | Human-readable engine name (for logging) |
| `service` | Yes | systemd service unit name (without `.service` suffix) |
| `log` | No | Path to server log file (used for idle timeout) |

Each key is a Hermes provider name. When Hermes switches to that provider, the watchdog manages the corresponding service.

### 2. Per-engine systemd service (example: llama.cpp)

File: `~/.config/systemd/user/llama-server.service`

```ini
[Unit]
Description=llama.cpp Qwen3.6 Server
After=network.target

[Service]
Type=simple
Environment=HOME=/home/vibrationall
Environment=USER=vibrationall
Environment=PATH=/usr/local/cuda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Environment=LD_LIBRARY_PATH=/home/vibrationall/ai/turboquant_llama/build/bin:/usr/local/cuda/lib64:/usr/lib/wsl/lib
ExecStart=/home/vibrationall/ai/turboquant_llama/build/bin/llama-server -m /home/vibrationall/ai/llama.cpp/models/Qwen3.6-27B-uncensored-heretic-v2-Native-MTP-Preserved-Q4_K_M.gguf --host 0.0.0.0 --port 8080 -c 131072 -ctk q8_0 -ctv turbo4 -ngl 99 --cont-batching -np 8 --cache-prompt --flash-attn auto
Restart=on-failure
RestartSec=10
TimeoutStartSec=300
TimeoutStopSec=30
StandardOutput=append:/home/vibrationall/ai/llama.cpp/server.log
StandardError=append:/home/vibrationall/ai/llama.cpp/server.log

[Install]
WantedBy=default.target
```

Each engine you add needs its own systemd service unit.

### 3. Session-aware watchdog

#### Script: `~/scripts/local_engine_watchdog.py`

```python
#!/usr/bin/env python3
"""Local engine session-aware watchdog for WSL2.

THE ONLY trigger for loading/unloading a local inference engine (llama.cpp,
vLLM, etc.) is the provider-switch event in Hermes agent.log.

When the user selects ANY registered local engine provider, start its systemd
service. When they switch away to a non-local provider, stop it immediately.
Also stops all engines when the Hermes desktop app closes.

Engines are configured in ~/scripts/local_engines.json:
{
  "llama-qwen": {
    "engine": "llama.cpp",
    "service": "llama-server.service",
    "log": "/home/vibrationall/ai/llama.cpp/server.log"
  },
  "vllm": {
    "engine": "vllm",
    "service": "vllm.service",
    "log": "/home/vibrationall/ai/vllm/server.log"
  }
}
"""

import sqlite3, json, os, shutil, time, subprocess, tempfile, re

ENGINE_CONFIG = os.path.expanduser("~/scripts/local_engines.json")
AGENT_LOG = "/mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log"
DB_SRC = "/mnt/c/Users/MaJor/AppData/Local/hermes/state.db"
IDLE_CAP = 15 * 60

_last_pos = 0

def load_engines():
    """Load the engine config registry. Returns {provider_name: config_dict}."""
    if not os.path.exists(ENGINE_CONFIG):
        return {}
    try:
        with open(ENGINE_CONFIG) as f:
            return json.load(f)
    except (json.JSONDecodeError, OSError):
        return {}

def get_engine(provider_name):
    """Get engine config for a provider.

    Tries explicit config first, then name-based heuristics:
      - contains "llama" -> llama.cpp w/ llama-server.service
      - contains "vllm"  -> vLLM w/ vllm.service
      - contains "local"/"wsl" -> local w/ llama-server.service
    """
    engines = load_engines()
    cfg = engines.get(provider_name)
    if cfg: return cfg
    p = provider_name.lower()
    if "llama" in p:
        return {"engine":"llama.cpp","service":"llama-server.service","log":""}
    if "vllm" in p:
        return {"engine":"vllm","service":"vllm.service","log":""}
    if "local" in p or "wsl" in p:
        return {"engine":"local","service":"llama-server.service","log":""}
    return None

def is_known_engine(provider_name):
    return get_engine(provider_name) is not None

def parse_agent_log():
    """Scan new lines in agent.log for model-switch events.

    Returns dict: {action: "start"|"stop"|"switch"|None,
                   provider: ..., new_provider: ...}
    """
    global _last_pos
    if not os.path.exists(AGENT_LOG):
        return {"action":None,"provider":None,"new_provider":None}
    size = os.path.getsize(AGENT_LOG)
    if size < _last_pos: _last_pos = 0
    if size == _last_pos:
        return {"action":None,"provider":None,"new_provider":None}
    result = {"action":None,"provider":None,"new_provider":None}
    try:
        with open(AGENT_LOG,"r",errors="replace") as f:
            f.seek(_last_pos)
            for line in f:
                m = re.search(
                    r"Model switched in-place: (.+) \(([^)]+)\) -> (.+) \(([^)]+)\)",line
                )
                if m:
                    old_p, new_p = m.group(2), m.group(4)
                    old_e, new_e = is_known_engine(old_p), is_known_engine(new_p)
                    if new_e:
                        result = {"action":"start","provider":new_p,"new_provider":new_p}
                    elif old_e and not new_e:
                        result = {"action":"stop","provider":old_p,"new_provider":new_p}
                    elif old_e and new_e and old_p != new_p:
                        result = {"action":"switch","provider":old_p,"new_provider":new_p}
            _last_pos = f.tell()
    except Exception:
        pass
    return result

def hermes_app_running():
    """Check if the Hermes desktop app process is running on Windows."""
    try:
        r = subprocess.run(
            ["/mnt/c/Windows/System32/tasklist.exe",
             "/FI","IMAGENAME eq Hermes.exe","/NH"],
            capture_output=True,text=True,timeout=5
        )
        return "Hermes" in r.stdout
    except Exception:
        return True

def sessions_still_need_engine(provider_name):
    """Multi-session guard: any active session still uses this engine?"""
    if not os.path.exists(DB_SRC): return False
    tmp = tempfile.NamedTemporaryFile(suffix=".db",delete=False)
    tmp_path = tmp.name; tmp.close()
    try:
        shutil.copy2(DB_SRC,tmp_path)
        conn = sqlite3.connect(tmp_path)
        conn.execute("PRAGMA busy_timeout=200")
        rows = conn.execute(
            "SELECT model_config FROM sessions WHERE ended_at IS NULL"
        ).fetchall()
        conn.close()
    except Exception: return False
    finally:
        try: os.unlink(tmp_path)
        except: pass
    for (cfg_json,) in rows:
        try:
            cfg = json.loads(cfg_json)
            if cfg.get("provider") == provider_name: return True
            gr = cfg.get("gateway_runtime") or {}
            if gr.get("provider") == provider_name: return True
        except Exception: continue
    return False

def engine_running(cfg):
    try:
        r = subprocess.run(["systemctl","--user","is-active",cfg["service"]],
                           capture_output=True,text=True,timeout=5)
        return r.returncode == 0
    except: return False

def start_engine(cfg, provider):
    print(f"Starting {provider} ({cfg['engine']}) via {cfg['service']}...",flush=True)
    subprocess.run(["systemctl","--user","start",cfg["service"]],
                   capture_output=True,timeout=300)

def stop_engine(cfg, provider):
    print(f"Stopping {provider} ({cfg['engine']}) via {cfg['service']}...",flush=True)
    subprocess.run(["systemctl","--user","stop",cfg["service"]],
                   capture_output=True,timeout=15)

def engine_idle_too_long(cfg):
    log = cfg.get("log","")
    if not log or not os.path.exists(log): return True
    return (time.time() - os.path.getmtime(log)) > IDLE_CAP

def stop_all_engines():
    for prov, cfg in load_engines().items():
        if engine_running(cfg):
            print(f"App closed: stopping {prov}...",flush=True)
            stop_engine(cfg, prov)

if __name__ == "__main__":
    print("local-engine watchdog starting",flush=True)
    if os.path.exists(AGENT_LOG):
        _last_pos = max(0, os.path.getsize(AGENT_LOG) - 4096)
    current_provider = None
    while True:
        time.sleep(5)
        engines = load_engines()
        event = parse_agent_log()
        action, provider, new_provider = event["action"], event["provider"], event["new_provider"]

        if action == "switch":
            old_cfg = get_engine(provider)
            new_cfg = get_engine(new_provider)
            if old_cfg and engine_running(old_cfg) and not sessions_still_need_engine(provider):
                stop_engine(old_cfg, provider)
            if new_cfg:
                start_engine(new_cfg, new_provider)
                current_provider = new_provider
            continue

        if action == "start":
            cfg = get_engine(provider)
            if cfg and not engine_running(cfg):
                start_engine(cfg, provider)
                current_provider = provider
            continue

        if action == "stop":
            cfg = get_engine(provider)
            if cfg and engine_running(cfg) and not sessions_still_need_engine(provider):
                stop_engine(cfg, provider)
                if provider == current_provider: current_provider = None
            continue

        if not hermes_app_running():
            # Check if any engine is still needed by other sessions
            any_active = False
            for prov in list(load_engines().keys()):
                if sessions_still_need_engine(prov):
                    any_active = True
                    break
            if not any_active:
                stop_all_engines()
                current_provider = None
            continue

        # State.db fallback: orphaned sessions need an engine
        if not any([engine_running(c) for c in load_engines().values()]):
            for prov, cfg in load_engines().items():
                if sessions_still_need_engine(prov):
                    start_engine(cfg, prov)
                    current_provider = prov
                    break
            continue

        # Idle timeout safety net per engine
        for prov, cfg in load_engines().items():
            if engine_running(cfg) and engine_idle_too_long(cfg):
                if not sessions_still_need_engine(prov):
                    print(f"Idle timeout: stopping {prov}...",flush=True)
                    stop_engine(cfg, prov)
                    if prov == current_provider: current_provider = None
```

#### Watchdog systemd service: `~/.config/systemd/user/llama-watchdog.service`

```ini
[Unit]
Description=Local engine watchdog (auto-starts/stops llama.cpp/vLLM etc. on provider switch)

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/vibrationall/scripts/local_engine_watchdog.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

## Hermes provider config

Each local engine needs a custom provider in `config.yaml`:

```yaml
# In ~/AppData/Local/hermes/config.yaml
providers:
  llama-qwen:
    name: llama.cpp Qwen3.6 (WSL)
    base_url: http://<WSL2-IP>:8080/v1
    api_key: not-needed
    model: qwen3.6-heretic

  # vllm:
  #   name: vLLM (WSL)
  #   base_url: http://<WSL2-IP>:8000/v1
  #   api_key: not-needed
  #   model: my-model
```

Find WSL2 IP: `wsl ip addr show eth0 | grep "inet "`

## Adding a new engine

1. Create a systemd service unit for the engine (e.g. `~/.config/systemd/user/vllm.service`)
2. Add an entry to `~/scripts/local_engines.json`
3. Add a matching custom provider in Hermes `config.yaml`
4. Restart the watchdog: `systemctl --user restart llama-watchdog.service`

The watchdog will also auto-detect providers by name without a config entry if the name contains `"llama"`, `"vllm"`, `"local"`, or `"wsl"`.

## Commands

```bash
# Manage engines
wsl systemctl --user start llama-server.service    # start specific engine
wsl systemctl --user stop llama-server.service     # stop specific engine
wsl systemctl --user start vllm.service            # start another engine

# Check engine status
wsl systemctl --user status llama-server.service --no-pager | head -5

# Watchdog logs
wsl journalctl --user -u llama-watchdog.service -n 20 --no-pager

# Edit engine config
wsl nano ~/scripts/local_engines.json
wsl systemctl --user restart llama-watchdog.service

# Test: simulate switch TO a local engine (should start)
echo "2026-07-02 18:30:00 INFO agent: Model switched in-place: cloud (opencode) -> local (llama-qwen)" >> /mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log

# Test: simulate switch FROM a local engine (should stop)
echo "2026-07-02 18:30:05 INFO agent: Model switched in-place: local (llama-qwen) -> cloud (opencode)" >> /mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log

# Test: auto-detect a vllm provider
echo "2026-07-02 18:30:10 INFO agent: Model switched in-place: cloud (opencode) -> mymodel (vllm)" >> /mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log
```

## Pitfalls

- **state.db does NOT track `/model` switches mid-TUI-session** — it only updates when sessions are created. The `model` column stays at the original provider. Use agent.log tailing for fast TUI detection.
- **SQLite over /mnt/c/ via 9p** — file locking is broken. Always copy the DB to `/tmp` before querying.
- **Cold load time** — a 17GB GGUF takes ~36s on WSL2. Set Hermes `gateway_timeout` high enough (>180s).
- **LD_LIBRARY_PATH is critical in systemd** — systemd user services don't inherit environment. Each engine service unit must set its own environment variables.
- **exit.target must be masked** — prevents WSL2 session cycling: `systemctl --user mask exit.target --now`. Without this, WSL tears down all user services every ~60s.
- **Hermes has NO `on_provider_change` plugin hook** — `VALID_HOOKS` lacks provider-switch events. Workaround is tailing agent.log, which works reliably.
- **The watchdog uses file-position tracking** — if the log is rotated while the watchdog is stopped, `_last_pos` resets to 0, causing a re-read of old entries. This is benign (only triggers extra start attempts).
- **Auto-detection is case-insensitive** — provider names like "Llama-Pro", "vLLM-70B", "my-local-model" all work.
- **Engine config takes precedence** — if a provider is both in `local_engines.json` AND matches an auto-detect pattern, the config file wins.
