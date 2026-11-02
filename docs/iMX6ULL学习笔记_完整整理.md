## 1. 实验环境与整体关系

### 1.1 本次实际使用的环境

| 项目      | 本次记录                                                       |
| ------- | ---------------------------------------------------------- |
| 开发板     | 大象嵌入式剑齿虎 V2.4，i.MX6ULL，512 MiB DDR                         |
| 主机环境    | Windows，Ubuntu 22.04 虚拟机负责构建和提供 TFTP                       |
| U-Boot  | 后来下载并编译的 NXP `lf_v2026.04`；与早先研究的 OBC 改版 U-Boot 区分         |
| 内核源码    | `~/imx6ull/imx6ull_sdk_source/kernel`，Linux 6.12.3         |
| 内核构建目录  | `~/imx6ull/build-kernel`                                   |
| 交叉工具链   | Arm GNU Toolchain 11.3.Rel1，前缀 `arm-none-linux-gnueabihf-` |
| 内核默认配置  | `imx_v7_defconfig`                                         |
| 设备树     | `imx6ull-14x14-evk-emmc.dtb`，作为本次启动基线，尚未证明完全匹配板上外设         |
| 根文件系统基础 | Alpine minirootfs 3.24.3，armv7                             |
| 当前启动方式  | U-Boot 从 SD 卡启动，再通过 TFTP 将内核、DTB、外部 initramfs 加载到 DDR      |

### 1.2 四个组成部分分别负责什么

```text
芯片内固化的 Boot ROM
    ↓ 从启动介质读取启动镜像，按镜像信息初始化并加载
U-Boot：初始化、加载镜像、准备启动参数
    ↓
Linux 内核 zImage：内存、进程、驱动、文件系统等核心功能
    + DTB：描述板上硬件
    + initramfs：提供用户程序、运行库和 /init
    ↓
执行 /init，进入 Shell
```

| 概念        | 文件或工具                | 是否是“库”                          |
| --------- | -------------------- | ------------------------------- |
| 内核源码      | `.c`、`.h`、汇编等        | 不是；它们被构建成内核                     |
| 内核配置      | `.config`            | 不是；决定功能的编译选择                    |
| 设备树源码     | `.dts`、`.dtsi`       | 不是；描述硬件                         |
| DTC       | Device Tree Compiler | 不是；它是编译工具                       |
| DTB / FDT | 设备树的二进制表示            | 不是；DTB 通常指文件/blob，FDT 指其扁平化数据表示 |
| initramfs | 本次为 `rootfs.cpio.gz` | 不是；是归档形式的初始根文件系统内容              |
| musl      | C 标准库及相关运行时          | 是；Alpine 使用的 C 运行库实现            |
| ncurses   | 字符终端界面库              | 是；`menuconfig` 这类界面会用到它         |

原笔记“FDT = DTB 在内存里的二进制数据格式”可用于入门理解，但不必把 FDT 限定为“只存在内存里”；磁盘上的 DTB 保存的也是这种格式。

initramfs 与 DTC 的比较。

图中并排比较 initramfs 与 DTC。需要抓住的是：一个提供用户空间文件内容，一个负责把硬件描述编译成 DTB；两者不处于同一类工具层次。

<a id="s2"></a>
## 2. 交叉工具链与 U-Boot 构建

### 2.1 `CROSS_COMPILE` 是工具名称前缀

在 Ubuntu 的当前终端执行：

```bash
export CROSS_COMPILE="$HOME/toolchains/arm-gnu-toolchain-11.3.rel1-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf-"
```

这不是开始编译，也不是“编译地址”或输出文件夹。它设置的是**工具链的路径和名称前缀**。构建规则把前缀与工具名拼接：

```text
${CROSS_COMPILE}gcc       → C 编译器
${CROSS_COMPILE}ld        → 链接器
${CROSS_COMPILE}objcopy   → 二进制格式转换等工具
```

`Makefile` 被 `make` 读取执行，不是让 GCC 把 Makefile 当作 C 源码编译。

检查当前选择：

```bash
"${CROSS_COMPILE}gcc" --version
"${CROSS_COMPILE}gcc" -dumpmachine
```

本次得到 Arm GNU Toolchain 11.3.Rel1，以及 `arm-none-linux-gnueabihf`。如果变量为空，拼出来的就只是 `gcc`，可能调用 Ubuntu 本机编译器。`export` 在当前 Shell 及其后续子进程中生效，不按文件夹分别设置；新终端是否保留取决于启动配置。

### 2.2 生成 U-Boot 配置

原笔记命令：

```bash
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" mx6ull_14x14_evk_defconfig
```

含义：

- `ARCH=arm`：目标架构是 ARM。
- `CROSS_COMPILE=...`：指定目标工具链前缀。
- `mx6ull_14x14_evk_defconfig`：选择板级默认配置目标，生成 `.config`。

这一步主要配置构建选项，不代表已经生成 `u-boot.imx`。本实验采用独立输出目录时，配置和后续构建都要带同一个 `O=`：

```bash
# 在 U-Boot 源码根目录执行
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-uboot-imx" mx6ull_14x14_evk_defconfig

make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-uboot-imx" -j4
```

### 2.3 早期 OBC 改版源码报错的含义

原记录：

```text
Kconfig:11: can't open file "/bootloader/obcbase/Kconfig"
make[1]: *** [scripts/kconfig/Makefile:97：mx6ull_14x14_evk_defconfig] 错误 1
make: *** [Makefile:586：mx6ull_14x14_evk_defconfig] 错误 2
```

这说明那份 U-Boot 的 Kconfig 依赖外部 OBC 路径，不能把它当作完全独立的上游源码直接构建。路径退化成 `/bootloader/...`，通常还要检查 `OBC_TOP_DIR` 是否正确导出，以及 OBC 文件是否存在。它和后来成功启动的 NXP 官方 U-Boot 是两条不同的构建路线。第 10 章保留并解释 OBC 关系。

<a id="s3"></a>
## 3. SD 卡、MBR、IVT 与烧录校验

### 3.1 上电时谁加载 U-Boot

```text
电脑把 u-boot.imx 写入 SD 卡规定的原始位置
    ↓
开发板上电，CPU 执行芯片内部 Boot ROM
    ↓
ROM 读取启动介质上的初始区域，检查 IVT 等启动信息
    ↓
在本次 DCD 启动路径中，依据 DCD 初始化 DDR
    ↓
按镜像里的加载信息装入后续内容
    ↓
跳转到 U-Boot 入口
```

原图用“先读 4 KB 到片内 RAM”解释本次 SD/MMC 启动的初始加载过程。这里的 4 KiB 是初始读取区域的概念，不是整个 U-Boot 只能有 4 KiB，也不代表每种启动介质、每种 SPL/plugin 路径都完全相同。


| 启动信息                          | 作用                    |
| ----------------------------- | --------------------- |
| IVT，Image Vector Table        | 提供入口及其他启动结构的位置        |
| DCD，Device Configuration Data | 描述 DDR 等硬件初始化所需的寄存器配置 |
| Boot Data                     | 提供镜像加载地址、长度等信息        |
SD 启动：Boot ROM、IVT、DCD 与 DDR。

### 3.2 找到 SD 卡，理解分区表

在 Ubuntu 查看设备：

```bash
lsblk -o NAME,SIZE,MODEL,TYPE,MOUNTPOINTS
sudo fdisk -l /dev/sdb
```

`lsblk` 显示块设备、容量、型号、类型、挂载点；`fdisk -l` 读取分区表，显示扇区大小、起止扇区、分区类型等。本次 `/dev/sda` 是虚拟机系统盘，`/dev/sdb` 是约 3.7 GiB 的 SD 卡。**设备名会变化，重新插拔后必须按实际输出确认；这里的 `/dev/sdb` 是历史实验值。**

原笔记对 MBR 的总结保留为：

fdisk -l 读取磁盘的分区表（MBR/GPT），展示磁盘总容量、扇区大小、分区列表，包含每个分区的起始扇区、占用扇区、分区类型、是否为启动分区。

本次是 DOS/MBR 分区表，每扇区 512 字节，第一分区从扇区 225280 开始，即 110 MiB 处。



| 项目   | MBR(DOS)           | GPT                  |
| ---- | ------------------ | -------------------- |
| 全称   | Master Boot Record | GUID Partition Table |
| 分区上限 | 最多4主分区，靠扩展分区加逻辑分区  | 最多128个主分区            |
| 最大磁盘 | 2TB                | 超大（几十EB）             |
| 备份   | 只有一份，容易丢失          | 头部一份、磁盘末尾还有备份        |
| 配套固件 | BIOS               | UEFI                 |
|      |                    |                      |
原笔记中的 MBR 与 GPT 对照。

原图比较 MBR 与 GPT。需要补充三个条件：MBR 的约 2 TiB 限制对应常见 512 字节逻辑扇区；GPT 的“128 分区”是常见初始化数量而非格式绝对上限；BIOS/MBR、UEFI/GPT 是常见 PC 组合，不是普遍强制关系，更不能直接拿来解释 i.MX Boot ROM 的裸镜像启动。
### 3.3 `0x400` 是整张卡上的字节偏移

本次 `u-boot.imx` 的 IVT 在文件开头；需要让它落在**整张 SD 卡起点后的 `0x400 = 1024` 字节位置**，不是 H 盘或任意分区起点后的 1024 字节。

