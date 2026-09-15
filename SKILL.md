---
name: security-baseline
description: 个人/小团队 AI 工作的安全基线与凭证治理横切技能，跨所有业务项目与两台 Mac 全局生效，是和 project-manager 平级的全局治理能力。覆盖：①凭证分级与"到底存哪里"决策——优先 passkey/生物识别消除记忆，本地只记一句解锁口令、密码管理器管网站密码、加密 .enc 管开发者 token、macOS 钥匙串与 SSH agent 分工，人的密码与 AI agent 密钥分轨；②弱记忆友好的人因密码管理——记不住复杂口令、不想用纸备份时，用 passkey 通行密钥/生物识别自动填充把"要记要输"降到最低、本地口令取"记得住的中等强度"、数字冗余代替纸质备份、密码管理器与 Microsoft Authenticator 分工、迁移与更换路线；③AI/agent 凭证治理——密钥不进模型上下文、agent 只持引用、执行层从加密库取后即弃、环境变量只是传输层；④公开仓防泄漏分层防御（明文不入库、.gitignore/.enc、pre-commit gitleaks、GitHub push protection、误提交与泄漏应急=先轮换再清历史、交付前安全清单、打码脱敏、最小权限）；⑤网络与 VPN/代理的安全使用与 Agent 操作指针。当用户提到"密码怎么存/记不住密码/嫌密码难输、主密码/主口令/passphrase、passkey/通行密钥/无密码登录、指纹/面容/生物识别解锁、Microsoft Authenticator/验证码/TOTP、密码管理器/1Password/Bitwarden/KeePass/钥匙串/Google 密码管理器、AI agent 密钥/LLM 密钥/自动化怎么取密钥/.env 怎么处理、API key/token/secret/私钥/凭证放哪、这个能不能提交 git/公开仓、.enc/密钥泄漏/误提交、token 轮换、脱敏打码、最小权限、交付前安全检查、VPN/代理能不能用、安全基线/安全规范"等任何安全与凭证相关需求时使用。纯方法论与规范，不含可执行脚本：加解密/明文巡检复用 mac-system-toolkit，双机具体凭证台账在 dual-machine-manager。
compatibility: 纯方法论与 Markdown 规范，不随附可执行脚本，因此 Windows/macOS/Linux 三平台通用、无需 uname 判平台、无平台适配缺口；正文示例命令（secrets、~/.doubao/secrets、代理端口）以 macOS 双机现状为例，实际加解密/巡检/代理动作由 mac-system-toolkit 等工具技能按其各自的平台标注落地。
---

# security-baseline · 安全基线与凭证治理

> 一句话：**让"安全"成为不用每次重新判断的默认基线——什么是敏感信息、它该存哪、能不能进 git / 发出去、人和 AI 各自怎么用密钥、出事了先做什么。**

## 平台适用（执行前先读）

- 纯方法论 + Markdown，**无脚本、不依赖 OS**，三平台通用，无需判平台。
- 路径示例一律用 `$HOME` / `~`，不写死 `/Users/<用户名>`（两台 Mac 家目录名不同）。
- 本技能只定"规矩与决策"；真正执行加解密、巡检、代理开关时，按文中指针调用对应工具技能，不重复造脚本。

## 三层分工：先定位，别走错门

| 层 | 角色 | 由谁负责 |
|---|---|---|
| **策略 / 总纲（脑）** | 什么算敏感、怎么分级、存哪、红线、检查清单、出事怎么办 | **本技能** |
| **机制 / 工具（手）** | 怎么加密解密、怎么扫明文泄漏、怎么开关代理 | mac-system-toolkit（`secrets.sh` / `audit-secrets.sh` / `vpn-control.md`），本技能只给指针 |
| **资产 / 台账（账）** | 两台机器具体有哪些凭证、落点路径、账号现状 | dual-machine-manager（`credentials.md` / `security-and-git.md`） |

## 核心心智模型（5 条，先建立再动手）

1. **先消除"要记/要输"，再区分"人记的"和"机器存的"**：在线账户优先 passkey/生物识别，不支持的交密码管理器随机密码+自动填充；人只记**本地解锁层一句口令**，开发者密钥交加密库、AI 走执行层取密；绝不用一个弱口令走天下，也不要求人背难输的强口令。
2. **位置优先于值**：文档、仓库、聊天只记"凭证叫什么、在哪、怎么取"，**永远不记值本身**；AI 对话里尤其只出现引用、不出现明文密钥。
3. **默认不落盘；要落盘必加密；明文绝不进 git**：能不存就不存；必须存就加密；公开仓里出现明文密钥即视同泄漏。
4. **最小暴露 + 最小权限**：对外材料、日志、截图、交付物一律打码；token 默认授予任务所需最小权限与有效期，长期自动化主控 token 若有意放宽，必须加密保管、不外露、泄漏即轮换。
5. **一旦进了公开历史，先轮换、再清历史**：删除提交 / 重写历史**不能**让已经暴露的凭证失效，轮换/吊销才是真补救。

## 凭证分级与"存哪"：30 秒决策

