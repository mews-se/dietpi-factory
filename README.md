# dietpi-factory

[![ShellCheck](https://github.com/mews-se/dietpi-factory/actions/workflows/shellcheck.yml/badge.svg)](https://github.com/mews-se/dietpi-factory/actions/workflows/shellcheck.yml)
![Shell: Bash](https://img.shields.io/badge/shell-bash-4EAA25.svg?logo=gnubash&logoColor=white)
![Platform: DietPi / Debian](https://img.shields.io/badge/platform-DietPi%20%2F%20Debian-A81D33.svg?logo=debian&logoColor=white)
![Deploys to: Proxmox](https://img.shields.io/badge/deploys%20to-Proxmox-E57000.svg?logo=proxmox&logoColor=white)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

dietpi-factory sets up new [DietPi](https://github.com/MichaIng/DietPi) machines unattended. Describe the setup once in a profile - hostname, network, SSH, software - and deploy it to Proxmox containers and VMs, flashable images for the Pi and bare metal, or a Debian system that is already running. The machine boots headless and configures itself, driven by DietPi's own automation.

## Profiles

```
./factory.sh
```

The wizard asks for hostname, network, SSH server and key, password, software picks and an optional own first boot script, and writes profiles/\<name\>/. Profiles can contain passwords and keys, so the directory is gitignored - copy it to wherever the deployment script runs.

Every deployment script takes a profile directory as first argument (second for bake-image.sh) or the PROFILE_DIR variable. With the `bash -c` one liners the argument needs a placeholder in front, since the first word after the command becomes `$0`:

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/mews-se/dietpi-factory/main/proxmox/create-dietpi-lxc.sh)" _ profiles/myprofile
```

Without a profile, minimal defaults are used: DHCP, headless, unattended, DietPi's own choices for everything else. `ASSUME_DEFAULTS=1` skips all dialogs in the Proxmox scripts.

## Proxmox LXC

Run on the Proxmox host:

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/mews-se/dietpi-factory/main/proxmox/create-dietpi-lxc.sh)"
```

Answer the dialogs (container ID, resources, storage, DHCP or static). A Debian container is created and converted to DietPi with the official dietpi-installer, then restarted so the first boot setup finishes on its own.

## Proxmox VM

Run on the Proxmox host:

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/mews-se/dietpi-factory/main/proxmox/create-dietpi-vm.sh)"
```

The official Proxmox qcow2 is downloaded, the profile is injected into the disk image before the first boot and the VM sets itself up unattended. The disk is grown when a larger size than the image's 8 GiB is requested. Both BIOS (default) and UEFI images are supported, UEFI gets OVMF and an EFI disk with Secure Boot ready, the images ship the signed Debian boot chain. The disk is attached with discard and SSD emulation so TRIM reaches thin storage.

## Flashable image (Pi and bare metal)

Run as root on any Linux box:

```
sudo scripts/bake-image.sh RPi5 profiles/myprofile
```

The image argument is a name or search term matched against [dietpi.com/downloads/images](https://dietpi.com/downloads/images/), a URL or a local file. A partial match like `RPi5` opens a menu with the variants. Downloads are cached and verified in /var/cache/dietpi-factory and reused across runs. The result lands in build/ as a ready-to-flash .img for dd or Etcher. The output is published atomically; when two bakes write the same output name, the last one finished wins.

## Convert an existing Debian system

Run on the target machine (VPS, VM or bare metal):

```
bash -c "$(curl -fsSL https://raw.githubusercontent.com/mews-se/dietpi-factory/main/vps/convert-to-dietpi.sh)"
```

Treat it as a reinstall: everything outside the base system is removed, including user home directories. Hardware model and Debian release are detected automatically, and the profile's SSH key is installed for root before the reboot. Make sure the profile carries your SSH key before converting a remote machine. `ASSUME_YES=1` skips the confirmation.

The conversion purges whatever manages the network today, and the installer downloads its own packages only afterwards. A session running over a wireless interface that ifupdown does not manage is therefore refused outright: the run would die offline halfway and the machine would not come back. Convert those over Ethernet, or flash an image instead. On hardware that can use WiFi the script asks whether to install the WiFi stack; answer yes and the SSID, key and country code are carried over from NetworkManager, netplan or wpa_supplicant. `WIFI=0|1` answers up front.

## After deployment

The machine finishes its first boot setup by itself and registers its hostname over DHCP. Log in as root or `dietpi` with the profile password, or with the profile's SSH key.

With the default password the first run setup logs two FAILED lines about "password (software check)": DietPi notices the default password and tries to prompt for a better one, which no one can answer unattended. Harmless, and gone as soon as the profile carries an own password (the wizard always asks for one).

## Layout

```
factory.sh                      profile wizard, writes profiles/<name>/ (gitignored)
config/dietpi.txt               base profile
proxmox/create-dietpi-lxc.sh    LXC creator
proxmox/create-dietpi-vm.sh     VM creator from the official Proxmox qcow2
scripts/bake-image.sh           bakes a profile into an official image
vps/convert-to-dietpi.sh        converts a running Debian system to DietPi
```

A generic DietPi LXC script pair in community-scripts layout lives in the [ProxmoxVED fork](https://github.com/mews-se/ProxmoxVED/tree/add-dietpi), aimed at an upstream PR.

## Notes

DietPi reads /boot/dietpi.txt at first boot. With AUTO_SETUP_AUTOMATED=1 the whole setup runs unattended and /boot/Automation_Custom_Script.sh runs at the end when provided. See the [DietPi docs](https://dietpi.com/docs/usage/#how-to-do-an-automatic-base-installation-at-first-boot) and [dietpi.txt](https://github.com/MichaIng/DietPi/blob/master/dietpi.txt).

A few things learned the hard way: dietpi-installer replaces dietpi.txt (the profile is merged in afterwards), it removes ifupdown2 without a replacement (ifupdown is reinstalled), DISTRO_TARGET must be preset when there is no tty, and the installer stamps the branch it was launched from as the permanent dietpi-update target - the scripts point it back at master afterwards, or updates would never arrive.

## Credits

[DietPi](https://github.com/MichaIng/DietPi) by MichaIng does the actual heavy lifting, this factory just feeds its automation. The one liner style is borrowed from [community-scripts](https://community-scripts.github.io/ProxmoxVE/) and the VM import took inspiration from [dazeb's proxmox-dietpi-installer](https://github.com/dazeb/proxmox-dietpi-installer).

## License

MIT
