# Model Dialects — 按模型自动切换提示词方言

`final-prompt` 根据用户提到的模型，自动用**对应方言**产出最终提示词。**同一条片子，不同模型用的 prompt 结构、参考物处理、参数上限都不一样，不能套一套。**

## 路由表（先判模型，再写）

| 用户提到 | 判定模型 | 方言 |
|---|---|---|
| "seedance 2.5 / seedance-2-5" | Seedance 2.5 | 四模式(omni_reference/t2v/edit/extension) + 参考物角色/排除/保真度 + 括号语法 + 首末帧在正文 + 分段 staging + 720p 上限 |
| "seedance 2.0 / seedance（未说2.5）" | Seedance 2.0 | 六槽公式 + 50–80 词甜区 + 命名替代空形容词 + 正文无负面词 + 首末帧/参考物按上传统 + 4K 仅 mode=std |
| "veo / veo 3.1 / veo 3" | Veo 3 / 3.1 | 五段公式 + 摄影术语 + 音频方向 + 负向=描述所求缺席 + 单条约 8s + 模型名/时长/分辨率不入正文 |
| 没提 / 提到其他 | 默认 = Seedance 2.5 | （最贴合 30s 原生 + 参考物） |

> 规则：**先判模型，再写**；判不准时问一句或默认 2.5。Kling 等可在此基础上再扩。

---

## A. Seedance 2.0 方言

### 六槽公式（按此顺序，缺 3+ 个槽会触发 filter flag）
```
[Camera] + [Subject] + [Action] + [Setting] + [Style] + [Lighting]
```
- **Subject + Action 是承重梁**；Camera 等其余可省，但三缺以上会被审。
- **长度甜区 50–80 词**：前 50–80 词影响力最大（注意力随位置递减）；到 ~180 词开始崩，~220 词硬失败在文本编码器。**宁短勿长。**

### Prompt-Craft 铁律（2.0 特有）
1. **命名替代空形容词**：`cinematic`/`epic`/`beautiful`/`high quality`/`amazing` 是高频标签，被贴到海量训练素材上 → 采样到"什么都不是"。别只删，要**换成一个窄训练名字**：一个真实导演（`Wes Anderson symmetry` / `Kubrick one-point perspective`）、一个灯光 setup（`golden-hour backlight, long shadows stretching forward`）、一个镜头规格（`anamorphic 2.39:1, lens flare from a practical light source`）。
2. **"fast" 是最差降级词**（配复杂动作/运镜）：改用**描述物理**——`feet striking hard, each stride at full extension, arms pumping at 90 degrees` 传达速度，别写 `fast`。
3. **正文不写负面词**（无负面嵌入架构，每个 token 都当正面指令读）：`no jitter, bent limbs` 会被解析成"抖动、弯肢"。改用**正向约束声明句**：`Face stable. Lips anatomically natural. Consistent lighting, no flicker. Body proportions consistent throughout.`
4. **歧义动词 → 同形词坑**：`wind tearing at her coat` 的 `tearing` 会被读成撕/哭。改成只有一种东西能长成的描述：`her coat flutters violently in the wind`。

### 帧坐标系统 & 空间布局块
- 定性锚点：`left third`/`right third`/`center`；`upper third`/`lower third`；`foreground/midground/background`。
- 百分比记法：`x-position 0–100%`、`y-position 0–100%`、`frame occupancy %`。
- **两者成对使用**（定性给电影语言挂钩、百分比给精度）。坐标是**构图意图**，不是逐像素保证。
- **Spatial Layout Block**（>1 主体 / 特定构图意图 / 易错镜）每条：Identity、Screen position、Depth layer、Frame occupancy、Body orientation、Contact points。多人块加：relative distance、eyeline direction、screen-left/right consistency、是否跨中央轴、谁挡住谁。

### 参考物角色（按上传统分配 handle）
上传顺序决定 `@Image1`/`@Image2`…；**每个镜头固定角色**：
| handle | 默认角色 |
|---|---|
| @Image1 | Character identity |
| @Image2 | Costume |
| @Image3 | Environment + lighting |
| @Image4 | Composition |
| @Video1 | Motion only |
| @Video2 | Camera movement only |
| @Audio1 | Rhythm + atmosphere（对 timing 承重） |
- 参考物冲突时在正文显式解决：`@Image2 as costume reference, but recolored to blue for this shot.`
- **承重规则**：参考物管记忆（persistent properties），正文管动作（what happens）；参考物不能驱动新动作，正文不能替代参考物所载内容。**两层各司其职。**

