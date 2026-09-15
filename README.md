# security-baseline

个人 / 小团队 AI 工作的**安全基线与凭证治理**横切技能：凭证分级与"存哪里"决策、记忆力有限时的人因主口令方案、公开仓防泄漏与密钥泄漏应急、VPN/代理的安全使用。与 project-manager 平级，跨所有业务项目与机器全局生效；纯方法论，不含脚本（加解密/巡检复用 mac-system-toolkit，双机凭证台账在 dual-machine-manager）。

## 使用

这是豆包（及兼容 Agent）的**本地技能（Skill）**。完整能力、触发场景与决策流程见入口文档 **[`SKILL.md`](SKILL.md)**，Agent 命中时首先读取它；下列 references 按需加载，不必一次全读。

## 目录

- `SKILL.md`：安全心智模型、凭证分级与 30 秒决策、硬红线、按需加载索引、与其它技能边界
- `references/credential-storage.md`：五类凭证与存放位置、全局/项目 `.enc`、密码管理器选型
- `references/master-passphrase.md`：唯一主口令怎么造得又强又好记、离线备份、迁移与更换
- `references/repo-and-leak-defense.md`：公开仓五道防线、泄漏应急（先轮换）、打码脱敏、交付前清单
- `references/network-and-vpn.md`：代理连通性判断、临时走代理、VPN 安全边界与 Agent 检索渠道决策

## 许可

[MIT](LICENSE) © 2026 byte886
