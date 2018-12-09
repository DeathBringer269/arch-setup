# Assumptions

This repository contains scripts that will install ArchLinux on laptop with full disk encryption and Secure Boot enabled.

Installation scripts assume following configuration:

* Laptop has [Intel GPU][i915] with >1080p display
* One [NVMe][] drive that will be fully used for ArchLinux
* BIOS is capable of [UEFI][] [Secure Boot][]
* [systemd-boot][] will be used as bootloader
* NVMe drive will be using [full disk encryption][FDE] for root and swap partitions
* Root partition will use zstd compressed [btrfs][] with separate subvolumes for `/`, `/home`, `/var/log` and `/var/cache/pacman/pkg`
* Setup will use wifi device for network connection
* [systemd-networkd][], [systemd-resolved][] and [iwd][] will be used for network after installation
* NVMe trim will be enabled as a [systemd timer][periodic-trim]
* Enable larger font for [console][]
* One non-root user with access to [sudo][] with [bash][] shell and auto-login on boot
* Password for root user will be [disabled][no-root-password]
* [powertop][] autotune will be enabled on boot
* Various extra tweaks

# Installation

0. BIOS preparation.

    * delete default Secure Boot keys / disable Secure Boot
    * set up BIOS password
    * boot from ArchLinux live USB stick

1. Connect to wifi.

    ```
    wifi-menu
    ```

2. Update system clock.

    ```
    timedatectl set-ntp true
    ```

3. Download contents of this repository.

    ```
    curl -sfL https://github.com/mmozeiko/arch-setup/archive/master.tar.gz | tar zxf -
    cd arch-setup-master
    ```

4. Edit the `setup.sh` and `setup-chroot.sh` files to specify parameters at top of the file.

5. Run `setup.sh`.

    ```
    # during the installation it will ask two passwords
    # first is FDE password (two times for setup, once for using)
    # second one is user login password

    ./setup.sh
    ```

6. Reboot & remove live USB stick.

    ```
    reboot
    ```

# Notes

* After running manual `mkinitcpio -p linux` you need to run `sudo /boot/sign-kernel.sh` to prepare & sign new kernel image
* On shutdown you'll see harmless error when unmounting `/var/log` subvolume

# Basic user configuration

This will:

* generate ssh key
* set up [dotfiles][] alias
* install [yay][] AUR helper
* install [kernel-modules-hook][] for keeping kernel modules after kernel upgrade
* enable [plymouth][] for splashscreen during the boot

1. Connect to wifi.

    ```
    iwctl
    # station wlp3s0 scan
    # station wlp3s0 get-networks
    # station wlp3s0 connect SSID
    # quit
    ```

2. Generate [ed25519 ssh private key][ssh-ed25519].

    ```
    ssh-keygen -t ed25519
    ```

3. Add `~/.ssh/id_ed25519.pub` to github account, setup [dotfiles][my-dotfiles] (create your own repo).

    ```
    rm ~/.bashrc
    git clone --bare git@github.com:mmozeiko/dotfiles.git ${HOME}/.dotfiles
    git --git-dir=${HOME}/.dotfiles --work-tree=${HOME} checkout
    git --git-dir=${HOME}/.dotfiles --work-tree=${HOME} config --local status.showUntrackedFiles no
    ```

4. Logout and login again to use new `~/.bashrc` file.

5. Install [yay][yay-aur] AUR helper.

    ```
    curl -sfL https://aur.archlinux.org/cgit/aur.git/snapshot/yay-bin.tar.gz | tar xzf -
    cd yay-bin && makepkg -si && cd .. && rm -rf yay-bin
    ```

6. Install [kernel-modules-hook][].

    ```
    yay -S kernel-modules-hook
    sudo systemctl daemon-reload
    sudo systemctl enable linux-modules-cleanup
    ```

7. Install [plymouth][].

    ```
    yay -S plymouth ttf-dejavu
    sudo sed -i 's/ udev / udev plymouth /' /etc/mkinitcpio.conf
    sudo sed -i 's/ encrypt / plymouth-encrypt /' /etc/mkinitcpio.conf
    sudo sed -i 's/ quiet / quiet splash /' /boot/cmdline.txt
    cat << EOF | sudo tee /etc/plymouth/plymouthd.conf
    [Daemon]
    Theme=spinfinity
    ShowDelay=0
    EOF
    sudo mkinitcpio -p linux
    sudo /boot/sign-kernel.sh
    ```

