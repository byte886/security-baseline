# 开发凭证的"来处"：分级与落点

> 回答两个问题：**①开发/AI 要用的这个东西算哪类凭证？②它该存哪、从哪取？**
> 管理对象是**开发与自动化要消费的凭证**（PAT、API key、SSH 私钥、项目密码等）；用户个人各网站登录密码不在本技能收集/管理范围（见 [master-passphrase.md](master-passphrase.md)）。
> 配套：取密 SOP 与不泄漏纪律见 [ai-agent-credentials.md](ai-agent-credentials.md)；不进 git 见 [repo-and-leak-defense.md](repo-and-leak-defense.md)；加解密命令用法见 mac-system-toolkit `secret-encryption.md`（本文不复制）。

## 目录
- [一、总原则：每个开发凭证都有确定的加密来处](#一总原则每个开发凭证都有确定的加密来处)
- [二、凭证分类与各自的"家"](#二凭证分类与各自的家)
- [三、决策树：新凭证往哪放](#三决策树新凭证往哪放)
- [四、全局 `.enc` vs 项目 `.enc`](#四全局-enc-vs-项目-enc)
- [五、典型工具的凭证来处对照](#五典型工具的凭证来处对照)
- [六、要不要给 agent 装"密码管理器"](#六要不要给-agent-装密码管理器)
- [七、反模式](#七反模式)
- [八、来源](#八来源)

---

## 一、总原则：每个开发凭证都有确定的加密来处

目标是：**任何开发工具需要凭证时，不用临时找用户、不硬编码，直接到它固定的加密来处取；并且无论直接还是间接，都暴露不到用户主口令明文。**

```
                 用户只当次提供：<主口令>（解密钥匙，用后即弃）
                                     │
        ┌────────────────────────────┼───────────────────────────┐
        ▼                            ▼                           ▼
 全局 ~/.doubao/secrets/*.enc   项目 <项目>/.secrets/*.enc   macOS 钥匙串 / ssh-agent
 （跨项目、与人/机绑定）          （只服务该项目）              （SSH 私钥/系统凭据）
 例：github_pat、sudo            例：某平台 API、webhook       例：id_rsa_* passphrase
        └──────────── 解密后进内存，喂给 gh/git/脚本/API，用完 unset ───────────┘
```

- 主口令怎么向用户要、怎么不残留：[master-passphrase.md](master-passphrase.md)。
- AI/脚本具体怎么取、怎么不进对话：[ai-agent-credentials.md](ai-agent-credentials.md)。

## 二、凭证分类与各自的"家"

| 类别 | 例子 | 存放来处 | 进 git？ | 谁来解锁 |
|---|---|---|---|---|
| **① 主口令** | 解锁 `.enc`、SSH 私钥 passphrase、钥匙串的那一把 | 用户人脑，交互时给；非交互才 `master.pass`(600) | 永不 | 用户当次提供 |
| **② 全局开发者凭证** | GitHub PAT、sudo 密码、跨项目通用 API key | `~/.doubao/secrets/<name>.enc` | **永不入库**（家目录、仓库外） | 主口令解密 |
| **③ 项目凭证** | 项目专属平台 key/secret、webhook、项目数据库连接串 | `<项目>/.secrets/<name>.enc` | 仅 `.enc` 可随仓，明文绝不入库 | 主口令解密 |
| **④ SSH/签名私钥** | `~/.ssh/id_*`、GPG 私钥 | `~/.ssh`(600) + ssh-agent/钥匙串 | 私钥永不；公钥可公开 | passphrase 入 agent |
| **⑤ 会话/一次性凭证** | session、cookie、短期 access_token、Playwright 配对码、TOTP 动态码 | 内存/临时目录，过期即弃 | 不入库 | 即用即弃 |
| （旁类）他人/客户敏感数据 | 客户资料、私域数据、证件号 | 最小留存、加密、不进对外交付与公开仓 | 不进公开仓 | 按最严类别 |

> 用户个人网站登录密码不属于①–⑤：不收集、不迁移、不要求统一；只有当某个密码**被开发流程使用**（例如脚本要登录某平台）时，它才作为②/③加密落盘。

## 三、决策树：新凭证往哪放

```
开发中出现一个要用的秘密
│
├─ 是 SSH/GPG 私钥？ → ~/.ssh 或 keyring(600)，passphrase 交 ssh-agent/钥匙串（④）
│
├─ 是几分钟/几小时失效的临时票/cookie/验证码？ → 只在内存/临时目录，不落盘（⑤）
│
├─ 是 token/key/secret/密码/连接串，给代码或工具用？
│    ├─ 换项目/换机器也要用、跟人或机器绑定（如 GitHub PAT、sudo）
│    │     → 全局 ~/.doubao/secrets/<name>.enc（②，永不入库）
│    └─ 只服务当前项目
│          → 项目 <项目>/.secrets/<name>.enc（③，仅 .enc 可随仓）
│
└─ 是用来解密上面这些的主口令本身？ → 不落盘为常态；交互向用户要（①，见 master-passphrase）
```

## 四、全局 `.enc` vs 项目 `.enc`

统一用全局命令 `secrets`（权威源在 mac-system-toolkit，算法 `aes-256-cbc + pbkdf2 + base64` 固定不改，保证双机/新旧密文互解）。**本篇只定"放哪"，不重复实现。**

| 维度 | 全局凭证 | 项目凭证 |
|---|---|---|
| 路径 | `~/.doubao/secrets/<name>.enc` | `<项目>/.secrets/<name>.enc` |
| 口径 | 换项目/换机器也要用、与人或机器绑定 | 只服务这一个项目 |
| 进 git | 永不入库（家目录、仓库外） | 仅 `.enc` 密文可随仓，明文绝不入库 |
| 权限 | 目录 700、文件 600 | 同左，项目 `.gitignore` 屏蔽明文、放行 `!*.enc` |
| 典型 | `github_pat.enc`、`sudo.enc`、通用 API | 项目平台凭证、项目 webhook |

项目 `.gitignore` 必配：

```gitignore
.secrets/*.json .secrets/*.txt .secrets/*.raw .secrets/*.key .secrets/*.pem
!*.enc
```

取用（解密到变量、用完 unset，不回显）：

```bash
TOKEN="$(ENC_PASS='<主口令>' secrets decrypt "$HOME/.doubao/secrets/github_pat.enc")"
# …用 $TOKEN…
unset TOKEN
```

## 五、典型工具的凭证来处对照

| 工具/场景 | 用什么凭证 | 来处 | 一次性配置动作 |
|---|---|---|---|
| `gh`（建仓/API/PR） | GitHub PAT | `~/.doubao/secrets/github_pat.enc` | 解密管道 `gh auth login --with-token`，之后 gh 自持登录态 |
| `git push`（SSH） | SSH 私钥 | `~/.ssh/id_*` + ssh-agent/钥匙串 | `ssh-add --apple-use-keychain` 一次，之后免输 |
| `sudo` | 登录口令 | 优先用户终端亲输；自动化用 `sudo.enc` | 不建议把 sudo 口令长期塞进脚本 |
| 第三方平台脚本 | API key/secret | 项目 `.secrets/*.enc` | 脚本运行时解密进环境变量 |
| 2FA 登录 | TOTP 6 位码 | 用户 Microsoft Authenticator | 当次向用户要，用后即弃 |
| 浏览器自动化登录态 | storage-state/cookie | 临时文件、用后删、不进公开仓 | 见对应技能，按⑤对待 |

> 双机具体有哪些凭证、路径与账号现状（含 GitHub 账号、SSH 别名、各 `.enc` 清单）查 dual-machine-manager 的 `credentials.md` / `security-and-git.md`，本技能不记具体值。

## 六、要不要给 agent 装"密码管理器"

- **默认不需要**：个人开发场景下，`.enc` + `secrets` + macOS 钥匙串/ssh-agent 已覆盖"加密来处 + 非交互取用"，不必再引入 Bitwarden/1Password/KeePass。
- 这些个人密码管理器是给**人管理网站密码**用的；用户已明确不需要为其个人用途安装。
- 仅当出现下列情况再考虑 agent-side secret 工具（如 Bitwarden CLI `rbw`/`op`、系统钥匙串命令行）：需要在多机/多人共享大量密钥、需要细粒度访问审计、或 `.enc` 文件方案明显不够用。届时是"给自动化用"，不是给用户增加记忆负担。
- `.env` 只作**传输层**（把已解密的值传给进程），不是存储层；权威值永远在 `.enc`，`.env` 被 `.gitignore` 屏蔽。

## 七、反模式

- ❌ 把 token/密码/主口令写进代码常量、配置并提交、聊天记录、便签、云文档、commit message。
- ❌ 解密后 `echo`/打印到对话或日志，或写入会被提交/同步的文件。
- ❌ 到处复制明文副本"以防万一"；要冗余就加密多设备/多来处。
- ❌ 把 `.env` 当密钥权威存储；把全局凭证误放进项目目录导致可能入库。
- ❌ 为了"省事"去改造用户密码、或给用户装其不需要的个人密码管理器。
- ❌ 凭历史记忆猜测主口令/凭证值；拿不到就交互问用户。

## 八、来源

- 加解密两级落点与 `.gitignore` 规范：mac-system-toolkit `references/secret-encryption.md`（权威实现）。
- AI/agent 凭证取用与"环境变量是传输层"：[ai-agent-credentials.md](ai-agent-credentials.md) 及其来源（Auth0 / WorkOS 等）。
- OWASP Secrets Management Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- GitHub 防泄漏：https://docs.github.com/en/code-security/tutorials/secure-your-organization/prevent-data-leaks
