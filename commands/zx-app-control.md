# zx-app-control — Android App 自动化控制

通过 ADB 控制 Android 模拟器，配合 mitmproxy 抓包。

## 使用方式

```
/zx-app-control <动作描述>
```

## 示例

```
/zx-app-control 查看当前连接的设备
/zx-app-control dump当前页面所有按钮
/zx-app-control 打开球盟 app，进红单Tab，点第一条推单
/zx-app-control 截图
/zx-app-control 开始抓包
/zx-app-control 停止抓包并导出结果
```

## 工具路径

- **ADB**: `C:/Users/DEVTrump/AppData/Local/Microsoft/WinGet/Packages/Google.PlatformTools_Microsoft.Winget.Source_8wekyb3d8bbwe/platform-tools/adb.exe`（重启 shell 后可直接用 `adb`）
- **Python**: `C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/python.exe`
- **mitmdump**: `C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/Scripts/mitmdump.exe`
- **frida**: `C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/Scripts/frida.exe`
- **objection**: `C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/Scripts/objection.exe`
- **filter 脚本**: `C:/Users/DEVTrump/zx-projects/tools/app-intercept/filter.py`
- **抓包/截图输出目录**: `C:/Users/DEVTrump/.claude/captures/`
- **mitmproxy 证书**: `C:/Users/DEVTrump/zx-projects/capture/mitmproxy-ca-cert.crt`

## 已知 APP 信息

### 球盟会（qm）
- **包名**: `com.o763r.d68.rd0hy`
- **启动 Activity**: `com.gsmc.php.yahu.ui.module.app.splash.SplashActivity`
- **测试账号**: `qmtest0408` / `Test123456`
- **用户ID**: `514733`
- **friend 域名**: `gopt777.92friend.com:5656`
- **接口文档**: `C:/Users/DEVTrump/zx-projects/docs/球盟会-红单接口文档.md`
- **friend 服务 product 值**: `at`（抓包明文验证，app requestData 里传的是 `"product":"at"`，不是 `"qm"`）

### 球亿（qy）
- **包名**: `com.nj0sh.qg2dz.u0ao2`
- **APK**: `https://qy.supercndl888.com:12828/Android/apk/remote/qyh.apk`

## 执行步骤

### Step 1: 检查设备

```powershell
adb devices
```

没有设备 → 提示用户先启动模拟器。

### Step 2: 获取 UI 树（定位元素）

```powershell
adb shell uiautomator dump /sdcard/ui.xml
adb pull /sdcard/ui.xml C:/Users/DEVTrump/.claude/captures/ui.xml
```

解析 `ui.xml` 找匹配节点，bounds `[left,top][right,bottom]`，中心点 = `((left+right)/2, (top+bottom)/2)`。

### Step 3: 执行操作

```powershell
adb shell input tap <x> <y>                          # 点击
adb shell input swipe 540 1200 540 400 300           # 上滑
adb shell input text "内容"                           # 输入
adb shell input keyevent 4                           # 返回键
adb shell monkey -p <package> 1                      # 启动 app
adb shell am start -n <package>/.<Activity>          # 跳转页面
```

### Step 4: 截图确认

```powershell
adb shell screencap /sdcard/screen.png
adb pull /sdcard/screen.png C:/Users/DEVTrump/.claude/captures/screen.png
```

读取截图，确认操作结果。

### Step 5: 报告抓包结果

mitmproxy 运行中时，列出本次操作触发的接口（方法+URL+关键参数+响应状态）。

---

## 抓包命令

### 正确抓包流程（WSL2 + USB 连接真机）

**关键坑：不要用 PowerShell 或 Windows 命令，全程用 WSL Python 调 infra-mcp core 模块。**

```python
# WSL 环境，工作目录 /home/devtrump/gh-projects/infra-mcp/app-intercept
from core import adb, capture
from pathlib import Path
import time

# Step 1: 确认设备
print(adb.devices())  # 应有 4DXGP7XGNNT8JZE6

# Step 2: 设置 adb reverse（USB 连接时必须，否则手机代理到 10.255.255.254 不通）
import subprocess
ADB = "/mnt/c/Users/DEVTrump/AppData/Local/Microsoft/WinGet/Packages/Google.PlatformTools_Microsoft.Winget.Source_8wekyb3d8bbwe/platform-tools/adb.exe"
subprocess.run([ADB, "-s", "4DXGP7XGNNT8JZE6", "reverse", "tcp:8080", "tcp:8080"])

# Step 3: 启动 mitmdump
flow = Path('/home/devtrump/.claude/captures/session_xxx.flow')
pid = capture.start(flow_path=flow, port=8080)

# Step 4: 设置手机代理（127.0.0.1 走 adb reverse）
adb.shell("settings put global http_proxy 127.0.0.1:8080")

# Step 5: 操作 app 触发请求
adb.launch("com.o763r.d68.rd0hy", None)
time.sleep(8)
# ...界面操作...

# Step 6: 停止抓包，清除代理
capture.stop()
adb.shell("settings put global http_proxy :0")
```