### 引擎硬限制（2.0）
- 年龄盲写（用角色/体态/服装/动作，不写年龄数字）。
- 出镜帧=隐含切；画外=不存在；**禁镜面反射**；**≤3 个可跟踪角色**；双重对比切（double-contrast cuts）。
- 高险镜表：镜面反射、同角双胞胎、人群、文字渲染。
- **4K 仅 `mode=std`**；`mode=fast` 仅 480p/720p。
- 延伸已有片段：作为 video reference 附加 + 以 "The scene continues." 开头；**匹配源分辨率与时长**；链式上限 ~2（硬 3）；**从原始参考物重新锚定**。
- **480p 草稿验证的是 prompt 不是成片**；无 seed 参数；用 Hero Frame + start/end frames 承载一份 look。
- 始终先跑 `python3 scripts/seedance_lint.py --preflight --model seedance_2_0 "<prompt>"`。

---

## B. Seedance 2.5 方言

（详见 `skills/final-prompt/SKILL.md` 主体。要点：四模式先定；参考物带角色+排除+保真度；括号语法 `()音乐 / <>音效 / {}对白 / 《》字幕`，非中文对白先给语言行；首末帧/keyframe 在正文声明；长视频分段+end state；真人 7 槽公式；**720p 上限，无 start/end-frame 介质角色，无 genre hint**。）

---

## C. Veo 3 / 3.1 方言

### 五段公式
```
[Cinematography] + [Subject] + [Action] + [Context] + [Style & Ambiance]
```
- Cinematography：镜头运动（`dolly shot`/`tracking shot`/`crane shot`/`aerial view`/`slow pan`/`POV shot`）、构图（`wide shot`/`close-up`/`extreme close-up`/`low angle`/`two-shot`）、镜头&对焦（`shallow depth of field`/`wide-angle lens`/`soft focus`/`macro lens`/`deep focus`）。
- Subject：主角/焦点，给**具体细节**（服装/道具/人设）。
- Action：主体做什么，**2–4 个清晰节拍**。
- Context：环境/背景元素。
- Style & Ambiance：美学 + 情绪 + 灯光。

### 音频方向（Veo 3 原生音，直接指挥）
- 对白：用**引号**写精确台词：`A woman says, "We have to leave now."`
- 音效：`SFX: thunder cracks in the distance.`
- 环境音：`Ambient noise: the quiet hum of a starship bridge.`
- 想不要对白就直接说，只写 SFX/ambience。

### 负向 = 描述"所求的缺席"，不是 "no X"
- ✅ `a desolate landscape with no buildings or roads`
- ❌ `no man-made structures`

### 关键约束
- **模型名/版本、时长、宽高比、分辨率、生成设置一律不入正文**——当外部参数。
- **单条约 8s**（Veo 3 / 3.1 单次生成短）；要做 30s 就**多段生成再拼接**，或走平台的 autoshot/多镜流程。
- i2v / 有首帧：**让用户先给图**（可选但有用），用图锚定主体身份/服装/环境，再描述运动、镜头、soundstage；**除非用户要分析或某处必须变，否则不要深度描述输入图本身**。
- 首末帧变换：先生成起始帧图 + 结束帧图，再 prompt Veo 从首帧过渡到末帧，描述运镜/转变和关键节拍。

---

## 生成参数对照（都不入正文）

| 模型 | mode | 分辨率上限 | 时长上限 | 宽高比 | 参考物 |
|---|---|---|---|---|---|
| Seedance 2.0 | std / fast | 4K(std)/720p(fast) | 单次短；延续另做 | 16:9/9:16/… | start/end frame、video ref、角色图 |
| Seedance 2.5 | t2v / omni_reference / video_edit / video_extension | **720p** | **4–30s 原生** | auto/21:9/16:9/4:3/1:1/3:4/9:16 | image/video/audio reference 数组（无 start/end 角色） |
| Veo 3 / 3.1 | —(外部) | 1080p | **约 8s/条** | 16:9/9:16 等 | 支持 i2v（首帧/音频） |
