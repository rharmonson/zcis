# How to install Arch Linux

Guiding principle when working with complex systems: KISS; Keep It Simple, Stupid!

## Arch Linux installation media

Use the provide CD ISO and USB flash drive to create a Arch Linux USB boot installer.


Using "lsblk" to identify the flash device and "dd" to write the image to the flash drive.

```
lsblk
dd bs=4M if=path/to/archlinux.iso of=/dev/sdx status=progress oflag=sync
```

The dd command distroys all data for the destined device. Use carefully!

Alternatively, you can use Etcher. A safer alternative with a GUI.

Etcher
https://www.balena.io/etcher/

## BIOS or UEFI?

Begin by identifying if your system board supports UEFI by reviewing the provided technical specification sheet.

Do not assume UEFI is in use even if the system board supports UEFI. System boards often permit disabling UEFI using a BIOS or Legacy option in their firmware--firmware is often referred as BIOS.

The importance of identifying UEFI is it requires an additional partition to install.

## Initial boot

Connect the USB boot installer to an available USB port on your test workstation.

1. Power on workstation
1. Wait for the boot screen to be displayed
1. Depress FN + F8 keys
1. Select the UEFI boot option for SanDisk Cruzer Glide
1. Select Boot Arch Linux (x86_64)

To verify the workstation booted using UEFI, use "ls" to check for "efivars." If it exists, UEFI is in use.

`ls /sys/firmeware/efi/efivars`

## Network Interface

We are using DHCP, verify your network cable is connected, bring up the network interface, then test.

```
ip link
ping archlinux.org
```

Use ctrl+c to quit ping.

# System Time

Enable NTP to update the system time.

`timedatectl set-ntp true`

## Create partitions

Use `fdisk -l` view current devices. Using device sda as an example, we will create the following partitions for an UEFI installation which differs for a BIOS installation.

Mount   Device      Description	FS		Type		Size
-------------------------------------------------------------
/boot   /dev/sda1   EFI	        fat32	EFI			512M
/       /dev/sda2   root        ext4	Linux FS	50G
[swap]  /dev/sda3   swap        none	Linux Swap	6G
/home   /dev/sda4   home        ext4	Linux FS	remaining space

Begin by using wipefs to clean the target installation disk then start a text user interface "cfdisk" to create partitions.

```
wipefs -a /dev/sda
cfdisk /dev/sda
```

1. GPT
1. New & 1024M
1. Type & EFI System
1. Select Free space
1. New & 50G
1. Select Free space
1. New & 6G
1. Type and Linux swap
1. New & remaining space
1. Write and type "yes"
1. Quit

"Write" is responsible for saving our work. If you make a mistake, use "Quit" to exit without saving.

## Format partitions

Format partions using ext4 and swap file systems.

```
mkfs.fat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
mkfs.ext4 /dev/sda4
mkswap /dev/sda3
swapon /dev/sda3
```

---

* Note *

If you receive notification that data or a label exist, accept the default to proceed. You did select the right partition after all. Right?

---

## Mount partitions

Mount partitions in preparation for Arch Linux installation.

```
mount /dev/sda2 /mnt
mkdir /mnt/boot
mkdir /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
mkdir /mnt/home
mount /dev/sda4 /mnt/home
```

## mirrorlist

Obtain and enable a current list of servers for the United States prior to installing.

```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.old
curl -o /etc/pacman.d/mirrorlist "https://www.archlinux.org/mirrorlist/?country=US&protocol=https&ip_version=4&use_mirror_status=on"
sed -i 's/#Server/Server/g' /etc/pacman.d/mirrorlist
```

## Install

`pacstrap /mnt base linux linux-firmware grub efibootmgr cronie iproute2 iputils man-db man-pages nano networkmanager pulseaudio pulseaudio-alsa reflector sudo texinfo vi vim`

## fstab

Generate configuratin file responsible for mounting filesystem on boot.

`genfstab -U /mnt >> /mnt/etc/fstab

## chroot

Change the root directory to /mnt in preparatin for installing.

`arch-chroot /mnt`

## Time zone

Set the time zone.

```
ln -sf /usr/share/zoneinfo/US/Los_Angeles /etc/localtime
hwclock --systohc
```

## Localization

Update "locale.gen" to use "en_US.UTF8 UTF-8" by removing "#."

```
nano /etc/locale.gen
en_US.UTF-8 UTF-8
```

## Network configuration

Using hostname little, complete the following.

```
echo "little" >> /etc/hostname
```

Also, update /etc/hosts

```
nano /etc/hosts
```

Update to have

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   little.localdomain  little
```

## NetworkManager

Enable NetworkManager so on reboot we have a working network.

`systemctl enable NetworkManager`

## reflector

Update mirrorlist with a subset of US servers.

```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
reflector --country US --age 12 --protocol https --sort rate --save /etc/pacman.d/mirrorlist
```