**已知坑和排查手册：**

| 现象 | 原因 | 解决 |
|------|------|------|
| flow 文件为 0 字节 / 几乎没流量 | **手机代理没配好**，最常见坑。mitmdump 在跑但手机流量没经过它 | 先 `adb reverse tcp:8080 tcp:8080`，再 `adb.shell("settings put global http_proxy 127.0.0.1:8080")`，**USB 连接必须走这套，不能用机器 IP** |
| 截图/dump UI 报 cannot create file | `adb.exe` 是 Windows 进程，不能写 `/home/...` Linux 路径 | 已在 `core/adb.py` 中用 `wslpath -w` 自动转 UNC 路径，无需手动处理 |
| filter.py not found | `C:/Users/DEVTrump/zx-projects/` 和 WSL `/home/devtrump/zx-projects/` 是两个不同目录 | 弃用 filter.py，全程用 core 模块，不传 `-s` 参数给 mitmdump |
| 抓到全是加密/乱码 | mitmproxy 证书未装系统级，或 SSL Pinning | 先检查证书；SSL Pinning 用 objection 绕过 |

**每次抓包前必查：**
```python
from core import adb
# 1. 确认代理设置正确
print(adb.shell("settings get global http_proxy"))  # 应为 127.0.0.1:8080
# 2. 确认 adb reverse 在
import subprocess
ADB = "/mnt/c/Users/DEVTrump/AppData/Local/Microsoft/WinGet/Packages/Google.PlatformTools_Microsoft.Winget.Source_8wekyb3d8bbwe/platform-tools/adb.exe"
subprocess.run([ADB, "-s", "4DXGP7XGNNT8JZE6", "reverse", "--list"])  # 应有 tcp:8080 tcp:8080
```

### SSL Pinning 绕过

```python
# 列出进程找 PID（需要 frida）
import subprocess
subprocess.run(["frida-ps", "-U"])

# objection 绕过
subprocess.run(["objection", "-g", "<package>", "explore"])
# 进入后输入：android sslpinning disable
```

### 导出抓包结果（含 requestData 自动解密）

friend 服务接口的 `requestData` 参数是 AES-128-CBC 加密的，导出时自动解密明文。

```python
# 保存为临时脚本执行，或直接用 python3 -c 运行
import mitmproxy.io, urllib.parse, base64, json
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

KEY = b'smkldospdosldaaa'
IV  = b'0000000000000000'

def decrypt_request_data(encoded):
    try:
        raw = base64.b64decode(urllib.parse.unquote(encoded))
        cipher = Cipher(algorithms.AES(KEY), modes.CBC(IV), backend=default_backend())
        dec = cipher.decryptor()
        plain = dec.update(raw) + dec.finalize()
        # 去掉 PKCS5 padding
        pad = plain[-1]
        plain = plain[:-pad]
        return json.loads(plain.decode('utf-8'))
    except Exception as e:
        return f'[解密失败: {e}]'

with open('C:/Users/DEVTrump/.claude/captures/session.flow', 'rb') as f:
    for flow in mitmproxy.io.FlowReader(f).stream():
        if not flow.response:
            continue
        url = flow.request.pretty_url
        if '/api/' not in url and 'friend' not in url:
            continue
        print(f'\n{flow.request.method} {url}')
        # 解密 requestData
        body = flow.request.get_text(strict=False) or ''
        for part in body.split('&'):
            if part.startswith('requestData='):
                encoded = part[len('requestData='):]
                plain = decrypt_request_data(encoded)
                print(f'  requestData(明文): {json.dumps(plain, ensure_ascii=False)}')
        print(f'  -> {flow.response.status_code}')
        try:
            resp = json.loads(flow.response.get_text())
            print(f'  响应: {json.dumps(resp, ensure_ascii=False)[:300]}')
        except:
            pass
```

**直接命令行版（快速用）：**
```powershell
C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/python.exe C:/Users/DEVTrump/zx-projects/tools/app-intercept/decode_flow.py
```

---

## 常见问题

| 问题 | 解法 |
|------|------|
| `adb devices` 空列表 | 模拟器未开启或未开 USB 调试 |
| 抓包全是加密内容 | mitmproxy 证书未装系统级，或有 SSL Pinning |
| SSL Pinning 绕过后仍抓不到 | 用 Magisk + JustTrustMe 模块 |
| 坐标点击没反应 | 分辨率不对，先 dump UI 树取精确坐标 |
| frida 连不上模拟器 | 需要 root 权限 |
