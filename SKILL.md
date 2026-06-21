---
name: lark-chat-daily-report
description: 当用户需要把某个飞书群在指定日期或指定时段的消息整理成结构化日报、写回固定飞书文档、补抓链接/图片/附件、并在完成后发送通知时使用。适用于活动群日报、社群情报整理、嘉宾分享汇总、资料沉淀和固定文档续写。
---

# 飞书群日报整理

## 适用场景

- 用户说“整理昨天某个群的消息”“写群日报”“汇总某群核心信息”“把群聊精华写进固定文档”。
- 需要同时处理消息检索、图片/文件补抓、飞书文档更新、通知发送。
- 优先用于“固定文档持续更新”的日报，不是一次性摘要。

## 开始前必须读取

- `~/.agents/skills/lark-shared/SKILL.md`
- `~/.agents/skills/lark-im/SKILL.md`
- `~/.agents/skills/lark-doc/SKILL.md`
- `~/.agents/skills/lark-drive/SKILL.md`
- 需要给个人发通知时，再读 `~/.agents/skills/lark-contact/SKILL.md`
- 如果任务是 Demo Day #3，额外读取 [references/demo-day-profile.md](references/demo-day-profile.md)

## 默认原则

- 优先使用 `lark-cli` 的 shortcut，顺序是：shortcut > 已注册 API > `lark-cli api`
- 用用户身份执行群消息搜索、文档读取和文档写入，除非用户明确要求 bot 身份
- 开始执行前先检查 `lark-cli` 的授权状态；如果 user 身份未授权或 token 不可用，先发起 Feishu CLI 授权流程，拿到用户授权后再继续
- 写入前先检查当天日期区块是否已存在，避免重复追加
- 新增内容必须写回对应日期区块内部，不要把补充摘录、图片、链接单独堆到文档末尾
- 所有本次修改和新增内容都加浅绿色底色；如果用户要求只有当天区块高亮，则把历史日期区块恢复默认白底黑字，仅保留当天区块浅绿色
- 过滤表情、玩笑、纯附和和无上下文闲聊；保留高价值链接、图片、附件、产品名、案例、问题和行动项

## 需要先确认或自行推断的输入

- 群名或 `chat_id`
- 目标日期与时区
- 固定日报文档 URL 或 token
- 是否有固定关键词补抓清单
- 完成后是否通知
- 如果通知，通知给谁、通知到哪个群、用什么身份发

如果用户没有补充，优先按已有上下文、自动化记忆或默认 profile 推断；只有缺少关键信息且无法可靠发现时才提问。

## 工作流

### 1. 先检查 Feishu CLI 授权状态

先执行：

```bash
lark-cli auth status
```

判断规则：

- 如果 user identity 是 `ready` 且 `tokenStatus` 是 `valid`，继续后续步骤
- 如果 user identity 不可用、token 过期、或缺少后续步骤所需 scope，先按 `lark-shared` 流程发起授权
- 发起授权时，把 CLI 返回的授权链接原样转给用户，等待用户确认“已授权”后再继续

常见补救：

```bash
lark-cli auth login --scope "search:message im:message:readonly im:message.send_as_user docx:document:readonly docx:document:write_only docs:document.content:read docs:document.media:download docs:document.media:upload"
```

如果任务还需要发个人通知、发群通知或搜联系人，再按最小权限原则追加对应 scope。

### 2. 锁定时间范围

- 如果用户说“整理 6 月 20 日的”，按用户所在时区或任务指定时区解释成该自然日 `00:00:00` 到次日 `00:00:00`
- 如果用户说“昨天”，必须在回复或写入中用绝对日期落地，避免相对日期歧义
- 搜索时统一保存：
  - `date_label`
  - `timezone`
  - `start_time`
  - `end_time`

### 3. 找到目标群并确认可访问

优先用：

```bash
lark-cli im +chat-search --as user --query "<群名>"
```

如已知 `chat_id`，直接读取群信息确认群存在：

```bash
lark-cli im chats get --as user --params '{"chat_id":"oc_xxx"}'
```

至少确认：

- 群名是否匹配
- 当前身份能否读取消息
- 当前身份是否能写群消息
- 群是否为外部群，避免后续通知步骤失败

### 4. 必须同时做三类检索

#### A. 全量时段消息

```bash
lark-cli im +chat-messages-list --as user --chat-id oc_xxx --start-time "<ISO8601>" --end-time "<ISO8601>" --page-all
```

要求：

- 拉完整时间窗，不要只看搜索命中的消息
- 保留消息 `message_id`、发送人、发送时间、文本、消息类型、附件信息

