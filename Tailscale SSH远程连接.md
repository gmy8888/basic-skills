# Win11 通过 Tailscale + SSH 远程连接总结

> 占位符说明：  
> - `xxx`：用户名  
> - `xxxxxx`：目标机器的 Tailscale IP  
> - 实际使用时替换成你自己的用户名和 IP。  
> - Windows 本地路径中的 `C:/Users/xxx/.ssh/...` 也请把 `xxx` 替换成你的 Windows 用户名。

---

# 一、Win11 连接 Ubuntu

## 1. Ubuntu 端配置

### 1.1 安装并登录 Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

浏览器打开提示的 URL，登录 Tailscale 账户。  
查看 Ubuntu 的 Tailscale IP：

```bash
tailscale ip -4
```

假设输出为：

```text
xxxxxx
```

### 1.2 安装并启动 OpenSSH Server

```bash
sudo apt update
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
```

如果启用了 UFW，放行 SSH：

```bash
sudo ufw allow ssh
```

### 1.3 确认或创建登录用户

```bash
sudo adduser xxx
```

这里 `xxx` 是 Ubuntu 上的登录用户名。

### 1.4 放入 Win11 的公钥

先在 Win11 上生成密钥：

```powershell
ssh-keygen -t ed25519 -C "win11-to-ubuntu"
```

默认生成：

```text
C:\Users\xxx\.ssh\id_ed25519
C:\Users\xxx\.ssh\id_ed25519.pub
```

然后用 Git Bash 上传公钥：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub xxx@xxxxxx
```

输入 Ubuntu 用户密码即可。  
如果 Ubuntu 已禁用密码登录，则需要手动在 Ubuntu 上追加公钥：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "粘贴公钥内容" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chown -R xxx:xxx ~/.ssh
```

### 1.5 可选：禁用 Ubuntu 密码登录

确认密钥登录成功后，再操作：

```bash
sudo nano /etc/ssh/sshd_config
```

把：

```text
PasswordAuthentication yes
```

改成：

```text
PasswordAuthentication no
```

重启 SSH：

```bash
sudo systemctl restart ssh
```

---

## 2. Win11 端配置

### 2.1 安装 Tailscale 并登录

下载 Windows 版 Tailscale：

```text
https://tailscale.com/download/windows
```

安装后，用与 Ubuntu 相同的 Tailscale 账户登录。

验证连通性：

```powershell
tailscale status
tailscale ping xxxxxx
```

### 2.2 确认 OpenSSH 客户端

```powershell
ssh -V
```

如果没有，进入：

```text
设置 -> 应用 -> 可选功能 -> 添加功能 -> OpenSSH 客户端
```

安装。

### 2.3 生成 SSH 密钥

```powershell
ssh-keygen -t ed25519 -C "win11-to-ubuntu"
```

一路回车。

### 2.4 测试登录

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 xxx@xxxxxx
```

第一次问 `yes/no`，输入 `yes`。  
如果不再提示密码，直接进入 Ubuntu，说明成功。

### 2.5 配置 SSH config

编辑：

```powershell
notepad $env:USERPROFILE\.ssh\config
```

写入：

```text
Host ubuntu
    HostName xxxxxx
    User xxx
    Port 22
    IdentityFile C:/Users/xxx/.ssh/id_ed25519
    IdentitiesOnly yes
```

以后直接：

```powershell
ssh ubuntu
```

### 2.6 VS Code 连接 Ubuntu

1. 安装 VS Code 扩展 **Remote - SSH**。
2. `F1` -> `Remote-SSH: Connect to Host`。
3. 选择 `ubuntu`。
4. 第一次选择平台：Linux。

---

# 二、Win11 连接另一台 Win11

假设：

- 连接端：你的 Win11
- 目标端：另一台 Win11
- 目标端用户名：`xxx`
- 目标端 Tailscale IP：`xxxxxx`

## 1. 目标 Win11 配置

### 1.1 安装并登录 Tailscale

安装 Tailscale Windows 版，用同一账户登录。  
查看目标端 Tailscale IP：

```powershell
tailscale ip -4
```

假设为 `xxxxxx`。

### 1.2 确认登录用户名

在目标端执行：

```powershell
whoami
```

输出类似：

```text
计算机名\xxx
```

`\` 后面的 `xxx` 就是 SSH 登录用户名。

### 1.3 安装 OpenSSH Server

**必须以管理员身份打开 PowerShell**，然后执行：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
Get-Service sshd
```

看到 `Running` 即可。

### 1.4 放行防火墙

```powershell
Get-NetFirewallRule -Name *ssh*
```

如果没有规则，执行：

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

### 1.5 判断目标用户是否为管理员

在目标端管理员 PowerShell 执行：

```powershell
whoami /groups | findstr /i "S-1-5-32-544"
```

- 有输出：`xxx` 是管理员  
  公钥放到：

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

- 无输出：`xxx` 是普通用户  
  公钥放到：

```text
C:\Users\xxx\.ssh\authorized_keys
```

