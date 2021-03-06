# ASUS ROG Zephyrus-GA502DU Arch Linux
The ROG Zephryus GA502DU has quite a bit of issues out of the box after installing Arch. This GitHub readme should provide everything you need for this laptop to work flawlessly under Arch Linux.
<img src="https://i.imgur.com/w3UvUaX.png" width="1024">


## 1. Installing prerequisites
  - Install `asus-nb-ctrl-git`, `rog-core`, `rogauracore-git` and `optimus-manager-qt-git` via AUR
    - `yay -S rog-core asus-nb-ctrl-git rogauracore-git optimus-manager-qt-git`
## 2. Installing NVIDIA Proprietary drivers
#### I recommend using NVIDIA's stable driver branch as it has PRIME power management fixes since `460.39`
#### Use DKMS as we will be installing a patched kernel
  - Using TKGs `nvidia-all` PKGBUILD
    - ` git clone https://github.com/Frogging-Family/nvidia-all.git`
    - ` cd <yourclonedir>/nvidia-all`
    - `makepkg -sif` 
    - Choose the latest stable version (2nd from the top) (460.39 as of this READMEs creation)
    - Run `mkinitcpio` to regenerate initramfs
      - `sudo mkinitcpio -P`
    - Reboot
 ## 3. Realtek Wi-Fi/Bluetooth
 #### Realtek chips are a bit buggy on linux and most modern chips are not supported in the mainline kernel, so we will have to install a 3rd-party driver off the AUR

### DKMS Driver
 #### You will need some form of internet access to build this, either Ethernet if it works or USB tethering via your Phone
- `yay -S rtl8821ce-dkms-git`
- Reboot

