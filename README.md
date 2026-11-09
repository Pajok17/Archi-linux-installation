
# Arch Linux Installation on VMware — A Beginner's Step-by-Step Guide

**Written by:** Pajok Maluk  
**System built:** `archi-pam` — Arch Linux with Xfce and Zsh  
**Platform:** VMware Workstation Pro 17 on Windows 11  
**Installation method:** Manual installation with the official Arch Linux ISO

Welcome! In this guide, I walk through how I installed and customized my own Arch Linux virtual machine. The goal is to make each step easy to follow for someone learning Linux: what to click, what to type, why the command is needed, and how to confirm the result. Screenshots below are from my own installation.

By the end, the VM boots into the **Xfce graphical desktop**, supports **SSH**, has **two sudo-enabled users**, uses a **colored Zsh terminal**, runs **Firefox**, and includes an **AUR package built from source**.

![My Arch Linux system running the Xfce desktop](screenshots/22-xfce-desktop.png)
*My completed Arch Linux Xfce desktop, with the Thunar file manager open.*

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Create the Virtual Machine](#2-create-the-virtual-machine)
3. [Install Arch Linux Manually](#3-install-arch-linux-manually)
4. [Configure the System](#4-configure-the-system)
5. [Install and Use the Xfce Desktop](#5-install-and-use-the-xfce-desktop)
6. [Customize Zsh and Install Firefox](#6-customize-zsh-and-install-firefox)
7. [Build a Package from the AUR](#7-build-a-package-from-the-aur)
8. [Verify the Finished VM](#8-verify-the-finished-vm)
9. [References](#9-references)

---

## 1. Prerequisites

To follow the same process, prepare:

- A Windows computer with **VMware Workstation Pro** installed.
- The official **Arch Linux x86_64 ISO** from [archlinux.org/download](https://archlinux.org/download/).
- Internet access inside the virtual machine.
- At least **4 GB of RAM** and **30 GB of disk space** available to assign to the VM.
- Basic familiarity with copying Linux commands into a terminal.

**Read before partitioning:** This guide uses an **empty, dedicated VMware virtual disk** named `/dev/sda`. The partitioning and formatting commands erase data on that disk. Verify your device name with `lsblk` before running them. Do not use these commands against your Windows drive or a disk containing files you want to keep.

### My VM configuration

| Setting | Value |
|---|---|
| VM name / hostname | `archi-pam` |
| Guest operating system | 64-bit Linux (x86_64) |
| Firmware | UEFI |
| Processors | 2 virtual CPUs |
| Memory | 4096 MB (4 GB) |
| Virtual disk | 30 GB (SCSI) |
| Networking | NAT |
| Main username | `pam` |
| Second username | `wujek` |

## 2. Create the Virtual Machine

### Step 1 — Start the VMware setup wizard

1. Open **VMware Workstation Pro**.
2. Choose **Create a New Virtual Machine**.
3. Select the Linux guest operating system with 64-bit architecture.
4. Name the VM `archi-pam`.

![Starting VMware's new VM wizard](screenshots/vm-setup-wizard.png)
*The VMware Workstation new virtual machine wizard.*

### Step 2 — Allocate the VM hardware

1. Assign **4096 MB RAM**.
2. Set **2 virtual processors**.
3. Create a **30 GB virtual disk**.
4. Use a **NAT network adapter** so the VM can access the internet through the host computer.
5. Use **UEFI** firmware for the installation.

![Setting 4 GB of VM memory](screenshots/vm-memory-4gb.png)
*Allocating 4 GB of RAM.*

![Choosing NAT networking](screenshots/vm-network-nat.png)
*Selecting NAT networking.*

![Setting the virtual disk to 30 GB](screenshots/vm-disk-30gb.png)
*Creating a 30 GB virtual disk.*

### Step 3 — Connect the Arch Linux ISO

Open **VM Settings → CD/DVD**, choose the downloaded Arch Linux ISO, and ensure it is connected at power on. Start the VM and enter the Arch live terminal. You should see a prompt similar to `root@archiso`.

![Arch installation ISO attached to VMware](screenshots/02-iso-attached.png)
*The Arch ISO attached to the VM's virtual CD/DVD drive.*

## 3. Install Arch Linux Manually

The following commands are run in the **Arch ISO live terminal as root** until the `arch-chroot` step.

### Step 4 — Check UEFI, networking, and time

```bash
ls /sys/firmware/efi/efivars                  # Confirm that the VM booted using UEFI
ping -c 4 archlinux.org                      # Send four packets to test internet access
timedatectl status                           # Check that the system clock is synchronized
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS   # Identify the empty virtual disk
```

The `ping` command should return replies. My installation disk appeared as **`/dev/sda`**, with **30 GiB** of space. The Arch ISO is a separate, read-only device.

![Arch live environment network test](screenshots/03-live-network-test.png)
*Testing the network before installation.*

![Checking the target virtual disk](screenshots/04-disk-identification.png)
*Using `lsblk` to locate the empty 30 GB disk.*

### Step 5 — Partition the disk with cfdisk

```bash
cfdisk /dev/sda  # Open the partition editor for the empty VMware disk
```

Select **GPT** as the partition label type, then create the partitions below:

| Partition | Size | Type | Purpose |
|---|---:|---|---|
| `/dev/sda1` | 1 GiB | EFI System | Files needed to boot in UEFI mode |
| `/dev/sda2` | Remaining ~29 GiB | Linux filesystem | Root filesystem `/` |

In `cfdisk`, select **New** to create each partition, choose the correct **Type**, and finish with **Write → yes → Quit**. This setup uses no separate swap partition.

![Creating the EFI partition](screenshots/cfdisk-efi-1gb.png)
*Creating a 1 GiB EFI System Partition.*

![Completed partition layout](screenshots/05-partitions-gpt.png)
*The completed GPT layout with EFI and Linux partitions.*

### Step 6 — Format the new partitions

```bash
mkfs.fat -F 32 /dev/sda1  # Create a FAT32 filesystem for EFI
mkfs.ext4 /dev/sda2      # Create an ext4 filesystem for the main system
lsblk -f                 # Verify both filesystem types
```

FAT32 is used for the EFI partition; ext4 stores Arch Linux and the user files.

![Formatting the two partitions](screenshots/06-format-partitions.png)
*Formatting the EFI and root partitions.*

### Step 7 — Mount the filesystems

```bash
mount /dev/sda2 /mnt       # Mount the root filesystem for installation
mkdir -p /mnt/boot         # Create the EFI mount directory
mount /dev/sda1 /mnt/boot  # Mount the EFI partition at /boot
lsblk                     # Confirm both mount points
```

You should see `/dev/sda2` mounted at `/mnt` and `/dev/sda1` at `/mnt/boot`.

![Partitions mounted for installation](screenshots/07-mounted-partitions.png)
*Verifying the two mount points.*

### Step 8 — Install Arch Linux with pacstrap

`pacstrap` copies the base operating system and selected packages to the mounted disk. The command below combines the packages used for this finished setup into one reproducible install command:

```bash
pacstrap -K /mnt base linux linux-firmware networkmanager nano sudo grub efibootmgr openssh zsh git
```

The key packages include the `linux` kernel, `linux-firmware`, networking tools, Zsh, SSH, and GRUB. Let the package installation complete before moving on.

![Arch packages being installed](screenshots/08-pacstrap-completed.png)
*Installing the Arch base system with `pacstrap`.*

Check the boot images:

```bash
ls -lh /mnt/boot  # Look for vmlinuz-linux and initramfs-linux.img
```

![Kernel and initial RAM filesystem](screenshots/09-kernel-initramfs.png)
*The Linux kernel and initramfs are present under `/boot`.*

### Step 9 — Generate the filesystem table

```bash
genfstab -U /mnt > /mnt/etc/fstab  # Record mounted partitions using UUIDs
cat /mnt/etc/fstab                # Read the generated table
```

This file tells Arch which partitions to mount automatically at startup.

![Filesystem table](screenshots/10-fstab-check.png)
*Checking `/etc/fstab` entries for `/` and `/boot`.*

### Step 10 — Enter the installed environment

```bash
arch-chroot /mnt  # Work inside the installed Arch system rather than the live ISO
```

From this point until `exit`, run the commands **inside the chroot as root**.

## 4. Configure the System

### Step 11 — Set the time zone and system clock

```bash
ln -sf /usr/share/zoneinfo/America/Chicago /etc/localtime  # Use Central Time
hwclock --systohc                                          # Set the hardware clock
```

Replace `America/Chicago` with your own time zone if you live elsewhere.

### Step 12 — Set the system locale

Open the locale list:

```bash
nano /etc/locale.gen  # Edit supported language and character-encoding settings
```

Find the line `#en_US.UTF-8 UTF-8` and remove the leading `#` so it reads:

```text
en_US.UTF-8 UTF-8
```

In Nano, save with **Ctrl + O**, press **Enter**, then exit with **Ctrl + X**. Run:

```bash
locale-gen                                  # Generate the selected locale
echo 'LANG=en_US.UTF-8' > /etc/locale.conf # Make English UTF-8 the default
locale -a                                   # Verify en_US.utf8 is available
```

![Locale generation verified](screenshots/11-locale-validation.png)
*Generating and verifying the US English locale.*

### Step 13 — Name the computer

```bash
echo 'archi-pam' > /etc/hostname  # Set the VM's hostname
nano /etc/hosts                 # Add local name-resolution entries
```

Use the following local host entries in `/etc/hosts`:

```text
127.0.0.1   localhost
::1         localhost
127.0.1.1   archi-pam.localdomain archi-pam
```

The hostname appears in the shell prompt after startup. You may choose a different hostname, but use it consistently in both files.

### Step 14 — Enable the network service

```bash
systemctl enable NetworkManager      # Arrange for networking to start during boot
systemctl is-enabled NetworkManager  # Verify that the result is enabled
```

![NetworkManager enabled](screenshots/12-networkmanager-enabled.png)
*NetworkManager configured to start at boot.*

### Step 15 — Install the UEFI bootloader

Because the EFI partition was mounted at `/boot`, use `/boot` as the EFI directory:

```bash
findmnt /boot  # Confirm /boot is the EFI partition
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB  # Install the bootloader
grub-mkconfig -o /boot/grub/grub.cfg  # Generate the boot menu
ls /boot/grub/grub.cfg              # Confirm the configuration exists
```

The generated GRUB menu should detect `vmlinuz-linux` and `initramfs-linux.img`.

![GRUB bootloader confirmation](screenshots/13-grub-verified.png)
*Verification of the installed GRUB configuration and Linux boot image.*

### Step 16 — Create two user accounts

The `root` account administers the machine. It is better to use a regular account for everyday work. I created `pam` and `wujek`, placing both in the `wheel` group:

```bash
passwd                                 # Set a password for root
useradd -m -G wheel -s /bin/bash pam   # Create pam with a home directory
passwd pam                             # Set the pam password
useradd -m -G wheel -s /bin/bash wujek # Create a second user
passwd wujek                           # Set the second user's password
id pam                                 # Verify pam's group membership
id wujek                               # Verify wujek's group membership
```

Choose your own account names if following this tutorial on a different VM. Password characters will not appear while typing.

![Two user accounts in the wheel group](screenshots/14-users-created.png)
*Both accounts created with `wheel` membership.*

### Step 17 — Allow wheel-group users to use sudo

```bash
EDITOR=nano visudo  # Open the sudoers file with syntax checking
```

Find the wheel rule and uncomment it:

```text
%wheel ALL=(ALL:ALL) ALL
```

Save and exit. Check the configuration:

```bash
visudo -c         # Verify valid sudoers syntax
sudo -l -U pam    # Show pam's allowed commands
sudo -l -U wujek  # Show wujek's allowed commands
```

![Enabling the wheel group in sudoers](screenshots/visudo-wheel-setting.png)
*Allowing `wheel` members to use sudo with a password.*

![Sudo permission verification](screenshots/15-sudo-verified.png)
*Confirming sudo access for both accounts.*

### Step 18 — Enable SSH

```bash
pacman -Q openssh           # Check that OpenSSH is installed
systemctl enable sshd       # Start SSH automatically on future boots
systemctl is-enabled sshd   # Verify that it is enabled
```

`sshd` is the server component that accepts SSH connections. To connect from another computer later, verify the VM's IP address and network/firewall configuration.

![SSH enabled at startup](screenshots/16-ssh-enabled.png)
*Enabling the SSH service for future boots.*

### Step 19 — Set Zsh as the main user's shell

```bash
pacman -Q zsh                  # Confirm Zsh is installed
usermod -s /usr/bin/zsh pam   # Make Zsh pam's login shell
getent passwd pam             # Verify pam ends in /usr/bin/zsh
getent passwd wujek           # Verify wujek still uses Bash
```

Zsh is a command-line shell. It supports themes, a customizable prompt, and aliases.

![Verifying Zsh and Bash account shells](screenshots/17-zsh-shells.png)
*My main user uses Zsh and the second account uses Bash.*

### Step 20 — Reboot into the installed system

```bash
exit           # Leave the chroot environment
umount -R /mnt # Unmount the installed disk from the live system
reboot         # Restart the VM
```

In VMware, disconnect the ISO so the VM boots from the virtual hard disk instead of returning to the installer. Sign in as `pam` with the password you created.

## 5. Install and Use the Xfce Desktop

These commands run **after the first boot**, signed in as `pam` with working sudo permissions.

### Step 21 — Update system packages

```bash
sudo pacman -Syu  # Refresh package databases and install system updates
```

Confirm any proposed updates after reading the package list.

### Step 22 — Install Xfce and the LightDM login manager

Xfce provides the desktop panel, window manager, file manager, terminal, and settings. LightDM provides a graphical login screen.

```bash
sudo pacman -S --needed xorg-server xfce4-session xfce4-panel xfwm4 xfdesktop xfce4-settings xfce4-terminal thunar lightdm lightdm-gtk-greeter
```

After the packages finish, confirm the Xfce session and configure the graphical startup:

```bash
ls /usr/share/xsessions/             # Check that xfce.desktop exists
sudo systemctl enable lightdm.service # Enable LightDM at startup
systemctl is-enabled lightdm.service # Verify: enabled
sudo systemctl set-default graphical.target # Boot into the GUI by default
systemctl get-default                # Verify: graphical.target
sudo reboot                          # Restart into graphical login
```

![Xfce installation confirmation](screenshots/20-xfce-install-complete.png)
*Xfce and LightDM packages installed successfully.*

![Xfce session and LightDM service checks](screenshots/21-xfce-session-confirmed.png)
*The Xfce session exists and LightDM is enabled.*

### Step 23 — Sign into the desktop

On the LightDM login screen, select **Xfce Session** if prompted, enter the username `pam`, and sign in. You should see an Xfce panel, desktop background, and the **Thunar** file manager.

![The finished Xfce desktop](screenshots/22-xfce-desktop.png)
*Arch Linux booted into Xfce and ready to use.*

## 6. Customize Zsh and Install Firefox

### Step 24 — Add terminal colors and useful aliases

Open **Applications → Terminal** and edit the Zsh configuration file:

```bash
nano ~/.zshrc  # Customize the active user's shell
```

Paste the following configuration:

```zsh
# Colorful output for common commands
alias ls='ls --color=auto'
alias ll='ls -lah --color=auto'
alias grep='grep --color=auto'

# Useful shortcuts
alias update='sudo pacman -Syu'
alias cls='clear'
alias ..='cd ..'

# Green username and blue current directory
autoload -Uz colors
colors
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f %# '
```

Save with **Ctrl + O → Enter → Ctrl + X**, then reload the settings and test the aliases:

```bash
source ~/.zshrc  # Load the updated configuration now
alias           # List defined shortcuts
ll              # Show a detailed colored file listing
cd /tmp         # Change to /tmp
..              # Go up one directory
pwd             # Verify the current directory is /
```

![Customized Zsh terminal](screenshots/23-colored-zsh-aliases.png)
*Green and blue Zsh prompt, color-coded file listing, and working directory alias.*

### Step 25 — Install the Firefox browser

```bash
sudo pacman -S firefox  # Install Firefox
pacman -Q firefox       # Verify the installed version
firefox                 # Launch the browser
```

If Pacman asks you to select packages that provide `jack` or `ttf-font`, my installation used `pipewire-jack` and `noto-fonts`. Once Firefox opens, browse to [archlinux.org](https://archlinux.org) to confirm internet access in the desktop.

![Firefox running on the Arch desktop](screenshots/24-firefox-running.png)
*Firefox successfully opened in Xfce.*

## 7. Build a Package from the AUR

The **Arch User Repository (AUR)** provides community-maintained build recipes. My example is **Yay**, a command-line AUR helper written in Go. For safety, review a package's `PKGBUILD` before building it. Build AUR packages as a regular user, not as `root`.

### Step 26 — Install build prerequisites

```bash
sudo pacman -S --needed base-devel git  # Install compilers, make tools, and Git
pacman -Q base-devel git                # Confirm they are available
```

### Step 27 — Clone the Yay repository

```bash
cd ~                                      # Go to the home directory
mkdir -p aur-builds                        # Create a folder for AUR projects
cd aur-builds                              # Enter the new folder
git clone https://aur.archlinux.org/yay.git # Download Yay's build recipe
cd yay                                    # Enter the repository
ls -la                                    # Look for PKGBUILD and .SRCINFO
```

![Downloaded Yay build files](screenshots/25-yay-cloned.png)
*The Yay repository includes `PKGBUILD` and `.SRCINFO`.*

### Step 28 — Read the build recipe

```bash
sudo pacman -S less  # Install a terminal file viewer if it is not available
less PKGBUILD        # Read the package metadata and build instructions
```

Use the arrow keys to scroll, then press **Q** to close `less`. In the file, pay attention to `source=`, `depends=`, `makedepends=`, `build()`, and `package()`.

![Yay PKGBUILD details](screenshots/26-yay-pkgbuild-review.png)
*Inspecting the AUR build instructions before installing the package.*

### Step 29 — Build and install Yay

```bash
whoami       # Verify that the current user is pam (not root)
makepkg -si  # Download dependencies, build Yay, and install its package
```

`makepkg` prompts for administrative authorization when needed, so **do not prefix `makepkg -si` with `sudo`**.

![Yay installation completed](screenshots/27-yay-installed.png)
*Yay installed as an Arch package.*

### Step 30 — Verify Yay

```bash
yay --version  # Print the Yay version
pacman -Qm yay # Show Yay as a foreign/AUR-installed package
```

My VM showed:

```text
yay v13.1.0 - libalpm v16.0.1
yay 13.1.0-1
```

![Yay version verification](screenshots/29-yay-final-verification.png)
*Verifying the Yay executable and the installed package.*

## 8. Verify the Finished VM

### Step 31 — Display the final Arch system details

Install Fastfetch if it is not present, then display the system overview:

```bash
sudo pacman -S fastfetch # Install a terminal-based system information tool
fastfetch                # Show OS, kernel, RAM, disk, and desktop details
whoami                   # Confirm active user
cat /etc/os-release      # Confirm Arch Linux
```

![Arch Linux Fastfetch output](screenshots/28-fastfetch.png)
*Fastfetch showing the running Arch Linux VM and virtual hardware.*

### Final setup checklist

| Feature | Verification command or action |
|---|---|
| Arch installed | `cat /etc/os-release` |
| Correct user | `whoami` → `pam` |
| Two users created | `id pam` and `id wujek` |
| Sudo configured | `sudo -l` |
| SSH enabled | `systemctl is-enabled sshd` |
| Zsh shell | `echo $SHELL` |
| Xfce desktop | Log in to Xfce through LightDM |
| GUI boots by default | `systemctl get-default` → `graphical.target` |
| Terminal shortcuts | `ll`, `alias`, `..` |
| Browser | Launch Firefox and open a website |
| AUR package | `yay --version` and `pacman -Qm yay` |
| System overview | `fastfetch` |

## 9. References

- [Official Arch Installation Guide](https://wiki.archlinux.org/title/Installation_guide)
- [Arch Wiki — GRUB](https://wiki.archlinux.org/title/GRUB)
- [Arch Wiki — Xfce](https://wiki.archlinux.org/title/Xfce)
- [Arch Wiki — NetworkManager](https://wiki.archlinux.org/title/NetworkManager)
- [Arch Wiki — Arch User Repository](https://wiki.archlinux.org/title/Arch_User_Repository)
- [GitHub Pages documentation](https://docs.github.com/en/pages)

---

*This beginner guide is based on the Arch Linux virtual machine I created and the screenshots captured during installation. For different hardware, check the official Arch Wiki and adjust device names, time zone, and VM settings accordingly.*
