---
name: final-prompt
description: "把前序分析(产品画像/角色/分镜/表演/口播)合成**最终可执行生成提示词**，**按用户提到的模型自动切换方言**：Seedance 2.5(四模式+参考物角色/排除/保真度+括号语法+首末帧+分段 staging+720p)、Seedance 2.0(六槽公式+50–80词+命名替代空形容词+正文无负面词)、Veo 3/3.1(五段公式+摄影术语+音频方向+负向=描述所求缺席+单条约8s)。默认 Seedance 2.5。这是整套 skill 的**必需终点**。Use when: 需要产出最终可执行的生成提示词。"
user-invocable: true
metadata:
  tags: [final-prompt, generation, prompt, seedance-2.5, seedance-2.0, veo, model-router]
  version: 1.1.0
  updated: 2026-08-28
  parent: my-video-director
---

# Final Prompt — 最终可执行提示词（按模型方言）

这是整套流程的**终点和必需输出**。前面 ①~⑥ 是做分析，这一步把它们合成**一段可直接复制、丢进视频模型生成**的提示词。

> 规范来源：各模型的官方 prompt guide / 平台参数快照（Seedance 2.0、Seedance 2.5、Veo 3/3.1）。**不同模型是不同 dialect，不能套同一套。**

---

## 第零步 · 判模型（用户提到哪个就用哪个的方言）

用户没指定时**默认 Seedance 2.5**。判模型看关键词，然后**按该模型方言**产出（详细规范见 `references/model-dialects.md`）：

| 用户提到 | 模型 | 方言要点 |
|---|---|---|
| "seedance 2.5 / seedance-2-5" | Seedance 2.5 | 四模式 + 参考物角色/排除/保真度 + 括号语法 + 首末帧在正文 + 分段 staging + 720p 上限 |
| "seedance 2.0 / seedance"（没说2.5） | Seedance 2.0 | 六槽公式 + 50–80 词甜区 + 命名替代空形容词 + 正文无负面词 + 4K 仅 mode=std |
| "veo / veo 3.1 / veo 3" | Veo 3 / 3.1 | 五段公式 + 摄影术语 + 音频方向 + 负向=描述所求缺席 + 单条约 8s + 模型名/时长/分辨率不入正文 |
| 没提 | **Seedance 2.5** | 默认（最贴合 30s 原生 + 参考物） |

> 判定规则：**先判模型，再写**。判不准就明确问一句，或默认 2.5。**下面 ①~⑥ 的分析是平台无关的；只有 ⑦ 的最终 prompt 和"生成参数"跟着模型方言走。**

---

## 第一步 · 先定 mode（Seedance 2.5 写之前；2.0/Veo 不适用）

同一个句子在不同 mode 里意思不同，2.5 有**四种模式**：

| 用户想干嘛 | mode | 提示词是什么 |
|---|---|---|
| 纯文字描述、无参考物 | `t2v` | 一段场景 brief（下面的核心公式） |
| 用图片/视频/音频参考物构建 | `omni_reference` | **role map** + 场景 brief |
| 改一段已有视频里的东西 | `video_edit` | master + scope + preserve list |
| 往已有视频前后加画面 | `video_extension` | boundary 契约 + 新内容 |

**两条硬规则：**
1. 首末帧 / keyframe / 分镜网格 / blockout **全部属于 `omni_reference`**。
2. **编辑 ≠ 重生成**：想重拍某个镜头是 `omni_reference`（老镜头作 motion reference），不是 `video_edit`。

> 参数面：**480p/720p only，时长 4–30s**，没有 start/end-frame 媒体角色，没有 genre hint，`extension_mode` 只用于（且必须用于）`video_extension`。`video_edit` 忽略 duration/aspect_ratio，按源长度计费；`video_extension` 继承源的宽高比。**这些是生成参数，不是提示词正文。**

---

## 核心提示词公式（t2v / omni_reference 共用）

只写本镜头需要的部分；省略空槽，不要硬凑。四句骨架：

```text
<Subject> performs <primary action or event> in <scene and environment>.
The visuals feature <visual style>.
Use <shot size, camera angle, camera movement, or cuts>.
Audio includes <dialogue, ambience, sound effects, or music>.
```