| 扇区 | 字节范围 | 本次处理 |
|---|---|---|
| 0 | 0～511 | MBR，保留 |
| 1 | 512～1023 | 跳过 |
| 2 | 1024～1535 | 开始写入 `u-boot.imx`，开头为 IVT |
| 后续 | 1536 起 | 继续写入镜像 |
扇区编号和本次 u-boot.imx 写入位置。*

扇区本来就是块设备的寻址单位，不需要按表“创建扇区”。这张表只是解释写入位置。若换成 GPT 卡，前部可能已有 GPT 元数据，不能照搬本次写法。

### 3.4 检查文件头和长度

```bash
od -An -tx1 -N4 ~/imx6ull/build-uboot-imx/u-boot.imx
stat -c '%n：%s 字节' ~/imx6ull/build-uboot-imx/u-boot.imx
```

`od` 以十六进制查看文件前 4 字节；本次为 `d1 00 20 40`，与预期 IVT 头匹配。这是格式线索，不证明所有 DDR 参数、镜像内容都已匹配开发板。

`stat -c` 自定义输出：`%n` 是文件名，`%s` 是文件字节数。原笔记解释中的 `% n`、`% s` 带空格是排版笔误，实际命令不要加空格。

本次长度 613376 字节，即 599 KiB。起点 1024，加上长度得到结束边界 614400，即 600 KiB；最后实际写入字节为 614399。第一分区从 110 MiB 开始，因此这次镜像不覆盖已有分区。
（查看前面的ivt标识）
### 3.5 写入、校验、弹出

以下写入命令只适用于**已经确认目标设备、分区布局和镜像格式的本次场景**：

```bash
sudo dd if="$HOME/imx6ull/build-uboot-imx/u-boot.imx" \
  of=/dev/sdb bs=1K seek=1 conv=notrunc,fsync status=progress
```

| 参数                | 意思                                        |
| ----------------- | ----------------------------------------- |
| `sudo`            | 使用直接写块设备所需的权限                             |
| `if=...`          | 输入镜像文件；`$HOME` 在本次 Ubuntu 为 `/home/xq999` |
| `of=/dev/sdb`     | 输出到整张卡，不是 `/dev/sdb1`                     |
| `bs=1K`           | 每块 1024 字节                                |
| `seek=1`          | 输出端跳过 1 块，也就是 1024 字节                     |
| `notrunc`         | 不截断输出，对普通文件尤其重要                           |
| `fsync`           | 完成前同步输出数据                                 |
| `status=progress` | 显示复制进度；原笔记也保留了不含此参数的等价基本写法                |
dd 写入命令逐项解释。

校验：

```bash
sudo cmp -n 613376 -i 0:1024 \
  "$HOME/imx6ull/build-uboot-imx/u-boot.imx" /dev/sdb && echo "校验通过"
```

`-n 613376` 限制比较长度；`-i 0:1024` 表示第一个输入从 0 字节开始，第二个输入从 1024 字节开始；`&&` 仅在比较成功时输出“校验通过”。重新编译后镜像大小可能变化，不能一直沿用这个历史长度。cmp 校验长度与输入偏移。*

完成后：

```bash
sudo eject /dev/sdb
```

这一步请求弹出卡。随后开发板按 SD 启动模式上电，从卡上的 U-Boot 启动。本文保留的是镜像写入与校验记录，不把“校验通过”当成所有板级配置正确的证明。

<a id="s4"></a>
## 4. 虚拟机网络、MAC、PHY 与 TFTP

### 4.1 区分命令在哪个系统执行

| 提示符或环境 | 典型命令 |
|---|---|
| Ubuntu：`xq999@...$` | `ip`、`systemctl`、`make`、`sudo`、`find` |
| U-Boot：`=>` | `setenv`、`printenv`、`net list`、`ping`、`tftpboot`、`bootz` |
| 开发板 Linux：`~ #` | `ls`、`uname`、`cat /proc/...` |

Ubuntu 的 `ip -br addr` 不能直接在 U-Boot 中使用。U-Boot 的网络命令是编译进 U-Boot 的功能，不需要在其命令行临时安装 TFTP 库。

### 4.2 NAT 与桥接

原总结保留并细化：桥接让虚拟机直接接入所选物理网卡连接的网络，可作为独立网络节点；NAT 让虚拟机处于宿主机提供的私有网络，出站访问由宿主机转发，外部通常不能直接主动访问它，除非另设转发或路由。

本次保留 NAT 网卡 `ens33` 用于上网，另加桥接网卡 `ens37` 接电脑的有线网卡，与开发板直连。桥接直连网络和 NAT 网络不要求是同一个子网。

在 Ubuntu：

```bash
ip -br addr
sudo ip addr replace 192.168.50.10/24 dev ens37
sudo ip link set ens37 up
```

- `ip -br addr`：简洁显示地址和接口状态。
- 原笔记还记录过 `sudo ip addr add 192.168.50.10/24 dev ens37`；已经存在时会提示地址已分配，`replace` 适合重复设置同一地址。
- `/24` 对应掩码 `255.255.255.0`。
- `ip link ... up`：设置接口的管理状态为启用，不等同于“给网卡硬件上电”，也不能保证网线链路已经连通。
- 这里的 IP 是为直连实验分配的静态地址，并不是板子天生具有的地址；同一网络内避免与别的设备冲突。

### 4.3 MAC、MII、MDIO 与 PHY

| 名称 | 含义 |
|---|---|
| MAC 控制器 | 以太网介质访问控制硬件，本例集成在 SoC 中 |
| MAC 地址 | 以太网帧中的链路层地址；与 MAC 控制器这个硬件概念相关但不同 |
| PHY | 以太网物理层芯片，负责物理链路相关功能 |
| MII / RMII 等 | MAC 与 PHY 之间的数据接口规范；规范有对应硬件信号实现，不应简单理解为“没有硬件” |
| MDIO / MDC | 管理 PHY 寄存器、读取状态的管理接口，不承载普通业务帧数据 |

原笔记写过 `mdio list`、`mii device`、`mii info`。这些命令是否存在取决于 U-Boot 构建配置。`mdio list` 是查看已注册的 MDIO 总线/PHY 信息，不应一概说成扫描所有物理地址。`mii device` 用于查看/选择当前 MII 管理设备或总线，具体显示依实现而定，不宜直接等同于 MAC 控制器实例。

本次这两个命令组显示 `Unknown command`，于是改用：

```text
help
help net
net list
dm tree
```

`help net` 查看这个版本提供的网络子命令；`dm tree` 查看驱动模型里的设备层级及探测状态。显示设备不等于物理链路一定可用。

### 4.4 设置 U-Boot 网络参数

先查看：

```text
printenv ethaddr eth1addr ethact ipaddr serverip
```

| 变量         | 含义                     |
| ---------- | ---------------------- |
| `ethaddr`  | 第一网口使用的 MAC 地址         |
| `eth1addr` | 第二网口使用的 MAC 地址         |
| `ethact`   | 当前活动以太网设备名称            |
| `ipaddr`   | 开发板 IP                 |
| `serverip` | TFTP 服务器 IP，本次是 Ubuntu |

本次可用设置：

```text
setenv ethaddr 02:12:34:56:78:90
setenv eth1addr 02:12:34:56:78:91
setenv ethact ethernet@2188000
setenv ipaddr 192.168.50.20
setenv serverip 192.168.50.10
setenv netmask 255.255.255.0
ping 192.168.50.10
```

原笔记有一处第二行误写为 `setenv ethaddr ...91`，会覆盖第一网口地址。应使用 `eth1addr`，原图本身写法正确。

两个网口分别设置 MAC 地址。

`02:...` 是本地管理的单播 MAC 示例；两个网口分别取不同值。`ethernet@2188000` 来自本次 `net list` 的实际设备名称，不是随意命名。

当出现 `host 192.168.50.10 is alive` 时，证明这次 ping 成功。此前另一网口 `ethernet@20b4000` 出现 PHY 协商超时，而 `ethernet@2188000` 可用，因此本次明确选择后者。

恢复已验证的 U-Boot 网络参数。*

### 4.5 安装并检查 TFTP 服务

Ubuntu：

```bash
sudo apt update
sudo apt install tftpd-hpa tftp-hpa
cat /etc/default/tftpd-hpa
systemctl is-active tftpd-hpa
sudo systemctl start tftpd-hpa
```

`tftpd-hpa` 是服务端，`tftp-hpa` 是客户端工具包，都不是内核驱动。`start` 启动服务，`is-active` 查询是否运行。

本次配置：

```text
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"
```

在服务目录准备测试文件：

```bash
printf 'tftp-ok\n' | sudo tee /srv/tftp/test.txt
sudo chmod 644 /srv/tftp/test.txt
```

`printf` 输出带换行的内容，`tee` 将内容写进文件并通常回显，`644` 让文件可被读取。可以在 U-Boot 用 `tftpboot` 下载测试文件；使用哪个 DDR 测试地址仍要避开已有镜像及保留区。

<a id="s5"></a>
## 5. 编译内核与设备树

### 5.1 找到真正的内核源码根目录

```bash
find ~/imx6ull -maxdepth 6 -type d -name kernel
```

