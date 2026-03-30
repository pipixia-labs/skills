# Boss 直聘简历管理工具

## 项目说明

这是一个依赖官方 `playwright-cli` 命令的 Boss 直聘招聘端自动化 skill，用于管理职位、自动沟通候选人，以及下载候选人简历。

## 目录结构

- `skills/boss-zhipin/SKILL.md` — Boss 直聘自动化 Skill 主说明
- `README.md` — 安装前置条件、依赖和常见用法
- `auth/zhipin-auth.json` — 保存的登录态，不要提交到 git
- `output/` — 下载的简历按职位分文件夹存放
- `processed.json` — 候选人处理状态跟踪
- `.playwright-cli/` — `playwright-cli` 运行时工作目录和下载目录

## 使用方式

- `/boss-zhipin` — 调用 Boss 直聘 Skill
- CLI 统一使用 `playwright-cli -s=boss <command>`
- 浏览器会话名固定为 `boss`

## 前置依赖

- 已安装 `Node.js`
- 已全局安装 `@playwright/cli`
- 已安装需要使用的浏览器，例如：
  - `playwright-cli install-browser --browser chrome`

## 常用任务

- “帮我处理所有候选人” — 自动沟通新候选人并下载已收到的简历
- “帮我下载所有新简历” — 只下载已收到但未下载的简历
- “查看我的职位列表” — 列出职位及统计信息
- “查看处理状态” — 显示 `processed.json` 中的处理结果

## 权限说明

Skill 需要 Bash 权限来执行 `playwright-cli`、`node` 以及基础文件操作命令，本地配置见 `.claude/settings.local.json`。
