# ThinkPad X1 Extreme Gen 3 w/ Sonoma Hackintosh

A working log for this machine, not a generic EFI dump. Follow the install sections as they were actually done. Daily use is in decent shape. Optional quality-of-life items are at the end!

> Note that I used the debug version of OpenCore throughout, for easier troubleshooting. It is recommended that you do the same and then swap to a non-debug version after the machine is stable!

Hardware on this unit:

- CPU: Intel Core i7-10850H (Comet Lake-H)
- iGPU: Intel UHD 630 (this is what macOS uses for the built-in screen)
- dGPU: GTX 1650 Ti Max-Q — faulty on this laptop, and unsupported in macOS anyway. HDMI is wired to this chip, so HDMI will not work in macOS
- Display: 15.6" FHD, driven by the iGPU in BIOS Hybrid / iGPU mode (Optimus)
- Wi-Fi / BT radio: Intel AX201 — soldered (cannot be replaced). **Wi-Fi is faulty; Bluetooth works.** Leave `AirportItlwm.kext` off. `IntelBluetoothFirmware.kext` + `BlueToolFixup.kext` are on after install
- Storage: two 256 GB NVMe drives. One is Windows (Microsoft Basic Data). One is macOS (APFS, “Macintosh SSD”). Do not erase the Windows disk
- Extra Wi-Fi: TP-Link Archer T3U Nano (USB-A, Realtek). Used because AX201 is dead. USB-C also works after the USB map