`find` 从 `~/imx6ull` 开始；`-maxdepth 6` 最多向下 6 层；`-type d` 只找目录；`-name kernel` 匹配目录名。搜索会同时找到源码树内部的 `arch/.../kernel`、`rust/kernel` 等子目录，并不是每一个都是完整内核源码根目录。

本次进入：

```bash
cd ~/imx6ull/imx6ull_sdk_source/kernel
head -n 8 Makefile
ls arch/arm/configs/*imx*defconfig
find arch/arm/boot/dts -type f \( -iname '*imx6ull*.dts' -o -iname '*alientek*.dts' \)
```

`head` 用于查看 `VERSION`、`PATCHLEVEL`、`SUBLEVEL`，本次为 6.12.3。`ls` 查默认配置模板；`find` 查板级设备树文件。

`defconfig` 是预制的配置清单，控制架构和驱动、文件系统、调试等功能选择；不是所有硬件内存布局都写在 defconfig 里，具体设备地址、连接和内存描述往往还来自设备树、平台代码与启动参数。

### 5.2 源码树不干净：备份与清理

原报错：

```text
The source tree is not clean
please run 'make ARCH=arm mrproper'
```

源码目录已有 `.config`、`include/config/`、`arch/arm/include/generated/` 等生成物，影响采用独立输出目录的构建。先看：

```bash
ls -ld .config include/config arch/arm/include/generated
```

原笔记的完整备份循环，修正为可以直接执行的格式：

```bash
backup_dir=$(mktemp -d "$HOME/imx6ull/kernel-backup.XXXXXX")

for file in .config .config.old Module.symvers certs/signing_key.pem certs/x509.genkey; do
    if [ -f "$file" ]; then
        cp -a --parents "$file" "$backup_dir/"
    fi
done

echo "备份目录：$backup_dir"
ls -lh "$backup_dir/.config"
```

| 片段 | 含义 |
|---|---|
| `mktemp -d ...XXXXXX` | 创建不重名目录；实际随机字符替换末尾 X |
| `backup_dir=$(...)` | 把命令输出的目录路径存入变量 |
| `for file in ...` | 依次处理这五个文件名 |
| `[ -f "$file" ]` | 判断是否存在该普通文件 |
| `cp -a --parents` | 保留属性复制，并在备份目录保留相对路径 |
| `fi` / `done` | 分别结束条件与循环 |

原笔记还提供了只备份 `.config` 的简化版本，保留如下；它是上面方案的替代写法，不必重复执行：

```bash
backup_dir=$(mktemp -d "$HOME/imx6ull/kernel-backup.XXXXXX")
cp -a .config "$backup_dir/"
echo "$backup_dir"
ls -lh "$backup_dir/.config"
```

原文括号注释中“创建目录”和“复制配置”的解释顺序混在一起，以上已经对应回实际命令。本次确认备份目录为 `~/imx6ull/kernel-backup.e8UqHt`，之后执行：

```bash
make ARCH=arm mrproper
```

它清理配置和较彻底的构建生成物，不是每次编译前都要执行。备份是为了保留已有成果，不是 `mrproper` 命令强制要求的前置步骤。

### 5.3 `GLIBC_2.38` 报错与宿主工具冲突

原错误：

```text
/home/xq999/toolchains/host/bin/m4: /lib/x86_64-linux-gnu/libc.so.6:
version `GLIBC_2.38' not found
```

这表明正在执行的这个 `m4` 二进制依赖宿主机未提供的 glibc 符号版本。本次 Ubuntu 为 glibc 2.35，而旧 `host/bin` 里的工具要求 `GLIBC_2.38`。不要把这误认为目标板 Alpine 的 musl 出问题。

原笔记说“至少需要 2.38”可以作为这次报错的直观解释，但精确含义是缺少该版本的符号依赖；不是直接证明升级一个文件就能解决所有依赖。

| 构建工具 | 作用 |
|---|---|
| GNU m4 | 宏处理器 |
| flex | 词法分析器生成器 |
| bison | 语法分析器生成器 |

flex 与 bison 生成 Kconfig 解析所需的代码。它们运行在 Ubuntu 上，而交叉编译器生成 ARM 目标代码，两类工具要分开看。

本次调整：

```bash
export PATH="/usr/bin:/bin:$PATH"
export M4=/usr/bin/m4
hash -r
```

把 Ubuntu 系统工具目录放在 PATH 前面，而不是“去刚下载的地方找”；`M4` 显式指定系统 m4；`hash -r` 清除 Shell 缓存的命令路径，下次重新按 PATH 查找。

### 5.4 生成配置、编译 zImage

```bash
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-kernel" imx_v7_defconfig

make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-kernel" -j4 zImage
```

第一条生成构建目录中的 `.config`；第二条使用它编译、链接并产生 zImage。`-j4` 允许最多 4 个并行构建任务。`O=` 控制输出目录，不是目标板内存地址。

```text
内核源码
    ↓ imx_v7_defconfig 生成 .config
选择要编译的功能
    ↓ make 调用交叉工具链编译、链接、生成自解压镜像
zImage
```

<a id="img-20261031201751"></a>
内核源码与配置到 zImage。

原笔记中的 `zlmage` 拼写统一改为 `zImage`：这里是大写字母 I，不是小写 l。

### 5.5 DTC 编译设备树

DTC = Device Tree Compiler。源码在内核目录里；我们选择的板级 `.dts` 还引用其他 `.dts` / `.dtsi` 文件，构建时先预处理，再由 DTC 生成 `.dtb`。

```bash
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-kernel" \
  nxp/imx/imx6ull-14x14-evk-emmc.dtb
```

```text
imx6ull-14x14-evk-emmc.dts + 被引用的源码
    ↓ 预处理与 DTC 编译
imx6ull-14x14-evk-emmc.dtb
```

设备树源码到 DTB。

日志中的 `DTC arch/arm/boot/dts/...dtb` 表示正在生成这个输出，不是一条需要再次手工输入的命令。

### 5.6 放入 TFTP 目录

```bash
sudo install -m 644 \
  ~/imx6ull/build-kernel/arch/arm/boot/zImage \
  /srv/tftp/zImage

sudo install -m 644 \
  ~/imx6ull/build-kernel/arch/arm/boot/dts/nxp/imx/imx6ull-14x14-evk-emmc.dtb \
  /srv/tftp/imx6ull-14x14-evk-emmc.dtb
```

`install -m 644` 在这里是复制文件并设置权限，不是安装到 Ubuntu 的启动内核中。原文命令拆成多行但没有续行反斜杠，直接逐行执行会出错；这里已经修正。

<a id="s6"></a>
## 6. DDR 地址选择与镜像大小

### 6.1 用 bdinfo 看真实内存范围

在 U-Boot 执行：

```text
bdinfo
```

本次输出中的 DDR 起始地址为 `0x80000000`，大小 `0x20000000 = 512 MiB`，最后一个字节地址为 `0x9fffffff`。

地址不是随便选，也不是每个文件都有固定强制地址；要同时满足 DDR 范围、文件间不重叠、避开 U-Boot 保留区，并考虑内核解压及启动时搬移后的占用。

### 6.2 本次的三个加载地址

| 文件 | 加载地址 | 距 DDR 起点 | 本次记录的大小 |
|---|---|---|---|
| zImage | `0x82000000` | 32 MiB | 初次记录 9684856 字节，约 9.3 MiB；后续重编译可能变化 |
| DTB | `0x83000000` | 48 MiB | 45946 字节，约 45 KiB |
| 外部 initramfs | `0x84000000` | 64 MiB | 下载记录 3181928 字节，`ls -lh` 约显示 3.1M |


三个文件的加载地址和历史大小。

```text
0x80000000  DDR 起点
    │ 低地址空间供内核启动、解压等使用，具体占用由构建与启动路径决定
0x82000000  zImage 的初始下载位置
    │ 文件内容及后面的空隙
0x83000000  Linux DTB 的初始下载位置
    │ 文件内容及后面的空隙
0x84000000  rootfs.cpio.gz 的初始下载位置
    │ 文件内容及其他可用内存
    ……
高地址区域  U-Boot 运行及相关保留空间
0x9fffffff  DDR 最后一个字节
```

包含 initramfs 的 DDR 布局。*

上图右边缘有少量文字截断，地址和主要布局可读；文字版补全的是本次已明确说明的含义，不声称识别到了图外文字。

原笔记早期只画了内核和设备树，仍保留这张阶段图：

<a id="img-20261030170858"></a>
```text
0x80000000  DDR 起点
│
├─ 0x82000000  zImage（内核）下载位置     │ 存放内核镜像
│
├─ 0x83000000  DTB（设备树）下载位置     │ 存放dtb设备树文件
│
├─ 0x84000000  rootfs.cpio.gz（initramfs）下载位置     │ 存放ramdisk根文件系统压缩包
│
…… 高地址区域  U-Boot 运行及相关保留空间
│
0x9FFFFFFF  DDR 最后一个字节
```

最初的内核和设备树加载布局。

### 6.3 U-Boot 自己也在 DDR 中

下面是原笔记中的完整 `bdinfo` 记录。它是一轮启动时的快照，不应视为所有后续构建永久不变的内存布局。

```text
=> bdinfo
boot_params = 0x80000100
DRAM bank   = 0x00000000
-> start    = 0x80000000
-> size     = 0x20000000
flashstart  = 0x00000000
flashsize   = 0x00000000
flashoffset = 0x00000000
baudrate    = 115200 bps
relocaddr   = 0x9ee7a000
reloc off   = 0x1767a000
Build       = 32-bit
current eth = ethernet@2188000
ethaddr     = 02:12:34:56:78:90
IP addr     = 192.168.50.20
fdt_blob    = 0x9de6f2f0
Video       = lcdif@21c8000 active
FB base     = 0x9ef00000
FB size     = 480x272x32
lmb_dump_all:
 memory.count = 0x1
 memory[0]      [0x80000000-0x9fffffff], 0x20000000 bytes, flags: none
 reserved.count = 0x4
 reserved[0]    [0x82000000-0x8293c777], 0x93c778 bytes, flags: none
 reserved[1]    [0x83000000-0x8300b379], 0xb37a bytes, flags: none
 reserved[2]    [0x9ce64000-0x9ce6efff], 0xb000 bytes, flags: no-notify, no-overwrite
 reserved[3]    [0x9ce6f2d0-0x9fffffff], 0x3190d30 bytes, flags: no-overwrite
