---
name: security-baseline
description: 个人/小团队 AI 工作的安全基线与凭证治理横切技能，跨所有业务项目与两台 Mac 全局生效，是和 project-manager 平级的全局治理能力。覆盖：①凭证分级与"到底存哪里"决策——人脑只记一个主口令、密码管理器管网站/App 密码、加密 .enc 管开发者 token/密钥、macOS 钥匙串与 SSH agent 分工；②人因密码管理——记忆力不好时如何只记一句"够长又顺口"的主口令（Diceware/中文口令短语、强度与词数、离线备份、更换路线、密码管理器选型 Bitwarden/KeePassXC/iCloud 钥匙串）；③公开仓防泄漏分层防御（明文不入库、.gitignore/.enc、pre-commit gitleaks、GitHub push protection、误提交与泄漏应急响应=先轮换再清历史、交付前安全检查清单、对外打码脱敏、最小权限）；④网络与 VPN/代理的安全使用与 Agent 操作指针。当用户提到"密码怎么存/记不住密码/主密码/主口令/passphrase、API key/token/secret/私钥/凭证放哪、密码管理器/1Password/Bitwarden/KeePass/钥匙串、这个能不能提交 git/公开仓、.env/.enc/密钥泄漏/误提交、token 轮换、脱敏打码、最小权限、交付前安全检查、VPN/代理能不能用、外网访问安全、安全基线/安全规范"等任何安全与凭证相关需求时使用。纯方法论与规范，不含可执行脚本：加解密/明文巡检复用 mac-system-toolkit，双机具体凭证台账在 dual-machine-manager。
---

# security-baseline · 安全基线与凭证治理

> 一句话：**让"安全"成为不用每次重新判断的默认基线——什么是敏感信息、它该存哪、能不能进 git / 发出去、出事了先做什么。**

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

1. **区分"人记的"和"机器存的"**：人只记 **1 个主口令**；其余全部交给工具（密码管理器 / 加密库 / 钥匙串），绝不靠脑子记多个、也不重复使用同一弱口令。
2. **位置优先于值**：文档、仓库、聊天只记"凭证叫什么、在哪、怎么取"，**永远不记值本身**。
3. **默认不落盘；要落盘必加密；明文绝不进 git**：能不存就不存；必须存就加密；公开仓里出现明文密钥即视同泄漏。
4. **最小暴露 + 最小权限**：对外材料、日志、截图、交付物一律打码；token 只授予任务所需权限（长期自动化的主控 token 例外，可按需要求全权限但必须加密保管、不外露）。
5. **一旦进了公开历史，先轮换、再清历史**：删除提交 / 重写历史**不能**让已经暴露的凭证失效，轮换/吊销才是真补救。

## 凭证分级与"存哪"：30 秒决策

| 凭证类型 | 例子 | 存哪 / 谁记 | 人要不要背 |
|---|---|---|---|
| **唯一主口令** | 解锁 Mac、密码管理器、`.enc` 库、SSH 私钥 | 人脑 + 一张离线纸备份 | **只背这一个** |
| 网站 / App 登录密码 | 各网站、SaaS、社区账号 | 密码管理器随机生成、自动填 | 不背 |
| 开发者密钥 / token | GitHub PAT、API key、webhook、数据库连接串 | 加密 `.enc`（全局 `~/.doubao/secrets` 或项目 `.secrets`），用时解密 | 不背 |
| SSH / 签名私钥 | `~/.ssh/id_*`、GPG key | `~/.ssh` + ssh-agent / 钥匙串，私钥本身加 passphrase | passphrase 即主口令 |
| 一次性 / 会话凭证 | session、cookie、短期 token、Playwright 配对码 | 内存 / 临时文件，过期即弃、不落盘 | 不背 |
| 他人 / 客户敏感数据 | 客户资料、私域数据、身份证/手机 | 最小留存、加密、不进对外交付与公开仓 | 不背 |

> 分级细节、"全局 vs 项目"落点判断、密码管理器怎么选：见 [references/credential-storage.md](references/credential-storage.md)。

## 按需加载索引（不要一次全读）

| 你要做什么 | 读这篇 |
|---|---|
| 判断某类凭证存哪、全局还是项目、密码管理器选哪个 | [credential-storage.md](references/credential-storage.md) |
| 记不住复杂密码、想把主口令设得又强又好记、多久换、怎么备份 | [master-passphrase.md](references/master-passphrase.md) |
| 判断能不能提交公开仓、怎么防误提交、已经泄漏/误提交怎么办、交付前自查、怎么打码 | [repo-and-leak-defense.md](references/repo-and-leak-defense.md) |
| 什么时候该开 VPN/代理、端口怎么定、敏感信息能不能走外网或代理 | [network-and-vpn.md](references/network-and-vpn.md) |

## 硬红线（任何任务都适用）

1. 明文密码 / token / 私钥 / 连接串：**不进 git、不写进脚本常量、不长期留在命令历史、不进日志/截图/对外文档/聊天**；示例一律用占位符（`<主口令>`、`ghp_xxxx…后4位`）。
2. 加解密统一走 `secrets`（mac-system-toolkit），固定算法 `aes-256-cbc + pbkdf2 + base64` 不改；主口令只来自 `ENC_PASS` / 交互输入 / 本机 600 权限 `master.pass`，**不硬编码、不猜测，用户没给就问**。
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

- EFF Diceware（主口令词数与好记强度）；NIST SP 800-63B（memorized secret 长度/熵）；OWASP Password Storage Cheat Sheet / ASVS。
- GitHub Docs：secret scanning / push protection / preventing data leaks；gitleaks（pre-commit 与历史扫描）。
- 2026 年密码管理器横向评测（Bitwarden / KeePassXC / iCloud 钥匙串 / 1Password）。
