# Rebuilding the klipper host

The steps that rebuild the klipper host from an empty SD card: Raspberry Pi OS Lite, then Klipper, Moonraker and
Mainsail installed by KIAUH, with Klipper taken from the fork's printer branch. Every step after the first boot runs
from bonsai over `ssh klipper-server`; none needs the Pi's console. The printer's configuration and its bring-up belong
to the klipper project.

## 1. Write the card

In Raspberry Pi Imager on bonsai:

- Device Raspberry Pi 3; OS: Raspberry Pi OS (other), Raspberry Pi OS Lite (64-bit).
- Settings: hostname `klipper`; user `admin` with a password; SSH by public key only, with bonsai's key; keyboard
  layout US; no Wi-Fi (the Pi is on Ethernet).
- Imager's time zone choice also sets the keyboard; choose the US layout and fix the time zone in step 2.

Let Imager verify the card.

## 2. Before the first boot

Raspberry Pi OS on trixie sets the system up at its first boot with cloud-init, from `user-data` on the card's boot
partition (label `bootfs`). Imager writes `admin` there without sudo rights (`sudo: null`). With the card still in
bonsai's reader, give `admin` sudo without a password, set the time zone and add the full host name, so the Pi comes
up ready for the steps that follow:

```bash
(
  set -e
  parts=$(lsblk -rpno NAME,LABEL,RM | awk '$2 == "bootfs" && $3 == "1" {print $1}')
  [[ $(wc -w <<< "$parts") -eq 1 ]] || { echo "expected one removable bootfs partition, found: $parts"; exit 1; }
  mnt=$(mktemp -d)
  sudo mount "$parts" "$mnt"
  sudo sed -i \
    -e 's|^  sudo: null$|  sudo: "ALL=(ALL) NOPASSWD:ALL"|' \
    -e 's|^timezone: .*|timezone: "Europe/Istanbul"|' \
    -e 's|^hostname: "klipper"$|&\nfqdn: "klipper.home.arpa"|' \
    "$mnt/user-data"
  grep -E '^(hostname|fqdn|timezone):|^  sudo:' "$mnt/user-data"
  sudo umount "$mnt"
  rmdir "$mnt"
)
```

The block runs in a subshell, so a failed check stops it without closing the terminal.

The `grep` must show four lines: `hostname`, `fqdn`, `timezone` and `sudo` with the new values. A line missing means
Imager wrote `user-data` differently: edit it by hand to the same result. The edit was tested on a copy of
`user-data` from Imager and validated with `cloud-init schema`, but has not yet run through a first boot; on the last
rebuild the card booted before it, and the fallback below was used.

Fallback, after the first boot, if this step was skipped. Run the sudo rule from a terminal, because it asks for
`admin`'s password; `visudo` is in `/usr/sbin`, which is not on a user's `PATH`:

```bash
ssh -t klipper-server 'echo "admin ALL=(ALL) NOPASSWD: ALL" > /tmp/admin-nopasswd \
  && /usr/sbin/visudo -cf /tmp/admin-nopasswd \
  && sudo install -m 0440 -o root -g root /tmp/admin-nopasswd /etc/sudoers.d/010_admin-nopasswd \
  && rm /tmp/admin-nopasswd'
ssh klipper-server 'sudo timedatectl set-timezone Europe/Istanbul
  echo "fqdn: klipper.home.arpa" | sudo tee /etc/cloud/cloud.cfg.d/99-fqdn.cfg
  sudo cloud-init single --name update_etc_hosts --frequency always'
```

## 3. First boot

- Put the card in the Pi and boot it. The first boot grows the root filesystem to the card and takes a few minutes.
- The fresh install has new SSH host keys: update the pin for `klipper-server` before using the alias.
- The DHCP reservation is bound to the Pi's Ethernet MAC, not to the card, so the address and the DNS name stay.

Check: `ssh klipper-server 'sudo -n true && hostname -f'` prints `klipper.home.arpa`.

