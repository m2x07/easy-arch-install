# Easy Arch Install
A concise and detailed guide for effortless Arch Linux installation, offering a smooth installation journey

# 📑 Index
+ [Initial Setup](https://github.com/m2x07/easy-arch-install#1-initial-setup)
+ [Disk Configuration](https://github.com/m2x07/easy-arch-install#2-disk-configuration)
+ [Installing base system](https://github.com/m2x07/easy-arch-install#3-installing-base-system)
+ [Chrooting](https://github.com/m2x07/easy-arch-install#4-chrooting)
+ [Post Install Configuration](https://github.com/m2x07/easy-arch-install#5-post-install-configuration)
+ [Additional Links](https://github.com/m2x07/easy-arch-install#addtitional-links)

> *NOTE: wherever you see something like &lt;this&gt; (inside angular brackets), replace it with appropriate path/device name as needed*



# 1. Initial Setup
## Connecting to Internet

<details>
    <summary>Wi-Fi</summary>
Start the daemons required:

```zsh
systemctl start iwd
systemctl start dhcpcd
```

Get interface name:
```zsh
ifconfig
```

Connecting to wi-fi.
```zsh
iwctl station <interface_name> connect "ssid"
```
Hit ENTER to enter your wifi password and confirm connection
</details>

<details>
    <summary>USB Tethering</summary>
1. Connect your phone to your computer using a USB Cable<br>
2. Navigate to Portable Hotspot (or look for similar settings) section and enable the USB tethering option<br>
3. Make sure your phone is connected to the internet and now your computer should have internet access<br>
</details>

### Test your internet connection
```zsh
ping -c 3 8.8.8.8       // ensure that DHCP is working
ping -c 3 archlinux.org // ensure that DNS is working
```
If your output for both the above command looks something like this, it means that you are getting responce from the server and you are connected to the internet
```zsh
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=113 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=114 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=117 time=73.5 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2000ms
rtt min/avg/max/mdev = 73.480/100.150/114.050/18.864 ms

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
The "/dev/nvme0n1" in the first line is our disk we'll be working on. disk name may vary to something like "/dev/sda" or "/dev/sdb" or anything else. look for the disk that you want to install Arch Linux on and remember it or note it down somewhere<br>

## Partitioning:
There are various layouts and filesystems one can use to install Arch Linux
For this guide, we will be using EXT4 filesystem
We will be using `cfdisk` tool to create our partitions. anyone with a normal human brain can use it easily as it is a TUI
```bash
cfdisk /dev/nvme0n1     #replace disk name(nvme0n1) with your disk name
```
Delete any existing partions by following the on-screen option.<br>
Create 3 partions as follows:<br>

|Partition      | Size      | Filesystem|
|---------------|-----------|-----------|
|&lt;efi&gt;    | 1024M     | EFI System partition|
|&lt;swap&gt;    | same as your ram  | Linux Swap    |
|&lt;root&gt;    | remaining storage | Linux filesystem|

> Minimum of 512M is recommended for efi partition. If you will be installing multiple kernels, use atleast 1024M for efi partition. We are not going to install multiple kernel in this guide but i'm still alloting 1024M.<br>

The order doesn't matter but this is what i prefer.<br>
Write and Quit.<br>
Let's run the `fdisk -l` command again and compare the output.<br>

```
Disk /dev/nvme0n1: 476.94 GiB, 512110190592 bytes, 1000215216 sectors
Disk model: INTEL SSDPEKNU512GZ                     
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: C4F8B2FE-0074-4CC4-8445-F0C646127ACB

Device            Start        End   Sectors   Size Type
/dev/nvme0n1p1     2048    2099199   2097152     1G EFI System
/dev/nvme0n1p2  2099200   35653631  33554432    16G Linux swap
/dev/nvme0n1p3 35653632 1000214527 964560896 459.9G Linux filesystem
```
As you can see, we can see information about our newly created partitions in the last 3 lines. Remember/Note down the device names for each partition.

## Formatting:
Let's format our partitions with appropriate filesystem<br>
Like i said, there are multiple options and methods when it comes to partitioning and formatting, for this guide, we'll be covering ext4 filesystem
```bash
mkfs.fat -F 32 /dev/<efi>       # FAT32 for the efi partition
mkfs.swap /dev/<swap>           # swap partition
mkfs.ext4 /dev/<root>           # EXT4 for the root partition
```
*For BTRFS and other Filesystems, refer the friendly [ArchWiki](https://wiki.archlinux.org/title/btrfs) :)*

## Mounting:
Mount all of the partitions:
```bash
mkdir /mnt/boot
mkdir /mnt/boot/efi             # Create an EFI mount point
mount /dev/<efi> /mnt/boot/efi  # mount the EFI partition
swapon /dev/<swap>              # enable the swap partition
mount /dev/<root> /mnt          # mount the root partition
```
If you get any errors, make sure you did the partitioning correctly. Create the same partitions again if necessary

# 3. Installing base system
Now its time to install the base system and kernel on our partitions<br>
Before we do so, let's make some changes to the config file of our package manager
```bash
nano /etc/pacman.conf           # or any text editor of your preference
```
Locate and un-comment the following lines:

`Color`<br>
`ParallelDownloads = 5`<br><br>
`[multilib]`<br>
`Include = /etc/pacman.d/mirrorlist`<br>

For slower connections, you might want to change the ParallelDownloads to 3.<br>
Using the keybinds given on the bottom of the screen, Write the file and exit

Now lets install the packages and wait for the installation to finish
```bash
pacstrap /mnt base linux linux-firmware vim nano
```
> *NOTE: if you want to use any other kernel, then replace `linux` with your desired kernel. For example `linux-zen` or `linux-lts`. No need to replace the `linux-firmware` package<br>
Also, install `btrfs-progs` if you are using btrfs filesystem. it provides additional tools for btrfs filesystem*   

Now finally generate the fstab configuration
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

# 4. Chrooting

# 5. Post Install Configuration

# Addtional Links