devicetree  = separate
arch_number = 0x00000000
TLB addr    = 0x9fff0000
irq_sp      = 0x9de6f2e0
sp start    = 0x9de6f2d0
Early malloc usage: 800 / 2000
```

| 字段 | 应如何理解 |
|---|---|
| `relocaddr = 0x9ee7a000` | 这次 U-Boot 重定位后的起始位置 |
| `fdt_blob = 0x9de6f2f0` | U-Boot 自己使用的控制设备树；不能与 `0x83000000` 处的 Linux DTB 混淆 |
| `FB base = 0x9ef00000` | 显示帧缓冲起点 |
| `TLB addr = 0x9fff0000` | U-Boot 为 MMU 转换表保留的内存地址，不是 CPU 内部硬件 TLB 缓存的地址 |
| `sp start` / `irq_sp` | 与栈有关的位置 |
| `reserved[...]` | LMB 记录的保留区域；本次已加载镜像和 U-Boot 相关空间均有体现 |


0x80000000  —— DDR 起点
│
0x82000000  —— 下载的 Linux zImage
│              占用到 0x8293C777
│
0x83000000  —— 下载的 Linux 设备树
│              占用到 0x8300B379
│
│              其他内存空间
│
0x9CE64000  —— 高地址区域预留块起始
│              包含 U-Boot 运行所需各类空间
│
0x9DE6F2F0  —— U-Boot自用设备树：fdt_blob
│
0x9EE7A000  —— U-Boot重定位起点：relocaddr
│
0x9EF00000  —— 显示帧缓冲起点：FB base
│
0x9FFF0000  —— MMU页表区域：TLB addr
│
0x9FFFFFFF  —— DDR 末尾


bdinfo 对应的 U-Boot、页表、帧缓冲与镜像区域。*

原笔记“TLB（快表）本身是 CPU 内部的一小块高速硬件缓存，不是一块内存地址”是正确的概念提醒。之所以 `bdinfo` 有 `TLB addr`，是该字段使用了历史名称，应结合 U-Boot 页表用途解释。

### 6.4 通过什么命令指定地址

```text
tftpboot 0x82000000 zImage
tftpboot 0x83000000 imx6ull-14x14-evk-emmc.dtb
tftpboot 0x84000000 rootfs.cpio.gz
setenv ramdisk_size ${filesize}
bootz 0x82000000 0x84000000:${ramdisk_size} 0x83000000
```

`tftpboot` 的第一个参数是目的 DDR 地址，第二个参数是服务器上的文件名。每次成功下载后 `filesize` 更新为本次文件的字节数，以十六进制文本保存。紧接着下载 rootfs 后把它存进 `ramdisk_size`，防止后续下载覆盖该值。

`setenv ramdisk_size ...` 只是保存数值，不负责搬运文件、分配内存，也不是内核会直接读取这个同名 U-Boot 变量。真正生效的是 `bootz` 使用这个数值，按启动协议把 initramfs 的地址范围传给内核。

### 6.5 为什么 zImage 和 DTB 没有显式写长度，initramfs 却写了

原问题与总结保留为：

> 为什么 zImage 没有写长度，而根文件系统镜像需要 `ramdisk_size`？原理解是“image 和设备树有头部，根文件目录没有头部”。

更准确的解释是：zImage 有 U-Boot 能解析的镜像头信息；DTB 头有 `totalsize`。我们传的是**没有 U-Boot legacy/FIT 镜像封装的原始 initramfs 压缩包**，`bootz` 对这种输入要求显式提供大小。

不能说 gzip/cpio 完全没有头部：gzip 和 cpio 都有各自结构，只是不能因此省略该启动命令要求的原始 initramfs 输入长度。[U-Boot 官方 bootz 参数说明](https://docs.u-boot.org/en/latest/usage/cmd/bootz.html)

### 6.6 初始加载地址不等于最终位置

U-Boot 在启动准备阶段可能搬移 DTB 或 initramfs。本次日志已经出现把 DTB 从 `0x83000000` 搬到 `0x9ce55000`。解开 initramfs 后，文件内容和元数据由内核内存管理分配，根目录 `/` 不是“一个固定 DDR 地址”。

SD 卡 `0x400` 则是存储介质上的字节偏移，与 DDR 的 `0x80000000` 起点属于不同地址空间，不能混为一谈。

### 6.7 检查 eMMC 的记录

```text
mmc list
mmc dev 1
mmc info
mmc part
```

mmc 设备与分区检查命令。

图中解释了列设备、选设备、查看容量和分区。本次 `mmc dev 1` 实际报 `Card did not respond to voltage select! : -110`，并未成功切换到 eMMC；后续 `mmc info` 和 `mmc part` 仍可能显示原来的 SD 设备 0。不能把这份 SD 的分区表误认成 eMMC 的分区表，也不能仅凭这个错误断言 eMMC 硬件损坏。

<a id="s7"></a>
## 7. 制作 Alpine initramfs

### 7.1 initramfs 在整个流程里的位置

```text
下载 Alpine 根文件目录：已有 /bin/sh、BusyBox、运行库
    ↓ 添加 /init，准备 /dev/console
用 cpio + gzip 打包为 rootfs.cpio.gz
    【这是本次使用的外部 initramfs 镜像】
    ↓ U-Boot 通过 TFTP 加载到 DDR
bootz 告诉内核镜像的位置和大小
    ↓ 内核解压并解析 cpio
在内存中的根文件系统 / 创建对应文件
    ↓ 内核让 PID 1 执行 /init
/init 挂载需要的文件系统，最后 exec /bin/sh
    ↓
串口出现 ~ #，可以输入命令
```

<a id="img-20261031201304"></a>
外部 initramfs 从根文件目录到 Shell 的流程。

“外部”表示单独加载，并没有编译进 zImage。修改 `/init` 或根目录里的普通程序后，通常重新打包、更新 TFTP 文件即可；只有需求涉及内核功能变化时，才需要同步修改和重编内核。

### 7.2 下载与 SHA-256 校验

Ubuntu：

```bash
mkdir -p ~/imx6ull/rootfs-download
cd ~/imx6ull/rootfs-download

wget https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/armv7/alpine-minirootfs-3.24.3-armv7.tar.gz
wget https://dl-cdn.alpinelinux.org/alpine/latest-stable/releases/armv7/alpine-minirootfs-3.24.3-armv7.tar.gz.sha256
sha256sum -c alpine-minirootfs-3.24.3-armv7.tar.gz.sha256
```

这是本次使用的历史下载地址，版本固定为 3.24.3；以后 `latest-stable` 指向会变化，复现实验时可到官方历史版本目录寻找对应文件。校验文件提供预期哈希，`sha256sum -c` 计算本地文件并比对，输出“成功”表示内容匹配。它不等于把源码重新编译了一遍。

### 7.3 解压，保留权限与数字 UID/GID

```bash
mkdir -p ~/imx6ull/rootfs-alpine
sudo tar --numeric-owner -xzpf \
  ~/imx6ull/rootfs-download/alpine-minirootfs-3.24.3-armv7.tar.gz \
  -C ~/imx6ull/rootfs-alpine
