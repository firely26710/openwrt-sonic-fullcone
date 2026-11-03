# openwrt-sonic-fullcone

简体中文 | [English](./README.en.md)

适用于 OpenWrt 的 SONiC 风格全锥形 NAT（Full Cone NAT），支持 per-zone、per-protocol 粒度控制。

## 这是什么

将 [SONiC 的 fullcone NAT 内核补丁](https://github.com/sonic-net/sonic-linux-kernel/blob/master/patches-sonic/Support-for-fullcone-nat.patch)（Akhilesh Samineni / Broadcom）移植到 OpenWrt，并集成了支持细粒度控制的防火墙和 LuCI 界面。

### 工作原理

内核补丁在 conntrack 内部增加了第二张哈希表（`nat_by_manip_src`），以**转换后的 3-tuple**（协议、源 IP、源端口）为索引：

- **SNAT 方向**：3-tuple 唯一性保证——一个 (proto, src_ip, src_port) 只属于一个内网端点，这个端点的所有连接都复用它，实现端点无关映射（EIM）。SONiC 的补丁只在规则带端口范围时检查 3-tuple，fw4 和 fw3 生成的规则都带（fw3 的 UDP-Lite 规则除外），见[SNAT 规则的端口范围](#snat-规则的端口范围)
- **DNAT 方向**：反向查找——入站数据包通过哈希表找到原始内部主机，实现端点无关过滤（EIF）。和 SONiC 一样，内核不区分映射是 fullcone 规则还是普通 masquerade 建立的，fw4 和 fw3 只对放进 fullcone 的协议做反向查找
- **协议**：fw4 和 fw3 对 TCP、UDP，以及内核开启了对应连接跟踪的 UDP-Lite、SCTP、DCCP（OpenWrt 默认没开；6.18 内核已经去掉了 DCCP）使用 fullcone，其余协议走普通 masquerade。ICMP、GRE 这类没有端口的协议做 fullcone 没有意义
- **只影响 fullcone 规则**：fullcone 标志位是 per-rule 的，不是全局的。普通 NAT 的映射也会登记到这张哈希表里

### 补丁组成

内核部分保持 SONiC 原版，本项目自己的修改放在单独的补丁里，不和 SONiC 的代码混在一起：

| 补丁 | 内容 |
|---|---|
| 984 | SONiC 原版补丁（移植到 6.6+），加上已提交给 SONiC 上游的两处修复：外部主动连入时回包的方向（[sonic-net/sonic-linux-kernel#649](https://github.com/sonic-net/sonic-linux-kernel/pull/649)），以及 `nat_by_manip_src` 的锁（[sonic-net/sonic-linux-kernel#653](https://github.com/sonic-net/sonic-linux-kernel/pull/653)） |
| 985、986 | 本项目的 iptables `FULLCONE` target 和 nftables `fullcone` 表达式，分别编进 `xt_MASQUERADE.ko` 和 `nft_masq.ko`。`fullcone` 表达式可用于 ip、ip6、inet 表 |
| 987 | 本项目在 SONiC 之上的修复，没有提交给 SONiC：多核并发时的映射竞争、规则不带端口范围时的 3-tuple 检查、普通 NAT 的误丢包（例如 `masquerade fully-random` 下的 ESP）。删掉这个文件，内核部分就是“SONiC + 上游修复” |

用户态的 nftables 补丁也一样：002 保持原样，JSON 输入和 `nft -o` 的崩溃修复放在 003。

### 支持的内核版本

6.6、6.12、6.18（三个版本共用同一份补丁——相关的 nf_nat_core.c 结构完全一致）

## 安装

### 先决条件

- OpenWrt 源码树（master / 25.12 / 24.10）
- 内核 6.6 / 6.12 / 6.18
- 主机已安装 `git`、`curl`

> 23.05 已不再支持：该分支的 iptables 1.8.8 没有 `extensions/libxt_NAT.c`（1.8.9 才引入），firewall4 补丁也有 hunk 无法应用。

### 一键安装

在 OpenWrt 源码目录下执行：

```bash
# 先更新 feeds（LuCI 补丁需要）
./scripts/feeds update -a
./scripts/feeds install -a

# 一键应用所有补丁
curl -sSL https://raw.githubusercontent.com/mufeng05/openwrt-sonic-fullcone/master/add_sonic_fullcone.sh | bash

# 编译
make menuconfig   # 无需额外勾选，fullcone 编译进现有 nft_masq.ko / xt_MASQUERADE.ko 模块
make -j$(nproc)
```

脚本会自动 clone 仓库、检测内核版本、复制补丁到对应位置，完成后自动清理临时文件。

> **注意**：除了复制补丁，脚本还会往 `package/libs/libnftnl/Makefile` 里加一行 `PKG_FIXUP:=autoreconf`。
> 因为 libnftnl 补丁改了 `src/Makefile.am`，会触发 automake 用打包时的确切版本（libnftnl 1.3.1 是 `automake-1.17`）重新生成 `Makefile.in`，而该版本在构建环境中通常不存在，导致 `automake-1.17: command not found`。
> 这是对 OpenWrt 源码树本身的修改，`git pull` 更新源码后会被还原，届时**重新执行一次本脚本**即可。已经编译失败过的话，还需要 `make package/libs/libnftnl/clean` 再编。

### 卸载

删除补丁文件后重新编译即可：

```bash
rm -f target/linux/generic/hack-*/984-add-sonic-fullcone-*.patch
rm -f target/linux/generic/hack-*/985-add-sonic-fullcone-*.patch
rm -f target/linux/generic/hack-*/986-add-sonic-fullcone-*.patch
rm -f target/linux/generic/hack-*/987-sonic-fullcone-*.patch
rm -f package/network/utils/iptables/patches/901-sonic-fullcone.patch
rm -f package/network/config/firewall*/patches/001-sonic-fullcone.patch
rm -f feeds/luci/applications/luci-app-firewall/patches/001-add-fullcone-options.patch

make target/linux/clean
make package/network/utils/iptables/clean
make package/network/config/firewall/clean
make package/network/config/firewall4/clean
make package/feeds/luci/luci-app-firewall/clean
make -j$(nproc)
```

注意：libnftnl 和 nftables 补丁（添加 `fullcone` 关键字）保留即可——它们不会影响普通行为。

## 配置

### 配置逻辑

`defaults.fullcone` 是**全局总开关**：

- **关闭时**（默认）：所有 fullcone 功能禁用，zone 里的设置无效
- **开启时**：功能可用，但每个 zone 还需要**单独勾选** fullcone 才会生效

这和 OpenWrt 的 `flow_offloading` 是同样的模式。fw4 和 fw3 启动时都会检查内核是否支持 fullcone（fw3 看的是 `/proc/net/ip_tables_targets` 里有没有 `FULLCONE`），不支持就给出警告、关掉 fullcone，只做普通 masquerade。

### Web 界面（LuCI）

LuCI 补丁将 fullcone 选项直接集成到 OpenWrt 原生防火墙配置页面中，无需安装额外的 LuCI 应用。

**全局设置**（网络 → 防火墙 → 常规设置）：
- "Fullcone NAT" 复选框 — 全局总开关

![全局设置](img/luci-global-settings.png)

**Zone 编辑**（网络 → 防火墙 → 区域 → 点击编辑）：
- **常规标签**："Fullcone NAT" 复选框（仅在开启伪装时出现），在 zone 列表和编辑弹窗中均可见
- **高级标签**："Fullcone protocols" 多选框（TCP、UDP、UDP-Lite、SCTP、DCCP）— 限定协议

![Zone 编辑](img/luci-zone-edit.png)

### UCI 命令行

```bash
# 1. 打开全局总开关（必须）
uci set firewall.@defaults[0].fullcone='1'

# 2. 对 wan zone 启用 fullcone
uci set firewall.@zone[1].fullcone='1'

# 3. 可选：仅对 UDP 启用（推荐用于游戏/P2P）
uci add_list firewall.@zone[1].fullcone_proto='udp'

# 应用
uci commit firewall
/etc/init.d/firewall restart

# 验证
nft list ruleset | grep fullcone
```

### UCI 配置文件示例

**基本用法**：全 zone 全协议 fullcone

```
config defaults
    option fullcone '1'           # 全局总开关

config zone
    option name 'wan'
    option masq '1'
    option fullcone '1'           # 此 zone 启用 fullcone
```

生成的 nftables 规则：

```nft
chain srcnat_wan {
    meta nfproto ipv4 meta l4proto { tcp, udp, udplite, sctp, dccp } fullcone to :1024-65535
    meta nfproto ipv4 masquerade                 # 其余协议
}
chain dstnat_wan {
    meta nfproto ipv4 meta l4proto { tcp, udp, udplite, sctp, dccp } fullcone
}
```

在设备上用 `nft list ruleset` 查看时，UDP-Lite、SCTP、DCCP 会显示成协议号，例如 `meta l4proto { tcp, udp, 33, 132, 136 }`：OpenWrt 的 musl 不认识这几个协议名，规则本身没有区别。fw3 用 `iptables -t nat -S` 查看时，DCCP 同样显示成 33。

**仅 UDP**：

```
config zone
    option name 'wan'
    option masq '1'
    option fullcone '1'
    list fullcone_proto 'udp'     # 仅 UDP 走 fullcone，其余走标准 masquerade
```

生成的 nftables 规则：

```nft
chain srcnat_wan {
    meta nfproto ipv4 meta l4proto udp fullcone to :1024-65535  # UDP → fullcone
    meta nfproto ipv4 masquerade                                # 其余 → 标准 masquerade
}
chain dstnat_wan {
    meta nfproto ipv4 meta l4proto udp fullcone                 # 仅 UDP 反向映射
}
```

zone 设置了 `masq_src` / `masq_dest` 时，fullcone 规则带上同样的地址限制，后面紧跟一条同样限制的 masquerade 处理其余协议；不在限制范围内的流量既不走 fullcone 也不做伪装，和没开 fullcone 时一样。

`fullcone_proto` 只接受 `tcp`、`udp`、`udplite`、`sctp`、`dccp`（也可以写 `tcpudp`），其他值会被忽略并给出警告（fw4 可以用 `fw4 check` 查看，fw3 在启动时输出）；列出的协议全都不适用时，这个 zone 不启用 fullcone。

### 高级：手写 nftables 规则

如需更复杂的匹配条件，可使用 `/etc/nftables.d/`：

```nft
# /etc/nftables.d/10-fullcone-custom.nft
table inet fullcone-custom {
    chain srcnat {
        type nat hook postrouting priority srcnat + 1; policy accept;
        oifname "eth1" ip saddr 192.168.1.0/24 meta l4proto udp fullcone to :1024-65535
    }
    chain dstnat {
        type nat hook prerouting priority dstnat + 1; policy accept;
        iifname "eth1" meta l4proto udp fullcone
    }
}
```

注意：
- 使用自定义规则时，应关闭该 zone 的 fw4 fullcone 选项以避免冲突。
- `fullcone` 表达式可以写在 `inet`、`ip`、`ip6` 表里。
- **SNAT 一侧务必带端口范围**（`to :1024-65535`），原因见下一节。DNAT 一侧不需要。
- 内核不区分映射是 fullcone 规则还是普通 masquerade 建立的，DNAT 一侧的 fullcone 会把入站包转给这个地址上所有已有的映射。不要在同一个协议上混用两者，否则普通 masquerade 的映射也会向陌生来源开放。

### SNAT 规则的端口范围

fw4 和 fw3 生成的 SNAT 规则带端口范围 1024-65535（nft 写作 `to :1024-65535`，iptables 写作 `--to-ports 1024-65535`）。两者都只接受跟在协议匹配之后的端口范围，所以 fullcone 规则先匹配有端口的协议，其余协议交给后面那条 masquerade。DNAT 方向不带范围，与 SONiC 一致。

这个范围不只是为了避开低端口：SONiC 的补丁只在规则带端口范围时检查 3-tuple，不带范围时，转换后的端口只经过普通的 5-tuple 检查。两台内网主机用同一个源端口访问不同的目标时，会拿到同一个外部端口，陌生来源的包只会投给其中一台。SONiC 自己不会走到这条路径：它的 [natmgr](https://github.com/sonic-net/sonic-swss/blob/master/cfgmgr/natmgr.cpp) 只在配置了端口范围时才加 `--fullcone`。987 补丁让不带范围的规则也检查 3-tuple。fw3 的 UDP-Lite 规则就不带范围：iptables 不接受 UDP-Lite 规则的 `--to-ports`。

**范围从 1024 开始**，所以内网主机使用 1024 以下的源端口时会被映射到 1024 以上。这是有意的：全锥会把发往某个已映射外部端口的所有入站转给内网主机，若内网客户端以源端口 500 出站（IPsec IKE），而路由器自己也在 WAN 上跑 IPsec，路由器的 500 端口就会被这条映射占去（按机制推断，未实测）。1024 以上的路由器服务端口（例如 WireGuard 51820）不受这个范围保护。

## 验证

刷机后检查：

```bash
# 1. 确认内核支持 fullcone
echo 'add table ip test; add chain ip test t { fullcone; }' | nft -c -f -   # fw4：没报错说明支持
grep -x FULLCONE /proc/net/ip_tables_targets                              # fw3：输出 FULLCONE 说明支持

# 2. 检查规则是否生成（取决于 UCI 配置）
nft list ruleset | grep fullcone                            # fw4
iptables -t nat -S | grep FULLCONE                          # fw3

# 3. 测试 NAT 类型（局域网电脑上）
pystun3   # 或 Windows 上用 NatTypeTester
# 结果应为 "Full Cone"

# 4. 测试外部主动发起的 TCP 连接（局域网电脑上运行 Natter）
python3 natter.py   # 不带参数即测试模式，会打印 Please check [ http://公网IP:端口 ]
# 输出里 "WAN > 公网IP:端口" 应为 [ OPEN ]（检测服务访问不了时显示 [ UNKNOWN ]）。
# 更直接的是用局域网外的设备（如关掉 Wi-Fi 的手机）打开这个地址，应能看到 "It works!"
# 然后在路由器上找外部设备连进来的那条（第一个 src= 是外部设备的地址）：
grep "dport=端口 " /proc/net/nf_conntrack
# 应为 ESTABLISHED 且带 [ASSURED]；停在 SYN_SENT 且带 [UNREPLIED] 说明回包没有经过 NAT
```

第 3 步只能证明外部主动发来的包进得来：STUN 服务器换一个地址发响应，客户端收到就判定为 Full Cone，并不需要回复。内网主机对外部主动连接的回包有没有被正确转换，要靠第 4 步。TCP 必须完成三次握手，这条路径有问题时连接根本建不起来，WAN 口抓包还能看到源地址是内网 IP 的 SYN+ACK，[#6](https://github.com/mufeng05/openwrt-sonic-fullcone/issues/6) 就是这种情况。第 4 步用到 [Natter](https://github.com/MikeWang000000/Natter)，并要求路由器 WAN 口是公网 IP，或者上级 NAT 也是全锥形。

注意：已经建立的连接会沿用开启 fullcone 之前的 NAT 行为，`firewall restart` 不会改变它们。测试前重启路由器最稳妥；在 NSS/ECM 这类把连接交给硬件转发的平台上，清空 conntrack 也撤不掉已经加速的连接。

## 已知限制

### fw3 (iptables) 相关

- **fw3 代码只对 IPv4 生成 NAT 规则**：firewall3 的 `zones.c` 里 `print_zone_rule` 在 `FW3_TABLE_NAT` 分支中有 `if (zone->masq && handle->family == FW3_FAMILY_V4)` 判断，IPv6 走不同路径。因此本项目的 fw3 patch 只生成 IPv4 fullcone 规则。

  **手动添加 IPv6 fullcone**：内核侧的 984/985 补丁和 901 用户态补丁都是 IPv4/IPv6 通用的，`FULLCONE` target 对两个协议族都注册了。可以在 `/etc/firewall.user` 里手动加（端口范围的作用见[SNAT 规则的端口范围](#snat-规则的端口范围)）：

  ```bash
  ip6tables -t nat -A POSTROUTING -o wan -p tcp -j FULLCONE --to-ports 1024-65535
  ip6tables -t nat -A POSTROUTING -o wan -p udp -j FULLCONE --to-ports 1024-65535
  ip6tables -t nat -A POSTROUTING -o wan -j MASQUERADE
  ip6tables -t nat -A PREROUTING  -i wan -p tcp -j FULLCONE
  ip6tables -t nat -A PREROUTING  -i wan -p udp -j FULLCONE
  ```

- **fw3 的 UDP-Lite 规则没有端口范围**：iptables 只在 TCP、UDP、SCTP、DCCP、ICMP 规则上接受 `--to-ports`，所以 UDP-Lite 的 FULLCONE 规则不带范围。没有 987 补丁时，UDP-Lite 的 3-tuple 唯一性不生效（见[SNAT 规则的端口范围](#snat-规则的端口范围)）。有 987 时唯一性没问题，但源端口在 1024 以下的连接会保留原端口，被占用时也只在 1024 以下另选。

### 通用限制

- **端口奇偶保持（RFC 4787 REQ-3b）未实现**：fullcone 端口分配不保证奇数端口映射到奇数、偶数映射到偶数。现代应用（游戏、WebRTC、VoIP）几乎不依赖此特性。

- **Hairpinning（NAT 回流）不由本项目处理**：内网主机通过外网地址访问另一个内网主机的场景，需要另行配置 NAT reflection 规则。

## 与其他 fullcone 实现的对比

| | SONiC（本项目） | xt_FULLCONENAT | nft-fullcone | bcm-fullconenat |
|---|---|---|---|---|
| 映射存储 | conntrack 自身内 | 平行哈希表 | 平行哈希表 | conntrack 期望表 |
| 读路径锁 | RCU（987 下按哈希桶加锁） | 全局自旋锁 | 全局自旋锁 | 全局 expect 锁 |
| 查找复杂度 | O(1) 哈希 | O(1) 哈希 | O(1) 哈希 | **O(N) 全表扫描** |
| 协议支持 | TCP、UDP（及内核跟踪的 UDP-Lite/SCTP/DCCP） | 仅 UDP | 仅 UDP | 仅 UDP |
| 清理机制 | 自动（conntrack 生命周期） | Workqueue GC | Workqueue GC | 期望超时 |
| Per-rule 控制 | 支持（flag bit） | 不支持 | 不支持 | 不支持 |
| EIM + EIF | 双钩子（PRE+POST）自动 | 双钩子自动 | 双钩子自动 | 依赖 helper 机制 |
| 额外内核模块 | 无（嵌入 nf_nat/nft_masq/xt_MASQUERADE） | 独立 .ko | 独立 .ko | 无（in-tree patch） |

## 与 turboacc 的关系

本项目与 [turboacc](https://github.com/coolsnowwolf/luci/tree/openwrt-23.05/applications/luci-app-turboacc) 不是同一类东西，不能直接比较。

### 定位不同

| | openwrt-sonic-fullcone | turboacc |
|---|---|---|
| 是什么 | 只做一件事：全锥形 NAT | 加速引擎合集（Flow Offloading + SFE + fullcone + BBR） |
| fullcone 实现 | SONiC 内核补丁（Broadcom 出品，生产级） | Chion/llccd 的 xt_FULLCONENAT + bcm-fullconenat + nft-fullcone（社区实现） |

### fullcone 单项对比

详细技术指标参见上方[与其他 fullcone 实现的对比](#与其他-fullcone-实现的对比)表格，以下为与 turboacc 的补充对比：

| | sonic-fullcone（本项目） | turboacc 的 fullcone |
|---|---|---|
| 粒度控制 | per-zone、per-proto | 全局开关 |
| fw3 支持 | 嵌入 xt_MASQUERADE.ko | 独立 xt_FULLCONENAT.ko 或嵌入 nf_nat_masquerade（bcm） |
| fw4 支持 | 嵌入 nft_masq.ko | 独立 nft_fullcone.ko |
| LuCI | 集成到原生防火墙页面 | 独立 luci-app-turboacc 页面 |
| 代码来源 | SONiC 官方内核补丁 | 社区多个项目拼接 |

### 各自的优势

**sonic-fullcone 更好的地方**：
- 内核实现更优雅（映射就存在 conntrack 里、按哈希桶加锁、无独立模块、支持 TCP）
- 代码血统清晰（SONiC 官方，Broadcom 工程师编写）
- 功能纯粹，维护简单

**turboacc 更好的地方**：
- 不只是 fullcone——还有 Flow Offloading、SFE、BBR 等加速功能
- 社区生态成熟，用户基数大，实战验证充分
- Chion 的 xt_FULLCONENAT 虽然只支持 UDP，但 UDP 覆盖了 99% 的 fullcone 需求场景，且经过多年生产验证

### 结论

- **只要 fullcone** → sonic-fullcone 更好（架构更优，功能更完整）
- **要加速全家桶** → turboacc（fullcone 只是它的一个子功能）
- **两个可以共存吗** → 不建议，fullcone 内核补丁会冲突

对大多数家用路由器用户来说，turboacc 的 Chion fullcone（UDP only）已经够用——游戏、P2P、VoIP 全是 UDP。sonic-fullcone 的 TCP 支持和按桶加锁在高并发场景更有价值，家用感知不明显。

不过在当前 **fw4 (nftables)** 环境下，turboacc 的其他组件并非不可替代：

- **Flow Offloading**：OpenWrt 原生自带，`menuconfig` 勾选 `kmod-nft-offload` 即可
- **BBR**：内核自带，`menuconfig` 选择 `TCP_CONG_BBR` 即可，或 sysctl 一行命令
- **SFE**：依赖 iptables conntrack chain events 接口，与 fw4/nftables 不兼容，仅 fw3 可用

所以对 fw4 用户来说，turboacc 的独有价值主要就是 fullcone。turboacc 的真正价值在 **fw3 老固件**上：SFE + fullcone + 统一 LuCI 面板，这套组合在 fw3 环境下确实方便。但 OpenWrt 23.05+ 默认 fw4 后，这个价值在收窄。

## 文件说明

```
kernel/
  984-add-sonic-fullcone-support.patch      # nf_nat_core.c：SONiC 原版 + 已提交上游的修复（3-tuple 哈希表、EIM/EIF）
  985-add-sonic-fullcone-to-ipt.patch       # xt_MASQUERADE.c：iptables FULLCONE target (PRE+POST)
  986-add-sonic-fullcone-to-nft.patch       # nft_masq.c：nftables fullcone 表达式 (PRE+POST，ip/ip6/inet 表)
  987-sonic-fullcone-fix-*.patch            # nf_nat_core.c：本项目的修复（并发竞争、无端口范围、误丢包）

patches/
  iptables/901-sonic-fullcone.patch         # libxt_NAT.c：注册 FULLCONE 用户态 target
  libnftnl/001-libnftnl-*.patch             # fullcone 表达式序列化
  nftables/002-nftables-*.patch             # nft CLI "fullcone" 关键字
  nftables/003-nftables-*.patch             # fullcone 的 JSON 输入、nft -o 优化器
  luci-app-firewall/001-add-*.patch         # LuCI Web 界面集成

translations/
  zh_Hans.po                                # 中文翻译（安装时追加到 po 文件）

firewall/
  firewall3/001-sonic-fullcone.patch        # fw3：per-zone, per-proto, EIM+EIF
  firewall4/001-sonic-fullcone.patch        # fw4：per-zone, per-proto, EIM+EIF
```

## 致谢

- 内核补丁：Akhilesh Samineni (Broadcom)，来自 [sonic-net/sonic-linux-kernel](https://github.com/sonic-net/sonic-linux-kernel)
- nftables/libnftnl 表达式接口：Syrone Wong (fullcone-nat-nftables)
- OpenWrt 集成：openwrt-sonic-fullcone contributors
- 锁、并发和 fw4 地址限制等问题，最早是对照 [VIKINGYFY/immortalwrt](https://github.com/VIKINGYFY/immortalwrt) 和 [Canary233/openwrt-sonic-fullcone](https://github.com/Canary233/openwrt-sonic-fullcone) 的改动发现的
- fw3 探测 `FULLCONE` target、按协议生成带端口范围的规则，`fullcone` 表达式支持 ip/ip6 表，以及 `ipt_FULLCONE` 模块别名，参考了 VIKINGYFY 的实现

## 许可证

内核补丁：GPL-2.0（遵循 Linux 内核许可证）
用户态补丁：GPL-2.0
