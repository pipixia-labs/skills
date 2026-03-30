---
name: boss-zhipin
description: 管理 Boss 直聘职位和简历。自动沟通新候选人、求简历、下载已收到的附件简历到对应职位文件夹。通过 processed.json 跟踪状态避免重复操作。
allowed-tools: Bash(playwright-cli:*) Bash(node:*) Bash(mkdir:*) Bash(mv:*) Bash(ls:*) Bash(cat:*) Bash(sleep:*)
---

# Boss 直聘简历管理

通过系统中已经安装好的 `playwright-cli` 自动化管理 Boss 直聘(www.zhipin.com)招聘端的职位和简历下载。

## 约定

- CLI 命令: `playwright-cli`
- 浏览器会话名: `boss`
- 登录态文件: `auth/zhipin-auth.json`
- 简历输出目录: `output/<职位名>/`
- 状态跟踪文件: `processed.json`
- 下载目录: `.playwright-cli/`

## 前置条件

- 系统里已经安装官方 `playwright-cli`
- 已安装需要使用的浏览器，例如 Chrome
- 当前工作目录中允许生成 `auth/`、`output/`、`.playwright-cli/` 和 `processed.json`

参考准备命令：

```bash
npm install -g @playwright/cli@latest
playwright-cli install-browser --browser chrome
```

## 状态跟踪 (processed.json)

用 `processed.json` 记录每个候选人的处理状态，避免重复沟通和下载。文件格式：

```json
{
  "candidates": {
    "候选人姓名_职位名": {
      "name": "候选人姓名",
      "job": "职位名",
      "status": "messaged|downloaded",
      "messagedAt": "2026-03-30T09:00:00Z",
      "downloadedAt": "2026-03-30T10:00:00Z",
      "file": "output/职位名/候选人姓名.pdf"
    }
  }
}
```

状态说明：

- `messaged`：已发送沟通消息并点过“求简历”，等待对方发送简历
- `downloaded`：已下载简历，流程完成

每次操作前先读取 `processed.json`，操作后立即更新。

## 命令

### 1. 登录 (boss-login)

首次使用或登录态过期时，需要手动登录。

```bash
# 关闭旧会话（如果有的话）
playwright-cli -s=boss close

# 以 headed 模式打开浏览器，让用户手动登录
playwright-cli -s=boss open "https://www.zhipin.com/web/user/?ka=header-login" --headed --browser=chrome --persistent

# 用户登录完成后，保存登录态
playwright-cli -s=boss state-save auth/zhipin-auth.json
```

注意：Boss 直聘使用手机验证码或微信扫码登录，必须用户手动完成。

### 2. 恢复会话 (boss-session)

每次操作前确保浏览器会话存在且已登录：

```bash
# 打开浏览器并加载登录态
playwright-cli -s=boss open "https://www.zhipin.com/web/chat/index" --browser=chrome --persistent
playwright-cli -s=boss state-load auth/zhipin-auth.json
playwright-cli -s=boss goto "https://www.zhipin.com/web/chat/index"
```

如果被跳转到登录页，说明登录态过期，需要重新执行 `boss-login`。

### 3. 查看职位列表 (boss-jobs)

通过 API 获取所有职位：

```bash
playwright-cli -s=boss eval "async () => {
  const r = await fetch('/wapi/zpjob/job/data/list?position=0&type=0&searchStr=&comId=&tagIdStr=&page=1');
  const data = await r.json();
  return data.zpData.data.map(j => ({
    id: j.encryptId,
    name: j.jobName,
    city: j.locationName,
    salary: j.salaryDesc,
    type: j.jobTypeName,
    status: j.jobStatus === 0 ? '开放中' : '已关闭',
    interested: j.interestCount,
    chatted: j.concatCount,
    viewed: j.viewCount
  }));
}"
```

注意：如果 `hasMore: true`，需要翻页 `page=2, 3...` 获取全部职位。展示结果时用表格格式。

