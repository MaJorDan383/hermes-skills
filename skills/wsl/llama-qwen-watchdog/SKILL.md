---
name: llama-qwen-watchdog
description: WSL2 llama-server with TurboQuant + session-aware watchdog that auto-starts/auto-stops on provider switch and auto-stops on Hermes app close
version: 3.0.0
author: Hermes Agent
tags: [llama.cpp, wsl, watchdog, vram, provider-switch, turboquant, agent-log-tail]
metadata:
  hermes:
    related_skills: [llama-cpp-wsl-ops]
---

# llama-qwen Watchdog

Configures a WSL2 llama-server (TurboQuant fork) with a Python watchdog where
**the agent.log provider-switch event is THE ONLY trigger** for lifecycle:
- **Auto-starts** within 5s when you select `llama-qwen` provider (switch TO)
- **Auto-stops immediately** when you switch FROM llama-qwen to a non-local provider
- **Auto-stops on app close** — detects Hermes.exe process exit via tasklist and unloads within 5s
- Idle timeout (15 min) is a safety-net fallback only, not the primary unload mechanism
- state.db is polled only for **multi-session safety** (gateway sessions)

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
  llama-watchdog (systemd user service on WSL)
     ├─ agent.log tail — THE trigger (start ON switch-to, stop ON switch-away)
     ├─ tasklist.exe — check Hermes.exe process (app-close → stop)
     ├─ state.db poll — multi-session guard only (other gateway sessions)
     └─ log mtime check — idle timeout safety net only
             │
      start / stop
             ▼
  llama-server (systemd user service)
  -c 131072 -ctk q8_0 -ctv turbo4 -ngl 99 --cont-batching -np 8
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
The watchdog checks ALL active sessions (gateway + TUI) via state.db to decide:
- **Switch-away stop**: only stops if NO other session uses `llama-qwen`
- **Stop from other session end**: when a gateway session using `llama-qwen` ends and the local session is on a different provider, a 5s poll cycle catches it — next state.db poll sees no active sessions → stops

## Components

### 1. llama-server systemd service

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

Flags explained:
| Flag | Value | Why |
|------|-------|-----|
| `-c` | `131072` | 128K context (split across 8 parallel slots = 16384 each) |
| `-ctk` | `q8_0` | K cache in 8-bit (TurboQuant optimized) |
| `-ctv` | `turbo4` | V cache in TurboQuant's 4-bit format |
| `-ngl` | `99` | Full GPU offload |
| `--cont-batching` | — | Continuous batching |
| `-np` | `8` | 8 parallel slots |
| `--cache-prompt` | — | Prompt caching enabled |
| `--flash-attn` | `auto` | Flash Attention (auto-detect) |

### 2. Session-aware watchdog

#### Script: `~/scripts/llama_session_watchdog.py`

