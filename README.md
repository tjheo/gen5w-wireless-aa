# Gen5W Wi-Fi / Hotspot / Wireless AA/CarPlay release

## Installation order

1. `01-wifi-enable-phase1.zip`
2. `02-wifi-hotspot-install.zip`

Extract each archive to the root of a blank FAT32/exFAT USB drive. Do not copy the outer ZIP itself to the head unit.

### Phase 1: safe Wi-Fi activation check

Phase 1 changes no `/system` file. It snapshots the original Wi-Fi state, enables Wi-Fi, and waits for the stock HAL to create its genuine supplicant configuration and CRC file.

Continue only when `wifi-enable-report.txt` says `RESULT: SUCCESS` (or `NOTHING TO DO`). A `NOT YET` result now also shows the NEMESIS failure dialog; earlier builds showed a success dialog even then.

### Phase 2: integrated package 2.1.0

Upgrading from 2.0.8 or 2.0.9: uninstall the old package and reboot first, then install 2.1.0. Installed over 2.0.9 directly, 2.1.0 finds the 2.0.9 receiver patch, leaves it in place and skips the new USB fix.

Do not use 2.0.1. It changed the owner of `/data/misc/dhcp`, which broke DHCP for Wi-Fi and the LTE modem and made the unit lag. 2.0.2 removes that change and never alters existing system directories.

Phase 2 installs `Wi-Fi / Hotspot / Wireless AA/CarPlay`. It:

- changes the stock Settings telematics tile to Data/Network;
- exposes and separately labels Wi-Fi and Hotspot settings;
- enables wireless Android Auto and CarPlay feature/configuration gates;
- applies exact-firmware projection coexistence ODEX patches with verified stock backups;
- keeps vehicle LTE forwarding blocked from the hotspot;
- opens a package-specific dashboard when MAP is held for at least three seconds;
- lets the driver select Wi-Fi client, user Hotspot, wireless phone projection, or both off at boot;
- leaves ADB untouched unless a valid `wifi-adb/adbkey.pub` is deliberately supplied.

Reboot once after installation.

## Dashboard boot mode

The four dashboard choices are mutually exclusive. Selecting one saves it immediately and applies it to the current session. The saved choice is applied again when the package service starts after boot. Wireless phone projection turns both Wi-Fi client and user Hotspot off so the stock projection app can raise its own access point.

Wireless Android Auto: this head unit starts its access point with the Hotspot SSID/password but tells the phone a separate "connectivity" SSID/password, so the phone never finds the network. The package keeps the connectivity pair equal to the Hotspot pair shown in Settings (only while the access point is off, re-checked every 10 seconds), so changing the Hotspot name or password is followed automatically. Uninstall restores the original connectivity pair.

In wireless phone projection mode the package turns Wi-Fi client off but never stops the projection access point, so applying the mode at boot or saving it again cannot break a session that is starting.

Wireless Android Auto on this head unit also needs:

- no CarPlay/Android Auto USB device (for example a wireless adapter dongle) plugged into the car; the stock app does not start wireless projection while one is attached. USB sticks and other devices that are not phones no longer block it (2.1.0, see below);
- the phone paired once and accepted in the first-use consent. If the phone was deleted and re-paired, the package falls back to the phone's Bluetooth name (2.1.0, not yet confirmed on the car); if that does not work, connect it once again from Bluetooth > the phone > device connection settings > connect.

Switching modes without a reboot: the package applies one mode change at a time (repeated taps on save collapse into one). When leaving projection it first asks the stock PhoneProjection app to drop the wireless session (2.1.0), so the stock app does not restart its access point mid-switch, then waits until the access point is fully off before turning the Wi-Fi client on. Earlier builds could leave Wi-Fi failing until the next reboot.

Three stock-app problems stopped wireless Android Auto after power-on, and the package handles them:

- The stock Android Auto receiver (HKMC_GPM) treated the first wireless connection after boot as a brand-new device and sent a request the stock PhoneProjection app rejects. The package patches one instruction in `/system/app/HKMC_GPM.odex` so the receiver always uses the stored-device path that a retry uses. The patch is applied only to the exact stock file (hash checked), the original is backed up, and uninstall restores it.
- With a USB device that is not a phone plugged in at boot (for example a USB stick), the receiver reserved its single session for wired Android Auto, never started it, and never released it, so every wireless attempt was rejected. 2.1.0 changes one more instruction in the same file: the receiver checks whether the USB device looks like a phone before it reserves the wired session. Anything else gets the stock "charging only" handling and wireless Android Auto proceeds. Phones and devices that look like phones (for example a USB monitor) keep the stock behavior: the receiver tries wired Android Auto, and if the device cannot switch to accessory mode the stock code releases the session by itself (verified on the car with a USB monitor). Wired Android Auto with a phone and the stock wireless/wired switching popup are unchanged.
- The phone's Bluetooth can connect before the stock PhoneProjection wireless service has started, so the receiver's three device messages are lost and the projection access point never starts. Each time the phone's Bluetooth connects, the package watches for up to five minutes. In projection mode, if the phone's Android Auto Bluetooth channel and the stock wireless service are both up and no device has joined the projection access point for 12 seconds, it sends the same three messages again through the stock `phoneprojection` service, using the phone's entry in the stock device list (or the phone's Bluetooth name when the list has no entry). It turns the Wi-Fi client off first if it is on. At most two attempts, 40 seconds apart, per Bluetooth connection. It changes no file and does not touch Bluetooth.

If wireless Android Auto still does not appear, turn the phone's Bluetooth off and on once. The stock app can also stay stuck on an earlier attempt ("Already AA connecting") until the Bluetooth link drops.

Each time the head unit runs its own access point for about 20-30 seconds, the package writes a read-only diagnostic snapshot to `/data/nemesis/wifi.hotspot.adb.ap-capture.txt`.

While the saved mode is wireless phone projection, the package also records the projection, Wi-Fi and access-point logs from early boot to `/data/nemesis/aa-debug/aa.log` (rotating, at most about 2.5 MB). It only reads logs and writes those files. Uninstall stops the recorder and leaves the log files in place.

## Optional authenticated wireless ADB

The shared release contains **no ADB key**. To opt in, add your own Android ADB public key to the extracted phase-2 stick:

```text
wifi-adb/
  adbkey.pub
```

The package requires `ro.adb.secure=1`, adds only that key, enables tcp/5555, and restricts the port to `wlan+` rather than vehicle LTE interfaces.

## Validation and limitations

The EC2/QEMU replica completed installation, reboot, Android boot, PackageManager feature activation, PhoneProjection/DiPO ODEX loading, MAP long-press dashboard, boot-mode selections, normal uninstall, and clean boot after rollback. For 2.0.2 it also confirmed that `/data/misc` directory owners and modes are unchanged after install, reboot and uninstall, that exactly one dashboard process runs, and that every restored `/system` file hash matches the pre-install hash. QEMU has no real Broadcom/MICOM/modem hardware, so actual wireless phone discovery, pairing, and projection media transport must be tested on the vehicle.

## Rollback

Normal NEMESIS uninstall restores all owned XML, build.prop, onebin feature configuration, supplicant, ODEX, runtime property, firewall, optional ADB, and phase-1 changes. Reboot after uninstall.

## Redistribution

The public package contains patched proprietary OEM ODEX payloads. Confirm redistribution rights before sharing it. Never distribute `wifi-hotspot-adb-stick.zip`; it contains a personal ADB public key.
