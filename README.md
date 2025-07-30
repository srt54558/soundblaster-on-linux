# Guide: Sound Blaster Passthrough via VFIO on Debian

This guide provides a comprehensive walkthrough for passing through a Creative Labs Sound Blaster audio card to a Windows virtual machine on a Debian 12 host. This is necessary because native Linux drivers are often unavailable for these cards. The process involves overcoming IOMMU grouping issues by replacing the kernel and setting up network-based audio routing for a complete solution.

-----

## 1\. Initial System & Hardware Preparation

Before starting, you must enable virtualization and IOMMU support in your system's BIOS/UEFI.

  * **Goal**: Enable hardware virtualization features required for VFIO.
  * **Actions**:
    1.  Reboot your computer and enter the BIOS/UEFI setup.
    2.  Find and enable **SVM Mode** (for AMD CPUs) or **VT-d** (for Intel CPUs).
    3.  Find and enable the **IOMMU** setting.
    4.  Save changes and exit the BIOS/UEFI.

## IF YOUR BIOS DOES NOT HAVE THEM BOTH, DO NOT PROCEED
-----

## 3\. The IOMMU Grouping Problem & Solution

The most significant hurdle is often poor IOMMU grouping by the motherboard's firmware, which can prevent device passthrough.

### 3.2. Failed Initial Attempt

  * **Action**: A common first step is to add the `pcie_acs_override=downstream,multifunction` parameter to the `GRUB_CMDLINE_LINUX_DEFAULT` line in `/etc/default/grub`, followed by `sudo update-grub`.
  * To find your pci ID, run this command:
  * `lspci -nn | grep -i "creative" | sed 's/^.*\[\([0-9a-f:]*\)\].*$/\1/'`
  * (it should output something like this 1102:0010, if not **DO NOT CONTINUE**)
  * **Add** this to the GRUB_CMLINE_LINUX_DEFAULT line in your grub config and **replace the pci id with yours**:
    `iommu=pt tsc=reliable vfio-pci.ids=(pci id) pcie_acs_override=downstream,multifunction`
  * Save the file and run `sudo update-grub` to apply.
  * **Reboot** Your System, and run `lspci -k | grep -i "Creative" -A 2`. If the line "Kernel driver in use:" says  `vfio-pci`, then it was **successfull** so far.
  * If not, you need to troubleshoot why, because the VM doesnt see the card.



### 3 The Grouping issue

First run this command to check if you have this issue:
`for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d); do echo "IOMMU Group ${g##*/}:"; for d in $g/devices/*; do echo -e "\t$(lspci -nns ${d##*/})"; done; done`
If the device with the name "Creative" is in a seperate group (in some cases there is a "PCIe to PCI Bridge" with it, which is okay), you can skip this part.

If not, it means that your kernel does not seperate the devices, which means we cannot just take the Soundblastercard into the VM, we need to take ALL hardware of that group which is impossible.
To fix this, we need a custom Kernel, i personally use XanMod, and it works perfectly for me.
**NOTE**: For NVIDIA drivers to work, you NEED to use the LTS variant.

XanMod installation:
Here is their official site: `https://xanmod.org/`
Add the apt repository and PGP key from there.
Then install the LTS variant of the kernel `sudo apt install linux-xanmod-lts-x64vX` but replace the X with the version your CPU needs (listed on the site).
After installation reboot, you may need to reinstall some drivers (if you need wifi and it doesnt work, just reboot and select the debian kernel in GRUB).

After that is finished, run:
`for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d); do echo "IOMMU Group ${g##*/}:"; for d in $g/devices/*; do echo -e "\t$(lspci -nns ${d##*/})"; done; done`
and make sure the creative card is in a seperate group (in some cases there is a "PCIe to PCI Bridge" with it, which is okay).

If this is true, we are done with the hardest part! Nice!

## 6\. VM Creation and Optimization

I recommend the Tiny11 iso 24H2 to save on system resources:
`https://archive.org/details/tiny-11-24-h-2-x-64-26100.1742`
And its reccommended to install other VM drivers with the virtio iso:
`https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/` (use the latest or stable)

Run these commands to ensure your VM will have internet:
`sudo virsh net-start default`
`sudo virsh net-autostart default`

