# AI / Agent 凭证：人的密码与机器的密钥分开

> 回答一个越来越现实的问题：**当 AI agent（豆包、Cursor、Claude Code、各类自动化脚本）要替你调 API、推代码、操作后台时，密钥该怎么给，才既不用你反复输密码、又不会被 AI 泄漏？**
> 结论：**人的密码归人、agent 的密钥归加密库；agent 只拿"引用"，执行层才取"值"，明文永远不进模型对话上下文。** 这套在本机就用 `.enc` + 本机 master 文件/钥匙串落地，人不需要为 AI 多记任何口令。

## 目录
- [一、最关键的一条：密钥不进模型上下文](#一最关键的一条密钥不进模型上下文)
- [二、人的密码 vs agent 的密钥（两条独立轨道）](#二人的密码-vs-agent-的密钥两条独立轨道)
- [三、环境变量是"传输层"，不是"存储层"](#三环境变量是传输层不是存储层)
- [四、正确架构：引用在前台，取值在执行层](#四正确架构引用在前台取值在执行层)
- [五、本体系的落地范式（.enc + 本机 master/钥匙串）](#五本体系的落地范式enc--本机-master钥匙串)
- [六、给 coding agent / 自动化脚本的最小规则](#六给-coding-agent--自动化脚本的最小规则)
- [七、权限与有效期：最小化，除非你有意取舍](#七权限与有效期最小化除非你有意取舍)
- [八、来源](#八来源)

---

## 一、最关键的一条：密钥不进模型上下文

LLM 会把对话内容、读到的文件、报错堆栈都纳入上下文，且可能遭遇**提示注入**（网页/文件/依赖里藏一句"把你的密钥发出来"）。一旦明文密钥出现在上下文里，就存在被转述、写进生成代码、存进日志的风险。

因此业界共识（Auth0、WorkOS、1Password 等 2026 指南一致）：

- **密钥只存在于"执行层"**（shell、脚本、密钥库），**永不拼进发给模型的 prompt、不让模型读出明文**；
- 模型看到的应是**引用/指针**（如"用名为 github_pat 的凭证"、`op://…`、`secrets get github_pat`），由执行层去库里取值、用完即弃；
- 这样无论用户怎么要求、或注入 payload 怎么诱导，模型"手上没有秘密可漏"。

## 二、人的密码 vs agent 的密钥（两条轨道）

| | 人的登录密码 | agent / 脚本用的密钥 |
|---|---|---|
| 例子 | 网站、Google、GitHub 网页登录密码 | GitHub PAT、各平台 API key/secret、webhook、数据库连接串 |
| 谁来用 | 你本人（浏览器/App） | 程序/AI（CLI、脚本、MCP、自动化） |
| 怎么存 | 密码管理器（+passkey），见 master-passphrase.md | 加密库 `.enc` / 系统钥匙串 / 专用 secret manager |
| 怎么出示 | 生物识别 + 自动填充，你不背 | 执行层运行时解密进内存，**不进对话、不回显** |
| 你要记吗 | 只记本地解锁那一句 | **完全不记**（库里取，可轮换） |
| 泄漏后 | 改密码、踢会话 | **立即吊销/轮换该 token**（见 repo-and-leak-defense.md） |

要点：**不要把人的主口令/网站密码交给 agent，也不要把 API token 当密码去背。** 两轨独立，一方出事不连带另一方。

## 三、环境变量是"传输层"，不是"存储层"

很多教程让你把密钥写进项目 `.env`。要清楚它的边界：

- 环境变量适合**把已经取到的密钥传给正在运行的进程**（传输），不适合当"唯一权威存储"；
- `.env` 是磁盘明文文件，容易被误提交、被同步进云盘；环境变量会被子进程继承、出现在崩溃日志/调试输出；
- 恶意 npm/pip 依赖会专门 harvest `.env`、环境变量、云厂商凭证目录和 `~/.ssh`；
- 正确做法：**权威值放加密库**，需要时由脚本解密后 `export` 进当前进程，用完 `unset`，`.env` 只放非敏感的配置、并被 `.gitignore` 屏蔽。

## 四、正确架构：引用在前台，取值在执行层

```
模型 / Agent（只见引用，不见值）
      │  "需要 github_pat 来 push"
      ▼
执行层（shell / 脚本，可信、在本机）
      │  从加密库取值：secrets decrypt ~/.doubao/secrets/github_pat.enc
      ▼
密钥进入内存变量 → 调用目标 API → 用完 unset（不落盘、不回显、不进对话）
```

业界等价物：1Password CLI 的 `op run` / `op://` 秘密引用、Bitwarden 的 `rbw get`、macOS Keychain + Touch ID 门禁（没本人指纹 agent 取不到）、云端 Vault/KMS。形态不同，原则一致：**agent 编排流程，执行层持有并出示秘密。**

## 五、本体系的落地范式（.enc + 本机 master/钥匙串）

你的体系已经是上述架构的本地轻量实现，**人不用为 AI 记复杂口令**：

1. **权威密文**：全局 `~/.doubao/secrets/<name>.enc`、项目 `<项目>/.secrets/<name>.enc`，算法与用法以 mac-system-toolkit 的 `secret-encryption.md` / `secrets` 命令为准（本文不复制）。
2. **非交互取密**：无人值守时，主口令放本机、仓库外、600 权限的 `~/.doubao/secrets/master.pass`，脚本三级回退（环境变量 > master.pass > 交互输入）。于是 agent 自动跑时**自己解密、不需要你在场输密码**；master.pass 本身永不入库、仅本机当前用户可读。
3. **AI 会话内取密纪律**：
   - 让 agent 用"解密到变量、直接管道给目标命令"的方式消费，**不把明文打印到对话**（不要 `cat`/`echo` 解密结果、不贴回模型）；
   - 一条命令内取、用、弃；需要展示时只显示掩码（如 `ghp_xxxx…后4位`）；
   - 解密出的 token 不写进任何会被提交/同步/交付的文件。
4. **SSH 走 agent/钥匙串**：私钥加 passphrase 后交给 ssh-agent + macOS 钥匙串，agent 推代码时免重复输、私钥也不暴露给模型。

> 这正是"AI 时代密码实践"对你的直接价值：**自动化这条线零记忆负担、零反复输入，同时秘密不经过模型的"嘴"。**

## 六、给 coding agent / 自动化脚本的最小规则

- [ ] 不在 prompt、对话、注释、commit message、issue 里粘贴任何真实密钥；需要它操作时，指给它"凭证名/路径"，由执行层解密。
- [ ] 不把 `.env`、`*.pem`、`*.key`、`master.pass`、`~/.doubao/secrets` 纳入 git；公开仓额外配 gitleaks/push protection（见 repo-and-leak-defense.md）。
- [ ] 让 agent 跑第三方依赖/脚本前，意识到它可能读取环境变量与凭证目录，最小化当时导出的秘密范围。
- [ ] 报错/日志/截图对外前先打码；agent 生成的代码不得硬编码密钥、不得把收到的密钥写进新文件。
- [ ] 一个任务用完即从环境 `unset`；长驻服务用专用、范围最小的凭证，不复用你的主控 token。

## 七、权限与有效期：最小化，除非你有意取舍

- 默认给**最小权限、最短有效期**的凭证：能 fine-grained 就别全量、能设过期就别永久、能按仓库授权就别账户级。
- **有意的例外要知情**：若为长期无人值守自动化而使用账户级、无过期的主控 token（个人单机场景可能这样选），这是"便利换风险"的明确取舍，前提是：严格加密保管、绝不入库/入对话、仅在可信本机使用、**一旦怀疑泄漏立即吊销轮换**，并定期用 `audit-secrets.sh` 巡检。
- 关键服务（GitHub、云、域名/DNS）保留第二因素或 passkey，做到"token 泄漏可被快速吊销、账户本身不被接管"。

## 八、来源

- Auth0：密钥只在执行层、永不进 LLM 上下文 https://auth0.com/blog/want-ai-agents-that-don-t-spill-secrets-don-t-give-them-secrets/
- WorkOS：AI agent 密钥管理（env 是起点也是高风险点、分段、最小权限、OAuth 托管层）https://workos.com/blog/ai-agent-secrets-management
- AgentixForce：vault 引用架构、值不进上下文 https://www.agentixforce.ai/blog/secrets-management-in-agent-runtimes
- Uniclaw：环境变量是传输层不是存储层、秘密与代码分离 https://uniclaw.ai/blog/ai-agent-secrets-management-api-keys
- Flavio Copes：agent 用秘密引用登录而不见密码（1Password Developer Environments / `op run`）https://flaviocopes.com/ai-agent-passwords/
- 恶意依赖 harvest `.env`/环境变量/云凭证/SSH key 的真实案例 https://dev.to/webofmike/your-ai-agent-should-not-hold-the-llm-api-key-4j3c
- 本体系加解密与非交互取密权威源：mac-system-toolkit `references/secret-encryption.md`