### 4. 自动处理所有候选人 (boss-process)

这是核心命令。一次性完成：沟通新候选人和下载已收到的简历。

#### 整体流程

```text
读取 processed.json
  ↓
进入沟通页，获取聊天列表快照
  ↓
遍历每个聊天项：
  ├─ 已在 processed.json 且 status=downloaded → 跳过
  ├─ 聊天最后消息包含“附件简历” → 执行下载简历
  └─ 新候选人（不在 processed.json） → 执行自动沟通
  ↓
更新 processed.json
```

#### 第一步：读取状态并进入沟通页

```bash
cat processed.json
playwright-cli -s=boss goto "https://www.zhipin.com/web/chat/index"
sleep 3
playwright-cli -s=boss snapshot
```

#### 第二步：分析聊天列表

从快照中解析每个聊天项，提取：

- 候选人姓名
- 职位名
- 最后一条消息内容
- 聊天项可点击 ref

将每个候选人与 `processed.json` 比对，分为三类：

1. 已下载：`status=downloaded`，跳过
2. 有附件简历：最后消息包含“附件简历”，去下载
3. 新候选人：不在 `processed.json` 中，去沟通

#### 第三步A：自动沟通新候选人

```bash
# 1. 点击聊天项进入对话
playwright-cli -s=boss click <聊天项ref>
sleep 2

# 2. 获取快照，确认进入了对话
playwright-cli -s=boss snapshot

# 3. 优先尝试点击输入区域后直接 type
playwright-cli -s=boss click <输入区域ref>
playwright-cli -s=boss type "您好，感谢您对这个岗位的关注！方便发一份您的简历过来吗？"

# 4. 点击发送按钮
playwright-cli -s=boss click <发送按钮ref>
sleep 1

# 5. 获取新快照，点击“求简历”按钮
playwright-cli -s=boss snapshot
playwright-cli -s=boss click <求简历按钮ref>
sleep 1
```

然后更新 `processed.json`，将该候选人标记为 `messaged`。

补充说明：

- 输入框在聊天详情区底部，很多时候直接 `fill` 不如先 `click` 再 `type` 稳定
- “求简历”按钮在底部工具栏
- “发送”按钮在输入框右侧
- 发送前先检查是否已经发过同类消息，避免重复骚扰候选人

#### 第三步B：下载已收到的简历

```bash
# 1. 点击聊天项进入对话
playwright-cli -s=boss click <聊天项ref>
sleep 2

# 2. 获取快照，找到“点击预览附件简历”按钮
playwright-cli -s=boss snapshot

# 3. 点击预览按钮
playwright-cli -s=boss click <"点击预览附件简历"的ref>
sleep 3

# 4. 获取快照，找到下载按钮
playwright-cli -s=boss snapshot

# 5. 点击下载按钮
playwright-cli -s=boss click <下载图标ref>
sleep 2

# 6. 检查网络日志确认下载完成，获取文件名
playwright-cli -s=boss network

# 7. 创建职位文件夹并移动文件
mkdir -p "output/<职位名>"
mv ".playwright-cli/<下载的文件名>.pdf" "output/<职位名>/<候选人姓名>.pdf"

# 8. 关闭预览
playwright-cli -s=boss press Escape
sleep 1
```

然后更新 `processed.json`，将该候选人标记为 `downloaded`。

关于下载按钮的定位：

- 点击“点击预览附件简历”后，页面底部会出现预览层
- 预览层顶部通常有三个图标按钮：全屏、打印、下载
- 下载一般是第三个
- `network` 日志中通常会出现下载文件的路径，方便确认实际文件名

#### 第四步：更新 processed.json

每处理完一个候选人，立即用 Node.js 更新 `processed.json`：