Create a minimal and efficient Windows 11 VM using `virt-manager`.

  * **Hypervisor**: QEMU/KVM 
  * **Guest OS**: Windows 11 (Debloated recommended) 
  * **Firmware**: UEFI x86\_64 with Secure Boot (`OVMF_CODE.secboot.fd`) 
  * **CPU/RAM**: Start with 2 cores and \~3GB RAM.
  * **Storage**: Create a 20-30gb storage drive, probably too much but better more than less..
  * **Passthrough Device**: Add the Creative Labs PCI Host Device (add device -> pcie -> select the creative device).
  * **Remove Virtual Hardware**: Delete the default virtual sound card (e.g., ich9) and any unneeded USB redirectors.
  * **Drivers**: Attach the `virtio-win.iso` to a virtual CD-ROM drive to install guest drivers during the Windows setup.
  * **Install**: install the Windows OS and after that the virtio drivers
  * **After install**: After install, remove any windows password, and run
  * `reg ADD "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\PasswordLess\Device" /v DevicePasswordLessBuildVersion /t REG_DWORD /d 0 /f`
  * in cmd. After that open "netplwiz", and unckeck "Users must enter a password to use this computer".
  * **Set static ip**: go to system settings and set your ip adress to `192.168.122.2` to be static.
  * **Installing a web browser**: to install a web browser, use CMD and run `winget install firefox` and install your preferred firefox version.
  * **Installing Soundblaster driver**: Now open firefox and install the drivers for your soundblaster card.

  * Done!! After a reboot of the VM, your Soundblaster card should turn on!! :D

-----

## 7\. Host-to-Guest Lossless Audio Routing

Since the host's audio is now separate from the guest's, you need a way to route it. This method uses a high-quality network stream, which is ideal if you lack a physical S/PDIF loopback.

  ### Easiest and best option:
  * If your motherboard has a S/PDIF OUT port, just connect it to the S/PDIF IN port of your soundblaster (if it has one), and enable it to be heard in the soundblaster command app.

    If not, follow this:

  ### 7.1. Host Setup (PipeWire RTP Sink)

  **IMPORTANT**: You need ffmpeg on both Host and VM.
  On your linux host run: `sudo apt install ffmpeg`

  On the windows VM go to `https://github.com/GyanD/codexffmpeg/releases/latest` and download the "XXXXXXXXXXXXfull_build.zip" zip file.
  Extract the file and **place it in C:\ffmpeg**.
  **You need to MAKE SURE the `ffplay.exe` is EXACTLY in `C:\ffmpeg\bin\ffplay.exe` or else it will NOT work**

  ### In linux

  * 1. open your bashrc or zshrc: `nano ~/.zshrc` or `nano ~/.bashrc` based on what you use
  * 2. paste this code at the end of the file:
       `# Start streaming audio (runs only if not already running)
  audio-start() {
    if pgrep -f "ffmpeg.*udp://192.168.122.2:9999" > /dev/null; then
      echo "⚠️  Audio stream is already running."
    else
      echo "✅ Starting audio stream..."
      nohup /usr/bin/ffmpeg -f pulse -i default -channel_layout stereo -acodec pcm_s16le -ar 48000 -ac 2 -f s16le udp://192.168.122.2:9999 >/dev/null 2>&1 &
    fi
  }

  # Stop streaming audio
  audio-stop() {
    if pgrep -f "ffmpeg.*udp://192.168.122.2:9999" > /dev/null; then
      echo "⛔ Stopping audio stream..."
      pkill -f "ffmpeg.*udp://192.168.122.2:9999"
    else
      echo "ℹ️ No audio stream running."
    fi
  }
  `
  * 3. Save the file and exit
    4. Run `source ~/.zshrc` to apply those changes. (may need to close and open terminal for it to really apply)
   
    ### In windows
    1. create a text file, and type the following content inside:
       `
       @echo off
       C:\ffmpeg\bin\ffplay.exe -ch_layout stereo -f s16le -ar 48000 -nodisp -fflags nobuffer -flags low_delay -i udp://0.0.0.0:9999
       `
    2. rename it to `start_audio_reciever.bat`
    3. press Win+R and run `shell:startup` and place the `start_audio_reciever.bat` inside the folder.


## Thats it!! We got a PCIE soundblaster card to run on Linux and use its FULL potential!
