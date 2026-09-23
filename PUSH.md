# PUSH.md · 微信端推送层设计

> 报告本体永远在 Obsidian vault，微信收到的只是**摘要卡 + 指路**。本层是纯附加通知，故障不得影响报告生成。

## 1. 渠道抽象（config.push.channel 五选一）

| channel | 载体 | 消息出现在哪 | 前置条件 |
|---------|------|----------|----------|
| `wecom_connector`（**本机当前生效**） | WorkBuddy 企业微信连接器（wecom-cli，机器人主动通知） | 企业微信 App · 机器人单聊 | 连接器已连接并完成扫码授权即可，**零密钥管理** |
| `wecom_robot` | 企业微信群机器人 webhook | 企业微信 App · 群聊 | 建群 → 添加群机器人 → 复制 webhook 地址 |
| `serverchan` | Server酱·Turbo | **个人微信**"服务通知"（经服务号下发） | sct.ftqq.com 微信扫码登录 → 复制 SendKey；免费档有限流（以官网实时说明为准） |
| `wxpusher` | WxPusher | **个人微信**"服务通知"（经公众号下发） | wxpusher.zjiecode.com 注册 → 建 App 拿 appToken → 关注公众号绑定 UID；额度以官网为准 |
| `none` | 不推送 | 静默模式 | 无 |

### wecom_connector 实现要点（2026-09-23 实测）

- 发送命令：`wecom-cli message aibot send --chat-id <授权人ID，取自 whoami> --msg-type markdown --markdown '<JSON>'`，返回 `success: true` 即送达。
- **Windows 坑**：`wecom-cli.cmd`（cmd 壳）传中文参数会报"系统找不到指定的路径"——Git Bash 下必须用 sh 版 shim `~/.workbuddy/binaries/node/cli-connector-packages/wecom-cli`（直接 node 调用，绕过 cmd 编码）。
- 授权人 `chat_id` 由 `wecom-cli identity whoami` 动态获取，**不得写死进任何文件**；发授权人以外的目标须先 `sessions list` 现取（详见 wecomcli-message skill 的会话匹配规则）。
- ID 类字段（chat_id/userid 等）按 wecomcli-shared 约束**禁止出现在给用户的回复**中。

**关于"推到个人微信"的说明**：个人微信没有官方机器人 API。serverchan/wxpusher 都是把消息经**公众号/服务号**通道送到你个人微信的"服务通知"里——这是正规且稳的路径。宣称能直接进聊天窗口的方案（itchat/wechaty 等逆向协议）有封号风险，本系统不支持。

## 2. 配置（写进 config.yaml，真实密钥不入 git）

```yaml
push:
  channel: wecom_robot        # wecom_robot | serverchan | none
  webhook_url: ""             # wecom_robot: 完整 webhook 地址（含 key=...）；serverchan: SendKey
  push_targets: [daily, weekly, monthly]   # 哪些报告要推送
  digest_max_chars: 1200      # 摘要卡正文上限（字节级截断发生在 4096，这里是可读性上限）
  retry: 2                    # 失败重试次数，间隔 30s
```

`config.template.yaml` 只放占位符；**webhook_url 视同密码**：只存在于本机 config.yaml 与 vault 外 nowhere。若怀疑泄漏，去企业微信删掉该机器人即可使 key 立即失效。

## 3. 摘要卡格式（机械压缩，禁止二次创作）

推送内容由**已生成的 final 报告自动压缩**，AI 不得在摘要里添加报告没有的观点、情绪词或"机会提示"。结构固定：

```
📊 股票监控 · 日报 2026-09-23
标的：长鑫科技(688825)
判定：灰色地带（偏谨慎）
核心命题：G5 量产落地但资金面未确认
关键区间：支撑 53.16-53.47 / 压力 59.83-61.80
证据等级：已交叉验证（2 独立源）
⏳ 到期命题：9-30 回测支撑带（1 条待观察）
📄 全文：E:\我的笔记库\02投资\股票监控系统\日报\<文件名>.md
—— 未核实信息不进结论，不构成投资建议
```

字段直接取自报告 frontmatter 与"强制判定"栏；周报/月报同理（周报卡必含本周命题打分结果统计：成立 X / 证伪 Y / 待观察 Z）。

## 4. 发送规则（任何 AI 不可豁免）

1. **时机**：报告写入 vault 且 `status: final` 之后才推送；`draft` 不推。
2. **失败兜底**：重试后仍失败 → 写 `state.json.push_log`（时间/渠道/错误），**不得阻塞流程、不得重跑报告**；下次运行时检查 push_log 补推漏发摘要。
3. **降级链**：webhook 不可达 → 换 serverchan（若配置）→ 放弃并记录。禁止为推送引入新依赖（不装 SDK，用 `curl`/`requests` POST JSON 即可）。
4. **频率限制**：wecom_robot 20 条/分钟，系统天然不触线；serverchan/wxpusher 免费档有限流（额度以官网实时说明为准，未核实不写死数字）→ 触发限流时自动降为 `push_targets: [daily]` 并在周报里注明。
5. **内容纪律**：摘要卡同样过措辞黑名单检查（BOOTSTRAP 第 8 节）；发现 AI 在摘要里"加戏"→ 视为违反客观性红线。