#### B. 关键词补充搜索

优先用：

```bash
lark-cli im +messages-search --as user --chat-id oc_xxx --query "<关键词>" --start-time "<ISO8601>" --end-time "<ISO8601>" --page-all
```

关键词由用户给定；如果没有，基于活动群日报常见资产补抓：

- `文档`
- `分享文档`
- `活动文档`
- `嘉宾`
- `Skill`
- `回放`
- `妙记`
- `minutes`
- `智能纪要`
- `录屏`
- `PPT`
- `GitHub`
- `开源`
- `产品`
- `案例`
- `教程`
- `资料库`
- `指南`

#### C. 图片/文件类消息扫描

- 从全量消息里筛出 `image`、`file`、`media`、`post`、带附件卡片的消息
- 重点识别：
  - 长图
  - 架构图
  - 流程图
  - 截图
  - 作品图
  - 案例图
  - PPT 截图
  - 群友明确说“有用”“收下”“可以参考”的图

如需下载资源：

```bash
lark-cli im +messages-resources-download --as user --message-id om_xxx --file-key <resource_key> --output-dir "<local_dir>"
```

### 5. 去重并提炼

- 合并三类检索结果，按 `message_id` 去重
- 同一链接重复出现时只保留一次，但尽量保留最早的来源消息
- 提炼时优先保留：
  - 活动官方信息
  - 文档、回放、妙记、录屏、报名、投稿、资源包
  - 嘉宾身份、主题、案例、仓库、教程
  - 群友对 Agent、模型、工具、企业需求的具体经验
  - 待跟进问题

### 6. 生成日报内容

当天标题格式固定为：

```text
YYYY-MM-DD｜今日核心信息
```

其中“今日核心信息”必须概括当天最核心的信息，控制在 20 个汉字以内。

每天区块至少覆盖这些小节，可按信息量增减：

- `必看资产`
- `群主/管理员核心信息`
- `嘉宾分享与案例`
- `外部资料和链接`
- `有价值图片`
- `群友经验与产品讨论`
- `待跟进问题`

如果当天没有明显新增内容，也要写：

```text
昨日未发现高价值新增信息
```

每条内容尽量保留：

- 发送人
- 发送时间
- 原始链接、文档链接或消息链接

### 7. 更新固定飞书文档

先读取目标文档现状，再决定插入或更新当天区块。优先复用 `lark-doc` skill 的读取与更新流程，不要盲改全文。

写入规则：

- 最新日期区块按倒序放在最上面，位于基础信息之后、旧日期之前
- 所有新增链接、图片、补充摘录必须归入当天日期区块内
- 对有价值图片，优先下载原图并作为图片块插入，同时附发送时间、发送人、简短说明
- 不要只写 `[Image]`
- 如果更新的是已存在日期区块，覆盖更新该区块，不要重复追加第二个同日期区块

样式规则：

- 本次新增或更新的内容统一加浅绿色底色
- 如果用户规则要求“只有当天区块高亮”，则写入前先把历史日期区块恢复默认白底黑字，再只保留当天区块浅绿色

### 8. 发送完成通知

通知前必须明确三件事：

- 是否需要通知
- 收件人或群
- 发送身份
- 文案或卡片内容

不要把通知对象硬编码成某个人或当前群。通知对象必须来自：

- 用户这次明确指定
- 或者用户自己提供的默认配置 / 自动化记忆

如果两者都没有，就先问用户要发给谁，不要擅自代入操作者本人或当前工作群。

优先发卡片；如果群权限不允许，再降级为纯文本，但要在结果里说明原因。

个人通知可用：

```bash
lark-cli im +messages-send --as user --user-id ou_xxx --msg-type interactive --content '{...}'
```

群通知可用：

```bash
lark-cli im +messages-send --as user --chat-id oc_xxx --msg-type interactive --content '{...}'
```

如果失败，必须区分：

- 用户身份没权限发群
- bot 不在群里
- 外部群不允许当前操作者拉 bot 入群

这些结论要写进自动化记忆，避免下次重复试错。

## 结果输出

完成后向用户至少反馈：

- 日报文档链接
- 当日标题
- 新增链接数量
- 新增图片数量
- 通知是否发送成功，分别列出个人通知和群通知结果
- 如果权限受阻，明确给出错误码和阻塞点

## Demo Day #3 约定

当任务明确是 Demo Day #3 时，直接套用 [references/demo-day-profile.md](references/demo-day-profile.md) 中的默认群、文档、关键词和文档结构规则。
