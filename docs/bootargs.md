给 Linux 内核的启动参数”。
它告诉内核：
日志从哪里输出、
根文件系统在哪里、
网络如何配置、
最后运行哪个启动程序。

| 变量         | 谁使用      | 决定什么                     |
| ---------- | -------- | ------------------------ |
| `bootcmd`  | U-Boot   | 执行哪些命令，例如读取镜像、执行 `bootz` |
| `bootargs` | Linux 内核 | 内核启动后如何配置系统              |
全自动启动时怎么用到它
```
setenv bootcmd 'run start_emmc'
```
U-Boot 倒计时结束
    ↓
执行 bootcmd，也就是 run start_emmc
    ↓
读取内核、设备树，以及需要的 initramfs 到 DDR
    ↓
执行 bootz
    ↓
U-Boot 将当前 bootargs 传给内核
（我们使用设备树启动时，通常放在 DTB 的 /chosen/bootargs 属性中）
    ↓
Linux 解析参数，配置控制台、挂载根文件系统、运行启动程序

1.单独加载 `rootfs.cpio.gz`
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init'
`bootargs`：使用哪个串口，并运行 initramfs 中的 `/init`。
bootz 0x82000000 0x84000000:0x308d68 0x83000000
`bootz` 的参数：内核、外置 initramfs、设备树在哪里。

2.当前实验：initramfs 已经内置到内核
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 rdinit=/init'
bootz 0x82000000 - 0x83000000
区别在于**不用提供外置 initramfs**，内核本身已经包含它

3.NFS 根文件系统
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 root=/dev/nfs rw nfsroot=192.168.50.10:/srv/nfs/imx6ull-rootfs,vers=3,tcp,nolock ip=192.168.50.20:192.168.50.10::255.255.255.0:imx6ull:eth1:off init=/init'
```
root=/dev/nfs
    → 根目录使用 NFS
nfsroot=服务器IP:共享目录,选项
    → 到 Ubuntu 的指定目录找根文件系统
ip=...
    → 在挂载根目录之前配置 Linux 网络
init=/init
    → 根目录挂载完成后运行 /init
```
bootz 0x82000000 - 0x83000000

4.eMMC 的 ext4 根文件系统
setenv bootargs 'console=ttymxc0,115200 earlycon=ec_imx6q,0x02020000 root=/dev/mmcblk1p1 rootfstype=ext4 rootwait rw init=/init'
```
root=/dev/mmcblk1p1
    → 使用 eMMC 第一个分区作为根目录
rootfstype=ext4
    → 根文件系统格式是 ext4
rootwait
    → 等待根设备被内核识别
rw
    → 根文件系统可读写
init=/init
    → 根目录挂载完成后运行 /init
```

|参数|作用|
|---|---|
|`console=ttymxc0,115200`|指定正常控制台的串口及波特率|
|`earlycon=ec_imx6q,0x02020000`|开启早期串口日志，便于观察启动问题|
|`rdinit=/init`|指定初始 RAM 根文件系统中的启动程序|
|`init=/init`|指定正常根文件系统中的启动程序|
|`root=...`|指定要挂载的根设备或 NFS|
|`loglevel=8 ignore_loglevel`|我们排查问题时使用的详细控制台日志参数|
1. 用哪个串口观察系统？
   → console=...

2. 根文件系统从哪里来？
   → initramfs / eMMC ext4 / NFS

3. 为访问根文件系统，还需要什么？
   → ext4：rootfstype、rootwait
   → NFS：nfsroot、ip
   → initramfs：正确提供或内置归档

4. 根目录准备好后运行什么？
   → rdinit=/init 或 init=/init

配置后可在 U-Boot 检查：
```
printenv bootargs
```
启动后在 Linux 检查实际收到的参数：
```
cat /proc/cmdline
```