8. (optional) Get [UEFI shell][uefi-shell].

    ```
    yay -S uefi-shell-git
    sudo sbsign --key /boot/keys/db.key --cert /boot/keys/db.crt --output /boot/esp/shellx64.efi /usr/share/uefi-shell/shellx64_v2.efi
    ```

# Extra configuration

* Install [sway][] - a [Wayland][] compositor, [i3blocks][] status bar and [rofi][] for application menu.

    ```
    yay -S --needed wlroots-git sway-git i3blocks rofi rofi-dmenu j4-dmenu-desktop qt5-wayland
    ```

* Install [termite][] for terminal and [mako][] for notifications.

    ```
    yay -S termite mako
    ```

* Install [fonts][]. Enable LCD subpixel [fontconfig][] configuration for RGB pixel alignment.

    ```
    yay -S --needed ttf-bitstream-vera ttf-dejavu ttf-liberation ttf-inconsolata adobe-source-han-{sans,serif}-otc-fonts ttf-font-icons
    sudo ln -s /etc/fonts/conf.avail/10-sub-pixel-rgb.conf /etc/fonts/conf.d/
    sudo ln -s /etc/fonts/conf.avail/11-lcdfilter-light.conf /etc/fonts/conf.d/
    ```

* Install [vulkan][].

    ```
    yay -S vulkan-icd-loader vulkan-intel
    ```

* Install [VA-API][] driver for hardware accelerated video playback.

    ```
    yay -S libva-intel-driver libva-utils
    # check if it is working
    vainfo
    ```

