# Web DJ 代码审计报告

审计对象：`index.html`（1004 行，单文件）
审计日期：2026-08-09
提交：`e001d49`

---

## 总评

一个完成度相当高的个人项目。音频图（source → EQ → channelGain → crossGain → masterBus）设计得干净、正确，等功率交叉推子用 cos/sin、混响用程序生成 IR、BPM 用频谱通量 + 间隔直方图 + 八度折叠——这些都是**行业标准做法**，不是玩具级实现。中文注释写在了关键处（音频图、BPM 算法、AutoDJ 状态机），说明写代码时脑子是清醒的。

但有 5 个 🔴 级 bug 会实际影响使用，其中 2 个（单位错误、共享计时器）让「BPM 检测」和「调性检测」这两个核心卖点基本处于半失效状态。

---

## 🔴 严重问题

### R1. `detectBeat` 时间单位错误 —— BPM 检测的节流与去抖全部失效

**位置**：`index.html:406-419`

```js
d.detectBeat(now);                    // updateAssist 传入 performance.now()，单位=毫秒

detectBeat(now){
  if (now - this._lastAnalysis < 0.03) return false;   // ← 注释说"节流到约 30fps"
  ...
  if (flux > Math.max(threshold, 0.01) && now - this._lastBeat > 0.22){  // ← 最小拍间隔
```

`now` 是**毫秒**，但两个阈值是按**秒**写的。后果：

- `0.03` 毫秒的节流 ≈ 无节流。每一帧（~16.7ms）都会执行 `getFloatFrequencyData`，注释里承诺的「省 CPU」完全没生效。
- `0.22` 毫秒的最小拍间隔 ≈ 无去抖。相邻帧间隔 16.7ms 远大于 0.22ms，**任何一次低频能量抖动都会被记为一个拍点**。原本要过滤的「持续轰鸣」根本没被过滤掉。

这直接导致 `beatTimes` 里塞满假拍点，间隔直方图选出的众数偏小，BPM 偏高，再被八度折叠反复对折 —— 检测结果基本是噪声。

**修复**：阈值改成毫秒。

```js
if (now - this._lastAnalysis < 30) return false;        // 30ms ≈ 33fps
...
if (flux > Math.max(threshold, 0.01) && now - this._lastBeat > 220){  // 220ms → 上限 272 BPM
```

> 顺带一提：`getBeatPhase(cur)` 里 `cur` 是 `audio.currentTime`（秒），`60/this.bpm` 也是秒，那一处是**对的**。正是这种「同一个文件里两套时间基准」最容易出事。建议给变量名加单位后缀：`nowMs` / `curSec`。

---

### R2. 换歌时 revoke 了播放列表共享的 blob URL

**位置**：`index.html:379`

```js
load(url, name, index){
  this.ensureGraph();
  // 换歌时释放上一首的 blob URL，避免 Object URL 泄漏
  if (this.audio.src && this.audio.src.startsWith('blob:')) URL.revokeObjectURL(this.audio.src);
```

这个 URL 不是 Deck 私有的，它就是 `tracks[i].url` —— 播放列表里那首歌**唯一的**引用。revoke 之后：

- 那首歌在列表里还显示着，但再也载入不了（静默失败）。
- 更糟：A 和 B 同时载入同一首歌时，A 换歌会把 **B 正在播放的音源** revoke 掉。

**根因**：资源生命周期归属错了。blob URL 属于 track，不属于 deck。

**修复**：Deck 不管 revoke，交给 `removeTrack`（那里已经做对了，`index.html:664`）。

```js
load(url, name, index){
  this.ensureGraph();
  // blob URL 的生命周期由 tracks[] 拥有，Deck 只是借用；释放交给 removeTrack()
  this.audio.src = url;
  ...
}
```

---

### R3. 两个 Deck 共用一个调性检测计时器 —— Deck B 的调性几乎永远算不出来

**位置**：`index.html:921, 928`

```js
let _keyTimer=0;                       // ← 全局共享
function updateAssist(){
  ['a','b'].forEach(p=>{
    ...
    if(d.loaded && now-_keyTimer>2000){ d.estimateKey(); _keyTimer=now; }
```

`forEach` 顺序固定是 a → b。A 满足条件后立刻把 `_keyTimer` 刷成 `now`，同一帧里 B 判断 `now - _keyTimer > 2000` 必然为 false。**Deck B 的 `estimateKey()` 实际上永远不会被调用**，`key-b` 恒为 `--`，「调性兼容提示」这个功能因此对半瘫痪。

