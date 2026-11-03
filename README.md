# pi-card

Boot Raspberry Pi OS from a physical SD card in a macOS Terminal. Changes made inside the guest stay on the card. QEMU emulates a Pi 3B and uses your Mac’s internet connection. Only boot files are temporarily staged on the Mac.

## Install

Requires macOS, Homebrew, and an SD reader that macOS identifies as removable. In Terminal:

```sh
brew install python qemu dtc
git clone https://github.com/suitablebat9/pi-card.git
install -m 755 pi-card/pi-card "$(brew --prefix)/bin/pi-card"
```

## Boot a card

Insert the card and wait for its FAT boot volume to appear in Finder. Run:

```sh
pi-card
```

Check the displayed disk, capacity, and volume before confirming. Enter your Mac password when prompted. Use the Pi’s credentials at its login prompt. The launcher uses administrator privileges to open the physical disk, so use cards and guest software you trust. Keep the reader connected throughout the session.

* `pi-card --list` lists eligible mounted cards without unmounting anything.
* `pi-card disk4` selects one of several cards. Check the current identifier each time.
* `pi-card --offline` boots without networking.
* `pi-card --bits 32` or `pi-card --bits 64` selects a kernel when both are present.

Inside the Pi, `findmnt -no SOURCE,FSTYPE,OPTIONS /` should report `/dev/sda2`, `ext4`, and `rw`. Ctrl-C interrupts the foreground command inside the guest. To finish, run `sudo poweroff` in the Pi. Wait for power-off and the Mac prompt before ejecting the verified disk with `diskutil eject /dev/diskN`. Reinsert to start another session. A guest reboot ends the QEMU session.

## Wi-Fi and internet

Connect the Mac to Wi-Fi, then launch normally. The guest gets a virtual USB Ethernet adapter with DHCP and outbound internet through the Mac. The Mac keeps the Wi-Fi credentials. In the guest, check `ip route`, then try `curl -I https://www.raspberrypi.com`. The guest cannot scan nearby Wi-Fi networks through this adapter. Guest networking requires USB Ethernet drivers and DHCP in the installed OS. QEMU may print USB descriptor warnings during enumeration. Check the guest route and HTTPS response to see whether networking actually works.

To save Wi-Fi for a physical Pi running Raspberry Pi OS with NetworkManager, run `sudo nmtui` inside the guest. Choose **Edit a connection → Add → Wi-Fi**, enter a profile name and the SSID, leave **Device** empty, select the correct security mode, enter the Wi-Fi password, enable **Automatically connect**, and save. For a secured home network, use the WPA Personal option matching your router. This stores the profile on the SD card. Check the Wi-Fi country setting in `sudo raspi-config` for your location. Once the physical Pi boots, check `nmcli device status`. Older OS versions that use wpa_supplicant need their usual on-device configuration.

[QEMU networking](https://www.qemu.org/docs/master/system/devices/net.html) · [Raspberry Pi network configuration](https://www.raspberrypi.com/documentation/computers/configuration.html)

## SSH into the emulated Pi

Run `pi-card`. After you confirm the card, the launcher creates the Raspberry Pi OS SSH boot marker and forwards port 22 on the Mac’s IPv4 interfaces to the Pi. Sign in with your Pi username and its configured password or key. Standard Raspberry Pi OS includes the SSH server; a custom image needs OpenSSH Server installed.

From the same Mac, open a second Terminal and run `ssh USER@127.0.0.1`. From another device on the same network, use `ssh USER@MAC_IP` with the Mac’s Wi-Fi IPv4 address, or `ssh USER@MAC_NAME.local` on a network with Bonjour. Find the Mac’s hostname with `scutil --get LocalHostName` and append `.local`. Keep the emulator running. The card also retains SSH access when you move it to a physical Pi.

If another service such as macOS Remote Login already owns port 22, the launcher reports the conflict before touching the card. The existing `pi-card --ssh-lan` and `pi-card --ssh` options still use port 2222 for network and local-only access respectively. Use `pi-card --offline` for a session without networking. If a LAN connection times out, check that both devices can reach each other and that the Mac firewall permits QEMU’s incoming connections.

## Compatible cards

Designed for the standard two-partition Raspberry Pi OS layout: FAT boot partition followed by a Linux root partition. Requires a Pi 3B device tree, cmdline.txt, and kernel8.img (64-bit) or kernel7.img (32-bit). The launcher refreshes these from the selected card, recognizes compressed kernels, and loads the matching initramfs8 or initramfs7 when present. A Pi 4 or Pi 5 card can work if the installed OS also supports a Pi 3B. The 64-bit console and watchdog settings were developed against Raspberry Pi OS Lite based on Debian 13 on an Apple silicon Mac. The 32-bit path and other releases may need additional testing.

The launcher uses direct kernel boot. It does not reproduce Pi firmware config.txt settings, device-tree overlays, hardware Wi-Fi, GPIO accessories, cameras, custom kernel names, or alternative root layouts. Encrypted, multiboot, appliance, and Pi-5-only images fall outside the supported layout. Card detection intentionally excludes fixed disks. Automatic first-boot partition expansion is skipped for emulation. If you need the card’s full capacity, expand the filesystem on the physical Pi.

If a card is missing from the list, shut down any existing VM, eject and reinsert the card, and check that its boot volume mounts in Finder. If boot fails, preserve the last lines on screen before retrying. A console log may contain personal information; review it before posting to a public issue.
