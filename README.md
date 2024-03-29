![banner](assets/banner.png)

# Easy Arch Install

A concise and detailed guide for effortless Arch Linux installation, offering a smooth installation journey

> [!NOTE]
> The author assumes the reader has basic understanding of command line usage and shell operations. For beginners, it's advisable to opt for a user-friendly Linux distribution instead of Arch Linux.

## Requirements:

- UEFI supported computer.
- An internet connection.
- Patience and a will to _not_ give up.

# 📑 Index

- [Initial Setup](https://github.com/m2x07/easy-arch-install#1-initial-setup)
- [Disk Configuration](https://github.com/m2x07/easy-arch-install#2-disk-configuration)
- [Installing base system](https://github.com/m2x07/easy-arch-install#3-installing-base-system)
- [Chrooting](https://github.com/m2x07/easy-arch-install#4-chrooting)
- [Other GUIs](https://github.com/m2x07/easy-arch-install#5-other-desktop-environments-and-window-managers)
- [Post Install Configuration](https://github.com/m2x07/easy-arch-install#6-post-install-configuration)
- [Additional Links](https://github.com/m2x07/easy-arch-install#additional-links)

> [!IMPORTANT]
> Whenever you come across something like &lt;this&gt; (inside angle brackets), just swap it with the right path or device name as required. \
> Make sure you have the installation medium prepared and are booted from it.

# 1. Initial Setup

## Connecting to Internet

If you have an ethernet cable, plug it in right now and skip to next section.

### Wi-fi:

Start the daemons required:

```bash
systemctl start iwd
systemctl start dhcpcd
```

Get interface name:

```bash
ifconfig
```

Connecting to wi-fi.

```bash
iwctl station <interface_name> connect "ssid"
```

> [!NOTE] > `ssid` is the name of the wi-fi network you want to connect to

Hit ENTER to enter your wifi password and confirm connection

**Alternatively, through USB Tethering:**

1. Connect your phone to your computer using a USB Cable.
2. Navigate to Portable Hotspot (or look for similar settings) section and enable the USB tethering option.
3. Make sure your phone is connected to the internet and now your computer should have internet access.

### Test your internet connection:

```bash
ping 3 8.8.8.8       # ensure that DHCP is working
ping archlinux.org   # ensure that DNS is working
```

If the connection is successfull, you should see repeated output similar to this: \

```
64 bytes from 8.8.8.8: icmp_seq=3 ttl=117 time=73.5 ms
```

## Set Timezone

First set the timezone:

```bash
timedatectl set-timezone <Region/City>
```

> [!IMPORTANT]
> Replace `<Region/City>` with your region. For example, for India, it'll be `Asia/Kolkata`. So the command with look like `timedatectl set-timezone Asia/Kolkata`. \
> Hitting TAB a few times should give you a list. Thanks to the zsh shell in the live iso.

Enable NTP synchronization, allowing your system to automatically sync its clock with a network time server:

```bash
timedatectl set-ntp true
```

Next we sync the hardware clock of our computer with the timezone we just set instead of UTC

```bash
timedatectl set-local-rtc true
```

# 2. Disk Configuration

## Getting disk information:

Run the command `fdisk -l` and you should see output similar to this:

```
Disk /dev/nvme0n1: 476.94 GiB, 512110190592 bytes, 1000215216 sectors
Disk model: INTEL SSDPEKNU512GZ
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: C4F8B2FE-0074-4CC4-8445-F0C646127ACB
```

The `/dev/nvme0n1` in the first line is our disk for this installation. The disk name might be different, such as `/dev/sda` or `/dev/sdb` or something else. Find the name of the disk where you want to install Arch Linux and make a note of it.

## Partitioning:

For this guide, we will mainly use the EXT4 filesystem to install Arch Linux. The selected tool for partition creation is cfdisk, recognized for its user-friendly TUI, making it easy to navigate through options using the keyboard.

```bash
cfdisk /dev/nvme0n1     #replace disk name(nvme0n1) with your disk name
```

Delete any existing partions by following the on-screen option. \
Create 3 partions as follows:

| Partition    | Size              | Filesystem           |
| ------------ | ----------------- | -------------------- |
| &lt;efi&gt;  | 1024M             | EFI System partition |
| &lt;swap&gt; | same as your ram  | Linux Swap           |
| &lt;root&gt; | remaining storage | Linux filesystem     |

> [!IMPORTANT]
> Minimum of 512M is recommended for EFI partition. &ge;1024M if installing multiple kernel.

Write and Quit. \
Let's run the `fdisk -l` command again.

```
Device            Start        End   Sectors   Size Type
/dev/nvme0n1p1     2048    2099199   2097152     1G EFI System
/dev/nvme0n1p2  2099200   35653631  33554432    16G Linux swap
/dev/nvme0n1p3 35653632 1000214527 964560896 459.9G Linux filesystem
```

You should see similar output from the our previous `fdisk -l` command but with these lines at the end. \
This is an indication of successfull partitioning.

## Formatting:

Let's format our partitions with the suitable filesystem. \
As mentioned earlier, there are various filesystem options for partitioning and formatting. For this guide, we'll use EXT4.

```bash
mkfs.fat -F 32 /dev/<efi>       # FAT32 for the efi partition
mkfs.swap /dev/<swap>           # swap partition
mkfs.ext4 /dev/<root>           # EXT4 for the root partition
```

> [!NOTE]
> As of now, for BTRFS and other Filesystems, refer the friendly [ArchWiki](https://wiki.archlinux.org/title/btrfs)

## Mounting:

Mount all of the partitions:

```bash
mkdir /mnt/boot
mkdir /mnt/boot/efi             # Create an EFI mount point
mount /dev/<efi> /mnt/boot/efi  # mount the EFI partition
swapon /dev/<swap>              # enable the swap partition
mount /dev/<root> /mnt          # mount the root partition
```

If you encounter any errors, double-check your partitioning. Delete and recreate the same partitions if needed.

# 3. Installing base system

Now its time to install the base system and kernel.
Before we do so, let's make some changes to the config file of our package manager.

```bash
nano /etc/pacman.conf           # or any text editor of your preference
```

Locate and un-comment the following lines:

```
Color
ParallelDownloads = 3

[multilib]
Include = /etc/pacman.d/mirrorlist
```

Although not necessary, you may increase the parallel downloads number if you wish. \
Using the keybinds given on the bottom of the screen, Write the file and exit.

Now lets install the packages and wait for the installation to finish.

```bash
pacstrap /mnt base linux linux-firmware vim nano
```

> [!NOTE]
> If you want to use any other kernel, then replace `linux` with your desired kernel. For example `linux-zen` or `linux-lts`. No need to replace the `linux-firmware` package. \
> Also, install `btrfs-progs` if you are using btrfs filesystem. it provides additional tools for btrfs filesystem.

Now finally generate the fstab configuration.

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

# 4. Chrooting

Till now we were working on our disk through the live environment. \
Now we need to shift to our disk and do our thing there.

```bash
arch-chroot /mnt
```

## Setting up the host

First we set the hostname in `/etc/hostname` file. A host name is like a nickname for our computer. You may take a moment to choose one for yourself, and remember it. \
Once you've chosen, open up the file `/etc/hostname` and write your hostname into it.

```bash
vim /etc/hostname
```

> [!TIP]
> Use nano if not comfortable with using vim.

Now, Open the file `/etc/hosts` and _APPEND_ the following lines in there:

```
127.0.0.1       localhost
::1             localhost
127.0.0.1       <hostname>
```

In case you didn't guessed it yet, replace `<hostname>` with what you set in the `/etc/hostname` file.

## Configuring the Locale

Open the file `/etc/locale.gen` and un-comment your desired locale. For us, its `en_US.UTF-8`. \
Generate your locales based on the `locale.gen` file:

```bash
locale-gen
```

Set the locale in `/etc/locale.conf` file.

```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Export the environment variable for the locale.

```bash
export $(cat /etc/locale.conf)
```

## Set Timezone

Repeat the same 3 commands from the first section of this guide.

```bash
timedatectl set-timezone Asia/Kolkata
timedatectl set-ntp true
timedatectl set-local-rtc true
```

## Installing necessary packages

First, make some changes to the config file of our package manager.

```bash
vim /etc/pacman.conf           # or any text editor of your preference
```

Locate and un-comment the following lines:

```
Color
ParallelDownloads = 5

[multilib]
Include = /etc/pacman.d/mirrorlist
```

Feel free to change the value of ParallelDownloads. \
Install the packages using:

```bash
pacman -Syy sudo linux-headers efibootmgr grub amd-ucode git base-devel dkms avahi os-prober ntfs-3g
```

> [!IMPORTANT]
> Replace `amd-ucode` with `intel-ucode` for intel CPUs. If you installed `linux-zen` kernel, then replace `linux-headers` with `linux-zen-headers`. And include `grub-btrfs` package if you used btrfs filesystem

We'll also install a web browser of our choice. Firefox is our choice

```bash
pacman -S firefox
```

> [!TIP]
> I highly recommend avoiding Google Chrome or Chrome-based browers. Some better alternatives are firefox, mercury (fork of firefox), floorp (another firefox fork that i use personally), brave or vivaldi

## Adding a new user

We haven't created our user yet, and we'll do that now. \
First, let's change the password for the root user.

```bash
passwd
```

Adding our user. Replace &lt;username&gt; with your desired username.

```bash
useradd -s /bin/bash -mG wheel <username>
```

Set the password for our new user

```bash
passwd <username>
```

Let's allow the group "wheel" to execute any commands with sudo

```bash
EDITOR=vim visudo
```

> [!TIP] > `nano` can be used instead of `vim` in the above command.

Locate and un-comment the following line: \
`%wheel ALL=(ALL:ALL) ALL`

## Installing a bootloader (GRUB)

We'll install GRUB for this guide. If you wish to use any other bootloader, refer to the respective documentation.

```bash
grub-install --target=x86_64-efi --bootloader-id="Arch Linux" --efi-directory=/boot/efi --recheck
```

Open the file `/etc/default/grub` and un-comment the following line present at the end of the file:
`GRUB_DISABLE_OS_PROBER=false`

Creating the initial ramdisk.

```bash
mkinitcpio -P
```

Generate the GRUB configuration file.

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

## Installing a GUI (Desktop Environment OR Window Manager)

This guide covers several different Desktop Environment and Window Managers. \
We'll primarily cover KDE Plasma but if you wish to install any other, directly skip to [this section](https://github.com/m2x07/easy-arch-install#5-other-desktop-environments-and-window-managers)

### KDE Plasma:

Install the following packages:
`pacman -S plasma plasma-wayland-session kde-applications sddm gnu-free-fonts noto-fonts noto-fonts-emoji packagekit-qt5 gnome-keyring cronie pipewire-media-session pipewire-jack phonon-qt5-vlc tesseract-data-ind python-pyqt5 xdg-desktop-portal-kde`
You will be promped to choose which package to install couple of times. \
Simply go with default option. To do so, just hit ENTER each time you are asked.

Fix sreen's color issue by removing a few packages.

```bash
pacman -R colord colord-kde
```

Enable the NetworkManager, Login Manager and avahi daemon.

```bash
systemctl enable NetworkManager sddm avahi-daemon
```

This will take a while, so sit back and wait for the installation to finish. \

Once the installation is done, its finally time to exit the chroot.

```bash
exit
```

Now that you are back in live iso, reboot your system.

```bash
reboot now
```

Pull out the thumb drive with live iso once the screen turns off and before your computer turns back on. \
Once you see the login screen, make sure you have `Plasma (X11)` selected in the dropdown menu at top-left of the screen. Enter your password and login. \

> [!TIP]
> Now that KDE is installed, you might want to try out my [Konsole colorscheme](./assets/Terminator.colorscheme).

# 5. Other Desktop Environments and Window Managers

Skip this section if you already installed KDE Plasma.

## DWM

> For DWM setup, we'll use the following stack: \
> MENU: [`dmenu`](https://dmenu.suckless.org) \
> TERM: [`st`](https://st.suckless.org) \
> BAR: [`slstatus`](http://tools.suckless.org/slstatus/) \
> COMPOSITOR: [`xcompmgr`](https://wiki.archlinux.org/title/Xcompmgr) \
> IMAGE-VIEWER: [`sxiv`](https://wiki.archlinux.org/title/Sxiv)

Install dependencies:

```bash
sudo pacman -S xorg-server xcompmgr sxiv xorg-xinit xwallpaper xorg-xrandr libxft libxinerama
```

Clone the repositories at `~/.local/src/`: \
You may clone them anywhere you want.

```bash
git clone https://git.suckless.org/dwm
```

```bash
git clone https://git.suckless.org/dmenu
```

```bash
git clone https://git.suckless.org/st
```

```bash
git clone https://git.suckless.org/slstatus
```

Now to install them, run this in every repos you just cloned.

```bash
sudo make clean install
```

We also want our Xorg server and dwm to start automatically every time we login. \
Open your login shell initialization file (like `~/.bash_profile` or `~/.zprofile`) and put the following in there. Create the file if its not present.

```bash
if [ -z "$DISPLAY" ] && [ "$XDG_VTNR" -eq 1 ]; then
  exec startx
fi
```

Arch wiki has a lot of usefull information and tips and tricks for Xinit. It is highly recommend checking it out as it may help you further tweak your installation.
Find it [here](https://wiki.archlinux.org/title/Xinit#Autostart_X_at_login).

Get a copy of `.xinitrc`.

```bash
cat /etc/X11/xinit/xinitrc > ~/.xinitrc
```

Now open `~/.xinitrc` and replace the last 5 lines with:

```
xcompmgr &
xwallpaper --zoom path/to/wallpaper.jpg &
slstatus &
exec dwm
```

> [!IMPORTANT]
> The line `exec dwm` should always be last and all other line above it should end with `&`. Programs like xcompmgr runs forever and usually doesn't stop unless system is powered off. failing to put `&` at the end will just block the execution of script until the programs finishes execution and hence dwm will never be executed.

> [!TIP]
> You may now go ahead and **apply the patches** you want from [suckless.org](https://suckless.org). \
> Optionally, you may install a login manager like [ly](https://github.com/fairyglade/ly) or [lightdm](https://wiki.archlinux.org/title/LightDM) if you want.

## GNOME

Install required packages and groups:

```bash
pacman -Su gnome gnome-extra gnome-tweaks gdm
```

Start the GDM service to use GNOME everytime you login

```bash
systemctl enable gdm.service
```

By Default, GNOME runs on Wayland and runs some native X11 apps using Xwayland.
While on the login screen, click the gear icon on the bottom of the screen to switch to Xorg and GNOME Classic. \
Find more details [here](https://wiki.archlinux.org/title/GNOME).

## XFCE

Installation:

```bash
pacman -S xfce4 xfce4-goodies lightdm
```

Start the Display Manager service.

```bash
systemctl enable lightdm.service
```

Now you should be able to use XFCE4.
Find more details [here](https://wiki.archlinux.org/title/Xfce).

# 6. Post Install Configuration

## Installing a helper for Arch User Repository (AUR)

You cannot miss on the AUR. To get packages from AUR, you need an AUR Helper
There are many [AUR helpers](https://wiki.archlinux.org/title/AUR_helpers) available. We will be installing YAY.
Follow the commands below one by one:

```bash
cd /opt
sudo git clone https://aur.archlinux.org/yay-git.git
sudo chown -R $USER:$USER ./yay-git
cd yay-git
makepkg -si
```

Enter your password whenever prompted and hit ENTER whenever prompted. \
Now you can install packages from the Arch User Repository (AUR) just like you do in pacman: \
Example:

```bash
yay -S package_name         # install a package
yay -Ss <search_keyword>    # search for a package
```

## Installing NVIDIA Drivers for linux

> [!NOTE]
> This doesn't covers all the details for all NVIDIA cards. The steps shown here should work for most of the latest NVIDIA GPUs.

### Installing necessary packages

First of all, make sure you have these packages installed:
`base-devel`, `linux-headers` and `git`. \
If you don't have any of these packages, please install them first

Next step is to find out your card's code name [from this list](https://nouveau.freedesktop.org/CodeNames.html). \
Find your card from the list and remember/note down its codename as different cards needs different drivers. \
Carefully refer to the above list and then proceed down further this guide, it might be confusing at first.

- For `NV110` series and newer (should work for almost every newer cards):

```bash
yay -S nvidia nvidia-utils nvidia-settings nvtop lib32-nvidia-utils opencl-nvidia lib32-opencl-nvidia
```

> replace `nvidia` with `nvidia-lts` if you installed `linux-lts` kernel. `nvidia-dkms` for any other kernel.

- For `NVCx`_ and `NVDx`_ cards:

> \*where 'x' is any letter/number

```bash
yay -S nvidia-settings nvidia-390xx-dkms nvidia-390xx-utils lib32-nvidia-390xx-utils opencl-nvidia-390xx lib32-opencl-nvidia-390xx
```

- For `NVE0` family:

```bash
yay -S nvidia-settings nvidia-470xx-dkms nvidia-470xx-utils lib32-nvidia-470xx-utils opencl-nvidia-470xx lib32-opencl-nvidia-470xx
```

> [!NOTE]
> This is just examples of most common used drivers. \
> For more information, refer [this](https://wiki.archlinux.org/title/NVIDIA) page on Arch Wiki.

### Enable the DRM Kernel mode settings

Open the grub config file(/etc/default/grub) using your preffered text editor (sudo required). \
Find the line with `GRUB_CMDLINE_LINUX_DEFAULT` and append `nvidia-drm.modeset-1` to it. \
Save and Exit the file and then:

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

Open up the mkinitcpio config file(/etc/mkinitcpio.conf) the same way. \
Find the line with `MODULES=()`. This brackets doesn't have to be empty. \
Append the following into the brackets:
`nvidia nvidia_modeset nvidia_uvm nvidia_drm` \
Save and Exit. \
Finish off with `sudo mkinitcpio -P`.

### Add the pacman hook

Every time the NVIDIA driver updates, you need to update the initramfs as well. this step takes away that hassle from you by using a [pacman hook](https://wiki.archlinux.org/title/Pacman#Hooks). \
Create a new empty file: `/etc/pacman.d/hooks/nvidia.hook` \
You can do that either by using the `touch` command or just directly opening it using your preffered editor as if it already exists. First save will automatically create the file.

Put the following content into the `nvidia.hook` file:

```
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia
Target=linux
# Change the linux part above if a different kernel is used

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux*) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

> [!IMPORTANT]
> Replace both the values for "Target" according to what you installed. for example, `nvidia-lts` and `linux-lts` if you have `linux-lts` kernel and same goes if you have `nvidia-dkms`.

The complication in the Exec line above is in order to avoid running mkinitcpio multiple times if both nvidia and linux get updated. In case this does not bother you, the Target=linux and NeedsTargets lines may be dropped, and the Exec line may be reduced to simply Exec=/usr/bin/mkinitcpio -P.

Now reboot and you're done.

### Checking if the drivers are actually installed.

Run the command: `nvidia-smi`. \
This should show your GPU name and some other information in the output.

> [!NOTE]
> For more information on Hardware accelerated video decoding/encoding and few other NVIDIA-related topics, refer to [this page](https://wiki.archlinux.org/title/NVIDIA) on ArchWiki, or forums or try googling.

# Additional Links:

- https://wiki.archlinux.org/title/Installation_guide
