# Win11 ARM 23H2 "ACPI fix" EFI build in general Ubuntu x86 server #

![result.jpg](./result.jpg)

## How to use ##

*Success in THTF TK630 only. PT620K fails for all combinations even they are in same board (BC32MBHB). BIOS / UEFI are differnt.*

- **Prepare a empty FAT32 USB**. Recommended a 2GB USB DOM. I used Rufus "not bootable GPT USB" to format it.
- Place `tablesfix-tk630-nochain.efi` as `/efi/boot/bootaa64.efi`
- Set boot order "This USB bootloader > Win11 ARM installation USB or OS Disk"

### Hardware advice in UEFI layer ###

- *You can use RDIMM (useful on 2024), 4RX4 works. Up to 2933.* [dixyes](https://dixyes.dev/posts/2023/03/27/run-windows-on-d920s10/)
  - M2 / GPUs are all in PCIE 3.0.
  - **Kunpeng920 is a model series, and has server and desktop variants.** There are still active development for production servers and BIOS, but the microcode issue in "arm v8.2" is consistint, requires BIOS fix. 
  - The microcode may also *cascasde into issues dedicated to USB install media*. [6DammK9/hisi_raid-sp686c-dkms](https://github.com/6DammK9/hisi_raid-sp686c-dkms#extra-installing-ubuntu-2404-on-the-kunpeng920-server)
- For basic display, use **MTT S30** or **AMD Radeon RX series** as primary GPU.
  - JM9100 cannot get into winodws (works in x86 or linux).
  - Nvidia / ASPEED etc. are all no-no (cannot boot, UEFI crash)
- TPM slot refers to [BC11TPMA](https://itprice.com/huawei/bc11tpma.html), which is 16-pin.
  - However CSM / Secureboot / TPM etc. are all absent in BIOS.
- **Prepare a good power button.** OS shutdown / reboot etc. will return to last UEFI state (e.g. Winodws logo) and you need to perform power cycle manually.

### Driver support in Win11 ARM 23H2 ###

*This is a whitelist. Driver just don't exist.*

- **Nvidia 616.00 N1X preview** [Nvidia](https://forums.developer.nvidia.com/t/rtx-spark-developer-preview/377106)
  - Install from list > pick from floppy > Nvidia Desktop Device
  - Support from GTX16 (tested V100 and GT1030 and it fails), works on RTX3050 and A400.
  - Place in secondary PCIE slot.
- **Realtek USB GbE** [treexy](https://treexy.com/products/driver-fusion/database/network-adapters/realtek/usb-c-dock-gen2-ethernet/)
  - Usually common USB LAN adapter e.g. Ugreen are Realtek already
  - May have sleeping problem.
  - Yep, onboard LAN has no driver obviously.
  - Yep, Realtek USB Wifi is not included. Use Wifi extenders that has LAN port instead.
  - ~~LTE / 5G module are too much for a desktop.~~
- Qualcomm ~~NFA765 a.k.a. WCN6856~~ **NFA765A or NCM865A**. [Q6A forum.](https://forum.radxa.com/t/wi-fi-bt-support-for-windows-on-q6a/31091/5) [Q8B page.](https://docs.radxa.com/en/dragon/q8b/hardware-use/m2-e-key) 
  - ~~A for ARM I guess.~~
  - Revision "01" / "0A" / "0B" etc. doesn't count.
  - [WOA-Project/Qualcomm-Reference-Drivers](https://github.com/WOA-Project/Qualcomm-Reference-Drivers) has `qcwlanhmt8280.cab`, `qcwlanhsp8380.cab` matching `qcwlanhmt8280.inf`, `qcwlanhsp8380.inf` in [treexy](https://treexy.com/products/driver-fusion/database/network-adapters/qualcomm/fastconnect-6900-wi-fi-6e-dual-band-simultaneous-dbs-wificx-network-adapter/). However even the inf file may not in full list. Keep searching for the matching driver.
  - All common drivers `qcwlan64.inf` and hardwares (the NFA765, even NCM865) are **not compatable**. Only the [QCNCM865AAMD-865A](https://www.aliexpress.com/item/1005009811479844.html) or [NFA765A](https://www.amazon.com/-/zh_TW/QCNFA765-2400Mbps-802-11AX-Qualcomm-NFA765/dp/B0CJ4N5QSL) which are "for Qualcomm CPU only" are compatable. Buying straight from Quectel official store (they have store in Taobao) is a safe choice.
  - Forcing driver install will only see accessible networks but cannot connect them ~~or instant BSOD~~. Same module works fine in x64 (X99 mobo).

#### QCNFA765 or NFA765a as T99H294.70 ####

- This has been mentioned in [the Q8B page.](https://docs.radxa.com/en/dragon/q8b/hardware-use/m2-e-key)

- This appears in a Copilot+ PC. Source: [HKEPC](https://www.hkepc.com/23634/%E6%80%A7%E8%83%BD%E5%8B%81__24_%E5%B0%8F%E6%99%82%E9%9B%BB%E6%B1%A0%E7%BA%8C%E8%88%AA_ASUS_Vivobook_S16_S3607Q_Copilot%2B_PC_%E6%89%8B%E6%8F%90%E9%9B%BB%E8%85%A6)

- Finally, by sheer coincidentce, it appears in [goofish](https://www.goofish.com/item?spm=a21ybx.search.searchFeedList.6.524c323ceYMdAL&id=995000470667&categoryId=126862930).

- *None is found in Taobao. Image search result is close to nothing. They may have been discarded due to driver issues (It is "Qualcomm CPU only" which is WoA only, x86 will malfunction).*

- *I have no idea for any other model names matching the SUBSYS of NFA765A. The only way to know is disassemble known ARM notebooks.*

```ps1
(gwmi Win32_Bus -Filter 'DeviceID like "PCI%"').GetRelated('Win32_PnPEntity').GetDeviceProperties('DEVPKEY_Device_Address').deviceProperties | ft DeviceID,keyName,{$_.data -shr 16},{$_.data -band 0xFFFF}

# See below
# PCI\VEN_17CB&DEV_1103&SUBSYS_E130105B&REV_01\4&1DA6BDC0&0&0010
```

#### Driver / model reference between general and "ARM enabled" WiFi modules ####

*Just showing 1 sample for the SYBSYS issue, good luck on sourcing them.*

The result driver name are usually `WCN685x`, `QCA206x`, `FastConnect 6900` or `FastConnect 7800`.

This is for ARM `NTARM64` in `qcwlanhmt8280.inf` or `qcwlanhsp8380.inf`:

```inf
;NFA725a v2.1
%QcWlan.DeviceDesc.6490%     = QcWlan_NFA725a_2_1.ndi,               PCI\VEN_17CB&DEV_1103&SUBSYS_910A14CD

;NFA765a v2.1
%QcWlan.DeviceDesc.6490%     = QcWlan_NFA765a_2_1.ndi,            	 PCI\VEN_17CB&DEV_1103&SUBSYS_E105105B
%QcWlan.DeviceDesc.6490%     = QcWlan_NFA765a_2_1.ndi,           	 PCI\VEN_17CB&DEV_1103&SUBSYS_E130105B

;NCM825A
%QcWlan.DeviceDesc.NCM825A%     = QcWlan_NCM825A.ndi,      			   PCI\VEN_17CB&DEV_1107&SUBSYS_940A14CD

;NCM865A
%QcWlan.DeviceDesc.NCM865A%     = QcWlan_NCM865A.ndi,      			   PCI\VEN_17CB&DEV_1107&SUBSYS_E0EB105B
```

This is for x86 `AMD64` in `qcwlan64.inf`:

```inf
;NFA725 v2.0
%QcWlan.DeviceDesc.685x%     = QcWlan_NFA725_2_0.ndi,                      PCI\VEN_17CB&DEV_1103&SUBSYS_910814CD 

;NFA765 v2.0
%QcWlan.DeviceDesc.685x%     = QcWlan_NFA765_2_0.ndi,                      PCI\VEN_17CB&DEV_1103&SUBSYS_E0B6105B 

;NFA725 v2.1
%QcWlan.DeviceDesc.685x%     = QcWlan_NFA725_2_1.ndi,                      PCI\VEN_17CB&DEV_1103&SUBSYS_910914CD  

;NFA765 v2.1
%QcWlan.DeviceDesc.685x%     = QcWlan_RA_NFA765_2_1.ndi,                   PCI\VEN_17CB&DEV_1103&SUBSYS_A82D11AD 

;For Quectel
%QcWlan.DeviceDesc.NCM865%   = QcWlan_NCM865.ndi,                      PCI\VEN_17CB&DEV_1107&SUBSYS_80001EAC
```

### Software worth mentioning ###

- Microsoft store is missing. Restore with [DaBigBlob/ms-store-arm64](https://github.com/DaBigBlob/ms-store-arm64).
- FileZilla for development without VSCode. [minnyres/filezilla-windows-arm64](https://github.com/minnyres/filezilla-windows-arm64)
- CPU-Z / HWMonitor crash, HWiNFO is close to empty, GPU-Z / FurMark2 / CrystalDiskInfo works fine.
- Edge / Chrome / Firefox all has arm64, but they can crash at anytime.
- Docker Desktop / WSL etc. are working fine. I'm push this repo with this machine.
  - Barebone Ubuntu 24.04 does not figured out yet, but I can run 26.04 here can cross compile codes.

## Build guide ##

- Intention: My TK630 / PT620 doesn't work based from [#3](https://github.com/dixyes/d920s10/issues/3). Rebuild code is *a bit complicated*.
  - Getting the concept on cross compiling, especially Linux x86 to Winodws ARM (instead of Winodws x86 to Linux ARM), is a bit messy.
  - After a bit of research, it can be solved elegently by [a built toolchain](https://github.com/mstorsjo/llvm-mingw).

- Using the release version [20240709](https://github.com/dixyes/d920s10/releases/tag/20240709) instead of the recent dynamic one.

- ISO: [Windows 11 23H2 ARM64 (French)](https://archive.org/details/windows-11-23-h-2-arm-64)

```sh
wget https://github.com/dixyes/d920s10/archive/refs/tags/20240709.tar.gz
tar -xzf 20240709.tar.gz
# Export tgz from https://gitlab.com/bztsrc/posix-uefi
git pull https://gitlab.com/bztsrc/posix-uefi

# https://voxelmanip.se/2024/12/10/cross-compiling-for-windows-using-llvm-mingw/
wget https://github.com/mstorsjo/llvm-mingw/releases/download/20260922/llvm-mingw-20260922-ucrt-ubuntu-22.04-aarch64.tar.xz
tar -xf llvm-mingw-20260922-ucrt-ubuntu-22.04-aarch64.tar.xz

# Ref: https://www.biosconfessions.com/posts/from-silicon-to-shell/2-toolchain-setup/
# Ref: https://www.reddit.com/r/rust/comments/s45mn7/how_to_crosscompile_from_linux_to_windows/?show=original
# LLVM-MinGW Clang Toolchains. Warning: 2-3GB of space (SDK matters)
sudo apt install gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu clang llvm mingw-w64

# 11.4.0 (22.04), 13.3.0 (24.04)
aarch64-linux-gnu-gcc --version
# 14.0.0 (22.04), 18.1.3 (24.04)
clang -v
# exec format error
# ./llvm-mingw-20260922-ucrt-ubuntu-22.04-aarch64/bin/clang -v
# 23.1.2
./llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin/clang -v

# Build first: https://gitlab.com/bztsrc/posix-uefi
# 920 special: Must be ARM
cd posix-uefi-master/uefi

# USE_GCC=1
ARCH=aarch64 CROSS_COMPILE=aarch64-linux-gnu- make

cd ../../

cd d920s10-20240709
# Codes modified! See this repo: [tablefix.c, buildaa64.sh]
# It should works already.
POSIX_UEFI_PATH=../posix-uefi-master ./buildaa64.sh 

# Probably need MSVC (msvc-wine)
# unable to execute command: Executable "lld-link" doesn't exist!

# Need QEMU with --sysroot
# lld-link: error: ../posix-uefi-master/uefi/string.o: machine type x64 conflicts with arm64

# Ref: https://stackoverflow.com/questions/78720570/cross-compiling-for-aarch64-with-clang-built-from-source
# Just make clean and try again!
```

- Success build log:

```log
+ '[' '!' -d ../posix-uefi-master ']'
+ CFLAGS=(-target arm64-unknown-windows -ffreestanding -fshort-wchar -mno-red-zone -Ofast -Wall -I/usr/include/efi -I/usr/include/efi/protocol "-I${POSIX_UEFI_PATH}/uefi" -Wframe-larger-than=8192)
+ LDFLAGS=(-target arm64-unknown-windows -nostdlib '-Wl,-entry:uefi_init' '-Wl,-subsystem:efi_application' -fuse-ld=lld-link '-Wl,-DEBUG')
+ build_efi tablesfix tablesfix.c
+ target=tablesfix
+ shift
+ sources=tablesfix.c
+ objs=()
+ for x in $sources
+ /home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin/clang -target arm64-unknown-windows -ffreestanding -fshort-wchar -mno-red-zone -Ofast -Wall -I/usr/include/efi -I/usr/include/efi/protocol -I../posix-uefi-master/uefi -Wframe-larger-than=8192 -g -c -o tablesfix.o tablesfix.c
clang: warning: argument '-Ofast' is deprecated; use '-O3' to enable only conforming optimizations, or consult the documentation for the same behavior [-Wdeprecated-ofast]
+ objs+=("${x%.*}.o")
+ /home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin/clang -target arm64-unknown-windows -nostdlib -Wl,-entry:uefi_init -Wl,-subsystem:efi_application -fuse-ld=lld-link -Wl,-DEBUG -v -o tablesfix.efi tablesfix.o ../posix-uefi-master/uefi/crt_aarch64.o ../posix-uefi-master/uefi/stdio.o ../posix-uefi-master/uefi/stdlib.o ../posix-uefi-master/uefi/string.o ../posix-uefi-master/uefi/time.o
clang version 23.1.2 (https://github.com/llvm/llvm-project.git 85ac560262434c9ccfc0c183ec22d4138ed647fb)
Target: arm64-unknown-windows-msvc
Thread model: posix
InstalledDir: /home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin
 "/home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin/lld-link" -out:tablesfix.efi -libpath:lib/arm64 -libpath:atlmfc/lib/arm64 -libpath:/home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/lib/clang/23/lib/windows -nologo -entry:uefi_init -subsystem:efi_application -DEBUG tablesfix.o ../posix-uefi-master/uefi/crt_aarch64.o ../posix-uefi-master/uefi/stdio.o ../posix-uefi-master/uefi/stdlib.o ../posix-uefi-master/uefi/string.o ../posix-uefi-master/uefi/time.o
+ build_efi readnor readnor.c
+ target=readnor
+ shift
+ sources=readnor.c
+ objs=()
+ for x in $sources
+ /home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin/clang -target arm64-unknown-windows -ffreestanding -fshort-wchar -mno-red-zone -Ofast -Wall -I/usr/include/efi -I/usr/include/efi/protocol -I../posix-uefi-master/uefi -Wframe-larger-than=8192 -g -c -o readnor.o readnor.c
clang: warning: argument '-Ofast' is deprecated; use '-O3' to enable only conforming optimizations, or consult the documentation for the same behavior [-Wdeprecated-ofast]
+ objs+=("${x%.*}.o")
+ /home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin/clang -target arm64-unknown-windows -nostdlib -Wl,-entry:uefi_init -Wl,-subsystem:efi_application -fuse-ld=lld-link -Wl,-DEBUG -v -o readnor.efi readnor.o ../posix-uefi-master/uefi/crt_aarch64.o ../posix-uefi-master/uefi/stdio.o ../posix-uefi-master/uefi/stdlib.o ../posix-uefi-master/uefi/string.o ../posix-uefi-master/uefi/time.o
clang version 23.1.2 (https://github.com/llvm/llvm-project.git 85ac560262434c9ccfc0c183ec22d4138ed647fb)
Target: arm64-unknown-windows-msvc
Thread model: posix
InstalledDir: /home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin
 "/home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/bin/lld-link" -out:readnor.efi -libpath:lib/arm64 -libpath:atlmfc/lib/arm64 -libpath:/home/user/opt_261005/llvm-mingw-20260922-ucrt-ubuntu-22.04-x86_64/lib/clang/23/lib/windows -nologo -entry:uefi_init -subsystem:efi_application -DEBUG readnor.o ../posix-uefi-master/uefi/crt_aarch64.o ../posix-uefi-master/uefi/stdio.o ../posix-uefi-master/uefi/stdlib.o ../posix-uefi-master/uefi/string.o ../posix-uefi-master/uefi/time.o
```

- Codes changed:
  - **TK630 behaves like W510!** [ACPI_BIOS_ERROR](https://dixyes.dev/posts/2023/03/27/run-windows-on-d920s10/) is reproduced. [W510 case](https://github.com/dixyes/d920s10/issues/3) has been expended.
  - **TK630 does not have UEFI shell!** [Synchronous Excception](https://github.com/dixyes/d920s10/issues/2) appears after chainload. Therefore chainload has been disabled. Now it exits to BIOS screen then boot overridde.
  - *Boot menu key is F3.* (PT620K is F4)

- Post install EFI:
  - Revert the swap and use the no-chain EFI again. Now it is magically carry to the next UEFI option i.e. OS Boot.
  - BSOD `CLOCK_WATCHDOG_TIMEOUT` appears after first OS boot.
  - Removing USB key will have `ACPI_BIOS_ERROR`.
  - ~~Use the "non W510 no chain"?~~ **Use a tiny (say 2GB, must be FAT) USB with the exact same EFI file path.** [Ref.](https://developer.seco.com/how-to/uefi-bootable-usb/) will magically get into Windows!
  