| 凭证类型 | 例子 | 存哪 / 谁记 | 人要不要背 |
|---|---|---|---|
| **本地解锁口令（唯一要记）** | Mac/sudo、密码管理器主密码、`.enc` 库、SSH 私钥 | 人脑（记得住的中等强度）+ 数字冗余备份，不用纸 | **只记这一句** |
| 网站 / App 登录 | 各网站、SaaS、邮箱、社区 | **优先 passkey**（指纹/扫码）；否则管理器随机密码+生物识别自动填 | 不记、不输 |
| 开发者 / AI agent 密钥 | GitHub PAT、API key、webhook、连接串 | 加密 `.enc`（全局 `~/.doubao/secrets` 或项目 `.secrets`），执行层用时解密、**不进 AI 对话** | 不背 |
| SSH / 签名私钥 | `~/.ssh/id_*`、GPG key | `~/.ssh` + ssh-agent / 钥匙串，私钥加 passphrase | passphrase 即本地那句 |
| 一次性 / 会话凭证 | session、cookie、短期 token、Playwright 配对码 | 内存 / 临时文件，过期即弃、不落盘 | 不背 |
| 他人 / 客户敏感数据 | 客户资料、私域数据、证件号 | 最小留存、加密、不进对外交付与公开仓 | 不背 |

> 分级细节、"全局 vs 项目"落点、密码管理器/passkey 怎么选见 [references/credential-storage.md](references/credential-storage.md)；AI/agent 取密纪律见 [references/ai-agent-credentials.md](references/ai-agent-credentials.md)。

## 按需加载索引（不要一次全读）

| 你要做什么 | 读这篇 |
|---|---|
| 判断某类凭证存哪、全局还是项目、passkey/密码管理器怎么选 | [credential-storage.md](references/credential-storage.md) |
| 记不住/嫌难输复杂密码、想少记少输、passkey 怎么开、本地口令设多强、不用纸怎么防遗忘、怎么迁移 | [master-passphrase.md](references/master-passphrase.md) |
| AI/agent 要用密钥、怕密钥被模型泄漏、`.env` 怎么处理、自动化如何免输口令 | [ai-agent-credentials.md](references/ai-agent-credentials.md) |
| 判断能不能提交公开仓、怎么防误提交、已经泄漏/误提交怎么办、交付前自查、怎么打码 | [repo-and-leak-defense.md](references/repo-and-leak-defense.md) |
| 什么时候该开 VPN/代理、端口怎么定、敏感信息能不能走外网或代理 | [network-and-vpn.md](references/network-and-vpn.md) |

## 硬红线（任何任务都适用）

1. 明文密码 / token / 私钥 / 连接串：**不进 git、不写进脚本常量、不长期留在命令历史、不进日志/截图/对外文档/聊天与 AI 对话上下文**；示例一律用占位符（`<本地口令>`、`ghp_xxxx…后4位`）。
2. 加解密统一走 `secrets`（mac-system-toolkit），固定算法 `aes-256-cbc + pbkdf2 + base64` 不改；本地口令只来自 `ENC_PASS` / 交互输入 / 本机 600 权限 `master.pass`，**不硬编码、不猜测，用户没给就问**。
3. 任何要进公开仓的交付，提交前过 [repo-and-leak-defense.md](references/repo-and-leak-defense.md) 的"交付前安全清单"。
4. 对外分享 / 汇报 / 截图先打码；拿不准一串字符是不是敏感，**先按敏感处理**。
5. 怀疑泄漏：顺序固定为 **先让凭证失效（轮换 / 吊销）→ 再排查影响面 → 最后清理历史与加固**，不可颠倒。

## 与其它技能的关系 / 边界

- **vs mac-system-toolkit**：本技能定"规矩与决策"，它提供"工具实现"（`secrets` 加解密、`audit-secrets.sh` 巡检、`vpn-control.md` 代理开关、Chrome 控制）。不复制其脚本与详细用法，只放指针（DRY）。
- **vs dual-machine-manager**：它是"两台机器的凭证/账号台账与运维 SOP"，**遵循**本技能定的统一规范；本技能不绑定任何具体机器、不记某台机的具体凭证值。
- **vs project-manager**：同属全局横切治理技能。项目立项时按本技能规划该项目的 `.secrets` 结构与密钥责任，项目退役时把"凭证吊销/轮换"纳入关闭清单。
- **业务技能内的"安全红线"**（如某技能"不得绕过他人账户"）属于该技能自身使用边界，由其各自维护，本技能不替代。

### ADR：为什么独立成技能，而不是并入双机管理 / 只写进全局 AGENTS.md

- **不并入 dual-machine-manager**：安全是跨所有业务项目与机器的横切能力；并入"双机运维"会让非运维场景触发不到，且"安全 ≠ 运维"，定位错误。
- **不只写进 `~/Doubao/AGENTS.md`**：AGENTS 只承载精简硬红线与下钻指针，放完整方法论会臃肿、也不是可按需加载的知识体。
- **最终选择**：独立技能承载方法论（脑），AGENTS/README 只放红线 + 指针（入口），工具留在 mac-system-toolkit（手），具体台账留在 dual-machine-manager（账）。一处定义、多处指针，避免重复与漂移。

## 来源（权威依据，链接见各 reference 末尾）

- passkey / FIDO2（Google、Microsoft）与 Microsoft Authenticator 能力变更；弱记忆友好的生物识别解锁实践。
- AI/agent 密钥管理（Auth0、WorkOS 等：密钥不进上下文、执行层取值、环境变量只是传输层）。
- EFF Diceware、NIST SP 800-63B（何时需要更强主口令）；OWASP Password Storage Cheat Sheet / ASVS。
- GitHub Docs：secret scanning / push protection / preventing data leaks；gitleaks（pre-commit 与历史扫描）。
- 2026 年密码管理器横向评测（Bitwarden / KeePassXC / iCloud 钥匙串 / 1Password / Google 密码管理器）。
