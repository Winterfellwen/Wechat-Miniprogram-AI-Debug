# miniprogram-automator API 完整参考

基于微信官方文档整理，供 AI 在诊断/修复时直接查阅。

---

## 1. Automator (入口)

### `automator.connect(options)` → `Promise<MiniProgram>`
连接已运行的 DevTools。
```js
const m = await automator.connect({ wsEndpoint: 'ws://localhost:9420' });
```

### `automator.launch(options)` → `Promise<MiniProgram>`
启动并连接（中文路径会报错，推荐先 `cli.bat auto` 再 `connect`）。
```js
const m = await automator.launch({
  cliPath: 'path/to/cli.bat', projectPath: 'path/to/project',
  port: 9420, timeout: 60000,
  account: 'openid',     // 可选：多账号调试
  ticket: 'login_ticket', // 可选：远程登录票据
  projectConfig: { setting: { autoAudits: true } }
});
```

---

## 2. MiniProgram (小程序实例)

### 导航
| 方法 | 说明 | tabBar? |
|------|------|---------|
| `navigateTo(url)` → `Page` | 保留当前页跳转 | 不支持 tab |
| `redirectTo(url)` → `Page` | 关闭当前页跳转 | 不支持 tab |
| `navigateBack()` → `Page` | 返回上一页 | — |
| `reLaunch(url)` → `Page` | 关闭所有页跳转 | 全部支持 |
| `switchTab(url)` → `Page` | 跳转 tabBar 页面 | **只能用于 tab** |
| `currentPage()` → `Page` | 获取当前页面 | — |
| `pageStack()` → `Page[]` | 获取页面堆栈 | — |

### 数据与拦截
| 方法 | 说明 |
|------|------|
| `systemInfo()` → `Object` | 获取系统信息（`wx.getSystemInfo`） |
| `callWxMethod(method, ...args)` → `any` | 调用 wx 对象方法（如 setStorage） |
| `mockWxMethod(method, result\|fn, ...args)` | 覆盖 wx 方法返回结果。fn 内可用 `this.origin()` 调用原方法 |
| `restoreWxMethod(method)` | 恢复被 mock 的 wx 方法 |
| `evaluate(fn, ...args)` → `any` | 往 AppService 注入代码并执行（fn 不能使用闭包） |
| `pageScrollTo(scrollTop)` | 滚动页面 |
| `screenshot({path?})` → `string\|void` | 截图，返回 base64 或保存到 path |
| `exposeFunction(name, fn)` | 在 AppService 暴露全局方法（供小程序侧调用） |
| `testAccounts()` → `Account[]` | 获取多账号调试中已添加的用户列表 |
| `stopAudits({path?})` → `Object` | 停止体验评分并获取报告（需开启 autoAudits） |
| `getTicket()` → `{ticket, expiredTime}` | 获取登录票据 |
| `setTicket(ticket)` | 设置登录票据（更新失效票据） |
| `refreshTicket()` | 刷新登录票据（重置过期时间为2小时） |
| `remote(auto?)` | 开启真机调试（auto=true 自动调起小程序） |
| `disconnect()` | 断开连接 |
| `close()` | 断开连接并关闭项目窗口 |

### mockWxMethod 示例
```js
// 直接指定返回
await m.mockWxMethod('showModal', { confirm: true, cancel: false });
// 用函数动态处理
await m.mockWxMethod('getStorageSync', function(key, defVal) {
  if (key === 'name') return 'test';
  return defVal;
}, 'unknown');
// 包装原始方法
await m.mockWxMethod('getSystemInfo', function(obj, platform) {
  return new Promise(resolve => {
    this.origin({ success(res) { res.platform = platform; resolve(res); } });
  });
}, 'test');
```

### evaluate 示例
```js
// 注入代码片段
const info = await m.evaluate(() => wx.getSystemInfoSync());
const hasLogin = await m.evaluate(() => getApp().globalData.hasLogin);
await m.evaluate(key => wx.setStorageSync(key, 'test'), 'myKey');
```

### 事件监听
```js
m.on('console', msg => {
  // msg.type: 'log'|'info'|'warn'|'error'|'assert'
  // msg.args: array<any>
});
m.on('exception', err => {
  // err.message, err.stack
});
```

---

## 3. Page (页面实例)

### 属性
- `page.path` — 页面路径
- `page.query` — 页面参数

### 方法
| 方法 | 说明 |
|------|------|
| `$(selector)` → `Element` | 获取第一个匹配元素（CSS 选择器，不支持自定义组件标签名） |
| `$$(selector)` → `Element[]` | 获取所有匹配元素 |
| `waitFor(condition)` | 等待条件成立：ms / 选择器出现 / 函数返真 |
| `data(path?)` → `any` | 获取页面 data |
| `setData(data)` | 设置页面 data |
| `size()` → `{width, height}` | 页面可滚动宽高 |
| `scrollTop()` → `number` | 获取滚动位置 |
| `callMethod(method, ...args)` → `any` | 调用页面方法（如 `onShareAppMessage`） |

### waitFor 示例
```js
await page.waitFor(5000);                     // 等5秒
await page.waitFor('picker');                 // 等 picker 元素出现
await page.waitFor(async () => (await page.$$('picker')).length > 5);
```

