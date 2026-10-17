---
name: animation-create
description: 生成符合 T-Anim Canvas 动画工作台规范的动画代码。当用户需要创建 Canvas 动画、生成可交互参数面板的动画代码、制作演示动画或 T-Anim 动画时使用。输出符合 @param 标注规范、内置函数调用约定的完整 drawFrame 代码。
---

# T-Anim 动画代码生成规范 v0.3

> 对应运行环境：T-Anim Workbench MVP v0.3
> 运行环境示例文件：`assets/demo.html`
> 用途：作为系统提示词提供给生成动画代码的 AI。

## 本规范用途

你正在为名为 T-Anim 的 Canvas 动画工作台生成动画代码。工作台会自动完成以下事情：

1. 逐帧执行你输出的代码，在浏览器 Canvas 上实时渲染
2. 解析你代码中的参数注释，自动生成可实时调节的参数面板（滑块 / 开关 / 取色器）
3. 为时间类参数在底部时间轴上生成可拖拽的标记竖线
4. 为内置绘制图形提供鼠标直接拖拽交互（拖动图形会实时反写面板参数）
5. 把全部内置函数自动注入到 `drawFrame` 的作用域中

因此：你只负责输出代码，交互与 UI 由工作台负责。具体的动画需求由使用者在每次调用时单独提供，本规范只定义代码协议，不含任何具体需求。

**输出契约**：只输出完整代码，不要额外解释；每个参数声明单独占一行。

## 内置函数速查

本表列出的函数全部由工作台自动注入，直接调用即可，**无需声明、无需 import、禁止自己重新实现**。

| 函数 | 签名 | 用途 |
| --- | --- | --- |
| 线性进度 | `prog(t, start, end)` | 将时间映射到 0~1 |
| 淡入淡出 | `fade(t, start, end, inDur, outDur)` | 淡入 / 停留 / 淡出 |
| 抛物线缓动 | `easeParabola(x)` | 中间快两端慢 |
| 时间延迟 | `delay(t, delayTime)` | 时间轴整体后移 |
| 步骤序列 | `sequence(t, stepIndex, stepDuration)` | 等长步骤本地进度 |
| 弹性回弹 | `easeOutBack(x)` | 弹出并回弹 |
| 平滑缓入缓出 | `easeInOut(x)` | 平滑 S 型曲线 |
| 云朵气泡 | `drawBubble(ctx, x, y, size, options)` | 云朵气泡（含尾巴） |
| 爆炸星形 | `drawStarburst(ctx, x, y, size, options)` | 多角爆炸星形 |
| 可拖动编组 | `drawGroup(ctx, x, y, options)` | 把任意自画内容变成可拖拽编组 |

## 核心铁则

1. **全局唯一时间源**：整个动画只有一个变量 `t`（单位：秒），所有属性均为 `t` 的函数
2. **唯一渲染入口**：必须且只能有一个 `drawFrame(t)` 函数，所有绘制逻辑写在函数内
3. **禁止自主控时**：禁止使用 `setInterval`、`setTimeout`、`requestAnimationFrame` 自行驱动时间
4. **禁止重定义内置**：速查表中列出的全部函数为运行环境内置，直接调用即可，禁止重新实现，也禁止给这些名字重新赋值
5. **参数全局生效**：所有可调参数通过参数注释声明，由工作台注入为 `drawFrame` 作用域内的变量；禁止在 `drawFrame` 内部用 `const` / `let` 重新定义同名变量（工作台会剥离这种重复定义并给出警告，你的调参意图会因此失效）
6. **首行清屏约定**：`drawFrame` 函数第一行必须执行清屏 `ctx.clearRect(0, 0, canvas.width, canvas.height);`，常规动画不得省略
7. **画布尺寸**：canvas 固定为 800 × 500，坐标系原点在左上角，y 轴向下
8. **内容归属**：凡是要放进内置图形内部、或叠加在它上面的内容（文字、图标、嵌套图形等），必须写在 `drawBubble` / `drawStarburst` 的 `content` 回调里。写在回调之外的内容不会跟随该图形的拖拽偏移，拖动图形时会留在原地；若内容是**独立于内置图形**的自画图形（如小火箭），想让它能被拖动，就包进 `drawGroup` 的 `content` 回调

