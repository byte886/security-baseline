# 凭证分级与"到底存哪里"

> 回答两个问题：**①这东西算哪类凭证？②它该交给谁存、我自己要不要记？**
> 配套：弱记忆友好的"人要记的口令/passkey"方案见 [master-passphrase.md](master-passphrase.md)；AI/agent 用的密钥怎么与人的密码分开见 [ai-agent-credentials.md](ai-agent-credentials.md)；怎么保证不进 git 见 [repo-and-leak-defense.md](repo-and-leak-defense.md)；加解密工具用法见 mac-system-toolkit 的 `secret-encryption.md`（本文不复制）。

## 目录
- [一、总原则：先消除"要记/要输"，人只记本地层一句](#一总原则先消除要记要输人只记本地层一句)
- [二、五类凭证与各自的"家"](#二五类凭证与各自的家)
- [三、决策树：一个新凭证出现时往哪放](#三决策树一个新凭证出现时往哪放)
- [四、开发者凭证：全局 `.enc` vs 项目 `.enc`](#四开发者凭证全局-enc-vs-项目-enc)
- [五、网站/App 密码与 passkey：工具选型](#五网站app-密码与-passkey工具选型)
- [六、针对当前双机环境的推荐落地方案](#六针对当前双机环境的推荐落地方案)
- [七、反模式（不要这样做）](#七反模式不要这样做)
- [八、来源](#八来源)

---

## 一、总原则：先消除"要记/要输"，人只记本地层一句

记忆力有限、又怕手动输入时，正确做法**不是**逼自己记很多复杂密码，而是按顺序把记忆负担消掉：

```
 在线账户优先 passkey（指纹/扫码，免记免输）
        │  不支持 passkey 的
        ▼
 密码管理器随机密码 + 生物识别自动填充（人不记、日常不手输）
        │  剩下真正要手输的
        ▼
 本地解锁口令（Mac/sudo、管理器主密码、.enc、SSH passphrase）
        —— 唯一要记的一类，几处统一成"记得住的中等强度"一句
        │  给 AI / 脚本用的（并行独立轨道，人完全不记）
        ▼
 .enc 加密库 + 本机 master 文件/钥匙串，执行层自动取密
```

- 怎么把在线账户换成 passkey、本地那句口令设成什么强度、不用纸怎么防遗忘：见 [master-passphrase.md](master-passphrase.md)。
- AI/自动化密钥为何与人密码分轨、怎么不进对话：见 [ai-agent-credentials.md](ai-agent-credentials.md)。
- 除"本地层一句"外的一切密码/密钥，**都不该进你的记忆负担**：能 passkey 就 passkey，否则随机交给管理器，开发者密钥加密落盘用时解密。

## 二、五类凭证与各自的"家"

| 类别 | 例子 | 存放位置 | 人是否要记 | 备注 |
|---|---|---|---|---|
| **① 本地解锁口令** | Mac 登录/sudo、密码管理器主密码、`.enc` 解密口令、SSH 私钥 passphrase | 人脑（几处统一成一句）+ 数字冗余备份（见 master-passphrase §六，不用纸） | **唯一要记的一类** | 只在本地、从不上线；"记得住的中等强度"即可，靠生物识别压低输入频率 |
| **② 在线账户登录** | 网站、SaaS、邮箱、社区、后台 | **优先 passkey**（指纹/扫码）；不支持的用密码管理器随机不重复密码 | 不记 | passkey 免记免输抗钓鱼；随机密码靠自动填充；TOTP 用独立验证器 |
| **③ 开发者密钥/token** | GitHub PAT、API key/secret、webhook、数据库连接串、OAuth secret | **`.enc` 加密文件**（全局或项目，见下） | 不记 | `secrets` 加解密、运行时解密进内存；不进模型对话，详见 ai-agent-credentials.md |
| **④ SSH / 签名私钥** | `~/.ssh/id_*`、GPG 私钥 | `~/.ssh`（600）+ ssh-agent/钥匙串 | passphrase 同①那句 | 公钥可公开，私钥永不外泄，agent 免重复输 |
| **⑤ 会话/一次性凭证** | 登录 session、cookie、短期 access_token、Playwright 配对 token、验证码 | 内存/临时目录，**过期即弃、不落盘** | 不记 | 确需短暂跨进程保存放 `/tmp` 并及时删 |

> 另有一类**他人/客户敏感数据**（非凭证但同样敏感）：最小化采集、加密存储、不进公开仓与对外交付，按最严类别对待。

## 三、决策树：一个新凭证出现时往哪放

```
拿到一个新的秘密
│
├─ 是"我本人登录某网站/App"吗？
│    ├─ 该站支持 passkey → 直接开 passkey（指纹/扫码），不再依赖密码（类别②）
│    └─ 不支持 → 密码管理器生成 16+ 随机密码存进去，自己不记（类别②）
│
├─ 是给代码/脚本/AI 用的 token、key、secret、连接串吗？
│    ├─ 换项目/机器也要用、跟人或机器绑定（如 GitHub PAT）
│    │    └─ 全局：~/.doubao/secrets/<name>.enc（永不入库）
│    └─ 只服务某一项目
│         └─ 项目：<项目>/.secrets/<name>.enc（仅 .enc 可随仓，明文不入库）
│
├─ 是 SSH/GPG 私钥吗？
│    └─ ~/.ssh 或 GPG keyring，加 passphrase、纳入 agent/钥匙串（类别④）
│
├─ 是几分钟/几小时失效的临时票据吗？
│    └─ 只在内存/临时目录，不落盘（类别⑤）
│
└─ 是用来解锁上面这些、需要我脑子记住的吗？
     └─ 它只能有一类：本地解锁口令（类别①），见 master-passphrase.md
```

## 四、开发者凭证：全局 `.enc` vs 项目 `.enc`

加解密统一用全局命令 `secrets`（权威源在 mac-system-toolkit，算法 `aes-256-cbc + pbkdf2 + base64` 固定不改，保证双机/新旧密文互解）。**本文只讲"放哪"，不讲实现。**

| 维度 | 全局凭证 | 项目凭证 |
|---|---|---|
| 路径 | `~/.doubao/secrets/<name>.enc` | `<项目>/.secrets/<name>.enc` |
| 判断口径 | 换项目/换机器也要用、与个人或机器绑定 | 只服务这一个项目 |
| 是否进 git | **永不入库**（家目录、仓库外） | 仅 `.enc` 密文可随仓，明文绝不入库 |
| 典型 | GitHub PAT、sudo 密码、通用 API key | 项目专属平台 key、项目 webhook |
| 权限 | 目录 700、文件 600 | 同左，项目 `.gitignore` 放行 `!*.enc`、屏蔽明文 |

标准取用（解密到变量、用完 `unset`，**不回显明文、不贴进 AI 对话**）：

```bash
TOKEN="$(ENC_PASS='<本地解锁口令>' secrets decrypt "$HOME/.doubao/secrets/github_pat.enc")"
# ……用 $TOKEN 调命令……
unset TOKEN
```

> AI/agent 场景下更完整的取密纪律（引用 vs 值、环境变量边界、非交互 master.pass）见 [ai-agent-credentials.md](ai-agent-credentials.md)。

## 五、网站/App 密码与 passkey：工具选型

### 5.1 先 passkey，再密码管理器

- **第一选择是 passkey**：重要在线账户（主邮箱、GitHub、微软、云、银行）在安全设置里创建通行密钥，日常指纹/扫码登录，免记免输、通常连 TOTP 都可省。Android 侧系统级 passkey 提供方是 **Google 密码管理器**；跨平台统一管理可用 Bitwarden/1Password 作 passkey provider。无指纹的 Mac 用手机扫码 + 蓝牙近场认证。
- **不支持 passkey 的网站**才落到密码管理器随机密码 + 自动填充。

### 5.2 Microsoft Authenticator 的正确定位（重要，避免踩空）

- 它的**密码自动填充已于 2025 年 8 月停用**（已存密码改由 Microsoft Edge 查看管理，Chrome 扩展 2024 年底退役）——**不能再把它当全平台密码库**。
- 它仍保留两项能力：**TOTP 动态验证码**、**passkey 提供方**（passkey 需 Android 14+/iOS 17+）。
- 因此零成本分工：**密码管理器管密码，Microsoft Authenticator 管动态码/passkey**，不必为 TOTP 再买密码管理器付费版。

### 5.3 密码管理器选型

> 结论先行：**安卓 + 两台 Mac、要免费无感同步 → Bitwarden（免费版）为主线；完全不信任何云 → KeePassXC 本地库；纯苹果、想零设置 → iCloud 钥匙串过渡。**

| 方案 | 跨平台 | 同步 | 开源/审计 | passkey/TOTP | 成本 | 适合 |
|---|---|---|---|---|---|---|
| **Bitwarden** | 全平台 + 全浏览器 | 官方云，免费版无限条目/设备；可自托管 Vaultwarden | 开源、定期审计 | 可作跨平台 passkey provider；**TOTP 自动生成是付费功能**（故 TOTP 交给免费的 Authenticator） | 核心免费 | **主线推荐** |
| **Google 密码管理器** | Android/Chrome 原生 | Google 账号同步、零安装 | 闭源 | 系统级 passkey，安卓体验最顺 | 免费 | 安卓侧 passkey/轻量密码兜底；Mac 非 Chrome 场景较弱 |
| **KeePassXC** | 桌面强，手机靠兼容 App | **无云，自管库文件**同步 | 开源、KDBX 开放 | TOTP 可本地生成 | 完全免费 | "连云都不信"、愿自管同步（与"简单"相悖，个人一般不必） |
| **iCloud 钥匙串** | 仅苹果生态 | iCloud 自动、零设置 | 闭源 | passkey 顺 | 系统自带 | 纯苹果设备过渡 |
| **1Password** | 全平台 | 官方云 | 闭源、口碑好 | passkey/TOTP/SSH agent 强 | 付费 | 重度多身份/团队 |

选型要点：
- Bitwarden 与 KeePassXC 都支持标准格式导入导出，**先 Bitwarden 起步、以后想自持再换 KeePassXC，无锁定成本**。
- 管理器**主密码用本地解锁那句口令**（见 master-passphrase.md），并给管理器本身开第二因素；日常用生物识别解锁，主密码只偶尔手输。
- **开发者 token/API key 仍以 `.enc` + CLI 自动化为主**：管理器侧重"人登录网站"，无人值守脚本用 `.enc` 更顺（见 ai-agent-credentials.md）。
- 不再"一个常用密码走天下"：一处泄漏会导致所有网站被撞库（credential stuffing）。

## 六、针对当前双机环境的推荐落地方案

1. **passkey 先行**：给主邮箱、GitHub（已开 2FA，可直接升级）、微软账号在安卓手机创建 passkey，验证指纹/扫码可登录、旧密码保留兜底；每个重要账户配 2 份 passkey。
2. **密码管理器**：选 Bitwarden 免费版，两台 Mac 装浏览器扩展、手机装 App 并开指纹解锁；先把重要账号录入并换随机密码，其余随用随换。
3. **动态码**：TOTP 继续用 Microsoft Authenticator；各账户恢复码作为安全笔记存进 Bitwarden 对应条目（数字冗余，不用纸）。
4. **本地解锁口令**：Mac 登录/sudo、SSH passphrase、`.enc` 解密、Bitwarden 主密码逐步统一到 master-passphrase.md 第五节那句"记得住的中等强度"；开 FileVault、关自动登录（渐进、不一次全改）。
5. **开发者凭证**：维持 `.enc`——全局 `~/.doubao/secrets/`（如 `github_pat.enc`、`sudo.enc`），项目专属在各项目 `.secrets/`；AI/脚本经执行层解密取密、不进对话；双机经安全渠道同步 `.enc` 密文，本地解锁口令另走线下。
6. **密码库自身备份**：Bitwarden 定期导出加密备份（导出文件本身按类别③加密保存，绝不明文放云盘）；手机+两 Mac 同时登录即活冗余。

## 七、反模式（不要这样做）

- ❌ 一个常用弱密码用于所有网站——"只记一个"的正确形态是"本地一句 + passkey/管理器"，不是"一个弱密码复用"。
- ❌ 要求自己背 6 词随机强口令、还得手写纸备份，结果因为太难输而改弱/写得到处都是——先靠 passkey/生物识别消除输入，再谈强度。
- ❌ 把 token/密码写进代码常量、配置并提交、聊天记录、便签、备忘录、云文档明文；尤其别把解密出的密钥贴回 AI 对话。
- ❌ 把 `.env` 当密钥的唯一存储（它是传输层）；私钥/密码截图外发或贴日志（要打码）。
- ❌ 多处各留一份明文副本"以防万一"——副本越多泄漏面越大；要冗余就加密多设备同步。
- ❌ 用"提示文件"在电脑里明文记录本地解锁口令；防遗忘走 master-passphrase §六的数字冗余，而不是明文便签或纸张。

## 八、来源

- passkey 优先、免记免输/抗钓鱼/跨设备：Google for Developers https://developers.google.com/identity/passkeys ；2026 实操 https://www.dailycruncher.com/passkeys-2026-passwordless-guide
- Microsoft Authenticator 密码填充停用、保留 TOTP/passkey：https://support.microsoft.com/en-US/authenticator/changes-to-microsoft-authenticator-autofill
- passkey vs 密码管理器（管理器仍负责 fallback 密码/恢复码/API 密钥）：https://vucense.com/tech-guides/security-101/passkeys-vs-password-managers-2026/
- 恢复码存密码管理器（数字冗余）：https://2fa.zip/blog/store-backup-codes-securely/
- EFF Diceware / NIST SP 800-63B（何时需要更强主口令）：https://www.eff.org/dice ；https://pages.nist.gov/800-63-4/sp800-63b.html
- OWASP Password Storage Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html
- Bitwarden / KeePassXC / iCloud 钥匙串官方与 2026 横评：https://bitwarden.com/help/ ；https://keepassxc.org/docs/ ；https://www.panicvault.org/compare/best-free-password-managers/