---

## 4. Element (元素实例)

### 属性
- `element.tagName` — 标签名（小写）

### 查询
| 方法 | 说明 |
|------|------|
| `$(selector)` → `Element` | 在元素范围内查找 |
| `$$(selector)` → `Element[]` | 在元素范围内查找全部 |

### 读取
| 方法 | 说明 |
|------|------|
| `size()` → `{width, height}` | 元素宽高 |
| `offset()` → `{left, top}` | 元素绝对位置（左上角相对于页面原点） |
| `text()` → `string` | 元素文本 |
| `attribute(name)` → `string` | 标签特性值（如 src、class） |
| `property(name)` → `any` | 元素属性值（如 input 的 value），与 attribute 不同：返回类型不限，能获取组件属性值 |
| `wxml()` → `string` | 元素内部 WXML |
| `outerWxml()` → `string` | 包含元素本身的 WXML |
| `value()` → `string` | 表单元素的值 |
| `style(name)` → `string` | 计算样式值（如 'color' → 'rgb(136, 136, 136)'） |

### 交互
| 方法 | 说明 |
|------|------|
| `tap()` | 点击 |
| `longpress()` | 长按 |
| `touchstart(options)` | 手指触摸（含 touches/changedTouches） |
| `touchmove(options)` | 手指移动 |
| `touchend(options)` | 手指离开 |
| `trigger(type, detail?)` | 触发事件（不会触发 tap 等用户操作事件，适用于 picker 的 change 等） |
| `input(value)` | 输入文本（仅 input/textarea） |

### 组件专用方法
| 方法 | 适用组件 | 说明 |
|------|----------|------|
| `callMethod(method, ...args)` | 自定义组件 | 调用组件方法 |
| `data(path?)` | 自定义组件 | 获取组件 data |
| `setData(data)` | 自定义组件 | 设置组件 data |
| `callContextMethod(method, ...args)` | video（需 id） | 调用 Context 方法（如 play） |
| `scrollWidth()` | scroll-view | 滚动宽度 |
| `scrollHeight()` | scroll-view | 滚动高度 |
| `scrollTo(x, y)` | scroll-view | 滚动到位置 |
| `swipeTo(index)` | swiper | 滑动到指定滑块 |
| `moveTo(x, y)` | movable-view | 移动视图容器 |
| `slideTo(value)` | slider | 滑动到指定数值 |

---

## 5. 常用模式

### 连接 + 断开
```js
const a = require('miniprogram-automator');
(async () => {
  const m = await a.connect({ wsEndpoint: 'ws://localhost:9420' });
  // ... work ...
  await m.disconnect();
})();
```

### 页面导航 + 元素检测
```js
// tab 页
await m.switchTab('/pages/home/index');
// 非 tab 页
await m.redirectTo('/pages/detail/index');

const p = await m.currentPage();
const el = await p.$('.some-class');
if (el) {
  const wxml = await el.outerWxml();
  const txt = await el.text();
}
const data = await p.data('someField');
```

### WXML 扫描（检测自定义组件）
```js
const pageEl = await p.$('page');
if (pageEl) {
  const wxml = await pageEl.outerWxml();
  const hasComponent = /\bt-tabs\b/.test(wxml);
}
```
注意：`currentPage()` 返回的 Page 实例**没有** `outerWxml()`。必须 `p.$('page')` 获取元素再调 `outerWxml()`。

### 伪造请求结果
```js
const mockData = [{ rule: 'testRequest', result: { data: 'test', statusCode: 200 } }];
await m.mockWxMethod('request', function(obj, data) {
  for (const item of data) {
    if (new RegExp(item.rule).test(obj.url)) return item.result;
  }
  return new Promise(resolve => { obj.success = res => resolve(res); this.origin(obj); });
}, mockData);
```

### 错误处理 + 超时保护
```js
// 所有导航加超时
await Promise.race([
  m.switchTab('/pages/index/index'),
  new Promise((_, reject) => setTimeout(() => reject(new Error('超时')), 15000))
]);
// currentPage 加超时
const p = await Promise.race([
  m.currentPage(),
  new Promise((_, reject) => setTimeout(() => reject(new Error('currentPage超时')), 8000))
]);
```

---

## 6. 注意事项

1. **`.bat` 必须 `cmd.exe /c` 包装**，推荐用 `spawn('cmd.exe', ['/c', ...])`（完整中文路径）
2. **非 tab 页**用 `redirectTo` 而非 `navigateTo`，避免页面栈溢出
3. **`switchTab`/`redirectTo`/`navigateTo`** 必须加 `Promise.race` + 15s 超时
4. **`page.$()` 不支持自定义组件标签名**（如 `t-tabs`），用 `outerWxml()` + 正则
5. **`console` 事件捕获不完全**：框架级消息（`[system]`、`[Perf]`）不可达
6. **渲染层错误无法捕获**：WXML/WXSS/数据绑定异常属于框架内部
7. **编译错误**查 DevTools 日志
8. **`evaluate()` 的 fn 不能使用闭包**，所有依赖变量必须通过参数传入
9. **`mockWxMethod()` 的 fn 内 `this.origin` 指向原始方法**