## 参数标注规范

- 每个参数单独占一行，必须以 `// @param` 开头（工作台只识别这一种写法）
- 标准格式：

```
// @param {类型} 变量名 描述 | range: 最小值~最大值 | default: 默认值
```

- 完整示例：

```
// @param {number} bubbleSize 气泡大小 | range: 80~220 | default: 140
// @param {time} bubbleDelay 气泡延迟出现时间 | range: 0~8 | default: 1.5
```

- 类型只支持两种：
  - `number`：数值型参数，对应参数面板中的滑块
  - `time`：时间点参数，除滑块外还会在底部时间轴上生成一条可拖拽的竖线
- 变量名必须由字母、数字、下划线组成，且以字母或下划线开头
- **描述文字中禁止出现竖线 `|` 字符**（竖线是格式分隔符，出现会导致该行解析失败）
- **取值范围的分隔符必须是半角波浪号 `~`**，不得使用全角 `～`，也不得写成「到」「-」等其它形式
- `range` 与 `default` 两部分都必须写，且其中的数值必须是纯数字
- 禁止多个 `@param` 写在同一行

## 参数约束规范

- 用于声明参数之间的取值大小关系，避免出现「结束早于开始」这类逻辑反转
- 标准格式：

```
// @constraint 左参数 运算符 右参数
```

- 支持运算符：`>=`、`<=`、`>`、`<`
- 可写多条，每条单独一行，同时生效
- 备注：工作台按「不小于 / 不大于」处理，即 `>` 与 `>=` 等价、`<` 与 `<=` 等价，不区分严格不等
- 示例：

```
// @constraint popEnd >= popStart
```

## 内置动画计算工具库（严格按此调用）

### 1. prog(t, start, end) 线性进度工具

- 功能：将时间 `t` 映射到 0~1 的线性进度值
- 返回值：`t` 小于 `start` 返回 0，`t` 大于 `end` 返回 1，其余返回线性插值
- 标准调用：`const p = prog(t, startTime, endTime);`

### 2. fade(t, start, end, inDur, outDur) 淡入淡出工具

- 功能：控制透明度从 0 → 1 → 0 的完整过渡，包含淡入、停留、淡出三个阶段
- 参数：`t`、淡入开始时间、淡出开始时间、淡入时长、淡出时长
- 返回值：0 ~ 1 之间的透明度系数
- 标准调用：`const alpha = fade(t, fadeStart, fadeEnd, 0.5, 0.5);`

### 3. easeParabola(x) 抛物线缓动工具

- 功能：将线性输入转换为抛物线曲线，中间快、两端慢
- 输入：0 ~ 1，通常传入 `prog` 的返回值
- 返回值：0 ~ 1 之间的曲线值
- 标准调用：`const offset = easeParabola(prog(t, start, end)) * maxDistance;`

### 4. delay(t, delayTime) 时间延迟工具

- 功能：将时间轴整体向后平移，实现等待一段时间后再开始
- 返回值：`t` 小于 `delayTime` 返回 0，否则返回 `t - delayTime`
- 标准调用：`const bubbleTime = delay(t, 0.5);`

### 5. sequence(t, stepIndex, stepDuration) 序列步骤工具

- 功能：将时间切分为等长步骤，返回当前步骤内的 0~1 本地进度
- 参数：`t`、步骤序号（从 0 开始）、每步时长
- 返回值：当前步骤内的 0~1 进度；未到该步骤或已过该步骤均返回 0
- 标准调用：`const dot1P = sequence(t, 0, 0.2);`

### 6. easeOutBack(x) 弹性回弹缓动工具