**每句怎么写：**
- **Subject + action 是承重梁**。"男人跑" → "男人冲刺，外套在气流里向后翻"。动作和主体一起写实。
- **Scene and environment**：地点、时间、天气、空间关系、背景状态。
- **Visual style**：光、色、材质、纹理、氛围。**只写加信息的词**；堆叠的空洞词（"cinematic, 8K, masterpiece"）不采样到任何东西。
- **Camera**：景别、角度、运动、焦点主体、转场。运动要匹配动作（不是装饰）。
- **Audio**：对白、人声特征、环境音、SFX、音乐，与画面同步。

**生成参数不入正文**：分辨率、时长、宽高比在生成页/API 上设，写进 prose 没用（除非该 mode 硬锁）。

### 把"镜头工艺 / 质感"写进这三句（Seedance 2.5 口径）

`storyboard` 已把每镜的景别/焦段/机位/景深/运镜/纵深/光源/构图定好了，这里把它们**落到 Subject→Camera→Visual style 上**，只写加信息的词、不堆砌：

- **Subject + action**：动作要"实"。写"冲刺时外套在气流里向后翻"，不写"奔跑"。给主体一个正在发生的状态。
- **Camera**：直接用景别/角度/运动（push in / orbit / handheld / low angle / rack focus…）。有多个主体时补**哪个**主体、从哪到哪。**别把 Camera 当装饰**——运动匹配动作。
- **Visual style** 用"可观察结果"表述，别用空洞形容词：
  - **纵深机制**：只在 光分离 / 大气透视(保留结构) / 遮挡递退 里选一种表述，或写清楚远近层次；**不写无介质的雾/体积光**。
  - **光源动机**：写明光源类型与来源（如"warm low evening sun from the west, long shadows"；室内写"a window on the left"）。**不写无源"environmental light"**。
  - **质感锚点**：要"皮肤保留毛孔与次表面透明感""真实光学散景、背景结构可辨""影调有胶片肩部滚降、暗部沉实非发灰"——**这是反 AI 廉价感的关键**（对应 Seedance 2.5 对 grain/halation 敏感，写轻描淡写的中性措辞即可）。
  - **五层色彩**：调性给到 palette/饱和度/胶片颗粒/高光，不写"cinematic"。
  - **反廉价化**：可在 prose 里写入"avoid plastic sheen, no CG look, natural skin, coherent background"这类基础项（Seedance 架构不吃长负向清单，写短、写具体）。

> 注意 Seedance 2.5 的边界：它的 "visual style" 追求"加信息、非堆砌"；真实/质感锚点用中性措辞（grain/halation 写 barely-visible，避免触发噪点）。

---

## 参考物：角色 + 排除 + 保真度

只有一个以上的参考物、或参考物挨着文字描述时，**必须声明每个参考物的角色 and 排除**。

```text
@Image 1 defines <subject>'s appearance, clothing, structure, or material>.
Do not use the people in the image.

@Video 1 defines <motion, camera movement, or pacing>.
@Audio 1 defines <character or sound type>'s <voice, dialogue, ambience, or music>.
```

**每条参考物都要保真度分级**（声明多少必须活着）：

| 保真度 | 含义 | 例 |
|---|---|---|
| `full-preserve` | 主体原样出现：脸、体型、服装全保留 | "the jacket and the scarf; hairstyle may change" |
| `partial-preserve` | 指名部分保留，其余自由 | |
| `attribute-transfer` | 属性迁到**另一个命名目标**上 | "apply this fabric's weave/sheen to Mira's coat" |
| `loose-guide` | 只借 mood/色调/能量，不逐字照搬 | |

**规则：**
- **映射活在提示词里**，不在图片标签里——文字标签模型不会用来推断谁是谁。
- 多张同一主体视图必须显式说一句"All four images define one folding desk lamp."否则模型会复制成多个主体。
- **永远别把参考物 handle 放进一个该主体缺席的镜头**。
- 多人映射用"一行一主体"清单式写法，**别写"@Image 1 through 4 define four characters"**（这是经典失败——模型不知道哪张是谁）。

