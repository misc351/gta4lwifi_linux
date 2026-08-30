# gta4lwifi_linux

Native linux port for SM-T500.

Status: display, touch, wifi, bluetooth, audio, usb

Userspace: debian bookworm rootfs + KDE Plasma desktop + SDDM 0.20.0 
 
Boot: modified lineage dtb + kernel source, custom initramfs.


## About this repo

bootimg: contains prebuilt kernel, ramdisk, and patched dtb binaries. (modified kernel source available at gta4lwifi_kernel)

ramdisk: unpacked ramdisk.cpio.gz, contains custom init binary. (flash your debian rootfs to userdata partition)

rootfs: contains useful binaries, shell scripts, and systemd services for wifi, audio automation on boot.

*Bugs: NetworkManager route(manual set route via ip)

## Configure boot.img


Clone repo

```cmd
git clone https://github.com/misc351/gta4lwifi_linux.git

cd gta4lwifi_linux/bootimg
```

Generate boot.img (or download prebuild boot.img from release)

```cmd
mkbootimg --kernel kernel.gz --ramdisk ramdisk.cpio.gz --dtb dtb --header_version 2 --pagesize 4096 --cmdline "console=ttyMSM0,115200n8 earlycon=msm_geni_serial,0x4a90000 androidboot.console=ttyMSM0 androidboot.hardware=qcom androidboot.memcg=1 lpm_levels.sleep_disabled=1 video=vfb:640x400,bpp=32,memsize=3072000 msm_rtb.filter=0x237 service_locator.enable=1 swiotlb=2048 loop.max_part=7 firmware_class.path=/vendor/firmware_mnt/image" -o boot.img
```

## Configure rootfs

Generate disk image
```cmd
truncate -s 8G deb.img
mkfs.ext4 -L debian-rootfs deb.img
mount -o loop deb.img /mnt
```

```cmd
debootstrap --arch=arm64 --foreign --variant=minbase bookworm /mnt "http://deb.debian.org/debian"
cp /usr/bin/qemu-aarch64-static /mnt/usr/bin
chroot /mnt /debootstrap/debootstrap --second-stage
```

configure /etc/sources.list.d/, /etc/hostname, /etc/hosts, /etc/resolv.conf

sources.list
```text
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

install packages
```cmd
chroot /mnt apt update
chroot /mnt apt install systemd systemd-sysv dbus udev kmod iproute2 iptables iputils-ping openssh-server nftables procps util-linux ca-certificates curl wget nano htop fastfetch sudo rmtfs tqftpserv qrtr-tools protection-domain-mapper network-manager rfkill bluez file busybox-static wpasupplicant iw usbutils procps maliit-keyboard firmware-atheros
```

download rootfs_copy from releases
```cmd
cp -a rootfs_copy/. /mnt
```

chroot
```cmd
chroot /mnt
```

root password
```
passwd root
```

add user
```cmd
useradd -m -s /bin/bash user
passwd user
usermod -aG sudo user
```

setup machine id
```cmd
systemd-machine-id-setup
```

allow password login, allow root password login (optional)
/etc/ssh/sshd_config
```text
#allow root password login (optional)
PermitRootLogin yes
#use PermitRootLogin prohibit-password if needed

# To disable tunneled clear text passwords, change to "no" here!
PasswordAuthentication yes
```

enable services
```cmd
systemctl enable ssh.service rmtfs tqftpserv pd-mapper qrtr-ns smt500-wifi.service smt500-audio.service
```

add execute permission
```cmd
chmod +x /usr/local/sbin/startwifi
chmod +x /usr/local/sbin/startaudio
chmod +x /usr/local/sbin/wlfw
chmod +x /usr/bin/pd-mapper
chmod +x /usr/bin/tqftpserv

```

install kde
```cmd
apt install kde-plasma-desktop
```
mask fwupd (observed periodic crash)
```
systemctl disable fwupd.service
systemctl mask fwupd.service
```

install patched sddm-0.20.0
```cmd
cd /programs
apt install ./sddm_0.20.0-bookworm_arm64.deb
```


force software rednering, since current port lacks gpu support
(bookworm ships plasma 5.xx, could not port kde gpu accel)
(gpu works ok, but observed collides with this kde version)


## Runtime cmd

start audio
```cmd
systemctl --user stop pipewire pipewire-pulse wireplumber
amixer -c0 cset name='PRI_MI2S_RX Audio Mixer MultiMedia1' 1
systemctl --user restart pipewire pipewire-pulse wireplumber
```

only on first boot
```cmd
pactl list cards
pactl set-card-profile CARD_NAME HiFi
```
