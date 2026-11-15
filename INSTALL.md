# 阿里云 ECS 重装 Debian 12 操作记录

记录日期为 2026-11-14。本文保留实际排错过程与验证结果，省略密码、密码哈希、个人 IP 和私有域名；安装后的补充建议单独列出，不表示已经执行。

## 1. 下次使用时的最短路径

流程：**确认磁盘和连接方式 → 修复旧系统 DNS → 预览配置 → 准备安装环境 → 验证文件与 GRUB → 手动重启 → SSH 登录验收**。对应命令见后续各节。

本次对话成功使用的安装命令如下。在目标 ECS 上进入本仓库执行，先确认 `/dev/vda` 是允许清空的系统盘：

```bash
sudo ./debi.sh \
  --version 12 \
  --aliyun \
  --cloud-kernel \
  --timezone Asia/Shanghai \
  --user root \
  --disk /dev/vda
```

密码由脚本交互读取。脚本成功退出后，按第 6 节验证，再执行 `sudo reboot`。**重启后安装器会自动重新分区、格式化目标磁盘，原有系统及数据不会保留。** 提前完成数据备份，并准备公网 SSH 和 ECS 控制台连接；原系统的 Tailscale 配置也会被清除。

本仓库已经包含下面的 DNS 修复。请使用修改后的本地 `debi.sh`；重新下载其他版本覆盖文件后，需要重新检查修复是否存在。

## 2. 本次环境与最终结果

| 项目 | 对话中的配置或结果 |
| --- | --- |
| 平台 | 阿里云 ECS，amd64，约 2 GB 内存、40 GB 系统盘 |
| 目标系统 | Debian 12 Bookworm；必须显式传 `--version 12`，脚本默认是 Debian 13 |
| 软件源 | `https://mirrors.aliyun.com/debian` |
| 安全更新源 | `https://mirrors.aliyun.com/debian-security`，`bookworm-security` |
| 网络 | DHCP；旧系统网卡为 `eth0`，新系统为 `ens5` |
| DHCP 应答文件中的 DNS | `223.5.5.5 223.6.6.6` |
| 内核 | `linux-image-cloud-amd64`；登录后看到 `6.1.0-54-cloud-amd64` |
| 账户 | `root`，交互设置密码，SSH 端口 22 |
| 时区 | `Asia/Shanghai` |
| 磁盘与引导 | `/dev/vda`，GPT，UEFI，ext4 根分区 |
| 安装后的分区 | `vda1` 约 100 MiB，vfat，挂载 `/boot/efi`；`vda2` 约 39.9 GiB，ext4，挂载 `/` |
| 安装后的资源使用 | 对话验收时内存约 199 MiB，根分区已用约 973 MiB；仅为当时样本 |
| 待补充项目 | NTP 未同步、Swap 为 0、缺少 `curl` |
| BBR | 未传 `--bbr`，验收值为 `cubic`，符合本次配置 |

## 3. 脚本实际做了什么

**运行脚本只是在旧系统准备安装环境；真正的 Debian 安装发生在重启后。**

1. 用 `set -eu` 启动，定义下载、版本选择、密码输入等函数，初始化默认参数并解析命令行。
2. 检测架构、选择内核、计算 APT 组件和仓库；正常执行时要求 root 权限。
3. 删除并重建同版本的 `/boot/debian-bookworm/`，生成 `preseed.cfg`。配置包含网络、镜像源、用户、SSH、时区、分区、内核和软件包。
4. 下载 Debian Installer 的 `linux` 和 `initrd.gz`；启用 `--firmware` 时还下载固件。
5. 解压 initrd，用 `cpio` 追加 `preseed.cfg`，再压缩；`--cidata` 提供的 cloud-init 数据也在此阶段注入。
6. 创建 `/etc/default/grub.d/zz-debi.cfg`，生成 GRUB 配置并追加 ID 为 `debi` 的 Debian Installer 菜单项，将其设为默认启动项。
7. 用户手动重启，GRUB 加载安装器；安装器读取 preseed，重新分区并安装 Debian，最后自动重启进入新系统。

