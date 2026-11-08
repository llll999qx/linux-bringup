
1.可以把环境变量理解成 U-Boot 的**设置项和命令便条**：

| 变量或命令       | 作用                          |
| ----------- | --------------------------- |
| `bootargs`  | 给 Linux 的参数，例如串口、执行 `/init` |
| `bootcmd`   | 倒计时结束后，U-Boot 自动执行的命令       |
| `bootdelay` | 自动启动前等待多少秒                  |
| `setenv`    | 修改当前内存中的设置                  |
| `run 名称`    | 执行某个变量里保存的命令                |
| `saveenv`   | 将设置保存到持久存储，供下次启动使用          |
setenv hello 'echo Hello'
run hello
设置一个环境变量叫hello，然后run hello可以显示出Hello

2.配置从 eMMC 自动加载

setenv load_dtb 'mmc read 0x83000000 0x800 0x5a'
setenv load_kernel 'mmc read 0x82000000 0x1000 0x49e3'
setenv load_rootfs 'mmc read 0x84000000 0x11000 0x1847'
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init'
- `0x02020000`：**硬件手册给的 UART1 寄存器地址**
- `ec_imx6q`：**Linux 内核早期串口驱动自己起的名字标识**
setenv start_emmc 'if mmc dev 1 0 && run load_dtb && run load_kernel && run load_rootfs; then bootz 0x82000000 0x84000000:0x308d68 0x83000000; else echo Image load failed; fi'

setenv bootcmd 'run start_emmc'
setenv bootdelay 3


bootcmd：U-Boot 上电倒计时结束、没人按回车打断，**自动执行 bootcmd 里面的命令**。

```
上电 → U-Boot → 通过 TFTP 下载内核、设备树、initramfs 到 DDR → 启动 Linux
```

**1.在 Ubuntu 终端执行：准备网络和文件**
sudo ip link set ens37 up
sudo ip addr replace 192.168.50.10/24 dev ens37

sudo install -m 644 ~/imx6ull/build-kernel/arch/arm/boot/zImage /srv/tftp/zImage
（复制 zImage 到 tftp 目录，并且把文件权限设成 644，没有安装任何软件）

sudo install -m 644 ~/imx6ull/build-kernel/arch/arm/boot/dts/nxp/imx/imx6ull-14x14-evk-emmc.dtb /srv/tftp/imx6ull-14x14-evk-emmc.dtb

sudo install -m 644 ~/imx6ull/rootfs.cpio.gz /srv/tftp/rootfs.cpio.gz

sudo systemctl start tftpd-hpa （启动 TFTP 服务，sudo 是需要管理员权限）
systemctl is-active tftpd-hpa （询这个服务当前状态，输出 `active` 代表正在跑；`inactive` 就是没启动）
显示 `active` 就表示服务正在运行

**2.在开发板 U-Boot 的 `=>` 下执行：设置网络**
setenv ethaddr 02:12:34:56:78:90 （设置 U-Boot 中 `eth0` 的 MAC 地址）
setenv eth1addr 02:12:34:56:78:91 （设置 U-Boot 中 `eth1` 的 MAC 地址）
setenv ethact ethernet@2188000   
setenv ipaddr 192.168.50.20（板子自己的 IP 地址）
setenv serverip 192.168.50.10（电脑的ip）
setenv netmask 255.255.255.0

ping 192.168.50.10（ping 电脑Ubuntu）

net list
eth1 : ethernet@20b4000 02:12:34:56:78:91
eth0 : ethernet@2188000 02:12:34:56:78:90 active

| U-Boot 接口 | 设备名称               | 配置 MAC 的变量 |
| --------- | ------------------ | ---------- |
| `eth0`    | `ethernet@2188000` | `ethaddr`  |
| `eth1`    | `ethernet@20b4000` | `eth1addr` |
**Linux怎么知道要给什么文件夹的内容给 u-boot呢**

U-Boot 向 192.168.50.10 请求文件“zImage”
                  ↓
Ubuntu 的 tftpd-hpa 收到请求
                  ↓
根据服务配置，到 /srv/tftp 目录寻找 zImage
                  ↓
把 /srv/tftp/zImage 的内容通过网线发给开发板
                  ↓
U-Boot 把收到的数据放进 DDR 的 0x82000000

**1.Ubuntu 怎么知道去 `/srv/tftp` 找***
xq999@xq999-virtual-machine:**~$ cat /etc/default/tftpd-hpa**
 /etc/default/tftpd-hpa
TFTP_USERNAME="tftp"
TFTP_DIRECTORY=**"/srv/tftp"**
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure"

**2.U-Boot 怎么知道向哪台电脑请求**
setenv serverip 192.168.50.10

**3.这条下载命令分别是什么意思**
tftpboot 0x82000000 zImage

**3.定义下载和启动命令**
setenv net_kernel 'tftpboot 0x82000000 zImage'（内核）

setenv net_dtb 'tftpboot 0x83000000 imx6ull-14x14-evk-emmc.dtb' （设备树）

setenv net_rootfs 'if tftpboot 0x84000000 rootfs.cpio.gz; then setenv ramdisk_size ${filesize}; else false; fi' （根文件系统）

setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init'（内核启动参数）

setenv start_net 'if run net_kernel && run net_dtb && run net_rootfs; then bootz 0x82000000 0x84000000:${ramdisk_size} 0x83000000; else echo Network boot failed; fi'

**4.先保存这些新命令，手动测试**
saveenv
run start_net （试试）

**5.验证成功后，改成上电自动网络启动**
setenv bootcmd 'run start_net' （设置上电uboot执行run start_net自己执行）
setenv bootdelay 3
saveenv
若想改成emmc自启动
setenv bootcmd 'run start_emmc'
saveenv