**修复**：计时器下放到 Deck 实例。

```js
// Deck 构造函数里加：this._keyTimer = 0;

if(d.loaded && now - d._keyTimer > 2000){ d.estimateKey(); d._keyTimer = now; }
```

---

### R4. `F` 键「手动渐变过渡」在 Auto DJ 关闭时完全不工作

**位置**：`index.html:800, 909`

```js
'f':()=>{ autoDJOut=deckA; autoDJIn=deckB; autoDJState='transitioning'; autoDJIn.play(); ... },

function autoDJUpdate(){
  if(!autoDJOn) return;               // ← 这里直接 return 了
```

按 `F` 设置了 `autoDJState='transitioning'`，但驱动过渡的 `autoDJUpdate()` 第一行就因为 `autoDJOn === false` 返回。结果：**只是让 Deck B 开始播放，没有任何交叉渐变**。README 里宣传的「F：执行交叉渐变过渡（8 秒）」是失效的。

另一个问题：方向写死 A→B，不看当前哪个在播。B 在播时按 F，等于把静音的 A 淡出到 B。

**修复**：让过渡逻辑独立于 Auto DJ 开关，并按实际播放状态定方向。

```js
// 状态机守卫改成：正在过渡时也要执行
function autoDJUpdate(){
  if(!autoDJOn && autoDJState !== 'transitioning') return;
  ...
}

'f':()=>{
  const out = !deckA.audio.paused ? deckA : (!deckB.audio.paused ? deckB : null);
  if(!out){ feedback('没有正在播放的唱盘'); return; }
  const inn = out === deckA ? deckB : deckA;
  if(!inn.loaded){ feedback('另一路唱盘未载入歌曲'); return; }
  autoDJOut = out; autoDJIn = inn;
  autoDJDuration = 8; autoDJStart = performance.now();
  autoDJState = 'transitioning';
  inn.play();
  feedback('手动渐变过渡 8s');
},
```

---

### R5. `Tab` / Auto DJ 直接改 gain，交叉推子 UI 与音频状态脱节

**位置**：`index.html:862-863, 908`

```js
'Tab':(e)=>{ ... deckA.setCross(...); deckB.setCross(...); },   // 只改 gain
```

`crossCtrl` 这个控件内部维护着自己的 `val` 和 thumb 位置，这里绕过它直接写 gain。后果：Tab 快切后推子**视觉上没动**；紧接着按 `↑/↓`（走 `crossCtrl.set`）会从旧的 `val` 开始算，声音**跳回**去。Auto DJ 的 `transitioning` 分支同样直接 `setCross`，8 秒过渡期间推子一动不动。

**修复**：统一走 `crossCtrl.set()`，让控件成为唯一数据源（single source of truth）。

```js
// makeFader 的 onChange 里已经会调用 setCross，所以只需要 set 值
'Tab':(e)=>{ e.preventDefault(); crossCtrl.set(crossCtrl.get() > 0.5 ? 0 : 1); },

// autoDJUpdate 的 transitioning 分支里：
crossCtrl.set(autoDJOut === deckA ? p : 1 - p);   // 替换掉两行 setCross
```

---

### R6. 播放列表用 `innerHTML` 插入文件名 —— HTML 注入

**位置**：`index.html:645`

```js
li.innerHTML = `<span class="tname">${t.name}...`;
```

文件名直接拼进 HTML。一个叫 `<img src=x onerror=alert(1)>.mp3` 的文件会执行脚本。

本地单机应用，实际危害有限（能放文件进来的人已经能控制你的电脑了）。但这是**习惯问题**：同样的代码搬到任何联网场景就是真漏洞。

**修复**：结构用 `innerHTML`，文本用 `textContent`。

```js
const li = document.createElement('li');
const nameEl = document.createElement('span');
nameEl.className = 'tname';
nameEl.textContent = t.name;          // ← 自动转义
if(t.unavailable){
  const w = document.createElement('span');
  w.style.color = '#E24B4A'; w.textContent = ' ⚡缺失';
  nameEl.appendChild(w);
}
li.appendChild(nameEl);
// ...其余按钮同理用 createElement
```

---

## 🟡 中等问题

### Y1. 空值守卫写在解引用之后（等于没写）

**位置**：`index.html:521-523`、`537-539`

```js
function makeKnob(el, {min, max, value=0, onChange}){
  let val = value; const ind = el.querySelector('.knob-ind');   // ← 先解引用 el
  if (!el || !ind) return { set(){}, get:() => value };          // ← 才检查 el
```

