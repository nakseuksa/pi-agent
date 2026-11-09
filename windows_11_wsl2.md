# Windows 11 中文版：Pi Agent 加固型 WSL2 独立发行版沙盒部署指南

本指南旨在帮助你在 **Windows 11 中文版** 环境下，为自主 AI 智能体（如 Pi Agent）构建一个高安全性的 **WSL2 独立发行版沙盒**。

### 核心安全目标
1. **切断全盘暴露**：禁止 WSL2 默认将宿主机 `C:`、`D:` 盘挂载到 `/mnt/`，保护个人文件、浏览器 Cookie 与 `~/.ssh` 密钥。
2. **禁止跨系统调用**：禁止 Agent 在 Linux 内调用 Windows 宿主机的 `.exe` 程序（如 `powershell.exe`、`cmd.exe`）及读取 Windows `PATH` 环境变量。
3. **单目录精准授权**：通过 `/etc/fstab` 仅向 Agent 开放指定的**可读写工作区（`/workspace`）**与**只读参考资料库（`/reference`）**。
4. **阻断内网横向渗透**：通过 Windows 防火墙允许 Agent 访问公网大模型 API，但严禁其扫描或连接家庭/办公局域网（如路由器后台、NAS）。

---

## 阶段一：开启 Windows 11 WSL2 环境与宿主机目录准备

首先在 Windows 宿主机中完成底层虚拟化环境初始化，并创建专门提供给 Pi Agent 访问的物理文件夹。

### 1. 打开管理员终端
在 Windows 11 桌面底部任务栏中，**右键点击“开始”菜单图标**，选择 **终端(管理员)** 或 **Windows PowerShell(管理员)**，在弹出的用户账户控制（UAC）提示中点击“是”。

### 2. 启用并更新 WSL2 核心组件
在管理员终端中依次执行以下命令（仅安装 WSL2 核心底座，暂不安装默认发行版）：

```powershell
wsl --install --no-distribution
wsl --update
wsl --set-default-version 2
```

> **提示**：如果此前从未开启过虚拟化，系统可能会提示重启电脑。重启后再次打开“终端(管理员)”继续即可。

* **验证方法**：在 PowerShell 中运行 `wsl --status`，确认输出信息中包含 `默认版本: 2`。

### 3. 在 Windows 中创建专属工作区与只读资料库目录
这里以包含**空格与中文**的路径为例（你可根据实际需求修改盘符与目录名）：
* **读写工作区**：`C:\Agent Workspace\项目工作区`
* **只读资料库**：`D:\My Knowledge\参考资料库`

在 PowerShell 中执行创建命令：

```powershell
New-Item -ItemType Directory -Force -Path "C:\Agent Workspace\项目工作区"
New-Item -ItemType Directory -Force -Path "D:\My Knowledge\参考资料库"
```

* **验证方法**：按 `Win + E` 打开“文件资源管理器”，确认上述两个文件夹均已创建成功。

---

## 阶段二：安装独立的 WSL2 沙盒发行版

为避免影响你日常使用的其他 WSL 开发环境，建议安装一个专门用于运行 Agent 的独立发行版实例。

### 1. 安装 Ubuntu-24.04 发行版
在 PowerShell 中执行以下命令：

```powershell
wsl --install -d Ubuntu-24.04
```

### 2. 创建 Linux 普通用户与密码
安装完成后会自动进入 Ubuntu 终端初始化界面，提示输入新用户名和密码：
* `Enter new UNIX username:` 输入小写用户名（例如 `piagent`）并回车。
* `New password:` 输入一个强密码并回车（Linux 终端输入密码不会显示星号，直接盲打后回车即可）。
* `Retype new password:` 再次输入密码并回车。

### 3. 确认当前用户的 UID 与 GID
进入 Linux 命令行后，执行以下命令查看用户编号（首个创建的用户默认均为 `1000`）：

```bash
id
```

* **验证方法**：终端输出显示 `uid=1000(piagent) gid=1000(piagent)`，记下这两个数字供后续挂载配置使用。