```

| 参数                | 含义                               |
| ----------------- | -------------------------------- |
| `-x`              | 解包                               |
| `-z`              | 处理 gzip 压缩                       |
| `-p`              | 保留归档中的权限                         |
| `-f`              | 后面给出输入文件                         |
| `-C`              | 解压到指定目录；目录需先存在                   |
| `--numeric-owner` | 用归档中的 UID/GID 数字恢复归属，避免按主机同名用户映射 |

原笔记“加压进去”改为“解压进去”。解压后出现 `bin dev etc home lib media mnt opt proc root run sbin srv sys tmp usr var` 等目录。

### 7.4 检查 BusyBox 与动态加载器

```bash
ls ~/imx6ull/rootfs-alpine
file ~/imx6ull/rootfs-alpine/bin/busybox
```

本次输出：

```text
ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV),
dynamically linked, interpreter /lib/ld-musl-armhf.so.1,
BuildID[sha1]=bde5a63a28557b9b96048e9907d36b6d26315215, stripped
```

| 字段 | 说明 |
|---|---|
| ELF | Linux 常用的可执行文件格式 |
| 32-bit / ARM | ARM 32 位程序 |
| LSB | 小端；描述这个二进制的字节序，并不是所有 ARM 场景只能小端 |
| PIE | 位置无关可执行文件，允许配合地址随机化加载；不意味着每次必定随机到不同地址 |
| EABI5 | ARM EABI 信息；仅凭这个字段不能独立断定 ARMv7 或硬浮点属性 |
| dynamically linked | 动态链接，运行时需要加载器和相应运行库 |
| interpreter | 内核启动这个 ELF 时使用的动态加载器路径 |
| BuildID | 构建标识，有助于关联二进制和调试信息，不应当作绝对唯一性保证 |
| stripped | 常规符号/调试信息已剥离，调试更困难；有对应调试信息仍可用 GDB 分析 |

BusyBox ELF 字段原解释及正文修正。

原图把 `EABI5` 直接说成“ARMv7、hard-float”，把 stripped 说成“不能直接 GDB 调试”，表述偏绝对，以上作了限定。镜像选自 Alpine armv7，加上 `ld-musl-armhf.so.1` 等信息，可以判断本次用户空间目标；若需严格验证 ABI，可进一步用 `readelf` 看 ELF attributes。

`ld-musl-armhf.so.1` 在下载包里就有，不是我们添加 `/init` 时生成的。它是 musl 动态加载器，并结合 musl 的 C 运行库实现提供程序运行所需功能。Alpine 使用 musl，不使用这份 Ubuntu 工具报错所涉及的 glibc；这并不意味着 musl 应用永远不存在 ABI、库版本或架构兼容问题。

### 7.5 写 /init：把启动操作组织成脚本

以下是可执行版本，中文解释放在代码块外。原笔记在 `#!/bin/sh`、命令末尾直接接中文括号，是学习注释，不能原样写入脚本。

```bash
sudo tee ~/imx6ull/rootfs-alpine/init > /dev/null <<'EOF'
#!/bin/sh

export PATH=/usr/sbin:/usr/bin:/sbin:/bin

mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

mkdir -p /dev/pts /run
mount -t devpts devpts /dev/pts
mount -t tmpfs tmpfs /run

exec </dev/console >/dev/console 2>&1

echo
echo "Welcome to i.MX6ULL Linux!"
echo "Root filesystem: Alpine initramfs"
echo

exec /bin/sh
EOF

sudo chmod 755 ~/imx6ull/rootfs-alpine/init
```

外层写文件命令的含义：

- `sudo tee 文件`：以所需权限写入文件；`tee` 通常也把内容输出到屏幕。
- `> /dev/null`：这里丢弃 tee 的屏幕输出，不影响写目标文件。
- `<<'EOF'`：从下一行读取文本到独占一行的 `EOF`；引号使内容按字面传给 tee，不在 Ubuntu 这里提前展开变量。
- `chmod 755`：让 `/init` 可以执行。

tee、重定向和 here-document。

脚本本身的含义：

| 脚本行 | 目的 |
|---|---|
| `#!/bin/sh` | 指定脚本解释器 |
| `export PATH=...` | 让 Shell 在这些目录查命令 |
| proc / sysfs | 提供进程、内核、设备相关的虚拟文件接口 |
| devtmpfs | 在 `/dev` 提供内核注册设备对应的节点 |
| devpts | 准备伪终端文件系统 |
| tmpfs `/run` | 提供内存中的运行时文件目录 |
| `exec </dev/console >/dev/console 2>&1` | 将当前 Shell 及后续继承者的标准输入、输出、错误连接到控制台 |
| `echo ...` | 输出欢迎信息 |
| `exec /bin/sh` | 用交互 Shell 替换当前程序，PID 保持不变 |

原总结“把当前 init 进程的代码换成 /bin/sh”抓住了 exec 的替换含义，但不是创建新 PID 或随意丢弃所有进程状态；文件描述符等部分状态可以继承。这套最小方案让交互 Shell 成为 PID 1，退出它会导致系统无法继续正常运行，通常触发 panic。因此这里只用于学习启动流程，尚不是完整发行版的服务管理启动方式。

检查：

```bash
ls -l ~/imx6ull/rootfs-alpine/init
ls -l ~/imx6ull/rootfs-alpine/lib/ld-musl-armhf.so.1
```

### 7.6 /dev/console：控制台入口，不是存放进程的地方

```bash
if [ ! -e ~/imx6ull/rootfs-alpine/dev/console ]; then
    sudo mknod -m 600 ~/imx6ull/rootfs-alpine/dev/console c 5 1
fi
```

`[ ! -e 路径 ]` 判断不存在才创建。`mknod` 创建特殊设备节点，`c` 指字符设备，`5 1` 为控制台主/次设备号；`600` 只允许所有者读写。

内核在启动最初的用户程序前，会尝试打开 `/dev/console` 准备标准输入/输出/错误。提前在包里放入节点，可以在 `/init` 尚未挂载 devtmpfs 时提供这个入口。节点不是串口驱动，也不是保存文字的普通文件。

本次 `console=ttymxc0,115200` 选择串口控制台，`/init` 将 Shell I/O 接到它：

```text
电脑串口窗口 ←→ 开发板串口驱动/控制台 ←→ /dev/console ←→ Shell
```

内核早期日志依靠 earlycon 等内核机制，不依赖这个设备文件；此外，标准输入输出接到控制台不等于已经建立 Shell 作业控制所需的 controlling tty，所以后面仍会出现 job control 提示。

### 7.7 cpio + gzip 打包

```bash
sudo apt install cpio gzip
cd ~/imx6ull/rootfs-alpine

(
    set -o pipefail
    sudo find . -print0 |
        sudo cpio --null -o -H newc --owner=0:0 |
        gzip -9 > ~/imx6ull/rootfs.cpio.gz
)
```

```text
find 列出全部文件路径 → cpio 读取文件并归档 → gzip 压缩 → rootfs.cpio.gz
```

| 语法 | 作用 |
|---|---|
| `( ... )` | 子 Shell；里面的选项设置不影响外层 Shell |
| `set -o pipefail` | 管道前面失败时，也让整个管道返回失败 |
| `find . -print0` | 列出当前目录所有文件，以空字符分隔，支持含空格的文件名 |
| `cpio --null` | 用对应的空字符分隔读取文件名 |
| `-o` | 创建归档 |
| `-H newc` | 使用内核支持的 newc 格式 |
| `--owner=0:0` | 将这份简单根文件系统归档内的文件归属统一为 root |
| `gzip -9` | 用最高级别压缩 |
| `>` | 将输出写入指定文件，同名文件会被覆盖 |

cpio 与 gzip 打包命令逐项解释。*

本次统一 root 归属是最小启动实验的选择；以后扩展多用户、服务账户时要保留所需 UID/GID，不能机械地把所有文件归属都抹成 root。输出文件放在根文件目录之外，也避免把旧归档本身再次打进去。

看到 `10781 块` 是 cpio 的正常统计，不是报错。打包过程出现两遍时只是覆盖生成同名归档，执行一遍即可。cpio/gzip 不编译程序，也不生成设备树。

### 7.8 校验并发布到 TFTP 目录

```bash
gzip -t ~/imx6ull/rootfs.cpio.gz &&
sudo install -m 644 ~/imx6ull/rootfs.cpio.gz /srv/tftp/rootfs.cpio.gz

ls -lh /srv/tftp/rootfs.cpio.gz
```

`gzip -t` 检查 gzip 数据完整性，成功一般不打印文字；这还不是完整的 rootfs 启动能力检查。`install` 将新包复制到服务器目录。`ls` 仅查看文件，**重新打包不会自动更新 `/srv/tftp` 的旧副本**。

### 7.9 原笔记旧地址的说明

原文保留了一段：

```text
tftp 0x80800000 rootfs.cpio.gz
setenv ramdisk_size ${filesize}
```

这是历史记法，不采用为本次最终命令。统一使用第 6 章验证过的布局和本版明确提供的 `tftpboot` 命令，根文件系统镜像放 `0x84000000`。`0x80800000` 靠近低地址内核展开区域，不能未经检查沿用。内核读取的是启动协议传入的镜像地址范围，不是直接按名字读 U-Boot 的 `ramdisk_size`。

### 7.10 重新读一遍完整主线

以下两张图分别保留“详细过程”和“initramfs 所在阶段”的原总结，不能因为内容相近就删掉其中一张。


① 下载Alpine根文件系统（自带/bin/sh、BusyBox、库文件）
        ↓
② 添加 /init 启动脚本，定义挂载目录、后续启动程序
        ↓
③ 创建 /dev/console 设备节点，提供控制台入口
        ↓
④ 打包成 rootfs.cpio.gz（包含init、节点、程序、库）
        ↓
⑤ U-Boot TFTP下载到DDR，传递内核/设备树/ramdisk地址给Linux
        ↓
⑥ Linux内核解压cpio包，在内存生成根目录 /
        ↓
⑦ 内核准备控制台IO，PID=1进程执行 /init
        ↓
⑧ /init 脚本 #!/bin/sh，由/bin/sh解析脚本
        ↓
⑨ 脚本挂载 /proc /sys /dev 等伪文件系统，绑定IO到 /dev/console
        ↓
⑩ exec /bin/sh，shell替换init，PID保持1
        ↓
串口出现命令提示符，可输入ls命令

