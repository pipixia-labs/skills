# Boss 直聘简历管理工具

## 项目说明
这是一个基于 playwright-cli 的 Boss 直聘(www.zhipin.com)招聘端自动化工具，用于管理职位和下载候选人简历。

## 目录结构
- `playwright-cli/` — playwright-cli 工具（已 npm install）
- `skills/boss-zhipin/SKILL.md` — Boss 直聘自动化 Skill
- `auth/zhipin-auth.json` — 保存的登录态（不要提交到 git）
- `output/` — 下载的简历按职位分文件夹存放
- `processed.json` — 候选人处理状态跟踪（messaged/downloaded）
- `.playwright-cli/` — playwright-cli 工作目录

## 使用方式
- `/boss-zhipin` — 调用 Boss 直聘 Skill
- CLI 命令统一使用: `node playwright-cli/playwright-cli.js -s=boss <command>`
- 浏览器会话名固定为 `boss`

## 常用指令
- "帮我处理所有候选人" — 自动沟通新候选人 + 下载已收到的简历
- "帮我下载所有新简历" — 只下载已收到但未下载的简历
- "查看我的职位列表" — 列出所有职位及统计
- "查看处理状态" — 显示 processed.json 中的候选人状态

## 权限说明
Skill 需要 Bash 权限执行 playwright-cli 命令，已在 settings.local.json 中配置：
```
Bash(node playwright-cli/playwright-cli.js:*)
```