`in_target()` 和 `configure_sshd()` 在准备阶段收集命令，最终通过 `preseed/late_command` 在新系统中执行，不会立即修改旧系统 SSH。`--ntp` 配置的是安装器的时间同步，不保证新系统已安装并运行 NTP 服务。

`--dry-run` 将 preseed 输出到标准输出、GRUB 菜单输出到标准错误，不创建安装目录、不下载安装器、不修改 GRUB；仍会探测本机环境，可能提示输入密码，传入公钥 URL 时也会检查下载。`--hold` 控制安装完成后的行为，不是运行脚本后的确认步骤；`--no-part` 只省略自动分区配置。

### 下载失败不等于自动回滚

第一次执行在下载 `linux` 时因域名解析失败退出。本次失败前已经创建安装目录、写入 `preseed.cfg`，下载器也可能留下空的 `linux` 文件；尚未执行 initrd 注入和 GRUB 修改，没有开始重新分区。

脚本没有事务式自动回滚。上述结论只适用于对话中这次失败的位置；如果之前有成功执行留下的 GRUB 配置，或失败发生在更晚阶段，必须检查现存文件，不能仅凭非零退出码判断系统已恢复原状。

## 4. 两层 DNS 问题与修复

### 4.1 旧系统 DNS：决定能否下载安装器

首次报错：

```text
Resolving mirrors.aliyun.com (mirrors.aliyun.com)...
failed: Temporary failure in name resolution.
wget: unable to resolve host address 'mirrors.aliyun.com'
```

`--aliyun` 设置安装参数，不会修改当前运行系统的 `/etc/resolv.conf`。`wget` 或 `curl` 下载仍依赖旧系统自己的解析环境，单纯重跑脚本无法解决这个问题。

先查看解析管理方式、路由和连通性：

```bash
cat /etc/resolv.conf
ls -l /etc/resolv.conf
ip route show default
getent ahostsv4 mirrors.aliyun.com
resolvectl status
resolvectl query mirrors.aliyun.com
ping -c 3 223.5.5.5
```

本次 `/etc/resolv.conf` 链接到 `/run/systemd/resolve/resolv.conf`，由 systemd-resolved 管理。`eth0` 配置了 `100.100.2.136` 和 `100.100.2.138`，另有 Tailscale DNS。公网 IP 的 ping 成功，但域名查询阻塞；ping 只能证明 ICMP 连通，仍需实际 DNS 查询定位。

使用已有的 `dig` 对比上游服务器：

```bash
for dns in 100.100.2.136 100.100.2.138 223.5.5.5 223.6.6.6; do
    printf '\nDNS: %s\n' "$dns"
    dig @"$dns" mirrors.aliyun.com A +time=2 +tries=1
done
```

| DNS | 本次实际结果 |
| --- | --- |
| `100.100.2.136` | 超时，`no servers could be reached` |
| `100.100.2.138` | 超时，`no servers could be reached` |
| `223.5.5.5` | `NOERROR`，返回 A 记录，约 8 ms |
| `223.6.6.6` | `NOERROR`，返回 A 记录，约 4 ms |

这是该实例当时到 DNS 服务的查询结果，不能据此判断所有 ECS 内网 DNS 都不可用，也没有证据需要先关闭 Tailscale。

本次通过运行时修改 systemd-resolved 的网卡 DNS 恢复下载。网卡名应从实际默认路由确认：

```bash
install_iface=$(ip -4 route show default | awk 'NR == 1 {print $5}')
# 确认输出为实际出口网卡；本次旧系统为 eth0
printf '%s\n' "$install_iface"
sudo resolvectl dns "$install_iface" 223.5.5.5 223.6.6.6
sudo resolvectl flush-caches

resolvectl status "$install_iface"
resolvectl query mirrors.aliyun.com
getent ahostsv4 mirrors.aliyun.com
```

这是临时配置，网络服务重启或 DHCP 更新后可能被覆盖。当前文件由 resolved 管理时，通过 `resolvectl` 配置；无需直接覆盖 `/etc/resolv.conf`。

