# 文明6联机-基于IP的游戏发现 (IPv4/IPv6)

> 改自：Hook文明6游戏sendto函数dll和注入程序
>
> 原作者仓库：[https://gitcode.net/PeaZomboss/miscellaneous](https://gitcode.net/PeaZomboss/miscellaneous)

公网 UDP 单播联机，不再需要虚拟局域网！

利用 Hook 技术，拦截游戏对 UDP 的发送、接收等操作，实现基于 IP 的游戏发现和联机

至此，文明6联机不再需要诸如 n2n, OpenVPN, Hamachi 等虚拟局域网，直接公网联机！

主要实现以下两点

- 拦截客户端发现服务器时发出的 UDP 广播包，将其改为单播到指定服务器 IP，实现稳定的游戏发现 (IPv4/IPv6)
- 对于 IPv6
  - 重定向对 `0.0.0.0` 的监听到 `[::0]`
  - 建立 FakeIP 表，将 IPv6 地址映射到 `127.0.127.1 ~ 127.0.127.100` 并提供给 文明6 虚假的 IPv4 地址 (实现原理同 Clash 的 FakeIP)

## 使用方法

双击 `injciv6.exe` 自动注入正在运行的 文明6 进程（一个进程不要多次注入）

双击 `civ6remove.exe` 解除注入

第一次注入后，会在 文明6 目录下生成一个 `injciv6-config.txt` 文件，用于配置要连接的服务器地址。

`injciv6-config.txt` 文件默认只有一行，初始为 `255.255.255.255`

请改成要连接的服务器地址，如 `192.168.1.100`, 或 `2001::1234:5678:abcd:ef01`, 或 `abc.com` 等

要使配置生效，请解除注入后重新注入。

然后就可以使用基于 IP 的游戏发现功能了。

**如果发现不了房间，尝试关闭 Windows 防火墙，以及检查路由器防火墙是否放通**

**如果程序报毒，那我也不知道为啥报毒，把 Windows Defender 关了再运行吧**

- 当使用 ipv4 联机时，一般在客户端上进行注入，配置成服务器的 ipv4 地址即可。服务端无需注入
- 当使用 ipv6 联机时，在客户端注入并配置成服务器的 ipv6 地址的同时。服务端也需要注入，理论上地址可以任意 ipv6 地址，一般建议填 `::0` 。

注：文明6 目录一般为 `\Sid Meier's Civilization VI\Base\Binaries\Win64Steam` 或 `\Sid Meier's Civilization VI\Base\Binaries\Win64EOS`

仅支持 Windows 下的 ~~x86~~ 和 x64 平台 (x86 没测过)

---

也可以单独运行 `injector32.exe` 或者 `injector64.exe` 注入

例如你要注入文明6，那么请使用 `injector64.exe -x=CivilizationVI.exe` 或者 `injector64.exe -x=CivilizationVI_DX12.exe`

如果游戏以管理员权限运行，injector也要以管理员权限运行

## 目录说明

- bin存放编译后的二进制文件输出目录
- inc存放头文件
- src存放源代码
- hookdll存放dll的代码和Makefile
- injector存放注入程序的代码和Makefile
- utils存放实用程序代码
- test存放测试用代码

## 编译方法

首先确保同时安装了x86和x64的*MinGW-w64*工具链，已经有的可以略过
下载地址<https://github.com/niXman/mingw-builds-binaries/releases>  
注意要下载两个，一个带**i686**前缀，一个带**x86_64**前缀  
将它们分别解压到不同目录，然后将二者的bin目录都设置环境变量

当然如果你用msys2也可以，会配置完全没问题

切换工作目录到当前目录，运行`mingw32-make`即可  
如果你单独安装了*GNU make*，那么可以直接运行`make`

> xaxys:
>
> 下载上面的两个压缩包，一个解压到本文件夹下的 mingw32, 一个解压到本文件夹下的 mingw64
>
> 然后运行 env.bat, 会自动设置环境变量, 开箱即用

---

要编译utils下的辅助程序，请到[Lazarus官网](https://www.lazarus-ide.org/)下载并安装最新版的*Lazarus IDE*

然后用Lazarus打开auxtool.lpi工程文件，在菜单栏中依次选择**运行-构建**即可，exe会生成在bin目录下

## 原理简述

### 游戏发现

文明6原游戏发现机制是：

1. 点击刷新后，客户端向局域网发送 `eth.dst=ff:ff:ff:ff:ff:ff && ip.dst=255.255.255.255` 的 UDP 广播包，端口从 `62900` 到 `62999`。共发送 100 个包。包大小一般为 4 字节。

1. 服务器通常在 `62900` 端口监听，收到广播包后，向客户端回复多个大小为 1104 字节的 UDP 单播包，包含服务器信息，格式主要是 `json`, 包含房间信息。

1. 客户端在发送的端口上监听，此时可能会收到服务器回复。不管收没收到，很短的一段时间后，关闭该 Socket。

本程序的原理是：

1. 通过 Hook 程序 `sendto` 函数，在客户端发出 UDP 广播包时，将其改为单播到指定服务器 IP，端口不变。

    - 发送时，建立 Socket 替换表，并建立新的 IPv4 或 IPv6 Socket，将其 bind 到与原 Socket 相同的端口上。
  
    - 通过替换的新 Socket 发送 UDP 单播给目标服务器 IP。

1. 通过 Hook 程序 `recvfrom` 函数，在客户端监听端口时，如果监听的端口在上述替换表中，将其改为从 Socket 替换表中的新 Socket 上接收。

    - 接收时，如果收到的地址为 IPv4 的地址，直接返回给游戏程序。
  
    - 接收时，如果收到的地址为 IPv6 的地址，建立 FakeIP 表，将 IPv6 地址映射到 `127.0.127.1 ~ 127.0.127.100` (暂设) 中顺序分配的一个假 IPv4 地址，并返回给游戏程序。

1. 通过 Hook 程序 `select` 函数，在客户端 select 监听端口（等待服务器回复）时，将其改为 select Socket 替换表中的新 Socket。

### 游戏联机

如果是 IPv4 联机，完成游戏发现后，沿用原游戏联机机制即可，不再需要以下修改，以下修改仅针对 IPv6 联机。

文明6原游戏联机机制是：

1. 上述发现完成后，客户端选择联机房间后，在 `62056` 端口上打开监听，并向目标服务器 IP 的 `62056` 端口发送 UDP 单播包，首包大小一般为 120 字节。

1. 服务器收到后，向客户端 `62056` 端口发送 UDP 单播包，然后联机正常建立。

**注意：** 联机机制使用的是 `WSASendTo` 和 `WSARecvFrom` 函数，而不是 `sendto` 和 `recvfrom` 函数。

**注意：** 游戏联机时，会先 bind 后先在 `62056` 端口上 `WSARecvFrom`，然后再 `WSASendTo`。

本程序的原理是：

1. 通过 Hook 程序 `WSASendTo` 函数，在客户端发出 UDP 单播包时，查询 FakeIP 表，如果目标地址为假 IPv4 地址，则将其改为发送到对应的 IPv6 地址。

1. 通过 Hook 程序 `WSARecvFrom` 函数，操作和 `recvfrom` 函数完全一致。

1. 通过 Hook 程序 `bind` 函数，在客户端开启 `0.0.0.0:62056` 端口监听时，将其改为监听 `[::0]:62056` 端口。并写入 Socket 替换表。

**注意：** 游戏联机时，并未使用 Hook 程序 `select` 函数，而是创建了非阻塞的 Socket，然后不断进行 `WSARecvFrom` 操作，如果建立替换的新 Socket 不是非阻塞的，会导致游戏卡死。

**注意：** 游戏联机时，作为服务端的程序会在创建房间时，执行上述 2 和 3 两步，因此对于 IPv6 联机，服务端也需要注入。

## 后记

非常感谢原作者的注入框架，也非常感谢 DawningW 大佬的帮助
