# Qubes USB Keyfile — dracut Setup

Konfiguration für einen USB-Stick mit LUKS-Keyfile als Entsperrmethode (Plattform: Fedora/Qubes).

---

## `/usr/lib/dracut/modules.d/91qubes-usbkey/qubes-usb-keyfile.sh`

```sh
#!/bin/sh
LABEL="QUESUSBKEY"
KEYFILE="qubes-keyfile"
MOUNT="/mnt/usb"

# --- Versuch 1: USB-Stick ---
for i in 1 2 3 4 5 6 7 8 9 10; do
    DEV=$(blkid -L "$LABEL" 2>/dev/null)
    [ -n "$DEV" ] && break
    sleep 1
done

if [ -n "$DEV" ]; then
    mkdir -p "$MOUNT"
    if mount -o ro "$DEV" "$MOUNT" 2>/dev/null; then
        cat "$MOUNT/$KEYFILE"
        umount "$MOUNT"
        exit 0
    fi
fi

# --- Versuch 2: Passwort via Plymouth ---
if plymouth --ping 2>/dev/null; then
    plymouth ask-for-password --prompt="Disk password:"
    exit $?
fi

# --- Versuch 3: Konsole (Fallback) ---
echo "" > /dev/console
echo "=== USB-Stick nicht gefunden. Disk-Passwort: ===" > /dev/console
echo -n "> " > /dev/console
stty -echo < /dev/console
read -r PASSWORD < /dev/console
stty echo < /dev/console
echo "" > /dev/console
echo "$PASSWORD"
```

---

## `/usr/lib/dracut/modules.d/91qubes-usbkey/mnt-usb.mount`

```ini
[Unit]
Description=USB Stick with LUKS keyfile
Before=cryptsetup-pre.target
After=systemd-udevd.service

[Mount]
What=/dev/disk/by-uuid/DEINE-USB-UUID
Where=/mnt/usb
Type=auto
Options=ro,noauto

[Install]
WantedBy=multi-user.target
```

---

## `/usr/lib/dracut/modules.d/91qubes-usbkey/module-setup.sh`

```bash
#!/bin/bash
check() {
	return 0
}

depends() {
	echo "systemd systemd-cryptsetup crypt"
	return 0
}

install() {
	inst_simple "$moddir/mnt-usb.mount" "/etc/systemd/system/mnt-usb.mount"
	inst_simple "$moddir/luks.service.d/override.conf" "/etc/systemd/system/systemd-cryptsetup@.service.d/override.conf"
	mkdir -p "${initdir}/etc/systemd/system/mnt-usb.mount.wants"
	ln -sf "../cryptsetup-pre.target" "${initdir}/etc/systemd/system/mnt-usb.mount.wants/cryptsetup-pre.target"
}
```

---

## `/usr/lib/dracut/modules.d/91qubes-usbkey/luks.service.d/override.conf`

```ini
[Unit]
RequiresMountsFor=/mnt/usb
```

---

## `/etc/crypttab`

```fstab
luks-DEINE-LUKS-UUID /dev/sda2 /mnt/usb/qubes-keyfile luks,nofail
```

---

## `/etc/dracut.conf.d/qubes-usbkey.conf`

```bash
add_dracutmodules+=" qubes-usbkey plymouth systemd systemd-cryptsetup "
omit_dracutmodules+=" Plymouth "
force_drivers+=" usb-storage "
```

---

## `/etc/default/grub`

```bash
In GRUB_CMDLINE_LINUX:
remove: rd.luks.uuid=....
add: rd.luks.timeout=90
```

---

## Anwendung

```bash
sudo dracut -f && sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```
