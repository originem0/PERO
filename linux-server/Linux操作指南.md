# Linux 服务器操作指南

> 从 Windows 用户的视角理解 Linux，面向"在服务器上能独立干活"的目标。

---

## 核心哲学

Linux 有两条贯穿一切的主线：

**1. 一切皆文件**
普通文档是文件，目录是文件，硬件设备是文件，配置是文件。统一用读写方式操作，不需要为每种东西学一套不同的接口。

**2. 最小权限原则**
能不暴露的就不暴露，能不给的权限就不给。从用户权限、进程隔离、网络配置到安全加固，处处体现。

---

## 目录

1. [Shell 与命令行基础](#1-shell-与命令行基础)
2. [文件系统与目录结构](#2-文件系统与目录结构)
3. [用户与权限](#3-用户与权限)
4. [进程管理](#4-进程管理)
5. [包管理与软件安装](#5-包管理与软件安装)
6. [网络基础](#6-网络基础)
7. [服务管理（systemd）](#7-服务管理systemd)
8. [基本安全实践](#8-基本安全实践)
9. [常用命令速查表](#9-常用命令速查表)

---

## 1. Shell 与命令行基础

### 核心原理

计算机分三层：硬件 → 操作系统内核（Kernel）→ 程序。

- **Kernel（内核）**：直接管理硬件（CPU、内存、磁盘），用户不能直接操作它
- **Shell（外壳）**：包在 Kernel 外面的交互层，翻译人类的命令给系统执行

Shell 本身也是一个程序，权限取决于当前登录的用户。Linux 服务器上通常没有图形界面，默认的命令行 Shell 是 **Bash**。

> **Windows 对比**：cmd / PowerShell 就是 Windows 的命令行 Shell。

### Shell 的工作流

```
你输入命令 → Shell 接收文本 → 解析命令 → 通过 PATH 找到程序 → 启动程序 → 程序与内核交互 → 返回结果
```

Shell 不是直接把命令传给内核。Shell 是调度员——它找到对应的程序并启动它，程序自己去跟内核打交道。

### 命令的结构

```
命令名 选项 参数
```

- **命令名**：要运行的程序（如 `ls`）
- **选项（option/flag）**：告诉程序"怎么做"（如 `-l` 详细列表，`-a` 显示隐藏文件）
- **参数（argument）**：告诉程序"对谁做"（如 `/home`）

```bash
ls -l /home
#  ↑  ↑   ↑
# 命令 选项 参数
```

Shell 按空格拆分后全部传给程序，**程序自己负责解析**。选项用 `-` 开头是社区惯例，不是系统强制。

**选项的两种写法（短选项打字快，长选项可读性好）：**
- 短选项：`-l`（一个横杠 + 一个字母），可以合并：`-la`
- 长选项：`--all`（两个横杠 + 完整单词）
- `ls -a` 和 `ls --all` 是同一个意思

> **Windows 对比**：Windows 的选项用 `/`（如 `dir /w`），Linux 用 `-`。容易搞混。

### PATH 环境变量

Shell 通过 **PATH 环境变量** 找程序。PATH 记录了一串目录路径，Shell 按顺序去这些目录里找程序名。

```bash
echo $PATH
# 输出类似：/usr/local/bin:/usr/bin:/bin
```

**新装的程序找不到？** 通常是它的路径不在 PATH 里。解决方法：
- 把程序放到 PATH 已有的目录（如 `/usr/local/bin`）
- 或者把程序所在目录加到 PATH 里

### 内置命令 vs 外部程序

| 类型 | 例子 | 特点 |
|------|------|------|
| 外部程序 | `ls`、`grep`、`curl` | 独立的程序文件（在 `/usr/bin` 等目录下），Shell 启动新进程执行 |
| 内置命令 | `cd`、`export` | Shell 自己内部执行，不启动新进程 |

**`cd` 为什么必须内置？** 每个进程有独立的工作目录（进程隔离）。如果 `cd` 是独立程序，它只能改自己的工作目录，执行完退出后 Shell 的目录不变。所以需要改变 Shell 自身状态的命令必须内置。

### 管道与重定向

**管道 `|`**：把一个程序的输出传给下一个程序处理。

```bash
ls -l | grep log           # ls 的输出传给 grep 过滤
ls -l | grep log | sort     # 可以继续串联
```

**重定向**：控制输出流向。

```bash
ls -l > result.txt          # 输出写入文件（覆盖）
ls -l >> result.txt         # 输出追加到文件
command < input.txt         # 从文件读入
command 2> error.log        # 错误信息重定向到文件
```

**Linux 哲学**：小工具各做一件事，通过管道自由组合，而不是一个大程序包办一切。

> **Windows 对比**：Windows 倾向一个程序功能齐全（Word 什么都能干），Linux 倾向小程序组合。

---

## 2. 文件系统与目录结构

### 核心原理

Linux 只有一个起点：**`/`（根目录）**。所有东西从这里往下长，像一棵倒着的树。

> **Windows 对比**：Windows 有多个入口（C:、D:、E:），每个盘符一棵树。Linux 只有一棵树。

### 挂载——路径与设备解耦

Linux 没有盘符。第二块硬盘、U 盘等存储设备通过**挂载（mount）**接入目录树的某个位置。

```
/            ← 根目录（第一块硬盘）
├── home/    ← 用户目录
├── data/    ← 第二块硬盘挂载在这里
└── ...
```

**核心好处：程序只认路径，不认设备。** 硬盘换了、扩容了，只要挂载到同一个路径，上层程序完全无感。

> **Windows 对比**：`D:\data` 这个路径本身包含了"在 D 盘上"的物理信息。换盘所有引用都要改。

### "一切皆文件"的三层含义

1. **普通文件和目录是文件** — 没争议
2. **硬件设备也是文件** — 硬盘是 `/dev/sda`，键盘、网卡都有对应的文件
3. **配置也是文件** — 放在 `/etc` 下的纯文本文件，不需要注册表

**硬件怎么变成文件？** Kernel 的**驱动程序**负责翻译：你对 `/dev/sda` 执行"读"→ 驱动翻译成"从磁盘某个扇区读数据"。你看到的是文件，背后是驱动在做翻译。

**注意区分两个概念：**
- **设备文件**（如 `/dev/sda`）：硬件的文件化抽象，让你能用读写操作控制硬件
- **挂载**：把存储设备的内容接入目录树某个位置。不是所有设备文件都涉及挂载（如键盘）

### 目录按类型组织

Linux 不像 Windows 按软件分（Chrome 的程序、配置、日志都在 Chrome 文件夹下）。Linux 按类型统一归类：

| 目录 | 存放内容 | 例子 |
|------|----------|------|
| `/usr/bin` | 可执行程序 | `ls`、`grep`、`nginx` |
| `/etc` | 配置文件 | `nginx.conf`、`ssh/sshd_config` |
| `/var/log` | 日志文件 | `auth.log`、`nginx/access.log` |
| `/dev` | 设备文件 | `sda`（硬盘）、`null` |
| `/home` | 普通用户的家目录 | `/home/tom`、`/home/jerry` |
| `/root` | root 用户的家目录 | |
| `/tmp` | 临时文件 | 重启可能清空 |
| `/usr/local/bin` | 用户自己安装的程序 | |

**好处**：形成规范。你去任何一台 Linux 服务器，配置一定在 `/etc`，日志一定在 `/var/log`，不需要额外沟通。

> **Windows 对比**：Windows 按软件分，卸载方便。Linux 按类型分，系统管理方便。设计立场不同。

### 路径

**绝对路径**：从 `/` 开始，完整路径。不管你当前在哪，指向固定位置。

```
/home/samsara/documents
```

**相对路径**：从当前目录开始。

```
cd documents         # 进入当前目录下的 documents
cd ..                # 回到上一级（.. = 上一级目录）
cd ../..             # 回到上两级
cd .                 # 当前目录（. = 当前目录）
```

> **Windows 对比**：逻辑一样，只是 Windows 用反斜杠 `\`，Linux 用正斜杠 `/`。

---

## 3. 用户与权限

### 核心原理

Linux 权限回答一个问题：**谁能对哪个文件做什么事？**

### 为什么需要多用户

不只是人需要不同账户，**程序也需要**。一台服务器上跑着网站、数据库、日志收集程序：

- 如果都用 root 运行：网站被黑 → 攻击者拿到 root 权限 → 能读数据库、删日志、装后门、控制整台机器
- 如果各用专门用户运行：网站被黑 → 攻击者只拿到 `www` 权限 → 碰不到数据库和日志

这就是**最小权限原则**：给每个用户和服务刚好够用的权限，不多给。

### 用户分类

- **root**：超级管理员，不受权限限制，可以做任何事。家目录在 `/root`
- **普通用户**：只能做被允许的事。家目录在 `/home/用户名`

### 权限模型

每个文件身上带着这些信息：

```
-rwxr-xr-- 1 www webteam 1024 Feb 23 config.txt
↑↑↑↑↑↑↑↑↑↑   ↑    ↑
│├──┤├──┤├──┤ owner group
│ ↑   ↑   ↑
│owner group others
│
文件类型（- 普通文件，d 目录）
```

**三类人 × 三种操作：**

| 操作 | 字母 | 含义 |
|------|------|------|
| 读 | r | 能不能看内容 |
| 写 | w | 能不能改内容 |
| 执行 | x | 能不能当程序运行 |

| 角色 | 含义 |
|------|------|
| owner | 文件所有者（谁创建的文件谁就是 owner） |
| group | 文件所属的用户组 |
| others | 除了 owner 和 group 之外的所有人 |

**匹配逻辑**：系统拿你的身份去匹配——你是 owner？用第一组权限。不是 owner 但在 group 里？用第二组。都不是？用第三组（others）。

**关键洞察：权限是绑在文件上的，不是绑在用户上的。** 同一个用户 tom，对文件 A 可能是 owner，对文件 B 可能是 others。

### 用户组的作用

用户组让你批量管理权限。把 tom、jerry 都加入 `webteam` 组，文件标记所属组为 `webteam`，他们就共享第二组权限。不需要一个个设。

注意：一个文件只能标记一个 group。

### `ls -l` 输出解读

```
drwxr-xr-x 4 root root  4096 Feb 18 10:12 ssh
│            │  ↑    ↑     ↑       ↑         ↑
│            │ owner group 大小    时间      文件名
│            硬链接数
文件类型+权限
```

### chmod——修改权限

```bash
chmod u+x file      # 给 owner 加执行权限
chmod g-w file      # 去掉 group 的写权限
chmod o+r file      # 给 others 加读权限
chmod u+rwx file    # 给 owner 加全部权限
```

- `u` = owner，`g` = group，`o` = others
- `+` 加权限，`-` 去权限
- 只有文件的 owner 和 root 才能 chmod

> **Windows 对比**：Windows 也有文件权限，但大部分用户从不接触。Linux 上权限是日常操作。

---

## 4. 进程管理

### 核心原理

- **程序**：硬盘上的文件，静态的，死的（菜谱）
- **进程**：程序被加载到内存运行起来的实例，活的（正在炒的菜）

同一个程序可以启动多个进程（开三个 Chrome 窗口 = 硬盘一个 chrome.exe，内存多个 Chrome 进程）。

### 进程隔离

**每个进程有独立的内存空间、工作目录、环境变量。Kernel 强制隔离，进程之间不能互相访问。**

隔离的原因：
- **稳定性**：一个进程崩了不会把别的进程的内存搞乱
- **安全性**：进程之间看不到彼此的数据（网站进程看不到数据库进程的内存）

进程之间需要通信时，必须通过 Kernel 提供的专门机制，不能直接翻别人的内存。

### 进程 vs 线程

- **进程**：独立的内存空间，互相隔离。类比独立的办公室。
- **线程**：一个进程内部的多个执行单元，共享内存。类比办公室里的员工共用桌子。

在服务器上管理的主要对象是进程，不是线程。

### Shell 与进程的关系

```
Shell（长期运行的进程，一直等你输入）
  ├── 你输入 ls → Shell 启动新进程运行 ls → ls 执行完退出
  ├── 你输入 grep → Shell 启动新进程运行 grep → grep 执行完退出
  └── 你输入 cd → Shell 自己内部执行（不启动新进程）
```

### PID（进程号）

每个进程有唯一的 PID，按启动顺序递增。**PID 1 永远是 `systemd`**——系统启动后的第一个进程，所有其他进程的祖先。

### 常用命令

```bash
ps aux              # 查看所有进程快照（拍一张照片）
top                 # 实时动态显示（像 Windows 任务管理器）
kill 1234           # 杀掉 PID 为 1234 的进程
kill -9 1234        # 强制杀掉（进程不响应普通 kill 时用）
```

> **Windows 对比**：`ps` / `top` = 任务管理器，`kill` = 右键"结束任务"。

---

## 5. 包管理与软件安装

### 核心原理

包管理器 = **中央仓库 + 自动下载 + 自动处理依赖 + 自动放到正确位置**。

Ubuntu/Debian 用 **apt**，CentOS/RedHat 用 **yum**。

```bash
apt install nginx
```

这一条命令做了什么：去远程仓库找 nginx → 下载 → 自动下载所有依赖 → 程序放到 `/usr/bin`、配置放到 `/etc`、日志配置到 `/var/log` → 创建 systemd 服务文件。

> **Windows 对比**：类似 npm。`npm install xxx` 自动下载包和依赖，`apt install xxx` 自动下载软件和依赖。

### 三种安装方式

| 方式 | 自动化程度 | 适用场景 |
|------|-----------|---------|
| `apt install xxx` | 全自动（下载+依赖+放置） | **首选**，仓库里有的软件都用这个 |
| `.deb` 安装包 | 半自动（不自动下载依赖） | 仓库里没有，但有 .deb 包时 |
| 压缩包解压 | 全手动 | 只有源码或二进制包时 |

### 常用命令

```bash
apt update             # 更新仓库索引（不是更新软件，是更新"有什么软件可装"的列表）
apt upgrade            # 更新所有已安装的软件到最新版
apt install xxx        # 安装软件
apt remove xxx         # 卸载软件
apt search xxx         # 搜索软件
```

### 仓库管理

仓库地址记录在系统配置里（`/etc/apt/sources.list`）。可以添加第三方仓库来安装官方仓库里没有的软件。

### Linux 没有注册表

Windows 用注册表（集中的配置数据库，需要 regedit 操作）。Linux 的配置就是 `/etc` 下的**纯文本文件**，用任何文本编辑器都能改。这是"一切皆文件"的又一个体现。

---

## 6. 网络基础

### 核心原理

- **IP 地址**：找到哪台机器（大楼地址）
- **端口**：找到这台机器上的哪个程序（门牌号）

任何需要接收网络请求的程序都要监听某个 IP + 端口。一个端口同一时间只能被一个程序占用。

### 常见默认端口

| 端口 | 服务 |
|------|------|
| 22 | SSH（远程登录） |
| 80 | HTTP（网站） |
| 443 | HTTPS（加密网站） |
| 3306 | MySQL |

浏览器访问 `http://xxx` 时自动访问 80 端口，不需要写 `:80`。

### 网络接口与监听地址

一台机器可以有多个网络接口，每个有自己的 IP：

```
lo（回环）:     127.0.0.1         ← 本机跟自己通信，断网也能用
eth0（网卡）:   36.151.146.104    ← 公网 IP
```

程序监听时选择：

| 监听地址 | 含义 | 谁能访问 |
|---------|------|---------|
| `127.0.0.1:8000` | 只监听本机回环 | 只有服务器本机 |
| `36.151.146.104:80` | 只监听公网网卡 | 外部通过公网 IP |
| `0.0.0.0:80` | 监听所有接口（通配符） | 本机 + 外部都能访问 |

`0.0.0.0` 不是真实 IP，是"我全都要"的声明。

### 反向代理架构

典型的服务器架构：

```
外部用户 → nginx（0.0.0.0:80）→ gunicorn（127.0.0.1:8000）→ 你的代码
```

为什么不让应用服务器直接对外？
- **安全**：nginx 过滤恶意请求，应用服务器不暴露在公网
- **统一接待**：nginx 处理 HTTPS、负载均衡、静态文件
- **最小暴露**：gunicorn 只监听 `127.0.0.1`，外面碰不到

### 网络排查链（从内到外）

```
程序在跑吗？           → ps aux | grep nginx
端口在监听吗？         → ss -tlnp
本机能访问自己吗？     → curl localhost:80
网络通吗？             → ping 对方IP
防火墙挡了吗？         → 检查防火墙规则
```

**排查原则：从里往外查。** 先确认程序没问题，再看网络。大部分时候问题在前两步。

**各步失败分别说明什么：**
- `curl localhost` 失败 → 程序本身有问题（没跑起来、端口没监听、报错了）
- `curl localhost` 成功但外面访问不了，`ping` 不通 → 网络层面问题
- 都通但外面还是访问不了 → 防火墙挡了，或者程序只监听了 `127.0.0.1`

### `ss -tlnp` 输出解读

```
State   Local Address:Port   Process
LISTEN  0.0.0.0:80            nginx     ← 对外服务，所有接口
LISTEN  127.0.0.1:8000        gunicorn  ← 只限本机
LISTEN  0.0.0.0:22            sshd      ← SSH 远程登录
```

### curl——命令行浏览器

```bash
curl http://36.151.146.104       # 相当于浏览器打开这个网址
curl -I http://36.151.146.104    # 只看响应头（状态码、服务器信息）
curl -o file.html http://xxx     # 下载保存到文件
```

可以测试网页、API 接口、下载文件。

---

## 7. 服务管理（systemd）

### 核心原理

- **服务（service）**= 在后台持续运行的进程。区别于 `ls` 这种跑完就退出的短命进程。
- **systemd** = Linux 的服务管家。PID 1，系统启动后第一个跑起来的进程，负责按配置启动、停止、监控所有服务。

> **Windows 对比**：`Win+R` → `services.msc` 打开的服务管理器，但 Linux 通过命令行操作。

### systemctl 命令

```bash
systemctl start nginx      # 现在立刻启动
systemctl stop nginx       # 现在停止
systemctl restart nginx    # 重启（先停再启）
systemctl status nginx     # 查看状态（在不在跑、跑了多久、PID 是什么）
systemctl enable nginx     # 注册开机自启动（不会现在启动）
systemctl disable nginx    # 取消开机自启动
```

**`start` vs `enable` 的区别：**
- `start`：现在跑起来，但重启服务器后不自动启动
- `enable`：注册开机自启动，但不会现在立刻启动

新装的服务通常两个都要执行：
```bash
systemctl start nginx && systemctl enable nginx
```

### `systemctl status` 输出解读

```
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; ...)
                                                        ↑ 开机自启动已开启
     Active: active (running) since Mon 2026-02-16 12:03:39 CST; 1 week ago
                     ↑ 正在运行                                   ↑ 已经跑了一周
   Main PID: 53586 (nginx)
      Tasks: 5
     CGroup: /system.slice/nginx.service
             ├─53586 "nginx: master process"    ← 主进程，负责管理
             ├─53587 "nginx: worker process"    ← 工作进程，处理请求
             ├─53588 "nginx: worker process"
             ├─53589 "nginx: worker process"
             └─53590 "nginx: worker process"
```

### 服务配置文件

程序要被 systemd 管理，必须在 `/lib/systemd/system/` 下有 `.service` 配置文件。

- 用 `apt` 安装的服务类软件（nginx、mysql、sshd）会**自动创建** `.service` 文件
- 自己写的程序想当服务跑，需要**自己写** `.service` 文件注册

### 查看服务日志

```bash
journalctl -u nginx              # 查看 nginx 的日志
journalctl -u nginx --since today # 只看今天的
journalctl -u nginx -f            # 实时跟踪日志（类似 tail -f）
```

---

## 8. 基本安全实践

### 核心原则

安全的核心就是最小权限：**能不暴露的就不暴露。**

### 默认状态有多危险

你的服务器默认：SSH 监听 `0.0.0.0:22`，用 root + 密码登录。

攻击者需要猜的信息：
- IP — 扫描 IP 段就能找到
- 端口 — 22，人人都知道
- 用户名 — root，所有 Linux 都有
- **密码 — 唯一需要猜的**

自动化脚本 24 小时不停试，一秒几百个密码。你的服务器现在大概率已经被人扫了。

验证：
```bash
grep "Failed password" /var/log/auth.log | tail -20
```

### SSH 加固三件套

**1. 密钥登录，禁用密码**

密钥原理：你的电脑生成一对钥匙（公钥 + 私钥）。公钥放服务器，私钥留本地。登录时做数学验证，私钥不在网络上传输，不存在被暴力猜解的可能。

```bash
# 本地电脑生成密钥对
ssh-keygen -t ed25519

# 把公钥传到服务器
ssh-copy-id user@your-server-ip

# 确认密钥登录成功后，禁用密码登录
# 编辑 /etc/ssh/sshd_config：
# PasswordAuthentication no
# 重启 SSH 服务
systemctl restart sshd
```

**2. 禁止 root 直接登录**

创建普通用户，需要管理员权限时用 `sudo` 临时提权：

```bash
# 创建用户并加入 sudo 组
adduser tom
usermod -aG sudo tom

# 编辑 /etc/ssh/sshd_config：
# PermitRootLogin no
systemctl restart sshd
```

**3. 改 SSH 默认端口**

```bash
# 编辑 /etc/ssh/sshd_config：
# Port 59832（换一个不常见的端口）
systemctl restart sshd

# 连接时指定端口
ssh -p 59832 tom@your-server-ip
```

### 防火墙

只开放必要端口，其余全封：

```bash
# 使用 ufw（Ubuntu 默认防火墙工具）
ufw allow 80/tcp         # HTTP
ufw allow 443/tcp        # HTTPS
ufw allow 59832/tcp      # 你的 SSH 端口
ufw enable               # 启用防火墙
ufw status               # 查看规则
```

### 加固操作的安全顺序

**原则：先确保新方式能用，再关旧方式。** 防止把自己锁在门外。

```
1. 创建普通用户 + 配好密钥 + 测试能登录         ← 新方式就绪
2. 确认没问题 → 禁用密码登录 + 禁止 root 直接登录 ← 关旧方式
3. 改端口 + 防火墙放行新端口                     ← 新端口就绪
4. 确认新端口能连 → 封掉 22                      ← 关旧端口
```

如果先封了旧的再搞新的，中间出问题就连不上了。

---

## 9. 常用命令速查表

### 文件与目录

| 命令 | 作用 | 例子 |
|------|------|------|
| `ls` | 列出目录内容 | `ls -la /home` |
| `cd` | 切换目录 | `cd /etc/nginx` |
| `pwd` | 显示当前目录 | `pwd` |
| `cp` | 复制 | `cp file.txt /tmp/` |
| `mv` | 移动/重命名 | `mv old.txt new.txt` |
| `rm` | 删除 | `rm file.txt`，`rm -r dir/`（删目录） |
| `mkdir` | 创建目录 | `mkdir -p /data/logs`（-p 自动创建父目录） |
| `cat` | 查看文件全部内容 | `cat /etc/hostname` |
| `less` | 分页查看（按 q 退出） | `less /var/log/syslog` |
| `head` / `tail` | 看开头/结尾 | `tail -20 /var/log/syslog`（看最后20行） |
| `tail -f` | 实时跟踪文件尾部 | `tail -f /var/log/nginx/access.log` |
| `grep` | 文本搜索 | `grep "error" /var/log/syslog` |
| `find` | 查找文件 | `find /etc -name "*.conf"` |

### 权限与用户

| 命令 | 作用 | 例子 |
|------|------|------|
| `chmod` | 修改权限 | `chmod u+x script.sh` |
| `chown` | 修改文件所有者 | `chown www:webteam file.txt` |
| `adduser` | 创建用户 | `adduser tom` |
| `usermod` | 修改用户 | `usermod -aG sudo tom`（加入sudo组） |
| `sudo` | 临时以 root 权限执行 | `sudo apt install nginx` |
| `su` | 切换用户 | `su - tom` |
| `whoami` | 显示当前用户名 | `whoami` |

### 进程管理

| 命令 | 作用 | 例子 |
|------|------|------|
| `ps aux` | 查看所有进程 | `ps aux \| grep nginx` |
| `top` | 实时监控 | `top` |
| `kill` | 杀进程 | `kill 1234`，`kill -9 1234`（强制） |
| `bg` / `fg` | 后台/前台 | `ctrl+z` 暂停后 `bg` 放到后台 |

### 网络

| 命令 | 作用 | 例子 |
|------|------|------|
| `ip addr` | 查看 IP 和网卡 | `ip addr` |
| `ping` | 测试网络连通 | `ping 8.8.8.8` |
| `ss -tlnp` | 查看端口监听 | `ss -tlnp` |
| `curl` | HTTP 请求 | `curl http://localhost:80` |
| `curl -I` | 只看响应头 | `curl -I http://example.com` |

### 服务管理

| 命令 | 作用 |
|------|------|
| `systemctl start xxx` | 启动服务 |
| `systemctl stop xxx` | 停止服务 |
| `systemctl restart xxx` | 重启服务 |
| `systemctl status xxx` | 查看状态 |
| `systemctl enable xxx` | 开机自启动 |
| `systemctl disable xxx` | 取消自启动 |
| `journalctl -u xxx` | 查看服务日志 |

### 包管理（apt）

| 命令 | 作用 |
|------|------|
| `apt update` | 更新仓库索引 |
| `apt upgrade` | 更新所有已装软件 |
| `apt install xxx` | 安装 |
| `apt remove xxx` | 卸载 |
| `apt search xxx` | 搜索 |

### 防火墙（ufw）

| 命令 | 作用 |
|------|------|
| `ufw allow 80/tcp` | 放行端口 |
| `ufw deny 3306/tcp` | 封掉端口 |
| `ufw enable` | 启用防火墙 |
| `ufw status` | 查看规则 |