### Replacing the Wi-Fi Card
I replaced mine with an [Intel® Wi-Fi 6 AX200](https://www.amazon.com/gp/product/B07SH6GV5S/ref=ppx_yo_dt_b_asin_title_o00_s00?ie=UTF8&psc=1), everything works out of the box and provides a more fast and stable experience compared to the Realtek chip.

 ## 4. Patching the linux kernel
 #### A kernel bug involving `i8042` and `asus-nb-wmi` refusing to load will make the CPU frequency lock to 400MHz and will need the following [Patch](https://raw.githubusercontent.com/YHNdnzj/linux-zen-g14/master/i8042.patch) to fix it
 #### It is preferred to do this on another PC if possible as the 400MHz issue will make kernel compiling very slow
 ### `linux-zen` also has the fix and is the preferred option now
   - `sudo pacman -S linux-zen linux-zen-headers`

 ### [Pre-built pacman package](https://github.com/sambow23/ROG-Zephyrus-GA502DU-Arch-Linux/tree/main/kernel-builds)
 
 ### Building
   - Using TKGs `linux-tkg` PKGBUILD
     - `git clone https://github.com/Frogging-Family/linux-tkg.git`
     - `cd <yourclonedir>/linux-tkg`
     - Make a patches direcory for the CPU Frequency patch
     - `mkdir linux59-tkg-userpatches`
     - `cp <patchlocation> ~/linux-tkg/linux59-tkg-userpatches/cpu.mypatch`
     - We can now begin compiling the kernel
     #### We'll be using Linux 5.9 until PRIME switching has been fixed on newer kernels
     - `makepkg -sif`
     - Choose kernel `5.9` 
     - `Custom` for `predefined optimized profile`
     - `CPU sched variant`, `UPDS` has a good balance between battery life, performance, and system responsiveness, so we'll be using that
     - `GCC` for the compiler
     - `Round Robin interval` select the `PDS` default
     - `periodic ticks` for `CattaRappa` mode
     - `y` for `explicit preemption points`
     - You can ignore the rest until it asks you to enable `fsync`, select `y`
     - It will then ask you to install the patch, select `y`
     - `CPU Optimizations` select `11. AMD Zen (MZEN)`
     - `SMT (Hyperthreading) aware nice priority and policy support` select `y`
     - `Timer Frequency` select `4. 500 HZ (HZ_500)`
     - `Optional* For advanced users - Do you want to use make menuconfig or nconfig to configure the kernel before building it?` select `0. nope`
     #### The kernel will now start building, this might take a while ;)
     #### Go ahead and update your bootloader once its done building and installed.
     ##### Grub method
     - `sudo update-grub`
     - reboot
     
## 5. NVIDIA PRIME Setup
  - Open the `Optimus Manager` application
  - Copy the following settings
<img src="https://i.imgur.com/0q02e7n.jpg" width="410">
<img src="https://i.imgur.com/XOYAuqo.jpg" width="410">
<img src="https://i.imgur.com/9IOKEZh.jpg" width="410">
<img src="https://i.imgur.com/mldZvEG.jpg" width="410">


  - This should provide seamless switching between the GPUs and have proper power management for the dGPU when it's not in use.
  
## 6. ASUS NB Ctrl, rogauracore and Keyboard
  ### ASUS NB Ctrl - A command-line utlity to provide some of the software functionality found on Windows
  ##### Enable ASUS DBus notifications (for fan curve changes, etc)
  - ```systemctl --user enable asus-notify.service```
  - ```systemctl --user start asus-notify.service```
  ##### Fan Speed
  - You can switch between the 3 profiles ASUS provided with this laptop ( silent | normal | boost )
  - `sudo asusctl profile -p <profile>` 
  ### rogauracore - Restores keyboard backlight control
  ##### Enable Backlight on boot 
  1. `sudo nano /etc/rc.d/rc.local`
  
  ```
  #!/bin/sh
  rogauracore initialize_keyboard
  rogauracore brightness 3
  ```
  1a. `sudo chmod a+x /etc/rc.d/rc.local`
  
  2. `sudo nano /etc/systemd/system/keyboard.service`
  ```
[Unit]
Description=/etc/rc.local compatibility
ConditionFileIsExecutable=/etc/rc.d/rc.local

[Service]
Type=oneshot
ExecStart=/etc/rc.d/rc.local
TimeoutSec=0
StandardOutput=tty
RemainAfterExit=yes
SysVStartPriority=99

[Install]
WantedBy=multi-user.target
```

3. `sudo systemctl enable --now /etc/systemd/system/keyboard.service`
- It might output an error but the keyboard should light up
  ##### Change Backlight brightness
  - The Keyboard backlight brightness ranges from `0-3`
  - `sudo rogauracore brightness <numbergoeshere>`
## 7. Extras

##### 1. NVIDIA GPU Power Managment 
- `sudo nano /etc/modprobe.d/nvidia.conf`
```
options nvidia_drm modeset=1
options nvidia "NVreg_DynamicPowerManagement=0x02"
```

##### 2. AMD CPU TDP Control via [ryzenadj](https://github.com/FlyGoat/RyzenAdj)
 - The Ryzen 7 3750H runs hot, even with good quality thermal paste. To keep temperatures down, we can put a TDP limit on the CPU that reduces the clock speed under a reasonable balance between thermals and performance.

 - `yay -S ryzenadj-git`

 - Once it's installed, we can now begin setting limits for the `STAMP`, `fast`, `slow` limit to `25W`, and the `thermal` limit to `90C` to keep it from thermal throttling
 
 - `sudo ryzenadj --stapm-limit=25000 --fast-limit=25000 --slow-limit=25000 --tctl-temp=90`
 
 - With this configuration, the CPU boosts to 4GHz on single-core workloads and maxes out around 3.4GHz with a sustained all-core load.

##### 3. Wayland
  - If you prefer to use the laptop without the dGPU, I would recommend using a Wayland DE/WM as its a much smoother overall experience than X. I have noticed better battery life, UI responsiveness, display management, and touchpad experience in KDE Wayland than any other DE running in Xorg. Most of this is thanks to having the Wayland compositor as the display server. Libinput greatly takes advantage of this with the ELAN1200 touchpad, providing perfect multi-finger gestures with incredible tracking accuracy.
