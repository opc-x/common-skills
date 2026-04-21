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

### 开始抓包（用户在独立终端执行）

```powershell
C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/Scripts/mitmdump.exe `
  -p 8080 --flow-detail 0 `
  -s C:/Users/DEVTrump/zx-projects/tools/app-intercept/filter.py `
  -w C:/Users/DEVTrump/.claude/captures/session.flow
```

### SSL Pinning 绕过

```powershell
# 列出进程找 PID
C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/Scripts/frida-ps.exe -U

# objection 绕过
C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/Scripts/objection.exe -g <package> explore
# 进入后输入：android sslpinning disable
```

### 导出抓包结果

```powershell
C:/Users/DEVTrump/AppData/Local/Programs/Python/Python312/python.exe -c "
import mitmproxy.io
with open('C:/Users/DEVTrump/.claude/captures/session.flow', 'rb') as f:
    for flow in mitmproxy.io.FlowReader(f).stream():
        ct = flow.response.headers.get('content-type','') if flow.response else ''
        if 'json' in ct or '/api/' in flow.request.path:
            print(flow.request.method, flow.request.pretty_url)
            if flow.response: print(' ->', flow.response.status_code)
            print()
"
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
