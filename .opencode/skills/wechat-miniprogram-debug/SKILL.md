---
name: wechat-miniprogram-debug
description: 微信小程序自动化诊断。AI 直接用 miniprogram-automator API 驱动 DevTools，扫描问题或自动修复。
license: MIT
compatibility: opencode
metadata:
  platform: windows
  tooling: miniprogram-automator
---

# 关键规则

1. **禁止 `node -e` 内联**。改用 **Write 工具写临时 .js 文件 → `node <file>.js`**。原因：PowerShell 中 `$` 被解释为变量，escaping 极难。
2. **短脚本，逐步骤控制**。每个脚本只做一件事（连接、导航、获取 WXML、断开），AI 看到结果后决定下一步。避免把全部逻辑写在一个长脚本里。
3. **修改项目源码不受限**。修复模式下直接 edit `.js`/`.wxml`/`.wxss` 等。
4. `automator.launch()` 遇中文 cliPath 会报错，改用 `cli.bat auto` + `automator.connect()`。
5. **启动 CLI 必须用 Node.js `spawn()`**（非 PowerShell `cmd.exe /c`）：
   ```js
   const { spawn } = require('child_process');
   const proc = spawn('cmd.exe', ['/c', cliPath, 'auto', '--project', projectPath, '--auto-port', '9420'], { stdio: ['ignore', 'pipe', 'pipe'] });
   ```
   原因：`spawn()` 传递完整中文路径给 cmd.exe，CLI 能正确定位 User Data 目录读取登录 token。PowerShell 的 `cmd.exe /c "短路径"` 会导致 CLI 找不到已有登录态。
6. **推荐用 subagent 执行脚本**：写脚本后通过 `task -t general` 让子 agent 执行 `node script.js`。主进程设超时，超时则终止子 agent 并检查输出。脚本避免用 `throw` 做超时逻辑，改用 `reject`。

# 通用流程

## 路径修正
- 自动纠正盘符（C: ↔ E:）
- `Test-Path <项目路径>\app.json` 验证，不存在则提示用户

## SDK 检测
- 检查 `package.json` 和 `node_modules/miniprogram-automator`
- 未安装则 `npm install miniprogram-automator --save-dev`
- 提示用户：**手动开启"服务端口"**（设置 → 安全设置 → CLI/HTTP 调用）

## 清理残留进程（⚠ 关键：避免登录态损坏）

**`Stop-Process -Force` 只有在 DevTools 完全无响应时才使用。强制杀进程会损坏登录 token 缓存，导致后续所有操作报"需要重新登录"。**

推荐优先级：
1. **优雅关闭**（DevTools 仍在运行）：
   ```powershell
   cmd.exe /c "<cli.bat>" quit
   ```
2. **CloseMainWindow**（DevTools 无响应但窗口存在）：
   ```powershell
   Get-Process -Name "wechatdevtools" | ForEach-Object { $_.CloseMainWindow() | Out-Null }
   Start-Sleep -Seconds 3
   ```
3. **最后手段**（确认登录态正常后再 Force）：
   ```powershell
   Get-Process -Name "wechatdevtools","微信开发者工具*" -ErrorAction SilentlyContinue | Stop-Process -Force
   ```
   强制杀进程后，用户必须手动打开 DevTools 重新扫码登录。

## 启动 DevTools（短脚本模式）

不要一个长脚本包办所有。每个步骤写一个独立短脚本，AI 逐步骤控制：

### 步骤模板

**Step A — 启动 CLI（必须用 spawn，不能用 PowerShell cmd.exe /c）：**
```js
// 写到 temp 目录后用 node 执行
const { spawn } = require('child_process');
const CLI_PATH = 'E:\\Program Files (x86)\\Tencent\\微信web开发者工具\\cli.bat';
const PROJECT_PATH = '项目绝对路径';
const proc = spawn('cmd.exe', ['/c', CLI_PATH, 'auto', '--project', PROJECT_PATH, '--auto-port', '9420'], { stdio: ['ignore', 'pipe', 'pipe'] });
let out = '';
proc.stdout.on('data', d => { process.stdout.write(d); out += d; });
proc.stderr.on('data', d => { process.stderr.write(d); out += d; });
const timer = setTimeout(() => { console.log('\nTIMEOUT'); proc.kill(); process.exit(1); }, 60000);
proc.on('exit', code => { clearTimeout(timer); process.exit(out.includes('√ auto') ? 0 : 1); });
```
执行后看输出：必须有 `√ auto` 才成功。若看到 `需要重新登录` → DevTools 未登录，让用户手动打开 DevTools 扫码后重试。

