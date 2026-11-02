
To begin, a new VM was created from the Proxmox GUI:
<img width="1917" height="983" alt="image" src="https://github.com/user-attachments/assets/7abfe71d-ecee-441c-8b46-f4039b242c4f" />



The following parameters were then selected before creating the VM:

**Name:** *Name*
**OS:** archlinux-2026.10.01-x86_64.iso
**Qemu Agent:** TRUE
**Disk Size:** 20GiB
**CPU:** 4 cores
**Memory:** 4000MiB
**Bridge:** *System network bridge*

With the VM created, the system could be booted which opens the installation environment.

To begin the installation, the disk needs to be properly partitioned.

To view the current disk(s):
```linux
fdisk -l
```

To begin partitioning the disk:
```linux
fdisk /dev/<selected disk>
```

This will open the fdisk editor.

To create a new MBA partition:
```fdisk
n
p
1
2048
+4G
```

Set the new partition to swap space:
```fdisk
t
82
```

Partition the rest of the drive as main memory:
```fdisk
n
p
2
Enter
Enter
```

Write new partition to the drive:
```fdisk
w
```

Format the swap space and main space as .ext4 format:
```linux
mkswap /dev/sda1
mkfs.ext4 /dev/sda2
```

Activate the main space and swap space
```linux
mount /dev/sda2 /mnt
swapon /dev/sda1
```

Install packages needed for boot:
```linux
pacstrap -K /mnt base linux linux-firmware
```

Format the keyboard layout:
```linux
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Change the system hostname and install/enable a network manager:
```linux
echo "libertyarch" > /etc/hostname
pacman -S NetworkManager
Y
systemctl enable NetworkManager
```

Change the root password:
```linux
passwd
<new password>
```

Install remaining packages and set up grub:
```linux
pacman -S grub nano sudo zsh openssh git base-devel man-db man-pages texinfo qemu-guest-agent
grub-install --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

Leave chroot editor, mount the now completed drive, and reboot:
```linux
exit
umount -R /mnt
reboot
```

*This didn't boot on my first try because silly me forgot to configure grub*

*Resolved by rebooting into the .iso, re-entering chroot, running the setup commands for grub, and rebooting*

Setup new users as outlined by the assignment instructions:
```linux
useradd -m -G wheel -s /usr/bin/zsh <username>
passwd <username>
<password>
useradd -m -G wheel -s /usr/bin/zsh codi
passwd codi
<codi's bank password>
```

Enable wheel group:
```
EDITOR=nano visudo
```
Uncomment *%wheel ALL=(ALL:ALL) ALL* line

Install DE associated packages and enable DE before rebooting:
```linux
pacman -S xorg-server lxqt openbox sddm firefox ttf-dejavu
systemctl enable sddm
reboot
```

Login and open terminal:

Enable remote ssh access and customize the terminal environment with a couple aliases:
```linux
systemctl --now enable sshd
echo "alias ls='ls --color=auto'" >> ~/.zshrc
echo "alias grep='grep --color=auto'" >> ~/.zshrc
su codi
cd /home/codi
echo "alias ls='ls --color=auto'" >> ~/.zshrc
echo "alias grep='grep --color=auto'" >> ~/.zshrc
exit
```

**Setup a default background for all users**

Download desired background picture to Pictures directory.

Create a new shared directory for the background image, copy it, adjust the needed permissions, and open the `settings.conf` file to write in the new background.

```linux
sudo mkdir -p /usr/share/backgrounds
sudo cp ~/Pictures/background.png /usr/share/backgrounds/background.png
sudo chmod 644 /usr/share/backgrounds/background.png
sudo nano /etc/xdg/pcmanfm-qt/lxqt/settings.conf
```

In the `settings.conf` file, replace the current background with the **FULL ADDRESS** of the new background (it will not work with an incomplete address):
```nano
Wallpaper=/usr/share/backgrounds/background.png
WallpaperMode=stretch
```

Remove the `settings.conf` file from the user so it will default to the shared `settings.conf` we just setup.
```linux
rm ~/.config/pcmanfm-qt/lxqt/settings.conf
```

Additionally, `su` in for each user and remove it from their home directory.

*While this worked for current users, it left a black screen for new users, the following lines fixed it by setting the default background for new users by adjusting the template file for new users*

Set the background for the new user template by copying over our adjusted `settings.conf` file to the new user template:
```linux
sudo mkdir -p /etc/skel/.config/pcmanfm-qt/lxqt
sudo cp /etc/xdg/pcmanfm-qt/lxqt/settings.conf /etc/skel/.config/pcmanfm-qt/lxqt/
```

**Setup a default login screen for all users**


Create a new theme directory:
```linux
sudo cp -r /usr/share/sddm/themes/maldives /usr/share/sddm/themes/<newtheme>
ls /usr/share/sddm/themes/<newtheme>
```

Replace the background file in the new theme directory with the new background file and enter the `theme.conf` file to point to it:
```linux
sudo cp /usr/share/backgrounds/<old background file> /usr/share/sddm/themes/newtheme/<new background file>
sudo nano /usr/share/sddm/themes/newtheme/theme.conf
```

Replace the old file path with the new file path:
```nano
[Theme]
background=/usr/share/sddm/themes/newtheme/background.png
```

*Note: You **NEED** the full path to be included here for the image to load, I know from experience*

Complete necessary setup actions to have SDDM use the new theme we just created:
```linux
sudo mkdir -p /etc/sddm.conf.d
sudo nano /etc/sddm.conf.d/theme.conf
```
```nano
[Theme]
Current=newtheme
```

*After these adjustments, I was left with a black and blank login screen, after investigating the logs and consulting possible causes, I determined that my SDDM and script were running two different versions of Qt to load the DE*

Tell SDDM to run the new theme with Qt 6:
```linux
sudo nano /usr/share/sddm/themes/newtheme/metadata.desktop
```
```nano
[SddmGreeterTheme]
QtVersion=6
Name=Maldives
...
```

Restart SDDM to switch themes:
```linux
sudo systemctl restart sddm
```

**Create a universal constitution file so all users know the rules**

Create the constitution and adjust permissions so normal users cannot change it:
```linux
nano /etc/constitution.txt
```
```nano
<constitution text>
```
```linux
chown root:root /etc/constitution.txt
chmod 444 /etc/constitution.txt
```

Copy the constitution to the new user template so it is downloaded to their desktop by default:
```linux
mkdir -p /etc/skel/Desktop
ln -s /etc/constitution.txt /etc/skel/Desktop/constitution.txt
```

Create some more aliases to make terminal traversal easier:
```linux
echo "alias ls='ls -lah'" >> ~/.zshrc
echo "alias c='clear'" >> ~/.zshrc
echo "alias const='more /etc/constitution.txt'" >> ~/.zshrc
```
