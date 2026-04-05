---
layout: post
title: "Building a Custom Yocto Linux Image for Raspberry Pi 3 on WSL2"
date: 2026-04-05
categories: [embedded-linux, yocto]
tags: [yocto, raspberry-pi, wsl2, embedded-linux, bitbake, linux]
---

Building a custom Linux distribution from scratch used to feel like dark magic to me. Coming from a deeply embedded background (AUTOSAR, TriCore, bare-metal ECUs), the idea of assembling an entire OS — bootloader, kernel, root filesystem — seemed like overkill. Then I started working with the Yocto Project, and it clicked: it's just a very sophisticated build system for embedded targets.

This post walks through the end-to-end process of building a highly optimized, headless `core-image-minimal` for a Raspberry Pi 3 Model B using Yocto, with the build environment running on Ubuntu via WSL2. I'll also cover the WSL-specific gotchas I ran into — particularly the infamous clock-drift issue that wrecks long builds.

## What You'll Need

**Hardware:**
- Raspberry Pi 3 Model B
- MicroSD card (8GB or larger)
- 5V/2.5A power supply
- 3.3V USB-to-TTL serial cable (PL2303, CP210x, or similar)

**Host system:**
- Windows 11 with WSL2 running Ubuntu
- A high-speed external SSD with plenty of free space (Yocto builds are large — plan for 50GB+)

**Software:**
- BalenaEtcher (for flashing the SD card)
- PuTTY (for serial console access)

---

## Part 1: Host Environment Setup (WSL2 Ubuntu)

### Configuring the Default User

By default, WSL runs as the assigned standard user. To make sure file ownership and permissions work correctly throughout the build, append your user configuration to `/etc/wsl.conf`:

```bash
echo "[user]" | sudo tee -a /etc/wsl.conf
echo "default=saeid" | sudo tee -a /etc/wsl.conf
```

Then exit the terminal, run `wsl --shutdown` in Windows PowerShell, and reopen your WSL session to apply the changes.

### Installing Yocto Dependencies

Yocto has a long list of build dependencies. On modern Ubuntu (22.04/24.04), a few legacy package names have been replaced — `libegl1` instead of `libegl1-mesa`, and `libsdl2-dev` instead of `libsdl1.2-dev`:

```bash
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
  chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
  iputils-ping python3-git python3-jinja2 libegl1 libsdl2-dev pylint xterm \
  python3-subunit mesa-common-dev zstd liblz4-tool
```

### Fixing the Locale (Important!)

BitBake — the build engine at the heart of Yocto — strictly requires a UTF-8 locale. If you skip this step, you'll get cryptic parsing errors from the BitBake server. Generate and export the locale:

```bash
sudo apt install -y locales
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

---

## Part 2: Yocto Project Initialization

### Cloning the Repositories

Create a workspace and clone the core Poky repository along with the Raspberry Pi BSP layer. I'm using the stable `scarthgap` release for both to keep things consistent:

```bash
mkdir -p ~/yocto-rpi
cd ~/yocto-rpi
git clone -b scarthgap git://git.yoctoproject.org/poky.git
git clone -b scarthgap git://git.yoctoproject.org/meta-raspberrypi
```

### Setting Up the Build Environment

Source the initialization script. It automatically changes into the new build directory for you:

```bash
source poky/oe-init-build-env build-rpi
```

### Adding Layers and Configuring the Target

Add the Raspberry Pi layer to the project:

```bash
bitbake-layers add-layer ../meta-raspberrypi
```

Then append the target machine and enable the UART port — the UART is essential for headless debugging since we won't have a monitor attached:

```bash
echo 'MACHINE = "raspberrypi3"' >> conf/local.conf
echo 'ENABLE_UART = "1"' >> conf/local.conf
```

---

## Part 3: Building the Image (and Troubleshooting WSL Quirks)

Kick off the build with:

```bash
bitbake core-image-minimal
```

Now go get coffee. Or lunch. Or sleep. Depending on your machine, the first build can take anywhere from 1 to 6+ hours since every package is compiled from source.

### The WSL Clock-Drift Problem

Here's the gotcha that cost me the most time: when WSL sleeps or suspends, the Linux kernel's internal clock pauses while the Windows host clock keeps ticking. When WSL wakes back up, there's a time discrepancy — and `make` starts seeing files with timestamps "from the future." This manifests as strange compilation errors in packages like `perl-native` or ELF linking failures in `libcap`.

**The fix** is an atomic cleanup combined with a clock resync:

1. Exit WSL completely and run `wsl --shutdown` in Windows PowerShell.
2. Re-enter WSL and source the environment again:

    ```bash
    cd ~/yocto-rpi
    source poky/oe-init-build-env build-rpi
    ```

3. Clean the corrupted package completely:

    ```bash
    bitbake -c cleanall libcap
    ```

4. Rebuild that package individually, then resume the main build:

    ```bash
    bitbake libcap
    bitbake core-image-minimal
    ```

This has saved me multiple times. If you see weird time-related build failures on WSL, this is almost always the cause.

---

## Part 4: Flashing the Image

Once the build completes, your image lives in the deploy directory:

- **Path via Windows Explorer:** `\\wsl$\Ubuntu\home\saeid\yocto-rpi\build-rpi\tmp\deploy\images\raspberrypi3`
- **Target file:** Look for the `.wic.bz2` file, something like `core-image-minimal-raspberrypi3.rootfs-[timestamp].wic.bz2`

To flash it:

1. Open BalenaEtcher on Windows.
2. Select the `.wic.bz2` file directly — no need to extract it first.
3. Select your MicroSD card as the target.
4. Click **Flash!**

After flashing, Windows will probably pop up several dialogs asking you to format the drive. **Ignore and cancel all of them.** Windows can't read ext4 Linux partitions, so it assumes the card is broken.

---

## Part 5: Connecting via Serial Console

Since we built a headless image, we need the UART to interact with the board on first boot.

### Wiring the UART

**Warning:** Never connect the 5V wire from the serial cable if the Pi is already powered via its Micro-USB port. You'll back-power the board through the logic pins and potentially damage it.

Connect three wires only:

| Serial Cable | Raspberry Pi | Function |
|---|---|---|
| GND (black) | Pin 6 | Ground |
| RX (green) | Pin 8 | TXD / GPIO 14 |
| TX (white) | Pin 10 | RXD / GPIO 15 |

Note the crossover: the cable's RX goes to the Pi's TX, and vice versa.

### Configuring PuTTY

1. Plug the USB side of the cable into your host PC and check Windows Device Manager to find the assigned COM port.
2. Open PuTTY.
3. Select **Serial** as the connection type.
4. Enter the COM port number.
5. Set the baud rate (Speed) to **115200**.
6. Open the connection, then power on the Pi.

### First Boot

You'll see U-Boot messages stream by, followed by the Linux kernel log. Once boot completes, a login prompt appears:

- **Username:** `root`
- **Password:** (none by default)

And that's it — you're logged into your very own custom Linux distribution that you compiled from source.

---

## Closing Thoughts

Coming from AUTOSAR Classic and bare-metal ECU work, the Yocto workflow felt overwhelming at first. But once you internalize that it's "just" a recipe-driven build system producing bootable images, it stops being mysterious. The hardest part, for me, has been learning to trust the build system and resist the urge to manually patch things that should be fixed in recipes.

Next steps I'm exploring: custom layers for application code, SystemD service integration, and eventually an automotive-flavored image with CAN stack support. If you're coming from a similar deeply-embedded background, I'd love to hear how your Yocto journey is going.