EFI starting point: [nvcuong1312 X1E Gen 3 Sonoma OpenCore](https://github.com/nvcuong1312/Thinkpad-X1-Extreme-Gen-3-Hackintosh-OpenCore-Sonoma)

Full theory: [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/) (laptop Coffee Lake Plus / Comet Lake) and [OpenCore Post-Install](https://dortania.github.io/OpenCore-Post-Install/)

---

## What we already know about this chassis

The 1650 Ti cannot be used in macOS (NVIDIA dropped). This laptop’s 1650 Ti is also hardware-faulty. BIOS should stay on Hybrid / iGPU so the lid panel is driven by UHD 630. Leave the dGPU enabled in BIOS so Windows can still try to use it; the OpenCore EFI includes `SSDT-dGPU-Off` and `disable-external-gpu` so macOS does not attach to NVIDIA. HDMI is tied to the 1650 Ti, so it will stay dead in macOS

AX201 is soldered. **Wi-Fi stays dead** — `AirportItlwm.kext` hung recovery (kernel stuck after `nfs_kext`) and must stay off. **Bluetooth works** with `IntelBluetoothFirmware.kext` + `BlueToolFixup.kext`. Daily Wi-Fi is the Archer T3U Nano. USB-C data works after stub B (USB 2 on the PCH XHCI, USB 3 on the Thunderbolt XHCI)

SMBIOS: the GitHub EFI ships MacBookPro16,4. GenSMBIOS was run for this machine as **MacBookPro16,1**. Serials must stay unique. Do not clone someone else’s serials

---

## Mounting the macOS EFI (do this before every plist/kext edit)

Identifiers **swap**. Always:

```bash
diskutil list
```

Pick the ~**210 MB** partition named **EFI** on the APFS / Macintosh SSD disk. That has been `disk0s1` or `disk1s1`. Do **not** mount Windows **NO NAME** (~105 MB)

```bash
sudo mkdir -p /Volumes/ESP-MAC
sudo diskutil mount -mountPoint /Volumes/ESP-MAC diskXs1
ls /Volumes/ESP-MAC/EFI/OC/config.plist
```

Replace `diskXs1` with the id you just found. When finished:

```bash
diskutil unmount /Volumes/ESP-MAC
```

If unmount fails: `sudo diskutil unmount force /Volumes/ESP-MAC`

Keep the installer USB as a lifeboat (WEG off / `SecureBootModel` Disabled on the stick). F12 firmware boot of that stick does **not** use ScanPolicy on the SSD

---

## 1. BIOS (F1)

The following settings were set (those not visible are not available in the BIOS):

| Setting | Value |
|------|--------|
| Secure Boot | Disabled |
| TPM / Security Chip | Disabled |
| VT-d | Disabled |
| SATA mode | AHCI |
| Fast Boot | Disabled |
| Wake on LAN | Disabled |
| Graphics | Hybrid / iGPU (not Discrete). Discrete would put the lid on the dead 1650 Ti. |

This X1E BIOS has no USB Legacy Support or XHCI Hand-off switches. Skip those Dortania rows

You can turn Wireless LAN / Bluetooth off in BIOS during install to cut AX201 noise. They are not required for the T3U Nano. Bluetooth is back on in BIOS for daily use

---

## 2. Build the installer USB (Windows)

1. Install Python 3.
2. Download [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases).
3. In `OpenCorePkg\Utilities\macrecovery\` run:

```bat
py macrecovery.py -b Mac-226CB3C6A851A671 -m 00000000000000000 download
```

4. Format the 4 GB USB as FAT32 (MBR is fine; this stick was FDisk + one FAT32 volume named EFI)
5. Create `com.apple.recovery.boot\` on the USB root and copy both the `.dmg` and `.chunklist`
6. Put the X1E repo `EFI` folder on the USB root next to `com.apple.recovery.boot`
7. Run [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) for MacBookPro16,1 (this install). The repo sample is 16,4. Write unique MLB / serial / ROM into `EFI\OC\config.plist` → PlatformInfo → Generic

Layout:

```text
USB:\
  EFI\
    BOOT\
    OC\
  com.apple.recovery.boot\
    BaseSystem.dmg
    BaseSystem.chunklist
```

On this machine the USB volume is named EFI, so in macOS the files show up as `/Volumes/EFI/EFI/OC/`, not `/Volumes/ESP-USB/EFI/`. If a mount point looks empty, list `/Volumes/EFI` as well

---

## 3. OpenCore config changes that were required to install

The stock repo `config.plist` is a post-install EFI. It will not boot a USB recovery image until you change a few keys. Edit `EFI\OC\config.plist` on the USB (ProperTree on Windows is fine)

### ScanPolicy = 0

Repo value was 17760515 (0x010F0103). That allows APFS on NVMe only and blocks USB and FAT. Recovery lives on FAT32 USB, so the picker showed tools (OpenShell) and no macOS. Set Misc → Security → ScanPolicy to 0 for install. Daily EFI can stay at 0 so USB volumes still appear in the OpenCore picker

### Disable AirportItlwm.kext

The AX201 is faulty. With AirportItlwm enabled, verbose boot froze after `nfs_kext_start: successfully loaded NFS kext`. That line is a red herring; AirportItlwm attaches right after BSD/networking. Set Kernel → Add → AirportItlwm.kext → Enabled to false. **Leave it false forever on this unit!**

### Disable IntelBluetoothFirmware.kext and BlueToolFixup.kext (install only)

bluetoothd still launches in recovery even with BIOS wireless off. The firmware kexts made KeyStore/ramdisk noise worse. They were left off until after first boot, then **turned back on**. AX201 Bluetooth works. bluetoothd may still log; that is not a kernel panic

### Display: you will see verbose before you see a GUI

WhateverGreen + ig-platform-id `06009B3E` plus `enable-hdmi20` made WindowServer paint somewhere that is not the laptop panel. HDMI is on the 1650 Ti. Recovery userspace was already running (`Language Chooser`, `Installer Progress`) while the lid stayed on OpenCore GOP text or a frozen Apple logo

What was tried:

- `-igfxvesa`, `igfxonln=1`, `-wegnoegpu`, laptop id `00009B3E`, removed `enable-hdmi20`: GUI processes ran, still no picture with `-v` off (frozen logo)
- Forced 1920x1080 + ForceResolution + no VESA: worse. Reverted.
- WhateverGreen was left **off** for install so GOP/`-v` stayed on the lid. After first boot, QE/CI was already present from DeviceProperties (`00009B3E`). WEG was turned **back on** later (see §7). Do not boot the installer with WEG + `06009B3E` + `enable-hdmi20`

Keep `-v` in boot-args until you have a visible installer GUI. Without `-v`, a frozen Apple logo usually means the same Language Chooser hang, not a stuck download

Useful boot-args used while debugging:

```text
-v debug=0x100 keepsyms=1 alcid=11 -igfxvesa igfxonln=1 -wegnoegpu
```

`SMCLightSensor.kext` was disabled (`alsd: No iterator`). It is only auto-brightness, not sleep

SecureBootModel was Default and blocked booting the installed volume. Set Misc → Security → SecureBootModel to **Disabled**. Leave it Disabled; apps do not care

---

## 4. Install macOS

1. F12 → USB (UEFI).
2. OpenCore → macOS Base System / Install Sonoma. If the list is empty, ScanPolicy is still wrong
3. Need network: USB-A Ethernet (or later T3U). USB-C docks fail until the USB map (stub B)
4. Disk Utility → Show All Devices → erase only the empty NVMe (not Windows): APFS, GUID. This install named it **Macintosh SSD**, not Macintosh HD
5. Install onto that volume. Sonoma downloads to the SSD, not the 4 GB USB
6. Several reboots are normal. Keep choosing “macOS Installer” until it finishes. If the same short `/mnt5/` copy repeats forever, you are stuck in leftover Install Data; do not keep selecting Installer after the system volume exists

> After Disk Utility showed Macintosh SSD (~10 GB sealed system) and Macintosh SSD - Data (almost empty), `boot.efi` was present (`diskutil` via Recovery). The picker still only showed Installer because SecureBootModel was Default and NVRAM still pointed at the installer

7. Set SecureBootModel to Disabled on the USB EFI, reboot, Reset NVRAM in OpenCore (Space if needed), then pick **Macintosh SSD**, not Installer, not Windows, not the USB DMG.
8. Complete Setup Assistant

---

## 5. Copy OpenCore to the macOS SSD EFI

Until this is done, pulling the USB means macOS will not boot. Confirm ids with `diskutil list` first (see “Mounting the macOS EFI”)

Example (this machine after install used macOS EFI `disk0s1`; yours may be `disk1s1`):

```bash
sudo mkdir -p /Volumes/ESP-MAC
sudo diskutil mount -mountPoint /Volumes/ESP-MAC disk0s1

ls /Volumes/EFI/EFI/OC/config.plist
ls /Volumes/ESP-MAC/EFI

sudo ditto /Volumes/ESP-MAC/EFI /Volumes/ESP-MAC/EFI-backup
sudo rm -rf /Volumes/ESP-MAC/EFI
sudo ditto /Volumes/EFI/EFI /Volumes/ESP-MAC/EFI
ls /Volumes/ESP-MAC/EFI/OC/config.plist

diskutil unmount /Volumes/ESP-MAC
```

If `/Volumes/ESP-USB` is empty, the stick is `/Volumes/EFI`. Reboot, F12, choose the macOS SSD EFI, not Windows Boot Manager

**Verify:** unplug the installer USB, reboot, F12 → macOS SSD EFI → Macintosh SSD. You should reach the desktop

---

## 6. Archer T3U Nano Wi-Fi

Chipset is Realtek (RTL8812BU class). Plug into USB-A (or USB-C after stub B)

Path used: [chris1111 Wireless-USB-OC-Big-Sur-Adapter](https://github.com/chris1111/Wireless-USB-OC-Big-Sur-Adapter) V17, SIP-on method from [discussion 167](https://github.com/chris1111/Wireless-USB-OC-Big-Sur-Adapter/discussions/167)

Do not leave SIP fully off (`ff0f0000`). `csrutil disable` from a normal Terminal does nothing useful; OpenCore overwrites `csr-active-config` every boot

Order that was followed:

1. Set NVRAM `csr-active-config` to `03080000` (bytes `03 08 00 00`) in config.plist on the macOS EFI. Reboot, reset NVRAM
2. Download the notarized zip (Consent.command) and V17. Ignore `__MACOSX`. Open:
   `~/Downloads/Wireless USB OC Big Sur Adapter-V17/Wireless USB OC Big Sur Adapter.app`
3. Recovery (OpenCore picker, Space, Recovery) → Terminal:

```bash
/usr/sbin/spctl kext-consent add ZYM2ETK3E7
```

4. Reboot to macOS, run the app, Install into OpenCore EFI
5. Set `csr-active-config` back to `00000000`, reset NVRAM

**Verify:**

```bash
csrutil status
```

Should say `enabled`. Use the Realtek **menu-bar** Wi-Fi (`/Library/Application Support/WLAN/StatusBarApp.app`), not Control Centre, if Apple’s Wi-Fi UI is empty

If macOS says the status app is damaged:

```bash
APP="/Library/Application Support/WLAN/StatusBarApp.app"
sudo xattr -cr "$APP"
open "$APP"
```

Byte order for SIP: full on is `00000000`. Disabled-style dump is `ff0f0000` as `ff 0f 00 00`, not `00 00 0f ff`

---

## 6b. Bluetooth

AX201 Wi-Fi stays off. After first boot, Kernel → Add was set back to true for `IntelBluetoothFirmware.kext` and `BlueToolFixup.kext`. BIOS wireless/BT on

**Verify:** System Settings → Bluetooth shows the adapter; pair a device. `kextstat | grep -i IntelBluetooth` should list the firmware kext

---

## 7. Post-install

Checked against [Dortania OpenCore Post-Install](https://dortania.github.io/OpenCore-Post-Install/). FileVault, DRM, and CFG-Lock unlock are **not** pursued. `SecureBootModel` stays **Disabled**. ScanPolicy can stay **0**. Keep `-v` on purpose

EFI already had: `SSDT-PLUG-DRTNIA` + `CPUFriend` / `CPUFriendDataProvider`, `NVMeFix`, `SSDT-PNLF` + `BrightnessKeys`, `SSDT-EC-USBX-LAPTOP`, `SSDT-AWAC`, `SSDT-dGPU-Off`, `HibernateMode` = None

### Feature checklist (verify on the running Mac)

**iGPU / WhateverGreen**

Install used WEG-off. After first boot, UHD 630 already had QE/CI from `AAPL,ig-platform-id` `00009B3E`. WEG was then enabled for Displays and brightness keys.

SSD EFI now: `WhateverGreen.kext` Enabled; `SSDT-dGPU-Off`; `disable-external-gpu` / `-wegnoegpu`; id `00009B3E`; no `-igfxvesa`; `igfxonln=1`. Daily boot-args:

```text
-v debug=0x100 keepsyms=1 alcid=11 igfxonln=1 -wegnoegpu
```

If WEG blacks the screen: F12 USB OpenCore (WEG off on the stick) → Macintosh SSD → disable WEG on the **internal** EFI or restore `~/Desktop/config.plist.bak-pre-weg`.

**Verify:**

```bash
system_profiler SPDisplaysDataType
```

Look for Intel UHD Graphics 630, Metal, VRAM well above 7 MB (this unit ~1536 MB). Brightness keys should change the panel. System Settings → Displays section in settings should not be blank

**USB map**

Stock repo `UTBMap.kext` was PCH Type-A + internals only. USB-C is split (USB 2 on PCH, USB 3 on TB XHCI). Mapped on **native Windows** (F12 → Windows Boot Manager) with [USBToolBox](https://github.com/USBToolBox/tool) `Windows.exe`. Companion binding on. Output filename **`UTBMap.kext`**. Old map: `~/Downloads/UTBMap.kext.old`. Copied over `EFI/OC/Kexts/UTBMap.kext`. Left `USBToolBox.kext`. `XhciPortLimit` stays false. Do not add `UTBDefault.kext`

Ports: `1,4,5,6,8,14,17,22,29,30`

| Items | Controller | What | Type |
|-------|------------|------|------|
| 1 + 17 | PCH | USB-A | 3 |
| 6 + 22 | PCH | USB-A | 3 |
| 4 + 29 | PCH HS + TB SS | USB-C | 9 |
| 5 + 30 | PCH HS + TB SS | USB-C | 9 |
| 8 | PCH | Camera | 255 |
| 14 | PCH | AX201 Bluetooth | 255 |

**Verify:** plug a stick into both USB-A and both USB-C (both orientations). Camera in Photo Booth. T3U still connects

**Sleep**

Lid close, Apple menu Sleep, and wake work. Re-test after any USB map change

**Verify:** close the lid 30+ seconds, open; Apple menu → Sleep, wake with keyboard. No instant wake loop

**Audio / battery / input**

**Verify:** speakers and headphone jack (`alcid=11`). Menu bar battery percent. Keyboard, TrackPoint, trackpad. Camera (above). Fingerprint and SD reader are expected dead

---

### Apple ID / iMessage / FaceTime

[Dortania iServices](https://dortania.github.io/OpenCore-Post-Install/universal/iservices.html) was used a refernece

This chassis has **no PCI Ethernet**. The T3U is USB, so it is **not** a built-in `en0`. iMessage wants a built-in `en0`. Internet can (and should) stay on the T3U as `en1`

#### Hackintool quirks (read this first)

[Hackintool](https://github.com/headkaze/Hackintool/releases) → **System** → **Peripherals** (network icon), **not** the PCI tab

- In this case, it **did not list** USB Wi-Fi (T3U) or ACPI NullEthernet at all. Empty Hackintool ≠ missing `en0`.
- Trust Terminal (`networksetup -listallhardwareports`, `ifconfig en0`) over Hackintool for this machine
- PCI `built-in` DeviceProperties cannot fix a USB NIC. Do not chase a `PciRoot` inject for the T3U

#### 1. NVRAM (this machine: pass)

```bash
sudo nvram IServicesTest=ok
```

Reboot, then:

```bash
nvram IServicesTest
sudo nvram -d IServicesTest
```

If `ok` is missing after reboot, fix [emulated NVRAM](https://dortania.github.io/OpenCore-Post-Install/misc/nvram.html) before Apple ID. Here it survived reboot

#### 2. Serial (this machine: pass)

About This Mac serial → [Apple Check Coverage](https://checkcoverage.apple.com/). **Invalid / unable to check coverage is the good result** (unused fake Mac). Do **not** generate a new serial. A real purchase date would mean GenSMBIOS for MacBookPro16,1 and a new MLB/serial/UUID

#### 3. NullEthernet for built-in en0

After deleting `NetworkInterfaces.plist` / `preferences.plist`, there was still no `en0`. T3U showed as `en1` (`50:3d:d1:da:56:3a`)

Used [stevezhengshiqi NullEthernet 1.0.9 RELEASE](https://github.com/stevezhengshiqi/OS-X-Null-Ethernet/releases) zip (contains `NullEthernet.kext` **and** `SSDT-RMNE.aml`). Do not use `NullEthernetInjector.kext` (PCI method). Do not fetch RehabMan’s raw `SSDT-RMNE.aml` (404); use the SSDT **inside the same zip** so HID `NULE0000` matches the kext

Mount macOS EFI (`disk0s1` or `disk1s1`). Backup `config.plist`. Copy kext → `EFI/OC/Kexts/NullEthernet.kext`, SSDT → `EFI/OC/ACPI/SSDT-RMNE.aml`. Add ACPI + Kernel entries; set ROM later to **match whatever ends up as en0**

Example (EFI already mounted at `/Volumes/ESP-MAC`, zip already unzipped):

```bash
sudo ditto ~/Downloads/NullEthernet-install/unzipped/NullEthernet.kext \
  /Volumes/ESP-MAC/EFI/OC/Kexts/NullEthernet.kext
sudo cp ~/Downloads/NullEthernet-install/unzipped/SSDT-RMNE.aml \
  /Volumes/ESP-MAC/EFI/OC/ACPI/SSDT-RMNE.aml
```

Then Python on the Mac (`read_bytes` with no spaces) to append `SSDT-RMNE.aml` and `NullEthernet.kext` if missing. Backup was `~/Desktop/config.plist.bak-pre-nullethernet`

Reboot, Reset NVRAM

**First boot after NullEthernet:** Ethernet (fake NIC, MAC `00:16:cb:00:11:22` from the SSDT) was **en1**; T3U was **en0**. That is backwards for iMessage

**Fix:** System Settings → Network → remove **all** services → Apply, then:

```bash
sudo rm /Library/Preferences/SystemConfiguration/NetworkInterfaces.plist
sudo rm /Library/Preferences/SystemConfiguration/preferences.plist
```

Reboot. Add **Ethernet first**, then Wi-Fi (802.11ac NIC)

**Verify (required before Apple ID):**

```bash
networksetup -listallhardwareports
ifconfig en0 | grep ether
```

Must look like:

```text
Hardware Port: Ethernet
Device: en0
Ethernet Address: 00:16:cb:00:11:22

Hardware Port: 802.11ac NIC
Device: en1
Ethernet Address: 50:3d:d1:da:56:3a
```

`IOMACAddress` `<0016cb001122>` is NullEthernet; `<503dd1da563a>` is the T3U. Both can appear in `ioreg`

#### 4. ROM must match en0, not the T3U

After the swap, ROM was still the T3U (`503dd1da563a`). Change it to NullEthernet’s MAC:

```text
0016cb001122
```

Mount EFI, then:

```bash
python3 -c '
import plistlib
from pathlib import Path
p = Path("/Volumes/ESP-MAC/EFI/OC/config.plist")
d = plistlib.loads(p.read_bytes())
print("ROM", d["PlatformInfo"]["Generic"]["ROM"].hex())
'
```

Set Generic → ROM Data to `0016cb001122` if it is not. Reset NVRAM, reboot.

**Verify:**

```bash
# after mounting EFI
python3 -c '
import plistlib
from pathlib import Path
rom = plistlib.loads(Path("/Volumes/ESP-MAC/EFI/OC/config.plist").read_bytes())["PlatformInfo"]["Generic"]["ROM"]
print(rom.hex())
print(":".join("%02x" % b for b in rom))
'
ifconfig en0 | grep ether
```

Both must be `00:16:cb:00:11:22` / `0016cb001122`. Do not change MLB, SystemSerialNumber, SystemUUID, or MacBookPro16,1

#### 5. Do not send Apple traffic out the fake NIC

Service order was already Wi-Fi first, but **en0 still got `169.254.x.x`**. Apple ID then spun forever on the 2FA globe

```bash
networksetup -listnetworkserviceorder
ifconfig en0 | grep "inet "
ifconfig en1 | grep "inet "
```

Wi-Fi must be (1). en1 should have a real LAN IP. Then:

```bash
networksetup -setv4off Ethernet
networksetup -setv6off Ethernet
ifconfig en0 | grep "inet "
```

en0 must show **no** `inet`. Ethernet stays in the service list (so `en0` still exists). Internet stays on **802.11ac NIC**

**Verify:** `ping -c 3 appleid.apple.com` works. `date` is correct (Set automatically)

#### 6. Sign-in order that actually worked

Signing into **System Settings → Apple ID** first spun forever after the verification code

**What worked:** sign into the **App Store** first (same Apple ID, 2FA). After that, Apple ID / iCloud in System Settings succeeded. Then Messages and FaceTime!

If Apple asks you to **call support**, stop. That is an account/serial lock, not a kext miss

**Verify:** App Store shows your account. System Settings → Apple ID is signed in. Messages and FaceTime activate. `csrutil status` still enabled

---

### Dortania items skipped on this machine

| Dortania page | This machine |
|---|---|
| [Security / Apple Secure Boot](https://dortania.github.io/OpenCore-Post-Install/universal/security.html) | `SecureBootModel` **Disabled**. Apps do not care |
| [CFG Lock](https://dortania.github.io/OpenCore-Post-Install/misc/msr-lock.html) | **Skip.** Repo already has `AppleCpuPmCfgLock` and `AppleXcpmCfgLock` **true**. Sleep works. Lenovo unlock is a BIOS hack and not worth the hassle |
| [DRM](https://dortania.github.io/OpenCore-Post-Install/universal/drm.html) | **Not pursuing.** UHD 630-only |
| FileVault / Vault | **Not pursuing.** Vault stays `Optional`, since I will not be using it |
| [OpenCore GUI / drop `-v`](https://dortania.github.io/OpenCore-Post-Install/cosmetic/verbose.html) | Optional. This install keeps `-v` because I like it |
| [LauncherOption](https://dortania.github.io/OpenCore-Multiboot/) | Optional. F12 → Windows Boot Manager already works |
| [Updating OpenCore / macOS](https://dortania.github.io/OpenCore-Post-Install/universal/update.html) | Maintenance. Repo has `BlacklistAppleUpdate` true |

BIOS: first boot = macOS SSD EFI for OpenCore by default; F12 → Windows Boot Manager for native Windows

---

## What will not work

- GTX 1650 Ti and HDMI (this unit’s 1650 Ti is also hardware-faulty)
- USB-C / Thunderbolt **as a display out** (USB-C to DP/HDMI cable, or a dock’s HDMI/DP that is DisplayPort alt-mode). The NHI kext loads (`8086:15eb`), but System Profiler still says Thunderbolt/USB4 has no drivers (no switch / no path), and the iGPU’s two DP pipes stay idle. Native USB-C picture is unproven on this chassis; do not flash TB firmware on the soldered chip. A **DisplayLink** dock or adapter (USB graphics + DisplayLink’s macOS app) will work though!
- Intel AX201 **Wi-Fi** (soldered; AirportItlwm hung recovery). **Bluetooth works**
- Fingerprint reader, often the SD reader
- Hardware DRM in Safari (typical iGPU-only limit)
- AirDrop / Continuity / Sidecar that need Apple Wi-Fi (T3U is Realtek USB)
- Auto-brightness (`SMCLightSensor` off; no ALS)

---

## Links

- EFI: https://github.com/nvcuong1312/Thinkpad-X1-Extreme-Gen-3-Hackintosh-OpenCore-Sonoma
- Dortania post-install: https://dortania.github.io/OpenCore-Post-Install/
- Dortania USB map: https://dortania.github.io/OpenCore-Post-Install/usb/
- USBToolBox (Windows): https://github.com/USBToolBox/tool
- NullEthernet 1.0.9: https://github.com/stevezhengshiqi/OS-X-Null-Ethernet/releases
- Dortania iServices: https://dortania.github.io/OpenCore-Post-Install/universal/iservices.html
- Dortania sleep: https://dortania.github.io/OpenCore-Post-Install/universal/sleep.html
- Dortania Comet Lake laptop: https://dortania.github.io/OpenCore-Install-Guide/config-laptop.plist/coffee-lake-plus
- Installer from Windows: https://dortania.github.io/OpenCore-Install-Guide/installer-guide/windows-install.html
- Dual disk: https://dortania.github.io/OpenCore-Multiboot/empty/diffdisk.html
- T3U / Realtek USB: https://github.com/chris1111/Wireless-USB-OC-Big-Sur-Adapter
- SIP + kext consent: https://github.com/chris1111/Wireless-USB-OC-Big-Sur-Adapter/discussions/167
- Hackintool: https://github.com/headkaze/Hackintool/releases
