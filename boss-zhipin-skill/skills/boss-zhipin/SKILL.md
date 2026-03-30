---
name: boss-zhipin
description: 管理 Boss 直聘职位和简历。自动沟通新候选人、求简历、下载已收到的附件简历到对应职位文件夹。通过 processed.json 跟踪状态避免重复操作。
allowed-tools: Bash(playwright-cli:*) Bash(node:*) Bash(mkdir:*) Bash(mv:*) Bash(cp:*) Bash(ls:*) Bash(cat:*) Bash(sleep:*)
---

# Boss 直聘简历管理

通过 playwright-cli 自动化管理 Boss 直聘(www.zhipin.com)招聘端的职位和简历下载。

## 约定

- CLI 路径: `node playwright-cli/playwright-cli.js`
- 浏览器会话名: `boss`
- 登录态文件: `auth/zhipin-auth.json`
- 简历输出目录: `output/<职位名>/`
- 状态跟踪文件: `processed.json`
- playwright-cli 下载目录: `.playwright-cli/`

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
- `messaged` — 已发送沟通消息+求简历，等待对方发送简历
- `downloaded` — 已下载简历，流程完成

每次操作前先读取 processed.json，操作后立即更新。

## 命令

### 1. 登录 (boss-login)

首次使用或登录态过期时，需要手动登录。

```bash
# 关闭旧会话（如果有的话）
node playwright-cli/playwright-cli.js -s=boss close

# 以 headed 模式打开浏览器，让用户手动登录
node playwright-cli/playwright-cli.js -s=boss open "https://www.zhipin.com/web/user/?ka=header-login" --headed --browser=chrome --persistent

# 用户登录完成后，保存登录态
node playwright-cli/playwright-cli.js -s=boss state-save auth/zhipin-auth.json
```

注意：Boss 直聘使用手机验证码/微信扫码登录，必须用户手动完成。

### 2. 恢复会话 (boss-session)

每次操作前确保浏览器会话存在且已登录：

```bash
# 打开浏览器并加载登录态
node playwright-cli/playwright-cli.js -s=boss open "https://www.zhipin.com/web/chat/index" --browser=chrome --persistent
node playwright-cli/playwright-cli.js -s=boss state-load auth/zhipin-auth.json
node playwright-cli/playwright-cli.js -s=boss goto "https://www.zhipin.com/web/chat/index"
```

如果被跳转到登录页（URL 包含 `/web/user/`），说明登录态过期，需要重新执行 boss-login。

### 3. 查看职位列表 (boss-jobs)

通过 API 获取所有职位：

```bash
node playwright-cli/playwright-cli.js -s=boss eval "async () => {
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

**这是核心命令。** 一次性完成：沟通新候选人 + 下载已收到的简历。

#### 整体流程

```
读取 processed.json
  ↓
进入沟通页，获取聊天列表快照
  ↓
遍历每个聊天项：
  ├─ 已在 processed.json 且 status=downloaded → 跳过
  ├─ 聊天最后消息包含"附件简历" → 执行【下载简历】
  └─ 新候选人（不在 processed.json） → 执行【自动沟通】
  ↓
更新 processed.json
```

#### 第一步：读取状态 + 进入沟通页

```bash
# 读取已处理状态
cat processed.json

# 进入沟通页
node playwright-cli/playwright-cli.js -s=boss goto "https://www.zhipin.com/web/chat/index"
sleep 3
node playwright-cli/playwright-cli.js -s=boss snapshot
```

#### 第二步：分析聊天列表

从快照中解析每个聊天项，提取：
- 候选人姓名（generic 元素中带引号的名字，如 `generic "王宇"`)
- 职位名（紧跟在姓名后面的 generic 元素，如 `generic "web前端实习"`)
- 最后一条消息内容
- 聊天项的可点击 ref

将每个候选人与 processed.json 比对，分为三类：
1. **已下载** (status=downloaded) → 跳过
2. **有附件简历** (最后消息包含"附件简历") → 去下载
3. **新候选人** (不在 processed.json) → 去沟通

#### 第三步A：自动沟通新候选人

对于每个新候选人：

```bash
# 1. 点击聊天项进入对话
node playwright-cli/playwright-cli.js -s=boss click <聊天项ref>
sleep 2

# 2. 获取快照，确认进入了对话
node playwright-cli/playwright-cli.js -s=boss snapshot

# 3. 在输入框输入沟通消息
node playwright-cli/playwright-cli.js -s=boss fill <输入框ref> "您好，感谢您对这个岗位的关注！方便发一份您的简历过来吗？"

# 4. 点击发送按钮（快照中文本为"发送"的元素）
node playwright-cli/playwright-cli.js -s=boss click <发送按钮ref>
sleep 1

# 5. 获取新快照，点击"求简历"按钮
node playwright-cli/playwright-cli.js -s=boss snapshot
node playwright-cli/playwright-cli.js -s=boss click <求简历按钮ref>
sleep 1
```

然后更新 processed.json，将该候选人标记为 `messaged`。

**重要提示：**
- 输入框在聊天详情区底部，snapshot 中没有明显的 textbox。需要先点击输入区域再 type。
- 实际操作时优先尝试用 `type` 命令而非 `fill`：
  ```bash
  # 先点击输入区域
  node playwright-cli/playwright-cli.js -s=boss click <输入区域ref>
  # 然后输入文字
  node playwright-cli/playwright-cli.js -s=boss type "您好，感谢您对这个岗位的关注！方便发一份您的简历过来吗？"
  # 发送
  node playwright-cli/playwright-cli.js -s=boss click <发送按钮ref>
  ```
- "求简历"按钮在底部工具栏，快照中文本为 `求简历`
- "发送"按钮在输入框右侧，快照中文本为 `发送`

#### 第三步B：下载已收到的简历

对于聊天列表中最后消息包含"附件简历"的候选人：

```bash
# 1. 点击聊天项进入对话
node playwright-cli/playwright-cli.js -s=boss click <聊天项ref>
sleep 2