如果 `el` 是 null，第 2 行就抛 `TypeError` 了，第 3 行的守卫永远执行不到。`makeFader` 同样问题。**顺序反了**，纯属白写。

```js
function makeKnob(el, {min, max, value=0, onChange}){
  if (!el) return { set(){}, get:() => value };
  const ind = el.querySelector('.knob-ind');
  if (!ind) return { set(){}, get:() => value };
  let val = value;
```

### Y2. 页面加载即创建 AudioContext，违反自动播放策略

**位置**：`index.html:605-606`

```js
makeFader(qs('master'), {..., onChange:v => { ensureAudio(); masterBus.gain.value = v; }});
```

`makeFader` 构造时会立即调用一次 `render()` → `onChange()` → `ensureAudio()`。所以**页面一打开就 new AudioContext**，还没有任何用户手势。Chrome 会在控制台打印 `The AudioContext was not allowed to start`，且 context 处于 `suspended`。README 里写的「首次播放时激活」与实现不符。

**修复**：把待应用的音量缓存起来，等 `ensureAudio()` 真正建好再落盘。

```js
let pendingMaster = 0.9, pendingCue = 0.7;
makeFader(qs('master'), {min:0,max:1,value:0.9,vertical:true,
  onChange:v => { pendingMaster = v; if(AC) masterBus.gain.value = v; }});
// ensureAudio() 里创建完 masterBus 后：masterBus.gain.value = pendingMaster;
```

### Y3. `keyCompat` 没做模 12 环绕

**位置**：`index.html:724-733`

```js
const d = Math.abs(ia - ib);
if(d===7||d===5) return 'compatible';
if(d===1||d===2) return 'energy';
if(d===6) return 'avoid';
```

音高类是**循环**的（B 和 C 相差 1 个半音，不是 11 个）。`A#(10)` 与 `C(0)` 距离算出 10，实际是大二度（应判 `energy`），却落到了 `'ok'`。

```js
const raw = Math.abs(ia - ib);
const d = Math.min(raw, 12 - raw);    // 环绕距离，取值 0..6
if(d === 0) return 'same';
if(d === 5) return 'compatible';      // 四度/五度（5 和 7 环绕后同为 5）
if(d === 1 || d === 2) return 'energy';
if(d === 6) return 'avoid';           // 三全音
return 'ok';
```

> 更根本的局限：chromagram 只取主导音高，**分不出大小调**。真正的 Camelot 混音轮需要 major/minor 维度。可以考虑用 Krumhansl-Schmuckler 调性профиль做相关性匹配（把 24 个大小调模板和 chroma 向量算相关系数取最大），代码量增加约 30 行，准确率提升明显。

### Y4. `getLowEnergy` 硬编码 FFT bin 索引

**位置**：`index.html:403`

```js
for (let i=2;i<12;i++) sum += Math.pow(10, buf[i]/20);
```

bin 宽度 = `sampleRate / fftSize`。48kHz 下 bin 2–11 ≈ 47–258 Hz（底鼓区，合理）；但 44.1kHz 下变成 43–237 Hz，滑动了。设备采样率不同结果就不一致。

```js
const bw = AC.sampleRate / this.analyser.fftSize;
const lo = Math.floor(40 / bw), hi = Math.ceil(250 / bw);   // 固定按赫兹取
let sum = 0; for (let i = lo; i < hi; i++) sum += Math.pow(10, buf[i]/20);
return sum / (hi - lo);
```

### Y5. `addFiles` 在循环内部反复 `renderPlaylist()`

**位置**：`index.html:627`

一次拖入 50 首歌 = 50 次全量重建 DOM + 50 次重新绑定所有监听器。应该移到循环外调用一次。

配套问题：`renderPlaylist` 每次都 `innerHTML=''` 全量重绘并逐个 `addEventListener`，没有**事件委托**。曲库上百首时每次选中都会重建整个列表。

```js
// 一次性委托，renderPlaylist 里就不用再绑了
listEl.addEventListener('click', e => {
  const btn = e.target.closest('button');
  const li  = e.target.closest('li');
  if(!li) return;
  const i = [...listEl.children].indexOf(li);
  if(!btn){ selectedIndex = i; renderPlaylist(); return; }
  if(btn.classList.contains('del-btn')) removeTrack(i);
  else loadTrack(i, btn.dataset.deck);
});
```

### Y6. IndexedDB 每次读写都重新 `openDB()`

**位置**：`index.html:289-300`

`dbPut` / `dbGet` / `dbDelete` 各自 `await openDB()`。`restorePlaylist` 恢复 N 首歌 = N+1 次打开连接。应该缓存单例。

