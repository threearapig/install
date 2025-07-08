# Archlinux

## 基础安装

### 更改终端中的键盘布局

```bash
loadkeys colemak
```

> 临时更改  


### 关闭蜂鸣器

```bash
rmmod pcspkr
```

> 临时更改  


### 禁用 reflector

```bash
systemctl stop reflector.service
```

> reflector 会自动选择速度合适的镜像源，但是结果不是很准确，同时也会清空配置文件中的内容，所以开始就先将其禁用  


### 连接互联网

```bash
rfkill unblock all
```

> 有时电脑的无线网卡是block状态，需要解锁  

```bash
iwctl
device list
station wlan0 scan
station wlan0 get-networks
station wlan0 connect wireless-name
exit
```

> 开始安装是使用镜像自带的互联网连接工具：iwd  

```bash
ping www.baidu.com
```

> 使用 ping ，进行联网检测  


### 更新系统时钟

```bash
timedatectl set-ntp true        # 将系统时间与网络时间进行同步
timedatectl status              # 检查服务状态
```

> 需要下面的状态  
> clock synchronized：yes  
> NTP service：active  


### 分区

需要先将磁盘分区类型改成GPT格式，默认是MBR

1. 查看需要操作的是哪块盘

    ```bash
    lsblk
    ```

2. 设置分区类型

    ```bash
    parted /dev/nvme0n1
    (parted)mktable
    New disk label type? gpt
    quit
    ```

进行磁盘分区

|目录|大小|
|----|----|
|/efi|800M|
|/   |100G|
|/home|剩余全部|

> 在进行 /efi 分区的时候，需要在 cfdisk 工具中将文件类型改成 EFI System  

```bash
cfdisk /dev/nvme0n1
fdisk -l
```

### 格式化

|目录|大小|
|----|----|
|/efi|vfat|
|/   |ext4|
|/home|ext4|

```bash
mkfs.vfat /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3
```


### 挂载

```bash
mount /dev/nvme0n1p2 /mnt
mkdir /mnt/efi
mount /dev/nvme0n1p1 /mnt/efi
mkdir /mnt/home
mount /dev/nvme0n1p3 /mnt/home
```

### 安装系统

需要对镜像源进行选择  

```bash
vim /etc/pacman.d/mirrorlist
```

选择南京大学的镜像站（离的最近）

```bash
Server = https://mirrors.cernet.edu.cn/archlinux/$repo/os/$arch
```

在设置完镜像源之后，需要对 archlinux-keyring 进行更新  

```bash
pacman -Sy archlinux-keyring
```

安装必须要的基础包  

```bash
pacstrap /mnt base base-devel linux linux-headers linux-firmware
```

安装其他需要的包  

```bash
pacstrap /mnt vim bash-completion networkmanager
```


### 生成 fstab 文件

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

检查一下  

```bash
cat /mnt/etc/fstab
```


### 切换到新的系统

```bash
arch-chroot /mnt
```

### 设置时区

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

将时间写入硬件  

```bash
hwclock --systohc
```

### 本地化设置

编辑 /etc/locale.gen，去掉 en_US.UTF-8 所在行以及 zh_CN.UTF-8 所在行的注释符号（#）  

```bash
vim /etc/locale.gen
```

生成locale  

```bash
locale-gen
```

指定系统使用的语言格式  

```bash
echo 'LANG=en_US.UTF-8'  > /etc/locale.conf
```

### 设置主机名

```bash
vim /etc/hostname
```

加入以下内容：  

```bash
arch
```

设置ip  主机名 域名的对应  

```bash
vim /etc/hosts
```

加入以下内容：  

```bash
127.0.0.1   localhost
::1         localhost
127.0.1.1   arch
```

### 为 root 用户设置密码

```bash
passwd root
```


### 安装微码

```bash
pacman -S amd-ucode
```


### 安装引导程序

1. 单系统

    ```bash
    pacman -S grub efibootmgr
    grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
    ```

    生成 GRUB 所需要的配置文件  

    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

1. 双系统（windows + linux）

    ```bash
    pacman -S grub efibootmgr os-prober
    ```
    由于 grub 默认禁用 os-prober，所以需要需要手动启用  
    ```bash
    vim /etc/default/grub
    ```
    找到 `GRUB_DISABLE_OS_PROBER=false` 选项，取消注释即可  

    安装 grub：  

    ```bash
    grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
    ```

    生成 grub 配置文件：  

    ```bash
    grub-mkconfig -o /boot/grub/grub.cfg
    ```

    ---

    由于 os-prober 可能识别不到 windows，需要手动添加 windows 启动项  
    ```bash
    blkid [windows的efi分区]
    ```
    需要记录好此分区的 UUID  

    ```bash
    vim /boot/grub/grub.cfg
    ```

    找到 os-prober 所在位置，加入中间内容  

    ```txt
    ### BEGIN /etc/grub.d/30_os-prober ###
    menuentry 'Microsoft Windows 10' {
        insmod part_gpt
        insmod fat
        insmod chain
        search --no-floppy --fs-uuid --set=root [windows的efi分区的UUID]
        chainloader (${root})/EFI/Microsoft
    }
    ### END /etc/grub.d/30_os-prober ###
    ```



### 设置必要应用的开机自启

```bash
systemctl enable NetworkManager
```

### 更改终端中的键盘布局

```bash
echo "KEYMAP=colemak" > /etc/vconsole.conf
```


### 关闭蜂鸣器

```bash
echo "blacklist pcspkr" > /etc/modprobe.d/nobeep.conf
```

### 完成安装并重启

```bash
exit
umount -R /mnt
reboot
```


## 安装后的基本设置

### 连接wifi

```bash
nmcli device wifi list
nmcli device wifi connect wifiname password wifipassword
```

> 这里使用NetworkManager包的一个工具 nmcli  


### 准备非 root 用户

```bash
useradd -m -G wheel -s /bin/bash thejc
```

设置用户密码  

```bash
passwd thejc
```

编辑/etc/sudoers配置文件 
将需要的配置对应的#去掉


### swap

```bash
dd if=/dev/zero of=/swapfile bs=1M count=8192 status=progress
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

向/etc/fstab中追加如下内容：  

```text
/swapfile none swap defaults 0 0
```

### 开启 32 位支持库

编辑/etc/pacman.conf  
去掉[multilib]一节中两行的注释，来开启 32 位库支持  
追加内容：  
> 这里使用南大的源  

```bash
[archlinuxcn]
Server = https://mirrors.nju.edu.cn/archlinuxcn/$arch
```

安装 `archlinuxcn-keyring`  

```bash
pacman -Syyu
pacman -S archlinuxcn-keyring
```

### 安装 paru

```bash
pacman -S paru
```

**重启，并使用普通用户登陆**


## 图形化

### 安装浏览器

```bash
paru -S firefox wqy-microhei
```

### 安装clash-verge-rev

```bash
paru -S clash-verge-rev-bin
```

> Clash-Verge 目前的继任者  
> 不能通过 root 用户进行安装

### 拉取配置

```bash
# 安装 dotfiles 管理工具
paru -S chezmoi

# 拉取并启用 dotfiles
chezmoi init --apply https://github.com/threearapig/dotfiles
```
> 如果超时，可以连接手机热点进行下载

[后续环境安装](https://github.com/threearapig/dotfiles)
