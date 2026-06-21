---
name: chaoxing-auto-homework
description: 自动完成超星学习通(chaoxing.com)作业。当用户打开学习通作业页面（mooc1.chaoxing.com/mooc-ans/mooc2/work/dowork 或 view），要求帮忙答题、完成作业、做选择题/多选题/简答题时使用此技能。支持读取题目、AI分析答案、自动点击选项、自动填入简答题代码。
---

# 超星学习通自动答题

## 概述

通过 Codex 内置浏览器连接学习通作业页面，读取题目内容，由 AI 分析正确答案，然后自动完成作答。支持单选题、多选题和简答题（含代码填空）。

## Bootstrap（每次新会话执行一次）

**始终从 js_reset 开始**：

```js
// Step 0: 用 mcp__node_repl__js_reset 工具重置内核

// Step 1: 连接浏览器
var browserClient = "file:///" + nodeRepl.homeDir.replace(/\\/g, "/") + "/.codex/plugins/cache/openai-bundled/browser/latest/scripts/browser-client.mjs";
const { setupBrowserRuntime } = await import(browserClient);
await setupBrowserRuntime({ globals: globalThis });
var br = await agent.browsers.get("iab");
var ct = await br.tabs.selected();
```

**注意**：`ct` 是当前 tab，`br` 是浏览器。使用 `var` 声明以便内核重置后可重新声明。tab id 通常是 "1"（用 `br.tabs.list()` 确认）。

---

## 一、多选题 (Multiple Choice)

### 页面结构

学习通使用自定义 `<div>` checkbox，不是标准 `<input type="checkbox">`：

```html
<div class="clearfix answerBg workTextWrap" role="checkbox"
     onclick="addMultipleChoice(this);" aria-checked="false">
  <span data="A" class="num_option_dx fl">A</span>
  <div class="fl answer_p"><p>选项内容</p></div>
</div>
```

- 选项定位：`[role="checkbox"]` 内 `span[data]` 获取字母
- 选中状态：`aria-checked="true"` 表示已选
- 隐藏答案输入：`input[id^="answer"]` 存储已选字母

### 关键约束

- `page.evaluate` 沙箱极其受限：无法访问 `addMultipleChoice`、`$` (jQuery)、`MouseEvent`、`Event`
- 不能从 evaluate 注入脚本或操作 DOM
- **必须串行点击**，并行点击 ~80% 静默失败

### 工作流程

```js
// 1. 定义答案
var answers = {
  1: ["A","C"], 2: ["A","B","C"], /* ... */
};

// 2. 清除已有选择
var checkedIndices = await ct.playwright.evaluate(() => {
  var boxes = document.querySelectorAll('[role="checkbox"]');
  var checked = [];
  for (var i = 0; i < boxes.length; i++) {
    if (boxes[i].getAttribute('aria-checked') === 'true') checked.push(i);
  }
  return checked;
});

for (var i = 0; i < checkedIndices.length; i++) {
  await ct.playwright.getByRole("checkbox").nth(checkedIndices[i]).click({ timeoutMs: 2000 });
}

// 3. 计算目标索引
var optMap = {A:0, B:1, C:2, D:3};
var toClick = [];
for (var q = 1; q <= 100; q++) {
  var opts = answers[q];
  for (var i = 0; i < opts.length; i++) {
    toClick.push((q - 1) * 4 + optMap[opts[i]]);
  }
}

// 4. 串行点击（绝对不要并行化）
for (var i = 0; i < toClick.length; i++) {
  try {
    await ct.playwright.getByRole("checkbox").nth(toClick[i]).click({ timeoutMs: 5000 });
  } catch(e) {}
  if ((i + 1) % 20 === 0) nodeRepl.write("Progress: " + (i + 1) + "/" + toClick.length);
}
```

### 验证

```js
var check = await ct.playwright.evaluate(() => {
  var qs = document.querySelectorAll('.questionLi');
  var unselected = [];
  for (var qi = 0; qi < qs.length; qi++) {
    var hasChecked = false;
    qs[qi].querySelectorAll('[role="checkbox"]').forEach(o => {
      if (o.getAttribute('aria-checked') === 'true') hasChecked = true;
    });
    if (!hasChecked) unselected.push(qi + 1);
  }
  return unselected;
});
nodeRepl.write("Unselected: " + JSON.stringify(unselected));
```

