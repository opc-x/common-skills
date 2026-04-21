SSH into a server via JumpServer bastion to fetch and search logs. **ALWAYS use this skill** whenever the user mentions: viewing logs, reading server logs, checking errors on a test/prod server, connecting via JumpServer, querying a remote server, "上跳板机", "查日志", "看日志", "服务器日志", "测试环境日志", "看报错", "上服务器", "跳板机", "ssh连服务器". Do NOT attempt paramiko, REST API, curl, or Bearer tokens — use this skill directly.

## Usage
/ssh-log [server] [keyword] [options]

- server: alias (topicinfo) or bare IP. Default: topicinfo (18.166.181.114)
- keyword: grep term to filter logs (optional)
- options: --tail N (lines), --log-path /custom/path, --cmd "raw command"

## How to execute

**Script:** `C:/Users/DEVTrump/plugins/jumpserver-ssh/ssh_exec.py`
**Python:** `C:\Users\DEVTrump\AppData\Local\Programs\Python\Python312\python.exe`
**Config:** `C:/Users/DEVTrump/plugins/jumpserver-ssh/config.json`

### Step 1 — Run directly (no OTP needed, uses session + token cache)

```powershell
powershell.exe -Command "& 'C:\Users\DEVTrump\AppData\Local\Programs\Python\Python312\python.exe' 'C:/Users/DEVTrump/plugins/jumpserver-ssh/ssh_exec.py' --server <server> --grep '<keyword>' --log-tail 500"
```

Or with a raw command:
```powershell
powershell.exe -Command "& 'C:\Users\DEVTrump\AppData\Local\Programs\Python\Python312\python.exe' 'C:/Users/DEVTrump/plugins/jumpserver-ssh/ssh_exec.py' --server <server> --cmd '<shell_cmd>'"
```

Auth priority (automatic, no user action needed):

1. Cached Bearer token (~7h TTL)
2. `jms_sessionid` from config.json → auto-generates token, no OTP
3. OTP fallback (only if session expired)

### Step 2 — Handle result

- **If output starts with `OTP_REQUIRED`**: both token cache and `jms_sessionid` are expired. Ask user for OTP, then re-run with `--otp <code>`. Also update `jms_sessionid` in config.json by grabbing a fresh one from browser DevTools (Network tab → any request to fhx-jps.servicea6.com → Cookie header → `jms_sessionid=...`).
  Use AskUserQuestion: "请打开手机认证器，输入当前6位验证码："

  ```powershell
  ... --otp <6digit_code>
  ```

- **On success**: display log output directly.
- **`ERROR: Authentication failed`**: credentials wrong, check config.json.

### Step 3 — Clear cache if needed

```powershell
powershell.exe -Command "& 'C:\Users\DEVTrump\AppData\Local\Programs\Python\Python312\python.exe' 'C:/Users/DEVTrump/plugins/jumpserver-ssh/ssh_exec.py' --clear-cache"
```

## Common patterns

```bash
# Search batchReadTopic logs in topicinfo
--server topicinfo --grep "batchReadTopic" --log-tail 200

# Last 100 lines of default log
--server topicinfo --log-tail 100

# Custom server IP with grep
--server 18.166.26.104 --grep "Exception" --log-tail 300

# Raw command
--cmd "grep -A 20 'batchReadTopic.*huat1234' /data/docker/data/api/api-topicinfo15400/logs/server.log | tail -30"

# Find running services
--cmd "docker ps --format 'table {{.Names}}\t{{.Status}}'"
```

## Server aliases (from config.json)
| Alias | IP | Default Log |
|---|---|---|
| topicinfo | 18.166.181.114 | /data/docker/data/api/api-topicinfo15400/logs/server.log |

To add more servers: edit `C:/Users/DEVTrump/plugins/jumpserver-ssh/config.json`.
