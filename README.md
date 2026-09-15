# security-baseline

个人 AI 开发工作的**安全基线与凭证"来处"治理**横切技能：开发/AI 工具（gh、ssh-agent、sudo、脚本、第三方 API）要用密码或 key 时，确定它从哪个加密来处取、怎么取才不暴露用户主口令；主口令只在交互时由用户当次提供、用后即弃，二段因子走 Microsoft Authenticator；另含公开仓防泄漏与密钥泄漏应急、VPN/代理安全使用。与 project-manager 平级，跨所有业务项目与机器全局生效；纯方法论，不含脚本（加解密/巡检复用 mac-system-toolkit，双机凭证台账在 dual-machine-manager）。

> 边界：本技能面向**开发/AI 侧凭证**，不改造、不收集用户个人各网站登录密码，也不主动给用户安装个人密码管理器。

## 使用

这是豆包（及兼容 Agent）的**本地技能（Skill）**。完整能力、触发场景与决策流程见入口文档 **[`SKILL.md`](SKILL.md)**，Agent 命中时首先读取它；下列 references 按需加载，不必一次全读。

## 目录

- `SKILL.md`：安全心智模型、开发场景 30 秒决策（凭证来处 + 用户配合）、硬红线、按需加载索引、与其它技能边界
- `references/ai-agent-credentials.md`：**操作核心**——AI/开发工具取密 SOP（gh/ssh-agent/sudo/脚本标准取法、秘密不进上下文、取—用—弃、交付前自检）
- `references/credential-storage.md`：开发凭证分类与加密落点、全局/项目 `.enc`、典型工具凭证来处对照、何时才需要 agent-side 密钥库
- `references/master-passphrase.md`：主口令怎么向用户取得、怎么用后不残留、2FA/TOTP 怎么配合、非交互 master.pass 的启用条件
- `references/repo-and-leak-defense.md`：公开仓五道防线、泄漏应急（先轮换）、打码脱敏、交付前清单
- `references/network-and-vpn.md`：代理连通性判断、临时走代理、VPN 安全边界与 Agent 检索渠道决策

## 许可

[MIT](LICENSE) © 2026 byte886