xq999@xq999-virtual-machine:**~$ grep -E '^CONFIG_(ENV_|SYS_MMC_ENV)' \
  ~/imx6ull/build-uboot-emmc-test/.config**
CONFIG_ENV_SOURCE_FILE=""
CONFIG_ENV_SIZE=0x2000（为环境变量预留 **8 KiB**）
CONFIG_ENV_OFFSET=0xE0000（从存储区域开头偏移 **896 KiB** 的位置保存）
CONFIG_ENV_SUPPORT=y
CONFIG_ENV_CALLBACK_LIST_STATIC=""
CONFIG_ENV_OVERWRITE=y
CONFIG_ENV_MIN_ENTRIES=64
CONFIG_ENV_MAX_ENTRIES=512
CONFIG_ENV_IS_IN_MMC=y
CONFIG_ENV_RELOC_GD_ENV_ADDR=y
CONFIG_ENV_MMC_DEVICE_INDEX=1
CONFIG_ENV_MMC_EMMC_HW_PARTITION=0（使用 **用户区**，不是 boot0）
CONFIG_ENV_VARS_UBOOT_RUNTIME_CONFIG=y

```
eMMC
├─ boot0：U-Boot 启动镜像
│
└─ 用户区
   ├─ 896 KiB ～ 904 KiB：保存的环境变量
   ├─ 1 MiB 起：设备树
   ├─ 2 MiB 起：内核
   └─ 34 MiB 起：initramfs
```

```
起点：
0xE0000 = 917504 字节
917504 ÷ 1024 = 896 KiB

长度：
0x2000 = 8192 字节
8192 ÷ 1024 = 8 KiB

结束边界：
896 KiB + 8 KiB = 904 KiB
```


**1.boot指令是u-boot自带的 当停在u-boot指令界面的时候 boot指令是做bootcmd的工作**

**2.TFTP简单文件传输协议，只用来下载文件**

**3.NFS网络文件系统，TCP，相当于把电脑上的文件夹直接挂载到板子上**

```
U-Boot 加载内核和设备树
          ↓
     选择一种根文件系统方案
          │
          ├─ 方案一：initramfs
          │   把 rootfs.cpio.gz 加载到 DDR
          │   Linux 展开它，在内存中运行根文件系统
          │
          ├─ 方案二：NFS
          │   根目录放在 Ubuntu 上
          │   Linux 通过网线访问它
          │
          └─ 方案三：ext4
              根目录放在 TF / eMMC 的 ext4 分区里
              Linux 挂载这个分区使用
```
**1.Ubuntu 终端执行，检查内核支持**
```
grep -E '^(CONFIG_(NFS_FS|NFS_V3|ROOT_NFS|IP_PNP)=|# CONFIG_(NFS_FS|NFS_V3|ROOT_NFS|IP_PNP) is not set)' \
  ~/imx6ull/build-kernel/.config
```
 （ CONFIG_NFS_FS=y               ← 匹配左边 CONFIG_NFS_FS=
 CONFIG_ROOT_NFS is not set  ← 匹配右边 # CONFIG_ROOT_NFS is not set）
 在.config的文件里面找CONFIG_NFS_FS=和# CONFIG_

| 输出                  | 意思                    |
| ------------------- | --------------------- |
| `CONFIG_IP_PNP=y`   | 内核启动早期能够配置 IP 等网络参数   |
| `CONFIG_NFS_FS=y`   | 内核支持访问 NFS 文件系统       |
| `CONFIG_NFS_V3=y`   | 支持 NFS 第 3 版协议        |
| `CONFIG_ROOT_NFS=y` | 支持把 NFS 目录作为根文件系统 `/` |
看挂载 NFS 根目录后该如何启动用户空间
```
cat ~/imx6ull/rootfs-alpine/init
```

1. 创建 NFS 根目录，复制原来的文件
sudo mkdir -p /srv/nfs/imx6ull-rootfs （`-p`：递归创建多级目录）
sudo cp -a ~/imx6ull/rootfs-alpine/. /srv/nfs/imx6ull-rootfs/ （做根文件系统拷贝必须用 `-a`）
2. 修改新目录中的欢迎文字
sudo sed -i 's/Root filesystem: Alpine initramfs/Root filesystem: Alpine NFS/' /srv/nfs/imx6ull-rootfs/init
`sed -i`：直接修改文件（-i 是 in-place，原地改，不输出到屏幕）
3. 安装 Ubuntu 的 NFS 服务
sudo apt update
sudo apt install nfs-kernel-server
通过网络，把 Ubuntu 上指定的目录提供给开发板访问

sudo cat /etc/exports
它告诉 Ubuntu 的 NFS 服务：共享哪个目录、允许哪台设备访问、允许怎样访问。

1. 在 Ubuntu 打开文件：
sudo nano /etc/exports
/srv/nfs/imx6ull-rootfs 192.168.50.20(rw,sync,no_subtree_check,no_root_squash)
（在末尾新加一行，**开头不要加 `#`**）

|内容|意思|
|---|---|
|`/srv/nfs/imx6ull-rootfs`|共享这个根文件系统目录|
|`192.168.50.20`|允许这个 IP 的开发板访问|
|`rw`|允许读取和写入|
|`sync`|写操作提交到存储后再回复客户端|
|`no_subtree_check`|不进行子目录树检查|
|`no_root_squash`|保留开发板 root 用户的权限，供这个根文件系统实验使用|
2. 让配置生效并检查：
sudo exportfs -ra （重新加载共享配置）
sudo systemctl restart nfs-kernel-server  （重启 NFS 服务）
sudo exportfs -v （查看实际生效的共享目录和权限）

**Ubuntu 允许开发板 `192.168.50.20` 读写 `/srv/nfs/imx6ull-rootfs`**