```bash
# 标记为已发消息
node -e "
const fs = require('fs');
const f = 'processed.json';
const d = fs.existsSync(f) ? JSON.parse(fs.readFileSync(f)) : {candidates:{}};
const key = '候选人姓名_职位名';
d.candidates[key] = {...(d.candidates[key]||{}), name:'候选人姓名', job:'职位名', status:'messaged', messagedAt:new Date().toISOString()};
fs.writeFileSync(f, JSON.stringify(d, null, 2));
console.log('Updated:', key, '-> messaged');
"

# 标记为已下载
node -e "
const fs = require('fs');
const f = 'processed.json';
const d = fs.existsSync(f) ? JSON.parse(fs.readFileSync(f)) : {candidates:{}};
const key = '候选人姓名_职位名';
d.candidates[key] = {...(d.candidates[key]||{}), name:'候选人姓名', job:'职位名', status:'downloaded', downloadedAt:new Date().toISOString(), file:'output/职位名/候选人姓名.pdf'};
fs.writeFileSync(f, JSON.stringify(d, null, 2));
console.log('Updated:', key, '-> downloaded');
"
```

### 5. 单独下载简历 (boss-download)

只下载已收到但未下载的简历，不发送新消息。

按照第三步B的流程，只处理聊天列表中包含“附件简历”且 `processed.json` 中状态不是 `downloaded` 的候选人。

### 6. 单独沟通新候选人 (boss-greet)

只对新候选人发送沟通消息，不下载简历。

按照第三步A的流程，只处理不在 `processed.json` 中的新候选人。

### 7. 查看处理状态 (boss-status)

```bash
cat processed.json | node -e "
const d=JSON.parse(require('fs').readFileSync(0));
const c=Object.values(d.candidates);
console.log('总计:', c.length);
console.log('已沟通待简历:', c.filter(x=>x.status==='messaged').length);
console.log('已下载简历:', c.filter(x=>x.status==='downloaded').length);
console.table(c.map(x=>({姓名:x.name, 职位:x.job, 状态:x.status, 文件:x.file||'-'})));
"
```

## 页面结构参考

### 沟通页面 (`/web/chat/index`)

```text
左侧: 聊天列表
  listitem:
    generic [cursor=pointer]:
      generic: "17:29"
      generic:
        generic "候选人姓名"
        generic "职位名"
      generic: "最后一条消息内容"

右侧: 聊天详情
  消息列表:
    ...历史消息...
    generic:
      heading "附件简历-姓名-职位.pdf"
      generic [cursor=pointer]: "点击预览附件简历"
  底部工具栏:
    generic [cursor=pointer]: "求简历"
    generic [cursor=pointer]: "换电话"
    generic [cursor=pointer]: "换微信"
    generic: "发送"

简历预览层:
  generic:
    img [cursor=pointer]
    img [cursor=pointer]
    img [cursor=pointer]
  iframe: 简历内容
```

## 关键 API

| API | 用途 |
| --- | --- |
| `/wapi/zpjob/job/data/list?position=0&type=0&page=1` | 获取职位列表 |
| `/wapi/zprelation/interaction/bossGetGeek?jobid=<id>&page=1&tag=4&status=4` | 获取某职位的感兴趣候选人 |
| `/wapi/zpchat/boss/historyMsg?src=0&gid=<uid>&maxMsgId=0&c=20&page=1` | 获取聊天历史消息 |
| `/wflow/zpgeek/download/preview4boss/<geekId>` | 下载简历文件 |

## 注意事项

- 登录态有效期有限，如果被跳转到登录页，需要重新执行 `boss-login`
- 每次 `snapshot` 后 ref 都可能变化，必须用最新快照中的 ref
- 页面存在 iframe 时，ref 可能带 iframe 前缀
- 页面和简历预览都需要等待加载完成，通常要 `sleep 2` 到 `sleep 3`
- `playwright-cli` 默认会将下载文件放到 `.playwright-cli/`
- 操作不要太快，尽量每步间隔 1 到 2 秒，防止触发反爬
- 如遇弹窗，用 `dialog-accept` 或 `dialog-dismiss`
- 聊天列表默认只加载一部分，必要时用 `mousewheel 0 500` 滚动加载更多