> **⚠️ 密码安全警告：切勿开启免密 sudo**
> 在沙盒环境中，**绝对不要**修改 `/etc/sudoers` 将该用户设为 `NOPASSWD`（免密执行 `sudo`）。保留 `sudo` 密码验证，能有效防止 Pi Agent 被恶意提示词注入后自行执行 `sudo mount` 重新挂载你的 C 盘，或使用 `remount,rw` 解除只读资料库的锁定。

---

## 阶段三：配置 `/etc/wsl.conf` 切断 C/D 盘挂载与 `.exe` 互操作

这一步将彻底关闭 WSL2 默认挂载 Windows 所有磁盘的行为，并禁止 Agent 在 Linux 内调用 Windows 宿主程序。

### 1. 编辑 WSL 核心配置文件
在 Ubuntu 终端中执行：

```bash
sudo nano /etc/wsl.conf
```
*（系统会提示输入你在阶段二设置的密码。）*

### 2. 写入安全加固配置
将以下内容完整粘贴到编辑器中：

```ini
[automount]
# 严禁自动把 Windows 的 C 盘、D 盘挂载到 /mnt/
enabled = false
# 允许开机自动读取 /etc/fstab 中的单目录精准挂载规则
mountFsTab = true

[interop]
# 严禁在 WSL2 中直接调用 Windows 的 .exe 应用程序
enabled = false
# 严禁把 Windows 宿主机的环境变量 PATH 注入到 Linux 中
appendWindowsPath = false

[network]
# 自动生成 DNS 解析配置，保障正常访问公网大模型 API
generateResolvConf = true
```

### 3. 保存并退出编辑器
在 `nano` 界面中：
1. 按 **`Ctrl + O`**
2. 按 **`回车键 (Enter)`** 确认文件名保存
3. 按 **`Ctrl + X`** 退出编辑器

* **验证方法**：在终端运行 `cat /etc/wsl.conf`，确认上述 3 个配置段内容完整无误。

---

## 阶段四：配置 `/etc/fstab` 实现双目录精准挂载（读写区 + 只读区）

接下来将 Windows 上的两个目录分别映射到 WSL2 内部的 `/workspace`（可读写）与 `/reference`（只读）。

### 1. 在 Linux 内创建挂载点目录
在 Ubuntu 终端中执行：

```bash
sudo mkdir -p /workspace /reference
sudo chown 1000:1000 /workspace /reference
```

* **验证方法**：执行 `ls -ld /workspace /reference`，确认两个目录均已存在且所有者为你的用户名。

### 2. 掌握 `/etc/fstab` 路径转义铁律
`/etc/fstab` 严格使用空格或制表符（Tab）分隔每一列，在编写 Windows 路径时必须遵守以下规则：