* Install [opencl][] for Intel CPU and Intel GPU.

    ```
    yay -S ocl-icd intel-opencl-runtime compute-runtime-bin
    # check if it is working
    yay -S clinfo
    clinfo

* Install [avahi][] for resolving \*.local hostnames.

    ```
    yay -S --needed avahi nss-mdns
    sudo systemctl enable --now avahi-daemon
    sudo sed -i 's/ resolve / mdns_minimal [NOTFOUND=return] resolve /' /etc/nsswitch.conf
    ```

* Install [udiskie][] for automounting removable drives (to /media) & extra filesystems

    ```
    yay -S udiskie ntfs-3g exfat-utils f2fs-tools
    echo 'ENV{ID_FS_USAGE}=="filesystem|other|crypto", ENV{UDISKS_FILESYSTEM_SHARED}="1"' | sudo tee /etc/udev/rules.d/99-udisks2.rules
    echo 'D /media 0755 root root 0 -' | sudo tee /etc/tmpfiles.d/media.conf
    ```

* Install [Sublime Text][sublime-text] and [Sublime Merge][sublime-merge].

    ```
    curl -sfO https://download.sublimetext.com/sublimehq-pub.gpg
    sudo pacman-key --add sublimehq-pub.gpg
    sudo pacman-key --lsign-key 8A8F901A
    rm sublimehq-pub.gpg
    echo -e "\n[sublime-text]\nServer = https://download.sublimetext.com/arch/stable/x86_64" | sudo tee -a /etc/pacman.conf
    yay -Syu sublime-text sublime-merge
    ```

* Install extra packages.

    ```

    # misc utilities
    yay -S --needed tar cpio bzip2 gzip lrzip lz4 zstd lzip lzop xz p7zip unrar zip unzip
    yay -S --needed acpi sysstat lsof strace jq fzf ripgrep light nvme-cli

    # terminal software
    yay -S htop ncdu mosh tmux weechat micro-bin

    # FAR manager
    yay -S far2l-git

    # PulseAudio
    yay -S pulseaudio pulseaudio-alsa pulseaudio-bluetooth ponymix pavucontrol-qt

    # media software
    yay -S mpv youtube-dl ffmpeg-libfdk_aac mkvtoolnix-cli mkclean gpac sox

    # network software
    yay -S --needed rsync rclone tcpdump nmap socat openbsd-netcat

    # Wireguard VPN
    yay -S wireguard-dkms wireguard-tools

    # Wireshark
    yay -S wireshark-qt
    sudo gpasswd -a ${USER} wireshark

    # Docker
    yay -S docker docker-compose
    sudo gpasswd -a ${USER} docker
    sudo systemctl enable --now docker

    # Google Chrome
    yay -S google-chrome

    # Zathura pdf/djvu reader
    yay -S zathura zathura-pdf-mupdf zathura-djvu

    # VCS
    yay -S --needed git git-lfs tig subversion subversion mercurial

    # development tools
    yay -S --needed cmake ninja meson clang llvm gdb nemiver nasm
    yay -S --needed valgrind perf python-pip python-virtualenv
    yay -S --needed intel-gpu-tools renderdoc apitrace vulkan-devel opencl-headers

    # MinGW
    yay -S mingw-w64-binutils mingw-w64-headers mingw-w64-headers-bootstrap mingw-w64-gcc-base mingw-w64-crt
    yay -S mingw-w64-winpthreads
    sudo libtool --finish /usr/x86_64-w64-mingw32/lib
    yay -S mingw-w64-gcc mingw-w64-clang

    # QEMU
    yay -S qemu qemu-arch-extra qemu-user-static-bin

    # Android stuff
    yay -S android-udev android-tools android-bash-completion

    # Unity3D & Visual Studio Code
    yay -S unity-editor visual-studio-code-bin dotnet-runtime dotnet-sdk msbuild-stable mono

    # Other software
    yay -S pinta gimp dia inkscape calibre
    yay -S tor-browser

    # TODO: lm_sensors bc
    # TODO: imv grim wlstream cmus ufw libreoffice-fresh
    # TODO: steam wine
    ```

# TODO

* https://wiki.archlinux.org/index.php/GTK%2B#Themes
* https://wiki.archlinux.org/index.php/snapper
* https://wiki.archlinux.org/index.php/Power_management#Power_management_with_systemd

[i915]: https://wiki.archlinux.org/index.php/Intel_graphics
[NVMe]: https://wiki.archlinux.org/index.php/Solid_state_drive/NVMe
[UEFI]: https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface
[Secure Boot]: https://wiki.archlinux.org/index.php/Secure_Boot
[systemd-boot]: https://wiki.archlinux.org/index.php/Systemd-boot
[FDE]: https://wiki.archlinux.org/index.php/Disk_encryption
[btrfs]: https://wiki.archlinux.org/index.php/btrfs
[systemd-networkd]: https://wiki.archlinux.org/index.php/systemd-networkd
[systemd-resolved]: https://wiki.archlinux.org/index.php/systemd-resolved
[iwd]: https://wiki.archlinux.org/index.php/iwd
[periodic-trim]: https://wiki.archlinux.org/index.php/Solid_state_drive#Periodic_TRIM
[console]: https://wiki.archlinux.org/index.php/Linux_console#Fonts
[sudo]: https://wiki.archlinux.org/index.php/sudo
[no-root-password]: https://wiki.archlinux.org/index.php/sudo#Disable_root_login
[powertop]: https://wiki.archlinux.org/index.php/Powertop
[yay]: https://github.com/Jguer/yay
[yay-aur]: https://aur.archlinux.org/packages/yay/
[uefi-shell]: https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#Obtaining_UEFI_Shell
[ssh-ed25519]: https://wiki.archlinux.org/index.php/SSH_keys#Ed25519
[bash]: https://wiki.archlinux.org/index.php/bash
[my-dotfiles]: https://github.com/mmozeiko/dotfiles
[dotfiles]: http://lukehinds.com/2016/09/02/dotfiles-without-symlinks.html
[kernel-modules-hook]: https://aur.archlinux.org/packages/kernel-modules-hook/
[plymouth]: https://wiki.archlinux.org/index.php/Plymouth
[sway]: https://wiki.archlinux.org/index.php/Sway
[i3blocks]: https://vivien.github.io/i3blocks/
[rofi]: https://wiki.archlinux.org/index.php/rofi
[Wayland]: https://wiki.archlinux.org/index.php/Wayland
[termite]: https://wiki.archlinux.org/index.php/termite
[mako]: https://wayland.emersion.fr/mako/
[fonts]: https://wiki.archlinux.org/index.php/Fonts
[fontconfig]: https://wiki.archlinux.org/index.php/Font_configuration
[vulkan]: https://wiki.archlinux.org/index.php/vulkan
[VA-API]: https://wiki.archlinux.org/index.php/Hardware_video_acceleration#Intel
[opencl]: https://wiki.archlinux.org/index.php/OpenCL
[avahi]: https://wiki.archlinux.org/index.php/avahi
[udiskie]: https://wiki.archlinux.org/index.php/Udisks
[sublime-text]: https://www.sublimetext.com/
[sublime-merge]: https://www.sublimemerge.com/