下载 rootfs 到 PID 1 与 Shell 的详细流程。

① 下载 Alpine 根文件目录
        ↓
② 添加 /init
        ↓
③ 创建 /dev/console
        ↓
④ 用 cpio + gzip 打包，得到 rootfs.cpio.gz
   【这就是我们的外部 initramfs 镜像】
        ↓
⑤ U-Boot 把这个 initramfs 镜像下载到 DDR，告诉内核它的地址和大小
        ↓
⑥ 内核解压并展开 initramfs，里面的文件成为内存中的根文件系统 /
        ↓
⑦ 执行其中的 /init
        ↓
⑧ 挂载目录，启动 Shell
        ↓
出现 ~ #

明确标出 initramfs 所在阶段的流程。

按图理解时记住：内核创建 PID 1，让它运行 `/init`；脚本的 `#!/bin/sh` 指定解释器；最后 `exec /bin/sh` 替换程序但不换 PID。没有把 `/init` 或进程“放进 `/dev/console`”。

内核自动解包的相关源码在 `kernel/init/initramfs.c`：`populate_rootfs()` 调度 `do_populate_rootfs()`，其中用 `unpack_to_rootfs()` 处理外部包，再按压缩格式解压并创建归档中的文件。这些函数已经编译进 zImage，不需要在 Shell 再手动执行一个解包命令。

<a id="s8"></a>
## 8. 启动 Linux 与崩溃修复

### 8.1 第一次现象：停在 Starting kernel

先通过 TFTP 下载三个文件，再执行：

```text
setenv ramdisk_size ${filesize}
setenv bootargs 'console=ttymxc0,115200 rdinit=/init'
bootz 0x82000000 0x84000000:${ramdisk_size} 0x83000000
```

只看到 `Starting kernel ...`，不能马上认定 rootfs 有问题，也不能认为继续等就会好。它表示 U-Boot 已到交接阶段，可能是内核卡住，也可能是输出未建立。

### 8.2 检查串口配置，开启早期日志

Ubuntu：

```bash
grep -E 'CONFIG_(SERIAL_IMX|SERIAL_IMX_CONSOLE|SERIAL_EARLYCON|CMDLINE|ARM_APPENDED_DTB|ARM_ATAG_DTB_COMPAT)' \
  ~/imx6ull/build-kernel/.config
```

`grep -E` 用扩展正则表达式找出这些相关配置行。它只查看，不修改；原笔记“找任意一个”还不完整，应分别确认所需选项是否启用。

```text
CONFIG_SERIAL_IMX=y
CONFIG_SERIAL_IMX_CONSOLE=y
CONFIG_SERIAL_EARLYCON=y
```

开启串口驱动、控制台和早期控制台配置。

重启板子，按键停在 U-Boot，恢复网络并确认 ping，再重新下载。用下面这组启动参数：

```text
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init loglevel=8 ignore_loglevel'
bootz 0x82000000 0x84000000:${ramdisk_size} 0x83000000
```

| 参数 | 作用 |
|---|---|
| `console=ttymxc0,115200` | 正式控制台选择 UART1 对应的 ttymxc0，115200 波特率 |
| `earlycon=ec_imx6q,0x02020000` | 在正式串口驱动就绪前，经本平台 UART1 寄存器地址输出日志 |
| `rdinit=/init` | 运行 initramfs 中的 `/init` |
| `loglevel=8 ignore_loglevel` | 增加可见日志，适合这次排错 |

开启早期日志的启动参数。*

