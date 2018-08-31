# Instructions to create Ubuntu 16.04 LTS install media for ASUS EeeBook X205TA

After Ubuntu 16.04.2 updated fresh installs to have Linux kernel 4.8.0, which broke several things such as the keyboard, and many asking if I can just provide an ISO for the X205TA, I have changed this doc to give instructions on how to create an ISO with X205TA changes and Linux kernel 4.10. I also provided the resulting ISO. I provided the instructions on how to build the ISO from trusted sources because it can be dangerous running code from someone you don't know.

## Installation on X205TA

### BIOS setup

* Make sure the X205TA is off
* Plug in the USB flash drive
* Start the X205TA and continue to press `F2` to get into BIOS.
* Under `Advanced` tab, `USB Configuration` -> `USB Controller Select` set to `EHCI` otherwise mouse and keyboard will not work
* Under `Security` tab, `Secure Boot menu` -> `Secure Boot Control` set to `Disabled`. Otherwise, you may get a **SECURE BOOT VIOLATION** on boot. This is due to the factory defaut keys not working for Ubuntu's EFI bootloader. Instead of disabling `Secure Boot Control`, you can 1) `Key Management` -> `Delete All Secure Boot Variables` or 2) `Key Management` and manually load all of the variables.  Disabling `Secure Boot Control` was easier. [More information on SecureBoot and Ubuntu](http://web.dodds.net/~vorlon/wiki/blog/SecureBoot_in_Ubuntu_12.10/)
* Under `Save & Exit` tab, `Save Changes` (NOT `Save Chances and Exit`)
* Lastly, while still in `Save & Exit` tab, under `Boot Override`, select the USB flash drive.

### Installation

安装已经编译好的内核即可。  （内核更新的话可以查看文件列表最新内核安装）
sudo wget https://github.com/cchuyacc/instructions/blob/master/X205TA-kernel-sound-64bit.tar  
sudo tar xvf X205TA-kernel-sound-64bit.tar  
sudo ./install-sound-kernel.sh  


## *EXPERIMENTAL* Kernel changes for audio support  
Add some kernel boot parameters for the grub bootloader to use:    
in the file /etc/default/grub ; append the line:    
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"  
to:  
    GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_idle.max_cstate=1 button.lid_init_state=open"  
    #(intel_idle.max_cstate=1 to prevent freezes)  
    #(button.lid_init_state=open to prevent a suspend loop after closing/opening the lid)  



##下面是如何自己编译内核的方法(完成上面安装即可，下面是自己编译内核的方法介绍)：

#STEP 1: Install prerequisites
sudo apt-get update && sudo apt-get install -y build-essential fakeroot libncurses5-dev libssl-dev ccache dialog libelf-dev bc ### ubuntu/mint/debian/etc.
#or
sudo pacman -Sy --noconfirm wget base-devel ncurses bc dialog openssl ### arch/manjaro/antergos
#or
sudo dnf -y groupinstall "C Development Tools and Libraries" && sudo dnf -y install ncurses-devel elfutils-libelf-devel dialog openssl-devel patch ### fedora,etc.

#STEP 2: Download, extract and change directory into source tree:
wget https://git.kernel.org/torvalds/t/linux-4.15-rc6.tar.gz
tar linux-4.15-rc6.tar.gz
cd linux-4.15-rc6

#STEP 3: Patch the kernel sources:
#patch no. 1 (attempts to prevents baytrail kernel freezes - thanks to jbMacAZ)
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/fix_c-state_patch_v4.15.patch
#patch -p1 --dry-run < fix_c-state_patch_v4.15.patch #run a --dry-run for each patch to test it first
patch -p1 < 4.14-patches/fix_c-state_patch_v4.15.patch

patch no. 2 (reverts several files to pre 4.15-rc1 state to enable suspend with power led off - likely a more energy efficient suspend state)
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/Revert-several-pm-4.15-rc1-merges-for-low-power-suspend.patch
patch -p1 < Revert-several-pm-4.15-rc1-merges-for-low-power-suspend.patch
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/Revert-several-pm-4.15-rc1-merges-for-low-power-suspend-2-rc6.patch
patch -p1 < Revert-several-pm-4.15-rc1-merges-for-low-power-suspend-2-rc6.patch

#patch no. 2 (hide unnecessary mmcblkXrpmb block devices, which can cause hangups)
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/rpmb.patch
patch -p1 < rpmb.patch

#patch no. 3 (hide dmesg errors of brcmfmac trying to enable a p2p device)
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/brcmfmac-p2p-and-normal-ap-access-are-not-always-possible-at-the-same-time.patch
patch -p1 < brcmfmac-p2p-and-normal-ap-access-are-not-always-possible-at-the-same-time.patch