**多人映射卡片式（5 步流程：map → group → profile → select-by-scene）：**
```text
<Character A> corresponds to @Image 1. Use only the appearance, hairstyle, and clothing.
<Character B> corresponds to @Image 2. Use only the appearance, hairstyle, and clothing.
```

---

## 音频与文字 · 括号语法

音乐 / SFX / 对白 / 字幕必须用括号显式标出：

| 内容 | 符号 | 例 |
|---|---|---|
| 音乐 | `()` | `(Soft, rhythmic piano music plays in the background)` |
| 音效 | `<>` | `<A bell rings in the distance>` |
| 对白 | `{}` | `{Hello, welcome back.}` |
| 字幕 | `《》` | `《Chapter One: Departure》` |

**非中文对白——先给语言行，再给台词**：
```text
Dialogue language: authentic Los Angeles English.
The young man says in natural Los Angeles vernacular: {No way, you actually made it.}
```

**两条老规矩：**
- **对白只活在音频从句里**，别塞进动作描述（否则模型把它当行为念出来）。要写就写可见行为（"jaw sets, eyes hold"），并注意"no dialogue"不撤销它——可读文本仍会被要求。
- **纯字幕是项目选择，2.5 认它**。随机字幕和没要求的 BGM 是 2.0 最常被投诉的噪音；要写就当 `**NO BGM**`，而不是 `(no music)`。

---

## 首末帧 / 多 keyframe（是提示词里的陈述，不是 mode）

2.5 平台面没有 start/end-frame 角色，所以首末帧**在正文里声明**：

```text
@Image 1 is the first frame. It defines the opening composition, subject position, pose,
prop state, scene, and camera direction.
@Image 2 is the last frame. It defines the ending composition, subject position, pose,
prop state, scene, and camera direction.
@Image 3 defines <Subject A>'s appearance, clothing, structure, or material. Do not change
the first-frame composition defined by @Image 1 or the last-frame composition defined by @Image 2.

<Describe one continuous action or event>.
The video begins naturally from the first frame defined by @Image 1 and reaches the last
frame defined by @Image 2 after the continuous action.
Between the first and last frames, maintain continuity in character identity, prop structure
and ownership, scene layout, and camera direction.
```

**三条失败源（都可避开）：**
1. 别把一个角色写成"@Image 1 and 2 are the first and last frames"，一图一角色；一个 role sentence 一张图。
2. 首末帧必须**共享同一宽高比**，否则末帧会被拉伸；输出比锁到**首图**；时长仍可设。
3. 补充参考物**只补充其命名属性**，每条都重申"不改变首/末帧构图"。

**多 keyframe 序列（3+ 张有序 stage 图）**：以 "Use @Image 1 through @Image N as keyframes in this order" 开头，然后描述每个 key 状态。独立 keyframe 图比把多帧拼成一张 grid 更可靠。

---

## 长视频 · 分段 staging（不是一段长 prose）

- 每个 stage 只做**一个主要变化** + 一个**明确的 end state**；时间戳是预算，不是逐帧剪辑点。
- 场景被一个变化断开 → 拆成两条提示词，不要塞进一次生成。
- 时间戳仍要**累加到声明时长**（2.5 累加是模型近似，别当逐帧承诺）。

---

## 真人角色 7 槽公式

针对"我的角色看着像 AI"或"像双胞胎"——七个槽，**槽 1 是"角色/身份"，永远不写年龄**：

```text
[Role] [Skin color / skin texture] [Facial details] [Eyes / soul]
[Hairstyle / hair color] [Clothing / clothing texture] [Body type / mood / temperament]
```

- 槽 1 是角色；**写角色、体格、服装、动作，不写年龄**（引擎规则 1 本身 age-blind，内容过滤器见"未成年"字样会收紧）。写"一个在烟雾缭绕的公园里站着的瘦削教官"，而不是"一个 22 岁青年"。
- **skin 带真实纹理**：可见微孔、透光、毛细血管——这一槽最能区分"真人照片"和"AI 渲染"。
- **eyes/soul** 是"眼神在**做什么**"，不是"眼神长什么样"；死鱼眼是第一破绽。

---

## 情绪 · 抽象词 + 可见线索

