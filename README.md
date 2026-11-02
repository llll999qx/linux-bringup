# i.MX6ULL Linux 启动学习记录

本项目保存我的 i.MX6ULL 开发板启动实验、内核修改、配置和学习笔记。

## 完整学习笔记

[阅读：i.MX6ULL 学习笔记](docs/iMX6ULL学习笔记_完整整理.md)

## 仓库目录

- kernel：内核配置和修改过的源码
- device-tree：设备树源码
- rootfs：根文件系统启动脚本
- docs：完整学习笔记及图片

## 当前进度

已完成 U-Boot 从 SD 卡启动，通过 TFTP 加载内核、设备树和 initramfs，并进入 Linux Shell。

部分外设仍需进一步适配和验证。
