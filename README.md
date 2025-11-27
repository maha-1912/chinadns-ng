## 簡介

[ChinaDNS](https://github.com/shadowsocks/ChinaDNS) 的個人重構版本，功能簡述：

- 基於 epoll、netlink(ipset/nftset) 實現，性能更強。
- 完整支援 IPv4 和 IPv6 協定，相容 EDNS 請求和響應。
- 手動指定牆內 DNS 和可信 DNS，而非自動識別，更加可控。
- 修覆原版對保留位址的處理問題，去除過時特性，只留核心功能。
- 修復原版對可信 DNS 先於牆內 DNS 返回而導致判斷失效的問題。
- 支援 `chnlist/gfwlist` 網域列表，並深度優化了性能以及內存佔用。
- 支援純網域分流：要麽走mainland china上遊，要麽走trust上遊，不進行IP測試。
- 可動態收集網域的解析結果IP至`ipset/nftset`，輔助各類代理/分流場景。
- 支援`nftables set`，並針對 IP add 操作進行了性能優化，避免操作延遲。
- 更加細致的 no-ipv6(AAAA) 控制，可根據域名類型，IP測試結果進行過濾。
- DNS 緩存、stale 緩存模式、緩存預刷新、緩存忽略清單（不緩存的網域）。
- 支持 tag:none 網域的判定結果緩存，避免重覆請求和判定，減少DNS泄露。
- 除默認的 china、trust 組外，還有 6 個自定義組（上遊DNS、ipset/nftset）。

---

對於常規的使用模式，大致原理和流程可總結為：

- 兩組DNS上遊：china組(大陸DNS)、trust組(國外DNS)。

- 兩個域名列表：chnlist.txt(大陸域名)、gfwlist.txt(受污染域名)。

- chnlist.txt域名(tag:chn域名)，轉發給china組，保證大陸域名不會被解析到國外，對大陸域名cdn友好。

- gfwlist.txt域名(tag:gfw域名)，轉發給trust組，trust需返回未受污染的結果，比如走代理，具體方式不限。

- 其他域名(tag:none域名)，同時轉發給china組和trust組，如果china組解析結果(A/AAAA)是大陸ip，則采納china組，否則采納trust組。是否為大陸ip的核心依據，就是測試ip是否在`ipset-name4/6`指定的地址集合中。

- 若啟用了tag:none域名的判決緩存，則同一域名的後續請求只會轉發給特定的上遊組（china組或trust組，具體取決於之前的ip測試結果），建議啟用此功能，避免重覆請求、判定，減少DNS泄露。

- 如果使用純域名分流模式，則不存在tag:none域名，因此要麽走china組，要麽走trust組，可避免dns泄露問題。

- 若啟用`--add-tagchn-ip`，則tag:chn域名的解析結果IP會被動態添加到指定的ipset/nftset，配合chnroute透明代理分流時，可用於實現大陸域名必走直連，使dns分流與ip分流一致；類似 dnsmasq 的 ipset/nftset 功能。

- 若啟用`--add-taggfw-ip`，則tag:gfw域名的解析結果IP會被動態添加到指定的ipset/nftset，可用來實現gfwlist透明代理分流；也可配合chnroute透明代理分流，用來收集黑名單域名的IP，用於iptables/nftables操作，比如確保黑名單域名必走代理，即使某些黑名單域名的IP是大陸IP。

> chinadns-ng 根據域名 tag 來執行不同邏輯，包括 ipset/nftset 的邏輯（test、add），見 [原理](#tagchntaggfwtagnone-%E6%98%AF%E6%8C%87%E4%BB%80%E4%B9%88)。

## 編譯

> 請前往 [releases](https://github.com/zfl9/chinadns-ng/releases) 頁面下載可執行文件，添加可執行權限，放到 PATH 路徑下（如 `/usr/local/bin/`）。

<details><summary><b>點我展開編譯說明</b></summary><p>

---

**zig 工具鏈**

- 從 2024.03.07 版本起，程序使用 Zig + C 語言編寫，`zig` 是唯一需要的工具鏈。
- 從 [ziglang.org](https://ziglang.org/download/) 下載 zig 0.10.1，請根據當前（編譯）主機的架構來選擇合適的版本。
- 將解壓後的目錄加入 PATH 環境變量，執行 `zig version`，檢查是否有輸出 `0.10.1`。
- 注意，目前必須使用 zig 0.10.1 版本，因為 0.11、master 版本暫時不支持 async 特性。

---

如果要構建 DoT 支持，請帶上 `-Dwolfssl` 參數，構建過程需要以下依賴：
- `wget` 或 `curl` 用於下載 wolfssl 源碼包；`tar` 用於解壓縮
- `autoconf`、`automake`、`libtool`、`make` 用於構建 wolfssl

針對 x86_64(v3/v4)、aarch64 的 wolfssl 構建已默認啟用硬件指令加速，若目標硬件(CPU)不支持相關指令（部分樹莓派閹割了 aes 相關指令），請指定 `-Dwolfssl-noasm` 選項，避免運行 chinadns-ng 時出現 SIGILL 非法指令異常。

---

如果遇到編譯錯誤，請先執行 `zig build clean-all`，然後重新執行相關構建命令。

可執行文件在 `./zig-out/bin` 目錄，將文件安裝（覆制）到目標主機 PATH 路徑下即可。

---

```bash
git clone https://github.com/zfl9/chinadns-ng
cd chinadns-ng

# 本機 (若構建失敗，請手動指定"-Dtarget"和"-Dcpu")
zig build # 鏈接到glibc
zig build -Dtarget=native-native-musl # 靜態鏈接到musl

# x86
zig build -Dtarget=i386-linux-musl -Dcpu=i686
zig build -Dtarget=i386-linux-musl -Dcpu=pentium4

# x86_64
zig build -Dtarget=x86_64-linux-musl -Dcpu=x86_64 # v1
zig build -Dtarget=x86_64-linux-musl -Dcpu=x86_64_v2
zig build -Dtarget=x86_64-linux-musl -Dcpu=x86_64_v3
zig build -Dtarget=x86_64-linux-musl -Dcpu=x86_64_v4

# arm
zig build -Dtarget=arm-linux-musleabi -Dcpu=generic+v5t+soft_float
zig build -Dtarget=arm-linux-musleabi -Dcpu=generic+v5te+soft_float
zig build -Dtarget=arm-linux-musleabi -Dcpu=generic+v6+soft_float
zig build -Dtarget=arm-linux-musleabi -Dcpu=generic+v6t2+soft_float
zig build -Dtarget=arm-linux-musleabi -Dcpu=generic+v7a # soft_float
zig build -Dtarget=arm-linux-musleabihf -Dcpu=generic+v7a # hard_float

# aarch64
zig build -Dtarget=aarch64-linux-musl -Dcpu=generic+v8a
zig build -Dtarget=aarch64-linux-musl -Dcpu=generic+v9a

# mips + soft_float
# 請先閱讀 https://www.zfl9.com/zig-mips.html
ARCH=mips32 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH+soft_float
ARCH=mips32r2 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH+soft_float
ARCH=mips32r3 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH+soft_float
ARCH=mips32r5 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH+soft_float

# mipsel + soft_float
# 請先閱讀 https://www.zfl9.com/zig-mips.html
ARCH=mips32 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH+soft_float
ARCH=mips32r2 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH+soft_float
ARCH=mips32r3 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH+soft_float
ARCH=mips32r5 && MIPS_M_ARCH=$ARCH MIPS_SOFT_FP=1 zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH+soft_float

# mips + hard_float
# 請先閱讀 https://www.zfl9.com/zig-mips.html
ARCH=mips32 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH
ARCH=mips32r2 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH
ARCH=mips32r3 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH
ARCH=mips32r5 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mips-linux-musl -Dcpu=$ARCH

# mipsel + hard_float
# 請先閱讀 https://www.zfl9.com/zig-mips.html
ARCH=mips32 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH
ARCH=mips32r2 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH
ARCH=mips32r3 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH
ARCH=mips32r5 && MIPS_M_ARCH=$ARCH zig build -Dtarget=mipsel-linux-musl -Dcpu=$ARCH

# mips64/mips64el 請前往 releases 頁面下載預編譯的可執行文件
# 如果想自己編譯，請先前往 zig 根目錄，按順序 apply 這兩個補丁
# - https://github.com/ziglang/zig/pull/14541.patch
# - https://github.com/ziglang/zig/pull/14556.patch

# riscv64
zig build -Dtarget=riscv64-linux-musl
```

</p></details>

## Docker

因為要訪問內核的 ipset/nftset，docker run 時請帶上 `--privileged` 參數。

請前往 [releases](https://github.com/zfl9/chinadns-ng/releases) 頁面下載可執行文件（無依賴），cp 至目標容器，運行即可。

## OpenWrt

- 某些科學上網插件附帶了 chinadns-ng，請查看對應插件的文檔。
- https://github.com/pexcn/openwrt-chinadns-ng (未適配 2.0 的新功能)。

## 配置示例

- chinadns-ng 通常與 iptables/nftables 透明代理一起使用。
- chinadns-ng 也可作為單純的 DNS 轉發器（如 UDP->TCP）使用。
- 下面列舉的 3 種分流模式配置其實都是 [zfl9/ss-tproxy](https://github.com/zfl9/ss-tproxy) 中的相關用例。
- 程序不會自動創建 ipset/nftset 集合；如需 ip test/add，請先導入/創建 set。
- 所有帶 ipset 字眼的配置都支持 nftset，如需使用 nftset，請閱讀 [nftset 說明](#ipsetnftset-相關說明)。

---

### chnroute 分流

- chnlist.txt (tag:chn) 走國內上遊，將 IP 收集至 `chnip,chnip6` ipset（可選）
- gfwlist.txt (tag:gfw) 走可信上遊，將 IP 收集至 `gfwip,gfwip6` ipset（可選）
- 其他域名 (tag:none) 同時走國內和可信上遊，根據 IP 測試結果決定最終響應

<details><summary><b>點我展開</b></summary><p>

```shell
# 監聽地址和端口
bind-addr 0.0.0.0
bind-port 53

# 國內上遊、可信上遊
china-dns 223.5.5.5
trust-dns tcp://8.8.8.8

# 域名列表，用於分流
chnlist-file /etc/chinadns/chnlist.txt
gfwlist-file /etc/chinadns/gfwlist.txt
# chnlist-first

# 收集 tag:chn、tag:gfw 域名的 IP (可選)
add-tagchn-ip chnip,chnip6
add-taggfw-ip gfwip,gfwip6

# 測試 tag:none 域名的 IP (針對國內上遊)
ipset-name4 chnroute
ipset-name6 chnroute6

# dns 緩存
cache 4096
cache-stale 86400
cache-refresh 20

# verdict 緩存 (用於 tag:none 域名)
verdict-cache 4096

# 詳細日志
# verbose
```

</p></details>

---

### gfwlist 分流

- gfwlist.txt (tag:gfw) 走可信上遊，將 IP 收集至 `gfwip,gfwip6` ipset（可選）
- 其他域名 (tag:chn) 走國內上遊，不需要收集 IP（未指定 add-tagchn-ip）

<details><summary><b>點我展開</b></summary><p>

```shell
# 監聽地址和端口
bind-addr 0.0.0.0
bind-port 53

# 國內上遊、可信上遊
china-dns 223.5.5.5
trust-dns tcp://8.8.8.8

# 域名列表，用於分流
# 未被 gfwlist.txt 匹配的歸為 tag:chn
gfwlist-file /etc/chinadns/gfwlist.txt
default-tag chn

# 收集 tag:gfw 域名的 IP (可選)
add-taggfw-ip gfwip,gfwip6

# dns 緩存
cache 4096
cache-stale 86400
cache-refresh 20

# 詳細日志
# verbose
```

</p></details>

---

### chnlist 分流

- chnlist.txt (tag:chn) 走國內上遊，將 IP 收集至 `chnip,chnip6` ipset（可選）
- 其他域名 (tag:gfw) 走可信上遊，不需要收集 IP（未指定 add-taggfw-ip）

<details><summary><b>點我展開</b></summary><p>

```shell
# 監聽地址和端口
bind-addr 0.0.0.0
bind-port 53

# 國內上遊、可信上遊
china-dns 223.5.5.5
trust-dns tcp://8.8.8.8

# 域名列表，用於分流
# 未被 chnlist.txt 匹配的歸為 tag:gfw
chnlist-file /etc/chinadns/chnlist.txt
default-tag gfw

# 收集 tag:chn 域名的 IP (可選)
add-tagchn-ip chnip,chnip6

# dns 緩存
cache 4096
cache-stale 86400
cache-refresh 20

# 詳細日志
# verbose
```

</p></details>

---

### DNS 轉發器

- 轉發器不執行分流操作，但其他功能仍可正常使用，如 DNS 緩存、ip add、ipv6 過濾等。
- 核心在於 `-d chn`，由於未指定域名列表，因此所有查詢都是 tag:chn，從而實現單純轉發。

```bash
# 127.0.0.1:53(udp/tcp) => 1.1.1.1(udp/tcp/tls)
# 允許指定任意多個 upstream，多次給出 -c 選項即可
# 使用 -d gfw 也是一樣的，只不過將 -c 改為 -t 選項
chinadns-ng -b 127.0.0.1 -l 53 -d chn -c 1.1.1.1
chinadns-ng -b 127.0.0.1 -l 53 -d chn -c udp://1.1.1.1
chinadns-ng -b 127.0.0.1 -l 53 -d chn -c tcp://1.1.1.1
chinadns-ng -b 127.0.0.1 -l 53 -d chn -c tls://1.1.1.1
```

## 命令選項

- `-f` 短選項、`--foobar` 長選項
- `--foo <value>`：選項值是 required 的
- `--bar [value]`：選項值是 optional 的
- `--flag`：沒有選項值，即 bool/flag 選項
- 配置文件中使用長選項格式（沒有`--`），如 `verbose`

<details><summary><b>點我展開所有選項</b></summary><p>

```console
$ chinadns-ng --help
usage: chinadns-ng <options...>. the existing options are as follows:
 -C, --config <path>                  format similar to the long option
 -b, --bind-addr <ip>                 listen address, default: 127.0.0.1
 -l, --bind-port <port[@proto]>       listen port number, default: 65353
 -c, --china-dns <upstreams>          china dns server, default: <114 DNS>
 -t, --trust-dns <upstreams>          trust dns server, default: <Google DNS>
 -m, --chnlist-file <paths>           path(s) of chnlist, '-' indicate stdin
 -g, --gfwlist-file <paths>           path(s) of gfwlist, '-' indicate stdin
 -M, --chnlist-first                  match chnlist first, default gfwlist first
 -d, --default-tag <tag>              chn or gfw or <user-tag> or none(default)
 -a, --add-tagchn-ip [set4,set6]      add the ip of name-tag:chn to ipset/nftset
                                      use '--ipset-name4/6' setname if no value
 -A, --add-taggfw-ip <set4,set6>      add the ip of name-tag:gfw to ipset/nftset
 -4, --ipset-name4 <set4>             ip test for tag:none, default: chnroute
 -6, --ipset-name6 <set6>             ip test for tag:none, default: chnroute6
                                      if setname contains @, then use nftset
                                      format: family_name@table_name@set_name
 --group <name>                       define rule group: {dnl, upstream, ipset}
 --group-dnl <paths>                  domain name list for the current group
 --group-upstream <upstreams>         upstream dns server for the current group
 --group-ipset <set4,set6>            add the ip of the current group to ipset
 -N, --no-ipv6 [rules]                tag:<name>[@ip:*], ip:china, ip:non_china
                                      if no rules, then filter all AAAA queries
 --filter-qtype <qtypes>              filter queries with the given qtype (u16)
 --cache <size>                       enable dns caching, size 0 means disabled
 --cache-stale <N>                    use stale cache: expired time <= N(second)
 --cache-refresh <N>                  pre-refresh the cached data if TTL <= N(%)
 --cache-nodata-ttl <ttl>             TTL of the NODATA response, default is 60
 --cache-min-ttl <ttl>                if record.ttl < min_ttl, set ttl to min_ttl
 --cache-max-ttl <ttl>                if record.ttl > max_ttl, set ttl to max_ttl
 --cache-ignore <domain>              ignore the dns cache for this domain(suffix)
 --cache-db <path>                    dns cache persistence (from/to db file)
 --verdict-cache <size>               enable verdict caching for tag:none domains
 --verdict-cache-db <path>            verdict cache persistence (from/to db file)
 --hosts [path]                       load hosts file, default path is /etc/hosts
 --dns-rr-ip <names>=<ips>            define local resource records of type A/AAAA
 --cert-verify                        enable SSL certificate validation, default: no
 --ca-certs <path>                    CA certs path for SSL certificate validation
 --no-ipset-blacklist                 add-ip: don't enable built-in ip blacklist
                                      blacklist: 127.0.0.0/8, 0.0.0.0/8, ::1, ::
 -o, --timeout-sec <sec>              response timeout of upstream, default: 5
 -p, --repeat-times <num>             num of packets to trustdns, default:1, max:5
 -n, --noip-as-chnip                  allow no-ip reply from chinadns (tag:none)
 -f, --fair-mode                      enable fair mode (nop, only fair mode now)
 -r, --reuse-port                     enable SO_REUSEPORT, default: <disabled>
 -v, --verbose                        print the verbose log, default: <disabled>
 -V, --version                        print `chinadns-ng` version number and exit
 -h, --help                           print `chinadns-ng` help information and exit
bug report: https://github.com/zfl9/chinadns-ng. email: zfl9.com@gmail.com (Otokaze)
```

</p></details>

---

### config

- `-C/--config <path>` 指定配置文件路徑，支持多個配置文件。
  - 配置文件是一個 UTF-8 純文本文件，沒有特定的文件擴展名。
  - 格式 `optname [value]`，`optname` 是不帶 `--` 的長命令行選項名。
  - 例如 `bind-addr 127.0.0.1`、`bind-port 65353`、`noip-as-chnip`。
  - 空白行、`#`開頭的行 被忽略；不支持行尾注釋（如`verbose #foo`）。
  - 文件中可多次使用 `config <path>` 配置行來實現配置文件包含的效果。
- `-C/--config` 只是從給定文本文件讀取“命令行選項”並處理，無其他特別之處。
- `-C/--config` 和其他命令行選項可隨意混用，不可重覆的選項以最後一個為準。

### bind-addr、bind-port

- `bind-addr` 用於指定監聽地址，默認為 127.0.0.1。
  - 2023.10.28 版本起，若監聽地址為 `::`，則允許來自 IPv4/IPv6 的 DNS 查詢。
  - 2024.03.07 版本起，`bind-addr` 允許指定多次，以便監聽多個不同的 ip 地址。
- `bind-port` 用於指定監聽端口，默認為 65353。
  - 2024.03.07 版本起，將同時監聽 UDP 和 TCP 端口，之前只監聽了 UDP。
  - 2024.03.25 版本起，可以給 `bind-port` 指定要監聽的協議（UDP、TCP）：
    - `--bind-port 65353`：監聽 UDP 和 TCP（默認）。
    - `--bind-port 65353@udp`：只監聽 UDP。
    - `--bind-port 65353@tcp`：只監聽 TCP。
  - 2024.07.16 版本起，`bind-port` 允許指定多次，以便監聽多個不同的 port。

### china-dns、trust-dns

- `china-dns` 選項指定國內上遊 DNS 服務器，多個用逗號隔開。
- `trust-dns` 選項指定可信上遊 DNS 服務器，多個用逗號隔開。
  - 國內上遊默認為 `114.114.114.114`，可信上遊默認為 `8.8.8.8`。
  - 組內的多個上遊服務器是並發查詢的模式，采納最先返回的那個結果。
  - 2024.03.07 版本起，允許多次指定 `china-dns`、`trust-dns` 選項/配置。
  - 2024.03.07 版本起，每組上遊的服務器數量不受限制（之前最多兩個）。

### 上遊服務器的地址格式

- 完整格式：`proto:// host@ ip #port ?count=N ?life=N`
- 注：加空格只是為了方便閱讀和說明，實際格式中並沒有空格。
- `proto://`：可省略，查詢協議，默認為`無`。
  - `無`：UDP/TCP 上遊，根據查詢方的傳入協議來決定使用 UDP 查詢還是 TCP 查詢。
  - `udp://`：UDP 上遊。
  - `tcp://`：TCP 上遊。
  - `tls://`：DoT 上遊（需使用 wolfssl 版本）。
- `host@`：可省略，用於 DoT 上遊。
  - 提供 SSL/TLS 握手時的 SNI（服務器名稱指示）信息。
  - 啟用 SSL/TLS 證書驗證時，將檢查證書中的域名是否與之匹配。
- `ip`：不可省略，支持 IPv4 和 IPv6 地址（不需要用 `[]` 括起來）。
- `#port`：可省略，默認為所選定協議的標準端口（UDP/TCP 是 53，DoT 是 853）。
- `?count=N`：可省略，默認為 10，表示單個會話最多處理多少查詢，見 [#189](https://github.com/zfl9/chinadns-ng/issues/189)。
  - 0 表示不限制，只要上遊不主動斷開連接，對應 TCP/TLS 會話就一直存在。
- `?life=N`：可省略，默認為 10，表示單個會話最多存活多少秒，見 [#189](https://github.com/zfl9/chinadns-ng/issues/189)。
  - 0 表示不限制，只要上遊不主動斷開連接，對應 TCP/TLS 會話就一直存在。

### chnlist-file、gfwlist-file、chnlist-first

- `chnlist-file` 白名單 [域名列表文件](#域名列表)，命中的域名只走國內 DNS。
- `gfwlist-file` 黑名單 [域名列表文件](#域名列表)，命中的域名只走可信 DNS。
  - 2023.04.01 版本起，可指定多個路徑，逗號隔開，如 `-g a.txt,b.txt`。
  - 2024.03.07 版本起，可多次指定 `chnlist-file`、`gfwlist-file` 選項。
- `chnlist-first` 選項表示優先加載 chnlist，默認是優先加載 gfwlist。
  - 只有 chnlist 和 gfwlist 文件都提供時，`*-first` 才有實際意義。

### default-tag

- `default-tag` 可用於實現"純域名分流"，也可用於實現 [gfwlist分流](#chinadns-ng-也可用於-gfwlist-透明代理分流)。
- 其核心邏輯是設置 **未匹配任何列表的域名** 的`tag`，並無其他特別之處。
- 通常與`-g`或`-m`選項一起使用，比如下述例子，實現了"純域名分流"模式：
  - `-g gfwlist.txt -d chn`：gfw列表的域名走可信上遊，其他走國內上遊。
  - `-m chnlist.txt -d gfw`：chn列表的域名走國內上遊，其他走可信上遊。
- 如果想了解更多細節，建議看一下 [chinadns-ng 的核心處理流程](#tagchntaggfwtagnone-是指什麽)。

### add-tagchn-ip、add-taggfw-ip

- `add-tagchn-ip` 用於動態添加 tag:chn 域名的解析結果 ip 至 ipset/nftset 集合。
- `add-taggfw-ip` 用於動態添加 tag:gfw 域名的解析結果 ip 至 ipset/nftset 集合。
  - 參數為`ipv4集合名[,ipv6集合名]`，nftset 格式和注意事項見 [nftset 相關說明](#ipsetnftset-相關說明)。
  - 對於`add-tagchn-ip`，若未給出集合名，則使用`ipset-name4/6`的那個集合。
  - 2024.04.13 版本起，可使用特殊集合名 `null` 表示對應集合不會被使用：
    - `--add-tagchn-ip null,chnip6`：表示不需要收集 ipv4 地址
    - `--add-tagchn-ip chnip,null`：表示不需要收集 ipv6 地址
    - `--add-tagchn-ip chnip`：表示不需要收集 ipv6 地址
    - 注意：僅當 ipv6 集合為 `null` 時，才可被省略

### ipset-name4、ipset-name6

- `ipset-name4` 大陸 IPv4 地址的 ipset/nftset 集合名，默認為 `chnroute` (ipset)。
- `ipset-name6` 大陸 IPv6 地址的 ipset/nftset 集合名，默認為 `chnroute6` (ipset)。
- 這兩個集合用於 tag:none 域名，用於判定 china 上遊的解析結果是否為大陸 IP。
- 2024.04.13 版本起，也用於 `--no-ipv6` 的 `ip:china`、`ip:non_china` 規則。
- 2024.04.13 版本起，可使用特殊集合名 `null` 表示對應集合不會被使用。

### ipset/nftset 相關說明

相關配置/命令行選項：

- **ip test**：`ipset-name4`、`ipset-name6`（默認為 chnroute、chnroute6）
- **ip add**：`add-tagchn-ip`、`add-taggfw-ip`、`group-ipset`（無默認行為）
- ip add 選項的參數為`ipv4集合名[,ipv6集合名]`，`null`可作為空集合的占位符
- 注意：所有相關配置要麽使用 ipset 後端，要麽使用 nftset 後端，不允許混用
- 程序不會自動創建 ipset/nftset 集合，如果需要，請先手動導入/創建相關集合

ipset 相關說明：

- 集合名正常給出即可，如 chnroute
- res/chnroute.ipset 文件的集合名：`chnroute`
- res/chnroute6.ipset 文件的集合名：`chnroute6`

nftset 相關說明：

- 集合名的完整格式：`family名@table名@set名`
- 創建 nftset 集合時，必須帶上 `flags interval` 標志
- 支持的 family：`ip`、`ip6`、`inet`、`arp`、`bridge`、`netdev`
- res/chnroute.nftset 文件的集合名：`inet@global@chnroute`
- res/chnroute6.nftset 文件的集合名：`inet@global@chnroute6`

### group、group-*

- `group` 聲明一個自定義組（tag），參數值是組（tag）的名字。
  - 支持最多 6 個自定義組，每個組都有 3 個信息可配置，其中 ipset 可選。
  - 加載域名列表文件時，優先加載自定義組，然後加載內置組（chn、gfw）。
  - 內置組的加載順序沒有改變，依舊默認 gfw 優先，使用 `-M` 切換為 chn 優先。
  - 對於多個自定義組，按照命令行參數/配置順序，後聲明的組具有更高優先級。
  - 用例 1：將 DDNS域名 劃分出來，單獨一個組，用域名提供商的 DNS 去解析。
  - 用例 2：將 公司域名 劃分出來，單獨一個組，用公司內網專用的 DNS 去解析。
  - 2024.04.27 版本起，使用 `null` 作為 group 名時，表示過濾該組的域名查詢。
    - null 組只有 `group-dnl` 信息，查詢相關域名時，將返回 NODATA 響應消息。
- `group-dnl` 當前組的[域名列表文件](#域名列表)，多個用逗號隔開，可多次指定。
- `group-upstream` 當前組的上遊 DNS，多個用逗號隔開，可多次指定。
- `group-ipset` 當前組的 ipset/nftset (可選)，用於收集解析出的結果 IP。

以配置文件舉例：

```shell
# 聲明自定義組 "foo"
group foo
group-dnl foo.txt
group-upstream 1.1.1.1,8.8.8.8
group-ipset fooip,fooip6

# 聲明自定義組 "bar"
group bar
group-dnl bar.txt
group-upstream 192.168.1.1
# 沒有 group-ipset，表示不需要 add ip
```

### no-ipv6

- `no-ipv6` 過濾 AAAA 查詢（查詢域名的 IPv6 地址），默認不啟用。
  - 此選項可多次指定，選項參數為`過濾規則`。
    - 如果沒有選項參數，則表示過濾所有 AAAA 查詢。
    - 選項參數中可以有多條`過濾規則`，中間用逗號隔開。
    - 被`過濾規則`匹配的查詢將以 NODATA 形式進行響應。
  - `過濾規則`的完整形式為`tag:域名組@ip:測試結果`。
    - 每個域名組都有一個 AAAA 過濾器，不同域名組的過濾規則互相獨立。
    - `tag:域名組`是域名組選擇器，表示`過濾條件`將要添加到哪個域名組中。
    - `ip:測試結果`是`過濾條件`，若域名組中的查詢符合其條件，則被“過濾”。
    - 若未指定`tag:域名組@`部分，則該`過濾條件`將添加到每個域名組中。
    - 若未指定`@ip:測試結果`部分，則該域名組的所有查詢都將被“過濾”。
  - `ip:測試結果`只有以下兩種（二選一）：
    - `ip:china`：若域名解析結果為 **大陸 IPv6 地址**，則“過濾”。
    - `ip:non_china`：若域名解析結果為 **非大陸 IPv6 地址**，則“過濾”。
    - IPv6 數據庫由 `--ipset-name6` 選項提供，默認為 `chnroute6` (ipset)。
  - 列舉一些 `過濾規則`，以及對應的 AAAA 過濾效果：
    - `tag:none`：過濾 none 域名組中 所有 AAAA 查詢。
    - `ip:non_china`：過濾 所有 域名組中 解析結果為 '非大陸 IPv6' 的 AAAA 查詢。
    - `tag:none@ip:non_china`：過濾 none 域名組中 解析結果為 '非大陸 IPv6' 的 AAAA 查詢。

### filter-qtype

- `filter-qtype` 過濾給定 qtype 的查詢，多個用逗號隔開，可多次指定。
  - `--filter-qtype 64,65`：過濾 SVCB(64)、HTTPS(65) 查詢

### cache、cache-*

- `cache` 啟用 DNS 緩存，參數是緩存容量（最多緩存多少個請求的響應消息）。
- `cache-stale` 允許使用 TTL 已過期的（陳舊）緩存，參數是最大過期時長（秒）。
  - 向查詢方返回“陳舊”緩存的同時，自動在後台刷新緩存，以便稍後能使用新數據。
  - 2024.04.13 版本起，數據類型從 `u16` 改為 `u32`，以允許設置更大的過期時長。
- `cache-refresh` 若當前查詢的緩存的 TTL 不足初始值的百分之 N，則提前在後台刷新。
- `cache-nodata-ttl` 給 NODATA 響應提供默認的緩存時長，默認 60 秒，0 表示不緩存。
- `cache-min-ttl` 若響應記錄的 TTL 小於此值，則將其 TTL 修改為此值，0 表示禁用。
- `cache-max-ttl` 若響應記錄的 TTL 大於此值，則將其 TTL 修改為此值，0 表示禁用。
- `cache-ignore` 不要緩存給定的域名（後綴，最高支持 8 級），此選項可多次指定。
- `cache-db` 啟用緩存持久化，參數是 db 文件路徑（可以不預先創建）。
  - 進程啟動時，自動從 db 恢覆緩存；進程退出時，自動將緩存寫回至 db。
  - “進程退出”是指進程收到`SIGTERM/SIGINT`信號，即`kill <PID>`或`CTRL+C`。
  - “緩存寫回”可通過`SIGUSR1`信號強制觸發（未啟用持久化則寫至`/tmp/chinadns@cache.db`）。
  - 為了降低性能開銷，恢覆緩存時不進行校驗，請勿修改 db 文件，請勿跨平台共享 db 文件。
  - 有時可能需要手動清空 db 文件來丟棄舊緩存（關進程，清空文件，重新啟動），例如：
    - 更改了`cache-ignore`、域名列表（內容更改、優先級更改等）。
    - ~~需要重新觸發 add ip 操作（有緩存的情況下不會觸發 add ip）~~。
    - 2024.07.21 版本起，從 db 恢覆的緩存被首次查詢時將觸發 add-ip。
  - tool/dns_cache_mgr 可用於操縱 db 文件，進入 tool 目錄，`./make.sh` 即可。
    - `./dns_cache_mgr`：列出 db 中的所有緩存條目（域名、qtype、TTL、size 等）。
    - `./dns_cache_mgr -r 域名後綴`：刪除給定域名的緩存條目，-r 選項可以多次指定。
    - 默認 db 文件路徑是當前目錄下的 `dns-cache.db`，可通過 `-f 文件路徑` 選項修改。

### verdict-cache

- `verdict-cache` 啟用 tag:none 域名的判決結果緩存，參數是緩存容量。
  - tag:none 域名的查詢會同時轉發給 china、trust 上遊，根據 china 上遊的 ip test 結果，決定最終響應。
  - 這里說的 **判決結果** 就是指這個 ip test 結果，即：給定的 tag:none 域名是 **大陸域名** 還是 **非大陸域名**。
  - 如果記下此信息，則後續查詢同一域名時（未命中 DNS 緩存時），只轉發給特定上遊組，不同時轉發。
  - 緩存容量上限是 65535，此緩存沒有 TTL 限制；緩存滿時會隨機刪除一個舊緩存數據。
  - 建議啟用此緩存，可幫助減少 tag:none 域名的重覆請求和判定，還能減少 DNS 泄露。
  - 注意，判決結果緩存與 DNS 緩存是互相獨立的、互補的；這兩個緩存系統可同時啟用。
- `verdict-cache-db` 啟用緩存持久化，參數是 db 文件路徑（可以不預先創建）。
  - 進程啟動時，自動從 db 恢覆緩存；進程退出時，自動將緩存寫回至 db。
  - “進程退出”是指進程收到`SIGTERM/SIGINT`信號，即`kill <PID>`或`CTRL+C`。
  - “緩存寫回”可通過`SIGUSR1`信號強制觸發（未啟用持久化則寫至`/tmp/chinadns@verdict-cache.db`）。
  - db 是“純文本”文件，可手動編輯和共享，第一個字段為“是否大陸域名(1是0否)”，第二個字段為“域名”。

### hosts、dns-rr-ip

- `hosts` 加載 hosts 文件，參數默認值為 `/etc/hosts`，此選項可多次指定。
- `dns-rr-ip` 定義本地的 A/AAAA 記錄（與 hosts 類似），此選項可多次指定。
  - 格式：`<names>=<ips>`，多個 name 使用逗號隔開，多個 ip 使用逗號隔開。

### cert-verify、ca-certs

- `cert-verify` 驗證 DoT 上遊的 SSL 證書（有效性，是否受信任，域名是否匹配）。
  - wolfssl 在某些平台（arm32、mips）可能無法正確驗證 SSL 證書，見 [#169](https://github.com/zfl9/chinadns-ng/issues/169)。
- `ca-certs` CA 根證書路徑，用於驗證 DoT 上遊的 SSL 證書。默認自動檢測。

### no-ipset-blacklist

- `no-ipset-blacklist` 若指定此選項，則 add-ip 時不進行內置的 IP 過濾。
  - 默認情況下，以下 IP 不會被添加到 ipset/nftset 集合，見 [#162](https://github.com/zfl9/chinadns-ng/issues/162)
  - `127.0.0.0/8`、`0.0.0.0/8`、`::1`、`::` (loopback地址、全0地址)

### 其他雜項配置

- `timeout-sec` 用於指定上遊的響應超時時長，單位秒，默認 5 秒。
- `repeat-times` 針對可信 DNS (UDP) [重覆發包](#trust上遊存在一定的丟包怎麽緩解)，默認為 1，最大為 5。
- `noip-as-chnip` 接受來自 china 上遊的沒有 IP 地址的響應，[詳細說明](#--noip-as-chnip-選項的作用)。
- `fair-mode` 從`2023.03.06`版本開始，只有公平模式，指不指定都一樣。
- `reuse-port` 用於多進程負載均衡（實踐證明沒必要，單進程已經夠用）。
- `verbose` 選項表示記錄詳細的運行日志，除非調試，否則不建議啟用。

## 域名列表

**文件格式**

域名列表是一個純文本文件（不支持注釋），每一行都是一個 **域名後綴**，如`baidu.com`、`www.google.com`。域名後綴不能以`.`開頭或`.`結尾。出於性能考慮，最多允許 8 級域名（2025.03.27 版本之前是 4 級），超出部分將截斷。

---

**加載順序**

所有組的域名列表都被 **加載** 到同一個數據結構，一個 **域名後綴** 一旦被加載，其內部屬性就不會被修改。因此，當一個 **域名後綴** 存在於多個組的域名列表時，優先加載的那個組將“獲勝”。舉個例子：假設 `foo.com` 同時存在於 tag:chn、tag:gfw 組的域名列表內，且優先加載 tag:gfw 組，則 `foo.com` 屬於 tag:gfw 組。

先加載“自定義組”的域名列表，然後再加載“內置組”的域名列表（chn 和 gfw 誰先，取決於`--chnlist-first`）。

---

**匹配順序**

收到 dns query 時，會對 qname 進行 **最長後綴匹配**。舉個例子，若 qname 為 `x.y.z`，則匹配順序為：

- `x.y.z`，檢查數據結構中是否存在此域名後綴。
- `y.z`，檢查數據結構中是否存在此域名後綴。
- `z`，檢查數據結構中是否存在此域名後綴。

一旦其中某個 **域名後綴** 匹配成功，匹配就結束，並獲取該 **域名後綴** 所屬的 tag(group)，並將 tag 信息記錄到該 dns query 的相關數據結構，後續所有邏輯（分流、ipset/nftset）都基於這個 tag 信息，與 qname 無關。

如果都匹配失敗，則該 dns query 的 tag 被設為 `default-tag` 選項的值，默認情況下，`default-tag` 是 none。global 分流、gfwlist 分流都基於此機制實現。你可以將 `default-tag` 設為不同的 tag，來實現各種目的。

---

**性能、內存開銷**

chinadns-ng 在編碼時特意考慮了性能和內存占用，並進行了深度優化，因此不必擔心查詢效率和內存開銷。域名條目數量只會影響一點兒內存占用，對查詢速度沒影響，也不必擔心內存占用，這是在`Linux x86-64`的實測數據：

- 沒有加載域名列表時，內存為 `140` KB
- 加載 5700+ 條 `gfwlist.txt` 時，內存為 `304` KB
- 加載 5700+ 條 `gfwlist.txt` 以及 73300+ 條 `chnlist.txt` 時，內存為 `2424` KB
- 如果確實內存吃緊，可以使用更加精簡的 chnlist 源（不建議只使用 gfwlist.txt 列表）
- 2023.04.11 版本針對域名列表的內存占用做了進一步優化，因此占用會更少，測試數據就不貼了

## 簡單測試

導入 chnroute/chnroute6 大陸 IP 數據庫（用於 tag:none 域名的 IP 測試）：

```bash
# ipset (chnroute, chnroute6)
ipset -R <res/chnroute.ipset
ipset -R <res/chnroute6.ipset

# nftset (inet@global@chnroute, inet@global@chnroute6)
nft -f res/chnroute.nftset
nft -f res/chnroute6.nftset

# 只需導入一次，除非對應集合已從內核移除（比如重啟了）
```

運行 chinadns-ng，我自己配了全局透明代理，所以訪問 `8.8.8.8` 會走代理。

```bash
# ipset
chinadns-ng -m res/chnlist.txt -g res/gfwlist.txt -v

# nftset
chinadns-ng -m res/chnlist.txt -g res/gfwlist.txt -v -4 inet@global@chnroute -6 inet@global@chnroute6
```

chinadns-ng 默認監聽 `127.0.0.1:65353`，可以給 chinadns-ng 加上 -v 參數，使用 dig 測試，觀察其日志。

## 常見問題

### tag:chn、tag:gfw、tag:none 是指什麽

這是 chinadns-ng 對域名的一個簡單分類：

- 被 chnlist.txt 匹配的域名歸為 `tag:chn`
- 被 gfwlist.txt 匹配的域名歸為 `tag:gfw`
- 被 `group-dnl` 匹配的域名歸為 `自定義組`
- 其它域名默認歸為 `tag:none`，可通過 -d 修改

**域名分流** 和 **ipset/nftset** 的核心流程，可以用這幾句話來描述：

- `tag:chn`：只走 china 上遊（單純轉發），如果啟用 --add-tagchn-ip，則添加解析結果至 ipset/nftset
- `tag:gfw`：只走 trust 上遊（單純轉發），如果啟用 --add-taggfw-ip，則添加解析結果至 ipset/nftset
- `自定義組`：只走 所屬組的 上遊（單純轉發），如果啟用 --group-ipset，則添加解析結果至 ipset/nftset
- `tag:none`：同時走 china 和 trust，如果 china 上遊返回國內 IP，則接受其結果，否則采納 trust 結果

> `tag:chn`和`tag:gfw`和`自定義組`不存在任何判定/過濾；`tag:none`的判定/過濾也僅限於 china 上遊的響應結果

---

### 如何以守護進程形式在後台運行 chinadns-ng

```bash
# 純 shell 語法：
(chinadns-ng 參數... </dev/null &>>/var/log/chinadns-ng.log &)

# 也可借助 systemd 的 service 來實現，此處不展開敘述
```

---

### 如何更新 chnroute.ipset、chnroute6.ipset

```bash
cd res
./update-chnroute.sh
./update-chnroute6.sh
ipset -F chnroute
ipset -F chnroute6
ipset -R -exist <chnroute.ipset
ipset -R -exist <chnroute6.ipset
# 支持運行時重載 ipset/nftset，無需操作 chinadns-ng 進程
```

---

### 如何更新 chnroute.nftset、chnroute6.nftset

```bash
cd res
./update-chnroute-nft.sh
./update-chnroute6-nft.sh
nft flush set inet global chnroute
nft flush set inet global chnroute6
nft -f chnroute.nftset
nft -f chnroute6.nftset
# 支持運行時重載 ipset/nftset，無需操作 chinadns-ng 進程
```

---

### 如何更新 chnlist.txt、gfwlist.txt

```bash
cd res
./update-chnlist.sh
./update-gfwlist.sh
# pkill chinadns-ng # 關閉舊的 chinadns-ng 進程
chinadns-ng -m chnlist.txt -g gfwlist.txt 其他參數... # 重新運行 chinadns-ng
```

---

### 如何使用 TCP 協議與 DNS 上遊進行通信

從 2024.03.07 版本開始，有以下更改：

- 若上遊地址為 `1.1.1.1`，則根據 **查詢方的傳入協議** 來選擇與上遊的通信協議：
  - 若查詢方的傳入協議為 UDP，則 chinadns-ng 與該上遊的通信協議為 UDP。
  - 若查詢方的傳入協議為 TCP，則 chinadns-ng 與該上遊的通信協議為 TCP。
- 若上遊地址為 `udp://1.1.1.1`，則 chinadns-ng 與該上遊的通信方式為 UDP。
- 若上遊地址為 `tcp://1.1.1.1`，則 chinadns-ng 與該上遊的通信方式為 TCP。
- 若上遊地址為 `tls://1.1.1.1`，則 chinadns-ng 與該上遊的通信方式為 TLS(DoT)。

---

### 為什麽不內置 ~~TCP~~、~~DoT~~、DoH 等協議的支持

- 2024.03.07 版本起，已內置完整的 TCP 支持（傳入、傳出）。
- 2024.04.27 版本起，支持 DoT 協議的上遊，DoH 不打算實現。

我想讓代碼保持簡單，只做真正必要的事，其他事讓專業工具去做。

換句話說，讓程序保持簡單和愚蠢，只做一件事，並認真做好這件事。

---

### chinadns-ng 並不讀取 chnroute.ipset、chnroute6.ipset

啟動時也不會檢查這些 ipset/nftset 集合是否已存在，程序只在收到來自國內上遊的 DNS 響應時（僅針對 tag:none 域名），通過 netlink 查詢給定 ipset/nftset 集合，對應 IP 是否存在。這種機制使得我們可以在 chinadns-ng 運行時直接更新 chnroute、chnroute6 列表，它立即生效，不需要重啟 chinadns-ng。

> 只有 tag:none 域名存在 ipset/nftset 判斷&&過濾。

---

### 接受 china 上遊返回的 IP為保留地址 的解析記錄

將對應的地址(段)加入到 `chnroute`、`chnroute6` 集合即可。chinadns-ng 判斷是否為"大陸 IP"的核心依據就是查詢 chnroute、chnroute6 集合，程序內部並沒有其他隱含的判斷規則。

注意：**只有 tag:none 域名需要這麽做**；對於 tag:chn 域名，chinadns-ng 只是轉發，不涉及 ipset/nftset 判定；所以你也可以將相關域名加入到 chnlist.txt 列表（支持從多個文件加載域名列表）。

為什麽沒有默認將保留地址加入 `chnroute*.ipset/nftset`？因為我擔心 GFW/ISP 會給受污染域名返回保留地址，所以沒放到 chnroute 去。不過現在受污染域名都走 gfwlist.txt 機制了，只會走 trust 上遊，加進去應該是沒問題的。

---

### received an error code from kernel: (-2) No such file or directory

- 如果是 ip test 報錯，說明未導入 chnroute、chnroute6，請導入相關 ipset/nftset 集合。
- 如果是 ip add 報錯，說明未提前創建給定的 ipset/nftset，請創建配置中給出的相關集合。

---

### trust上遊存在一定的丟包，怎麽緩解

- 方法1：**重覆發包**，也即 `--repeat-times N` 選項，這里的 `N` 默認為 1，可以改為 3，表示在給一個 trust 上遊（UDP）轉發查詢消息時，同時發送 3 個相同的查詢消息。
- 方法2：**TCP查詢**，對於新版本（>= 2024.03.07），在 trust 上遊的地址前加上 `tcp://`；對於老版本，可以加一層 [dns2tcp](https://github.com/zfl9/dns2tcp)，來將 chinadns-ng 發出的 UDP 查詢轉為 TCP 查詢。

推薦方法2，因為 QoS 等因素，TCP 流量的優先級通常比 UDP 高，且 TCP 本身就提供丟包重傳等機制，比重覆發包策略更可靠。另外，很多代理程序的 UDP 實現效率較低，很有可能出現 TCP 查詢總體耗時低於 UDP 查詢的情況。

---

### 為何選擇 ipset/nftset 來處理 chnroute 查詢

因為使用 ipset/nftset 可以與 iptables/nftables 規則共用一份 chnroute；達到聯動的效果。

---

### 是否打算支持 geoip.dat 等格式的 chnroute

目前沒有這個計劃，因為 chinadns-ng 通常與 iptables/nftables 一起使用（配合透明代理），若使用非 ipset/nftset 實現，會導致兩份重覆的 chnroute，且無法與 iptables/nftables 規則實現聯動。

---

### 是否打算支持 geosite.dat 等格式的 gfwlist/chnlist

目前沒有這個計劃，這些二進制格式需要引入 protobuf 等庫，我不想引入依賴，而且 geosite.dat 本身也很大。

---

### chinadns-ng 也可用於 gfwlist 透明代理分流

```bash
# 創建 ipset，用於存儲 tag:gfw 域名的 IP (nftset 同理)
ipset create gfwlist hash:net family inet # ipv4
ipset create gfwlist6 hash:net family inet6 # ipv6

# 指定 gfwlist.txt，default-tag，add-taggfw-ip 選項
chinadns-ng -g gfwlist.txt -d chn -A gfwlist,gfwlist6
```

傳統上，這是通過 dnsmasq 來實現的，但 dnsmasq 的 server/ipset/nftset 功能不擅長處理大量域名，影響性能，只是 gfwlist.txt 域名數量比 chnlist.txt 少，所以影響較小。如果你在意性能，如低端路由器，可使用 chinadns-ng 來實現。

---

### 將 default-tag 設置為 null，可實現白名單解析

若只想解析特定域名（如本地 hosts），並在查詢其它域名時返回 NODATA 結果（相當於過濾），該如何實現？

chinadns-ng 在收到查詢時，會先檢查本地資源記錄（比如加載的 hosts 文件），再執行域名分組邏輯（tag:chn、tag:gfw、tag:none、自定義組），其中有一個特殊自定義組 `null`，此域名組的任何查詢均會返回 NODATA 結果。

因此若想實現上述“白名單解析”邏輯，可將 default-tag 設為 `null`，此配置下，任何不在指定範圍內的域名查詢均會返回 NODATA 結果。以下是幾個相關配置示例：

> chinadns-ng 2025.06.20 版本為此用例調整了邏輯順序，如有需要，請最新到最新版本。

```shell
# 只允許查詢 /etc/hosts 中的域名
chinadns-ng --hosts --group null --default-tag null

# 只允許查詢 chnlist.txt 中的域名
chinadns-ng -m chnlist.txt --group null --default-tag null

# 只允許查詢 gfwlist.txt 中的域名
chinadns-ng -g gfwlist.txt --group null --default-tag null

# 只允許查詢 /etc/hosts、chnlist.txt、gfwlist.txt、x.txt 中的域名
chinadns-ng --hosts -m chnlist.txt -g gfwlist.txt \
          --group x --group-dnl x.txt --group-upstream 1.1.1.1 \
          --group null --default-tag null
```

---

### 使用 chinadns-ng 替代 dnsmasq 的注意事項

chinadns-ng 2.0 已經足以替代經典用例下的 dnsmasq：

- 域名分流效率比 dnsmasq 高得多，即使存在大量域名規則，也不影響性能，內存占用也很低
- nftset 操作效率比 dnsmasq 高，即使在短時間內寫入大量 IP 也沒問題（dnsmasq 可能會報錯）
- dnsmasq 的 TCP DNS 實現方式非常低效，每個 TCP 連接都可能在後台 fork 一個 dnsmasq 進程
- chinadns-ng 可強制使用 TCP 上遊，且原生支持 DoT，wolfssl 的體積/性能/內存開銷都非常優秀

對於路由器等場景，你可能仍然需要 dnsmasq 的 DHCP 等功能，這種情況下，建議關閉 dnsmasq 的 DNS：

- 修改 dnsmasq 配置，將`port`改為0，關閉 dnsmasq 的 DNS 功能，其他功能不受影響（如 DHCP）
- 此時請務必配置 `dhcp-option=option:dns-server,0.0.0.0`，確保會下發 dns-server 給 DHCP 客戶端
- 因為關閉 DNS 功能後，在未顯式配置相關 dhcp-option 的情況下，dnsmasq 不會自動下發 dns-server
- 0.0.0.0 是一個特殊 IP，dnsmasq 在內部會替換為“dnsmasq 所在主機的 IP”，避免寫死 IP 地址，更靈活

---

### --noip-as-chnip 選項的作用

> 此選項只作用於 `tag:none 域名` && `qtype=A/AAAA` && `china 上遊`，trust 上遊不存在過濾。

chinadns-ng 對 tag:none 域名的 A/AAAA 查詢有特殊處理邏輯：對 china 上遊返回的 reply 進行 ip test (chnroute)，如果測試結果是 china IP，則采納 china 上遊的結果，否則采納 trust 上遊的結果（為了減少重覆判定，可啟用 verdict-cache 來緩存該測試結果）。

要進行 ip test，顯然要求 reply 中有 IP 地址；如果沒有 IP（如 NODATA 響應），就沒辦法 test 了。

- 默認情況下，chinadns-ng 將 no-ip 視為 **非 china IP**，也即：采納 trust 上遊結果。
- 若指定了 `--noip-as-chnip`，則將 no-ip 視為 **china IP**，也即：采納 china 上遊結果。

默認拒絕 china 上遊的 no-ip 結果是為了減少 GFW/ISP 污染，防止其故意對某些域名返回空 answer (no-ip)。

---

### 如何以普通用戶身份運行 chinadns-ng

向內核查詢 ipset/nftset 需要 `CAP_NET_ADMIN` 權限，使用非 root 用戶身份運行 chinadns-ng 時將產生 `Operation not permitted` 錯誤。解決方法有很多，這里介紹其中一種：

```shell
# 授予 CAP_NET_ADMIN 權限
# 用於執行 ipset/nftset 操作
sudo setcap cap_net_admin+ep /usr/local/bin/chinadns-ng

# 授予 CAP_NET_ADMIN + CAP_NET_BIND_SERVICE 權限
# 用於執行 ipset/nftset 操作、監聽小於 1024 的端口
sudo setcap cap_net_bind_service,cap_net_admin+ep /usr/local/bin/chinadns-ng
```