抽象词（"tense", "warm", "oppressive"）只定方向，剩下交给解读。**必须配可直接看见/听见的线索**——眼部动作、眉弓紧绷、嘴部动作、呼吸、视线方向、手部动作。2–4 个线索就够，列尽五官细节反而适得其反。

```text
The overall emotion shifts from <starting> to <ending>.
After <triggering event>, <subject> first shows <immediate observable reaction>.
Then, <eyes, brows, mouth, breathing, gaze, or hand movement> gradually <changes>.
Finally, <subject> expresses <target emotion> through <restrained or explicit outward behavior>.
```

只有在情绪确实多次变化时，才用 event-triggered multi-stage 形式。

---

## 机位词

- **基础词直接用**：景别（extreme wide / wide / medium / close-up / extreme close-up）、运动（push in / pull out / pan / lateral move / follow / orbit / dive / dolly out / tilt up / handheld shake）、机位（low angle / overhead / first-person）。
- **技法词**（one-take / dolly zoom / aerial / FPV / bullet time / handheld / bounce speed ramp）能用，但有多个主体在场时就补上**哪个主体**、从哪开始、到哪结束。
- **"cinematic/ambitious"这类形容**：保留词，但**翻成可观察结果**。

```text
Cinematography term + target subject + visual change + foreground/background relationship + direction or speed
```
```text
Rack focus: shift focus smoothly from the leaves in the foreground to the person in the background.
The leaves gradually blur while the person’s face changes from soft to sharp.
```

---

## 别过度承诺（这些是硬底线）

- 时间戳只是**分配时间**，不是逐帧剪辑点。别暗示"beat 会精确落在某帧"。
- `video_edit` 提示词会提高关键事件和源对齐的风险，不保证逐帧叠合。
- 多参考物是**按场景选对**，不是一个镜头塞满所有参考物。
- **框级文字准确不承诺**：字幕、公式、标识、产品规格需要预制参考物 + 后期。
- 首末帧生成锁宽高比到首图；比例不匹配末帧会变形。
- `video_extension` 锁宽高比到源；扩展段音量可能略有偏差。
- 边界帧**自然连接**（不是逐像素相同）；要审查接缝两侧 + 整个扩展段。
- 无缝转场追求**视觉/听觉连续性**，不是像素级保留。

---

## 提交前清单

- [ ] Subject + 主要动作/事件 直白说出
- [ ] 每个参考物都写了**用 + 不用**，且每个主体/产品/道具命名并绑定到具体参考物
- [ ] 参考物**按场景选中**，不是一个镜头全上
- [ ] 每个长片 stage 有一个主要变化 + 明确 end state
- [ ] 角色数、服装、道具归属、空间关系全程一致
- [ ] `video_edit` 定义 master、scope、目标数量、preserve list
- [ ] 抽象情绪/机位词配了**可观察线索**
- [ ] 首末帧：一图一 role、宽高比一致、角色锚点未合并
- [ ] 分段：coarse vs fine 先分清，inherit list 已写
- [ ] 自动锁参对 edit / first-last / extension 已尊重
- [ ] extension：boundary frame、motion trend、audio 连续性均查过
- [ ] 无年龄词（引擎规则 1）
- [ ] 用 `python3 scripts/seedance_lint.py --preflight --model seedance_2_5 "<prompt>"` 跑一遍并干净

---

## 输入映射（从哪儿取）

| 提示词部分 | 取自 |
|---|---|
| Subject + action | `treatment` 的 concept + `storyboard` 的镜头内容 |
| Scene & environment | `storyboard` + `product-intake`(使用场景) |
| Visual style | `product-intake`(调性/色板) |
| Camera | `storyboard`(机位/景别) |
| Audio(对白/口播) | `voice`(口播类型 + 台词/VO/字幕) |
| 角色/服装/人设 | `character`(角色卡) |
| 表演/眼神/情绪 | `acting`(objective/beats/eye-life) |

## 输出

一段**完整、可复制**的最终提示词（按上面公式 + 括号语法组装），并**另起一行**给出生成参数（mode / resolution / duration / aspect_ratio）。这是整套 skill 的必需终点。