| 特殊字符类型 | 在 `/etc/fstab` 中的处理规则 | 错误写法示例 | 正确写法示例 |
| :--- | :--- | :--- | :--- |
| **路径分隔符 (`\`)** | 必须统一替换为正斜杠 **`/`** | `C:\Agent\项目` | `C:/Agent/项目` |
| **空格 (` `)** | 必须替换为八进制转义符 **`\040`** | `C:/Agent Workspace` | `C:/Agent\040Workspace` |
| **单/双引号 (`'` / `"`)** | **严禁使用任何引号**（会被当成路径字符报错） | `'C:/Agent Workspace'` | `C:/Agent\040Workspace` |
| **中文字符** | 直接输入（确保文件为 UTF-8 无 BOM 编码） | 无需转义 | `C:/Agent\040Workspace/项目工作区` |

### 3. 编辑 `/etc/fstab` 添加双挂载规则
打开配置文件：

```bash
sudo nano /etc/fstab
```

在文件末尾另起新行，追加以下两行配置（如果阶段二查到的 `uid` 和 `gid` 不是 `1000`，请替换为实际数值）：

```fstab
# 1. 读写工作区（允许 Agent 创建、修改、运行代码）
C:/Agent\040Workspace/项目工作区  /workspace  drvfs  rw,noatime,uid=1000,gid=1000,umask=022,fmode=0644,dmode=0755,nosuid,nodev  0  0

# 2. 只读资料库（仅允许 Agent 检索查阅，内核级禁止修改、删除或执行二进制程序）
D:/My\040Knowledge/参考资料库  /reference  drvfs  ro,noatime,uid=1000,gid=1000,umask=0222,fmode=0444,dmode=0555,noexec,nosuid,nodev  0  0
```

按 **`Ctrl + O`** $\rightarrow$ **`回车键`** 保存，按 **`Ctrl + X`** 退出。

### 4. 执行挂载干跑测试（重启前必做）
每次修改 `/etc/fstab` 后，务必先在当前终端测试挂载语法是否正确：

```bash
sudo mount -a
```

* **验证方法**：执行 `sudo mount -a` 后**没有任何报错输出**，且运行 `df -h | grep -E '/workspace|/reference'` 能同时看到两个目录的挂载信息。

---

## 阶段五：重启 WSL2 并执行沙盒隔离验收测试

配置完成后，需要彻底重启 WSL2 虚拟机以卸载原有的 `/mnt/c` 全盘挂载并应用新边界。

### 1. 彻底重启 WSL2
切换回 **Windows PowerShell** 窗口，执行关闭与重启命令：

```powershell
# 彻底关闭 WSL2 虚拟机
wsl --shutdown

# 重新进入 Ubuntu-24.04 沙盒实例
wsl -d Ubuntu-24.04
```

### 2. 执行 5 项安全边界验收指令
在 Ubuntu 终端中依次运行以下测试命令，核对隔离是否完全生效：

```bash
# 测试 1：验证 C 盘、D 盘是否已彻底隐藏
# 预期结果：输出为空（总计 0），看不到 c 或 d 目录
ls -la /mnt

# 测试 2：验证是否已禁止调用 Windows 宿主程序
# 预期结果：提示 powershell.exe: command not found
powershell.exe -Command "Get-Process"

# 测试 3：验证向上目录穿越是否会逃逸到 Windows C 盘
# 预期结果：输出为 /（停留在 Linux 根目录，无法访问 C:\Agent Workspace 的上一级）
cd /workspace/.. && pwd

# 测试 4：验证读写工作区（/workspace）是否双向互通
# 预期结果：正常输出 Hello Agent Sandbox，且在 Windows 对应文件夹中能看到 test_ok.txt
echo "Hello Agent Sandbox" > /workspace/test_ok.txt && cat /workspace/test_ok.txt

# 测试 5：验证只读资料库（/reference）是否防篡改、防删除
# 预期结果：直接拦截并报错 touch: cannot touch '/reference/hack.txt': Read-only file system
touch /reference/hack.txt
```

---

## 阶段六：配置 Windows 防火墙阻断局域网横向扫描（强烈推荐）

为了防止 Pi Agent 被恶意网页或文档提示词注入后，利用 SSRF 或扫描工具探测你的家庭/公司内网（如路由器管理后台 `192.168.1.1`、内网 NAS 等），可在 Windows 宿主机添加一条出站拦截规则：**仅拦截私有局域网 IP 段，放行公网访问**。

### 1. 添加出站拦截规则
打开 Windows **终端(管理员)**，在 PowerShell 中执行：

```powershell
New-NetFirewallRule `
  -DisplayName "WSL2-Sandbox-Block-Private-LAN" `
  -Direction Outbound `
  -InterfaceAlias "vEthernet (WSL*)" `
  -RemoteAddress 192.168.0.0/16,10.0.0.0/8,172.16.0.0/12 `
  -Action Block
```

* **验证方法**：
  1. 在 WSL2 终端中运行 `curl -I https://www.baidu.com`，依然能正常返回 `HTTP/1.1 200 OK`（访问公网大模型 API 不受任何影响）。
  2. 在 WSL2 终端中尝试 `ping 192.168.1.1`（你的局域网网关或内网其他设备 IP），请求会被直接阻断。

### 2. 日后如何移除该防火墙规则（如需恢复内网访问）
若日后需要让 WSL2 访问局域网设备，只需在 Windows **终端(管理员)** 中执行：

```powershell
Remove-NetFirewallRule -DisplayName "WSL2-Sandbox-Block-Private-LAN"
```

---

## 附录 A：`/etc/fstab` 结构与安全参数速查表

### 1. 6 列字段结构说明
```fstab
[第1列:源路径]  [第2列:挂载点]  [第3列:类型]  [第4列:安全与权限参数]  [第5列:备份]  [第6列:自检]
```
* **第 1 列**：Windows 宿主机文件夹路径（正斜杠，空格写为 `\040`）。
* **第 2 列**：映射到 Linux 内部的空目录路径（如 `/workspace`）。
* **第 3 列**：WSL2 挂载 Windows 目录固定填 `drvfs`。
* **第 4 列**：挂载参数，多个参数之间用**英文逗号**紧密连接，中间不能有空格。
* **第 5、6 列**：备份标记与开机磁盘自检顺序，跨系统目录固定填 `0  0`。

### 2. 读写区与只读区核心参数对照

| 参数项 | `/workspace` (读写区) | `/reference` (只读区) | 安全防御作用解析 |
| :--- | :--- | :--- | :--- |
| **挂载模式** | `rw` | **`ro`** | `ro` 为内核级只读锁，任何写入、追加、删除或重命名操作直接拒绝 |
| **文件权限 (`fmode`)** | `0644` (`rw-r--r--`) | **`0444`** (`r--r--r--`) | `0444` 剥夺所有文件的写权限位（`w`），所有用户仅可读取 |
| **目录权限 (`dmode`)** | `0755` (`rwxr-xr-x`) | **`0555`** (`r-xr-xr-x`) | `0555` 保留进入目录（`x`）和查看列表（`r`）权限，剥夺增删文件（`w`）权限 |
| **权限掩码 (`umask`)** | `022` | **`0222`** | 屏蔽对应的权限写入位 |
| **禁止执行 (`noexec`)** | 不加 | **`noexec`** | 禁止直接运行资料库中的二进制可执行程序，防止恶意载荷执行 |
| **防提权 (`nosuid`)** | `nosuid` | `nosuid` | 禁止目录下的程序利用 SUID 特殊权限位临时获取 `root` 身份 |
| **防设备文件 (`nodev`)** | `nodev` | `nodev` | 禁止将该目录内的特殊文件解析为字符设备或块设备 |

---

## 附录 B：不重启 WSL2 热重载 `/etc/fstab` 挂载指南

当你在日常使用中修改了 `/etc/fstab` 的权限参数，或需要临时解锁 `/reference` 写入权限时，无需重启 WSL2，可按以下方式热更新。

### 场景 1：修改了 `fmode`、`dmode`、`umask`、`uid/gid` 或更换了目录路径
WSL2 的 `drvfs` 驱动仅在挂载初始化时解析具体权限掩码，因此**必须先卸载再重新挂载**：

```bash
# 1. 先切回主目录，避免当前终端占用挂载点
cd ~

# 2. 卸载目标目录（以 /reference 为例）
sudo umount /reference

# 3. 重新读取 /etc/fstab 并应用新参数挂载
sudo mount -a

# 4. 验证新权限是否生效
mount | grep /reference
ls -la /reference
```

### 场景 2：仅临时切换 `/reference` 的只读（`ro`）与读写（`rw`）状态
如果未修改权限掩码，仅需临时在 Linux 内向 `/reference` 写入文件，可使用 `remount` 原地切换：

```bash
# 临时解锁为可读写
sudo mount -o remount,rw /reference

# 【操作完毕后务必立刻执行】重新锁回只读状态
sudo mount -o remount,ro /reference
```

### 场景 3：卸载时报错 `umount: /reference: target is busy` 排障
如果执行 `sudo umount /reference` 提示设备正忙，说明后台有 Agent 进程或终端正打开了该目录下的文件：

```bash
# 1. 查看具体是哪个进程（PID）占用了该目录
sudo fuser -vm /reference

# 2. 强制终止占用该目录的所有进程，随后即可正常卸载
sudo fuser -km /reference
sudo umount /reference && sudo mount -a

# 备选方案：如果不希望强杀进程，可执行延迟卸载（Lazy Unmount）并重挂
sudo umount -l /reference && sudo mount -a
```