# openwrt-sonic-fullcone

[简体中文](./README.md) | English

SONiC-style Full Cone NAT for OpenWrt, with per-zone and per-protocol granularity.

## What is this

A port of the [SONiC fullcone NAT kernel patch](https://github.com/sonic-net/sonic-linux-kernel/blob/master/patches-sonic/Support-for-fullcone-nat.patch) (Akhilesh Samineni / Broadcom) to OpenWrt, bundled with the firewall and LuCI integration needed for fine-grained control.

### How it works

The kernel patch adds a second hash table inside conntrack (`nat_by_manip_src`), keyed by the **post-translation 3-tuple** (protocol, source IP, source port):

- **SNAT direction**: 3-tuple uniqueness — a (proto, src_ip, src_port) belongs to one internal endpoint, and every connection of that endpoint reuses it. This gives Endpoint-Independent Mapping (EIM). The SONiC patch checks the 3-tuple only when the rule carries a port range; the rules fw4 and fw3 generate do (except fw3's UDP-Lite rule), see [The port range on SNAT rules](#the-port-range-on-snat-rules).
- **DNAT direction**: reverse lookup — inbound packets find the original internal host through the hash table, providing Endpoint-Independent Filtering (EIF). As in SONiC, the kernel does not tell mappings of fullcone rules from those of ordinary masquerading; fw4 and fw3 only do the reverse lookup for the protocols they put under fullcone.
- **Protocols**: fw4 and fw3 use fullcone for TCP and UDP, plus UDP-Lite, SCTP and DCCP when the kernel tracks them (OpenWrt does not enable that by default; kernel 6.18 dropped DCCP), and ordinary masquerading for the rest. Fullcone makes no sense for protocols without ports, such as ICMP and GRE.
- **Only fullcone rules are affected**: the fullcone flag is per-rule, not global. Mappings of ordinary NAT are recorded in the hash table as well.

### Patch layout

The kernel part keeps the SONiC patch as it is; this project's own changes are in separate patches, not mixed into SONiC's code:

| Patch | Contents |
|---|---|
| 984 | The SONiC patch (forward-ported to 6.6+), with the two fixes submitted to SONiC upstream: the direction of replies to connections opened from outside ([sonic-net/sonic-linux-kernel#649](https://github.com/sonic-net/sonic-linux-kernel/pull/649)), and locks for `nat_by_manip_src` ([sonic-net/sonic-linux-kernel#653](https://github.com/sonic-net/sonic-linux-kernel/pull/653)) |
| 985, 986 | This project's iptables `FULLCONE` target and nftables `fullcone` expression, built into `xt_MASQUERADE.ko` and `nft_masq.ko`. The `fullcone` expression works in ip, ip6 and inet tables |
| 987 | This project's fixes on top of SONiC, not submitted there: mapping races on several CPUs, the 3-tuple check for rules without a port range, and packets of ordinary NAT being dropped (ESP through `masquerade fully-random`, for example). Without this file, the kernel part is SONiC plus the upstream fixes |

The userspace nftables patch follows the same rule: 002 is kept as it is, and the fixes for crashes in JSON input and `nft -o` are in 003.

### Supported kernel versions

6.6, 6.12, 6.18 (all three share the same patch — the relevant `nf_nat_core.c` layout is identical across them).

## Installation

### Prerequisites

- An OpenWrt source tree (master / 25.12 / 24.10)
- Kernel 6.6 / 6.12 / 6.18
- `git` and `curl` available on the build host

> 23.05 is no longer supported: its iptables 1.8.8 has no `extensions/libxt_NAT.c` (added in 1.8.9), and two hunks of the firewall4 patch no longer apply.

### One-shot install

From the root of your OpenWrt source tree:

```bash
# Update feeds first (required for the LuCI patch)
./scripts/feeds update -a
./scripts/feeds install -a

# Apply all patches
curl -sSL https://raw.githubusercontent.com/mufeng05/openwrt-sonic-fullcone/master/add_sonic_fullcone.sh | bash

# Build
make menuconfig   # No extra options needed — fullcone is compiled into the existing nft_masq.ko / xt_MASQUERADE.ko
make -j$(nproc)
```

The script clones the repo, detects the kernel version, copies the patches into the right places, and cleans up temporary files when done.

> **Note**: besides copying patches, the script also adds `PKG_FIXUP:=autoreconf` to `package/libs/libnftnl/Makefile`.
> The libnftnl patch touches `src/Makefile.am`, which makes automake try to regenerate `Makefile.in` using the exact version that produced the tarball (`automake-1.17` for libnftnl 1.3.1). That version is usually absent from the build environment, so the build fails with `automake-1.17: command not found`.
> This edits the OpenWrt source tree itself, so a `git pull` of the tree reverts it — just **re-run this script** afterwards. If the build already failed once, also run `make package/libs/libnftnl/clean` before rebuilding.

### Uninstall

Remove the patch files and rebuild:

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

Note: the `libnftnl` and `nftables` patches (which add the `fullcone` keyword) can stay — they do not affect normal behavior.

## Configuration

### Configuration logic

`defaults.fullcone` is the **global master switch**:

- **Off** (default): all fullcone functionality is disabled, settings inside zones have no effect.
- **On**: the feature is available, but each zone still has to **opt in** with its own fullcone checkbox.

This mirrors how OpenWrt's `flow_offloading` works. On start, fw4 and fw3 both check that the kernel supports fullcone (fw3 looks for `FULLCONE` in `/proc/net/ip_tables_targets`); if it does not, they warn, turn fullcone off and only masquerade.

### Web UI (LuCI)

The LuCI patch wires the fullcone options directly into OpenWrt's native firewall configuration pages — no extra LuCI app needed.

**Global settings** (Network → Firewall → General Settings):
- "Fullcone NAT" checkbox — global master switch.

![Global settings](img/luci-global-settings.png)

**Zone editing** (Network → Firewall → Zones → Edit):
- **General tab**: "Fullcone NAT" checkbox (only shown when masquerading is enabled), visible both in the zone list and the edit dialog.
- **Advanced tab**: "Fullcone protocols" multi-select (TCP, UDP, UDP-Lite, SCTP, DCCP) — restrict by protocol.

![Zone editing](img/luci-zone-edit.png)

### UCI command line

```bash
# 1. Turn on the global master switch (required)
uci set firewall.@defaults[0].fullcone='1'

# 2. Enable fullcone on the wan zone
uci set firewall.@zone[1].fullcone='1'

# 3. Optional: limit to UDP only (recommended for gaming / P2P)
uci add_list firewall.@zone[1].fullcone_proto='udp'

# Apply
uci commit firewall
/etc/init.d/firewall restart

# Verify
nft list ruleset | grep fullcone
```

### UCI config example

**Basic**: fullcone on all zones, all protocols

```
config defaults
    option fullcone '1'           # global master switch

config zone
    option name 'wan'
    option masq '1'
    option fullcone '1'           # enable fullcone for this zone
```

Generated nftables rules:

```nft
chain srcnat_wan {
    meta nfproto ipv4 meta l4proto { tcp, udp, udplite, sctp, dccp } fullcone to :1024-65535
    meta nfproto ipv4 masquerade                 # other protocols
}
chain dstnat_wan {
    meta nfproto ipv4 meta l4proto { tcp, udp, udplite, sctp, dccp } fullcone
}
```

On the device, `nft list ruleset` shows UDP-Lite, SCTP and DCCP as protocol numbers, for example `meta l4proto { tcp, udp, 33, 132, 136 }`: OpenWrt's musl does not know these protocol names, and the rules are the same. With fw3, `iptables -t nat -S` likewise shows DCCP as 33.

**UDP only**:

```
config zone
    option name 'wan'
    option masq '1'
    option fullcone '1'
    list fullcone_proto 'udp'     # only UDP gets fullcone, the rest goes through standard masquerade
```

Generated nftables rules:

```nft
chain srcnat_wan {
    meta nfproto ipv4 meta l4proto udp fullcone to :1024-65535  # UDP → fullcone
    meta nfproto ipv4 masquerade                                # other → standard masquerade
}
chain dstnat_wan {
    meta nfproto ipv4 meta l4proto udp fullcone                 # UDP-only reverse mapping
}
```

When the zone sets `masq_src` / `masq_dest`, the fullcone rule takes the same address restrictions and is followed by a masquerade rule with the same restrictions for the other protocols. Traffic outside the restrictions gets neither fullcone nor masquerading, as without fullcone.

`fullcone_proto` accepts `tcp`, `udp`, `udplite`, `sctp` and `dccp` (and `tcpudp`). Other values are ignored with a warning (`fw4 check` shows it for fw4; fw3 prints it on start); if none of the listed protocols applies, the zone gets no fullcone.

### Advanced: hand-written nftables rules

For more complex match conditions, drop a file into `/etc/nftables.d/`:

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

Notes:
- When using custom rules, disable the fw4 fullcone option for that zone to avoid conflicts.
- The `fullcone` expression can be used in `inet`, `ip` and `ip6` tables.
- **The SNAT side must carry a port range** (`to :1024-65535`); see the next section. The DNAT side needs none.
- The kernel does not tell mappings of fullcone rules from those of ordinary masquerading, so the fullcone DNAT rule sends inbound packets to any existing mapping on the address. Do not mix the two on one protocol, or mappings of ordinary masquerading are open to unknown senders too.

### The port range on SNAT rules

The SNAT rules fw4 and fw3 generate carry the port range 1024-65535 (`to :1024-65535` in nft, `--to-ports 1024-65535` in iptables). Both only accept a port range after a match on a protocol, so the fullcone rule first matches the protocols with ports and leaves the rest to the masquerade rule after it. DNAT carries no range, as in SONiC.

The range is not only there to stay off low ports: the SONiC patch checks the 3-tuple only when the rule carries a port range, and without one a translated port only gets the ordinary 5-tuple check. Two inside hosts using the same source port towards different destinations then get the same external port, and unknown senders reach only one of them. SONiC itself never takes that path: its [natmgr](https://github.com/sonic-net/sonic-swss/blob/master/cfgmgr/natmgr.cpp) only adds `--fullcone` when a port range is configured. Patch 987 checks the 3-tuple for rules without a range as well. fw3's UDP-Lite rule has none: iptables takes no `--to-ports` for UDP-Lite rules.

**The range starts at 1024**, so an inside host using a source port below 1024 is mapped above it. That is deliberate: full cone forwards every inbound packet sent to a mapped external port to the inside host, so if a LAN client goes out from source port 500 (IPsec IKE) while the router itself runs IPsec on the WAN, that mapping takes the router's port 500 (reasoned from the mechanism, not tested). Router services above 1024 (WireGuard's 51820, for example) are not protected by the range.

## Verification

After flashing the firmware:

```bash
# 1. Confirm the kernel supports fullcone
echo 'add table ip test; add chain ip test t { fullcone; }' | nft -c -f -   # fw4: no error means it does
grep -x FULLCONE /proc/net/ip_tables_targets                              # fw3: prints FULLCONE if it does

# 2. Check that rules were generated (depends on UCI configuration)
nft list ruleset | grep fullcone                            # fw4
iptables -t nat -S | grep FULLCONE                          # fw3

# 3. Test NAT type from a LAN host
pystun3   # or NatTypeTester on Windows
# Expected result: "Full Cone"

# 4. Test a TCP connection opened from outside (run Natter on a LAN host)
python3 natter.py   # no arguments: test mode, prints Please check [ http://PUBLIC_IP:PORT ]
# "WAN > PUBLIC_IP:PORT" should read [ OPEN ] ([ UNKNOWN ] if the check service is unreachable).
# More direct: open that address from outside the LAN (a phone with Wi-Fi off) and expect "It works!"
# Then, on the router, find the outside device's entry (its first src= is that device):
grep "dport=PORT " /proc/net/nf_conntrack
# Expect ESTABLISHED with [ASSURED]; SYN_SENT with [UNREPLIED] means replies were not NATed
```

Step 3 only proves that packets sent from outside get in: the STUN server answers from a different address, and the client reports Full Cone as soon as that answer arrives, without replying. Whether the LAN host's replies to a connection opened from outside are translated is what step 4 checks. TCP needs a three-way handshake, so when this path is broken the connection never comes up, and a capture on the WAN shows a SYN+ACK leaving with the LAN source address; that is what [#6](https://github.com/mufeng05/openwrt-sonic-fullcone/issues/6) ran into. Step 4 uses [Natter](https://github.com/MikeWang000000/Natter) and needs a public IP on the router's WAN, or a full-cone NAT upstream.

Note: connections that already exist keep the NAT behaviour they started with; `firewall restart` does not change them. Rebooting the router before testing is the safe choice. On platforms that hand connections to hardware forwarding, such as NSS/ECM, flushing conntrack does not remove connections that are already accelerated.

## Known limitations

### fw3 (iptables)

- **fw3 only generates IPv4 NAT rules**: in firewall3's `zones.c`, `print_zone_rule` checks `if (zone->masq && handle->family == FW3_FAMILY_V4)` inside the `FW3_TABLE_NAT` branch, and IPv6 takes a different path. As a result, this project's fw3 patch only emits IPv4 fullcone rules.

  **Adding IPv6 fullcone manually**: kernel-side patches 984/985 and the userspace 901 patch are protocol-family agnostic — the `FULLCONE` target is registered for both IPv4 and IPv6. Add the rules in `/etc/firewall.user` (what the port range is for is explained in [The port range on SNAT rules](#the-port-range-on-snat-rules)):

  ```bash
  ip6tables -t nat -A POSTROUTING -o wan -p tcp -j FULLCONE --to-ports 1024-65535
  ip6tables -t nat -A POSTROUTING -o wan -p udp -j FULLCONE --to-ports 1024-65535
  ip6tables -t nat -A POSTROUTING -o wan -j MASQUERADE
  ip6tables -t nat -A PREROUTING  -i wan -p tcp -j FULLCONE
  ip6tables -t nat -A PREROUTING  -i wan -p udp -j FULLCONE
  ```

- **fw3's UDP-Lite rule carries no port range**: iptables only accepts `--to-ports` on TCP, UDP, SCTP, DCCP and ICMP rules, so the FULLCONE rule for UDP-Lite has no range. Without patch 987, 3-tuple uniqueness is not enforced for UDP-Lite (see [The port range on SNAT rules](#the-port-range-on-snat-rules)). With 987 it is, but a connection from a source port below 1024 keeps that port, or gets another one below 1024 if it is taken.

### General

- **Port parity preservation (RFC 4787 REQ-3b) is not implemented**: fullcone port allocation does not guarantee odd-to-odd / even-to-even mapping. Modern applications (gaming, WebRTC, VoIP) almost never depend on this.

- **Hairpinning (NAT loopback) is out of scope**: a LAN host reaching another LAN host through the WAN address requires a separate NAT reflection rule.

## Comparison with other fullcone implementations

| | SONiC (this project) | xt_FULLCONENAT | nft-fullcone | bcm-fullconenat |
|---|---|---|---|---|
| Mapping storage | Inside conntrack itself | Parallel hash table | Parallel hash table | conntrack expectation table |
| Read-path locking | RCU (per-hash-bucket spinlock with 987) | Global spinlock | Global spinlock | Global expect lock |
| Lookup complexity | O(1) hash | O(1) hash | O(1) hash | **O(N) full scan** |
| Protocol coverage | TCP, UDP (and UDP-Lite/SCTP/DCCP when tracked) | UDP only | UDP only | UDP only |
| Cleanup | Automatic (conntrack lifecycle) | Workqueue GC | Workqueue GC | Expectation timeout |
| Per-rule control | Yes (flag bit) | No | No | No |
| EIM + EIF | Dual hooks (PRE+POST) automatically | Dual hooks automatically | Dual hooks automatically | Relies on the helper mechanism |
| Extra kernel module | None (embedded in nf_nat / nft_masq / xt_MASQUERADE) | Standalone .ko | Standalone .ko | None (in-tree patch) |

## Relationship with turboacc

This project and [turboacc](https://github.com/coolsnowwolf/luci/tree/openwrt-23.05/applications/luci-app-turboacc) are not the same kind of thing and cannot be directly compared.

### Different scope

| | openwrt-sonic-fullcone | turboacc |
|---|---|---|
| What it is | Does one thing: full cone NAT | A bundle of acceleration engines (Flow Offloading + SFE + fullcone + BBR) |
| Fullcone implementation | SONiC kernel patch (Broadcom, production-grade) | Chion / llccd's xt_FULLCONENAT + bcm-fullconenat + nft-fullcone (community implementations) |

### Side-by-side on fullcone

For detailed technical comparison see the [fullcone implementations table](#comparison-with-other-fullcone-implementations) above; here is the diff against turboacc specifically:

| | sonic-fullcone (this project) | turboacc's fullcone |
|---|---|---|
| Granularity | per-zone, per-proto | global toggle |
| fw3 support | Embedded in xt_MASQUERADE.ko | Standalone xt_FULLCONENAT.ko, or embedded in nf_nat_masquerade (bcm) |
| fw4 support | Embedded in nft_masq.ko | Standalone nft_fullcone.ko |
| LuCI | Integrated into the native firewall pages | Separate luci-app-turboacc page |
| Code provenance | Official SONiC kernel patch | Patchwork from several community projects |

### Where each one wins

**sonic-fullcone is better at**:
- Cleaner kernel implementation (mappings live in conntrack, per-bucket locking, no separate module, TCP included)
- Clear code provenance (official SONiC, written by Broadcom engineers)
- Focused feature set, easier to maintain

**turboacc is better at**:
- It's not just fullcone — also Flow Offloading, SFE, BBR, and other acceleration features
- Mature community ecosystem, large user base, well battle-tested
- Chion's xt_FULLCONENAT only supports UDP, but UDP covers ~99% of fullcone use cases and has years of production usage behind it

### Bottom line

- **Fullcone only** → sonic-fullcone (better architecture, more complete coverage)
- **Full acceleration bundle** → turboacc (fullcone is just one of its sub-features)
- **Can the two coexist?** → Not recommended; the fullcone kernel patches conflict.

For most home router users, turboacc's Chion fullcone (UDP only) is enough — gaming, P2P, and VoIP are all UDP. The TCP support and per-bucket locking of sonic-fullcone matter more under high concurrency, which home users rarely notice.

That said, in today's **fw4 (nftables)** environment, the rest of turboacc is no longer irreplaceable:

- **Flow Offloading**: shipped with OpenWrt — just enable `kmod-nft-offload` in `menuconfig`.
- **BBR**: in the kernel — pick `TCP_CONG_BBR` in `menuconfig`, or set it via a one-line `sysctl`.
- **SFE**: depends on the iptables conntrack chain events interface, incompatible with fw4 / nftables, only usable on fw3.

So for fw4 users, turboacc's unique value is essentially fullcone. Its real value lives on **older fw3 firmware**: SFE + fullcone + a unified LuCI panel is genuinely convenient there. But since OpenWrt 23.05+ defaults to fw4, that value keeps shrinking.

## File layout

```
kernel/
  984-add-sonic-fullcone-support.patch      # nf_nat_core.c: SONiC patch + fixes submitted upstream (3-tuple hash table, EIM/EIF)
  985-add-sonic-fullcone-to-ipt.patch       # xt_MASQUERADE.c: iptables FULLCONE target (PRE+POST)
  986-add-sonic-fullcone-to-nft.patch       # nft_masq.c: nftables fullcone expression (PRE+POST, ip/ip6/inet tables)
  987-sonic-fullcone-fix-*.patch            # nf_nat_core.c: this project's fixes (races, no port range, dropped packets)

patches/
  iptables/901-sonic-fullcone.patch         # libxt_NAT.c: register the FULLCONE userspace target
  libnftnl/001-libnftnl-*.patch             # fullcone expression serialization
  nftables/002-nftables-*.patch             # nft CLI "fullcone" keyword
  nftables/003-nftables-*.patch             # fullcone in JSON input and the nft -o optimizer
  luci-app-firewall/001-add-*.patch         # LuCI Web UI integration

translations/
  zh_Hans.po                                # Simplified Chinese translation (appended to po files at install time)

firewall/
  firewall3/001-sonic-fullcone.patch        # fw3: per-zone, per-proto, EIM+EIF
  firewall4/001-sonic-fullcone.patch        # fw4: per-zone, per-proto, EIM+EIF
```

## Acknowledgements

- Kernel patch: Akhilesh Samineni (Broadcom), from [sonic-net/sonic-linux-kernel](https://github.com/sonic-net/sonic-linux-kernel)
- nftables / libnftnl expression interface: Syrone Wong (fullcone-nat-nftables)
- OpenWrt integration: openwrt-sonic-fullcone contributors
- The locking, concurrency and fw4 address restriction issues were first found by comparing with the changes in [VIKINGYFY/immortalwrt](https://github.com/VIKINGYFY/immortalwrt) and [Canary233/openwrt-sonic-fullcone](https://github.com/Canary233/openwrt-sonic-fullcone)
- fw3's `FULLCONE` probe and per-protocol rules with a port range, the `fullcone` expression in ip/ip6 tables, and the `ipt_FULLCONE` module alias follow VIKINGYFY's version

## License

Kernel patches: GPL-2.0 (in line with the Linux kernel license)
Userspace patches: GPL-2.0