`0x02020000` 是 UART 外设寄存器地址，不是前面 TFTP 下载文件的 DDR 地址。earlycon 的类型及地址应匹配平台。[内核启动参数说明](https://docs.kernel.org/admin-guide/kernel-parameters.html)

### 8.3 早期日志揭示真正错误

日志关键部分：

```text
Unable to handle kernel paging request at virtual address f4000000 when write
PC is at mmioset...
mmioset from imx6_pm_common_init...
Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000000b
```

<a id="img-20261031193912"></a>

![内核写 f4000000 时的地址映射异常](iMX6ULL学习笔记_assets/20261031193912.png)

*原图：`Pasted image 20261031193912.png`。内核写 f4000000 时的地址映射异常。*

这里是内核尝试写一个没有有效映射的虚拟地址，发生异常。调用栈定位到 i.MX6 电源管理初始化，错误发生在内核的 PID 1 初始化路径中，还没运行我们写的用户空间 `/init`。不能仅因出现 `init` 字样就去修改 Shell 脚本。

### 8.4 源码问题：映射初始化被注释

Ubuntu 源码根目录执行：

```bash
sed -n '1,100p' arch/arm/mach-imx/mach-imx6ul.c
nano arch/arm/mach-imx/mach-imx6ul.c
```

`sed -n '1,100p'` 查看前 100 行；`nano` 编辑该文件。它是**内核平台初始化 C 源码**，既不是 `.config`，也不是设备树。

原文件中以下函数被整体注释：

```c
/*
static void __init imx6ul_map_io(void)
{
    imx6_pm_map_io();
    imx_busfreq_map_io();
}
*/
```


*原图：`Pasted image 20261031194102.png`。被注释的 imx6ul_map_io 函数。*

恢复为：

```c
static void __init imx6ul_map_io(void)
{
    imx6_pm_map_io();
    imx_busfreq_map_io();
}
```

同时把 `.map_io` 恢复到机器描述结构内部：

```c
DT_MACHINE_START(IMX6UL, "Freescale i.MX6 Ultralite (Device Tree)")
    .map_io       = imx6ul_map_io,
    .init_irq     = imx6ul_init_irq,
    .init_machine = imx6ul_init_machine,
    .init_late    = imx6ul_init_late,
    .dt_compat    = imx6ul_dt_compat,
MACHINE_END
```

原来注释掉的 `.map_io` 行在 `DT_MACHINE_START` 前面，不能只删 `//` 而留在外面。须放到上面示例的位置。

保留原笔记总结：

```text
没有建立所需内存映射
    ↓
电源管理初始化仍然访问这段地址
    ↓
访问异常，内核崩溃
```

这项修复针对本次修改过的源码。它不是所有 Linux 6.12.3 都有的通用缺陷，也不是任意 `Starting kernel` 问题的通用解法。

### 8.5 重新编译并更新服务器镜像

```bash
export PATH="/usr/bin:/bin:$PATH"
export M4=/usr/bin/m4
hash -r
export CROSS_COMPILE="$HOME/toolchains/arm-gnu-toolchain-11.3.rel1-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf-"

make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-kernel" -j4 zImage

sudo install -m 644 \
  ~/imx6ull/build-kernel/arch/arm/boot/zImage \
  /srv/tftp/zImage
```

使用Ubuntu子弹的构建辅助工具，以及之前验证过的ARM交叉编译器，避开旧host目录里的工具

构建系统会重新编译受影响部分，并完成 zImage 所需的链接和生成步骤；不用先清理整个工程，也不用重新生成配置。本次只修改内核 C 代码，所以设备树和 rootfs 不需要因此重新生成。

### 8.6 修复后完整的 U-Boot 启动命令

以下按本次已知网络和内存布局组织。每次 TFTP 都要确认成功再继续；不要在失败后继续启动旧或不完整内容。

```text
setenv ethaddr 02:12:34:56:78:90
setenv eth1addr 02:12:34:56:78:91
setenv ethact ethernet@2188000
setenv ipaddr 192.168.50.20
setenv serverip 192.168.50.10
setenv netmask 255.255.255.0
ping 192.168.50.10

tftpboot 0x82000000 zImage
tftpboot 0x83000000 imx6ull-14x14-evk-emmc.dtb
tftpboot 0x84000000 rootfs.cpio.gz
setenv ramdisk_size ${filesize}

setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init loglevel=8 ignore_loglevel'
bootz 0x82000000 0x84000000:${ramdisk_size} 0x83000000
```

<a id="s9"></a>
## 9. 成功启动后的日志与挂载

### 9.1 出现欢迎信息与 Shell

```text
Welcome to i.MX6ULL Linux!
Root filesystem: Alpine initramfs
/bin/sh: can't access tty; job control turned off
~ # ls
bin dev etc home init lib media mnt opt proc root run sbin srv sys tmp usr var
```

<a id="img-20261031200649"></a>

![进入 Shell 后的终端、音频、无线数据库日志](iMX6ULL学习笔记_assets/20261031200649.png)

进入 Shell 后的终端、音频、无线数据库日志。*

这证明内核已启动、根文件系统已展开、`/init` 已执行，并进入了交互 Shell。图中还出现音频和无线法规数据库提示，不能把它们漏掉。

| 提示 | 当前能得出的结论 |
|---|---|
| `can't access tty; job control turned off` | Shell 没有获得作业控制所需的控制终端；普通命令仍可运行，Ctrl+Z、fg 等功能受限 |
| `sound-wm8960: deferred probe pending` | 音频设备探测仍未完成；需要进一步查依赖和本板设备树配置 |
| `cfg80211: failed to load regulatory.db` | 无线子系统未能加载法规数据库；不能据此断言板子有可用 Wi-Fi，也不代表整个 Linux 启动失败 |

控制终端与音频延迟探测解释。

第一张日志图中声卡消息在右侧截断并换行，正文保留可确认的内容，不凭截图补出被截掉的字符；完整声卡错误在此前对话日志里为 `snd_soc_register_card failed`。

### 9.2 验证当前内核、启动参数和挂载

在开发板 Linux 的 `~ #` 下执行：

```sh
uname -a
cat /proc/cmdline
cat /proc/mounts
```

本次结果：

```text
Linux (none) 6.12.3 #2 SMP PREEMPT Sat Oct 31 19:49:34 CST 2026 armv7l Linux

console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init loglevel=8 ignore_loglevel

rootfs / rootfs rw,size=164752k,nr_inodes=41188 0 0
proc /proc proc rw,relatime 0 0
sysfs /sys sysfs rw,relatime 0 0
devtmpfs /dev devtmpfs rw,relatime,size=164752k,nr_inodes=41188,mode=755 0 0
devpts /dev/pts devpts rw,relatime,mode=600,ptmxmode=000 0 0
tmpfs /run tmpfs rw,relatime 0 0
```

<a id="img-20261031201023"></a>

![uname、cmdline、mounts 以及后续固件日志](iMX6ULL学习笔记_assets/20261031201023.png)

uname、cmdline、mounts 以及后续固件日志。*

原串口记录里出现 `proc/mountsLinux ...`，是粘贴/终端显示混杂，`proc/mounts` 不是 `uname` 版本字符串的一部分。`#2` 是构建序号；结合编译时间可确认这是重新构建的内核。`armv7l` 表示 ARMv7 小端目标；`(none)` 表示还没设置主机名，不是内核错误。

| 挂载点 | 用途 |
|---|---|
| `/` | 内存中的根文件系统，可读写 |
| `/proc` | 进程、内核等信息接口 |
| `/sys` | 设备、驱动等信息接口 |
| `/dev` | 内核提供的设备节点 |
| `/dev/pts` | 伪终端文件系统 |
| `/run` | 内存中的运行时文件目录 |

| 文件系统 | 挂载点 | 作用 |
|---|---|---|
| rootfs | / | 根目录 / 是内存中的根文件系统，可读写 |
| proc | /proc | 可以通过 /proc 查看进程、内存等信息 |
| sysfs | /sys | 可以通过 /sys 查看设备和驱动信息 |
| devtmpfs | /dev | 内核提供的设备节点挂载 |
| devpts | /dev/pts | 伪终端文件系统挂载 |
| tmpfs | /run | 运行时文件存放在内存中 |

各挂载点的用途。

`size=164752k` 是该挂载显示的容量上限，不是已经用了约 161 MiB。当前根文件系统在 RAM 中，断电后其中的新建和修改内容不会保留。

这张终端图还包含两条后续消息：

```text
imx-sdma 20ec000.dma-controller: external firmware not found, using ROM firmware
random: crng init done
```

前者表明驱动没有找到外部固件而改用 ROM 固件；是否足够满足具体 DMA 功能还要针对外设检查。后者是随机数发生器初始化完成的通知。这些日志不应从整理版中省去，也不能把“还能出现消息”误读成系统再次死机。

<a id="s10"></a>
## 10. 普通 U-Boot 与 U-Boot＋OBC：保留两条源码主线

### 10.1 普通 U-Boot：构建、烧录、运行

原总结以编译期与运行期串成一条线，文字化如下：

```text
U-Boot 源码
    ↓ Makefile：根据配置组织编译、链接、生成镜像
u-boot.imx
    ↓ 电脑执行 dd：写入 SD 卡对应位置
开发板上电，执行 Boot ROM
    ↓ NXP 固化的程序：解析镜像、按 DCD 初始化 DDR、加载并跳转
reset → _main
    ↓ 准备 CPU、栈等 C 运行条件
board_init_f()
    ↓ 早期初始化、确定内存信息、规划运行位置
relocate_code()
    ↓ 将 U-Boot 重定位到规划好的位置
board_init_r()
    ↓ 执行 init_sequence_r，完成后期初始化
run_main_loop()
    ↓
main_loop()
    ↓
bootdelay_process() → autoboot_command()
    ↓ 选择启动命令、倒计时、执行 bootcmd
加载内核/设备树等 → 启动 Linux
    └─ 若中断自动启动，或失败后返回，通常进入 cli_loop()，显示 =>
```

| 文件（相对 U-Boot 源码根目录）          | 函数或内容                                      | 目的                  |
| ---------------------------- | ------------------------------------------ | ------------------- |
| `Makefile`                   | 构建规则                                       | 配置、编译、链接和镜像生成       |
| `arch/arm/cpu/armv7/start.S` | `reset`                                    | ARM 入口和 CPU 初始处理    |
| `arch/arm/lib/crt0.S`        | `_main`                                    | 建立运行环境并串联 C 初始化与重定位 |
| `common/board_f.c`           | `board_init_f()`                           | 重定位前初始化             |
| `arch/arm/lib/relocate.S`    | `relocate_code()`                          | 重定位代码，由启动汇编调用       |
| `common/board_r.c`           | `board_init_r()`、`run_main_loop()`         | 后期初始化并进入主循环         |
| `common/main.c`              | `main_loop()`                              | 组织自动启动和命令行处理        |
| `common/autoboot.c`          | `bootdelay_process()`、`autoboot_command()` | 启动倒计时与命令执行          |

普通 U-Boot 构建与启动的源码调用主线。*

图底部的滚动按钮遮住了函数名的一部分；文字版 `cli_loop()` 来自已核对的源码流程和原讨论，不声称遮挡字符在图上完整可见。路径对应这份被阅读的源码组织，不保证所有 U-Boot 版本都保持相同位置。

### 10.2 OBC 增加了什么

原笔记的两句主线完整保留：

> 设备和地址信息 → 启动命令 → 设备树 → 内核参数。

> U-Boot 完成基础初始化 → OBC 准备项目数据、启动命令、设备树和内核参数 → U-Boot 的 main_loop 使用这些准备结果 → 启动 Linux，或者进入命令行。

OBC 是这位开发者添加的项目框架/扩展代码，不是 U-Boot 或 Linux 必须依赖的通用标准模块。其思路是将项目的存储、加载地址、环境变量和启动策略集中组织，再接回 U-Boot 原有启动流程。

### 10.3 构建时如何连接

```text
OBC 顶层 Makefile
    ↓ 导出 OBC_TOP_DIR，选择平台规则
OBC scripts/imx6ull/uboot.mk
    ↓ 提供配置、调用 U-Boot Makefile
U-Boot Makefile
    ↓ libs-y += $(OBC_TOP_DIR)/bootloader/obcbase/
把 OBC 扩展代码一起编译、链接进 U-Boot
    ↓
生成包含 OBC 的镜像
```

这里实际路径是 `scripts/imx6ull/uboot.mk`，不是单数 `script`。原始启动镜像与 OBC 另行封装的升级包要区分，不能因为后缀或名字相似就按相同偏移烧录。

### 10.4 运行时在哪里插入

```text
Boot ROM → reset → _main → board_init_f → 重定位
    ↓
board_init_r() / init_sequence_r
    ↓ 执行已有步骤后，到作者加入的 do_obcboot
do_obcboot()
    ↓
obc_board_init()
    ↓ 选择 g_imx6ull_board 函数表
imx6ull_board_hw_init()
    ↓ 准备设备编号、介质类型、块设备、DDR 加载地址
imx6ull_board_env_init()
    ↓ 准备 bootcmd、加载命令、网络参数、倒计时
imx6ull_board_fdt_init()
    ↓ 尝试加载、检查设备树，解析项目存储分区信息
imx6ull_board_args_init()
    ↓ 设置 bootargs，准备给 Linux 的参数
返回 board_init_r 初始化列表
    ↓
run_main_loop() → main_loop()
    ↓
bootdelay_process() → autoboot_command()
    ↓ 使用 OBC 准备的环境变量
加载内核，使用设备树和参数，启动 Linux
    └─ 若中断或失败后返回，通常进入命令行
```

| 文件（相对相应项目根目录） | 关键位置 | 目的 |
|---|---|---|
| U-Boot `common/board_r.c` | `do_obcboot` 位于 `run_main_loop` 前的初始化列表中 | 将 OBC 接入运行期 |
| OBC `bootloader/obcbase/obc_base.c` | `do_obcboot()` | 调用 OBC 板级初始化 |
| OBC `bootloader/obcbase/board/board_config.c` | `obc_board_init()` | 选择能力/函数表并调用各初始化步骤 |
| OBC `bootloader/obcbase/board/imx/board_config_imx6ull.c` | 四个 `imx6ull_board_*_init()` | 实现 i.MX6ULL 项目配置 |

U-Boot 加 OBC 的构建和运行主线。*

这张图底部滚动按钮有遮挡，最下方还有被裁掉的下一段标题；不可见部分未凭空补写。完整流程中能确认的内容已转为上面的文字和表格。

“解析项目分区信息”是从设备树里读取项目自定义存储布局，供加载或参数构造使用，并不自动等同于 SD 卡 MBR 分区表。`pstAbi` 是指向板级能力结构体的指针，结构体可能存储地址等配置；它不是 DDR 这块硬件本身。

这章描述的是源码组织和设计意图，不声称当前 OBC 版本每条启动分支已经在板上验证成功。本次成功启动 Linux 使用的是普通 NXP U-Boot 的手工 TFTP 路线。

<a id="s11"></a>
## 11. 总线、设备、驱动与文件描述符

`bus总线.md` 主要由三张图组成。原图全部保留；以下将其中的逻辑展开，并标记摄像头示例的简化之处。

### 11.1 先理解几个对象

| 对象 | 含义 |
|---|---|
| bus | 内核驱动模型中组织设备与驱动、执行匹配及相关管理的总线类型；还会关联具体硬件总线控制 |
| device | 内核表示一个设备的对象，不等于 `/dev` 下的文件 |
| driver | 能操作相应设备的驱动代码 |
| probe | 设备与驱动匹配后，驱动尝试初始化和注册接口的回调；可能失败或延迟 |
| `/dev/xxx` | 用户空间可打开的设备节点，并非每个 device 都必须有一个 |
| fd | 某进程成功 `open()` 后得到的文件描述符整数，在该进程的文件描述符表中使用 |

### 11.2 一般字符设备的简化关系

```text
设备树或其他枚举机制描述设备
    ↓ 相应总线/控制器代码创建设备对象
总线匹配 device 与 driver
    ↓
调用 probe，驱动初始化并注册用户接口
    ↓ 若该设备提供字符设备接口
注册设备号、字符设备对象和相应设备模型信息
    ↓ devtmpfs/用户空间规则或手工 mknod 等方式提供节点
应用 open("/dev/xxx")
    ↓ 成功后返回 fd
应用通过 fd 执行 read/write/ioctl 等操作
```
原始 device、driver、probe、fd 关系。

原图把 `cdev_add` / `register_chrdev` 与“生成 `/dev/xxx`”直接等同，需修正：它们主要负责内核中的字符设备注册，文件系统里的节点还涉及设备模型、devtmpfs/udev 或手动 mknod。并不是调用 `cdev_add` 就自动保证 `/dev` 出现节点。

原总结“fd 是应用打开设备后才有的东西”保留；更准确说是成功打开后，内核为这个进程提供一个可用的 fd，打开失败则返回错误。总线的职责也不只匹配，还参与探测、移除、电源管理等驱动模型操作。

### 11.3 I²C 摄像头 OV5640 示例

原图表达的入门路径：I²C 上有 OV5640 → 与驱动匹配 → probe → 用户打开视频设备采集。

| 步骤  | 说明                                 |
| --- | ---------------------------------- |
| 1   | I2C总线上存在OV5640摄像头设备                |
| 2   | I2C总线做设备和驱动匹配，匹配成功执行probe函数        |
| 3   | probe里面注册V4L2字符设备，生成节点 /dev/video0 |
| 4   | 应用程序打开video0拿到文件描述符，使用V4L2接口采集图像   |

*原始 I2C 摄像头匹配示例。*

<a id="img-20261031155136"></a>

举个例子（I2C 摄像头）

1. i2c bus 上有一个 i2c device（ov5640）
        ↓
2. i2c bus 拿 ov5640 的匹配 id 和 ov5640 driver 对比，匹配成功 → probe
        ↓
3. probe 注册字符设备，生成 /dev/video0
        ↓
4. 应用 open("/dev/video0") → 得到 fd，然后 v4l2 去采集图像

总线只管前面配对；fd 是后面应用打开之后才有的东西。

原始摄像头例子与 fd 总结。

更准确地看，一般摄像头链路需要区分控制通路与图像数据通路：

```text
I²C 控制通路：SoC I²C 控制器 ↔ OV5640 寄存器配置
图像数据通路：OV5640 → 接收/采集控制器 → 内存缓冲区
```

OV5640 传感器驱动通常注册为 V4L2 sub-device，与接收端/采集端驱动协作。不是“OV5640 的 I²C probe 单独、必然生成 `/dev/video0`”。最终视频节点由媒体采集链路的相应驱动注册，编号也未必为 0。应用再打开实际视频节点，借助 V4L2 接口配置和采集。[Linux V4L2 子设备官方说明](https://docs.kernel.org/driver-api/media/v4l2-subdev.html)

这里是原理示例，不代表本次板子的摄像头已经适配或测试通过。

<a id="s12"></a>
## 12. 重新上电、成果保存与知识修正

### 12.1 断电以后哪些要重新做

| 内容 | 所在位置 | 断电后 |
|---|---|---|
| U-Boot 镜像 | SD 卡 | 保留，按 SD 模式上电可再启动 |
| 内核、DTB、initramfs 下载内容 | DDR | 不应依赖保留，重新加载 |
| 正在运行的内存根文件系统改动 | RAM | 丢失 |
| 构建产物和 TFTP 文件 | Ubuntu 磁盘 | 保留，不用为重启而重新编译 |
| 用 `ip addr` 临时配置的地址 | Ubuntu 当前运行状态 | VM 重启后需检查并恢复 |
| 没有保存的 U-Boot 环境变量 | 当前 U-Boot 会话 | 板子复位后需恢复 |

只重启电脑而开发板未断电、未复位时，板子可能仍在运行，不必因为电脑重启就断言 DDR 已清空。当前方案叫“网络加载后启动”，不是每次都“烧录 DDR”。

### 12.2 保存代码与实验成果

值得保留的内容包括 `.config`、修改过的 `mach-imx6ul.c`、板级 DTS、`/init`、构建与启动命令、成功及失败日志。完整复现还需要记录原始源码仓库及版本；单独一个 DTS 通常依赖其他 DTSI，单独一个修改过的 C 文件也不是完整内核。

原笔记在这里链接了“其他知识 → git学习笔记.md”，并保留 SSH 密钥命令：

```bash
ssh-keygen -t rsa -b 4096 -C "你的邮箱@example.com"
cat ~/.ssh/id_rsa.pub
```

第一条生成用于 SSH 认证的密钥对，`-C` 添加标注；第二条查看公钥，可用于配置代码托管平台的 SSH 认证。`id_rsa.pub` 是公钥，`id_rsa` 是私钥，不能把私钥提交进仓库。生成密钥不是 `git init`，也不会自动把代码上传。

原文的另一个知识导航“linux性能启动笔记 → imx6ull启动调试笔记.md”一并保留。上述两个被提到的笔记并未包含在本次指定的四份文件中，本文没有擅自声称读取其内容。

### 12.3 原笔记中容易混淆的说法集中对照

| 原记录或图中说法 | 整理后的准确表述 |
|---|---|
| 设置 CROSS_COMPILE 就“进行交叉编译” | 它只选择工具前缀，真正构建由 make 目标触发 |
| 用 GCC 编译 Makefile | make 读取 Makefile，再调用 GCC 等工具处理源码 |
| 第二网口用第二次 `setenv ethaddr` | 应是 `eth1addr`，避免覆盖第一网口 |
| ip link up 就是网卡上电 | 它设置管理状态，链路是否可用还需检查 |
| PATH 优先系统目录是“从刚下载的地方找” | 是优先使用 Ubuntu `/usr/bin`、`/bin` 下的辅助工具 |
| raw initramfs 完全没有头部 | gzip/cpio 有自身格式头，但 bootz 仍要求显式传 raw initramfs 的长度 |
| 内核直接读取 ramdisk_size 变量 | U-Boot 使用变量值，按启动协议传入镜像范围 |
| rootfs 固定放在 `0x80800000` | 这是原笔记旧示例；本次成功布局采用 `0x84000000` |
| 根目录 `/` 是一个固定 DDR 地址 | `/` 是路径；展开后的文件由内核分配管理内存 |
| PID 1 是 `/init` 脚本创建的 | 内核创建 PID 1，脚本是其执行内容之一 |
| `/dev/console` 里装进程 | 它是设备节点，用作输入输出入口 |
| EABI5 一项就证明 hard-float、ARMv7 | 需要结合 ABI attributes、架构信息及发行包目标判断 |
| musl 完全没有兼容问题 | 本次不依赖 glibc，但仍有架构、ABI、库依赖等条件 |
| OV5640 probe 必然创建 `/dev/video0` | 需要整个 V4L2 采集链路协作，传感器通常为子设备 |
| MBR/GPT 必须分别搭配 BIOS/UEFI | 这是常见 PC 配置关系，不是普遍强制规则 |
| 改 mach-imx6ul.c 是改设备树 | 它是内核 C 源码，修改后重编 zImage |

这些修正不删除历史记录。附录完整保留原文，便于理解错误最初来自哪里，也避免日后把旧尝试当作最终配置。

### 12.4 本次达成的完整主线

```text
选择 ARM32 交叉工具链
    ↓
配置并编译 U-Boot → 将 u-boot.imx 写入 SD 卡并校验
    ↓
从 SD 卡启动 U-Boot → 配置有线网络、Ubuntu TFTP 服务
    ↓
编译 Linux zImage + 编译板级 DTB
    ↓
下载 Alpine 根文件系统 → 添加 /init 和控制台节点 → cpio/gzip 打包
    ↓
TFTP 将三个文件分别加载进 DDR → bootz
    ↓
earlycon 揭示映射崩溃 → 恢复内核 .map_io → 重编 zImage
    ↓
重新下载启动 → 执行 /init → 进入开发板 Linux Shell
```

后续待完善项仍包括控制终端的作业控制、音频及其他外设设备树适配、固件/数据库需求，以及是否改用持久化根文件系统或自动启动。本文记录的是当前已经验证的阶段，不把待办写成完成。




<a id="appendix"></a>




