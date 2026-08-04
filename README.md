# Web DJ · 无硬件控制台

一个**纯浏览器运行**的双唱盘 DJ 应用，不需要任何外接设备。用 Web Audio API 实时合成音频路由，把鼠标操作还原成旋钮、推子、转盘的手感。整个程序是**单个 `index.html` 文件**，双击即可运行，方便放进 U 盘带走。

## 功能一览

- **双唱盘（Deck A / B）**：播放/暂停、CUE 监听、SYNC 一键 BPM 同步、调速推子、三段 EQ（低/中/高）、通道推子、交叉推子
- **可视化**：转盘进度环 + 进度条 + **波形显示**，可点击进度条跳转、拖动转盘刮擦（seek）
- **切歌辅助系统**：
  - 实时 **BPM 检测**（低频频谱通量法 + 八度折叠 + 间隔直方图）
  - **调性估算**（chromagram 取主导音高）
  - BPM 差值 / 调性兼容提示 / 节拍对齐闪点 / 一句话切歌建议
  - **TAP 打拍**手动测速
- **快捷键**：应用内按 `?` 查看完整列表
- **Auto DJ 模式**：模仿真实打歌的自动混音 —— BPM 同步、相位对齐、渐变 EQ、混响过渡，并按兼容度**智能选曲**（详见下方架构）
- **混音导出**：用 `MediaRecorder` 录制主输出为 `webm` 下载
- **播放列表持久化**：IndexedDB 保存音频文件与元数据，刷新 / 重启不丢
- **智能排序**：按 BPM 升序排列

## 运行方式

直接用 **Chrome** 双击 `index.html` 打开即可（无需服务器、无需安装）。

> ⚠️ 不要通过某些预览沙箱打开——里面的 `<iframe>` 通常是沙箱化的，文件选择器会被拦截。
> 首次播放会在你点击「播放」时激活 `AudioContext`（浏览器自动播放策略要求用户手势）。

## 操作速览

| 想做 | 怎么做 |
|------|--------|
| 添加歌曲 | 点「+ 添加歌曲」选本地 MP3 / WAV |
| 载入唱盘 | 选中列表歌曲 → 点「载入」，或直接点歌曲右侧 `→A` / `→B` |
| BPM 同步 | 点唱盘「SYNC」或按 `S`，会让当前唱盘按 BPM 比例对齐另一路 |
| 自动混音 | 点「Auto DJ: OFF」或按 `T` 开启连续自动打歌 |
| 录制混音 | 点「● 录制」开始 / 停止，导出 `webm` 到下载目录 |
| 查看快捷键 | 按 `?` |

## 架构（单文件，分层）

```
界面层 UI ── 旋钮/推子/转盘/进度环/波形（纯 DOM + CSS）
    │
控制层 ── 事件绑定 + requestAnimationFrame 时钟 tick() 统一刷新
    │
播放列表模型 ── tracks[]（文件名 / URL / 时长 / BPM / 调性 / intro/outro / fileId）
    │
音频引擎 AudioEngine
    ├─ 每路 Deck 音频图：
    │     source → [low shelf → mid peaking → high shelf] EQ
    │            → channelGain → crossGain → masterBus（扬声器）
    │                           └→ cueGain  → cueBus（监听）
    │                           └→ reverbSend → 全局混响 → masterBus
    ├─ Mixer：交叉推子（等功率 cos/sin 混音）、主音量、监听音量
    └─ 导出：masterBus → MediaStreamDestination → MediaRecorder
    │
持久化 ── IndexedDB（files 存音频 Blob，playlistMeta 存元数据，防抖写入）
```

- **调速听感**：`audio.preservesPitch = true`，变速不变调，±15% 内基本顺滑
- **混响**：`ConvolverNode` + 程序生成的指数衰减脉冲响应（免费"房间混响"）
- **BPM 八度折叠**：电子/舞曲常把半拍误判成双倍 BPM（如 75→150），检测后折回 60–200 合理区间
- **Auto DJ 状态机**：`idle → scanning`（监听曲末）→ `ready`（对齐 BPM/相位）→ `transitioning`（8 秒渐变：等功率交叉 + 低音交换 + 中频缓入 + 混响 washout）→ 回到 `scanning`

## 已知限制

- **浏览器为单输出**：CUE 监听与原曲会一起从扬声器出，真正的耳机分离需要外置声卡
- 调速超过 ±15% 时时间伸缩算法可能出现瑕疵
- BPM / 调性为启发式估算，极端曲目可能有偏差
- 波形解码对超大文件会占用一定内存（已用单声道峰值降采样缓解）

## 技术说明（给想改的同学）

- 全部逻辑在 `index.html` 的 `<script>` 内，关键处有中文注释（音频图、BPM 算法、AutoDJ 状态机、持久化）
- 调试：F12 → Console 有全链路日志；页面左上角有全局错误捕获面板
- 想加功能建议从 `Deck` 类（单路音频图）和 `autoDJUpdate()`（状态机）入手
