# 🍊 Orange Pi Zero 3: Logic Analyzer & Bus Pirate Tool 🛠️

A comprehensive guide to transforming the **Orange Pi Zero 3** Single Board Computer (SBC) into a powerful **Logic Analyzer** and **Bus Pirate** tool by leveraging its on-board peripherals (I2C, SPI, UART, GPIO).

---

## ✨ Project Overview

This project utilizes the low-cost, powerful Orange Pi Zero 3 and a Linux-based operating system (Ubuntu Server) to create a versatile hardware debugging tool. By enabling and configuring specific kernel overlays, we unlock the necessary communication interfaces (I2C, SPI, UART) and install key software components like `sigrok` and `pyftdi` (though the latter is implicitly prepared for by the library installs) to perform protocol analysis and interactive bus communication.

### 🎯 Key Capabilities Enabled

* **Logic Analyzer:** Using tools like `sigrok` for protocol decoding (requires additional setup of the sigrok frontend like PulseView, which is not covered but the backend is prepared).
* **Bus Pirate:** Enabling interactive communication with peripheral devices over **I2C**, **SPI**, and **UART** using Python libraries and command-line tools.
* **Remote Access:** Full remote SSH access after initial serial setup.

---

## 🚀 Prerequisites

* **Hardware:**
    * Orange Pi Zero 3 SBC
    * 16GB or larger MicroSD Card
    * USB-C Power Cable (for powering the Pi)
    * **USB to Serial Cable** (essential for initial headless setup)
    * PC with a USB port and a terminal program (e.g., PuTTY, Tera Term).
* **Software/OS Image:**
    * **Ubuntu Server OS** for Orange Pi Zero 3.
    * (Download from the [Orange Pi Zero 3 Support Page](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-Pi-Zero-3.html))
    * An image burning tool (e.g., Balena Etcher, Rufus).

---

## ⚙️ Setup Guide: From Zero to Pirate

### **STEP 1 & 2: Initial OS Setup**

1.  **Get the Orange Pi Zero 3.**
2.  **Burn the OS:** Download the **Ubuntu Server OS** image and burn it onto your microSD card. Follow the instructions in the Orange Pi instruction manual/guide for this process.

### **STEP 3: Serial Connection for First Boot**

Initial configuration requires a serial connection to set up the network.

1.  **Wiring:** Connect the **RX**, **TX**, and **GND** pins of the Orange Pi Zero 3 to your USB to Serial cable.
2.  **PuTTY Setup:**
    * Connect the serial cable to your PC.
    * Open **PuTTY**.
    * Select **Serial** connection type.
    * Enter your USB to Serial cable's **COM Port** number.
    * Set **Baud Rate** to **115200**.
    * Click **Open**. (The window will be blank initially.)
3.  **Power Up:** Power on the Pi using the USB-C cable. You will see the boot process output in the PuTTY terminal. Wait for the login prompt.

### **STEP 4: Network Setup via Serial**

We'll use the serial connection *only* to set up Wi-Fi for better SSH access later.

1.  **Connect to Wi-Fi:** Use the `nmcli` (NetworkManager command-line interface) tool.
    ```bash
    sudo nmcli --ask dev wifi connect <SSID>
    ```
    * *Example:* `sudo nmcli --ask dev wifi connect MY-HOME-WIFI`
    * **Default OS Password:** `orangepi`
    * **Second Password Prompt:** Enter your Wi-Fi password.
    * Wait for the confirmation: `Device 'wlan0' successfully activated...`
2.  **Set Connection Priority (Optional but Recommended):**
    ```bash
    nmcli connection modify "<connection name>" connection.autoconnect-priority <priority number>
    ```
    * *Example:* `nmcli connection modify "MY-HOME-WIFI" connection.autoconnect-priority 10` (Higher number = higher priority)
3.  **Verify Connections:**
    ```bash
    nmcli --fields name,autoconnect-priority connection show
    ```
4.  **Apply Changes:** Restart the NetworkManager service.
    ```bash
    sudo systemctl restart NetworkManager
    ```
5.  **Find IP Address:**
    ```bash
    ifconfig
    ```
    * Look under the `wlan0` section for the **`inet`** address (e.g., `192.168.1.40`).
    * Verify the flags show **`UP,BROADCAST,RUNNING`**.
    * **NOTE:** Save this IP address for the next step! You can now exit the serial connection.

### **STEP 5: SSH Connection and OS Update**

1.  **Connect via SSH:** Open your PC's command prompt or PowerShell.
    * **Normal User:**
        ```bash
        ssh orangepi@<inet ip address>
        ```
        * *Example:* `ssh orangepi@10.0.0.33`
    * **Root User (Admin):**
        ```bash
        ssh root@<inet ip address>
        ```
    * **Password:** `orangepi` (for the `orangepi` user).
2.  **System Update and Upgrade:**
    ```bash
    sudo apt update
    sudo apt upgrade
    ```
    * Type **`Y`** when prompted to continue the upgrade and install the package maintainer's version for configuration options.
3.  **Reboot:**
    ```bash
    sudo reboot now
    ```
    * Wait for the onboard **Green LED** to start flashing, indicating the system is fully booted before attempting to SSH again.

### **STEP 6: Enabling Peripherals (I2C, SPI, UART)**

We need to enable the hardware interfaces required for the project.

#### **Option A: Using `orangepi-config` (Easy Method)**

1.  Open the configuration utility:
    ```bash
    sudo orangepi-config
    ```
2.  Navigate to **System** -> **Hardware**.
3.  Enable the following peripherals:
    * `ph-i2c3`
    * `ph-uart5`
    * `spi1-cs0-spidev` (Note: We will confirm/overwrite this with the manual edit).
4.  Select **Save**, then **Back**.
5.  Select **Reboot** when prompted to apply changes.

#### **Option B: Manual File Edit (Recommended for Reliability)**

This method ensures the correct, non-deprecated SPI setup is used.

1.  Open the environment file:
    ```bash
    sudo nano /boot/orangepiEnv.txt
    ```
2.  Ensure the file contains the following lines (add/modify as needed), keeping everything else *exactly* as it is:
    ```txt
    # ... other settings
    overlays=ph-i2c3 ph-uart5 spi-spidev
    param_spidev_spi_bus=1
    # ... other settings
    ```
    * *Note:* The deprecated `spi1-cs0-cs1-spidev` is replaced by the official `spi-spidev` + `param_spidev_spi_bus=1` syntax for the Zero 3.
    * **I2C3:** Enables I2C on pins 3 (SDA) and 5 (SCL).
    * **UART5:** Enables the extra UART communication interface.
3.  **Save & Exit:** Press **Ctrl+S** then **Ctrl+X**.
4.  **Execute Commands to Ensure Correct Overlays (Optional Cleanup):**
    ```bash
    sudo sed -i '/^overlays=/d' /boot/orangepiEnv.txt
    sudo sed -i '/^param_spidev_/d' /boot/orangepiEnv.txt
    echo "overlays=spi-spidev i2c3" | sudo tee -a /boot/orangepiEnv.txt
    echo "param_spidev_spi_bus=1" | sudo tee -a /boot/orangepiEnv.txt
    ```
5.  **Reboot:**
    ```bash
    sudo reboot now
    ```

#### **Verification**

After logging back in, check for the device nodes:

```bash
ls -l /dev/i2c* /dev/spi*