```python
#!/usr/bin/env python3
"""llama-server session-aware watchdog.

THE ONLY trigger for loading/unloading the llama model is the provider
switch detected via agent.log. When the user selects llama-qwen, load it.
Also unloads when the Hermes desktop app closes (no local or gateway session
uses llama-qwen anymore).

state.db is checked for multi-session safety — if another active session
(gateway) still uses llama-qwen, don't unload even on local switch-away.

Idle timeout is a safety-net fallback only (not the primary unload mechanism).
"""

import sqlite3, json, os, shutil, time, subprocess, tempfile, re

AGENT_LOG = "/mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log"
DB_SRC   = "/mnt/c/Users/MaJor/AppData/Local/hermes/state.db"
LOG      = "/home/vibrationall/ai/llama.cpp/server.log"
SERVICE  = "llama-server.service"
IDLE_CAP = 15 * 60

_last_pos = 0

def parse_agent_log():
    """Scan new lines in agent.log for model-switch events.

    Returns: "start" (switched TO llama-qwen),
             "stop"  (switched AWAY from llama-qwen),
             None    (no relevant event).
    """
    global _last_pos
    if not os.path.exists(AGENT_LOG):
        return None
    size = os.path.getsize(AGENT_LOG)
    if size < _last_pos:
        _last_pos = 0
    if size == _last_pos:
        return None
    action = None
    try:
        with open(AGENT_LOG, "r", errors="replace") as f:
            f.seek(_last_pos)
            for line in f:
                m = re.search(
                    r"Model switched in-place: (.+) \(([^)]+)\) -> (.+) \(([^)]+)\)",
                    line
                )
                if m:
                    old_p = m.group(2); new_p = m.group(4)
                    if new_p == "llama-qwen":
                        action = "start"
                    elif old_p == "llama-qwen" and new_p != "llama-qwen":
                        action = "stop"
            _last_pos = f.tell()
    except Exception:
        pass
    return action

def hermes_app_running():
    """Check if the Hermes desktop app process is running on Windows."""
    try:
        r = subprocess.run(
            ["/mnt/c/Windows/System32/tasklist.exe",
             "/FI", "IMAGENAME eq Hermes.exe", "/NH"],
            capture_output=True, text=True, timeout=5
        )
        return "Hermes" in r.stdout
    except Exception:
        return True

def any_session_uses_llama_qwen():
    """Multi-session guard: check if any active session still needs llama-qwen."""
    if not os.path.exists(DB_SRC):
        return False
    tmp = tempfile.NamedTemporaryFile(suffix=".db", delete=False)
    tmp_path = tmp.name; tmp.close()
    try:
        shutil.copy2(DB_SRC, tmp_path)
        conn = sqlite3.connect(tmp_path)
        conn.execute("PRAGMA busy_timeout=200")
        rows = conn.execute(
            "SELECT model_config FROM sessions WHERE ended_at IS NULL"
        ).fetchall()
        conn.close()
    except Exception:
        return False
    finally:
        try: os.unlink(tmp_path)
        except: pass
    for (cfg_json,) in rows:
        try:
            cfg = json.loads(cfg_json)
            if cfg.get("provider") == "llama-qwen": return True
            gr = cfg.get("gateway_runtime") or {}
            if gr.get("provider") == "llama-qwen": return True
        except Exception:
            continue
    return False

def idle_too_long():
    if not os.path.exists(LOG): return True
    return (time.time() - os.path.getmtime(LOG)) > IDLE_CAP

def server_running():
    try:
        r = subprocess.run(["systemctl","--user","is-active",SERVICE],
                           capture_output=True,text=True,timeout=5)
        return r.returncode == 0
    except: return False

def stop_server():
    subprocess.run(["systemctl","--user","stop",SERVICE],
                   capture_output=True,timeout=15)

def start_server():
    subprocess.run(["systemctl","--user","start",SERVICE],
                   capture_output=True,timeout=300)

if __name__ == "__main__":
    if os.path.exists(AGENT_LOG):
        _last_pos = max(0, os.path.getsize(AGENT_LOG) - 4096)
    while True:
        time.sleep(5)
        running = server_running()
        action = parse_agent_log()

        # Fast stop: switched away from llama-qwen
        if running and action == "stop" and not any_session_uses_llama_qwen():
            stop_server(); continue

        # Fast start: switched to llama-qwen
        if not running and action == "start":
            start_server(); continue

        # App closed: Hermes.exe no longer running
        if running and not hermes_app_running() and not any_session_uses_llama_qwen():
            stop_server(); continue

        # State.db fallback: another session needs it
        if not running and any_session_uses_llama_qwen():
            start_server(); continue

        # Idle timeout safety net (no session needs it)
        if running and idle_too_long() and not any_session_uses_llama_qwen():
            stop_server()
```

#### Watchdog systemd service: `~/.config/systemd/user/llama-watchdog.service`

```ini
[Unit]
Description=llama-server session-aware watchdog

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/vibrationall/scripts/llama_session_watchdog.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

## Hermes provider config

```yaml
# In ~/AppData/Local/hermes/config.yaml
providers:
  llama-qwen:
    name: llama.cpp Qwen3.6 (WSL)
    base_url: http://<WSL2-IP>:8080/v1
    api_key: not-needed
    model: qwen3.6-heretic
```

Find WSL2 IP: `wsl ip addr show eth0 | grep "inet "`

## Commands

```bash
# Start server
wsl systemctl --user start llama-server.service

# Stop server (free VRAM)
wsl systemctl --user stop llama-server.service

# Check status
wsl systemctl --user status llama-server.service --no-pager | head -5

# Watchdog logs
wsl journalctl --user -u llama-watchdog.service -n 20 --no-pager

# Test watchdog: simulate switch TO llama-qwen (should start)
echo "2026-07-02 18:30:00 INFO agent: Model switched in-place: test (test) -> test (llama-qwen)" >> /mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log

# Test watchdog: simulate switch FROM llama-qwen (should stop if no other session needs it)
echo "2026-07-02 18:30:05 INFO agent: Model switched in-place: test (llama-qwen) -> test (opencode)" >> /mnt/c/Users/MaJor/AppData/Local/hermes/logs/agent.log
```

## Pitfalls

- **state.db does NOT track `/model` switches mid-TUI-session** — it only updates when sessions are created. The `model` column stays at the original provider. Use agent.log tailing for fast TUI detection.
- **SQLite over /mnt/c/ via 9p** — file locking is broken. Always copy the DB to `/tmp` before querying.
- **Cold load time** — 17GB GGUF takes ~36s on WSL2. Hermes `gateway_timeout: 1800` (30 min) is plenty.
- **LD_LIBRARY_PATH is critical in systemd** — systemd user services don't inherit environment. Must include TurboQuant build dir + CUDA libs.
- **exit.target must be masked** — prevents WSL2 session cycling: `systemctl --user mask exit.target --now`. Without this, WSL tears down all user services every ~60s.
- **Hermes has NO `on_provider_change` plugin hook** — `VALID_HOOKS` lacks provider-switch events. Workaround is tailing agent.log, which works reliably.
- **The watchdog uses file-position tracking** — if the log is rotated while the watchdog is stopped, `_last_pos` resets to 0, causing a re-read of old entries. This is benign (only triggers extra start attempts).
- **Multiple sessions** — if any active session in state.db has `provider: llama-qwen`, server stays up.