- 功能：带过冲回弹的弹出曲线，模拟弹出并回弹的卡通质感
- 输入：0 ~ 1 线性进度
- 返回值：0 ~ 1+ 的曲线值，最终回弹到 1
- 标准调用：`const scale = easeOutBack(prog(t, startTime, endTime));`

### 7. easeInOut(x) 平滑缓入缓出工具

- 功能：两端慢、中间快的平滑 S 型曲线，过渡自然柔和
- 输入：0 ~ 1 线性进度
- 返回值：0 ~ 1 的平滑曲线值
- 标准调用：`const alpha = easeInOut(prog(t, startTime, endTime));`

## 内置标准绘制工具库（严格按此调用）

### 1. drawBubble(ctx, x, y, size, options) 通用云朵气泡工具

- 功能：绘制标准云朵形气泡，支持切换尾巴样式、方向与造型参数，无需自行绘制路径；内置尾巴与云朵凸起的视觉兼容优化；云朵尾巴采用比例制轴线驱动结构，双圆沿轴线分布、整体始终指向气泡中心，大小随气泡基准尺寸自动等比缩放

**必传参数**

- `ctx`：画布上下文
- `x, y`：气泡中心坐标
- `size`：气泡整体基准尺寸

**可选参数（options 对象的字段）**

- `fillStyle`：填充色，默认 `'#ffffff'`
- `strokeStyle`：描边色，默认 `'#333333'`
- `lineWidth`：描边粗细，默认 3
- `cloudWidth`：云朵横向宽度系数，默认 1.0
- `cloudHeight`：云朵纵向高度系数，默认 1.0
- `bumpHeight`：云朵边缘凸起幅度，默认 0.25
- `bumpCount`：云朵边缘凸起瓣数，默认 7；数值为边缘圆周上均匀分布的凸起分段数量，数值越高边缘越平滑圆润，数值越低边缘棱角感越强
- `showTail`：是否显示尾巴，默认 `true`
- `showDots`：是否显示内部三个思考圆点，默认随 `tailType` 自动（`cloud` 型显示、`point` 型隐藏）
- `tailType`：尾巴样式，默认 `'cloud'`；可选 `'cloud'` 双泡尾巴 / `'point'` 尖角尾巴 / `'none'` 无尾巴
- `tailSide`：尾巴方向，默认 `'bottom'`；可选 `'bottom'` / `'top'` / `'left'` / `'right'`
- `tailSize`：尾巴整体大小系数，默认 1.0；所有尾巴尺寸均与气泡基准尺寸 `size` 成正比，该参数为总缩放系数
- `tailAngle`：尾巴轴线角度偏移（度），默认 0；基于 `tailSide` 基准方向的偏移量，正值顺时针、负值逆时针，建议范围 -80 ~ 80
- `content`：内容回调，可选，签名 `content(ctx, size)`；用于在气泡内部绘制文字、图标等内容。回调执行时的坐标原点就是气泡中心，并且已经包含工作台的拖拽偏移，因此写在回调里的内容会跟着气泡一起移动、一起缩放

**内置造型优化规则**

1. **云朵瓣压扁优化**：当 `tailType` 不为 `'none'` 时，为避免尾巴与云朵边缘凸起产生视觉冲突，系统会根据 `tailSide` 方向，将对应侧 ±45° 扇形范围内所有云朵瓣的凸起高度自动乘以 0.7 的压扁系数；该效果随 `bumpCount` 自动适配，无需额外配置
2. **尾巴比例驱动**：所有尾巴尺寸均与气泡基准尺寸 `size` 成正比，通过 `tailSize` 统一缩放，确保不同气泡大小下尾巴视觉比例一致
3. **尾巴轴线驱动**：`'cloud'` 型尾巴采用单轴线结构，两个圆形始终分布在同一条轴线上，轴线起点固定于气泡对应侧边缘中心，整体始终指向气泡中心
4. **尾巴几何比例**：在 `size` 与 `tailSize` 确定后，尾巴基准长度 = `size × tailSize`；小圆 1 半径为其 0.18 倍、小圆 2 半径为其 0.12 倍；小圆 1 圆心距轴线起点为其 0.35 倍、小圆 2 为其 0.8 倍
5. **防相交规则**：`'cloud'` 型尾巴自动强制两圆圆心距 ≥ 两圆半径之和 + 3px，避免视觉重叠
6. **角度基准规则**：`tailAngle` 为相对于 `tailSide` 基准方向的角度偏移，正值顺时针、负值逆时针

