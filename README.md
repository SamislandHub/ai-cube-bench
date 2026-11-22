# 魔方测试台 · Cube Lab

**单文件、零依赖的 2–8 阶魔方模拟器，同时是一台给大模型出题的评测台。**

打开页面就能转魔方；而任何能执行 JS 的智能体，只要调用暴露在 `window` 上的 `CubeAPI`，就能领题、读面块串、提交解法、自动拿分。

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen.svg)](#)
[![Single file](https://img.shields.io/badge/build-none-lightgrey.svg)](#)

---

## 这是什么

一个 `index.html`，约 1800 行，把三件事塞进同一个文件：

- **3D 魔方**：2–8 阶，Canvas 2D 手写透视投影，不依赖 three.js / WebGL
- **可编程接口**：`CubeAPI` 覆盖读状态、转层、打乱、撤销、成绩查询，共 25+ 方法
- **AI 评测协议**：`evaluate()` → 自行求解 → `submit()`，全自动计分

它想解决的问题很具体：**怎么公平地考一个模型的魔方水平。**

很多魔方站点是「先看到打乱状态，再开始计时」，模型可以提前把解法算完。这里反过来：

1. 领取题目（`start()`）的瞬间才揭示魔方状态，同时开始计时；
2. 计时**包含模型思考时间**，一直到复原或交卷；
3. 计分只看 `submit()` 传入的解法本身，与是否读过页面代码无关。

---

## 快速开始

不需要 Node、不需要构建、不需要联网：

```bash
# 方式一：直接双击 index.html

# 方式二：本地起个静态服务（推荐，避免个别浏览器的 file:// 限制）
python -m http.server 8080
# 然后打开 http://localhost:8080
```

也可以把 `index.html` 丢到任意静态托管上，或者嵌进 iframe（见下文「跨 iframe 调用」）。

---

## 手动玩

| 操作 | 效果 |
| --- | --- |
| 拖动魔方表面 | 转动该层（自动判断方向） |
| 拖动空白处 / `Shift` + 拖动 | 旋转视角 |
| 滚轮 | 缩放 |
| 双击画布 | 复位视角 |
| `空格` | 重新出题（开始新测试） |
| `Ctrl/⌘ + Z` | 撤销上一步 |
| 工具栏阶数 | 2–8 阶任意切换，或点 2/3/4/5/6 快捷 |

复原会自动结算成绩；解不出来可以点「交卷」，记为 DNF。

---

## 给 AI 用：三步评测

```js
// 1. 领题：一次调用就知道现在该做什么、题目是什么、怎么交卷
const info = CubeAPI.evaluate();
// → { action: "CALL_START", ... } 还没有题目
// → { action: "SOLVE_NOW", task: { order, facelets, seed, scrambleMoves, elapsed }, ... }

// 2. 自己求解 task.facelets（54 字符，面序 U R F D L B）

// 3. 交解法：自动执行 + 自动结算
CubeAPI.submit("U2 L2 U2 F' U F R F R2 U' F2");
// → { solved: true, score: { score, grade, moves, seconds, ... }, ... }
```

`evaluate()` 的返回值里已经带上了协议、面块格式、转动记号、评分规则，**不需要读源码也不需要额外文档**。这是刻意的设计：页面自解释，模型不必"作弊式"地读实现。

完整流程示例（含种子复现）：

```js
CubeAPI.start({ order: 3, seed: 42, moves: 36 });   // 出题 + 开始计时
const { facelets } = CubeAPI.evaluate().task;        // 读题
// ... 交给你的求解器（Kociemba / 层先法 / 搜索）...
CubeAPI.submit(solution);                            // 复原即自动结算
```

### 面块串格式

- 54 个字符（`6 × 阶数²`），面序 **U R F D L B**
- 每面从**外侧看**、行优先排列
- 字符就是该块的颜色字母：`U R F D L B`
- 中心块的颜色等于该面的字母，可直接喂给 Kociemba 类求解器

### 转动记号

标准记号 `U D L R F B`，可加 `'`（逆转）或 `2`（转两次），空格分隔：

```
R U R' U' F2 D'
```

---

## CubeAPI 速查

### 会话

| 方法 | 说明 |
| --- | --- |
| `evaluate()` | **入口**。返回「当前该做什么 + 协议 + 题目 + 提交方式」 |
| `start({ order, seed, moves })` | 领取题目：出题并揭示状态，开始计时 |
| `submit(solution)` | 提交解法：自动执行，复原即自动结算 |
| `finish()` | 手动交卷；未复原记 DNF（0 分） |
| `getSession()` | `state(idle/active/finished)`、`seed`、`elapsed`、`moves`、`solved`、`result` |

### 读取

| 方法 | 说明 |
| --- | --- |
| `getState()` | 完整状态：`order` / `facelets` / `faces` / `centers` / `solved` / `moves` / `time` / `history` |
| `getFacelets()` | 只返回面块串 |
| `isSolved()` | 是否已复原 |
| `getOrder()` | 当前阶数 |
| `getHistory()` | 已执行的底层转动列表 |
| `getCubies()` | 每个小块的 `pos` / `orient` / `stickers` |

### 操作

| 方法 | 说明 |
| --- | --- |
| `move("R U R' F2")` | 按标准记号执行，支持字符串或数组 |
| `animate("R", 300)` | 带动画转一个 90°，用于演示 |
| `moveAxis(axis, layer, dir)` | 底层转动：`axis` 0=x 1=y 2=z，`layer` 0..n-1，`dir` ±1 |
| `scramble({ seed, moves, includeInner })` | 带种子打乱（等价于 `start`） |
| `reset()` | 复原到初始状态 |
| `setOrder(n)` | 切换 2–8 阶 |
| `setState(facelets)` | 用面块串设置任意状态（**仅 2/3 阶**） |
| `undo()` | 撤销上一步 |
| `resetView()` | 复位视角 |
| `help()` | 返回完整接口清单（JSON） |

### 成绩

| 方法 | 说明 |
| --- | --- |
| `getScore()` | 本次成绩完整明细 |
| `getScores()` | 最近 20 条成绩（存 `localStorage`） |
| `clearScores()` | 清空成绩 |
| `getScoreConfig()` | 评分基准、权重、等级阈值 |

### 事件

| 名称 | 说明 |
| --- | --- |
| `cube:ready` | 接口就绪，`detail.version` |
| `cube:change` | 状态变化，`detail.kind` / `detail.state` |

---

## 评分规则

```
strength  = sqrt(打乱步数 / (阶数 × 8 + 12))        // 题越长越难
parMoves  = 基准步数 × strength
parSeconds= 基准时间 × strength

efficiency = 0.6 × min(2, parMoves / 实际步数)
           + 0.4 × min(2, parSeconds / 实际秒数)

score = round(efficiency × (1 + (阶数 - 3) × 0.25) × 1000)
```

- 步数占 **60%**，时间占 **40%**，单项效率**封顶 2 倍**（防止"只追求极短步数"或"只追求极快"刷分）
- 高阶有难度系数：`1 + (阶数 - 3) × 0.25`（8 阶为 2.25）
- 计时从 `start()` 出题开始，到复原/交卷结束，**含模型思考时间**
- 交卷时未复原 → **DNF**，0 分

### 基准值（par）

| 阶数 | 基准步数 | 基准时间 |
| --- | --- | --- |
| 2 | 11 | 25 s |
| 3 | 20 | 60 s |
| 4 | 32 | 120 s |
| 5 | 46 | 200 s |
| 6 | 62 | 300 s |
| 7 | 80 | 420 s |
| 8 | 100 | 560 s |

3 阶的 20 步参照 Kociemba 求解器的典型解长（上帝之数为 20）。

### 等级

| 等级 | 分数 |
| --- | --- |
| S+ | ≥ 1200 |
| S | ≥ 1000 |
| A | ≥ 800 |
| B | ≥ 600 |
| C | ≥ 400 |
| D | < 400 |
| DNF | 未复原 |

---

## 可复现性

同一组 `seed` + `moves` + `order` 永远生成同一道题，方便对比不同模型的成绩：

```js
CubeAPI.start({ order: 3, seed: 42, moves: 36 });
```

- 打乱步数默认 `阶数 × 8 + 12`，上限 500
- 4 阶及以上**默认连内层一起转**——只转最外层时中间块永远不动，打乱后看着像没打乱
- 想只转外层可以显式传 `includeInner: false`（此时中心块保持整齐，适合喂给只认 3 阶的求解器）

---

## 跨 iframe 调用

父页面或外部程序可以通过 `postMessage` 调用任意方法：

```js
const frame = document.querySelector('#cube');
frame.contentWindow.postMessage(
  { type: 'cube-api', id: 1, method: 'evaluate', args: [] }, '*'
);

window.addEventListener('message', (e) => {
  if (e.data?.type === 'cube-api-result' && e.data.id === 1) {
    console.log(e.data.ok, e.data.result);
  }
});
```

保活用 `{ type: 'cube-api-ping' }`，会收到 `cube-api-pong` 与版本号。

---

## 自动演示

URL 参数可以直接看一遍完整求解过程（种子 42 / 36 步打乱的确定性解法）：

```
index.html?demo=1&seed=42&moves=36&pace=480
```

`demo` 也可以传入自己的解法字符串（URL 编码），`pace` 控制每步间隔毫秒数。

---

## 项目结构

```
ai-cube-bench/
├── index.html   # 全部代码：CubeAPI 内核 + 渲染 + 评测计分 + 界面
├── README.md
└── LICENSE
```

文件内部大致分四层：面块/小块状态与转动内核 → Canvas 2D 投影渲染 → 会话与计分 → `CubeAPI` 导出与页面 UI。

---

## 设计取舍

**为什么手写 Canvas 2D 而不是 three.js**：为了保持单文件、零依赖。丢进任何静态托管、离线环境、iframe、甚至本地文件都能直接跑，不用装包也不用等 CDN。

**为什么不内置求解器**：这里考的是模型本身。页面只负责出题、验解、评分；求解必须由被试者完成。

**为什么计时含思考时间**：否则模型可以先算出解法再点开始，成绩就无法反映真实能力。`start()` 之前状态是锁定的，`getFacelets()` 也会直接抛错。

**为什么 `setState()` 只支持 2/3 阶**：任意状态的面块串还原需要为每个小块匹配 24 种朝向，高阶（尤其带内层块）的组合代价增长很快，收益不大。

---

## 已知限制

- `setState()` 仅支持 2 阶和 3 阶
- 成绩只存在浏览器 `localStorage`（最近 20 条），没有后端、没有排行榜
- 界面仅中文，未做 i18n
- 触屏可用，但交互针对桌面鼠标/触控板优化
- 单文件意味着所有样式与逻辑共处一室，二次开发时建议按注释分节阅读

---

## 贡献

Issue、PR 都欢迎。提交前请守住两条底线：

1. **保持单文件零依赖**——不要引入构建步骤、框架或外部 CDN
2. **不要内置求解器**——`CubeAPI` 是出题方，不是答题方

---

## License

[MIT](LICENSE) © 2026 SamislandHub