```js
let _dbPromise = null;
function openDB(){ return _dbPromise || (_dbPromise = new Promise((res, rej) => { ... })); }
```

### Y7. `tick()` 每帧重建 conic-gradient 字符串

**位置**：`index.html:986`

```js
ring.style.background = `conic-gradient(var(--accent) ${...}deg, rgba(255,255,255,0.06) 0deg)`;
```

每帧为两个 deck 各重新解析一次渐变语法，是这个循环里最贵的操作。改成 CSS 变量能让浏览器走更便宜的路径：

```css
.progress-ring{ background:conic-gradient(var(--accent) var(--p,0deg), rgba(255,255,255,0.06) 0deg); }
```
```js
ring.style.setProperty('--p', (dur ? cur/dur*360 : 0) + 'deg');
```

同理，`fmt(cur)` 的时间文本每帧写一次 `textContent`，实际上**每秒变一次就够了** —— 缓存上次的秒数，变了才写。

另外 `tick()` 里所有 `qs(...)` 都是每帧现查 DOM。建议在 Deck 构造时把 `-fill` / `-cur` / `-dur` / `-ring` / `-jog` / `-wavehead` 缓存成实例字段。

### Y8. Auto DJ 的两个 `if` 缺 `else`，且用逗号表达式

**位置**：`index.html:805-806`

```js
if(!deckA.audio.paused && deckA.audio.duration-deckA.audio.currentTime<30) playing=deckA,idleDeck=deckB;
if(!deckB.audio.paused && deckB.audio.duration-deckB.audio.currentTime<30) playing=deckB,idleDeck=deckA;
```

两路同时接近曲末时，第二个 `if` 会**静默覆盖**第一个的结果。逗号表达式也让「这里赋了两个值」很难一眼看出来。加 `else if` 并用花括号。

### Y9. `onDeckEnded` 自动续播与 Auto DJ 抢方向盘

**位置**：`index.html:678-682`

曲子自然播完时 `onDeckEnded` 会载入下一首并播放；同一时刻 Auto DJ 状态机可能正处于 `ready`/`transitioning`。两套逻辑都在改同一个 deck。应该在 `onDeckEnded` 开头加 `if(autoDJOn) return;`。

### Y10. `loadTrack` 的副作用泄漏到 Auto DJ

**位置**：`index.html:676`

`loadTrack` 末尾会 `selectedIndex = i; renderPlaylist();`。Auto DJ 自动选曲时调用它，会**擅自改掉用户手动选中的高亮项**。建议拆出 `loadTrackInternal(i, deckKey)` 不碰 selection，UI 路径再包一层。

### Y11. 缺少 `pointercancel` 处理

**位置**：所有 `makeKnob` / `makeFader` / jog 的指针事件

只监听了 `pointerdown/move/up`。系统手势、触摸中断等场景会派发 `pointercancel`，此时 `active` 卡在 `true`，控件「粘住」跟着鼠标跑。给每处补上 `el.addEventListener('pointercancel', () => { active = false; });`。

### Y12. 全局错误面板漏掉 Promise 拒绝

**位置**：`index.html:255-257`

只监听了 `window.onerror`。但项目里 IndexedDB、`decodeAudioData`、`audio.play()` 全是 Promise，拒绝时不触发 `error` 事件，面板里什么也看不到。

```js
window.addEventListener('unhandledrejection', e => {
  const el = qs('errors'); if(el){ el.style.display='block'; el.textContent += '[REJ] '+(e.reason?.message||e.reason)+'\n'; }
});
```

### Y13. 调速推子给了 ±50%，README 说 ±15% 才顺滑

推子范围 `0.5–1.5`，但 README「已知限制」写着超过 ±15% 会有瑕疵。UI 上既没有刻度也没有安全区提示，用户很容易拉到听感崩坏的区间。建议在推子旁显示实时百分比（如 `+3.2%`），并给 ±15% 画一条参考线。

### Y14. 「转盘刮擦」名不副实

jog 的实现是**纵向拖动映射到 seek**（`index.html:592-593`，用 `clientY`）。真实唱盘是**转圈**手势，而且 scratch 的灵魂在于拖动时音高随速度实时变化。当前实现既不转圈也不变调，叫「进度微调」更诚实。

想做真 scratch，得放弃 `<audio>` 元素，改用 `AudioBufferSourceNode` 自己管播放位置和 `playbackRate`（包括负值倒放）。工作量不小，可以列为 v2 目标。

---

## 工程实践建议

