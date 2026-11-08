
Alpine 根目录 → 制作 initramfs
                      ↓
                配置进内核构建
                      ↓
        生成内置 initramfs 的 zImage
                      ↓
       写入 eMMC 预留的内核区域
                      ↓
U-Boot 读取这个 zImage 和独立 DTB 到 DDR
                      ↓
      bootz 内核地址 - 设备树地址
                      ↓
内核展开自身携带的 initramfs → 执行 /init

1.检查压缩文件是否损坏
gzip -t ~/imx6ull/rootfs.cpio.gz

2.解压并保存成另一个文件
gzip -dc ~/imx6ull/rootfs.cpio.gz > ~/imx6ull/rootfs-builtin.cpio

| 部分                    | 含义                     |
| --------------------- | ---------------------- |
| `-d`                  | 解压 gzip                |
| `-c`                  | 将解压结果输出到标准输出，同时保留原文件   |
| `>`                   | 把输出写入右边的文件；同名文件已存在时会覆盖 |
| `rootfs-builtin.cpio` | 保存解压结果的新文件名            |
rootfs.cpio.gz
    ↓ gzip 解压：去掉压缩层
rootfs-builtin.cpio

3.设置交叉工具链

export PATH="/usr/bin:/bin:$PATH"
**先找 /usr/bin
    ↓
再找 /bin
    ↓
再找原来 PATH 中的其他目录**

export CROSS_COMPILE="$HOME/toolchains/arm-gnu-toolchain-11.3.rel1-x86_64-arm-none-linux-gnueabihf/bin/arm-none-linux-gnueabihf-"

hash -r (清除当前 Shell 缓存的命令路径，让后续命令重新查找)

4.配置内置 initramfs(得在内核的config里面设置假如initramfs才可以加入)

cd ~/imx6ull/imx6ull_sdk_source/kernel

scripts/config --file "$HOME/imx6ull/build-kernel-builtin/.config" \
  --enable BLK_DEV_INITRD \
  --set-str INITRAMFS_SOURCE "$HOME/imx6ull/rootfs-builtin.cpio"
  
  (`--file xxx/.config`：指定要修改的配置文件)
- `--enable CONFIG_XXX` → 布尔型，设 `=y`（开关）
- `--disable CONFIG_XXX` → 布尔型，注释掉 `# CONFIG_XXX is not set`
- `--set-str CONFIG_XXX "abc"` → 字符串类型，写成 `CONFIG_XXX="abc"`

再让内核整理配置及依赖：
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-kernel-builtin" olddefconfig   
   (make + 构建参数 + 任务目标)
make ... olddefconfig
    ↓
读取已有 .config 和 Kconfig 配置规则
    ↓
处理依赖，为缺失选项采用默认值
    ↓
写回整理后的 .config

grep -E '^CONFIG_(BLK_DEV_INITRD|INITRAMFS_SOURCE)=' \
  ~/imx6ull/build-kernel-builtin/.config(拿出来看看)
(CONFIG_BLK_DEV_INITRD=y
CONFIG_INITRAMFS_SOURCE="/home/xq999/imx6ull/rootfs-builtin.cpio")

5.编译新的内核
make ARCH=arm CROSS_COMPILE="$CROSS_COMPILE" \
  O="$HOME/imx6ull/build-kernel-builtin" -j4 zImage
  
make ... zImage
    ↓
根据 .config 确定启用哪些功能
    ↓
调用编译器、汇编器、链接器等工具
    ↓
把指定的 initramfs 纳入构建
    ↓
生成 ARM 内核镜像 zImage

6.查看生成文件大小
stat -c '%n：%s 字节' \
  ~/imx6ull/build-kernel-builtin/arch/arm/boot/zImage

~/imx6ull/build-kernel-builtin/arch/arm/boot/zImage  
/home/xq999/imx6ull/build-kernel-builtin/arch/arm/boot/zImage：12840240
12840240个字节 大约12.25Mib
没有超过32m

1.在 Ubuntu 终端，把新内核放进 TFTP 目录
sudo install -m 644 \
  ~/imx6ull/build-kernel-builtin/arch/arm/boot/zImage \
  /srv/tftp/zImage-builtin
确认 Ubuntu 有线网络地址和 TFTP 服务
sudo ip link set ens37 up
sudo ip addr replace 192.168.50.10/24 dev ens37
sudo systemctl start tftpd-hpa
2.重启开发板，按空格进入 U-Boot 的 `=>`
setenv ethaddr 02:12:34:56:78:90
setenv eth1addr 02:12:34:56:78:91
setenv ethact ethernet@2188000
setenv ipaddr 192.168.50.20
setenv serverip 192.168.50.10
setenv netmask 255.255.255.0

ping 192.168.50.10

3.下载新内核，读取现有设备树
tftpboot 0x82000000 zImage-builtin
读取设备树
mmc dev 1 0
mmc read 0x83000000 0x800 0x5a

4.设置内置 initramfs 的启动参数
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init'
bootz 0x82000000 - 0x83000000

5.进入 `~ #` 后验证
cat /proc/cmdline
cat /proc/mounts

cat /etc/default/tftpd-hpa
里面这一行决定 TFTP 从哪里提供文件：
```
TFTP_DIRECTORY="/srv/tftp"
```

|服务|配置文件|提供的目录|
|---|---|---|
|TFTP：下载内核等文件|`/etc/default/tftpd-hpa`|`/srv/tftp`|
|NFS：远程访问根文件系统|`/etc/exports`|`/srv/nfs/imx6ull-rootfs`|
