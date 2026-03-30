# boss-zhipin-skill

一个用于 Boss 直聘招聘端自动化处理的 skill。它依赖系统中已经安装好的官方 `playwright-cli`，不再在仓库里内置 `playwright-cli` 源码。

## 能做什么

- 查看 Boss 直聘上的职位列表
- 自动给新候选人发送沟通消息并点“求简历”
- 下载候选人已经发送的附件简历
- 用 `processed.json` 跟踪处理状态，避免重复操作

## 前置依赖

在使用这个 skill 之前，需要先准备好下面这些东西：

1. `Node.js`
2. 官方 `playwright-cli`
3. 至少一个已安装的浏览器，通常建议 Chrome

参考命令：

```bash
npm install -g @playwright/cli@latest
playwright-cli install-browser --browser chrome
```

## 依赖说明

这个 skill 的运行依赖主要有两类：

- 外部工具依赖
  - `playwright-cli`
  - `node`
- skill 依赖
  - 不再内置或 vendoring `playwright-cli` skill
  - 当前 `boss-zhipin` 的核心流程已经直接写在 `skills/boss-zhipin/SKILL.md` 中，所以只要系统里能执行 `playwright-cli`，就可以运行

简单说，这个仓库现在依赖的是“外部已安装的官方 CLI”，不是“仓库内带一份 Playwright skill”。

## 目录约定

- `skills/boss-zhipin/SKILL.md`
  - 主 skill 文档，定义操作流程和命令约定
- `auth/zhipin-auth.json`
  - 登录态文件
- `output/<职位名>/`
  - 简历下载后的存放目录
- `processed.json`
  - 候选人处理状态文件
- `.playwright-cli/`
  - `playwright-cli` 的运行目录和下载缓存目录

这些运行时文件都不应该提交到 git。

## 快速开始

### 1. 首次登录

```bash
playwright-cli -s=boss close
playwright-cli -s=boss open "https://www.zhipin.com/web/user/?ka=header-login" --headed --browser=chrome --persistent
playwright-cli -s=boss state-save auth/zhipin-auth.json
```

说明：

- Boss 直聘通常需要手机验证码或扫码登录，这一步需要人工完成
- 登录完成后再执行 `state-save`

### 2. 恢复会话

```bash
playwright-cli -s=boss open "https://www.zhipin.com/web/chat/index" --browser=chrome --persistent
playwright-cli -s=boss state-load auth/zhipin-auth.json
playwright-cli -s=boss goto "https://www.zhipin.com/web/chat/index"
```

如果页面被跳回登录页，说明登录态过期了，需要重新登录。

### 3. 常见使用方式

- 处理全部候选人
  - 读取 `processed.json`
  - 遍历聊天列表
  - 给新候选人发消息并点“求简历”
  - 下载已经收到的附件简历
- 只下载新简历
  - 只处理最后消息包含“附件简历”的候选人
- 查看处理状态
  - 从 `processed.json` 汇总已沟通和已下载人数

完整的操作细节请看 `skills/boss-zhipin/SKILL.md`。

## 使用建议

- 每次操作前先确认登录态是否有效
- 每次 `snapshot` 之后都要用最新的 ref，不要复用旧 ref
- 操作节奏不要太快，适当 `sleep 1` 到 `sleep 3`
- 下载完成后及时把文件从 `.playwright-cli/` 移动到 `output/<职位名>/`

## 本地权限配置

如果在 Claude/Codex 类环境中运行，通常需要允许下面这些命令：

- `playwright-cli`
- `node`
- `mkdir`
- `mv`
- `ls`
- `cat`
- `sleep`

当前示例配置在 `.claude/settings.local.json`。