### 4.2 安装配置 DNS：修复 `--aliyun` 遗漏的 `dns10`

原脚本的 `--aliyun` 只设置 `dns` 和 `dns6`，而未指定静态 `--ip` 时，生成 preseed 的分支读取 `dns10`。所以虽然软件源是阿里云，应答文件中却仍是默认 Cloudflare DNS：

```text
d-i netcfg/get_nameservers string 1.1.1.1 2606:4700:4700::1111
```

本次修改后的完整预设为：

```sh
--aliyun)
    dns='223.5.5.5 223.6.6.6'
    dns6='2400:3200::1 2400:3200:baba::1'
    dns10='223.5.5.5 223.6.6.6'
    mirror_host=mirrors.aliyun.com
    ntp=time.amazonaws.cn
    ;;
```

新增的 `dns10` 使 DHCP 模式的 preseed 写入两个阿里公共 IPv4 DNS；静态 IPv4 和 IPv6 仍分别使用原有 `dns`、`dns6` 逻辑。地址可参考[阿里云公共 DNS 官方说明](https://help.aliyun.com/zh/dns/binding-public-network-exit-ip-address/)。

注意当前参数实现的范围：`--dns` 修改静态网络使用的 `dns`，DHCP 分支仍读取 `dns10`。本次仅修复阿里云预设，不能把单独传入 `--dns` 当作 DHCP DNS 的覆盖方式。

## 5. 安装前验证与配置预览

在目标服务器检查磁盘、架构和启动模式：

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
dpkg --print-architecture
if [ -d /sys/firmware/efi ]; then echo UEFI; else echo BIOS; fi
```

本次磁盘是 `/dev/vda`、40 GB，架构为 `amd64`，`/sys/firmware/efi` 存在，所以脚本自动选择 UEFI。其他实例按实际结果选择磁盘，不直接照搬设备名。

确认镜像可访问，检查内核和 initrd 两个文件：

```bash
getent ahostsv4 mirrors.aliyun.com
installer_base=https://mirrors.aliyun.com/debian/dists/bookworm/main/installer-amd64/current/images/netboot/debian-installer/amd64
wget --spider "$installer_base/linux"
wget --spider "$installer_base/initrd.gz"
```

对话中内核地址返回 `200 OK`。需要验证实际下载时可执行：

```bash
wget -O /tmp/debian-linux "$installer_base/linux"
test -s /tmp/debian-linux
rm -f /tmp/debian-linux
```

用 dry-run 检查同一套参数。临时目录默认仅当前用户可访问，避免随意公开包含密码哈希的 preseed：

```bash
preview_dir=$(mktemp -d)
./debi.sh \
  --dry-run \
  --version 12 \
  --aliyun \
  --cloud-kernel \
  --timezone Asia/Shanghai \
  --user root \
  --disk /dev/vda \
  > "$preview_dir/preseed.cfg" 2> "$preview_dir/grub.cfg"

grep -E 'get_nameservers|root-login|partman-auto/disk|kernel/image|time/zone' \
  "$preview_dir/preseed.cfg"
cat "$preview_dir/grub.cfg"
```

预期包含以下配置；出现顺序可能不同：

```text
d-i netcfg/get_nameservers string 223.5.5.5 223.6.6.6
d-i passwd/root-login boolean true
d-i time/zone string Asia/Shanghai
d-i partman-auto/disk string /dev/vda
d-i base-installer/kernel/image string linux-image-cloud-amd64
```

检查完可执行 `rm -r "$preview_dir"`。随后使用第 1 节命令正式准备安装环境，省略 `--dry-run` 和输出重定向。

## 6. 脚本成功退出后的验证与重启

### 安装文件和 preseed

```bash
sudo ls -lh /boot/debian-bookworm/
sudo test -s /boot/debian-bookworm/linux
sudo test -s /boot/debian-bookworm/initrd.gz
sudo test -s /boot/debian-bookworm/preseed.cfg
sudo gzip -t /boot/debian-bookworm/initrd.gz
sudo grep -E 'get_nameservers|root-login|partman-auto/disk|kernel/image|time/zone' \
  /boot/debian-bookworm/preseed.cfg
```

三个文件都应存在且非空，关键配置应与上一节一致。对话中 `linux` 约 7.9 MB、`initrd.gz` 约 44 MB、preseed 约 3.2 KB；版本变化会改变大小，不应按固定大小判断。

### GRUB 配置

```bash
sudo cat /etc/default/grub.d/zz-debi.cfg
sudo grep -n 'Debian Installer' /boot/grub/grub.cfg
sudo grep -nE '^[[:space:]]*set (default|timeout)=' /boot/grub/grub.cfg
sudo sed -n "/menuentry 'Debian Installer' --id debi {/,/^}/p" /boot/grub/grub.cfg
sudo grub-script-check /boot/grub/grub.cfg
sudo grub-editenv /boot/grub/grubenv list
```

`zz-debi.cfg` 预期：

```text
GRUB_DEFAULT=debi
GRUB_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu
```

Debian Installer 菜单应有 `--id debi`，`linux`、`initrd` 指向安装目录。本次 `/boot` 没有独立分区，GRUB 中的路径为：

```text
linux /boot/debian-bookworm/linux lowmem/low=1
initrd /boot/debian-bookworm/initrd.gz
```

有独立 `/boot` 分区时，`grub-mkrelpath` 生成的 GRUB 路径可能不含 `/boot` 前缀，应按实际分区核对。

本次踩坑与判读：

- 普通用户读取 `/boot/grub/grub.cfg` 报 `Permission denied`：使用 `sudo`。
- `grep '^set default='` 没有输出：配置行有缩进，用上面的 `[[:space:]]*` 匹配，不能直接判断默认项缺失。
- 同时出现 `set default="${next_entry}"` 和 `set default="debi"`：前者是一次性启动项分支，正常无 `next_entry` 时才使用 `debi`。`grub-editenv` 用于确认是否有遗留覆盖项；对话未提供此命令的实际结果。
- 多个 timeout 值可能来自条件分支；对话中正常为 5 秒，`recordfail` 分支为 30 秒。
- `grub-script-check` 正常时没有输出，应确认退出码为 0；语法通过还需结合文件、路径和默认启动项检查。

确认验证通过、目标盘数据已备份且连接方式可用，再手动重启：

```bash
sudo reboot
```

等待安装器完成自动安装及再次重启。第 1 节命令没有启用网络控制台；安装期间需要观察进度时使用 ECS 控制台。若另行启用 `--network-console`，安装器登录用户为 `installer`，与新系统的 `root` 不同。

## 7. 登录与安装验收

在自己的电脑终端执行，将 `YOUR_ECS_PUBLIC_IP` 替换成实际 ECS 公网 IP：

```bash
ssh -p 22 root@YOUR_ECS_PUBLIC_IP
```

使用成功运行脚本时设置的密码，输入时不显示字符。安全组需允许你的来源地址访问 TCP 22；原来的 `ecs-user` 和 Tailscale 连接方式不会自动恢复。若指定过 `--authorized-keys-url`，脚本会关闭 SSH 密码认证，需要对应私钥。

无法连接时可经 ECS 控制台检查 `systemctl status ssh --no-pager` 和 `sshd -T`。不要在安装器尚未结束时，用新系统 root 账户判断登录是否成功。

以下命令在**安装完成的新系统**中以 root 执行：

```bash
cat /etc/os-release
whoami
uname -r
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
df -h
free -h
timedatectl
systemctl status ssh --no-pager
sshd -T | grep -Ei 'permitrootlogin|passwordauthentication|pubkeyauthentication'
ip route
cat /etc/resolv.conf
getent ahostsv4 mirrors.aliyun.com
sysctl net.ipv4.tcp_congestion_control
```

本次实际验收确认 Debian 12、root 登录、Cloud Kernel、UEFI 分区、SSH 和时区正常，域名解析成功。但还需要区分：

- **网卡改名为 `ens5`**：重装前的 `eth0` 不适合继续硬编码，用 `ip route` 确认当前出口。
- **`resolvectl: command not found`**：最小安装没有该命令，不表示 DNS 失效。先看 `/etc/resolv.conf` 和 `getent`，无需仅为命令缺失安装 resolved。
- **DNS 能解析，不等于已经证明上游是谁**：对话验证了 preseed 中的阿里 DNS 和安装后的解析可用性，未贴出新系统 `/etc/resolv.conf`。要确认实际使用阿里 DNS，应检查文件及当前网络管理服务，DHCP 下发配置也可能影响最终结果。
- **时区正确不等于已同步时间**：对话中 `System clock synchronized: no`、`NTP service: n/a`，需要补装时间服务。
- **Swap 为 0**：本次自动分区没有创建 Swap。
- **拥塞控制为 `cubic`**：本次未开启 BBR，属于预期结果。

## 8. 安装后补充操作

这一节是对话中的后续建议；分享记录没有给出这些操作全部完成的验证结果。

### 常用工具与缺少 curl

遇到 `-bash: curl: command not found`，直接安装并验证：

```bash
apt update
apt install -y curl
curl --version
```

需要常用运维工具时可一起安装：

```bash
apt install -y \
  wget git vim nano sudo unzip zip tar ca-certificates gnupg \
  lsb-release jq htop dnsutils net-tools rsync chrony
```

下次重装也可以在第 1 节命令中追加 `--install 'curl wget git vim sudo dnsutils chrony'`，减少最小系统缺少工具的问题。该参数属于下次可选改进，不是对话中已执行命令的一部分。

### 启用时间同步

```bash
apt install -y chrony
systemctl enable --now chrony
systemctl status chrony --no-pager
chronyc sources -v
chronyc tracking
timedatectl
```

等待可用时间源完成同步后，再确认 `System clock synchronized: yes`。服务启动不代表已同步成功；如果迟迟不同步，检查时间源可达性。

### 可选：增加 2 GiB Swap

适用于本次 ext4 根分区。先用 `free -h`、`swapon --show` 确认现状；下面的命令在 `/swapfile` 不存在时创建，已有文件时停止，避免覆盖或重复初始化：

```bash
if [ -e /swapfile ]; then
    echo '/swapfile 已存在，请先检查其用途和启用状态'
else
    fallocate -l 2G /swapfile &&
    chmod 600 /swapfile &&
    mkswap /swapfile &&
    swapon /swapfile &&
    { grep -Eq '^[[:space:]]*/swapfile[[:space:]]' /etc/fstab ||
      printf '/swapfile none swap sw 0 0\n' >> /etc/fstab; }
fi

free -h
swapon --show
```

Swap 提供内存压力缓冲，不代替物理内存。已有 Swap 时按实际工作负载决定是否需要新增。

### 恢复远程管理与业务

对话中的后续顺序是：补齐工具和时间同步 → 按需配置 Swap → 配置普通管理用户及 SSH 密钥 → 重新安装认证 Tailscale → 恢复 Docker、1Panel 等服务。先验证新的密钥登录方式可用，再调整 root 或密码登录设置，避免中断唯一可用连接。

## 9. 重启前取消本次安装准备

适用于本文 Debian/GRUB 路径，且**尚未重启进入安装器**的情况。删除默认项覆盖配置并重新生成菜单，成功后再删除本次安装目录：

```bash
sudo rm -f /etc/default/grub.d/zz-debi.cfg
sudo update-grub && sudo rm -rf /boot/debian-bookworm
sudo grep -n 'Debian Installer' /boot/grub/grub.cfg
sudo grub-script-check /boot/grub/grub.cfg
```

`grep` 应不再匹配本次安装入口，此时退出码 1 表示未找到；语法检查应成功。如果 `update-grub` 失败，先修复并核对实际 GRUB 配置，避免重启进入残留安装项。取消准备不会恢复安装器已经格式化的数据，也不会撤销手工执行的 `resolvectl dns` 临时设置。
