# RASPERRY-PI-3B-OLED-MONITOR-3GB-SWAP-MEMORY-
A lightweight OLED system monitor for Raspberry Pi that displays real-time CPU, RAM, swap, temperature, and USB status. Includes 3GB USB-based swap for improved performance on 1GB RAM boards. Auto-starts on boot using systemd.
-----------------------------------------------ANBU-SELVAN-----------------------------------------------------------------------------------------------------
# ANBU\_R3-PI: OLED System Monitor + USB Swap for Raspberry Pi

> A lightweight OLED system monitor for Raspberry Pi that displays real-time CPU, RAM, swap, temperature, and USB status. Includes 3GB USB-based swap for improved performance on 1GB RAM boards. Auto-starts on boot using systemd.

---

## 📆 Features

* 📏 OLED header label: `ANBU_R3-PI`
* 📊 Real-time CPU load
* 📀 RAM and 3GB USB swap usage
* 🌡️ CPU temperature via `vcgencmd`
* 🔌 USB detection
* 🔄 Auto-starts at boot using `systemd`
* 📍 Uses ext4 USB pendrive for swap
* 🧠 Very low memory usage (<3MB)

---

## 🛠️ Hardware Requirements

| Component        | Notes                    |
| ---------------- | ------------------------ |
| Raspberry Pi 3B+ | RPi OS Bookworm or later |
| SSD1306 OLED     | 128x64 via I²C (SCL/SDA) |
| USB Pendrive     | 4GB+ formatted to ext4   |

---![37bc2b40-830f-4da6-87b2-5a40daa509f6](https://github.com/user-attachments/assets/19b1db27-b0d5-4f45-ae3b-f599f2005e3d)
![image](https://github.com/user-attachments/assets/a8f6717f-e653-4fda-b1c0-0045abd6fd39)



## 🔌 OLED Pin Wiring

| OLED Pin | Pi GPIO Pin | Function  |
| -------- | ----------- | --------- |
| VCC      | Pin 1 or 2  | Power     |
| GND      | Pin 6       | Ground    |
| SDA      | Pin 3       | I²C Data  |
| SCL      | Pin 5       | I²C Clock |

---

## 📀 Format USB Pendrive to ext4 and Create Swap

1. **Format the drive**:

```bash
sudo mkfs.ext4 /dev/sda -L SWAPUSB
```

2. **Mount USB**:

```bash
sudo mkdir /mnt/swapusb
sudo blkid /dev/sda
# Copy UUID
```

Edit fstab:

```bash
sudo nano /etc/fstab
# Add:
UUID=xxxx-xxxx /mnt/swapusb ext4 defaults,noatime 0 0
```

3. **Create swap file**:

```bash
sudo dd if=/dev/zero of=/mnt/swapusb/swapfile bs=1M count=3072
sudo chmod 600 /mnt/swapusb/swapfile
sudo mkswap /mnt/swapusb/swapfile
sudo swapon /mnt/swapusb/swapfile
```

4. **Make swap permanent**:

```bash
sudo nano /etc/fstab
# Add:
/mnt/swapusb/swapfile none swap sw 0 0
```

---

## 📁 Install OLED Libraries

```bash
sudo apt update
sudo apt install python3-pip i2c-tools -y
pip3 install --break-system-packages adafruit-circuitpython-ssd1306 pillow
```

Enable I2C:

```bash
sudo raspi-config
# Interface Options → I2C → Enable
```
![Screenshot 2025-07-07 221932](https://github.com/user-attachments/assets/7224e7e2-3577-4280-9d9f-89086248dc98)
![Screenshot 2025-07-09 225014](https://github.com/user-attachments/assets/74dda084-c4f1-4a28-8c76-49a8082c6cf6)
![Screenshot 2025-07-09 225027](https://github.com/user-attachments/assets/00cee1ea-d77f-45c3-8558-e429a2f5155b)


---

## 📄 OLED Monitor Script

Create the file:

```bash
nano /home/anbu/oledenv/oled_monitor-final.py
```

Paste:

```python
import time, os
oled = None
oled_found = os.path.exists("/dev/i2c-1")
if oled_found:
    try:
        import board, busio
        import adafruit_ssd1306
        from PIL import Image, ImageDraw, ImageFont
        i2c = busio.I2C(board.SCL, board.SDA)
        oled = adafruit_ssd1306.SSD1306_I2C(128, 64, i2c)
        font = ImageFont.load_default()
    except:
        oled_found = False

def get_stats():
    cpu = os.getloadavg()[0]
    mem = os.popen("free -m | awk '/Mem/ {print $3\"/\"$2\"MB\"}'").read().strip()
    swap = os.popen("free -m | awk '/Swap/ {print $3\"/\"$2\"MB\"}'").read().strip()
    temp = os.popen("vcgencmd measure_temp").read().strip().replace("temp=", "")
    usb = "Yes" if "sd" in os.popen("lsblk").read() else "No"
    return cpu, mem, swap, temp, usb

def draw_oled(lines):
    if not oled_found: return
    image = Image.new("1", (128, 64))
    draw = ImageDraw.Draw(image)
    draw.rectangle((0, 0, 128, 64), fill=0)
    draw.text((0, 0), "ANBU_R3-PI", font=font, fill=255)
    for i, line in enumerate(lines):
        draw.text((0, (i + 1) * 10), line, font=font, fill=255)
    oled.image(image)
    oled.show()

while True:
    cpu, mem, swap, temp, usb = get_stats()
    lines = [
        f"CPU: {cpu:.2f} Load",
        f"RAM: {mem}",
        f"Swap: {swap}",
        f"Temp: {temp}",
        f"USB: {usb}",
    ]
    print(" | ".join(lines))
    draw_oled(lines)
    time.sleep(6)
```

Make it executable:

```bash
chmod +x /home/anbu/oledenv/oled_monitor-final.py
```

---

## ⚙️ Auto-Start Script at Boot with systemd

Create systemd service:

```bash
sudo nano /etc/systemd/system/oled-monitor.service
```

Paste:

```ini
[Unit]
Description=OLED Monitor Display
After=multi-user.target

[Service]
ExecStart=/usr/bin/python3 /home/anbu/oledenv/oled_monitor-final.py
WorkingDirectory=/home/anbu/oledenv
Restart=always
User=anbu

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable oled-monitor
sudo systemctl start oled-monitor
```

---

## 📷 OLED Output Example

```
ANBU_R3-PI
CPU: 0.35 Load
RAM: 264/925MB
Swap: 124/3072MB
Temp: 47.0'C
USB: Yes
```
![4537e361-e40e-4455-b9db-810abe666eae](https://github.com/user-attachments/assets/aad8ef01-02bb-4ec0-a69c-5d63884202c6)
![dd9f7554-5ad3-4b7b-9995-7c274895e39d](https://github.com/user-attachments/assets/af2ce373-a1cd-4a29-8741-3f04d42b85ae)

---

## 📂 Folder Structure

```
/home/anbu/oledenv/
├── oled_monitor-final.py
/etc/systemd/system/
└── oled-monitor.service
```

---

## 🛠️ Manage Service

```bash
sudo systemctl restart oled-monitor
sudo systemctl stop oled-monitor
sudo systemctl status oled-monitor
```

---

## 🗛️ Uninstall

```bash
sudo systemctl disable oled-monitor
sudo rm /etc/systemd/system/oled-monitor.service
rm /home/anbu/oledenv/oled_monitor-final.py
```

---

## ✨ Author

Made with ❤️ by [Anbu](https://github.com/your-username)

---

## 🗶 License

MIT License
