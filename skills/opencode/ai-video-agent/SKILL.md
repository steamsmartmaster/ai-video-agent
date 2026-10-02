---
name: ai-video-agent
description: 零配置用当前模型直接制作可播放的 HTML 动画视频——输入主题即产出大纲、逐镜 HTML 动画分镜、SRT 字幕和浏览器播放器，无需服务器、数据库或 API key。Use when 用户要做视频/生成视频/制作宣传片/科普视频/HTML 动画视频/动画分镜/视频草稿，或提到 ai-video-agent。只写脚本和生图清单用 ai-video-script；本 skill 直接产出能播放的成片。
---

# AI 视频制作（模型即智能体）

不启动任何服务、不调用外部 API。当前模型自己完成全部工作：
大纲 → 逐镜 HTML 动画 → 字幕 → 播放器。参考了
[xuanyuanzhifeng/ai-video-agent](https://github.com/xuanyuanzhifeng/ai-video-agent)
的工作流与风格预设（本地克隆：`D:\sss-data\workplace\ai-video-agent`）。

## 项目目录约定

按用户惯例，新项目建在 `D:\sss-data\workplace\<英文短横线项目名>\`：

```
D:\sss-data\workplace\<project>\
├── outline.json          # 大纲（结构见下）
├── frames\
│   ├── frame-01.html     # 每镜一个独立可运行的 HTML
│   ├── frame-02.html
│   └── ...
├── subtitles.srt         # 全片字幕
└── player.html           # 播放器（串联所有分镜 + 字幕 + 朗读旁白）
```

## 流程

### 0. 确认需求（含可选项询问）

**先问清可选项再动手**。用 `question` 工具一次性问（用户在初始请求里
已说明的项跳过；全部已说明则不问，直接开做）：

| 选项 | 问题措辞 | 选项值 | 默认 |
| --- | --- | --- | --- |
| 主题 | 视频主题/脚本是什么？ | 自由输入 | 必问（无默认） |
| 风格 | 选用哪种视觉风格？ | `cyber-clean` 科技博主（深色青色霓虹·测评讲解）/ `terminal-matrix` 黑客风（Matrix 绿终端·极客技术）/ `warm-story` 暖色系（奶油米色衬线·人文故事） | cyber-clean |
| 字幕 | 视频中是否显示旁白字幕？ | 显示 / 不显示 | 显示 |
| 朗读 | 是否用浏览器语音朗读旁白？ | 朗读 / 静音播放 | 朗读 |
| 时长 | 目标时长？ | 30 秒内 / 60-90 秒 / 3 分钟左右 | 60-90 秒 |

- 风格问题的 description 里附上 `references/styles.md` 对应风格的一句话
  气质描述，帮助用户决策
- 本 skill 只产出 HTML 动画模式；上游的"图片轮播模式"需要外部生图 API，
  不在零配置范围内——用户主动要图片模式时说明这一点
- 选项写入 outline.json 的 `config` 字段（见下），后续步骤和播放器都
  以它为准
- 每步完成后简短汇报再进下一步

### 1. 生成大纲 → outline.json

规则（源自上游 `OUTLINE_SYSTEM_HTML` 提示词）：

- 先写 60-180 秒的逐字旁白稿（中文、口语化、有节奏感），再拆成
  6-30 个分镜
- 每镜旁白 15-60 字；节奏：开头 1-2 镜引入 → 中间展开 → 结尾 1-2 镜收束
- 每镜的 `htmlPrompt` 写清：展示什么元素、做什么动画（淡入/缩放/计数/
  路径动画等）、动效节奏、配色倾向

```json
{
  "title": "不超过 20 字",
  "totalDuration": 75,
  "style": "cyber-clean",
  "config": {
    "style": "cyber-clean",
    "subtitles": true,
    "speak": true,
    "targetSeconds": 75
  },
  "frames": [
    {
      "id": "frame-01",
      "index": 1,
      "title": "不超过 15 字",
      "narration": "分镜旁白（中文）",
      "htmlPrompt": "网页动画描述",
      "duration": 6
    }
  ]
}
```

`duration` 按旁白估算：`clamp(ceil(字数 / 4) + 1, 4, 12)` 秒
（中文约 4 字/秒），全片 `totalDuration` 取各镜之和。

### 2. 逐镜生成 HTML 动画 → frames/frame-NN.html

每镜一个**完整独立可运行**的 HTML（含 `<!DOCTYPE html>`），规则源自上游
`FRAME_HTML_SYSTEM`：

- 内联 CSS/JS，可用 SVG/Canvas；**禁止任何外部资源**（CDN、外链图片、
  外链字体）、禁止 alert/prompt/confirm
- 画布 16:9，1280×720；动画时长与该镜 `duration` 匹配，播完静止或循环
- 文字要大：标题 ≥ 48px，正文 ≥ 24px，对比度强
- **画面中绝不出现旁白文字/句子/段落**——旁白由播放器字幕层和朗读呈现；
  画面只放数字、单词、图标、装饰、几何元素
- 风格严格按 `references/styles.md` 中选定风格的完整提示词执行
- 与上一镜视觉一致（配色、字体、版式语言延续）

生成时把上一镜 HTML 读出来作参照，保持连贯。多镜可并行写文件。

### 3. 字幕 → subtitles.srt（始终生成，开关只影响播放显示）

- 把每镜旁白按标点拆成 10-20 字的 cue，按字数比例分配该镜 `duration`
- 时间轴按分镜顺序累加，格式：

```
1
00:00:00,000 --> 00:00:03,250
第一句旁白
```

- 用户选择"不显示字幕"：仍生成 subtitles.srt（方便日后用），但
  player.html 中 `SHOW_SUBTITLES = false`，播放时字幕层隐藏

### 4. 播放器 → player.html

复制 `references/player-template.html` 到项目根，替换模板中的三处占位：

```js
const FRAMES = [
  { file: "frames/frame-01.html", duration: 6, narration: "该镜旁白" },
  // ...
];
const TITLE = "视频标题";
const SHOW_SUBTITLES = true;   // ← config.subtitles
const SPEECH = true;           // ← config.speak
```

播放器功能（已内置，勿改逻辑）：按序加载分镜 iframe、底部大字字幕
（`SHOW_SUBTITLES=false` 时隐藏）、Web Speech API（zh-CN）朗读每镜旁白
（`SPEECH=false` 时静音）、进度指示、点击开始、空格暂停、`R` 重播、
`←/→` 切换分镜。file:// 直接打开即可用。

### 5. 交付

1. `Start-Process player.html` 打开播放器，让用户点一下开始预览
2. 汇报：标题、镜数、总时长、所选风格、字幕/朗读开关状态、文件清单
3. 需要 MP4 时告知零配置路径：播放器全屏后用 Win+G（Xbox Game Bar）
   录屏导出；`SPEECH=true` 时旁白经浏览器朗读会被一并录进去，
   `SPEECH=false` 则录出无声画面需后期配音

## 校验

- 每个 frame-NN.html 必须通过：`node -e "const s=require('fs').readFileSync('<file>','utf8'); if(!/<!DOCTYPE html>/i.test(s)) throw new Error('not standalone'); if(/https?:\/\//.test(s)) throw new Error('external resource')"`
- player.html 里 `FRAMES.length` 必须等于 outline.json 帧数
- player.html 的 `SHOW_SUBTITLES`/`SPEECH` 必须与 outline.json 的
  `config.subtitles`/`config.speak` 一致
- 字幕最后一条结束时间 ≈ totalDuration

## 修改与重生成

- 改某一镜：只重写那个 `frame-NN.html`，保持风格提示词不变
- 改旁白/大纲：同步更新 outline.json、subtitles.srt、player.html 的
  FRAMES 和 duration
- 换风格：全部分镜按新风格提示词重写
