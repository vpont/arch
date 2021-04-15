# Useful resources
* https://itsfoss.com/install-arch-linux/
* https://itsfoss.com/things-to-do-after-installing-arch-linux/
* http://martinsosic.com/linux/2018/11/16/arch-linux-on-dell-xps-15.html
* https://wiki.archlinux.org/index.php/Dell_XPS_13_(9370)
* https://wiki.archlinux.org/index.php/Silent_boot
* https://securityhacklabs.net/forum/distribuciones-gnulinux/archlinux/2018-06-14/systemd-networkd-un-dhcp-mas-rapido-que-dhcpcd

# Installation procedure (encrypted LVM on EFI):
  1. In Archiso boot menu, hit `e` button and add following to the kernel line: `video=1920x1080`
  1. `loadkeys es`
  1. Connect to the internet with `wifi-menu`
  1. Update package indexes:
        * `curl -s "https://www.archlinux.org/mirrorlist/?country=ES&protocol=https" | sed -e 's/^#Server/Server/' > /etc/pacman.d/mirrorlist`
        * `pacman -Syy`
  1. Create boot partition:
        * `fdisk /dev/nvme0n1`
        * g (to create an empty GPT partition table)
        * n
        * 1
        * enter
        * +512M
        * t
        * 1 (for EFI System)
  1. Create LVM partition:
        * n
        * 2
        * enter
        * enter
        * t
        * 2
        * 31 (for Linux LVM)
        * w
  1. `mkfs.fat -F32 /dev/nvme0n1p1`
  1. Set up encryption:
        * `cryptsetup luksFormat /dev/nvme0n1p2`
        * `cryptsetup open /dev/nvme0n1p2 cryptdisk`
  1. Set up lvm:
        * `pvcreate /dev/mapper/cryptdisk`
        * `vgcreate irbis /dev/mapper/cryptdisk`
        * `lvcreate -C y -L 25G -n swap irbis`
        * `lvcreate -C n -L 30G -n root irbis`
        * `lvcreate -C n -l 100%FREE -n home irbis`
  1. `mkfs.ext4 /dev/mapper/irbis-root`
  1. `mount /dev/mapper/irbis-root /mnt`
  1. `mkfs.ext4 -m 0 /dev/mapper/irbis-home`
  1. `mkdir /mnt/home`
  1. `mount /dev/mapper/irbis-home /mnt/home`
  1. `mkdir /mnt/boot`
  1. `mount /dev/nvme0n1p1 /mnt/boot`
  1. `mkswap /dev/mapper/irbis-swap`
  1. `swapon /dev/mapper/irbis-swap`
  1. `pacstrap -i /mnt base base-devel linux linux-firmware lvm2 intel-ucode mkinitcpio`
  1. `genfstab -U -p /mnt > /mnt/etc/fstab`
  1. `arch-chroot /mnt /bin/bash`
  1. `ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime`
  1. `hwclock --systohc`
  1. `pacman -Syyu`
  1. Configure locale settings:
        * `echo "es_ES.UTF-8 UTF-8" > /etc/locale.gen`
        * `locale-gen`
        * `echo "LANG=es_ES.UTF-8" > /etc/locale.conf`
        * `localectl set-locale es_ES.UTF-8`
        * `localectl set-keymap --no-convert es`
        * `echo "KEYMAP=es" > /etc/vconsole.conf`
  1. Edit `/etc/hosts` and add:
        ```
        127.0.0.1   localhost
        ::1         localhost
        127.0.1.1	irbis.neobookings.com irbis
        ```
  1. `pacman -S dosfstools openssh nano zsh man`
  1. Edit `/etc/mkinitcpio.conf` and change `HOOKS` to `systemd autodetect modconf block sd-encrypt sd-lvm2 filesystems keyboard fsck`
  1. `mkinitcpio -P`
  1. Set up systemd-boot:
        * `bootctl --path=/boot install`
        * Get UUID: `lsblk -f | grep LUKS | awk '{print $3}' > /boot/loader/entries/arch.conf`
        * Edit `/boot/loader/entries/arch.conf` with this content:
            ```
            title   Arch Linux
            linux   /vmlinuz-linux
            initrd  /intel-ucode.img
            initrd  /initramfs-linux.img
            options rd.luks.name=crypto_LUKS_UUID=irbis rd.luks.options=discard rd.lvm.lv=irbis/swap rd.lvm.lv=irbis/root rd.lvm.lv=irbis/home root=/dev/mapper/irbis-root resume=/dev/mapper/irbis-swap video=1920x1080 rw quiet
            ```
        * Edit `/boot/loader/loader.conf` and put this:
            ```
            default      arch
            timeout      2
            console-mode max
            editor       yes
            ```
        * `bootctl --path=/boot update`
  1. Enable network:
        * `pacman -S wpa_supplicant wireless_tools networkmanager`
        * `systemctl enable NetworkManager`
        * `systemctl enable wpa_supplicant`
  1. Enable SSH:
        * `systemctl enable sshd.service`
        * `passwd`
  1. Create user:
        * `useradd -m -g users -G wheel,storage,power -s /bin/zsh victor`
        * `passwd victor`
        * `echo "%wheel ALL=(ALL) ALL" > /etc/sudoers.d/wheel`
  1. Full system upgrade:
        * `pacman -Syyu`
  1. Reboot:
        * `exit`
        * `umount -a`
        * `reboot`
  1. `hostnamectl set-hostname irbis.neobookings.com`
  1. Set timezone:
        * `timedatectl set-timezone Europe/Madrid`
        * `timedatectl set-ntp true`
  1. Install window manager and desktop environment:
        * `pacman -S xorg xorg-xinit budgie-desktop budgie-extras gnome lightdm lightdm-gtk-greeter xf86-video-intel network-manager-applet`
  1. Enable and start lightdm:
        * `systemctl enable lightdm`
        * `systemctl start lightdm`
  1. Install software:
        * `pacman -S git gedit hplip duplicity rsync firefox firefox-i18n-es-es thunderbird thunderbird-i18n-es-es`
  1. Install paru:
        * `cd /tmp`
        * `git clone https://aur.archlinux.org/paru-bin.git`
        * `cd paru-bin`
        * `makepkg -si`
  1. paru optimizations:
        * Create `~/.makepkg.conf` and add:
            ```
            CFLAGS="-march=native -O2 -pipe -fno-plt"
            CXXFLAGS="${CFLAGS}"
            ```
  1. Install themes, cursors and icons:
        * `paru materia-theme-git`
        * `paru papirus-icon-theme-git`
        * `paru breeze-default-cursor-theme`
  1. Install zsh themes and fonts:
        * `paru oh-my-zsh-git`
        * `paru powerline-fonts-git`
        * `paru oh-my-zsh-powerline-theme-git`
        * `paru ttf-google-fonts-git`
        * `paru nerd-fonts-terminus`
  1. Install other packages:
        * `paru google-chrome`
        * `paru franz`
  1. Install Lightdm Slick Greeter:
        * `paru lightdm-slick-greeter`
        * `paru lightdm-settings`
        * Edit `/etc/lightdm/lightdm.conf` and add `greeter-session=lightdm-slick-greeter` in the `[Seat:*]` section
        * Create `/etc/lightdm/slick-greeter.conf` and add:
            ```
            [Greeter]
            background=/home/victor/Imágenes/neobookings-background.png
            theme-name=Materia-dark-compact
            icon-theme-name=Papirus-Dark
            show-a11y=false
            ```
  1. Install Plymouth (not recommended):
        * `paru plymouth-git`
        * Edit `/etc/mkinitcpio.conf` and add `plymouth plymouth-encrypt` after `base` and `udev`, and remove `encrypt`
        * `mkinitcpio -p linux`
        * Edit `/boot/loader/entries/arch.conf` and add `splash` after `quiet` in `options`
        * `bootctl --path=/boot update`
        * `paru plymouth-theme-arch-charge`
        * `plymouth-set-default-theme -R arch-charge`
  1. Install budgie applets:
        * `paru indicator-sysmonitor-budgie-git`
        * Add new sensor `coretemp` which executes: `echo "$(expr $(cat /sys/devices/platform/coretemp.0/hwmon/hwmon4/temp1_input) / 1000)ºC"`
  1. Install i3-gaps:
        * `pacman -S xorg xorg-xinit xorg-server lightdm lightdm-gtk-greeter i3-gaps dunst feh`
        * `setxkbmap es`
        * `paru polybar-git`
