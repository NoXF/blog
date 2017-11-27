准备好 u 盘，下载 Arch 镜像，用 rufus 烧录，选择 dd 模式

## 安装 Arch

### 网络
```
# dhcpcd
```

### 挂载分区

```
选择要安装 arch 的分区
# mkfs.ext4 /dev/sda3
# mount /dev/sda3 /mnt

fdisk -l
找到 windows 安装好的 EFI 分区
# mkdir -p /mnt/boot/EFI
# mount /dev/nvme0n1p1 /mnt/boot/EFI
```

### 设置镜像
```
编辑 /mirrorlist 把国内的镜像放在前面
$ vim /etc/pacman.d/mirrorlist
```

### 安装 Arch
```
# pacstrap -i /mnt base base-devel
# genfstab -U -p /mnt >> /mnt/etc/fstab
```

## 系统配置

```
# arch-chroot /mnt /bin/bash
# pacman -S vim
# echo archgnome > /etc/hostname
# vim /etc/hosts 添加 archgnome 到 127.0.0.1 和 ：：1 这两条记录后执行下面一句
# mkinitcpio -p linux
设置密码
passwd
```

### 设置时区
```
# ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
# hwclock --systohc --utc
```

### Locale
```
# vim /etc/locale.gen
en_US.UTF-8 UTF-8
zh_CN.UTF-8 UTF-8
zh_TW.UTF-8 UTF-8
执行
# locale-gen
# echo LANG=en_US.UTF-8 > /etc/locale.conf
```

### grub安装
```
# pacman -S grub efibootmgr
# grub-install --target=x86_64-efi --efi-directory=/boot/EFI --bootloader-id=arch_grub --recheck
更新 microcode
# pacman -S intel-ucode
# grub-mkconfig -o /boot/grub/grub.cfg
```

### 注册 window boot 到 EFI

```
# grub-probe --target=fs_uuid /boot/EFI/EFI/Microsoft/Boot/bootmgfw.efi >> /root/fs_uuid
# grub-probe --target=hints_string /boot/EFI/EFI/Microsoft/Boot/bbootmgfw.efi >> /root/hints_string
如果获取 hints_string 命令失效，可以直接在 grub.cfg 中找到 hints_string
```

在 /boot/grub/grub.cfg 中找到下面这段
```
### BEGIN /etc/grub.d/10_linux ###
...
### END /etc/grub.d/10_linux ###
```
在中间加上
```
menuentry "Microsoft Windows 10 x86_64 UEFI-GPT" {
    insmod part_gpt
    insmod fat
    insmod search_fs_uuid
    insmod chain
    search --fs-uuid --set=root \$hints_string \$fs_uuid
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}
```

`hints_string` 和 `fs_uuid` 用上面 grub-probe 取到的替换，如果 hints_string 取不到，可以直接在 grub.cfg 文件中找到

### 重启
```
# exit
# umount -R /mnt
# reboot
```

## 安装显卡驱动

添加用户
```
useradd -m -g users -G wheel -s /bin/bash ${username}
passwd ${username}
```
添加用户权限
```
在 visudo 中增加 ${username} ALL=(ALL) ALL
```

安装显卡驱动
```
lspci | grep VGA 查看显卡，这里安装 nvidia 驱动
pacman -S nvidia nvidia-utils
```
由于系统自带 nouveau 驱动，安装官方驱动需要将 nouveau 禁掉，将 nouveau 写入 /etc/modprobe.d/ blacklist
```
/etc/modprobe.d/blacklist.conf ，在文件后面加入blacklist nouveau

或者

# vim nvidia-installer-disable-nouveau.conf
blacklist nouveau
options nouveau modeset=0
```
通过 lspci -k | grep -A 2 -i "VGA" 可能会发现 `Kernel driver in use: nouveau` 还是使用的 nouveau 驱动，这时需要把 nomodeset 加到 kernel line 里
```
在 /boot/grub/grub.cfg 中找到 "linux /boot/vmlinuz-linux root=${fs_uuid} rw quiet"

改为 "linux /boot/vmlinuz-linux root=${fs_uuid} rw nomodeset quiet"

# reboot

检查 nouveau 确保没有被加载！
# lsmod | grep nouveau

检查 nvidia 驱动是否安装正确
# nvidia-smi
```

## 安装 Gnome 图形界面
```
pacman -S xorg xorg-server xorg-xinit xorg-apps
pacman -S gnome gnome-extra
systemctl enable gdm.service
systemctl enable NetworkManager.service
```

## 启动
```
# arch 启动目标默认链接到 graphical.target 可以改为非图形界面
systemctl set-default multi-user.target
systemctl set-default graphical.target
如果是非图形界面可以 telinit 5 切换图形界面
```

## 问题备忘：
由于 EFI 模式启动时 SSD 盘无法找到，会加载超时，导致网卡驱动无法自动加载、nouveau DRM 报错，只能进入 emergency model。
解决办法：注释掉 /etc/fstab 中的 ssd 磁盘。
[Job dev-disk-by\x2duuid.device/start timed out.](https://bbs.archlinux.org/viewtopic.php?id=196083)


## Refenrece

- https://wiki.archlinux.org/index.php/Installation_guide_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#.E5.AE.89.E8.A3.85.E5.87.86.E5.A4.87
- https://wiki.archlinux.org/index.php/GNOME_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)
- https://wiki.archlinux.org/index.php/NVIDIA_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#.E5.90.AF.E7.94.A8.E6.A1.8C.E9.9D.A2.E7.BB.84.E5.90.88
- https://wiki.archlinux.org/index.php/Systemd_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#.E8.8E.B7.E5.8F.96.E5.BD.93.E5.89.8D.E7.9B.AE.E6.A0.87
- https://nouveau.freedesktop.org/wiki/KernelModeSetting/
- https://bbs.archlinux.org/viewtopic.php?id=169574
- https://askubuntu.com/questions/38780/how-do-i-set-nomodeset-after-ive-already-installed-ubuntu
- https://www.reddit.com/r/archlinux/comments/3l686i/help_with_nvidia_proprietary_drivers/