## root password

Set the root user password.

```
passwd root
```

## Create a user account

Before continuing, create a user account with administrator privilges.

```
useradd -m zeth
usermod -aG wheel zeth
passwd zeth
visudo
```

Remove the "# " for "# %wheel ALL=(ALL) ALL" then type `:wq` to save your changes.

## Boot loader

Install and configure grub boot loader.

```
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

## Complete install

Exit chroot, unmount, and shutdown.

```
exit
umount -f /mnt/boot/efi
umount -r /mnt/home
umount -r /mnt
poweroff
```

# Install Desktop Environment (DE)

Log on to Arch Linux using your user account, zeth.

## Display driver

Identify which display driver to use.

`sudo lspci | grep -e VGA -e 3D`

For the "Intel Corporation HD Graphics 630" support, we will install "xf86-video-intel" and "mesa" driver packages.

## Install DE packages

Install drivers, X server, XFCE4, LightDM, LightDM GTK+ greeter, etc.

```
sudo pacman -S xf86-video-intel mesa xorg xfce4 xfce4-goodies lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings network-manager-applet gvfs pavucontrol firefox
```

Enable the new display manager.

```
sudo systemctl enable lightdm 
```

Before enabling the DE on boot, test using startxfce4.

`startxfce4`

If you successfully launch xfce, then open a terminal using the bottom dock and use systemctl to enable graphical.

`sudo systemctl set-default graphical.target`

Select your user name in the upper-right corner --> Log Out --> Reboot. After reboot, logon to the DE in preparation for customizing the DE.

---

* Did you know? *

The checkmark option "Save session for future logins" is used when making wanting to save the current DE state including applications? If you don't want to use the feature, uncheck the feature on "Log Out."

---

References
https://wiki.archlinux.org/index.php/Xorg#Driver_installation
https://wiki.archlinux.org/index.php/Xfce

# Customize Desktop Environment

Customize the newly installed XFCE4 Desktop Environment (DE).

## Terminal

Before we start, we need to identify our Terminal Emulator which is how we access the command line in our DE for executing programs. At the bottom of the screen, we have a bar with icons. The bar is called a dock. Go ahead and click on the icons on the bottom doc to see what they do.

In order from left to right, the icons are:

1. Display Desktop - minimizes applications windows
1. Terminal Emulator - shell
1. File Manager also known as Thunar or Thunar File Manager
1. Web Browser - Firefox
1. Find or Application Finder
1. Home - list of locations to open Thunar

The second icon from the left that looks like a black square with ">_" is XFCE4 Terminal Emulator. It is the application used to execute commands just like we did before installing the XFCE4 Desktop Environment (DE). Prior to the DE, we were presented with the default "shell" on login. The default system shell is bash or Bourne Again SHell. Moving forward, the term "shell" will be used instead of Terminal.

## Desktop: Background

First, let's create a place to save our pictures. Select the Terminal icon to open a shell and type the following commands.

```
mkdir ~/Pictures
cd ~/Pictures
exit
```

Next, open Firefox using the bottom dock. Once open, go to "duckduckgo.com" and search using the terms "dragon" and "1366x768." Select Images to view the pictures resulting from the search.

1366x768 is used due to the the physical display limit of the monitor. Larger or smaller images can be used, but may not display correctly. Experiment! 

Download several images you like to /home/zeth/Pictures then close Firefox.

To use the new images, select

Applications --> Settings --> Desktop --> Background

Select the "Folder" dropdown --> Other... --> Home --> zeth --> Pictures --> Open button

The downloaded images in your Pictures directory are now available to select. Select one and experiement with the options until satisfied. When done, select the "Close" button.

## Desktop Icons

Let's remove unneeded icons to unclutter are desktop.

Applications --> Settings --> Desktop --> Icons tab

Disable or uncheck each item under "Default Icons" the select the "Close" button. These can be renabled at any time.

## Appearance

Open a shell and install packages.

`sudo pacman -S moka-icon-theme adobe-source-code-pro-fonts adobe-source-sans-pro-fonts`

Navigate to Appearance

Applications --> Settings --> Appearance

    Style: Adwaita-dark
    Icons: Moka
    Fonts
        Default Font: Source Sans Pro Regular 12
        Default Monospace Font: Source Code Pro Regular 12
    Select the "Close" button

## Screensaver

Select a screensaver.

Application --> Settings --> Screensaver

    Theme: Floating Xfce
    Regard the computer idel after: 15 minutes
    Select "Close" button

## Window Manager

Applications --> Settings --> Windows Manager

    Style
        Theme: Cruxish
        Title font: Source Sans Pro Regular 10
        Title alignment: Left
        Button layout: remove minimize and maxamize
    Select the "Close" button

Using Tweaks, there are additional items to customize.

Applications --> Settings --> Window Manager Tweaks

    Cycling: Enable "Cycle through windows in a list"
    Compositor: Disable "Show shadows under dock windows"
    Select the "Close" button

## Panel

With the removal of the Trash icon from the desktop, add a Trash applet on Panel to access Trash for recovering or clearing deleted items.

Applications --> Settings --> Panel --> Items tab

	Items Tab
		1. Select "+" button
		1. Type "trash" in the Search box
		1. Select "Trash Applet"
		1. Select "Add" button
		1. Use the up and down arrow button to place it above the Separator and Clock applet
		1. Select the "Close" button
		1. Done!

---

* Note *

Package gvfs is a dependency for the Trash applet. If gvfs is not installed, the applet will display a red icon with a exclamation mark! You installed gvfs in the section titled "Install Desktop Environment."

---

## Terminal

Open a shell --> Edit menu --> Preferences

    Appearance
        Font: Source Code Pro Regular 12
        Background
            Change from "None" to "Transparent background"
            Opacity: 0.65
        Select the "Close" button

## LightDM GTK+ Greeter Settings

Applications --> Settings --> LightDM GTK+ Greeter Settings

    Appearance
        Theme: Adwaita-dark
        Icons: Moka
        Font: Source Sans Pro Regular 12
        User image: disable
    Select the "Save" button
    Select the "Close" button

Customization is complete!

# Applications

Arch Linux has thousands of applications. The system utility pacman, Arch Linux's package manager, is the most common means to install, update, and remove software. 

## pacman

List of common pacman options and explanations. Use of "pckg" represent the package name to be managed.

* pacman -Q pckg - displays installed packages
* pacman -Ql pckg - lists all packages
* pacman -Qm - lists manually installed packages
* pacman -Rns pckg - removes package, its dependencies and configuration files
* pacman -S pckg - install package
* pacman -Si pckg - shows package information
* pacman -Ss name - searches packages where name is a string
* pacman -Sy - update pacman database, -Syy force update
* pacman -Syu - complete package upgrades
* pacman -U pckg.tar.xz - installs a package from a file

The most common options are install (S), search (Ss), update (Syu), and remove (Rns).

## Install a calculator

Use pacman and update the local pacman database, then install a calculator application called galculator. Note the use of "-Sy" need only done if there are packages updates which obsoletes packages in the local pacman database. As a habit, advise "pacman -Sy" if it has been more than a day and installing or updating packages.

```
sudo pacman -Sy
sudo pacman -S galculator
```

The application galculator is now available for use and if found under accessories.

---

* Note *

When installing packages and receiving 404 errors, it may indicate the package doesn't exist due to an update. Refresh the pacman database using `sudo pacman -Sy`.

---

## Additional applications

The list format below gives the application category, application name, and application package. Use pacman to install each application. Review the syntax for galculator above as a reminder.

    Category
        Applicaton Name: package-name

Please install the following applications using pacman. Skip items between "( )" for now. We will install them using alternatives to pacman, later. Also, in the course of our installatin, we have installed several applications. They are denoted with "<-- we already installed" and provided as a reference for your future Arch Linux installations.

------------------------------------------------------------------------------
Calculator
    Galculator: galculator  <-- we already installed

CD/DVD Image Burning
    xfburn: xfce4-goodies <-- we already installed
	
CD/DVD Backup
	(MakeMKV: flatpak install org.makemkv.MakeMKV)
	
CD/DVD Encoding
	Handbrake: handbrake

Developer tools
    Git: git
    Common developer tools: base-devel
    (Sublime Text (IDE): flatpak install com.sublimetext.three)
	(Sublime Merge: flatpak install com.sublimemerge.App)

    Both git and base-devel are needed to use the Arch User Repository (AUR).

eBook Manager
    (Calibre: calibre)

Games
    (Steam: flatpak install com.valvesoftware.Steam)


Internet browser
    Firefox: firefox <-- we did this in the example above, 
    Chromium: chromium

Image editor
    GIMP: gimp

Image viewer:
    Ristretto Image Viewer: xfce4-goodies <-- we already installed

Music editor
    Audacity: audacity

Music meta-tag editor
    (Puddletag: yay -S puddletag)

Music player
    DeaDBeeF: deadbeef
    Parole: xfce4-goodies <-- we already installed

Notes
    Notes: xfce4-goodies <-- we already installed

Office suite - word processor, spreadsheet, diagrams, presentation, and other business productivity software.
    LibreOffice: libreoffice-fresh

Recording (Audio/Video)
    (OBS Studio: obs-studio)

Themes:
    ARC: arc-gtk-theme, arc-icon-theme
    Faba: faba-icon-theme <-- we already installed
    Moka: moka-icon-theme <-- we already installed
    Papirus: papirus-icon-theme (derived from paper theme)

Text editor
    Mousepad: xfce4-goodies <-- we already installed

Timer
    XFCE Timer: xfce4-goodies <-- we already installed
        Remember to create you timers before first use!

Video editor
    (Shotcut: flatpak install org.shotcut.Shotcut)

Video player
    Kodi: kodi
	VLC: vlc
    Parole: xfce4-goodies <-- we already installed
------------------------------------------------------------------------------

Applications installs using pacman are complete!

# Alternatives to pacman

## Flatpak

Flatpak is an alternative to pacman where applications are abstracted and partially sandboxed. This permits an application flatpak run on differing Linux distributions.

Reference:
https://wiki.archlinux.org/index.php/Flatpak

### Install flatpak

Open a shell and execute

`sudo pacman -S flatpak`

When prompted to select "xdg-desktop-portal-impl" accept the default for "gtk" and "y" for "Proceed with installation."

After the installation, close the shell and a new shell. This is to inherit the new global environment variables.

Install the flatpak repository flathub.

```
sudo flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
```

### Using flatpak

To obtain and update "appstream" information from flathub, use the "update" option.

`sudo flatpak update`

After the update, you can query or search the flathub using "search."

`sudo flatpak search libreoffice`

To list installed flatpaks, use the "list" option.

`sudo flatpak list`

We will use the "install" option next and the complete list of options and descriptions are found using "man flatpak."

### Steam

Steam is not available using pacman for it is not in the official Arch Linux package repositories. One option to install Steam is to use a flatpak. The steps to install are as follows:

Open a shell and maximize, so you can see flatpak's output.

```
sudo flatpak update
sudo flatpak search steam <-- Note "Steam" and its App ID of "com.valvesoftware.Steam"
sudo flatpak install com.valvesoftware.Steam
```

Respond with "y" for "Use this remote" and "Do you want to install."

Steam is now ready for use!

References:
https://wiki.archlinux.org/index.php/steam#Installation
https://www.archlinux.org/packages/multilib/x86_64/steam-native-runtime/
https://wiki.archlinux.org/index.php/Steam/Troubleshooting#Steam_runtime

## MakeMKV

Install MakeMKV using flatpak.

```
sudo flatpak search makemkv
sudo flatpak install org.makemkv.MakeMKV
```

## User provided packages (AUR)

AUR stands for Arch User Repository. Archlinux.org communicates on thier AUR webpage the following warning:

"DISCLAIMER: AUR packages are user produced content. Any use of the provided files is at your own risk."

Use AUR only if an alternative package or program is not available from the official repository using pacman--this is true for flatpak, too, for they are often user created.  

Before proceeding, you must have git and base-devel installed. Both were installed as part of our earlier activities.

### yay

yay plays the same role as pacman, but it supports AUR. In fact, it is found in AUR.

To install yay, its source code must be downloaded using git then compiled using the base-devel tools.

```
mkdir ~/gitcode
cd ~/gitcode
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

