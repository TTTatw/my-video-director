# My Video Director

一个**导演级视频脚本生成器**（模块化 skill 包）。拿一张产品图片 → 输出分析（产品画像、Big Idea、角色卡、分镜、表演层、口播类型）**＋必需结尾的最终可执行生成提示词**。**⑦ 是必须交付**——你拿到就能直接复制去生成。

## 结构

```
my-video-director/
├── SKILL.md                 ← 根路由器（触发 + ①~⑥ 分析 + ⑦ 交付格式）
├── README.md                ← 本文件
├── references/
│   ├── routing.md           ← 路径判定 + 视频类型字典 + 角色数判定
│   ├── model-dialects.md    ← 各模型提示词方言 + 路由表 + 生成参数对照
│   └── short-video-craft.md ← 短视频工艺层：3秒钩子/留人/心理杠杆/反转/服装美妆心理
├── skills/
│   ├── product-intake/      ← ① 产品画像（产品图片 → 卖点/人群/调性）
│   ├── treatment/           ← ② Big Idea + 创意方向 + 视频类型选择
│   ├── character/           ← ③ 单人/多人角色卡（长相+穿搭+人设）
│   ├── storyboard/          ← ④ 逐镜头分镜（镜头规格：景别/焦段/机位/景深/运镜/纵深三选一/光源动机/构图 + 电影工艺规则）
│   ├── acting/              ← ⑤ 表演层（objective/beats/subtext/eye-life + master profile）
│   ├── voice/               ← ⑥ 口播类型（口播/无口播+字幕/背景配音/ASMR）
│   ├── templates/           ← 10 种 UGC 模板 + 概念种子（模板路径）
│   └── final-prompt/        ← ⑦ **最终可执行提示词〔必需终点〕· 按模型自动切方言（Seedance 2.5 / 2.0 / Veo 3.1）**
└── examples/
    ├── portable-coffee-tumbler.md   ← 完整示例(①~⑥ + ⑦)
    └── black-dress-final-prompt.md  ← 完整示例(①~⑥ + ⑦·Seedance 2.5)
```

## 两条路径（都是全品质，无"精简"）

- **导演路径(creative)**：洞察 + Big Idea 起，走完整 ①~⑦。默认。
- **模板路径(template)**：选模板+概念种子起，**仍走完整 ①~⑦**。差别只在创意从哪来。
- **没有"精简/广告版"**：⑦ 永远按**对应模型的方言**做满（用户提哪个模型就用哪个方言），表演层永远做满。

## 用法

把产品图片（+ 需求）给运行它的 agent。例如：
> "这张咖啡杯，做一条 20s 口播 UGC，单人、带字幕。"

它输出六段完整拍摄案。也可单独点名某层（"只要角色卡"/"只要分镜"）。

## 实现说明（提取自哪些开源）

本包是把已有的 Higgsfield / Claude skills 里**散落的导演/角色/表演/分镜方法**抽取、重组，并补齐开源所缺的**路由表**与**视频类型字典**：

| 层 | 抽取/借鉴自 |
|---|---|
| 表演层 acting | [OSideMedia/higgsfield-ai-prompt-skill](https://github.com/OSideMedia/higgsfield-ai-prompt-skill) `higgsfield-acting`(objective/beats/subtext/eye-life/status) |
| 角色&一致性 | 同仓库 `higgsfield-character-design` + `higgsfield-soul` |
| 分镜/状态 | 同仓库 `higgsfield-seedance`(状态非过程) / `seedance-2-5`(分镜) |
| 口播/音频 | 同仓库 `higgsfield-audio` + [higgsfield-ai/skills](https://github.com/higgsfield-ai/skills) `seed_audio`/Marketing Studio |
| UGC 模板 | 同仓库 `higgsfield-content-factory`(5 格式) + 官方 Marketing Studio 9 mode |
| 产品画像 | 官方 `higgsfield-generate`(`products fetch`) + content-factory 识品 |

**开源没有、这里新写的**：① 需求→路径路由表；② 视频类型字典（口播/无口播+字幕/背景配音/ASMR/对话）；③ 单人 vs 多人的角色产出结构。

## 安装

作为一个 Claude/Agent skill：把 `my-video-director/` 目录放进 `~/.claude/skills/`（或对应 agent 的 skills 目录），或用 `npx skills add` 指向本目录。详见各 agent 的 skill 安装方式。

> 注：本包只有"导演案"输出，不含任何 Higgsfield 账户/API 依赖，纯文本 skill。