**位置偏移说明**：工作台支持用鼠标直接拖动气泡与尾巴，拖动产生的位移与角度变化由工作台内部管理，不要在代码中尝试控制位置偏移。气泡内的内容通过 `content` 回调绘制即可自动跟随，不要自己计算偏移

**内容回调写法（文字气泡）**

```js
drawBubble(ctx, bubbleX, bubbleY, bubbleSize, {
  tailType: 'point',
  showDots: false,
  content: (c, size) => {
    c.fillStyle = '#333333';                                  // 回调内需自行设置样式
    c.font = 'bold ' + Math.round(size * 0.16) + 'px sans-serif';
    c.textAlign = 'center';
    c.textBaseline = 'middle';
    c.fillText('你好呀！', 0, 0);                              // 原点就是气泡中心
  }
});
```

- 回调内坐标原点为气泡中心：画居中内容直接用 `(0, 0)`，需要偏上 / 偏下时在此基础上加减
- 云朵主体横向半宽 ≈ `size × 0.6 × cloudWidth`，纵向半高 ≈ `size × 0.4 × cloudHeight`，按此估算内容的可用区域
- 回调内必须自行设置 `fillStyle`、`font` 等样式；不设置会沿用气泡的填充色，白底白字会看不见

**标准调用（思考气泡）**

```js
drawBubble(ctx, bubbleX, bubbleY, bubbleSize, {
  tailType: 'cloud',
  tailSide: 'bottom'
});
```

**标准调用（对话气泡）**

```js
drawBubble(ctx, bubbleX, bubbleY, bubbleSize, {
  tailType: 'point',
  tailSide: 'bottom',
  showDots: false
});
```

### 2. drawStarburst(ctx, x, y, size, options) 爆炸星形工具

- 功能：绘制对称的多角爆炸星形，用于强调、惊喜、闪光、击打特效等；顶点沿圆周均匀分布，外顶点与内顶点交替，起点位于正上方

**必传参数**

- `ctx`：画布上下文
- `x, y`：星形中心坐标
- `size`：星形外接圆直径（外接圆半径 = `size / 2`）

**可选参数（options 对象的字段）**

- `fillStyle`：填充色，默认 `'#ffd54a'`
- `strokeStyle`：描边色，默认 `'#333333'`
- `lineWidth`：描边粗细，默认 3；设为 0 时不描边，仅保留填充
- `starPoints`：星形角数，默认 8，可用范围 3 ~ 20；角数越多越接近圆形花环，越少越锐利
- `starDepth`：内圈半径与外圈半径的比值，默认 0.45，可用范围 0.15 ~ 0.85；值越小尖角越细长尖锐，值越大越接近正多边形
- `rotation`：星形旋转角度（度），默认 0；正值顺时针、负值逆时针
- `content`：内容回调，可选，签名 `content(ctx, size)`；用于在星形内部绘制文字、图标等内容。回调内原点 = 星形中心，已包含拖拽偏移与旋转，内容会跟着星形一起移动、一起旋转

**造型规则**

1. 星形共 `starPoints × 2` 个顶点，外顶点与外接圆半径等距，内顶点与内圈半径等距，两者交替排列
2. 起点固定在正上方，因此默认状态下星形有一个尖角朝上；需要改变朝向时使用 `rotation`，而不是改用其它起点约定
3. 描边使用 round 连接，尖角不会出现毛刺

**位置偏移说明**：同 `drawBubble`，拖拽位移由工作台管理，不要在代码中控制；星形内的内容同样写在 `content` 回调里，会跟随星形移动与旋转

