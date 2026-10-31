# dsh-skills

个人 DSH 技能合集。每个技能是一个目录，内含 `SKILL.md`。

## 已包含

### github-project-lookup-install

GitHub 项目查找与选择性安装：

- 联网定位项目 → 通读 README → 生成《项目概述》（功能 / 使用方式 / 是否有 skill / 项目类型）
- 先交付概述；用户确认安装方式并指定安装路径后，再 git clone、下载 zip 或按 README 安装
- 安装过程监控：单任务失败或超过 5 分钟时定位原因
- 系统级操作先汇报；结束时给出最优先的下一步
- 每个新项目独立检索，不沿用会话上下文

安装到 DSH：

```powershell
# 项目级（仅当前项目）
Copy-Item -Recurse .\skills\github-project-lookup-install '<项目>\.dsh\skills\'

# 用户级（所有项目）
Copy-Item -Recurse .\skills\github-project-lookup-install "$env:DSH_HOME\skills\"
```

使用：在 DSH 输入 `/github-project-lookup-install <项目名或链接>`。

## 目录结构

```
skills/
  github-project-lookup-install/
    SKILL.md
```
