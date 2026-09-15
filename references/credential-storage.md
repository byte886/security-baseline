# 凭证分级与"到底存哪里"

> 回答两个问题：**①这东西算哪类凭证？②它该交给谁存、我自己要不要记？**
> 配套：怎么设唯一主口令见 [master-passphrase.md](master-passphrase.md)；怎么保证不进 git 见 [repo-and-leak-defense.md](repo-and-leak-defense.md)；加解密工具用法见 mac-system-toolkit 的 `secret-encryption.md`（本文不复制）。

## 目录
- [一、总原则：人只记一个，其余全交给工具](#一总原则人只记一个其余全交给工具)
- [二、五类凭证与各自的"家"](#二五类凭证与各自的家)
- [三、决策树：一个新凭证出现时往哪放](#三决策树一个新凭证出现时往哪放)
- [四、开发者凭证：全局 `.enc` vs 项目 `.enc`](#四开发者凭证全局-enc-vs-项目-enc)
- [五、网站/App 密码：密码管理器选型](#五网站app-密码密码管理器选型)
- [六、针对当前双机环境的推荐落地方案](#六针对当前双机环境的推荐落地方案)
- [七、反模式（不要这样做）](#七反模式不要这样做)
- [八、来源](#八来源)

---

## 一、总原则：人只记一个，其余全交给工具

记忆力有限时，安全的做法**不是**逼自己记住很多复杂密码，而是把"要记的东西"压缩到最小：

```
              你只需要记住 1 个：主口令（master passphrase）
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
     密码管理器（网站/App    本地 .enc 加密库          macOS 钥匙串 /
     登录密码，随机不重复）  （开发者 token/密钥）       ssh-agent（SSH 私钥）
```

- 主口令怎么造、怎么备份：见 [master-passphrase.md](master-passphrase.md)。
- 除主口令外的一切密码/密钥，**都不该出现在你的记忆负担里**：要么随机生成交给管理器，要么加密落盘用时解密。

## 二、五类凭证与各自的"家"

| 类别 | 例子 | 存放位置 | 人是否要记 | 备注 |
|---|---|---|---|---|
| **① 主口令** | 解锁 Mac/sudo、密码管理器主密码、`.enc` 解密口令、SSH 私钥 passphrase | 人脑 + 离线纸质备份 | **唯一要记** | 尽量让这几处用同一句主口令，降低记忆负担 |
| **② 网站/App 密码** | 网站、SaaS、社区、邮箱、各类后台 | 密码管理器，每站一个**随机且不重复**的密码 | 不记 | 自动填充；支持 TOTP 二步验证更佳 |
| **③ 开发者密钥/token** | GitHub PAT、各平台 API key/secret、webhook token、数据库连接串、OAuth client secret | **`.enc` 加密文件**（全局或项目，见下） | 不记 | 用 `secrets` 加解密；脚本运行时解密进内存 |
| **④ SSH / 签名私钥** | `~/.ssh/id_*`、GPG 私钥 | `~/.ssh`（600 权限）+ ssh-agent/钥匙串 | 私钥 passphrase = 主口令 | 公钥可公开，私钥永不外泄 |
| **⑤ 会话/一次性凭证** | 登录 session、cookie、短期 access_token、Playwright 配对 token、验证码 | 内存/临时目录，**过期即弃、不落盘** | 不记 | 确需跨进程短暂保存时放 `/tmp` 并及时删 |

> 另有一类**他人/客户敏感数据**（不是凭证但同样敏感）：最小化采集、加密存储、不进公开仓与对外交付，按最严类别对待。

## 三、决策树：一个新凭证出现时往哪放

```
拿到一个新的秘密
│
├─ 是"我本人登录某个网站/App"的密码吗？
│    └─ 是 → 密码管理器生成 16+ 随机密码存进去，自己不记（类别②）
│
├─ 是给代码/脚本/API 用的 token、key、secret、连接串吗？
│    ├─ 换个项目也要用、跟"我这个人/这台机器"绑定（如 GitHub PAT）
│    │    └─ 全局：~/.doubao/secrets/<name>.enc（永不入库）
│    └─ 只服务某一个项目（如该项目专属平台 key）
│         └─ 项目：<项目>/.secrets/<name>.enc（.enc 可随私有仓，明文不入库）
│
├─ 是 SSH/GPG 私钥吗？
│    └─ ~/.ssh 或 GPG keyring，加 passphrase、纳入 agent/钥匙串（类别④）
│
├─ 是几分钟/几小时就失效的临时票据吗？
│    └─ 只在内存/临时目录，不落盘（类别⑤）
│
└─ 是需要我脑子记住、用来解锁上面这些的吗？
     └─ 它只能有一个：主口令（类别①），见 master-passphrase.md
```

## 四、开发者凭证：全局 `.enc` vs 项目 `.enc`

加解密统一用全局命令 `secrets`（权威源在 mac-system-toolkit，算法 `aes-256-cbc + pbkdf2 + base64` 固定不改，保证双机/新旧密文互解）。**本文只讲"放哪"，不讲怎么实现。**

| 维度 | 全局凭证 | 项目凭证 |
|---|---|---|
| 路径 | `~/.doubao/secrets/<name>.enc` | `<项目>/.secrets/<name>.enc` |
| 判断口径 | 换项目/换机器也要用、与个人或机器绑定 | 只服务这一个项目 |
| 是否进 git | **永不入库**（在家目录、仓库外） | 仅 `.enc` 密文可随仓，明文绝不入库 |
| 典型 | GitHub PAT、sudo 密码、通用 API key | 项目专属平台 key、项目 webhook |
| 权限 | 目录 700、文件 600 | 同左，并在项目 `.gitignore` 放行 `!*.enc`、屏蔽明文 |

标准取用（不要把明文写进命令历史长留；可从 stdin/变量走）：

```bash
# 解密到变量，用完 unset；不要 echo 全量
TOKEN="$(ENC_PASS='<主口令>' secrets decrypt "$HOME/.doubao/secrets/github_pat.enc")"
unset TOKEN
```

## 五、网站/App 密码：密码管理器选型

> 结论先行：**怕麻烦、要在两台 Mac 和手机间无感同步 → Bitwarden（免费版即可）；完全不想让密码数据上任何云 → KeePassXC 本地库；纯苹果设备、想零设置 → iCloud 钥匙串过渡。**

| 方案 | 跨平台 | 同步 | 开源/审计 | 开发者友好 | 成本 | 适合 |
|---|---|---|---|---|---|---|
| **Bitwarden** | 全平台（Mac/Win/Linux/iOS/Android/全浏览器） | 官方云，免费版无限条目无限设备；可自托管 Vaultwarden | 开源、定期第三方审计 | 有 CLI、可存 API key/安全笔记 | 核心免费，高级功能订阅 | **大多数人的首选，推荐主线** |
| **KeePassXC** | 桌面端强；手机靠 KeePassium/Strongbox 等兼容 App | **无云，自己同步库文件**（iCloud/git/NAS/U 盘） | 开源、KDBX 开放格式、可移植性最佳 | 本地库、可脚本化 | 完全免费 | 威胁模型包含"连云都不信"、愿自己管同步 |
| **iCloud 钥匙串 / Apple 密码** | 仅苹果生态（Mac/iPhone/iPad；Windows 有有限插件） | iCloud 自动、零设置 | 闭源、苹果生态内 | 弱（不适合存工程密钥/CLI） | 系统自带免费 | 纯苹果设备的省心兜底/过渡 |
| 1Password | 全平台 | 官方云 | 闭源、口碑好 | **SSH agent、多身份管理强** | 付费订阅 | 团队付费、或重度 SSH 多身份 |

选型要点：
- **Bitwarden 与 KeePassXC 都可平滑迁移**（支持标准格式导入导出），所以"先 Bitwarden 起步、以后想完全自持再换 KeePassXC"没有锁定成本。
- 密码管理器的**主密码就用唯一主口令**（见 master-passphrase.md）；开启二步验证（TOTP）。
- **开发者 token/API key 仍以 `.enc` + CLI 自动化为主**：密码管理器侧重"人登录网站"，工程脚本无人值守取数用 `.enc` 更顺；Bitwarden CLI 可作为补充，但不替代 `.enc` 体系。
- 不再用"一个常用密码走天下"：一处泄漏会导致所有网站被撞库（credential stuffing）。

## 六、针对当前双机环境的推荐落地方案

1. **主口令**：按 master-passphrase.md 造一句 6 词级口令短语；Mac 登录/sudo、SSH 私钥 passphrase、`.enc` 解密、密码管理器主密码统一到它（更换是渐进过程，不必一次全改）。
2. **网站密码**：选 Bitwarden 免费版，两台 Mac 装桌面端+浏览器扩展、手机装 App，先把重要账号（邮箱、GitHub、云、支付、域名）迁进去并换成随机密码，其余随用随换。
3. **开发者凭证**：维持现有 `.enc` 体系——全局在 `~/.doubao/secrets/`（如 `github_pat.enc`、`sudo.enc`），项目专属在各项目 `.secrets/`；双机靠加密文件拷贝/私有渠道同步（密文可过非密通道，但主口令另走线下）。
4. **SSH 私钥**：维持 `~/.ssh` + ssh-agent/钥匙串，私钥 passphrase 与主口令一致。
5. **密码管理器库本身的备份**：Bitwarden 定期导出加密备份（导出文件本身敏感，按类别③加密保存，不要明文放云盘）。

## 七、反模式（不要这样做）

- ❌ 一个常用弱密码（如短单词+数字）用于所有网站——记一个的正确方式是"一个强主口令 + 管理器"，不是"一个弱密码复用"。
- ❌ 把 token/密码写进代码常量、配置文件并提交、聊天记录、便签、备忘录、云文档明文。
- ❌ 把私钥/密码截图发出去或贴日志（要打码，见 repo-and-leak-defense.md）。
- ❌ 在多个地方各存一份明文副本"以防万一"——副本越多泄漏面越大；要冗余就加密备份。
- ❌ 用"提示文件"明文记录主口令在电脑里；主口令只走"脑子 + 离线纸"。

## 八、来源

- EFF Diceware / 长词表与主口令建议：https://www.eff.org/dice
- NIST SP 800-63B（memorized secret 长度与熵）：https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP Password Storage Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- Bitwarden 官方（跨平台/自托管/CLI）：https://bitwarden.com/help/
- KeePassXC（KDBX、本地库）：https://keepassxc.org/docs/
- Apple 密码/iCloud 钥匙串：https://support.apple.com/zh-cn/1026014
- 2026 密码管理器横向评测（Bitwarden/KeePassXC/Apple/1Password 取舍）：https://www.webtoolkit.tech/guides/best-password-managers-for-developers-2026 ；https://www.panicvault.org/compare/best-free-password-managers/
