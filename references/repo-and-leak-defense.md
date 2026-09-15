# 公开仓防泄漏、应急响应与交付前检查

> 回答：**这个能不能提交/推送/公开？怎么防止手滑把密钥发出去？已经发出去了先做什么？对外材料怎么打码？**
> 配套：凭证该存哪见 [credential-storage.md](credential-storage.md)；加解密工具见 mac-system-toolkit `secret-encryption.md`、明文巡检见其 `audit-secrets.sh`。

## 目录
- [一、为什么公开仓要当"全世界实时可见"对待](#一为什么公开仓要当全世界实时可见对待)
- [二、分层防御（五道闸，纵深防御）](#二分层层防御五道闸纵深防御)
- [三、加密 `.enc` 能不能进公开仓](#三加密-enc-能不能进公开仓)
- [四、已经误提交 / 泄漏：应急响应 SOP（先轮换）](#四已经误提交--泄漏应急响应-sop先轮换)
- [五、对外打码与脱敏规范](#五对外打码与脱敏规范)
- [六、最小权限原则（含"主控 token 全权限"的权衡）](#六最小权限原则含主控-token-全权限的权衡)
- [七、交付 / 推送 / 公开前安全清单](#七交付--推送--公开前安全清单)
- [八、来源](#八来源)

---

## 一、为什么公开仓要当"全世界实时可见"对待

- 公开仓**包括全部 git 历史**：当前版本删掉不代表历史里没有，`git log -p` 就能翻出来。
- 自动化扫描器会持续抓取新推送的公开仓库，**密钥从 push 到被机器人扫到通常只需约 2 分钟**；云厂商长期密钥一旦被识别，可能被秒级拿去薅资源。
- 所以防线必须前移到"提交之前"，而不是指望"推上去再删"。

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

- 本体系已有轻量自研巡检 `mac-system-toolkit/scripts/audit-secrets.sh`（扫 HEAD 快照/工作区，退出码 0 干净 / 1 待确认 / 2 高置信），可先用它；要工业级规则覆盖（几百种已知 token 形态）再上 gitleaks。
- 紧急情况确需跳过：`SKIP=gitleaks git commit`（仅限已确认是假阳性，别养成习惯）。

### L3：GitHub 服务端

- 仓库 **Settings → Code security and analysis**：开启 **Secret scanning** 与 **Push protection**。
- Push protection 会在 push 当下拦截"长得像已知厂商密钥"的内容，是本地钩子失效时的兜底。

## 三、加密 `.enc` 能不能进公开仓

- `.enc` 是 `aes-256-cbc + pbkdf2 + salt + base64` 密文，**没有主口令在计算上解不开**，因此密文本身可随公开仓（便于双机/项目同步）。
- 但遵循"**能不入库优先不入库**"：跨项目/个人级凭证放全局 `~/.doubao/secrets/`（仓库外，天然不入库）；只有确实需要随项目分发、且不介意密文可见的项目级 `.enc` 才放进项目。
- **主口令在任何情况下都不进仓**（它能解开所有 `.enc`，是真正的根）；`.enc` 文件名也不要暗示内容敏感细节。

## 四、已经误提交 / 泄漏：应急响应 SOP（先轮换）

> 铁律：**密钥一旦进入公开历史，"轮换/吊销"是唯一真正的补救。** 改写历史、删除文件都不能让已经被别人看到/爬走的凭证重新变安全。顺序不能反。

1. **第一步：立即让凭证失效（最高优先，先做再说）**
   - GitHub PAT/token：到 Settings 直接 **Revoke/删除**并按需重建；
   - 云/平台 API key、webhook secret：控制台作废、重新签发；
   - 若是密码：立即改密码并检查同密码的其它账号；
   - 涉及服务器：轮换对应 SSH 私钥/部署密钥。
2. **第二步：评估暴露面**：是否 public、从 push 到发现多久、有无被 fork/clone/扫描告警、该密钥权限范围（决定还要不要查异常调用/账单/审计日志）。
3. **第三步：清痕迹（在凭证已失效之后）**
   - 从**全部历史**移除：`git filter-repo --invert-paths --path <敏感文件>` 或 BFG Repo-Cleaner；普通 `git rm` 只删当前版本，历史仍在。
   - 强推更新所有分支与 tag；通知已 clone 的协作者重新同步。
4. **第四步：加固，防止复发**：补 `.gitignore`、装 L2 钩子、开 L3 push protection、把明文改造成 `.enc`/环境变量。
5. **第五步：记录复盘**：泄漏了什么、影响、处置时间线、改进项（写入对应项目决策记录/双机台账，只记非密信息）。

> 仅在**私有仓、从未公开、且确认没有任何泄漏迹象**时，才可以"只清历史、不轮换"；拿不准就按已泄漏处理（先轮换）。

## 五、对外打码与脱敏规范

对外 = 截图、日志、文档、聊天、演示、交付物、录屏、报错贴。默认假设会被无关方看到。

- **统一掩码格式**：只保留可辨识的少量前后缀，中间用 `…`，如 `ghp_HEB…8bC5`、邮箱 `49***@qq.com`、手机 `138****0000`。
- **文本类不要只靠"打马赛克"**：命令输出/日志/代码里的明文要**替换成占位符再发**；图片截图用色块完全覆盖（注意别只盖一半、别留可还原的模糊）。
- 必查的高风险串：token/API key/secret/密码/私钥、cookie/Authorization 头、`.enc` 主口令、内网 IP/端口、账号邮箱/手机/身份证/客户数据、`.doubao/secrets` 绝对路径里的用户名（双机用户名不同，对外用 `$HOME`）。
- 命令历史：含明文的命令避免反复执行；需要展示命令时用 `ENC_PASS='<主口令>'` 这类占位。
- 拿不准某串是不是敏感 → 按敏感处理。

## 六、最小权限原则（含"主控 token 全权限"的权衡）

- 一般原则：token/key 只授予完成任务所需的最小 scope、最短有效期、限定可访问资源；能用只读不用写、能用单仓不用全账户。
- **例外与权衡**：用于长期自动化、需要持续管理多个仓库的**主控 PAT** 可以申请较全权限，但必须同时满足：加密落盘（`.enc`）、绝不进仓/不外露/对外打码、限定在可信本机、出现在日志即轮换。全权限不是问题，**全权限 + 明文散落**才是问题。
- 给外部/临时/CI 的凭证一律最小权限 + 短有效期 + 可独立吊销，不与主控凭证混用。

## 七、交付 / 推送 / 公开前安全清单

提交、推送、把仓库设为 public、发交付物或对外材料前逐项过：

- [ ] 全仓没有明文密码/token/私钥/连接串/`.env`（必要时跑 `audit-secrets.sh` 或 `gitleaks git .`，含历史）
- [ ] `.gitignore` 已屏蔽明文、仅 `!*.enc` 放行；全局凭证仍在仓库外
- [ ] 代码不硬编码秘密，改为环境变量/运行时解密；示例值都是占位符
- [ ] 要进仓的 `.enc` 确属项目级、且主口令未随仓；主口令只在本地/离线纸
- [ ] 提交信息、文件名、注释里没有真实口令/内部地址/客户信息
- [ ] 截图/日志/文档/录屏已按第五节打码，路径用 `$HOME`
- [ ] 对外凭证遵循最小权限 + 可吊销；主控全权限 token 已加密、未外泄
- [ ] 公开仓已开 Secret scanning / Push protection（能开则开）
- [ ] 若曾误提交：已先完成轮换/吊销，再清历史，再加固

## 八、来源

- GitHub Docs · Preventing data leaks / Secret scanning / Push protection：https://docs.github.com/en/code-security/tutorials/secure-your-organization/prevent-data-leaks
- gitleaks（pre-commit、历史扫描、CI）：https://github.com/gitleaks/gitleaks
- "提交后轮换是唯一真正补救、新推送密钥约 2 分钟被扫到"：https://www.aicodingguild.com/blog/gitleaks-stop-secrets-before-they-ship ；https://geekworkbench.com/blog/technical/git-secrets-management-pre-commit
- git filter-repo（清历史）：https://github.com/newren/git-filter-repo
- OWASP Secrets Management Cheat Sheet：https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- 12-Factor · 配置与代码分离：https://12factor.net/config
