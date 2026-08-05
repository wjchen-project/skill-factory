# Skill Factory

技能工厂（Skill Factory）是一个用于创建和管理各类 Agent 技能（Skills）的项目。

技能是面向特定任务的标准化指令集合，能够为 Agent 提供专业化的处理流程与执行规范。

## 目录结构

```
skill-factory/
├── AGENTS.md           # 项目说明与技能创建规范
├── README.md           # 本文件
├── .gitignore          # Git 排除规则
└── skills/             # 技能源文件目录
    ├── code-reviewer/
    └── git-commit/
```

## 快速开始

在 `skills/` 目录下创建你的技能目录，并按照 `AGENTS.md` 中定义的规范编写 `SKILL.md`。

### 创建技能的基本步骤

1. 确认技能的用途（purpose）与触发时机（trigger）
2. 在 `skills/<skill-name>/` 下编写 `SKILL.md`
3. 根据需要添加 commands、scripts、references、assets 等资源

## 详细规范

更多信息请参阅 [AGENTS.md](./AGENTS.md)。