### 1. 单文件是对的，但该拆的时候别硬扛

「双击即可运行、放 U 盘带走」是**明确的产品约束**，为此接受单文件完全合理——这个取舍你做对了。

但 1004 行已经到临界点。有两个办法既保住零依赖又拿回可维护性：

- **`<script type="module">` + 同目录 `.js` 文件**：拆成 `audio-engine.js` / `deck.js` / `autodj.js` / `ui.js`。代价：`file://` 协议下 ES module 会被 CORS 拦，必须起本地服务器——**与便携性冲突，不推荐**。
- **保持单文件，但源码分模块 + 构建时内联**（推荐）：源码放 `src/`，写个 20 行的 Node 脚本把各模块拼进 `index.html` 模板，输出到 `dist/index.html`。开发时享受多文件，交付时还是一个 HTML。这也顺便给你练了一次「构建流程」。

### 2. 补 LICENSE

`OI-Learn-Desktop` 已经有 MIT 了，这个仓库还没有。没有 LICENSE 的公开仓库在法律上**默认保留所有权利**，别人不能合法使用。加个 MIT，一分钟的事。

### 3. 开 GitHub Pages —— 这是最高性价比的一步

纯静态单文件项目，仓库设置里 Pages 选 `main` 分支根目录，立刻就有一个 `https://neko-huang.github.io/web-dj/` 的在线地址。

好处远超成本：
- README 顶部放个「在线体验」链接，别人不用 clone 就能玩
- `https://` 环境下 `MediaRecorder`、`AudioContext` 的行为比 `file://` 更标准
- 你自己在任何一台电脑上打开浏览器就能用，比 U 盘还便携

### 4. 提交粒度

目前整个项目**只有 1 个 commit**，1088 行一次性入库。这个习惯要改——不是为了好看，是为了：

- 出 bug 时能 `git bisect` 定位到具体那次改动
- 回滚时能只回滚一个功能，而不是整个项目
- 半年后回看能读懂「当时为什么这么写」

建议粒度：一个可独立描述的功能 = 一个 commit。比如这个项目本来应该是：`feat: 双唱盘音频图与基础播放` → `feat: 三段 EQ 与交叉推子` → `feat: BPM 检测` → `feat: Auto DJ 状态机` → `feat: IndexedDB 持久化`……

commit message 你写得挺规范（用了 `feat:` 前缀），保持。

### 5. 给纯前端项目加一层「能跑」的验证

`.gitignore` 里那行 `_check.js`（本地语法检查用）说明你已经有这个意识了。可以把它正式化：

- 最低成本：`npx eslint index.html --plugin html`，或者干脆 `node --check` 抽出来的脚本部分
- 进一步：把 `detectBeat` 的直方图逻辑、`keyCompat`、`fmt` 这几个**纯函数**抽出来，用 Node 跑几个断言。这些函数不依赖 DOM 和 AudioContext，测试成本极低，但正好覆盖了最容易出错的算法部分（比如 R1 那个单位 bug，一个「喂 120 BPM 的模拟拍点，断言输出 120±2」的测试就能抓到）

### 6. README 与实现的对齐

发现两处不一致，改代码或改文档都行，但要一致：
- 快捷键表里没有 `D`（切换 activeDeck），代码里有
- 「首次播放会在你点击播放时激活 AudioContext」——实际是页面加载就激活了（见 Y2）

---

## 修复优先级

| 优先级 | 编号 | 问题 | 预估工作量 |
|--------|------|------|-----------|
| P0 | R1 | detectBeat 时间单位错误 | 2 行 |
| P0 | R3 | 调性计时器共享 | 3 行 |
| P0 | R2 | blob URL 被误 revoke | 删 1 行 |
| P1 | R4 | F 键手动过渡失效 | ~10 行 |
| P1 | R5 | 交叉推子 UI 脱节 | ~5 行 |
| P1 | Y1 | 空值守卫顺序 | 4 行 |
| P2 | R6, Y3, Y4, Y8, Y9 | 注入 / 算法正确性 / 状态机竞争 | 半小时 |
| P2 | — | 加 LICENSE + 开 Pages | 五分钟 |
| P3 | Y5–Y7, Y10–Y14 | 性能与体验打磨 | 按需 |

**建议动作**：先花 15 分钟把 3 个 P0 修掉——它们加起来不到 10 行代码，但直接决定了 BPM 和调性这两个核心功能到底能不能用。修完自己拿两首歌实测一下检测出来的 BPM 准不准，这个反馈闭环比继续加新功能有价值得多。



