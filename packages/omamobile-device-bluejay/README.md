<!-- SPDX-License-Identifier: MIT -->
# omamobile-device-bluejay

Omarchy Mobile's first userspace package for the Google Pixel 6a (codename bluejay, Tensor / gs101), the package `device/bluejay/device.conf` names. It hides the Android partitions from desktop tools, holds the USB address, keeps suspend on suspend-to-idle, and gives pacman the same `[omamobile]` repository as the Pixel 7 Pro. It does not drive the radio, the charger, or a camera, it enables no service, it loads no driver, and it does not mark a boot slot successful.

The original 0.1.0 proposal predates the first Pixel 6a boot. A development
handset now boots with matched historical modules and startup files; see the
[boot/smoke report](https://github.com/derekross/omarchy-mobile/issues/348#issuecomment-6803936917)
and [device notes](../../device/bluejay/NOTES.md). That installed development
package has additional hardware files and is not this passive candidate.

The 0.1.1 follow-up passes the host package fixtures and builds as an aarch64
package. Its shared policy files are checked against the reference packages;
the development boot-update policy below is additional. This candidate has
not replaced the active development package on a phone: migration must first
preserve its currently owned startup files, services and module policy.


| Path | Installs to | What it does |
| --- | --- | --- |
| `vendor/udev/rules.d/10-omamobile-storage.rules` | `/usr/lib/udev/rules.d` | Hides `sda` and every Android partition except `userdata` from desktop tools. Same rule as the Pixel 7 Pro. The initramfs stops its guard loop before `switch_root`; the kernel's read-only flags are what remain |
| `vendor/systemd/network/20-usb-gadget.network` | `/usr/lib/systemd/network` | Sets usb0 to 172.16.42.1/24 after handover, as the last gateway. A replug may need a reboot: this package does not close the Type-C chip's USB2 switch again (raven's package does for its MAX77759, through a service; the 6a's chip, bus and address are unread) |
| `vendor/systemd/logind.conf.d/10-omamobile-power-key.conf` | `/usr/lib/systemd/logind.conf.d` | A short power press does not power the phone off. Same drop-in as the Pixel 7 Pro. The long press stays the PMIC reset |
| `vendor/systemd/sleep.conf.d/10-omamobile.conf` | `/usr/lib/systemd/sleep.conf.d` | Suspend means suspend-to-idle: the firmware advertises PSCI SYSTEM_SUSPEND, so the kernel offers `deep`, but Linux has no gs101 sleep sequence and the Pixel 6 Pro doesn't wake from it. No hibernation. Same drop-in as the Pixel 6 Pro package; bluejay's bootargs don't set `mem_sleep_default`, as oriole's don't, so this file is what keeps a `systemctl suspend` on s2idle |
| `vendor/NetworkManager/conf.d/10-omamobile-usb.conf` | `/usr/lib/NetworkManager/conf.d` | Keeps NetworkManager off that link |
| `etc/omamobile-mirrorlist` | `/etc/pacman.d/omamobile-mirrorlist` | Where pacman finds the `[omamobile]` repository. Same file as the Pixel 7 Pro package |
| `omamobile-device-bluejay.install` | pacman hook | Adds `[omamobile]` to `pacman.conf` before `[omarchy]`, once. Same script as the Pixel 7 Pro package |
| `src/reboot-to.c` | `/usr/bin/reboot-to` | Reboot with a reason the bootloader understands. On a first boot the reason most likely isn't stored: the descriptor lists no `INITRAMFS_MODULES`, and raven's records that without its PMIC and reboot-mode modules `reboot-to bootloader` just boots slot B again, so fastboot is by the buttons. Same bytes as the Pixel 7 Pro package; the test checks |
| `etc/boot-update.conf` | `/etc/omamobile/boot-update.conf` | Keeps automatic package boot-image writes disabled; pacman preserves a locally edited policy |
| `device.conf` | `/usr/share/omamobile/device.conf` | The descriptor, the same bytes as `device/bluejay/device.conf` |

`./build [DIR]` compiles `reboot-to` from this directory, `./install DESTDIR [DIR]` stages the files, and `test/all` checks the copies when it is run from the repository.

Not in this package: `omamobile-bootctl` and the boot-success marker (marking slot B on the gs101 phones is on hold, [#252](https://github.com/derekross/omarchy-mobile/issues/252); without the source, `bin/omamobile initramfs` builds its freestanding helpers, as for oriole), the USB rescue, Wi-Fi, the charger, haptics, the display's module options, cameras, any modprobe blacklist, and the slot-B image hook. Each comes in when a booted 6a has given it a runbook and the evidence.

Development policy: `AUTO_WRITE=no` keeps package updates from writing boot
images. **Merge #563 first**: its installer support is required for the
descriptor to give a new installation the same default. The policy file is
a pacman backup file, so a deliberate local setting is preserved on upgrades.
Historical retained-display triggers and experimental hardware startup remain
in the separate bring-up source; this passive package enables neither.