#patch no. 4 (suppresses deprecated hwmon warnings in dmesg)
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/fix-null-hwmon-info.patch
patch -p1 < fix-null-hwmon-info.patch

#patch no. 5 (bug fix - start-up race condition)
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/i2c_touch_fix_initialize_delay.patch
patch -p1 < i2c_touch_fix_initialize_delay.patch

#optional: patch no. 6 (this adds headphones support, but this can also be accomplished with a config file in kernels >=v4.13)
wget https://raw.githubusercontent.com/harryharryharry/x205ta-patches/master/4.15-patches/ASoC-rt5645-add-quirk-for-ASUS-EeeBook-X205TA.patch
patch -p1 < ASoC-rt5645-add-quirk-for-ASUS-EeeBook-X205TA.patch
#or add:
options snd_soc_rt5645 quirk=0x31
blacklist snd_hdmi_lpe_audio #(only necessary if this module is build, but better safe than sorry)
blacklist btsdio #(not necessary for headphones support, but this module breaks wifi during suspend on the x205ta so it makes sense to blacklist it here anyway)
#to /etc/modprobe.d/50-x205ta.conf

#source for quirk: https://bugzilla.kernel.org/show_bug.cgi?id=95681#c243
#source for blacklisting snd_hdmi_lpe_audio: https://bugzilla.kernel.org/show_bug.cgi?id=95681#c218

#optional: add command queue support for mmc (for bfq scheduler):
https://github.com/harryharryharry/x205ta-patches/tree/master/4.15-patches/bfq-patches

#STEP 4: configure, compile and install the kernel
#configure
wget http://x205ta.myftp.org:1337/kernel/.config -O ./.config ### Download my kernel config
make menuconfig ### to change kernel configuration (optional)

#compile - this will take a while
make -j6

#install kernel modules to /lib/modules
sudo make modules_install

#optionally install kernel headers
sudo make headers_install

#set variable KERNELRELEASE as kernelversion to use when installing kernel and initramfs
export KERNELRELEASE=$(<include/config/kernel.release)

#copy kernel to /boot
sudo cp -va arch/x86/boot/bzImage /boot/vmlinuz-$KERNELRELEASE

#generate initramfs to /boot
sudo update-initramfs -c -k $KERNELRELEASE ### ubuntu/mint/debian/etc.
#or
sudo mkinitcpio -k $KERNELRELEASE -c /etc/mkinitcpio.conf -g /boot/initramfs-$KERNELRELEASE.img ### arch/manjaro/antergos
#or
sudo dracut -fv /boot/initramfs-$KERNELRELEASE.img $KERNELRELEASE ### fedora,etc.

#rebuild grub
from 4.15-rc6 on, the linux kernel includes a patch to address the recently discovered Meltdown vulnerability which 
makes the kernel safer but also degrades its performance by quite a bit. I would advise you to accept this performance degradation, 
but if you prefer performance over safety you can disable the page table isolation (pti) by adding the kernel parameter pti=off to /etc/default/grub :
(between the quotes after GRUB_CMDLINE_LINUX_DEFAULT) before (re)generating grub.cfg
sudo update-grub ### ubuntu/mint/debian/etc.
#or
sudo grub-mkconfig -o /boot/grub/grub.cfg ### arch/manjaro/antergos
#or
sudo grub2-mkconfig -o /boot/grub2/grub.cfg ### fedora,etc.

#download UCM-files to /usr/share/alsa/ucm/chtrt5645
sudo mkdir -p /usr/share/alsa/ucm/chtrt5645
sudo wget https://raw.githubusercontent.com/plbossart/UCM/master/chtrt5645/HiFi.conf -O /usr/share/alsa/ucm/chtrt5645/HiFi.conf
sudo wget https://raw.githubusercontent.com/plbossart/UCM/master/chtrt5645/chtrt5645.conf -O /usr/share/alsa/ucm/chtrt5645/chtrt5645.conf


#STEP 5: install pavucontrol (optional) to change sound output (headphone jack detection does not work yet)
sudo apt-get install -y pavucontrol ### ubuntu/mint/debian/etc. (debian also needs the package firmware-intel-sound which is in the non-free repository)
#or
sudo pacman -Sy --noconfirm pavucontrol ### arch/manjaro/antergos
#or
sudo dnf -y install pavucontrol ### fedora,etc.

#STEP 6: reboot and select the kernel during boot
#STEP 7: run 'pavucontrol' and select 'speakers' instead of 'headphones'
```

## CLEAN install of Windows 10

If you wish to do a CLEAN install of Windows 10, please follow [these instructions](windows10-install-asus-x205ta.md)
