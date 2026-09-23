# stock-news-monitor · 股票新闻自动化监控系统（AI 自举包）

AI 驱动的股票新闻监控系统：新闻采集 → 短/中/长期三维筛选 → 「日期+核心命题」日报 → 周报/月报递进综合 → Obsidian 双链知识图谱。内置**命题登记打分回路**（防过拟合）、**冷热分离**与**反方论点必填**（防讨好）、**覆盖矩阵/盲区引擎**（防视角窄化）。

## 快速部署（给任何 AI 的一段话）

> 把本仓库下载/克隆到本地（GitHub 直连失败时，按 `BOOTSTRAP.md` 第 0.5 节的三级降级方案：镜像 clone → 逐文件抓取），然后读其中的 `BOOTSTRAP.md`，严格按章节顺序执行：检测环境（缺 Obsidian 则自动安装，数据源不可用则降级）→ 发现或创建 Obsidian vault → 复制 `config.template.yaml` 为 `config.yaml` 并填写标的 → 初始化骨架 → 注册定时任务。全程无需人工写代码，仅在配置路径和选择标的时需要确认。

### 以标准 Agent Skill 安装

本仓库兼容 [Agent Skills 规范](https://agentskills.io)（根目录含 `SKILL.md`），可直接装入 skills 兼容的 agent（Claude Code / Codex / OpenCode / WorkBuddy 等）：

```
npx skills add https://github.com/tinker1000a-design/stock-news-monitor
```

## 仓库结构

```
├── BOOTSTRAP.md          # AI 执行手册（唯一入口，任何接手 AI 从这里开始）
├── config.template.yaml  # 配置模板（部署时复制为 config.yaml）
├── 设计文档.md            # 设计原理：分层报告、命题打分、盲区引擎
└── templates/
    ├── 日报模板.md
    ├── 周报模板.md
    ├── 个股中枢模板.md
    └── 命题登记模板.md
```

## 核心设计

| 机制 | 防什么 |
|------|--------|
| 命题登记 + 周报到期打分 | 只记对的、忘错的 → 观点漂移 |
| 冷热分离（偏好只管排序，结论走冷通道） | AI 讨好用户 |
| 反方最强论点必填 + 措辞黑名单 | 单边叙事 |
| 覆盖矩阵 + 证据漏斗统计 | 盲区与信息噪声 |
| `.monitor/state.json` 增量钩子 | 重复处理 / 漏处理笔记 |
| 全配置驱动（config.yaml 唯一配置源） | 硬编码，无法扩容到任意标的 |

## 安全约定

- 真实 `config.yaml` 与 `.monitor/` 不入库（见 .gitignore），含个人路径与持仓信息。
- 报告生成禁止使用模型记忆补数据，数据源断供时降级并明示。
