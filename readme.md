Gentoo
---

### boot medium
you can ssh into the boot medium for ~~copy and pasting~~ ease of use
```
ifconfig
rc-service sshd start
```

### disk config
```
lsblk # change /dev/sda to whatever you want to use
parted -a optimal /dev/sda  # creating the partitions
>----->
>mklabel gpt
>print  # rm 2
>unit mib

>mkpart primary 1 3
>name 1 grub
>set 1 bios_grub on

>mkpart primary 3 153
>name 2 boot
>set 2 boot on

>mkpart primary 153 653  # this sets 500mb of swap
>name 3 swap

>mkpart primary 653 -1
>name 4 rootfs

>print
>quit
>-----<

# formatting the partitions
mkfs.vfat /dev/sda1
mkfs.vfat /dev/sda2
mkswap /dev/sda3
swapon /dev/sda3
mkfs.ext4 /dev/sda4

mount /dev/sda4 /mnt/gentoo
cd /mnt/gentoo
```


### download stage3
```
links gentoo.mirrors.tera-byte.com/releases/amd64/autobuilds  # significant point of failure
>----->
>[most recent YYYYMMDD]
>stage3-*.tar.xz  # not systemd or nomultilib
>[save]
>[ok]
>q
>[Yes]
>-----<

tar -xpf stage3-*.tar.xz --xattrs-include=`*.*` --numeric-owner  # unpack tarball
```

### preparing emerge
```
vi /mnt/gentoo/etc/portage/make.conf
>----->
CHOST="x86_64-pc-linux-gnu"
COMMON_FLAGS="-02 -pipe -march=native"

MAKEOPTS="-j6"  # 2gb of ram is required per thread. list the number of cores that fit within this requirement
#PORTAGE_NICENESS=1  # add this once finished the install as it prioritizes tasks
ACCEPT_LICENSE="*"

USE="X nvidia"  # for nvidia users
GRUB_PLATFORMS="efi-64"
VIDEO_CARDS="nvidia"  # for nvidia users
>-----<
# press space to select servers in your region. press enter to save and exit.
mirrorselect -io >> /mnt/gentoo/etc/portage/make.conf  # significant point of failure
mkdir /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf

cp -L /etc/resolv.conf /mnt/gentoo/etc/  # -L is --dereference
```

### mounting env
```
# significant point of failure
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --rbind /run /mnt/gentoo/run
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/sys
mount --make-rslave /dev /mnt/gentoo/dev
mount --make-slave /mnt/gentoo/run

chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

mount /dev/sda2 /boot
```

### emerging
i have listed the emerge times for my r5 2600
```
emerge-webrsync  # 30s
emerge --sync  # 1m

eselect news read  # man news.eselect

eselect profile list
eselect profile set 1  # if you plan on installing GNOME or KDE/Plasma, select their respective /desktop/ profiles

emerge -aqDNu @world  # 7m - 90m depending on selected profile
```

### misc config
```
ls /usr/share/zoneinfo
echo "America/undisclosed_canadian_city_eh" > /etc/timezone
emerge --config timezone-data
echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
locale-gen

eselect locale list
eselect locale set 4

env-update
source /etc/profile
export PS1="(chroot) ${PS1}"
```

### general config
```
emerge -aq gentoo-sources pciutils genkernel linux-firmware netifrc sysklogd vim  # install required packages
ls -l /usr/src/linux*
mv /usr/src/linux* /usr/src/linux

vim /etc/fstab
>----->  # please use tabs here bc they look so pretty :3
/dev/sda2	/boot	vfat	defaults	0 2
/dev/sda3	none	swap	sw		0 0
/dev/sda4	/	ext4	noatime 	0 1
>-----<

genkernel all  # 30m

ls /boot/vmlinu* /boot/initramfs*

# blkid
# vim /etc/fstab
# >----->
# # in case of multiple drives, add UUID of previous sda2 here
# >-----<

vim /etc/conf.d/hostname  # user@gentoo:$
>----->
hostname="gentoo"
>-----<

# the command `ip a` at this stage is untested
ip a  # enp4s0 is my network interface device name

vim /etc/conf.d/net
>----->
config_enp4s0="dhcp"  # use your network interface device name here
>-----<

cd /etc/init.d
ln -s net.lo net.enp4s0  # use your network interface device name here
rc-update add net.enp4s0 default  # use your network interface device name here

vim /etc/hosts
>----->  # again, please use tabs :3
127.0.0.1	gentoo localhost
>-----<

vim /etc/security/passwdqc.conf
>----->
min=1,1,1,1,1  # this is so that you can skimp on security :D
>-----<
passwd

date
vim /etc/conf.d/hwclock
>----->
EST  # your timezone here
>-----<

rc-update add sysklogd default
```

### boot loader
```
emerge -aq e2fsprogs dosfstools dhcpcd grub:2

# if you are installing to a removable drive, uncomment --removable. otherwise, exclude.
grub-install --target=x86_64-efi --efi-directory=/boot  #--removable
grub-mkconfig -o /boot/grub/grub.cfg

exit
cd
umount -l /mnt/gentoo/dev/shm
umount -l /mnt/gentoo/dev/pts
umount -R /mnt/gentoo
reboot
```

