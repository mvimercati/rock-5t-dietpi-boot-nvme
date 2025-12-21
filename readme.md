# Guide: Installing and Booting DietPi from NVMe on Radxa Rock 5T

This technical guide covers the end-to-end process of migrating DietPi to an NVMe SSD on the **Rock 5T**. It includes the essential steps for SPI flash updates, serial debugging, and correcting the Device Tree (DTB) configuration.

---

## 0. Debugging Setup (Serial Console)
Before starting, it is highly recommended to monitor the boot process to identify any "hangs" or errors.

1.  **Hardware:** Use a **USB-to-UART converter** (e.g., **CH340**).
2.  **Connections:**
    * **TX** (Board) → **RX** (Adapter)
    * **RX** (Board) → **TX** (Adapter)
    * **GND** (Board) → **GND** (Adapter)
3.  **Software:** Use a serial terminal (like PuTTY or Minicom) with the following settings:
    * **Baud rate:** `1500000` (1.5 Mbps)
    * **Data bits:** 8
    * **Stop bits:** 1
    * **Parity:** None

---

## 1. Update SPI Flash (via Official Radxa OS)
The SPI Flash must be updated so the board knows how to initialize the PCIe bus and boot from NVMe.

1.  Flash the **official Radxa OS** to a microSD card.
2.  Boot the Rock 5T and log in.
3.  Run the configuration utility:
    ```bash
    sudo rsetup
    ```
4.  Navigate to **Hardware** -> **Bootloader** -> **Update SPI Flash**.
5.  Power off the board once the update is successful.

---

## 2. Media Preparation
1.  **NVMe SSD:** Flash the **DietPi (Rock 5B)** image onto the SSD using a PC.
2.  **microSD Card:** Flash the **same DietPi image** onto a microSD card. This acts as a temporary "rescue" system.

---

## 3. The "Bridge" Boot (Fixing the NVMe from SD)
By default, the Rock 5B image tries to boot using a 5B Device Tree, which causes a "PCIe Link Fail" on the Rock 5T. We must fix this by booting from the SD card first.

1.  Insert both the **microSD** and the **NVMe** into the Rock 5T.
2.  Connect your Serial Adapter to monitor the logs.
3.  Power on. The system will boot from the SD card.
4.  Log in as `root` (password: `dietpi`).
5.  Identify the NVMe partition and mount it:
    ```bash
    lsblk
    # Usually /dev/nvme0n1p1
    mount /dev/nvme0n1p1 /mnt
    ```

---

## 4. Edit NVMe Configuration
Now, modify the configuration on the SSD to match the specific hardware of the Rock 5T.

1.  Open the environment file on the **mounted NVMe**:
    ```bash
    nano /mnt/boot/dietpiEnv.txt
    ```
2.  **Apply the following changes:**
    * **fdtfile:** Change to `rockchip/rk3588-rock-5t.dtb`
    * **overlay_prefix:** Change to start with `rock-5t`
    * **extraargs:** Add `rootwait` to ensure the kernel waits for the PCIe bus to be ready.

    **Example of a corrected `dietpiEnv.txt`:**
    ```text
    rootdev=UUID=2e174d60-3f57-435f-8ba9-139d86df3198
    rootfstype=ext4
    consoleargs=console=ttyFIQ0,1500000 console=tty1
    extraargs=fsck.repair=yes net.ifnames=0 rootwait
    overlay_path=rockchip
    overlay_prefix=rock-5t rock-5 rockchip-rk3588 rk3588 rockchip
    fdtfile=rockchip/rk3588-rock-5t.dtb
    ```

3.  Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).
4.  Unmount the drive and power off:
    ```bash
    umount /mnt
    poweroff
    ```

---

## 5. Final Boot from NVMe
1.  **Remove the microSD card.**
2.  Power on the Rock 5T.
3.  Watch your Serial Console. You should see the PCIe link successfully initialize, followed by the full DietPi boot sequence from the NVMe SSD.

---

**Note:** If `rk3588-rock-5t.dtb` is missing from the DietPi image, copy it from the official Radxa SD card (`/boot/dtb/rockchip/`) into the same folder on the NVMe partition.
