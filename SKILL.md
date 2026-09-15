---
name: security-baseline
description: 个人 AI 开发工作的安全基线与凭证"来处"治理横切技能，跨所有业务项目与两台 Mac 全局生效，与 project-manager 平级。核心定位：当开发过程或 AI agent 使用 gh、git/ssh-agent、sudo、脚本、第三方平台 API 时需要密码/token/私钥/2FA，来这里确定"凭证从哪个加密来处取、怎么取才不泄漏用户主口令"。约定：①用户本人的网站/登录密码习惯保持现状、本技能不改造、不索要、不记录；用户只在交互时提供一次本地主口令（口述/输入，当次内存使用），或用 Microsoft Authenticator 提供二段因子 TOTP；②一切开发/自动化凭证都有加密来处（全局/项目 .enc、macOS 钥匙串、ssh-agent），直接或间接都不暴露主口令明文，也不把目标凭证明文留在仓库/对话/日志/命令历史；③凭证分级与全局 vs 项目落点、加解密统一走 mac-system-toolkit 的 secrets（aes-256-cbc+pbkdf2）；④公开仓防泄漏（明文不入库、.gitignore/.enc、gitleaks、push protection、误提交/泄漏先轮换再清历史、交付前清单、打码脱敏、最小权限）；⑤网络与 VPN/代理安全使用指针。当用户提到"gh 登录/token 从哪来、git push/ssh-agent/私钥口令、sudo 密码、脚本或 API 要密码/key/secret、凭证放哪/存哪、.enc 怎么加解密、二段因子/验证码/TOTP/Authenticator 配合、这个能不能提交 git/公开仓、.env/密钥泄漏/误提交、token 轮换、脱敏打码、最小权限、交付前安全检查、VPN/代理、安全基线/凭证来处"等需求时使用。纯方法论与规范，不含可执行脚本：加解密/明文巡检复用 mac-system-toolkit，双机具体凭证台账在 dual-machine-manager。
compatibility: 纯方法论与 Markdown 规范，不随附可执行脚本，因此 Windows/macOS/Linux 三平台通用、无需 uname 判平台、无平台适配缺口；正文示例命令（secrets、~/.doubao/secrets、ssh-agent、代理端口）以 macOS 双机现状为例，实际加解密/巡检/代理动作由 mac-system-toolkit 等工具技能按其各自的平台标注落地。
---

# security-baseline · 安全基线与凭证来处

> 一句话：**开发工具或 AI 要用密码/key 时，不用临时问、不硬编码、不碰用户主口令明文——每个凭证都有确定的加密"来处"和标准取法。**

## 这个技能解决什么、不解决什么（先对齐边界）

- **解决（开发/AI 侧）**：gh、git/ssh-agent、sudo、自动化脚本、第三方 API 需要凭证时，"它存在哪、怎么加密、怎么取、怎么不泄漏、出事怎么办"。
- **不解决（用户侧）**：不改造、不评判、不收集用户在各网站/App 的登录密码习惯；不主动给用户安装个人密码管理器、不推动 passkey/换密码。用户唯一需要做的配合只有两件：**交互时提供一次本地主口令**、**需要二段因子时从 Microsoft Authenticator 读 TOTP**。
- 若用户**主动**想加强个人账户安全，再按需给方案；用户没提就不展开。

## 平台适用（执行前先读）

- 纯方法论 + Markdown，**无脚本、不依赖 OS**，三平台通用，无需判平台。
- 路径示例一律用 `$HOME` / `~`，不写死 `/Users/<用户名>`（两台 Mac 家目录名不同）。
- 本技能只定"规矩与决策"；真正执行加解密、巡检、代理开关时，按指针调用对应工具技能，不重复造脚本。

## 三层分工：先定位，别走错门

| 层 | 角色 | 由谁负责 |
|---|---|---|
| **策略 / 总纲（脑）** | 什么算敏感、怎么分级、凭证从哪来、红线、检查清单、出事怎么办 | **本技能** |
| **机制 / 工具（手）** | 怎么加密解密、怎么扫明文泄漏、怎么开关代理 | mac-system-toolkit（加解密 / 明文巡检 / 代理开关等执行能力），本技能只声明能力依赖、不写其内部路径 |
| **资产 / 台账（账）** | 两台机器具体有哪些凭证、落点路径、账号现状 | dual-machine-manager（双机凭证与账号台账能力） |

## 核心心智模型（5 条，先建立再动手）

1. **人的密码归人，开发凭证归加密库**：不索要/不记录/不改造用户各网站密码；开发与 AI 要消费的凭证（PAT、SSH 私钥、API key、项目密码）一律从加密来处取，**直接或间接都不暴露用户主口令明文**。
2. **主口令只在交互时由用户给、AI 当次内存用**：不写死、不猜测、不写进任何文件/脚本/提交、不回显到对话、不长期留在命令历史；无人值守才用本机、仓库外、600 权限的 `master.pass`（是否启用由用户决定）。
3. **默认不落盘；要落盘必加密；明文绝不进 git**：能不存就不存，必须存就 `.enc`；公开仓出现明文密钥即视同泄漏。
4. **位置优先于值 + 最小暴露/最小权限**：文档、仓库、聊天只记"凭证叫什么、在哪、怎么取"，永不记值；对外材料/日志/截图打码；token 只给任务所需权限，长期主控 token 若放宽必须加密保管、泄漏即轮换。
5. **一旦进了公开历史，先轮换、再清历史**：删提交/重写历史不能让已暴露凭证失效，轮换/吊销才是真补救。

## 开发场景 30 秒决策：凭证从哪来、要用户配合什么

