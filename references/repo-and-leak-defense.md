# 公开仓防泄漏：为什么要当"全世界可见"对待 + 五道闸

> 回答：**这个能不能提交/推送/公开？怎么防止手滑把密钥发出去？公开仓的分层防御怎么搭？**
> 配套：凭证该存哪见 [credential-storage.md](credential-storage.md)；加解密与明文巡检由 mac-system-toolkit 对应能力提供（巡检命令 `audit-secrets.sh`）。
> **已经误提交/泄漏怎么办、对外怎么打码、交付前怎么自查** → 见 [leak-response-redaction-checklist.md](leak-response-redaction-checklist.md)。

## 目录
- [一、为什么公开仓要当"全世界实时可见"对待](#一为什么公开仓要当全世界实时可见对待)
- [二、分层防御（五道闸，纵深防御）](#二分层防御五道闸纵深防御)
- [三、加密 `.enc` 能不能进公开仓](#三加密-enc-能不能进公开仓)
- [四、来源](#四来源)

---

## 一、为什么公开仓要当"全世界实时可见"对待

- 公开仓**包括全部 git 历史**：当前版本删掉不代表历史里没有，`git log -p` 就能翻出来。
- 自动化扫描器会持续抓取新推送的公开仓库，**密钥从 push 到被机器人扫到通常只需约 2 分钟**；云厂商长期密钥一旦被识别，可能被秒级拿去薅资源。
- 所以防线必须前移到"提交之前"，而不是指望"推上去再删"。
- **"公开"的不只是密钥**：真实姓名、账号 ID/昵称、群 ID/聊天标识、手机号、个人邮箱、身份证、客户数据，以及从本机 dump 的数据库/数据字典、本人实测日志等**个人数据与隐私内容**同样会被任何人看到，危害是身份关联、社工与隐私泄露，不止于资源被盗刷。
- **入不入库只看"能否对公众公开"，与它是运行面代码还是 `docs/` 文档无关**——"AI 加不加载"和"能不能被全世界看到"是两个正交维度。`docs/` 不是"私有区"，不要为图省事封掉整个目录（会误伤可公开的通用文档）。
- **含个人/本机数据的文件只有三种归宿，不设"挪到仓外私有区"第四态**：① 能脱敏成通用内容的（真实标识换占位符/虚构示例、剥离个人清单与统计）→ **明文随仓**；② 无法脱敏、但确有保留价值的 → 加密成 `.enc` **随仓**（无主口令解不开，见第三节）；③ 无保留价值、可随时重新生成的过程件（本人实测流水日志、原始 dump、临时导出）→ **删除**。`.gitignore` 精确到文件名只是防误提交的兜底，不是长期藏匿手段。
  - 实战范例：微信技能从本机 dump 的 4800 余行数据字典，删去账号行、200 个 `Msg_<MD5>` 会话分表清单、各表记录条数与库文件大小后，沉淀为 400 余行纯表结构文档，明文随仓（归宿①）；本人实测流水日志直接删（归宿③）。
  - 例外（不算"挪仓外"）：**跨项目根凭证**（GitHub PAT、sudo/主口令等）本就不属于任何项目仓，统一放仓外全局 `~/.doubao/secrets/*.enc`（见 credential-storage.md）；这与"项目资料三归宿"是两回事，不矛盾。

## 二、分层防御（五道闸，纵深防御）

| 层 | 目标 | 手段 | 何时做 |
|---|---|---|---|
| **L0 设计** | 秘密与代码分离 | 配置走环境变量 / `.enc` / 密钥管理，代码只读引用，遵循 12-factor | 写代码时 |
| **L1 不入库** | 让明文根本进不了暂存区 | `.gitignore` 屏蔽明文、统一加密落盘、不硬编码 | 日常 |
| **L2 提交前** | 拦在 commit 之前（shift-left） | pre-commit 钩子：`gitleaks protect --staged`；或本体系 `audit-secrets.sh` | 每次 commit |
| **L3 推送时** | 服务端最后一道闸 | GitHub Secret scanning + **Push protection**（识别到已知密钥形态直接拒绝 push） | push 时 |
| **L4 历史/CI** | 兜底存量与漏网 | `gitleaks git .` 扫历史、CI 里跑 gitleaks-action | 定期 / CI |

### L1：`.gitignore` 密钥段（每个含凭证的仓库都配）

```gitignore
# 凭证：明文绝不入库，仅 .enc 加密件放行
.secrets/*.json
.secrets/*.txt
.secrets/*.raw
.secrets/*.key
.secrets/*.pem
.env
.env.*
!.env.example
*.pem
*.key
id_rsa*
!*.enc
```

配套：代码里需要密钥时从环境变量读，或运行时用 `secrets decrypt` 解到变量，用完 `unset`。

### L2：本地提交前钩子（gitleaks）

```bash
# 安装 gitleaks（macOS: brew install gitleaks）
# 在仓库根写 .git/hooks/pre-commit：
#!/bin/sh
gitleaks protect --verbose --redact --staged     # 扫暂存区，命中即阻断提交
# chmod +x .git/hooks/pre-commit
```

- 本体系已有轻量自研巡检命令 `audit-secrets.sh`（由 mac-system-toolkit 提供；扫 HEAD 快照/工作区，退出码 0 干净 / 1 待确认 / 2 高置信），可先用它；要工业级规则覆盖（几百种已知 token 形态）再上 gitleaks。
- 紧急情况确需跳过：`SKIP=gitleaks git commit`（仅限已确认是假阳性，别养成习惯）。

### L3：GitHub 服务端

- 仓库 **Settings → Code security and analysis**：开启 **Secret scanning** 与 **Push protection**。
- Push protection 会在 push 当下拦截"长得像已知厂商密钥"的内容，是本地钩子失效时的兜底。

## 三、加密 `.enc` 能不能进公开仓

- `.enc` 是 `aes-256-cbc + pbkdf2 + salt + base64` 密文，**没有主口令在计算上解不开**，因此密文本身可随公开仓（便于双机/项目同步）。
- 但遵循"**能不入库优先不入库**"：跨项目/个人级凭证放全局 `~/.doubao/secrets/`（仓库外，天然不入库）；只有确实需要随项目分发、且不介意密文可见的项目级 `.enc` 才放进项目。
- **主口令在任何情况下都不进仓**（它能解开所有 `.enc`，是真正的根）；`.enc` 文件名也不要暗示内容敏感细节。

## 四、来源

- GitHub Docs · Preventing data leaks / Secret scanning / Push protection：https://docs.github.com/en/code-security/tutorials/secure-your-organization/prevent-data-leaks
- gitleaks（pre-commit、历史扫描、CI）：https://github.com/gitleaks/gitleaks
- OWASP Secrets Management Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- 12-Factor · 配置与代码分离：https://12factor.net/config
