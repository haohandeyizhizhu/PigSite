# PigSite — 项目长期记忆

## 项目是什么
一个**单屏交互小玩具**：中间一只卡通小猪，靠喂食 / 摸摸 / 睡觉维持它的饱食度、心情、体重。
唯一产物是 `index.html`。出现过又废弃的形态：v0.1 营销式示例展示页、v0.2 卡通贴纸风展示页（用户明确否掉，嫌"元素太多、不喜欢这个风格"）。

## 硬性约定
- **单文件、零外部依赖**。不引 CDN、不引字体、不用框架、不放位图。图标/形象一律内联 SVG（favicon 用 data URI，`#` 写成 `%23`）。
- **一屏不滚动**：`100dvh` 高度 + flex 三段式（状态 / 舞台 / 按钮），不出现导航栏、页脚、栏目列表。
- **视觉克制**：扁平、细描边（2.5）、暖色系；**不做**粗描边、0 模糊度硬投影、波点背景、糖果色这类"贴纸感"元素。
- 文字用暖棕紫 `#3A3238`，不要冷灰（冷灰会让页面变成"办公软件"）。

## 用户的工作方式偏好
- **先给方案再动手**。要改动较大的视觉或结构时，先出方案 + 布置示意（哪怕只是截图/线框图），等他点头再写码。他会说"你先给出方案让我看，不要急着写"。
- 授权范围内可自行拍板，别反复追问细节（"三个问题你自己决定即可"）。
- 讨厌展示型排版和元素堆砌，要"做减法"。

## 本机环境坑（很重要）
- **Bash 工具在这台机器上是坏的**：`ls` / `mkdir` / `dirname` 全部 `command not found`（PortableGit 的 shell-runtime shim 有问题）。一律用 **PowerShell** 工具，文件操作用 Read / Write / Edit / Glob / Grep。
- `Write` 覆盖已存在文件**前必须先 `Read`** 一次，否则报 "File has not been read yet"。
- PowerShell 抓子进程 stdout 会**按 GBK 解码**，中文必然乱码。判断文件编码要看 `ReadAllBytes` 的原始字节，或在页面里传 `charCodeAt` 数组出来比对。
- 可用浏览器：`C:\Program Files\Google\Chrome\Application\chrome.exe`（Edge 在 Program Files (x86)）。`agent-browser` 命令**未安装**。
- node：`D:\Work\nodejs\node.exe`（可 `--check` 验 JS 语法）。

## 怎么验证前端（本模型看不见图片）
用 **headless Chrome + 注入探针**，不要指望截图：
1. 探针把结果 JSON 用 `@@...@@` 包住写进 `<pre>`（**别用 `<@>`**，`<` 会被 HTML 序列化成 `&lt;`）。探针插在 `<body>` 后、页面脚本前，才能捕获页面脚本的 `window.onerror`。
2. `& $chrome --headless=new --disable-gpu --no-first-run --user-data-dir=<临时目录> --virtual-time-budget=6000 --window-size=420,860 --dump-dom <file:///...>` ，正则抠 `@@(.*?)@@`。
3. 断言手段：`getBoundingClientRect()` 查溢出/裁切；`getBBox()` 查手画 SVG 的几何关系；`getComputedStyle().display` 查到底哪组 SVG 元素可见；`charCodeAt` 数组验中文。
4. **坑**：`getBBox()` 对 `display:none` 子树返回 0 —— 测隐藏元素前先临时把 `data-face` 切到显示它的那一档。
5. **坑**：headless 里 `requestAnimationFrame` 可能被暂停，别用它来推进游戏逻辑；主循环要用挂钟时间驱动，并额外挂 `setInterval` 心跳。
6. 探针会把页面 DOM 换掉，所以**注入过的临时副本不能用来截图**（截出来是空白/极小体积）。

## SVG 形象的技术要点
- 一个 `<svg>` 靠 `data-face` / `data-mouth` 属性切表情：`.eyes`/`.mouth` 默认 `display:none`，再按 `[data-face="x"]` 显示对应组。**眨眼规则的优先级必须写在表情规则之后**，否则覆盖不了。
- 耳朵、尾巴的旋转**绝对不能用 CSS transform**（会绕 SVG 原点转飞）。用 JS 每帧 `setAttribute('transform', 'rotate(deg cx cy)')` 显式给旋转中心。
- 胖瘦缩放用 CSS `transform: scaleX()` 作用在 **HTML 包装元素**上（不是 SVG 内部节点），`transform-origin: 50% 100%` 保证脚不离地。
- 多层动画各占一层嵌套元素，避免 transform 互相覆盖：外层(胖瘦) → 中层(呼吸) → svg(咀嚼)。
- 掉落食物用**两层嵌套 div 分别跑 translateX 线性 / translateY 缓入**合成抛物线，不需要逐帧 JS。

## 交互元素必须"抗改写"（血的教训）
用户反馈「喂食键鼠标不移上去就看不见」。在干净浏览器里实测一切正常（底色粉、白字、透明度 1、中心点最顶层就是按钮自己、无遮挡、五种视口都不溢出），**说明是宿主环境的外部样式把它盖掉了**。

判断依据来自症状本身：`.btn--feed` 特异度 (0,1,0)，`:hover` 那条是 (0,3,0) —— **hover 生效、非 hover 被盖**，说明有个外来规则的特异度正好卡在两者之间。

所以本项目的按钮/进度条一律这样写：
- 选择器用 **`#id`**（(1,0,0)），不用 `.class`；
- `background` / `color` / `border-color` / `opacity` 全部加 **`!important`**（只钉这几个属性，**不要碰 JS 会写的 `width`**，否则 JS 的 inline style 会被压掉）；
- 颜色写成 `var(--x, #兜底色)` 带兜底值，防止变量解析失败导致「invalid at computed-value time」→ 属性变 initial → 背景变透明；
- hover / active / disabled / `is-on` 这些状态规则也都要带 `!important`，且靠更高特异度（如 `#feedBtn:hover:not(:disabled)` = (1,2,0)）压过基础规则；
- `:root` 加 **`color-scheme: light`**，防止浏览器/深色插件自动改写表单控件配色。

**通用结论**：凡是"底色一被抹掉就等于隐形"的元素（实心按钮、进度条填充），都必须按上面的方式加固。这不是洁癖——这个页面会被嵌进宿主预览面板，也可能被深色插件改写。

## 布局回归基线（改样式后请复核这几项）
用探针在 420×860 / 420×700 / 420×600 / 420×520 四个视口下检查，应为：
`errors: []`、`appOverflow: 0`、`buttonsInView: true`、`feedBox` 宽 166、`petBox`/`sleepBox` 宽 117、`stageOverlapsActions: false`。

## `flex` 简写的覆盖坑
`.actions .btn--feed { flex: 1.5 }` 写在 `.actions .btn { flex: 1 }` **之前**会被后者覆盖（同特异度、后者在后）。有先后依赖的规则要么调换顺序，要么改用子选择器 `>` 明确层级。

## 时间处理的铁律
`Date.now()`（绝对时间，用于逻辑/存档/天数）与 `performance.now()`（单调时间，用于动画相位）**绝不能相减**。之前混用导致页头显示「第 -20718 天」。另外 `Math.sin()` 的参数过大（1.8e12）会丢精度，动画相位只能用单调时间。