---

## 二、简答题 (Short Answer) — 核心重点

简答题使用百度 **UEditor**（contenteditable 富文本编辑器），极其抗拒自动化。以下方法是经过实战验证的唯一可行方案。

### 必须使用 `dom_cua` API

**不要用 `playwright.fill()`、`playwright.press()` 或 `tab.cua`**。简答题只能用 `ct.dom_cua` API：

| API | 用途 |
|-----|------|
| `ct.dom_cua.get_visible_dom()` | 获取可见 DOM，找编辑器 body 的 `node_id` |
| `ct.dom_cua.click({ node_id })` | 点击获取焦点 |
| `ct.dom_cua.type({ text })` | **输入文本（`\n` → `<br/>` 换行）** |
| `ct.dom_cua.keypress({ keys })` | 按键操作 |
| `ct.dom_cua.scroll({ ... })` | 滚动页面 |

### 步骤 1：准备答案文件

**先将每道题的 Python 代码保存到本地文件**。`dom_cua.type()` 操作很慢且可能超时，有文件可重复读取很重要。

**关键**：Python 注释行 `#` 会被 UEditor 当作 Markdown 标题吞掉！所有以 `#` 开头的行必须前面加零宽空格 `\u200B`：

```js
var fs = await import('node:fs');
var ZWSP = '\u200B';
var content = fs.readFileSync('C:\\path\\q1.py', 'utf-8');
// 给所有 # 开头行加保护
var lines = content.split('\n');
for (var i = 0; i < lines.length; i++) {
  if (lines[i].match(/^#/)) lines[i] = ZWSP + lines[i];
}
content = lines.join('\n');
```

建议将保护后的内容写回文件，方便 `dom_cua.type()` 直接读取。

### 步骤 2：获取 DOM 定位编辑器

```js
// 确保浏览器可见（dom_cua 需要）
var vis = await br.capabilities.get("visibility");
await vis.set(true);

// 滚动到顶部
await ct.playwright.evaluate(() => window.scrollTo(0, 0));
await new Promise(r => setTimeout(r, 2000));

var dom = await ct.dom_cua.get_visible_dom();
```

在输出中查找 `<body node_id=XXX contenteditable="true">` — 每个对应的 node_id 就是一个编辑器。

**node_id 是动态的**，每次获取 DOM 都不同，绝对不能缓存跨调用使用。

### 步骤 3：填入答案

每道题的流程：

```js
// 3a. 确保编辑器在视口内
await ct.playwright.evaluate(() => {
  var q = document.querySelector('#questionID');
  if (q) q.scrollIntoView({ block: 'center' });
});
await new Promise(r => setTimeout(r, 1500));

// 3b. 用 Playwright 点击编辑器获取焦点
var body = ct.playwright.frameLocator('#questionID iframe').locator('body');
await body.click();
await new Promise(r => setTimeout(r, 500));

// 3c. 用 dom_cua 输入答案
var fs = await import('node:fs');
var answer = fs.readFileSync('C:\\path\\q1_protected.py', 'utf-8');
await ct.dom_cua.type({ text: answer });

// 3d. 等 UEditor 同步（textarea 更新需要时间）
await new Promise(r => setTimeout(r, 3000));
```

### 步骤 4：验证

用 Playwright 检查 textarea（UEditor 同步目标，提交时发送此值）：

```js
var taVal = await ct.playwright.evaluate(() => {
  var ta = document.querySelector('#questionID textarea');
  return ta ? ta.value : 'NO';
});

// 关键检查：是否有 <br/> 标签（表示换行成功）
var hasBr = taVal.indexOf('<br/>') > -1;
var hasContent = taVal.length > 500;
nodeRepl.write('Q1: hasBr=' + hasBr + ', len=' + taVal.length);
```

**不要信任 `get_visible_dom()` 来验证** — 它可能不反映 iframe 内容的实时变化。验证 textarea 值才是最可靠的。

