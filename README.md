# PPT-Master Skill

专业 HTML 演示文稿生成器，默认采用泰康保险集团主题模板。

## 安装

Skill 位于 `~/.claude/skills/ppt-master/`，Claude Code 自动发现并加载。

## 触发方式

在任意项目中输入以下关键词即可触发：
- 演示文稿、PPT、幻灯片、presentation
- 制作演示、生成PPT、slides
- 做一份、帮我做一个演示

或直接使用：`/ppt-master`

## 功能

| 步骤 | 指令 | 说明 |
|------|------|------|
| 需求收集 | 自动 | 4个关键问题引导用户明确需求 |
| 内容大纲 | 自动 | 分析内容，生成结构化大纲 |
| 分镜方案 | `/分镜` | 生成 `{主题}-Slides.md` 分镜文档 |
| 生成演示 | `/开发` | 生成 `{主题}.html` + `{主题}-README.md` |

## 技术栈

- HTML5 + CSS3（泰康保险集团主题）
- GSAP 3.12（Timeline 动画编排）
- ECharts 5.5（数据可视化）
- html2canvas + jsPDF（PDF 导出）
- html2canvas + PptxGenJS（PPTX 混合模式导出）

## 文件结构

```
ppt-master/
├── SKILL.md                      ← 主指令文件
├── README.md                     ← 本文件
├── references/
│   ├── rules.md                  ← 7条强制规则（CDN/动画/导出/高度/图表/自检）
│   └── export-code.md            ← PDF/PPTX 导出完整实现代码
└── assets/
    └── taikang-logo-b64.txt      ← 泰康 Logo base64（24,244字符）
```

## 输出文件

生成在用户当前工作区根目录：
- `{主题}.html` — 可运行的演示文稿
- `{主题}-Slides.md` — 分镜方案文档
- `{主题}-README.md` — 项目说明

## 设计规范

- 模板：泰康保险集团（Taikang Insurance Group）主题
- 配色：6色系统（accent1 #EE7700 为主强调色）
- 字体：微软雅黑 + Arial
- 舞台：16:9，1200×675px，白色背景
- 动画：GSAP Timeline，tl.set() + tl.to() 模式
- 备选：Dark + Grid 科技美学（用户明确要求时启用）

## 版本

v1.0.0 — 从 PPT-Master 项目 CLAUDE.md 提取并封装为独立 Skill。
