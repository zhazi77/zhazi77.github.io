---
draft: false
date:
  created: 2025-12-27
categories:
  - SE2SE
authors:
  - zhazi
  - Qwen3-Max
  - ChatGPT5.2
---

# Clash for Linux 是怎么提供网关代理的？

![](./images/clash-for-linux-科研小故事一则.png)

科研小故事一则
{ .caption }

很多人第一次用 Clash for Linux 时，往往会下意识地这么想：

> 我把 Clash 启动了 → 它“提供代理” → 于是需要代理的地方就都会自动走代理。

但很快就会碰到现实的反例：同一台机器上，有的请求已经顺畅地走了代理，有的却像什么都没开一样。比如终端里某些命令行工具正常，而浏览器访问 Google 仍然失败；或者反过来，浏览器好了，换个程序就不行。

比较细心的同学可能会发现，[项目主页](https://github.com/wnlen/clash-for-linux) 上有这样一段“设置代理”的命令：

```bash
# 开启 IP 转发
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# 先清空旧规则
sudo iptables -t nat -F

# 允许本机访问代理端口（避免把自己绕进去）
sudo iptables -t nat -A OUTPUT -p tcp --dport 7890 -j RETURN
sudo iptables -t nat -A OUTPUT -p tcp --dport 7891 -j RETURN
sudo iptables -t nat -A OUTPUT -p tcp --dport 7892 -j RETURN

# 让所有 TCP 流量通过 7892 代理
sudo iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 7892

# 保存规则
sudo iptables-save | sudo tee /etc/iptables.rules
```

看起来是专业的解决方案。但照着做了，却发现问题并没有得到解决，更糟糕的是，还多了一个新的困惑：那这些命令到底在做什么？

因此，本文希望帮助你：

- **形成这样的认识**： Clash 自己并不会“自动接管所有流量”。它只是提供了几个本地端口作为“代理入口”；一个程序能不能走代理，最直接的区别在于——它的请求有没有被送到这些“入口”。

- **理解这段设置代理的命令**：这段 `iptables` 的命令并不是给本机所有程序“自动走代理”用的——它主要是为了让其他设备（比如你的手机或另一台电脑）通过这台 Linux 机器上网时能走 Clash 代理。它的核心原理很简单：把从外部进来的网络请求，用 REDIRECT 规则“改道”到 Clash 监听的透明代理端口（这里是 7892）。

甚至是

- **能够看懂别人的解决方案**：比如[记录一下配置Clash透明代理](https://payloads.online/archivers/2023-08-07/clash-config/)，了解怎么配置本机透明代理，让本机所有程序“自动走代理”。
   
!!! warning "注意"

    本文将基于 [Clash for Linux](https://github.com/wnlen/clash-for-linux) 的默认配置展开。要达成目标3会涉及到一些网络虚拟化的知识，因此（将来可能会）在文末增加拓展内容的形式来实现。

## Clash 提供的三个入口

Clash 启动后，默认会监听三个端口：

![](./images/clash-for-linux-端口监听情况.png)

Clash 默认端口监听情况
{ .caption }

??? info "命令解释"

    ```bash
    sudo ss -tulnp | grep -i clash
    ```

    - `sudo`：以超级用户权限执行命令，用于查看系统所有监听端口（包括需要权限的端口）
    - `ss`：Socket Statistics 的缩写，是现代 Linux 系统中用于查看网络连接状态的工具，功能类似 `netstat` 但更高效
    - `-t`：显示 **TCP** 连接
    - `-u`：显示 **UDP** 连接
    - `-l`：仅显示 **监听状态**（listening）的套接字
    - `-n`：以数字形式显示地址和端口（不解析主机名或服务名）
    - `-p`：显示使用该 socket 的 **进程信息**（包括 PID 和程序名）
    - `|`：管道符，将前一个命令的输出作为下一个命令的输入
    - `grep -i clash`：在输出中搜索包含 "clash" 的行，`-i` 表示忽略大小写

    **翻译：** 查看当前系统中所有正在监听的 TCP/UDP 端口，并筛选出与 Clash 相关的进程及其监听端口信息

它们都能让请求“走代理”，具体差异如下：

| 代理入口类型 | 默认端口 | 如何使用                                      | 是否需要 iptables |
| ------------ | -------- | --------------------------------------------- | ----------------- |
| HTTP 代理    | 7890     | 在支持 HTTP 代理的应用中设置 `127.0.0.1:7890` | ❌ 不需要          |
| SOCKS5 代理  | 7891     | 在支持 SOCKS5 的应用中设置 `127.0.0.1:7891`   | ❌ 不需要          |
| 透明代理     | 7892     | 配合 iptables 规则，自动代理所有流量          | ✅ 需要        |

> 💡 这些端口在 `config.yaml` 中分别由 `port`、`socks-port` 和 `redir-port` 控制，可根据需要修改。

最主要的差别是：

- 7890/7891 端口：程序自己把请求送来，即**显式代理**
- 7892 端口：系统把请求改道送来，**透明代理**

## 显式代理：程序自己送

7890（HTTP）和 7891（SOCKS5）端口提供的是“显式代理”。意思是：应用程序必须**主动连接**这些端口，并明确告诉 Clash：“我想访问 example.com”。

比如你在终端里这样用：
```bash
export https_proxy=http://127.0.0.1:7890
curl https://example.com
```
`curl` 就会直接连到本地 7890 端口，把目标地址一并告诉 Clash。Clash 拿到后，会按配置决定是代理、直连，还是拒绝。

这类代理对系统几乎没有要求。Clash 此时就像一个普通的本地服务——和你运行的 Web 服务器或数据库没本质区别。只要程序支持设置代理（大多数命令行工具都支持），就能正常工作。

但问题也很直接：如果某个程序压根不知道要设代理（比如图形界面软件、Docker 容器、systemd 服务），它的流量就完全绕过 Clash，该直连还是直连。

## 显式代理下的常见问题分析

到这里，我们已经可以解释开头“科研小故事”中的问题了：为什么开启代理后，我能正常下载 PyTorch, 但 sudo apt update 还是不能正常连接？

答案其实非常朴素——**因为 7890 是显式代理入口**。

这意味着，只有那些“知道自己要走代理”的程序，才会主动把网络请求发往这个端口。对于大多数命令行工具来说，它们是通过读取环境变量（如 `http_proxy` 和 `https_proxy`）来决定是否使用代理的。

而 Clash for Linux 提供的 `proxy_on` 函数，本质上就是在**当前终端会话**中设置这样一组环境变量，从而告诉这些命令行工具代理入口。

```bash title="/etc/profile.d/clash.sh"
proxy_on() {
        export http_proxy=http://127.0.0.1:7890
        export https_proxy=http://127.0.0.1:7890
        export no_proxy=127.0.0.1,localhost
        export HTTP_PROXY=http://127.0.0.1:7890
        export HTTPS_PROXY=http://127.0.0.1:7890
        export NO_PROXY=127.0.0.1,localhost
        echo -e "\033[32m[√] 已开启代理\033[0m"
}

# 关闭系统代理
proxy_off(){
        unset http_proxy
        unset https_proxy
        unset no_proxy
        unset HTTP_PROXY
        unset HTTPS_PROXY
        unset NO_PROXY
        echo -e "\033[31m[×] 已关闭代理\033[0m"
}
......
```

因此，相关问题总是出在：**需要代理的流量，没有被送到正确的位置上**。例如：

???+ example "代理开启了，为什么 sudo apt update 还是连不上？"

    这是一个极具迷惑性的坑：同一个终端里，普通 `curl` 好好的，一加 `sudo` 就不行。

    其实原因并不不复杂：**`sudo` 默认不会把这些环境变量原封不动地传给 root 的进程。**

    于是 `sudo apt update` 完全看不到 `http_proxy/https_proxy`，于是还是直连导致更新失败

    所以一个很简单的解决办法就是保留环境变量：

    ```bash
    sudo -E apt update # -E：保留当前环境变量
    ```

??? example "为什么这个终端可以，新开一个终端又不行了？"

    原因同样直接：`proxy_on` 设置的是**当前 shell 进程的环境变量**，属于临时生效的会话级配置，并非系统全局状态。

    - **当前终端**：已执行 `proxy_on` → 环境变量存在 → 程序知道代理入口  
    - **新开终端**：启动了全新的 shell 进程 → 没有继承之前的 `export` → 代理“消失”

    所以新终端里的命令（如 `pip`、`curl`）自然不会走代理。

    那么一个简单的解决办法就是在新终端中手动设置环境变量：

    ```bash
    echo 'export http_proxy=http://127.0.0.1:7890' >> ~/.bashrc
    echo 'export https_proxy=http://127.0.0.1:7890' >> ~/.bashrc
    echo 'export no_proxy=127.0.0.1,localhost' >> ~/.bashrc
    ```

??? example "终端能下 PyTorch，但浏览器还是打不开 Google?"

    这个问题和上一个本质相同，只是发生在不同类型的程序上。

    `proxy_on` 仅在**当前终端的环境**中设置了代理变量，因此只有**从该终端启动的程序及其子进程**才会读取这些变量并使用代理。

    而大多数桌面浏览器（如 Chrome、Firefox）是由图形界面（如 GNOME 或 KDE）直接启动的：

    - 它们**不是当前 shell 的子进程**
    - 也**不会主动读取你在某个终端里设置的环境变量**

    所以，浏览器的流量依然走直连，访问 Google 自然失败。

    解决方法并不神秘：**只需在浏览器的网络设置中手动配置代理**，例如：
    - HTTP 代理：`127.0.0.1:7890`
    - SOCKS5 代理：`127.0.0.1:7891`

    这本质上就是让浏览器**显式地把请求发送到 Clash 提供的代理入口**，从而纳入代理规则的管理范围。


## 透明代理：系统帮忙送

为了解决“程序不配合”的问题，Clash 还提供了 7892 端口（`redir-port`），用于透明代理。

所谓“透明”，指的是：**程序完全不知道自己在走代理**。它依然像平常一样直连目标站点（例如 `curl https://example.com`），但在数据包真正发出前，Linux 内核会把这条连接**悄悄改道**到本地的 7892 端口，让 Clash 来处理。

这依赖 Linux 内核的 **Netfilter 框架**——它在网络协议栈的关键路径上设置了可编程的检查点（hook），允许对流经的数据包进行拦截或修改。而开头那串命令中的 `iptables` 就是用户空间用来配置这些检查点行为的工具。

最典型的规则配置长这样：

```bash
iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-ports 7892
```

它的意思是：“本机进程发出的 TCP 连接的流量，先别出网，改道送到 7892”。

??? info "一些被扫到地毯下的细节"

    实际配置中，为了避免 Clash 自己的流量也被重定向（造成代理循环），通常需要专门创建一个新用户（比如叫 clash）来运行 Clash，并在 iptables 规则中跳过这个用户的流量。
    同时，还要排除发往本机（如 127.0.0.1）的连接，否则本地服务（比如开发服务器）也会被错误代理。
    
    另外，这些配置只对 TCP 有效。UDP（如 QUIC、DNS）和 ICMP（如 `ping`）不会走 7892 端口。想覆盖 UDP 通常要用 TProxy 等更“硬核”的方案。

??? note "Netfilter 的五条内置链（Chains）"

    Netfilter 在数据包穿越网络协议栈时设定了五个标准检查点（称为“链”），每条链对应不同的流量路径：

    - **PREROUTING**：刚进入网络接口的数据包（在路由决策前）
    - **INPUT**：发往本机进程的数据包（路由后，交付前）
    - **FORWARD**：需要本机转发的数据包（非本机目的地）
    - **OUTPUT**：本机进程主动发出的数据包（发送前）
    - **POSTROUTING**：即将离开本机的数据包（路由和 NAT 后）

    `iptables` 规则就是添加到这些链上的。例如，透明代理中用到的 `-A OUTPUT`，就是在 **本机出站流量** 路径上插入规则。
    
    更具体的介绍见：[Linux 网络虚拟化](https://icyfenix.cn/immutable-infrastructure/network/linux-vnet.html)

??? info "命令解释"

    ```bash
    iptables -t nat -A OUTPUT -p tcp -j REDIRECT --to-ports 7892
    ```

    - `-t nat`：使用 **nat 表**（存放以“改地址/改端口”为目的的规则的表）
    - `-A OUTPUT`：作用于 **本机进程发出的连接**
    - `-p tcp`：只处理 **TCP** 流量
    - `-j REDIRECT --to-ports 7892`：把匹配到的连接 **改道到本机 7892 端口**

    **翻译：**把“本机发出的 TCP 数据包”在出网前拦下来，统一送到 `127.0.0.1:7892` 让 Clash 接管。

## 网关模式：帮外部流量做代理

项目主页中“设置代理”的那一串命令其实是在配置**网关模式**——让这台 Linux 机器为局域网中的其他设备（比如连热点的手机）提供透明代理。

和本机透明代理使用 `OUTPUT` 链不同，网关模式需要在 **`PREROUTING` 链** 上添加规则：

```bash
iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 7892
```

因为来自外部设备的流量**不是本机进程发出的**，而是从网卡进入后、路由决策前就到达 `PREROUTING`。只有在这里拦截，才能把它们重定向到 Clash 的 7892 端口。

### 设置网关模式

现在，我们可以逐行解读项目主页给出的那段“设置代理”命令，看看它是怎么让 Clash 成为网关的，并完成开头约定的第二个目标。

#### 1. 开启 IP 转发

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

这两行命令启用 Linux 的 IP 转发功能，使这台机器具备“路由器/网关”的能力：当局域网设备把它设为默认网关时，**未被 Clash 接管的流量**也能被系统正常转发到外网。否则这些需要转发的包会在内核转发路径被丢弃/拒绝，表现为“部分网站/应用不通”。

??? info "命令解释"

    ```bash
    echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

    - `echo "net.ipv4.ip_forward = 1"`：生成一行内核参数配置，启用 IPv4 包转发功能  
    - `|`：将该行内容通过管道传递给下一个命令  
    - `sudo tee -a /etc/sysctl.conf`：以超级用户权限将内容**追加**写入 `/etc/sysctl.conf`（系统级内核参数配置文件）  
    - `sudo sysctl -p`：从默认配置文件（通常是 `/etc/sysctl.conf`）**重新加载**所有内核参数  

    **翻译：** 立即且永久性地开启 Linux 的 IP 转发能力，使本机可以作为路由器转发来自其他设备的网络流量。


#### 2. 清空旧的 NAT 规则（⚠️非必要，且有风险）

```bash
sudo iptables -t nat -F
```

这行命令清空 `nat` 表中的所有规则。这么做是为了避免旧规则（比如之前配置的 Docker 网络、手动添加的端口转发等）与新规则冲突。  

!!! warning "破坏性操作注意"
    
    这么做会导致系统中那些着依赖 NAT 的服务（如容器、虚拟机）的网络受到影响，直到你重新配置或重启相关服务。
    
    推荐的工程做法：不要 flush 全表，而是把 Clash 相关规则放到一个自定义链里（例如 CLASH），只维护这一小段规则，避免破坏 Docker/虚拟机等已有 NAT。

??? info "命令解释"

    ```bash
    sudo iptables -t nat -F
    ```

    - `-t nat`：指定操作 **nat 表**（用于地址/端口转换）  
    - `-F`：**清空（Flush）** 该表中所有链的规则  

#### 3. 保护本机对代理端口的直接访问（非必要）

```bash
sudo iptables -t nat -A OUTPUT -p tcp --dport 7890 -j RETURN
sudo iptables -t nat -A OUTPUT -p tcp --dport 7891 -j RETURN
sudo iptables -t nat -A OUTPUT -p tcp --dport 7892 -j RETURN
```

这三条规则的作用是：当本机程序**主动连接** Clash 的代理端口（7890/7891/7892）时，立即跳出 NAT 处理（`RETURN`），不再匹配后续规则。
这是为了防止“**流量自环**”——比如 Clash 自己发起的出站连接又被重定向回自己。  

> 这组 OUTPUT 规则主要用于“本机也做透明代理/本机显式代理与透明代理混用”的兼容性场景；如果你只关心局域网设备走网关（只用 PREROUTING 劫持外部流量），通常可以不加。

??? info "命令解释"

    ```bash
    sudo iptables -t nat -A OUTPUT -p tcp --dport 7890 -j RETURN
    sudo iptables -t nat -A OUTPUT -p tcp --dport 7891 -j RETURN
    sudo iptables -t nat -A OUTPUT -p tcp --dport 7892 -j RETURN
    ```

    - `-t nat`：使用 **nat 表**  
    - `-A OUTPUT`：向 **本机出站链** 末尾追加规则  
    - `-p tcp --dport 789X`：匹配目标端口为 Clash 各代理端口（7890/7891/7892）的 TCP 连接  
    - `-j RETURN`：立即跳出当前链，**不再应用后续 NAT 规则**  

#### 4. 重定向外部 TCP 流量到透明代理端口

```bash
sudo iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 7892
```

这是网关模式的核心：将所有**从网络接口进入的 TCP 流量**（即来自局域网设备的请求）重定向到本地 7892 端口，交由 Clash 的透明代理处理。  
此时，Clash 会根据其规则判断该连接是否需要代理，并完成后续的转发。  

> 前提条件：其他设备的默认网关已设为这台 Linux 主机（例如通过 Wi-Fi 热点共享）；

??? info "命令解释"

    ```bash
    sudo iptables -t nat -A PREROUTING -p tcp -j REDIRECT --to-ports 7892
    ```

    - `-t nat`：使用 **nat 表**  
    - `-A PREROUTING`：向 **路由前入站链** 末尾追加规则  
    - `-p tcp`：仅处理 **TCP 协议** 流量  
    - `-j REDIRECT --to-ports 7892`：将匹配的连接**重定向到本机 7892 端口**  

    **翻译：** 将来自局域网设备的 TCP 请求在进入系统前改道至 Clash 的透明代理端口，实现对外部设备的无感代理。

!!! warning "注意"
 
    注意：这条规则会把“局域网访问网关本机的 TCP 服务”（例如 SSH/管理页面）也一起重定向，可能导致连不上网关本机服务。更稳的做法是先绕过内网/保留地址：

    ```bash
    # 可选：先绕过内网/保留地址，避免局域网访问网关本机/内网服务也被劫持
    sudo iptables -t nat -I PREROUTING 1 -p tcp -d 10.0.0.0/8 -j RETURN
    sudo iptables -t nat -I PREROUTING 1 -p tcp -d 172.16.0.0/12 -j RETURN
    sudo iptables -t nat -I PREROUTING 1 -p tcp -d 192.168.0.0/16 -j RETURN
    sudo iptables -t nat -I PREROUTING 1 -p tcp -d 127.0.0.0/8 -j RETURN
    ```

    参考：[网关代理（透明代理）](https://blog.waynecommand.com/post/gateway-proxy)

#### 5. 启用转发放行 + 出口 NAT（补充步骤）

!!! tip "说明"

    项目主页给出的命令只包含了上面的 4 步（开启 IP 转发 + 配置透明代理重定向）。
    但在实际部署中，如果希望这台 Linux 稳定地作为局域网设备的默认网关工作，通常还需要补充“转发放行（FORWARD）+ 出口 NAT（MASQUERADE）”这一步。
    本节为补充内容，参考了 Wayne 的[网关代理（透明代理）](https://blog.waynecommand.com/post/gateway-proxy)一文中的网关配置思路。

    但即使如此，本文配置的透明代理规则依然仅覆盖 TCP。若出现部分 App/域名异常，建议优先检查客户端 DNS 是否指向网关，或参考 Wayne 的 DNS 配置一节。


第 4 步做的是“把局域网来的 TCP 连接交给 Clash 处理”，但这并不等于系统已经具备完整的“网关路由器能力”。

要让局域网设备**真的能通过这台 Linux 上网**，通常还需要补上两件事：

1. **允许转发（FORWARD）**：放行局域网 → 外网 的转发流量
2. **出口 NAT（MASQUERADE）**：把局域网私有地址“伪装”成网关自己的外网地址，否则上游路由/运营商无法把回包正确送回局域网设备

最简单的补丁如下（按需修改网段）：

```bash
# 允许转发（简单版：放行所有转发流量）
sudo iptables -t filter -A FORWARD -j ACCEPT

# 出口 NAT：把局域网地址伪装成网关自己的外网地址
# <LAN_CIDR>：你的局域网网段，例如 192.168.1.0/24
sudo iptables -t nat -A POSTROUTING -s <LAN_CIDR> -j MASQUERADE
```

> 如果只做第 4 步、不做这一步，可能出现“部分网站/应用不通、偶尔能用偶尔不行”，看起来像 Clash 配置问题，但本质是**网关转发/NAT 不完整**。

??? info "命令解释"

    ```bash
    sudo iptables -t filter -A FORWARD -j ACCEPT
    sudo iptables -t nat -A POSTROUTING -s <LAN_CIDR> -j MASQUERADE
    ```

    - `-t filter -A FORWARD -j ACCEPT`：在 **filter 表的 FORWARD 链**追加一条规则，允许本机转发通过的流量（让它像路由器一样“转发包”）  
    - `-t nat -A POSTROUTING ... -j MASQUERADE`：在 **nat 表的 POSTROUTING 链**追加规则，对从指定局域网网段出去的流量做“源地址伪装”（SNAT）  
      - `-s <LAN_CIDR>`：仅对这个源网段生效（请替换为你的 LAN 网段）  
      - `MASQUERADE`：一种动态 SNAT 方式，适合网关外网 IP 可能变化（如 DHCP、热点共享）

    **翻译：** 允许局域网设备的流量通过本机转发，并在流量离开网关时做 NAT，让外网回包能正确返回网关，再由网关转发回局域网设备。

??? warning "更安全的转发放行（可选）"

    上面的 `FORWARD -j ACCEPT` 是“最省事”的写法，但也最宽松。

    如果你希望更保守一点（只放行 “LAN → WAN” 并允许回包），可以用更严格的版本（需要你知道 LAN/WAN 网卡名）：

    ```bash
    # <LAN_IF>：内网入口网卡（如 wlan0/ap0/eth1）
    # <WAN_IF>：外网出口网卡（如 eth0/ens33）
    # <LAN_CIDR>：你的局域网网段（如 192.168.1.0/24）
    sudo iptables -A FORWARD -i <LAN_IF> -o <WAN_IF> -j ACCEPT
    sudo iptables -A FORWARD -i <WAN_IF> -o <LAN_IF> -m state --state ESTABLISHED,RELATED -j ACCEPT
    sudo iptables -t nat -A POSTROUTING -s <LAN_CIDR> -o <WAN_IF> -j MASQUERADE
    ```

### 持久化开启（可选）

前面的配置（特别是 `iptables` 规则）**仅在当前系统运行期间有效**。一旦重启，所有规则都会丢失，网关代理也将失效。
为了让配置持久化，你需要在系统启动时自动恢复 iptables 规则:

```bash
sudo iptables-save | sudo tee /etc/iptables.rules
```

这会将当前生效的规则导出到 `/etc/iptables.rules`。

??? info "命令解释"

    ```bash
    sudo iptables-save | sudo tee /etc/iptables.rules
    ```

    - `sudo iptables-save`：导出当前所有 iptables 规则为可读文本格式
    - `sudo tee /etc/iptables.rules`：以 root 权限将规则写入持久化配置文件

不过**保存 ≠ 自动加载**，要实现开机自启动，还需要搭配自动加载机制使用，常见的有两种方案：`rc.local` 或更现代化的 `systemd`。

#### 方案一：使用 `/etc/rc.local`

这是一种传统但直观的方法，也是项目主页推荐的方案，适用于快速部署。尽管现代 Linux 发行版已转向 systemd，但多数仍通过兼容服务保留对 /etc/rc.local 的支持。

> 在 Ubuntu 16.04+、Debian 9+、CentOS 7+ 等系统中，rc-local.service 默认存在但可能未启用。

```bash title="配置步骤"
# 创建 /etc/rc.local
sudo cat > /etc/rc.local <<EOF
#!/bin/bash
iptables-restore < /etc/iptables.rules
exit 0
EOF

# 赋予执行权限
sudo chmod +x /etc/rc.local

# 启用兼容服务
sudo systemctl enable rc-local
```

#### 方案二：创建专用 systemd 服务

systemd 推荐为每个独立任务定义专属服务。对于恢复防火墙规则，可以创建如下单元文件：

```ini
# /etc/systemd/system/iptables-restore.service
[Unit]
Description=Restore iptables rules at boot
After=network-pre.target
Before=network.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c '/sbin/iptables-restore < /etc/iptables.rules'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

然后启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable iptables-restore
```

通过以上任一方案，你的 iptables 规则即可在系统重启后自动恢复，确保网关、NAT、端口转发等网络功能持续可用。

## 结语

最初只是想搞清楚项目主页上那串 `iptables` 命令到底在做什么，顺手查了些资料，接触了一点网络虚拟化的概念。整理了不少笔记，觉得扔掉可惜，干脆写成一篇文章。

没想到写着写着，越挖越深，还发现自己之前的一些理解其实存在偏差。于是不断修正、补充，内容也越积越多……一度不得不提醒自己：**别跑太远，回到最初的问题**。

Anyway，总算完成了。

过程中借助了不少 AI 的帮助，虽然尽力核对，但难免疏漏。如果文中仍有错误.....那我先滑跪了。


## 参考文献
- [【全网最细】openclash新手入门教程指南！...](https://www.youtube.com/watch?v=s84CWgKus4U)
- [网关代理（透明代理）](https://blog.waynecommand.com/post/gateway-proxy)
- [记录一下配置Clash透明代理](https://payloads.online/archivers/2023-08-07/clash-config/)
- [What is the correct substitute for rc.local ...](https://unix.stackexchange.com/questions/471824/what-is-the-correct-substitute-for-rc-local-in-systemd-instead-of-re-creating-rc)