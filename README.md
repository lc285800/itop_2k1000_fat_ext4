# iTOP 龙芯 2K1000：FAT32 + ext4 SATA 启动

将 Buildroot 从内存根文件系统迁移为真实 SATA 根系统。2026-10-27 已确认 ext4 根启动成功，用户补齐 `saveenv` 后报告启动问题解决。

## 下载

[v1.0.0 Release](https://github.com/lc285800/itop_2k1000_fat_ext4/releases/tag/v1.0.0) 提供：

| 文件 | 用途 |
|---|---|
| `itop-2k1000-fat32-ext4.img.gz` | 完整 SATA 盘镜像，直接写盘 |
| `vmlinuz` | 已使用的 Linux 3.10.0 MIPS64 小端内核 |
| `rootfs.tar.gz` | 展开到 ext4 的 Buildroot 2021.02.3 根文件系统 |
| `boot.cfg`、`pmon-env.txt` | 最终启动配置与 PMON 命令 |
| `SHA256SUMS` | 下载校验值 |

发布文件已移除 SSH 主机密钥和 Shell 历史，首次启动自动生成主机密钥。完整盘发布版据成功运行的系统重建并做离线检查，未另行上板测试；多轮冷热启动稳定性需后续验证。

## 布局

- MBR，磁盘大小 16,013,942,784 字节；写入目标盘不能小于此容量。
- 第一分区：512 MiB FAT32，卷标 **LS2KBOOT**，存放 `/boot/vmlinuz` 和 `/boot/boot.cfg`。
- 第二分区：ext4，PARTUUID **2a102626-02**，直接挂载为 `/`。
- 系统运行后启动分区只读挂载到 `/mnt/bootdisk`，静态 IP 为 `192.168.2.140/24`。

## 写盘和启动

在 Ubuntu 下载 Release 附件后校验：

```sh
sha256sum -c SHA256SUMS
```

确认目标盘及备份，卸载其全部分区。以下 `/dev/sdX` 必须替换成实际 SATA 盘，写入会覆盖整盘：

```sh
gzip -dc itop-2k1000-fat32-ext4.img.gz | sudo dd of=/dev/sdX bs=4M conv=fsync status=progress
sync
```

板卡上电进入 PMON，执行下面四条命令，`set append` 为一整行：

```text
set bootdev /dev/fs/fat@wd0
set al1 /dev/fs/fat@wd0/boot/vmlinuz
set append "console=tty console=ttyS0,115200 root=PARTUUID=2a102626-02 rootfstype=ext4 rootwait rw init=/sbin/init"
saveenv
```

等 `saveenv` 完成再断电上电。**该版 PMON 的 `set` 只修改内存，`saveenv` 才保存到 Flash。** 恢复 SATA 镜像不会自动修改 PMON 环境。

启动后用 `cat /proc/cmdline`、`df -h /`、`cat /proc/mounts` 确认根为 ext4。已验证系统日志为 `VFS: Mounted root (ext4 filesystem) on device 8:2.`。

## 手动部署与配置

已有正确分区时，可在 Linux 下以 root 权限将 `rootfs.tar.gz` 解包到 ext4 分区根目录，保留权限和所有者；将 `vmlinuz` 和 [boot.cfg](configs/boot.cfg) 放到 FAT32 的 `boot/` 目录。换盘后检查 PARTUUID 并同步修改 cfg 和 PMON 参数。正常启动不加载 cpio，也不需要重编内核或重刷 PMON。

最终配置：[boot.cfg](configs/boot.cfg)、[PMON 命令](configs/pmon-env.txt)。

## 避免重踩的问题

1. FAT 卷标不要用 `BOOT`：它与 `boot` 目录同名，旧 PMON FAT 驱动会找错目录；使用 `LS2KBOOT`。
2. `root="(null)"` 表示未传入根设备参数。若自动启动仍打印旧 `g ... initcall_debug=1 ...`，应检查 PMON 的 `append` 和 `saveenv`，而非反复修改 cfg。
3. 需要手动启动时使用 `load`、带完整根参数的 `g`；不要用 `boot`，也不要对旧 PMON 使用 `bl --help`。

仓库只包含最终文档和配置。镜像、原始系统备份、私密连接文件及完整调试历史不进入 Git 历史。