| 开发场景 | 凭证来处 | 怎么取 / 用户配合 |
|---|---|---|
| `gh` 调 GitHub API / 建仓 / PR | 全局 `~/.doubao/secrets/github_pat.enc` | 解密后管道喂 `gh auth login --with-token`；主口令交互问用户，详见 ai-agent-credentials |
| `git push` over SSH | `~/.ssh/id_*` + ssh-agent/钥匙串 | 私钥 passphrase 一次性加入 agent，之后免输；agent 不持有主口令 |
| `sudo` / 系统级命令 | 全局 `sudo.enc` 或**直接让用户在终端交互输入** | 优先让用户本人输；自动化才解密，不回显 |
| 脚本调第三方平台 API | 项目 `<项目>/.secrets/<name>.enc` | 运行时 `secrets decrypt` 进内存变量，用完 `unset` |
| 跨项目通用 API/密钥 | 全局 `~/.doubao/secrets/<name>.enc` | 同上，全局凭证永不入库 |
| 需要二段因子（2FA/TOTP） | **用户的 Microsoft Authenticator** | 请用户读当下 6 位验证码；agent 不持有 TOTP 共享密钥（除非已加密存本机且用户授权） |
| 几分钟有效的临时票/session/cookie | 内存/临时目录 | 即用即弃、不落盘 |

> 全局 vs 项目 `.enc` 判断、加解密用法：[credential-storage.md](references/credential-storage.md)；取密 SOP 与不泄漏纪律：[ai-agent-credentials.md](references/ai-agent-credentials.md)。

## 按需加载索引（不要一次全读）

| 你要做什么 | 读这篇 |
|---|---|
| gh/ssh-agent/sudo/脚本要凭证、怎么从加密来处取、怎么不暴露主口令、非交互怎么办 | [ai-agent-credentials.md](references/ai-agent-credentials.md) |
| 某凭证算哪类、放全局还是项目 `.enc`、怎么加解密、钥匙串/ssh-agent 分工 | [credential-storage.md](references/credential-storage.md) |
| 主口令怎么向用户要、怎么用后即弃、2FA/TOTP 怎么配合、master.pass 何时可用 | [master-passphrase.md](references/master-passphrase.md) |
| 能不能提交公开仓、怎么防误提交、已泄漏/误提交怎么办、交付前自查、怎么打码 | [repo-and-leak-defense.md](references/repo-and-leak-defense.md) |
| 什么时候开 VPN/代理、端口怎么定、敏感信息能否走外网/代理 | [network-and-vpn.md](references/network-and-vpn.md) |

## 硬红线（任何任务都适用）

1. 明文主口令 / token / 私钥 / 连接串：**不进 git、不写进脚本常量、不长期留命令历史、不进日志/截图/对外文档/AI 对话**；示例一律占位（`<主口令>`、`ghp_xxxx…后4位`）。
2. 加解密统一走 `secrets`（mac-system-toolkit），固定算法 `aes-256-cbc + pbkdf2 + base64` 不改；主口令只来自交互输入 / `ENC_PASS` / 本机 600 权限 `master.pass`，**不硬编码、不猜测，用户没给就问，用完不残留**。
3. 任何要进公开仓的交付，提交前过 [repo-and-leak-defense.md](references/repo-and-leak-defense.md) 的"交付前安全清单"。
4. 对外分享 / 汇报 / 截图先打码；拿不准一串字符是否敏感，**先按敏感处理**。
5. 怀疑泄漏：顺序固定为 **先轮换/吊销 → 再排查影响面 → 最后清理历史与加固**，不可颠倒。

## 与其它技能的关系 / 边界

- **vs mac-system-toolkit**：本技能定"规矩与决策"，它提供"工具实现"（加解密、明文巡检、代理开关等执行能力，对外是 `secrets`、`audit-secrets.sh` 这类稳定命令接口）。不复制实现，只声明能力依赖（DRY）。
- **vs dual-machine-manager**：它是"两台机器的凭证/账号台账与运维 SOP"，遵循本技能规范；本技能不绑定具体机器、不记某台机的凭证值。
- **vs project-manager**：同属全局横切治理。项目立项时按本技能规划该项目 `.secrets` 结构与密钥责任，退役时把"凭证吊销/轮换"纳入关闭清单。
- 业务技能内自带的"安全红线"由各自维护，本技能不替代。

### ADR：为什么独立成技能，而不是并入双机管理 / 只写进全局 AGENTS.md

- **不并入 dual-machine-manager**：凭证来处与安全是跨所有项目与机器的横切能力，并入"双机运维"会让非运维场景触发不到。
- **不只写进 AGENTS.md**：AGENTS 只承载精简硬红线与下钻指针，完整方法论应是可按需加载的知识体。
- **定位是"给 AI/开发看的操作基线"，不是个人密码管理教程**：因此不展开 passkey/个人密码管理器/强口令改造（用户明确保持自身密码现状）；重点是开发凭证的加密来处与标准取法。
- 最终：独立技能承载方法论（脑），AGENTS/README 放红线+指针（入口），工具留 mac-system-toolkit（手），台账留 dual-machine-manager（账）。

## 来源（权威依据，链接见各 reference 末尾）

- AI/agent 密钥管理（密钥不进上下文、执行层取值、环境变量只是传输层）：Auth0 / WorkOS 等。
- 加解密约定由 mac-system-toolkit 的加解密能力提供（AES-256-CBC + PBKDF2）。
- GitHub Docs：secret scanning / push protection / preventing data leaks；gitleaks。
- OWASP Secrets / Password Storage Cheat Sheet；NIST SP 800-63B（主口令相关，仅在用户主动加强时参考）。