### users and cleanup
```
cd /
useradd -G users,wheel,video -m wncry
passwd wncry
visudo

rm /stage3-*.tar.xz
```

### ssh
```
emerge -aq openssh

rc-update add sshd default
rc-service sshd start

vim /etc/ssh/sshd_config
>----->
PermitRootLogin yes
>-----<
```

### nvidia drivers
this is where the *fun* begins
```
emerge -aq nvidia-drivers xorg-server

env-update && source /etc/profile
gpasswd -a root video

cd /usr/src/linux && make menuconfig
>----->
>Device Drivers --->
>   Graphics support --->
>      < >  Nouveau (NVIDIA) cards
>-----<

mkdir /etc/X11/xorg.conf.d
vim /etc/X11/xorg.conf.d/nvidia.conf
>----->
Section "Device"
   Identifier  "nvidia"
   Driver      "nvidia"
EndSection
>-----<

emerge @module-rebuild
lsmod | grep nvidia  # may require reboot
rmmod nvidia_drm
rmmod nvidia_modeset
rmmod nvidia
modprobe nvidia
modprobe nvidia_modeset
modprobe nvidia_drm
emerge -aqDNu @world  # rebuilds packages that benefit from USE flags
etc-update  # update packages

emerge -aq mesa-progs  # testing
glxinfo | grep direct
```

### display manager
#### ***!! this section is still under refinement !!***
https://github.com/fairyglade/ly  # og
https://github.com/Cavernosa/ly  # systemctl fork
```
git clone --recurse-submodules https://github.com/cavernosa/ly  # https://github.com/fairyglade/ly
cd ly
make && make run
make install installopenrc
rc-update add ly default
rc-service ly start
```

### desktop environment + windows manager
#### gnome (untested)
https://wiki.gentoo.org/wiki/GNOME/Guide
```
eselect profile set default/linux/amd64/17.1/desktop/gnome
emerge -aq gnome

env-update && source /etc/profile

rc-update add elogind boot
rc-service elogind start 

# for the gnome display manager (gdm):
emerge -aqn gui-libs/display-manager-init
vim /etc/conf.d/display-manager
>----->
DISPLAYMANAGER="gdm"
>-----<

rc-update add display-manager default 
rc-service display-manager start 
systemctl enable gdm
systemctl start gdm
```

#### KDE/Plasma (untested)
https://wiki.gentoo.org/wiki/KDE
https://wiki.gentoo.org/wiki/SDDM
```
eselect profile set default/linux/amd64/17.1/desktop/plasma

emerge -aq plasma-meta
vim ~/.xinitrc
>----->
#!/bin/sh
exec dbus-launch --exit-with-session startplasma-x11
>-----<

# for the simple desktop display manager
emerge -aq sddm
usermod -a -G video sddm

vim /etc/sddm.conf
>----->
[X11]
DisplayCommand=/etc/sddm/scripts/Xsetup
>----->

mkdir -p /etc/sddm/scripts

vim /etc/sddm/scripts/Xsetup
>----->
setxkbmap us
>-----<

chmod a+x /etc/sddm/scripts/Xsetup

emerge -aq display-manager-init

vim /etc/conf.d/display-manager
>----->
CHECKVT=7
DISPLAYMANAGER="sddm"
>-----<

rc-update add display-manager default
rc-service display-manager start
```

#### ratpoison
https://wiki.gentoo.org/wiki/Ratpoison
```
emerge -aq ratpoison alacritty ranger htop dev-vcs/git feh display-manager-init

#cp /etc/X11/sessions/ratpoison.desktop /usr/share/xsessions/
vim .ratpoisonrc
>----->
exec feh --bg-scale ~/.wallpaper.png  # sets a wallpaper
exec /usr/bin/rpws init 4 -k  # opens four environments
exec ratpoison -c "banish"  # moves mouse to the bottom left corner

set border 3  # window padding

# application execution
bind a exec alacritty  # executed by 'ctrl+t a'
bind l exec librewolf

# window management
bind M-R remove
bind R resize  # executed by 'ctrl+t shift+r'
bind V vsplit
bind H hsplit

# environments
bind 1 exec rpws 1  # executed by 'ctrl+t 1'
bind 2 exec rpws 2
bind 3 exec rpws 3
bind 4 exec rpws 4
>-----<
```

## librewolf
```
vim /etc/portage/repos.conf/librewolf.conf
>----->
[librewolf]
priority = 50
location = /var/db/repos/librewolf
sync-type = git
sync-uri = https://gitlab.com/librewolf-community/browser/gentoo.git
auto-sync = Yes
>-----<

emaint -r librewolf sync
emerge -aq librewolf  # beard time
```

# YOU JUST INSTALLED GENTOO :D
you can now tell others to 'install gentoo'
