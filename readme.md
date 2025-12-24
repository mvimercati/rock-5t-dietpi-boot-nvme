# Guide: Installing and Booting DietPi from NVMe on Radxa Rock 5T

This technical guide covers the end-to-end process of installing DietPi to an NVMe SSD on the **Rock 5T**. It includes the steps for SPI flash updates which enables booting from nvme, serial debugging, and correcting the Device Tree (DTB) configuration.

---

## 0. Debugging Setup (Serial Console)
Before starting, it is highly recommended to monitor the boot process to identify any "hangs" or errors.

1.  **Hardware:** Use a **USB-to-UART converter** (e.g., **CH340**, here I used an ESP32-S3, just overkill for the scope but it includes a CH340 uart-usb converter).
2.  **Connections:**
    * **TX** (pin 8) → white line → **RX** (Adapter)
    * **RX** (pin 10) → green line → **TX** (Adapter)
    * **GND** (pin 6) → black line → **GND** (Adapter)
  
   <img width="1000" height="600" alt="immagine" src="https://github.com/user-attachments/assets/837ca6c1-d94e-4dec-ba01-3ded98c988e0" />

4.  **Software:** Use a serial terminal (like PuTTY or Minicom) with the following settings:
    * **Baud rate:** `1500000` (1.5 Mbps)
    * **Data bits:** 8
    * **Stop bits:** 1
    * **Parity:** None
    * **Flow control:** Disabled
  
    <img width="1000" height="600" alt="immagine" src="https://github.com/user-attachments/assets/41eb8bfe-1952-48c4-99db-6aad529c4689" />


---

## 1. Update SPI Flash (via Official Radxa OS)
The SPI Flash must be updated so the board knows how to initialize the PCIe bus and boot from NVMe.
For this step I just followed the Radxa guide here: https://docs.radxa.com/en/rock5/rock5t/getting-started/install-os/nvme

1.  Flash the **official Radxa OS** to a microSD card.
2.  Boot the Rock 5T and log in.
3.  Run the configuration utility:
    ```bash
    sudo apt-get update
    sudo apt-get full-upgrade
    sudo rsetup
    ```
4.  Navigate to **Hardware** -> **Bootloader** -> **Update SPI Flash**.
5.  Power off the board once the update is successful.
6.  Now the board shall be able to boot from NVME, but we have to load something in it.

---

## 2. DietPI Media Preparation

1.  **microSD Card:** Flash the **DietPi image** onto a microSD card. This acts as a temporary "rescue" system. At the moment only the image for Rock 5B is available.
    I used this one: https://dietpi.com/downloads/images/DietPi_ROCK5B-ARMv8-Trixie.img.xz

3.  Start the sbc with the DietPI image on the microSD card. Follow the preliminary setup process, useless because we will repeat on the nvme.
4.  Once we have the prompt flash the DietPI image on the nvme too. This may be done by downloading again the diet pi image.
5.  **NVMe SSD:** Flash the **DietPi (Rock 5B)** image:
6.  ```bash
    wget https://dietpi.com/downloads/images/DietPi_ROCK5B-ARMv8-Trixie.img.xz
    DietPi_ROCK5B-ARMv8-Trixie.img.xz | dd of=/dev/
    xzcat DietPi_ROCK5B-ARMv8-Trixie.img.xz | dd of=/dev/nvme0n1 bs=64M status=progress
    ```
   
---

## 3. Fixing the device tree
By default, the Rock 5B image tries to boot using a 5B Device Tree, which causes a "PCIe Link Fail" on the Rock 5T. We must fix this by booting from the SD card first.

1.  Insert both the dietPI **microSD** and the **NVMe** into the Rock 5T.
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
4.  Proceed with the DietPI setup (Started now from the beginning, as we are booting from SSD)

