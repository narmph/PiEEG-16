# PiEEG Quick Start Guide

---

## 1. Install Raspbian on RaspberryPi5 (Any RAM)

Download and flash Raspbian from:
https://www.raspberrypi.com/software/

---

## 2. Connect PiEEG Directly to RaspberryPi5 via GPIO40

Seat the PiEEG board onto the 40-pin GPIO header of the Raspberry Pi 5.

---

## 3. Connect Screen, Keyboard and Mouse

- Monitor → HDMI
- Keyboard → USB
- Mouse → USB
- Power → USB-C power bank or wall adapter

---

## 4. Launch Raspberry Pi and Install Python

Open a terminal and run:

```bash
sudo apt-get install python3
sudo apt-get install idle3
```

---

## 5. Set Up the PiEEG Python Environment

### Enable SPI
SPI must be enabled before the libraries will work:

```bash
sudo raspi-config
# Navigate to: Interface Options → SPI → Enable
```

### Clone the repository

```bash
git clone <repo-url> /home/admin/pieeg/PiEEG-16
cd /home/admin/pieeg/PiEEG-16
```

### Create the virtual environment

```bash
python3 -m venv .venv --prompt pieeg
```

### Install dependencies

```bash
source .venv/bin/activate
pip install -r requirements.txt
```

> **Note:** `gpiod` is pinned to `1.5.4` — the API changed significantly in v2.x, do not upgrade without testing.

### Activate the environment

```bash
source /home/admin/pieeg/PiEEG-16/.venv/bin/activate
```

Your prompt will change to `(pieeg)` when active.

### Deactivate the environment

```bash
deactivate
```

### Run a script without activating

```bash
/home/admin/pieeg/PiEEG-16/.venv/bin/python script.py
```

---

## 6. Launch the Script

```bash
source /home/admin/pieeg/PiEEG-16/.venv/bin/activate
python3 2.Graph_Gpio_D\ _1_5_4.py
```

Script source:
https://github.com/pieeg-club/PiEEG-16/blob/main/GUI/2.Graph_Gpio_D%20_1_5_4.py

---

## 7. LEDs Should Be ON

Once the script is running, the LEDs on the PiEEG board should illuminate, confirming the board is powered and communicating.

---

## 8. Connect Electrodes and Start Recording

Connect the EEG electrodes to the board:
- Channels 1–8: lower connector
- Channels 9–16: upper connector
- **Bias** and **Reference** electrodes as indicated

Then start measuring biosignals using the same script:
https://github.com/pieeg-club/PiEEG-16/blob/main/GUI/2.Graph_Gpio_D%20_1_5_4.py

---

## Repository Notes

- The `.venv/` virtual environment folder is excluded from git (`.gitignore`) — it must be recreated on each device as above.
- `requirements.txt` is committed to the repo and pins all dependencies.
- To update dependencies after adding a new package: `pip freeze > requirements.txt` and commit.

