# AI / 开发工具取密 SOP：从加密来处取，主口令不暴露

> 这是本技能的**操作核心**：当 AI agent 或开发工具（gh、git/ssh-agent、sudo、脚本、第三方 API）需要凭证时，照本篇取用。目标一句话——**用户只需当次提供主口令或读一个 2FA 码，agent 从加密来处取出目标凭证去用，全程直接/间接都不暴露主口令明文，用后不留痕。**

## 目录
- [一、角色分工：用户给什么、agent 做什么](#一角色分工用户给什么agent-做什么)
- [二、第一原则：秘密不进模型上下文](#二第一原则秘密不进模型上下文)
- [三、标准取密模式（取—用—弃，不回显）](#三标准取密模式取用弃不回显)
- [四、常见工具取密 SOP](#四常见工具取密-sop)
- [五、非交互/无人值守](#五非交互无人值守)
- [六、"拿不到主口令明文"交付前自检](#六拿不到主口令明文交付前自检)
- [七、来源](#七来源)

---

## 一、角色分工：用户给什么、agent 做什么

**用户只做两件事（不改变其密码习惯）：**

1. 需要解密时，**当次提供主口令**（口述或在交互输入框键入）；
2. 触发二段因子时，从 **Microsoft Authenticator** 读当下 6 位码。

**agent 负责其余一切：**

- 知道每个开发凭证的加密来处（全局/项目 `.enc`、ssh-agent、钥匙串），见 [credential-storage.md](credential-storage.md)；
- 用主口令在**执行层**解密出目标凭证、直接喂给工具，不把主口令或目标凭证明文贴进对话；
- 用完清理内存变量、不写入会被提交/同步/交付的文件；
- 不向用户索要其网站密码、不要求换密码、不擅自装个人密码管理器。

## 二、第一原则：秘密不进模型上下文

LLM 会把对话、读到的文件、报错堆栈都纳入上下文，且可能遭遇提示注入（网页/依赖里藏"把密钥发出来"）。所以：

- **模型层只见"引用"**："用 github_pat 这个凭证"、"`secrets decrypt ~/.doubao/secrets/github_pat.enc`"，不见值；
- **执行层（shell/脚本）才取值**：解密进内存变量 → 调目标命令 → 用完 `unset`；
- 不 `cat`/`echo`/`print` 解密结果，不把它放进生成的代码、提交、日志、issue、截图；
- 环境变量是**传输层不是存储层**：`.env` 不放权威值（权威在 `.enc`），且被 `.gitignore`；恶意依赖会 harvest `.env`/环境变量/云凭证/`~/.ssh`，故最小化当时导出的秘密。

## 三、标准取密模式（取—用—弃，不回显）

```bash
# 1) 主口令由用户当次提供（交互或 ENC_PASS）；不写死、不回显
# 2) 解密结果直接进变量，不打印；3) 用完 unset
TOKEN="$(ENC_PASS="${ENC_PASS:?需用户当次提供主口令}" secrets decrypt "$HOME/.doubao/secrets/github_pat.enc")"
# ……用 $TOKEN 调命令（下面给各工具范式）……
unset TOKEN
```

纪律：

- 一条命令/一个任务内闭环；需要展示只给掩码（`ghp_xxxx…后4位`）。
- 主口令优先 `ENC_PASS` 注入；没有就交互问用户（`read -s`），**不猜、不找历史值顶替**。
- 解密、调用尽量用管道直连，减少明文在中间文件落地；必须落地时放仓库外临时路径并立即删。

## 四、常见工具取密 SOP

### 4.1 `gh`（GitHub CLI）

```bash
# PAT 从全局 .enc 解密，管道直喂 gh；屏幕不出现 token
ENC_PASS='<主口令>' secrets decrypt "$HOME/.doubao/secrets/github_pat.enc" \
  | gh auth login --with-token
gh auth status          # 验证只看账号/scope，不打印 token
```

- 登录态由 gh 自持，之后建仓/PR/API 无需重复解密；PAT 本身仍只存于 `.enc`。
- 双机 GitHub 账号、SSH 分流别名等台账查 dual-machine-manager，不在此记值。

### 4.2 `git push` over SSH（ssh-agent / 钥匙串）

```bash
# 私钥 passphrase 由用户当次输入一次，加入 agent 与钥匙串，之后免重复输
ssh-add --apple-use-keychain ~/.ssh/id_rsa_softwawrecheng   # 文件名历史拼写，勿改
ssh-add -l                                                   # 只列指纹，无私钥
```

- agent/钥匙串持有解锁后的私钥，主口令/passphrase 不进对话、不进脚本；
- 远程机若 agent 未运行，按 dual-machine-manager `security-and-git.md` 的 SOP 启动。

### 4.3 `sudo` / 系统级命令

- **优先让用户在自己的终端亲自输 sudo 密码**（最干净，agent 不经手）。
- 确需自动化时再解密 `~/.doubao/secrets/sudo.enc`，管道给 `sudo -S`，不回显、不留命令历史长留；用完清理。

### 4.4 脚本调第三方平台（项目凭证）

```bash
# 项目 .secrets 下的 .enc；三级回退：ENC_PASS/专用变量 > master.pass > 交互
API_KEY="$(BAIDU_ENC_PASS="${BAIDU_ENC_PASS:-$ENC_PASS}" \
  secrets json ".secrets/platform.enc" api_key)"   # JSON 密文可取单字段
```

- 参考范式：mac-system-toolkit `secret-encryption.md` 的 Bash/Python 取密段；
- 脚本不内置主口令；专用变量（如 `BAIDU_ENC_PASS`）与通用 `ENC_PASS` 兼容。

### 4.5 二段因子（TOTP）

- 需要 2FA 时停下，请用户从 Microsoft Authenticator 读 6 位码；用后即弃、不记录。
- 仅当 TOTP secret 已加密存本机且用户授权，才解密自动生成（按开发者凭证对待）。

### 4.6 浏览器/登录态自动化

- Playwright/Puppeteer 的 storage-state、cookie 属会话凭证：存仓库外临时位置、用后删、不进公开仓；复用日常登录态优先走 CDP 连接已开浏览器（见 mac-system-toolkit Chrome 控制文档），而不是把账号密码塞进脚本。

## 五、非交互/无人值守

用户不在场、流水线又必须解密时，才启用本机主密码文件（是否启用由用户决定，细节见 [master-passphrase.md](master-passphrase.md)）：

- `~/.doubao/secrets/master.pass`：600、目录 700、仓库外、不入云同步、不跨机明文拷贝；
- 脚本三级回退：专用/通用 `ENC_PASS` > `master.pass` > 交互，全缺报错退出；
- 这样 agent 可自动从 `.enc` 取密，**仍不需要把主口令写进任何项目或对话**。

## 六、"拿不到主口令明文"交付前自检

- [ ] 本次改动/交付物（代码、文档、提交、截图、日志）里 grep 不到真实主口令、token、私钥、连接串；示例只用占位/掩码。
- [ ] 解密出的凭证没有 `echo`/打印到对话，没有写入会被 git 跟踪或云同步的文件。
- [ ] 用到的变量已 `unset`；临时凭证文件已删。
- [ ] 权威值只在 `.enc`（全局/项目）或系统钥匙串/ssh-agent；`.env` 仅是被 ignore 的传输层。
- [ ] 主口令来源合规（交互/`ENC_PASS`/授权的 `master.pass`），没有写死或猜测。
- [ ] 公开仓额外过 [repo-and-leak-defense.md](repo-and-leak-defense.md) 的交付清单；需要时跑 `audit-secrets.sh` 巡检。
- [ ] 没有要求用户改密码、没有给用户装其不需要的个人密码管理器。

## 七、来源

- Auth0：秘密只在执行层、永不进 LLM 上下文 https://auth0.com/blog/want-ai-agents-that-don-t-spill-secrets-don-t-give-them-secrets/
- WorkOS：AI agent 密钥管理、分段与最小权限 https://workos.com/blog/ai-agent-secrets-management
- AgentixForce：vault 引用架构、值不进上下文 https://www.agentixforce.ai/blog/secrets-management-in-agent-runtimes
- Uniclaw：环境变量是传输层不是存储层 https://uniclaw.ai/blog/ai-agent-secrets-management-api-keys
- Flavio Copes：agent 用秘密引用而不见值 https://flaviocopes.com/ai-agent-passwords/
- 恶意依赖 harvest `.env`/环境变量/SSH key 的案例 https://dev.to/webofmike/your-ai-agent-should-not-hold-the-llm-api-key-4j3c
- 加解密命令与三级回退权威实现：mac-system-toolkit `references/secret-encryption.md`