**Step B — 写连接脚本并执行：**
```js
// 用 Write 工具写到 C:\Users\winte\AppData\Local\Temp\opencode\step_b.js
const a = require('miniprogram-automator');
(async () => {
  try {
    const m = await a.connect({ wsEndpoint: 'ws://localhost:9420' });
    const p = await Promise.race([m.currentPage(), new Promise((_, r) => setTimeout(() => r('TIMEOUT'), 8000))]);
    if (p === 'TIMEOUT') { console.log('currentPage超时'); await m.disconnect(); return; }
    console.log('page:', p.path, JSON.stringify(p.query));
    await m.disconnect();
  } catch (e) { console.error('err:', e.message); }
})();
```
执行：`node <temp_path>\step_b.js`

→ AI 看到结果后，决定下一步做什么

**Step C — 导航到目标页面：**
```js
const a = require('miniprogram-automator');
(async () => {
  const m = await a.connect({ wsEndpoint: 'ws://localhost:9420' });
  const msgs = [];
  m.on('console', msg => msgs.push({ type: msg.type, args: msg.args.join(' ') }));
  try {
    await Promise.race([m.switchTab('/pages/index/index'), new Promise((_, r) => setTimeout(r, 15000))]);
    await new Promise(r => setTimeout(r, 2000));
    const p = await m.currentPage();
    const el = await p.$('page');
    const wxml = el ? (await el.outerWxml()).substring(0, 500) : 'no page';
    const d = await p.data();
    console.log('path:', p.path);
    console.log('data:', JSON.stringify(d).substring(0, 300));
    console.log('wxml:', wxml);
    console.log('console:', JSON.stringify(msgs.filter(m => m.type === 'warn' || m.type === 'error')));
  } catch (e) { console.error('nav err:', e.message); }
  await m.disconnect();
})();
```

**Step D — 断开连接：**
```js
const a = require('miniprogram-automator');
(async () => {
  const m = await a.connect({ wsEndpoint: 'ws://localhost:9420' });
  await m.disconnect();
  console.log('disconnected');
})();
```

每步脚本通常 15-30 行，3s-15s 内完成。AI 看到输出后决定下一步，不会傻等。

## 事件监听
```js
const logs = []
miniProgram.on('console', msg => logs.push(msg))
miniProgram.on('exception', err => logs.push(err))
// msg.type: 'log'|'info'|'warn'|'error'|'assert'
// msg.args: array — 连接后尽早注册监听，避免遗漏
```

# 页面遍历（AI 自主决定顺序）

推荐策略：
- **tabBar 页**：`switchTab` → `currentPage()` → `page.$('page')` + `outerWxml()` → `page.data()` → `page.$$(selector)`
- **非 tab 页**：`redirectTo`（防栈溢出）→ 同上分析
- **搜索页**：`page.$('input')` → `.input('test')` → `page.data('searchValue')`
- **表单页**：找 `input`/`textarea` → 填入测试数据
- **滚动页**：找 `scroll-view` → 尝试 touch 手势
- **深度交互**：点击列表项查看详情页

所有导航：`Promise.race([navigate, new Promise((_,r)=>setTimeout(r,15000))])`

注意：`currentPage()` 返回的是 Page 实例，它没有 `outerWxml()` 方法。必须先 `page.$('page')` 获取元素，再调用 `outerWxml()`：
```js
const cp = await miniProgram.currentPage();
const pageEl = await cp.$('page');
const wxml = pageEl ? await pageEl.outerWxml() : '(无 page 元素)';
```