# 2. 获取快照，找到"点击预览附件简历"按钮
node playwright-cli/playwright-cli.js -s=boss snapshot

# 3. 点击预览按钮
node playwright-cli/playwright-cli.js -s=boss click <"点击预览附件简历"的ref>
sleep 3

# 4. 获取快照，找到下载按钮（预览工具栏第三个图标）
node playwright-cli/playwright-cli.js -s=boss snapshot

# 5. 点击下载按钮
node playwright-cli/playwright-cli.js -s=boss click <下载图标ref>
sleep 2

# 6. 检查网络日志确认下载完成，获取文件名
node playwright-cli/playwright-cli.js -s=boss network

# 7. 创建职位文件夹并移动文件
mkdir -p "output/<职位名>"
mv ".playwright-cli/<下载的文件名>.pdf" "output/<职位名>/<候选人姓名>.pdf"

# 8. 关闭预览
node playwright-cli/playwright-cli.js -s=boss press Escape
sleep 1
```

然后更新 processed.json，将该候选人标记为 `downloaded`。

**关于下载按钮的定位：**
- 点击"点击预览附件简历"后，页面底部会出现预览层
- 预览层顶部有三个 img 图标按钮: 全屏 | 打印 | 下载
- 下载是第三个，CSS 选择器: `.attachment-resume-btns > div:nth-child(3)`
- 在快照中表现为预览容器内连续的三个 `img [cursor=pointer]` 元素，取最后一个
- network 日志中会显示 `Downloading file ... Downloaded file ... to ".playwright-cli/xxx.pdf"`

#### 第四步：更新 processed.json

每处理完一个候选人，立即用 node 更新 processed.json：

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

只下载已收到但未下载的简历，不发送新消息：

按照第三步B的流程，只处理聊天列表中包含"附件简历"且 processed.json 中 status 不是 `downloaded` 的候选人。

### 6. 单独沟通新候选人 (boss-greet)

只对新候选人发送沟通消息，不下载简历：

按照第三步A的流程，只处理不在 processed.json 中的新候选人。

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

### 沟通页面 (/web/chat/index)

```
左侧: 聊天列表
  listitem:
    generic [cursor=pointer]:          ← 可点击的聊天项
      generic: "17:29"                 ← 时间
      generic:
        generic "候选人姓名"            ← 姓名
        generic "职位名"               ← 职位
      generic: "最后一条消息内容"       ← 如果是"附件简历-xxx.pdf"说明已发简历

右侧: 聊天详情（点击聊天项后显示）
  消息列表:
    ...历史消息...
    generic:
      heading "附件简历-姓名-职位.pdf"   ← 简历附件消息
      generic [cursor=pointer]: "点击预览附件简历"  ← 预览按钮
  底部工具栏:
    generic [cursor=pointer]: "求简历"   ← 求简历按钮
    generic [cursor=pointer]: "换电话"
    generic [cursor=pointer]: "换微信"
    generic: "发送"                      ← 发送按钮

简历预览层（点击预览后出现在页面底部）:
  generic:
    img [cursor=pointer]                 ← 全屏
    img [cursor=pointer]                 ← 打印
    img [cursor=pointer]                 ← 下载（点这个）
  iframe: 简历内容
```

### 关键 API

| API | 用途 |
|-----|------|
| `/wapi/zpjob/job/data/list?position=0&type=0&page=1` | 获取职位列表 |
| `/wapi/zprelation/interaction/bossGetGeek?jobid=<id>&page=1&tag=4&status=4` | 获取某职位的感兴趣候选人 |
| `/wapi/zpchat/boss/historyMsg?src=0&gid=<uid>&maxMsgId=0&c=20&page=1` | 获取聊天历史消息 |
| `/wflow/zpgeek/download/preview4boss/<geekId>` | 下载简历文件 |

## 注意事项

- **登录态有效期有限**：如果操作中被跳转到登录页，需要重新 boss-login
- **ref 会变化**：每次 snapshot 后 ref 值会刷新，必须用最新快照中的 ref
- **页面有 iframe**：职位管理和互动页面的内容在 iframe 中，ref 前缀为 `f<数字>e<数字>`
- **加载需要等待**：页面和简历预览都需要等待加载完成再操作，一般 sleep 2-3 秒
- **下载文件位置**：playwright-cli 默认将文件下载到 `.playwright-cli/` 目录
- **操作频率**：避免过快操作，每个动作之间至少间隔 1-2 秒，防止触发反爬
- **弹窗处理**：如果遇到 dialog 弹窗，用 `dialog-accept` 或 `dialog-dismiss` 处理
- **聊天列表滚动**：左侧聊天列表默认只显示可见区域的候选人。如需处理更多，需要向下滚动加载更多聊天项。使用 `mousewheel 0 500` 在聊天列表区域滚动
- **重复消息避免**：在发送沟通消息前，先检查聊天记录中是否已有自己发过的消息。如果已经发过消息（如快照中有"已读"标记或自己发送的消息），则只需要确认是否点过"求简历"即可，不要重复发送沟通文本