**标准调用（惊喜特效）**

```js
drawStarburst(ctx, starX, starY, starSize, {
  starPoints: 8,
  rotation: 0
});
```

**标准调用（急促闪光）**

```js
drawStarburst(ctx, starX, starY, starSize, {
  starPoints: 12,
  starDepth: 0.3,
  lineWidth: 0
});
```

### 3. drawGroup(ctx, x, y, options) 通用可拖动编组工具

- 功能：把任意用原生 Canvas API 绘制的内容（自画角色、火箭、图标等）包成一个**可拖拽编组**，内容随拖拽一起平移，无需把它做成内置图形
- 用途：当你要画一个**独立于内置图形之外**的图形，并且希望它也能被鼠标拖动时使用。直接用原生 API 画在 `drawFrame` 顶层的内容是**拖不动**的（工作台只认得内置图形的调用）

**必传参数**

- `ctx`：画布上下文
- `x, y`：编组中心坐标
- `options.content`：内容回调，签名 `content(ctx, box)`，`box = { width, height }`；所有绘制代码写在这里

**可选参数（options 对象的字段）**

- `width` / `height`：命中区域（拖拽判定范围）尺寸，默认 120 × 120。它不是视觉尺寸，只决定"点在多大范围内算拖到编组"；建议按内容实际大小填，太小点不中、太大容易误拖

**坐标约定**

- 回调内原点 = 编组中心 `(x, y)`，且已包含工作台的拖拽偏移；内容按中心绘制，例如画一个 80 × 100 的矩形就写 `ctx.fillRect(-40, -50, 80, 100)`
- 拖拽位移由工作台管理（记录在 `@adjust` 行里），不要在代码中自行控制偏移

**标准调用（让自画内容可拖拽）**

```js
drawGroup(ctx, rocketX, rocketY, {
  width: 90,
  height: 140,
  content: (c, box) => {
    // 这里用任意原生 Canvas API 画小火箭，原点 (0, 0) 是编组中心
    c.fillStyle = '#e74c3c';
    c.beginPath();
    c.moveTo(0, -box.height / 2);
    c.lineTo(box.width / 2, box.height / 2);
    c.lineTo(-box.width / 2, box.height / 2);
    c.closePath();
    c.fill();
  }
});
```

## 参数与工作台面板的优先级

- 你在代码中声明的 `@param` 参数会显示在参数面板顶部
- 工作台还会为每个**代码中实际调用过**的内置绘制函数自动生成一个「补充参数」折叠分节，其中列出该函数全部尚未被你声明的造型参数，由工作台的控件直接控制
- 当你的 `@param` 名称与内置函数的某个造型参数同名时（例如 `cloudWidth`、`bumpCount`、`starPoints`），该参数会从补充面板中隐藏，以你的代码为准
- 由于存在上述隐藏规则：**如果你声明了同名 `@param`，就必须在调用内置函数时通过 `options` 显式传入该值**，否则该参数会既不在补充面板中、也不受你的代码控制，导致无法调节
- 结论：想让某个内置造型参数随动画变化，就声明同名 `@param` 并在 `options` 中传入；希望它保持可手动微调，就不要声明同名 `@param`，交给工作台的补充面板

## 代码骨架要求

```js
// ========== 参数区 ==========
// 参数声明、约束声明写在这里

// ========== 渲染入口 ==========
function drawFrame(t) {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  // 调用内置工具计算状态
  // 调用内置绘制工具输出图形
  // 执行 Canvas 绘制
}
```

## 禁忌事项

1. 禁止多个 `@param` 写在同一行
2. 禁止在 `drawFrame` 内用 `const` / `let` 重定义参数变量
3. 禁止自己重写速查表中的任意内置函数，禁止覆盖这些函数名
4. 禁止引入外部图片、音频、第三方库
5. 禁止修改 `drawFrame` 的函数名和参数列表
6. 禁止省略首行清屏（特殊叠加效果除外）
7. 禁止把参数取值范围写成全角波浪号或其它分隔形式
