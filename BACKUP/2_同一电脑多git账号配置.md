# [同一电脑多git账号配置](https://github.com/Quitino/gitblog/issues/2)



> 如果你的电脑全局配置的是公司 Git 账号，但这个项目需要用个人 GitHub 账号，
> 需要解决两个问题：**提交身份** 和 **推送认证**。

## 1 设置本项目的提交身份

为本项目单独设置用户名和邮箱（不加 `--global`，不影响公司配置）：

```bash
cd /path/to/project

# 仅对本项目生效
git config user.name "Quitino"
git config user.email "booofeng@163.com"
```

验证：

```bash
git config user.name           # → Quitino（本项目）
git config --global user.name  # → 公司账号（不受影响）
```

## 2 配置推送认证（SSH 多密钥方案，推荐）

公司电脑的 SSH key 绑定的是公司 GitHub 账号，推送到个人仓库会被拒绝。
用 SSH 多密钥解决：

```
原理:
┌──────────────────────────────────────────────────────┐
│  ~/.ssh/                                              │
│                                                       │
│  id_ed25519          ← 公司 SSH 密钥                  │
│  │                      绑定公司 GitHub 账号            │
│  │                      用于: git@github.com:公司/...  │
│  │                                                    │
│  id_ed25519_personal ← 个人 SSH 密钥 (新生成)          │
│                         绑定个人 GitHub 账号            │
│                         用于: git@github-personal:...  │
│                                ↑ 自定义别名             │
│                                                       │
│  config              ← SSH 配置文件                    │
│                         根据 Host 别名选择不同密钥       │
└──────────────────────────────────────────────────────┘
```

**Step 1: 生成个人专用 SSH 密钥**

```bash
ssh-keygen -t ed25519 -C "booofeng@163.com" -f ~/.ssh/id_ed25519_personal
# 提示输入密码时可以直接回车（不设密码）
```

**Step 2: 把公钥添加到个人 GitHub 账号**

```bash
cat ~/.ssh/id_ed25519_personal.pub
# 复制输出的内容
```

然后在浏览器中：

```
GitHub (个人账号登录)
→ Settings
→ SSH and GPG keys
→ New SSH Key
→ 粘贴公钥，Title 填 "公司电脑"
→ Add SSH Key
```

**Step 3: 配置 SSH 别名**

编辑（或创建） `~/.ssh/config` 文件，添加：

```
# 公司 GitHub（默认，不用改）
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes

# 个人 GitHub（自定义别名 github-personal）
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal
    IdentitiesOnly yes
```

**Step 4: 测试连接**

```bash
ssh -T github-personal
# 期望输出: Hi Quitino! You've been authenticated, but GitHub
#          does not provide shell access.
```

**Step 5: 修改本项目的 remote 地址**

```bash
cd /path/to/project

# 使用别名 github-personal 替代 github.com
git remote set-url origin git@github-personal:Quitino/project.git
```

验证：

```bash
git remote -v
# origin  git@github-personal:Quitino/project.git (fetch)
# origin  git@github-personal:Quitino/project.git (push)
```

现在 `git push origin master` 会自动使用个人 SSH 密钥认证。

## 3 备选：HTTPS + Personal Access Token

如果公司网络限制 SSH，可以用 HTTPS + Token 方式：

```bash
# 1. 在个人 GitHub 生成 Token
#    GitHub → Settings → Developer settings
#    → Personal access tokens → Tokens (classic)
#    → Generate new token → 勾选 repo 权限 → 生成

# 1. 修改 remote 地址（把 <TOKEN> 替换为实际 Token）
git remote set-url origin https://Quitino:<TOKEN>@github.com/Quitino/project.git
```

> **安全提醒：** Token 会明文存在 `.git/config` 中。如果公司有代码审查工具
> 扫描磁盘文件，SSH 方案更安全。

## 4 两种方案对比

| 方面 | SSH 多密钥（推荐） | HTTPS + Token |
|------|-------------------|---------------|
| 安全性 | 高（私钥不离开本机） | 中（Token 明文存在 config） |
| 配置量 | 一次性配置，之后无感 | 简单，Token 有过期风险 |
| 网络限制 | 需要 22 端口（SSH） | 走 443 端口（HTTPS） |
| 多项目复用 | 所有个人项目共用一个别名 | 每个项目单独设 Token URL |

## 5 完整配置验证清单

```
□ git config user.name 显示个人账号名
□ git config user.email 显示个人邮箱
□ ssh -T github-personal 显示 "Hi Quitino!"
□ git remote -v 显示 github-personal 别名地址
□ git push origin master 能成功推送
```