When prompted, provide sudo credentials.
Respond "y" to "Proceed with installation," twice.

yay uses the same syntax as pacman, generally, and will search and update both pacman and yay packages, e.g. yay -Syu. See yay man pages using "man yay" for all options.

Reference:
https://aur.archlinux.org

### Minecraft

Games
    https://wiki.archlinux.org/index.php/List_of_games
    Minecraft: minecraft-launcher

Using yay install Minecraft.

`yay -S minecraft-launcher`

### Flat theme

Install the Flat theme using yay.

`yay -S flat-remix flat-remix-gtk`

Alternative applications installtions are complete.

# Security

Security in the context of computing or the use of computers has may facets.

## Unauthorized access & firewall

Firewalls are a preventative security control that goal is to prevent unauthorized access.

Install Uncomplicated Firewall and review its wiki webpage.

```
sudo pacman -S ufw gufw
sudo systemctl enable ufw
sudo systemctl start ufw
```

At this time there is a reported bug so no desktop icon. In the mean time, open a shell and launch `gufw` and toggle the "Status" button/toggle to enable the filewall. Close the gufw and done!

https://wiki.archlinux.org/index.php/Uncomplicated_Firewall

## Passwords

Do you have a lot of passwords? Use KeePassXC to generate random passwords, store website addresses, and retreive user names and passwords. KeePassXC is compatible with KeePass for Windows, but not the same application.

`sudo pacman -S keepassxc`

## Internet browsing & privacy 

You activity is being monitored on many websites and search engines. When using Firefox, there are number of extentions that can help protect your privacy. Don't forget to configure Firefox for better privacy too!

`sudo pacman -S firefox-ublock-origin firefox-umatrix firefox-extension-https-everywhere firefox-decentraleyes`

## Disaster recovery

Install Borg Backup, Vorta, and Timeshift. Borg is usined to create backup for your files. Timeshift is used to create snaps of system files.

```
sudo pacman -S borg
yay -S vorta
yay -S timeshift
```