### 1.6 放入连接端 Win11 的公钥

先在连接端 Win11 生成密钥：

```powershell
ssh-keygen -t ed25519 -C "win11-to-win11"
```

查看公钥：

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

复制输出的整行公钥。

#### 如果目标是管理员账户

在目标端管理员 PowerShell 中执行：

```powershell
Add-Content -Path C:\ProgramData\ssh\administrators_authorized_keys -Value "粘贴公钥" -Encoding ascii

icacls C:\ProgramData\ssh\administrators_authorized_keys /inheritance:r
icacls C:\ProgramData\ssh\administrators_authorized_keys /grant "Administrators:F" "SYSTEM:F"

Restart-Service sshd
```

#### 如果目标是普通用户

在目标端执行：

```powershell
mkdir $env:USERPROFILE\.ssh -Force
Add-Content -Path $env:USERPROFILE\.ssh\authorized_keys -Value "粘贴公钥" -Encoding ascii
Restart-Service sshd
```

### 1.7 确认 sshd 配置

```powershell
Get-Content C:\ProgramData\ssh\sshd_config | Select-String "Match Group|AuthorizedKeysFile"
```

默认应包含：

```text
Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

---

## 2. 连接端 Win11 配置

### 2.1 安装 Tailscale 并验证

```powershell
tailscale status
tailscale ping xxxxxx
```

### 2.2 确认 OpenSSH 客户端

```powershell
ssh -V
```

### 2.3 测试登录

```powershell
ssh -i $env:USERPROFILE\.ssh\id_ed25519 xxx@xxxxxx
```

如果不再提示密码，直接进入目标 Win11，说明成功。

### 2.4 配置 SSH config

编辑：

```powershell
notepad $env:USERPROFILE\.ssh\config
```

写入：

```text
Host win-desktop
    HostName xxxxxx
    User xxx
    Port 22
    IdentityFile C:/Users/xxx/.ssh/id_ed25519
    IdentitiesOnly yes
```

以后直接：

```powershell
ssh win-desktop
```

### 2.5 VS Code 连接另一台 Win11

1. 安装 VS Code 扩展 **Remote - SSH**。
2. `F1` -> `Remote-SSH: Connect to Host`。
3. 选择 `win-desktop`。
4. 第一次选择平台：Windows。

---

# 三、密钥与反向连接说明

## 1. 每台电脑都要单独配置密钥吗？

不强制，但推荐每台电脑单独生成密钥。

- 私钥：留在客户端，不要泄露。
- 公钥：放到目标机器的 `authorized_keys`。
- 一台客户端可以用同一个私钥连接多台目标机器。
- 多台客户端也可以把各自公钥放到同一台目标机器。

## 2. 之前配置过的密钥能直接复制到另一台电脑吗？

可以。把私钥复制过去即可：

```text
C:\Users\xxx\.ssh\id_ed25519
```

但安全性会降低。推荐新电脑单独生成密钥，然后把新公钥追加到目标机器。

## 3. 配置后能反向连接吗？

不能自动反向。SSH 密钥是单向的：

- A 的公钥放到 B，A 可以登录 B。
- 要让 B 登录 A，需要：
  - A 上运行 sshd；
  - B 有私钥；
  - B 的公钥放到 A 的 `authorized_keys`。

---

# 四、常见问题排查

| 问题 | 排查 |
|------|------|
| `tailscale ping` 不通 | 两台设备是否登录同一 Tailscale 账户；是否在线 |
| SSH 连接超时 | 目标机 sshd 是否运行；防火墙是否放行 22 |
| `Permission denied` | 用户名错；公钥位置错；权限错；用了 PIN 而不是账户密码 |
| Windows 管理员公钥不生效 | 公钥必须放 `C:\ProgramData\ssh\administrators_authorized_keys` |
| 权限报错 | 管理员公钥文件执行 `icacls` 设置权限 |
| 修改配置后连不上 | 保留一个已登录终端；改完再重启 sshd |
| VS Code 仍提示密码 | 先用 `ssh xxx@xxxxxx` 测试免密是否成功 |

---

# 五、最短命令版

## Win11 连 Ubuntu

Ubuntu：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo apt install openssh-server -y
sudo systemctl enable --now ssh
```

Win11：

```powershell
ssh-keygen -t ed25519 -C "win11-to-ubuntu"
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

Git Bash：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub xxx@xxxxxx
ssh xxx@xxxxxx
```

## Win11 连另一台 Win11

目标 Win11 管理员 PowerShell：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

连接端 Win11：

```powershell
ssh-keygen -t ed25519 -C "win11-to-win11"
type $env:USERPROFILE\.ssh\id_ed25519.pub
ssh -i $env:USERPROFILE\.ssh\id_ed25519 xxx@xxxxxx
```

公钥写入目标机：

- 管理员：`C:\ProgramData\ssh\administrators_authorized_keys`
- 普通用户：`C:\Users\xxx\.ssh\authorized_keys`

然后：

```powershell
Restart-Service sshd
```