# Scan 模式
1. 遍历页面，分类捕获消息：`noise`（不影响）/ `fixable`（可修）/ `unknown`（需人工）
2. 已知噪音：组件属性类型不匹配、框架日志、组件内部警告、setData 过大/频繁、Loading 频繁、插件日志
3. 已知可修：`http://`→`https://`、`wx.getUserInfo`、`console.log/info/debug`、`wx.show/hideNavigationBarLoading`、引用错误、资源加载失败、脚本异常、路由异常
4. 输出 `diagnose-report.txt`（页面状态 + 错误分类 + 噪音列表）和 `diagnose-fix-suggestions.txt`（文件路径+行号+修改建议+原因）
5. `miniProgram.disconnect()`，然后 `cmd.exe /c "<CLI_PATH>" open --project <项目路径>`

# Fix 模式
1. 扫描同 Scan
2. **先备份**：
   ```powershell
   $backupDir = "<项目路径>_backup_$(Get-Date -Format yyyyMMddHHmmss)"
   New-Item -Type Directory $backupDir
   Copy-Item "<项目路径>\pages","<项目路径>\app.json","<项目路径>\app.js","<项目路径>\app.wxss","<项目路径>\project.config.json" $backupDir -Recurse
   ```
3. **自动修复**：AI 直接 edit 源码。规则：
   - `http://`（非 localhost/127.0.0.1/10.）→ `https://`
   - `console.log(` / `console.info(` / `console.debug(` → 删除整行
   - `wx.showNavigationBarLoading(` / `wx.hideNavigationBarLoading(` → 删除整行
   - 引用错误 → 尝试添加声明
   - **不修**：node_modules、miniprogram_npm、mock
4. 生成 `diagnose-fix-log.txt`（文件路径 + 行号 + 操作 + 修改前/后 + 原因）
5. 断开 + 保持 DevTools

# 技术约束
1. `.bat` 必须 `cmd.exe /c` 包装。中文路径用 `ShortPath` 获取短路径。
2. 非 tab 页用 `redirectTo`（防栈溢出）
3. 所有导航加 `Promise.race` + 15s 超时
4. `page.$()` 无法选 npm 组件标签（如 `t-tabs`），改用 `outerWxml()` + 正则
5. `console` 事件捕获不全，报告需注明
6. 渲染层错误无法捕获，报告需注明
7. 编译错误查 DevTools 日志
8. **`currentPage()` 返回 Page 实例，没有 `outerWxml()`**，必须先 `page.$('page')` 获取元素再调 `outerWxml()`
9. **禁止 `Stop-Process -Force`** 杀 DevTools 进程（会损坏登录态），优先用 `cli.bat quit`
10. **禁止 `node -e` 内联脚本**。用 Write 工具写临时 `.js` 文件后用 `node` 执行。
11. **每步脚本控制在 15-30 行，15s 内完成**。长了就拆。AI 看到输出再决定下一步。

# 故障排查
- CLI 启动失败 → 确认开发者工具已安装、检查 cli.bat 路径
- **需要重新登录(code 10)** → DevTools 未登录或用了错误方式启动。
   - ❌ `cmd.exe /c "短路径\cli.bat auto"`（PowerShell 方式）→ CLI 定位不到 User Data 目录
   - ✅ `spawn('cmd.exe', ['/c', 完整中文路径, 'auto', ...])`（Node.js 方式）→ 正常
   - 若仍失败，让用户手动打开 DevTools 扫码登录，关掉后重试
- 连接超时 → 检查编译错误、端口 9420 被占用、从清理步骤重试。连接后用 `Promise.race` + 8s 超时检测 `currentPage()`，避免无限挂起。
- 页面导航失败 → 检查 app.json 路径、wxml 是否存在
- **PowerShell `$` 符号问题** → `node -e` 中 `page.$('page')` 的 `$` 被 PowerShell 解释为变量。必须用 `.js` 文件运行，不要用 `node -e`。
- WXML 找不到元素 → npm 组件用 `outerWxml()` 正则
- **中文路径问题** → 含中文的 cli.bat 路径在 cmd.exe 中会报错 `'E:\Program' 不是内部或外部命令`。必须用短路径（8.3 格式）：`(New-Object -ComObject Scripting.FileSystemObject).GetFile("path").ShortPath`