## 4. System upgrade

```bash
ssh klipper-server 'export DEBIAN_FRONTEND=noninteractive
  sudo -E apt-get update -q
  sudo -E apt-get -y -q -o Dpkg::Options::=--force-confold full-upgrade
  sudo apt-get clean
  sudo systemctl reboot'
```

The kernel and initramfs steps take several minutes on a Pi 3 and print little; do not cut the power during them.

## 5. Klipper, Moonraker and Mainsail

```bash
ssh klipper-server 'set -e; export TERM=xterm DEBIAN_FRONTEND=noninteractive; cd ~
  sudo -E apt-get install -y -q git
  git clone -q https://github.com/dw-0/kiauh.git
  sed "s|^    https://github.com/Klipper3d/klipper$|    https://github.com/timaydin/klipper, local_skrpro_v1.2|" \
    kiauh/default.kiauh.cfg > kiauh/kiauh.cfg
  grep -q "timaydin/klipper, local_skrpro_v1.2" kiauh/kiauh.cfg
  ./kiauh/kiauh.sh install klipper > ~/kiauh-klipper.log 2>&1
  ./kiauh/kiauh.sh install moonraker --create-example-cfg > ~/kiauh-moonraker.log 2>&1
  ./kiauh/kiauh.sh install mainsail --install-config > ~/kiauh-mainsail.log 2>&1 || true'
```

- The Mainsail step ends with "Mainsail-Config installation failed": it cannot add `[include mainsail.cfg]` to a
  `printer.cfg` that does not exist yet. Everything else of the step is done; the include is part of the printer's
  configuration.
- On a Pi 3 the whole step takes about half an hour, most of it compiling Python packages.

## 6. Moonraker's access

In `~/printer_data/config/moonraker.conf`, replace the `[authorization]` section with:

```ini
[authorization]
# Clients that may use the API without logging in: the LAN and this host.
trusted_clients:
    10.2.1.0/24
    127.0.0.0/8
    ::1/128
# Web origins allowed to call the API: our LAN domain and mDNS names.
cors_domains:
    *.home.arpa
    *.local
    *://localhost
    *://localhost:*
```

Then restart Moonraker and refresh its update manager, which can hold a stale state of `mainsail-config` from the
install:

```bash
ssh klipper-server 'sudo systemctl restart moonraker'
curl -s -X POST "http://klipper.home.arpa/machine/update/refresh?name=mainsail-config"
```

## 7. Checks

- `ssh klipper-server 'systemctl is-active klipper moonraker nginx'`: all three active.
- `http://klipper.home.arpa/` serves Mainsail; `http://klipper.home.arpa/server/info` shows no failed components and
  no warnings. Klippy is in `error` until `printer.cfg` exists.
- `http://klipper.home.arpa/machine/update/status`: `klipper`, `moonraker`, `mainsail` and `mainsail-config` valid.
  Klipper's warning about an unofficial remote is expected: updates come from the fork's branch.

## Gotchas

- **KIAUH needs a terminal type that `clear` accepts.** Its start script runs `clear` under `set -e`; with
  `TERM=dumb` it exits at once, with no output.
- **KIAUH's command-line mode cannot answer questions.** A leftover `~/klipper` or `~/moonraker` makes it ask whether
  to overwrite, and the install fails. Start from a home folder without them.
- **Raspberry Pi OS Lite has no git.** Install it before cloning KIAUH.
- **KIAUH installs the ARM and AVR firmware toolchains with Klipper,** and apt adds the recommended ARM C++ library,
  which Klipper does not use. Together they take about 3.4 GB, the C++ library alone about 2.1 GB. An 8 GB card
  runs out of space during the package step and leaves dpkg half done; `sudo apt-get clean` and
  `sudo dpkg --configure -a` repair it. Use a card of 16 GB or more.
- **The Pi 3 runs hot.** Without free air around it, it reaches its soft temperature limit and slows down; check with
  `vcgencmd get_throttled`.