### 步骤 5：逐题滚动

填入一题后滚动到下一题：

```js
await ct.playwright.evaluate(() => {
  var next = document.querySelector('#nextQuestionID');
  if (next) next.scrollIntoView({ block: 'center' });
});
await new Promise(r => setTimeout(r, 1500));
```

---

## 三、关键陷阱与解决方案

### 1. 编辑器清空不了

**现象**：`Ctrl+A`、`Ctrl+Home`+`Ctrl+Shift+End` 等全选快捷键被 UEditor 完全屏蔽。

**解决**：在页面刚加载、编辑器为空时立即填入。如果已有旧内容：
- 方案 A：追加新内容到末尾（`Ctrl+End` + 两个 Enter + type）
- 方案 B：用户手动在浏览器里 `Ctrl+A` + `Delete` 清空

### 2. `#` 注释被吃掉

**现象**：Python 代码中 `# 这是一行注释` 在编辑器中变成 `这是一行注释`。

**原因**：UEditor 把行首 `# ` 当作 Markdown 标题（H1）自动格式化。

**解决**：在所有 `#` 开头的行前面加零宽空格 `\u200B`（`\u200B# 注释`）。零宽空格不可见但阻止了 Markdown 检测。Python 解释器会忽略它（因为它在注释里）。

### 3. `dom_cua.type()` 换行机制

- `\n` 在 type 的文本中 → UEditor 转换为 `<br/>`（软换行）
- 连续两个 `\n`（空行）→ UEditor 创建新 `<p>`（段落分隔）
- 每个 `<p>` 开头的 `#` 会被吞 → 确认用 `\u200B` 保护

### 4. Playwright evaluate 沙箱限制

沙箱中以下全部不可用：
- `document.createElement` / `document.createEvent`
- `window.location.reload / assign`
- `XMLHttpRequest` / `fetch`
- `atob` / `btoa`
- 访问 iframe 的 `contentDocument` / `contentWindow`
- 修改 textarea 的 `.value`（只读 getter）
- jQuery `$`、`UE` 全局对象

**只能用 `ct.playwright.locator()`、`ct.playwright.evaluate()`（受限版）、`ct.dom_cua.*`**

### 5. 节点 ID 动态变化

`get_visible_dom()` 返回的 `node_id` 每次调用都可能不同。每个操作之前重新获取 DOM。

### 6. 超时问题

| 操作 | 建议 timeout_ms |
|------|----------------|
| `dom_cua.get_visible_dom()` | 15,000 |
| `dom_cua.click()` | 15,000 |
| `dom_cua.type()`（整段代码）| 30,000-60,000 |
| `dom_cua.scroll()` | 15,000 |
| 多选题批量（100题）| 300,000-600,000 |
| 简答题全部（3题）| 每步 60,000，总计 ~3-5 分钟 |

Statsig 网络错误可忽略（无关遥测）。

### 7. 浏览器必须可见

`dom_cua` 需要浏览器窗口可见才能工作：
```js
var vis = await br.capabilities.get("visibility");
await vis.set(true);
```

---

## 四、答案页面复盘

作业提交后（URL 含 `/mooc-ans/mooc2/work/view`），快速找错题：

```js
var errors = await ct.playwright.evaluate(() => {
  var items = document.querySelectorAll('.questionLi');
  var wrong = [];
  for (var i = 0; i < items.length; i++) {
    var text = items[i].innerText;
    var myMatch = text.match(/我的答案:(\S+)/);
    var correctMatch = text.match(/正确答案:(\S+)/);
    if (myMatch && correctMatch && myMatch[1] !== correctMatch[1]) {
      wrong.push({ q: i + 1, my: myMatch[1], correct: correctMatch[1] });
    }
  }
  return wrong;
});
```

---

## 五、不要自动提交

不要自动点击提交按钮。用户需自行检查并手动提交。

## 平台信息

- 域名：mooc1.chaoxing.com
- 作答页：/mooc-ans/mooc2/work/dowork
- 答案页：/mooc-ans/mooc2/work/view
- 需先登录，作业在截止时间窗口内
- 用 `br.tabs.list()` 确认 tab id
