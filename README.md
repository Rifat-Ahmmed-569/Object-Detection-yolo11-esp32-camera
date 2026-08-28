<div align="center">

# ESP32-CAM + YOLO11 Real-Time Person Detection

**A live camera on a $8 microcontroller. A neural network on your laptop. A pin that goes HIGH when a human walks into frame.**

[![Platform](https://img.shields.io/badge/platform-ESP32--CAM-blue)]()
[![Python](https://img.shields.io/badge/python-3.10%2B-brightgreen)]()
[![Model](https://img.shields.io/badge/model-YOLO11n-orange)]()
[![Inference](https://img.shields.io/badge/inference-CPU%20only-lightgrey)]()
[![License](https://img.shields.io/badge/license-MIT-purple)]()
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-success)]()

</div>

---

> [!IMPORTANT]
> **Placeholder filenames.** The two source files are referred to throughout as
> `YOUR_ESP32_CODE.ino` and `YOUR_YOLO_SCRIPT.py`. Replace them with your real filenames after you
> add your code. Every placeholder is marked **`<PLACEHOLDER>`**. There is a
> [find-and-replace recipe](#replacing-the-placeholders) at the end.

---

## Table of Contents

**Getting started**
- [What This Project Does](#what-this-project-does)
- [How It Works — 30-Second Version](#how-it-works--30-second-version)
- [Features](#features)
- [Hardware Requirements](#hardware-requirements)
- [Software Requirements](#software-requirements)
- [Wiring](#wiring)
- [Repository Structure](#repository-structure)
- [Quick Start — 10 Minutes](#quick-start--10-minutes)

**Full installation**
- [Part 1 — Flashing the ESP32-CAM](#part-1--flashing-the-esp32-cam)
- [Part 2 — Setting Up Your Computer](#part-2--setting-up-your-computer)
- [Part 3 — Running It](#part-3--running-it)

**Understanding it**
- [Architecture — The Nine Steps](#architecture--the-nine-steps)
- [Why Two Servers on Two Ports](#why-two-servers-on-two-ports)
- [The Threading Model](#the-threading-model)
- [Timing — Where the Milliseconds Go](#timing--where-the-milliseconds-go)
- [ESP32 Code Walkthrough](#esp32-code-walkthrough)
- [Python Code Walkthrough](#python-code-walkthrough)
- [YOLO for Absolute Beginners](#yolo-for-absolute-beginners)

**Going further**
- [Configuration Reference](#configuration-reference)
- [Modification Recipes](#modification-recipes)
- [Training YOLO on Your Own Objects](#training-yolo-on-your-own-objects)
- [Performance Tuning](#performance-tuning)

**Reference**
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Glossary](#glossary)
- [FAQ](#faq)
- [Future Improvements](#future-improvements)
- [Replacing the Placeholders](#replacing-the-placeholders)
- [License](#license)

---

## What This Project Does

An **ESP32-CAM** (a tiny camera board) streams live video over Wi-Fi. A **Python program** on your
computer catches that video, runs it through a **YOLO11 neural network**, and asks one question
about every single frame:

> Is there a person in this picture?

When the answer becomes *yes*, Python sends a command back to the ESP32 over Wi-Fi and the ESP32
drives **GPIO13 HIGH** — which can switch an LED, a relay, a buzzer, a door lock, or anything else
you wire to it. When the person leaves, GPIO13 goes **LOW** again.

**Why split the work like this?** The ESP32 is far too small to run a neural network. Your laptop is
far too large to sit on a robot. So the ESP32 does what it is good at (capturing images, driving
pins) and your computer does what *it* is good at (heavy AI math). They talk over Wi-Fi. This split
is called **edge–host architecture**, and a huge number of real robotics products are built this way.

### Who this is written for

- ✅ You have flashed an ESP32 before (you know what "upload sketch" means).
- ❓ You have **never** written Python.
- ❓ You have **never** opened VS Code.
- ❓ You have **never** heard of OpenCV, YOLO, virtual environments, or machine learning.

Every technical term is explained the first time it appears. Nothing is assumed.

---

## How It Works — 30-Second Version

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              ESP32-CAM                                    │
│                                                                           │
│   OV2640 Camera ──► JPEG frame ──► MJPEG server  (port 81)  ──────────┐   │
│                                                                       │   │
│   GPIO13 / GPIO12 ◄── pin driver ◄── control server (port 80) ◄───┐   │   │
└───────────────────────────────────────────────────────────────────┼───┼───┘
                                                                    │   │
                            Wi-Fi (same network)                    │   │
                                                                    │   ▼
┌───────────────────────────────────────────────────────────────────┼───────┐
│                          Your Computer (Python)                   │       │
│                                                                   │       │
│   requests ──► reader thread ──► newest JPEG ──► OpenCV decode ────┼───┐   │
│                                                                   │   │   │
│   HTTP GET /person_on  ◄── debounce logic ◄── YOLO11n inference ◄──┼───┘   │
│   HTTP GET /person_off │                                          │       │
│                        └──────────────────────────────────────────┘       │
└───────────────────────────────────────────────────────────────────────────┘
```

**Two servers, two ports, on purpose.** The video stream never ends — it is one HTTP response that
keeps going forever. If the GPIO commands shared that server, they would queue behind the stream and
never get answered. Video on **port 81**, commands on **port 80** means your LED reacts instantly no
matter how busy the video is.

---

## Features

| Feature | What it means for you |
|---|---|
| 🎥 **Live MJPEG streaming** | Real-time video over plain Wi-Fi — viewable in any browser too |
| 🧠 **YOLO11n object detection** | State-of-the-art detection, small enough for a laptop CPU |
| 🔌 **Zero-GPU requirement** | No graphics card needed |
| ⚡ **Dual-server design** | Video on 81, control on 80 — commands never lag |
| 🧵 **Threaded frame reader** | Always processes the *newest* frame, so video never falls behind |
| 🎛 **Anti-flicker debouncing** | 2 frames to switch ON, 8 to switch OFF — no strobing LED |
| 📉 **State-change-only HTTP** | Commands fire only when the answer *changes*, not 30× per second |
| 📊 **Live FPS counter** | See exactly how fast inference is running |
| 🛡 **Safe shutdown** | GPIO forced LOW on quit, even after a crash |
| 🔄 **Auto-reconnect** | If Wi-Fi drops, the reader retries every 2 seconds |
| 🎯 **Class filtering** | YOLO only looks for people — everything else is skipped, which is faster |

---

## Hardware Requirements

### Essential

| Item | Specification | Notes |
|---|---|---|
| **ESP32-CAM** | AI Thinker module | Must be AI Thinker — the pin map in the code is specific to it |
| **Camera** | OV2640 | Comes attached; check the ribbon cable is seated |
| **USB-to-Serial adapter** | FTDI FT232RL, CP2102, or CH340 | **Set the jumper to 5V**, not 3.3V |
| **Jumper wires** | Female-to-female × 5 | For programming |
| **Power supply** | 5V, **at least 1A** | The single most common cause of failure is weak power |
| **Micro-USB cable** | Data-capable | Many cheap cables are charge-only |

### Output side

| Item | Notes |
|---|---|
| **LED** | Any 5mm LED plus a **220Ω resistor** in series. Never connect an LED without a resistor |
| **Relay module** | Optional — 5V single-channel if you want to switch larger loads |

> [!WARNING]
> **Power is the #1 killer of ESP32-CAM projects.** The 3.3V pin on most FTDI adapters supplies only
> ~50–100mA. The ESP32-CAM draws **250–500mA** while streaming, with brief spikes higher.
> Underpowering causes: brownout resets, `Camera initialization failed`, corrupted frames, or a board
> that reboots the instant you open the stream.
>
> **Fix:** power the board from **5V + GND** using a phone charger or dedicated 5V supply. Use the
> FTDI only for the data lines. **The grounds must be connected together.**

### Computer

| Requirement | Minimum | Recommended |
|---|---|---|
| OS | Windows 10, macOS 11, Ubuntu 20.04 | Any current version |
| RAM | 4 GB | 8 GB+ |
| CPU | Any 64-bit dual-core | Quad-core or better |
| Storage | 3 GB free | For Python, PyTorch, and the model |
| GPU | **Not required** | An NVIDIA GPU gives ~5–10× more FPS |
| Internet | Required **once** | To download the YOLO model on first run |

---

## Software Requirements

| Software | Version | Purpose |
|---|---|---|
| **Arduino IDE** | 2.x | Flashing the ESP32 |
| **ESP32 board package** | 2.0.x or 3.x | Adds ESP32 support to Arduino |
| **Python** | 3.10 – 3.12 | Runs the detection program |
| **VS Code** | Latest | Code editor |
| **ultralytics** | ≥ 8.3.0 | The YOLO11 library |
| **opencv-python** | ≥ 4.8 | Decodes and displays images |
| **numpy** | ≥ 1.24 | Fast number arrays |
| **requests** | ≥ 2.31 | Talks HTTP to the ESP32 |

> [!NOTE]
> **Python 3.13 warning.** PyTorch wheels (which `ultralytics` depends on) can lag behind the newest
> Python release. **Use Python 3.10, 3.11, or 3.12** for a painless install.

---

## Wiring

### A. Programming mode (uploading code)

| ESP32-CAM pin | → | Connect to |
|---|---|---|
| `5V` | → | FTDI `VCC` (jumper on **5V**) or external 5V |
| `GND` | → | FTDI `GND` |
| `U0T` (TX, GPIO1) | → | FTDI `RX` |
| `U0R` (RX, GPIO3) | → | FTDI `TX` |
| `IO0` | → | **`GND`** ← the flash-mode jumper |

> **TX goes to RX and RX goes to TX. They cross over.** Think of it like a phone call: your mouth
> (TX) talks to their ear (RX). Connect TX→TX and you get `Failed to connect to ESP32`.

**Why `IO0` → `GND`?** GPIO0 is a **strapping pin** — a pin the chip reads once, at the exact moment
it powers on, to decide how to behave.

- GPIO0 **HIGH** (or floating) at boot → "run the program in my flash memory."
- GPIO0 **LOW** at boot → "wait, someone is about to send me a new program."

**Upload sequence:**

```
1. Connect IO0 → GND
2. Press and release RST
3. Click Upload in Arduino IDE
4. Wait for "Hard resetting via RTS pin..."
5. Remove the IO0 → GND jumper
6. Press RST once — the program now runs
```

### B. Run mode (normal operation)

| ESP32-CAM pin | Connect to |
|---|---|
| `5V` | 5V supply, ≥1A |
| `GND` | Supply GND, plus LED cathode |
| `GPIO13` | → 220Ω resistor → LED anode |
| `GPIO12` | Leave **unconnected** |
| `IO0` | Leave **unconnected** |

```
   GPIO13 ──[ 220Ω ]──►|── GND
                       LED
                   (long leg toward the resistor)
```

> [!CAUTION]
> **GPIO12 is a strapping pin.** On the ESP32, GPIO12 (`MTDI`) is read *during boot* to decide the
> internal flash voltage.
>
> - GPIO12 **LOW** at reset → 3.3V flash (correct)
> - GPIO12 **HIGH** at reset → 1.8V flash → **the board fails to boot or reboot-loops**
>
> The firmware always drives GPIO12 LOW *after* boot, which is safe — but **never attach a pull-up
> resistor, a lit LED, or a relay input that idles HIGH to GPIO12.**
> Need a second output? Use **GPIO14, GPIO15, or GPIO2**.

### Pins you cannot use on an ESP32-CAM

| Pin(s) | Reason |
|---|---|
| 0, 5, 18, 19, 21, 22, 23, 25, 26, 27, 32, 34, 35, 36, 39 | OV2640 camera |
| 16 | PSRAM |
| 1, 3 | Serial TX/RX |
| 4 | Onboard white flash LED (blindingly bright) |
| 33 | Onboard red LED (**inverted**: LOW = on) |
| **12** | Strapping pin — flash voltage select |
| 13, 14, 15, 2 | SD card — free if you are not using the card slot |

---

## Repository Structure

```
esp32-cam-yolo11-person-detection/
│
├── README.md                    ← you are here (everything is in this file)
├── LICENSE                      ← MIT
├── requirements.txt             ← Python dependency list
├── .gitignore                   ← keeps venv/, *.pt, and secrets out of Git
│
├── esp32/
│   ├── YOUR_ESP32_CODE.ino      ← <PLACEHOLDER> your Arduino sketch
│   ├── secrets.example.h        ← Wi-Fi credential template (copy → secrets.h)
│   └── secrets.h                ← NEVER COMMITTED — your real credentials
│
├── python/
│   ├── YOUR_YOLO_SCRIPT.py      ← <PLACEHOLDER> your detection script
│   └── diagnose.py              ← optional environment/network checker
│
├── images/
│   ├── screenshots/
│   ├── diagrams/
│   └── wiring/
│
└── examples/                    ← optional variants and extensions
```

### What must never be committed

| Path | Why |
|---|---|
| `esp32/secrets.h` | Your Wi-Fi password. Git history is permanent |
| `venv/` | Hundreds of MB, machine-specific, reproducible from `requirements.txt` |
| `*.pt`, `*.onnx` | Model weights — 5 MB+, re-downloadable |
| `runs/` | Training output — can be gigabytes |
| `__pycache__/` | Compiled bytecode |

> [!WARNING]
> **`.gitignore` only affects files Git is not already tracking.** If you committed `secrets.h`
> before adding it to `.gitignore`, adding the line changes nothing. Run:
> ```bash
> git rm --cached esp32/secrets.h
> git commit -m "Remove secrets from tracking"
> ```
> And if it was ever pushed publicly: **change your Wi-Fi password.**

---

## Quick Start — 10 Minutes

> [!TIP]
> If any step fails, do **not** guess. Jump to [Troubleshooting](#troubleshooting) and find your
> exact symptom.

### Step 1 — Flash the ESP32-CAM

1. Open `esp32/YOUR_ESP32_CODE.ino` **`<PLACEHOLDER>`** in Arduino IDE.
2. Copy `esp32/secrets.example.h` → `esp32/secrets.h` and fill in your Wi-Fi name and password.
3. **Tools → Board → ESP32 Arduino → AI Thinker ESP32-CAM**
4. **Tools → Partition Scheme → Huge APP (3MB No OTA/1MB SPIFFS)** ← *required, the sketch will not fit otherwise*
5. Wire for programming (`IO0` → `GND`), press **RST**, click **Upload**.
6. Remove the `IO0` jumper, press **RST**.

### Step 2 — Find your ESP32's IP address

**Tools → Serial Monitor**, baud **115200**, press **RST**:

```
Connecting to Wi-Fi....
ESP32 IP: 192.168.0.172
GPIO control server started on port 80
Camera stream server started on port 81
```

**Write that IP down.** It is the single most important number in this project.

### Step 3 — Verify the stream in a browser

Open `http://<YOUR_ESP32_IP>:81/stream` in Chrome or Firefox. You should see live video.

If you do, **the entire hardware half is finished.** ✅

> [!IMPORTANT]
> **Close that browser tab before running Python.** The ESP32 serves one stream client at a time. A
> forgotten tab is the single most common reason "the browser works but Python doesn't."

### Step 4 — Set up Python

```bash
cd esp32-cam-yolo11-person-detection

python -m venv venv

# Windows (PowerShell):
venv\Scripts\activate
# macOS / Linux:
source venv/bin/activate

pip install -r requirements.txt
```

Your prompt must now start with `(venv)`. If it does not, the environment is not active and
everything will install to the wrong place.

### Step 5 — Run it

Open `python/YOUR_YOLO_SCRIPT.py` **`<PLACEHOLDER>`** and change one line:

```python
ESP_IP = "192.168.0.172"   # ← replace with YOUR ESP32's IP from Step 2
```

Then:

```bash
python python/YOUR_YOLO_SCRIPT.py
```

**On first run only**, YOLO downloads `yolo11n.pt` (~5 MB).

A window opens. Walk in front of the camera. A box appears around you, the banner turns red, and
**GPIO13 goes HIGH**. Press **`Q`** to quit.

---

# Part 1 — Flashing the ESP32-CAM

## 1.1 Install Arduino IDE

**IDE** stands for *Integrated Development Environment* — a program you write code in. The Arduino
IDE is a text editor plus a **compiler** (translates your C++ into machine instructions the ESP32
can execute) plus an **uploader** (pushes those instructions onto the chip).

1. Go to **https://www.arduino.cc/en/software**
2. Download **Arduino IDE 2.x**, install with defaults.
3. Open it. You should see `void setup()` and `void loop()`.

> **macOS:** if it says "cannot be opened because the developer cannot be verified", right-click →
> **Open** → **Open**. Once only.
> **Linux:** `chmod +x arduino-ide_*.AppImage`

## 1.2 Add the ESP32 board package

Arduino IDE ships knowing about Arduino boards only. The ESP32 is made by Espressif, so you must
teach the IDE about it.

1. **File → Preferences** (macOS: **Arduino IDE → Settings**)
2. In **Additional boards manager URLs**, paste:

   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```

3. **OK** → **Tools → Board → Boards Manager…**
4. Search `esp32`, find **esp32 by Espressif Systems**, click **Install**.

> This downloads ~250 MB and takes several minutes. If it fails halfway, click Install again — it
> resumes.

**Verify:** **Tools → Board** now has an **esp32** submenu containing **AI Thinker ESP32-CAM**.

## 1.3 Wire for programming

The ESP32-CAM has **no USB port** — it omits the USB-to-serial chip to save space and cost. You
supply it yourself. See [Wiring → Programming mode](#a-programming-mode-uploading-code).

## 1.4 Add your Wi-Fi credentials safely

**Why not just type them into the code?** Because Git remembers everything. Put your Wi-Fi password
in the `.ino` and push to GitHub, and it is in your repository's history *forever* — deleting the
line later does not remove it. Bots scrape GitHub for exactly this.

**The safe pattern:**

1. Copy `secrets.example.h` → `secrets.h`:

   ```bash
   cp esp32/secrets.example.h esp32/secrets.h
   ```

2. Fill in your real values in `secrets.h`:

   ```cpp
   #pragma once
   #define SECRET_WIFI_SSID      "MyHomeNetwork"
   #define SECRET_WIFI_PASSWORD  "my-actual-password"
   ```

3. At the top of your `.ino`:

   ```cpp
   #include "secrets.h"

   const char *WIFI_SSID     = SECRET_WIFI_SSID;
   const char *WIFI_PASSWORD = SECRET_WIFI_PASSWORD;
   ```

4. `secrets.h` is in `.gitignore`, so Git never sees it. `secrets.example.h` *is* committed so
   others know what to create.

> [!CAUTION]
> **If you have already pushed real credentials to a public repository, change your Wi-Fi password
> now.** Deleting the file does not remove it from history.

### Two Wi-Fi gotchas that waste hours

1. **Trailing spaces in the SSID.** `"MyWiFi "` and `"MyWiFi"` are different networks as far as the
   ESP32 is concerned. Check character by character.
2. **5 GHz networks do not work.** The ESP32 has a **2.4 GHz-only** radio. If your router broadcasts
   separate names like `MyWiFi` and `MyWiFi_5G`, you **must** use the 2.4 GHz one. Your laptop can
   stay on 5 GHz; they just need the same router/subnet.

## 1.5 Board settings

Set **all** of these under **Tools**:

| Setting | Value | Why |
|---|---|---|
| **Board** | `AI Thinker ESP32-CAM` | Correct pin definitions and memory layout |
| **Partition Scheme** | `Huge APP (3MB No OTA/1MB SPIFFS)` | **Critical.** The camera libraries are large; the default partition is too small |
| **CPU Frequency** | `240MHz (WiFi/BT)` | Full speed — needed for streaming |
| **Flash Frequency** | `80MHz` | Default |
| **Flash Mode** | `QIO` | Default |
| **Core Debug Level** | `None` | Set to `Info` only when debugging |
| **Port** | Your adapter's port | Windows `COM3`… · macOS `/dev/cu.usbserial-*` · Linux `/dev/ttyUSB0` |
| **Upload Speed** | `115200` | Start here. `921600` is faster but less reliable |

> **No port appears?**
> - Windows: install your adapter's driver — [CP2102](https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers), [CH340](https://www.wch-ic.com/downloads/CH341SER_ZIP.html), [FTDI](https://ftdichip.com/drivers/vcp-drivers/)
> - Linux: `sudo usermod -a -G dialout $USER`, then **log out and back in**
> - Any OS: try a different USB cable. Charge-only cables are extremely common

## 1.6 Upload

Follow the [upload sequence](#a-programming-mode-uploading-code) exactly.

**Success looks like:**

```
Sketch uses 1,234,567 bytes (39%) of program storage space.
Writing at 0x00010000... (100%)
Leaving...
Hard resetting via RTS pin...
```

**Failure and what it means:**

| Message | Cause | Fix |
|---|---|---|
| `Failed to connect to ESP32: Timed out waiting for packet header` | Not in flash mode | Re-check `IO0`→`GND`, press RST, retry |
| `A fatal error occurred: Invalid head of packet` | TX/RX swapped, or bad power | Cross TX↔RX; add external 5V |
| `text section exceeds available space in board` | Wrong partition scheme | Set **Huge APP (3MB No OTA)** |
| `Serial port COM3 not found` | Board disconnected mid-upload | Replug, reselect the port |
| Upload succeeds but nothing runs | Jumper still attached | Remove `IO0`→`GND`, press RST |

## 1.7 Find the IP address

An **IP address** is your device's phone number on the local network.

1. **Tools → Serial Monitor**
2. Baud rate (bottom-right) → **115200**

   > **Baud rate** = signal changes per second. Both sides must agree or you get gibberish like
   > `⸮⸮⸮@⸮`. The sketch calls `Serial.begin(115200)`.

3. Press **RST** on the ESP32-CAM.

```
GPIO13 LOW: no person
Connecting to Wi-Fi.....
ESP32 IP: 192.168.0.172
GPIO control server started on port 80
Camera stream server started on port 81
```

> **Your IP will be different.** Use whatever your board prints.
>
> **The IP can change.** Routers lease addresses temporarily. If detection suddenly stops working
> weeks later, check the Serial Monitor again. To make it permanent, set a **DHCP reservation** in
> your router's admin page (look for "static lease" or "address reservation").

**If you see only dots forever:**

```
Connecting to Wi-Fi.................................
```

Causes, in order of likelihood:

1. Wrong password (typos, stray spaces)
2. Network is 5 GHz only
3. Weak signal — move closer to the router
4. Router has MAC filtering enabled
5. Insufficient power — the Wi-Fi radio draws a current spike when connecting

## 1.8 Verify the stream

Open `http://YOUR_ESP32_IP:81/stream` in a browser **on the same Wi-Fi network**.

You should see live video. ✅ **The hardware half is done.**

---

# Part 2 — Setting Up Your Computer

## 2.1 Install Python

Python is a **programming language**. The **Python interpreter** reads your instructions and
executes them line by line. The entire AI ecosystem — YOLO, PyTorch, OpenCV, NumPy — is Python-first.

Download **Python 3.12** (or 3.10 / 3.11) from **https://www.python.org/downloads/**

> [!WARNING]
> **Do not install Python 3.13+ yet.** `ultralytics` depends on PyTorch, and PyTorch releases wheels
> for the newest Python version months late. You will hit
> `ERROR: Could not find a version that satisfies the requirement torch`.

### Windows — the one checkbox that matters

On the first installer screen:

```
☑ Add python.exe to PATH
```

**Tick it.** Otherwise every command fails with
`'python' is not recognized as an internal or external command`.

> **PATH** is the list of folders your OS searches when you type a command. If Python's folder is not
> on that list, the system has no idea where `python` lives.
>
> Forgot? Re-run the installer → **Modify** → **Next** → tick **Add Python to environment variables**
> → **Install**.

### macOS

Download the `.pkg` and run it. macOS ships an old system Python — always use `python3`.

### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install python3 python3-pip python3-venv
```

`python3-venv` is a separate package and easy to forget. Without it, creating a virtual environment
fails.

### Verify

Open a **new** terminal:

```bash
python --version      # expect: Python 3.12.4
pip --version         # expect: pip 24.0 from ... (python 3.12)
```

> **You must open a NEW terminal after installing.** PATH changes only apply to terminals opened
> afterwards. This catches everyone once.

If `python` is not found but `python3` is, use `python3` everywhere below.
If pip is missing: `python -m ensurepip --upgrade`

## 2.2 Install VS Code

You *could* write Python in Notepad, but VS Code gives you syntax highlighting, autocomplete, error
squiggles before you run, an integrated terminal, and a debugger.

1. **https://code.visualstudio.com/** → download → install with defaults.
2. **Windows:** tick **"Add to PATH"** and **"Open with Code"** during install.

## 2.3 Install the VS Code extensions

`Ctrl+Shift+X` (`Cmd+Shift+X` on macOS) opens Extensions.

| Extension | Publisher | What it gives you |
|---|---|---|
| **Python** | Microsoft | Running/debugging, environment selection |
| **Pylance** | Microsoft | Fast autocomplete, type checking, jump-to-definition |

Installing **Python** usually pulls in **Pylance** automatically.

**Optional but genuinely useful:**

| Extension | Why |
|---|---|
| **Ruff** (Astral) | Instant linting — catches unused imports as you type |
| **Error Lens** | Shows errors inline instead of only on hover |
| **Arduino Community Edition** | Edit and upload `.ino` files without leaving VS Code |

## 2.4 Create a virtual environment

### What is it, and why

Imagine every Python project on your computer shares one toolbox. Project A needs hammer 1.0.
Project B needs hammer 2.0. Installing 2.0 breaks Project A. This is **dependency hell**, and it is
real and constant.

A **virtual environment** ("venv") is a private toolbox for one project — a folder containing its own
copy of Python and its own packages. Nothing inside can break anything outside.

| Without a venv | With a venv |
|---|---|
| Installing a package can break an unrelated project | Projects are fully isolated |
| Impossible to know which packages a project needs | `pip freeze` gives the exact list |
| "It works on my machine" | Anyone can reproduce your setup |
| Uninstalling is guesswork | Delete the folder — completely clean |

### Create and activate

In VS Code: **File → Open Folder…** → select the repo → press `` Ctrl+` `` for a terminal in the
right place.

```bash
python -m venv venv
```

| Part | Meaning |
|---|---|
| `python` | Run the Python interpreter |
| `-m venv` | Run the built-in module named `venv` |
| `venv` (second) | Name the created folder `venv` |

**Activate:**

```powershell
venv\Scripts\activate          # Windows PowerShell
```
```cmd
venv\Scripts\activate.bat      # Windows cmd
```
```bash
source venv/bin/activate       # macOS / Linux
```

**Verify** — your prompt now shows `(venv)`:

```
(venv) C:\Users\You\esp32-cam-yolo11-person-detection>
(venv) you@laptop:~/esp32-cam-yolo11-person-detection$
```

**That prefix is your proof.** Without it, every `pip install` goes to the wrong place.

> [!WARNING]
> **PowerShell says `running scripts is disabled on this system`?**
> Run PowerShell **as Administrator** once:
> ```powershell
> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
> ```
> Type `Y`. Then reopen a normal PowerShell and activate again.

> [!IMPORTANT]
> **You must activate in every new terminal.** It is not permanent. If a command that worked
> yesterday fails today, check for `(venv)` first.

Leave the environment with `deactivate`.

## 2.5 Install the packages

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

> This downloads **1–2 GB** and takes 3–10 minutes. The bulk is **PyTorch**. Do not interrupt it.

### What each package does

**`ultralytics`** — provides YOLO11. Gives you the `YOLO` class, downloads pretrained weights
automatically, runs inference, draws boxes. Pulls in `torch` and `torchvision` — PyTorch is the
engine performing the actual neural-network math.
*Without it:* no object detection at all. This is the AI.

**`opencv-python`** (imported as `cv2`) — *Open Source Computer Vision Library*, started at Intel in
1999, now the de-facto standard for image processing.

| Function | Job |
|---|---|
| `cv2.imdecode()` | Turn raw JPEG bytes into a pixel grid |
| `cv2.imshow()` | Open a window and display it |
| `cv2.putText()` | Draw the status banner and FPS counter |
| `cv2.waitKey()` | Check for a keypress, and pump the window's event loop |

*Without it:* frames stay as unreadable byte strings and nothing appears on screen.

> [!CAUTION]
> **Never install `opencv-python-headless` for this project.** The headless build has no GUI support
> — `cv2.imshow()` silently does nothing or crashes. It exists for servers with no display. If both
> are installed they conflict:
> ```bash
> pip uninstall opencv-python-headless opencv-python -y
> pip install opencv-python
> ```

**`numpy`** — *Numerical Python*. Provides the `ndarray`, a fast multi-dimensional array. An image is
just numbers: a 320×240 colour image is a 320 × 240 × 3 grid of values from 0 to 255 (blue, green,
red per pixel). NumPy manipulates that grid at C speed — roughly **50–100× faster** than pure Python.
Here: `np.frombuffer(jpg, dtype=np.uint8)` converts raw JPEG bytes into an array OpenCV can decode.

**`requests`** — an HTTP client library. **HTTP** is the protocol browsers use — the same one your
ESP32's servers speak. Here it (1) opens a long-lived streaming connection to `:81/stream`, and
(2) sends short `GET` requests to `/person_on` and `/person_off`.

### Verify

```bash
pip list
python -c "import cv2, numpy, requests; from ultralytics import YOLO; print('All packages OK')"
```

Any `ModuleNotFoundError` means that package did not install — most often because the venv was not
active.

## 2.6 Point VS Code at your venv

Skip this and VS Code underlines your imports in red even though the code runs fine.

1. `Ctrl+Shift+P` (`Cmd+Shift+P`) → **Command Palette**
2. Type `Python: Select Interpreter` → Enter
3. Choose the entry containing your project path and **('venv')**:

   ```
   Python 3.12.4 ('venv': venv)  ./venv/bin/python
   ```

4. The bottom-right status bar now shows `3.12.4 ('venv')`.

> [!TIP]
> **Make it automatic.** Create `.vscode/settings.json`:
> ```json
> {
>   "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python",
>   "python.terminal.activateEnvironment": true
> }
> ```
> On Windows use `"${workspaceFolder}/venv/Scripts/python.exe"`.

---

# Part 3 — Running It

## 3.1 Set the IP address

```python
ESP_IP = "192.168.0.172"
```

**This is the number one cause of "it doesn't work".** Everything below it is built from it:

```python
STREAM_URL     = f"http://{ESP_IP}:81/stream"
PERSON_ON_URL  = f"http://{ESP_IP}/person_on"
PERSON_OFF_URL = f"http://{ESP_IP}/person_off"
```

Get `ESP_IP` right and all three are right.

## 3.2 Pre-flight checklist

- [ ] ESP32-CAM powered, Serial Monitor shows an IP
- [ ] `IO0`→`GND` jumper **removed**
- [ ] Every browser tab pointing at `:81/stream` **closed**
- [ ] Computer and ESP32 on the **same Wi-Fi network**
- [ ] Terminal prompt shows `(venv)`
- [ ] `ESP_IP` matches the Serial Monitor
- [ ] Internet available (first run only)

## 3.3 First run

```bash
python python/YOUR_YOLO_SCRIPT.py
```

```
Loading YOLO model...
Downloading https://github.com/ultralytics/assets/releases/download/v8.3.0/yolo11n.pt ...
100%|███████████████████████████████| 5.35M/5.35M [00:03<00:00, 1.72MB/s]
ESP32: PERSON OFF -> GPIO12/13 LOW
Connecting to ESP32 camera stream...
ESP32 camera stream connected.
Press Q in the video window to stop.
```

> The download happens **once**. Later runs load from disk and need no internet.

## 3.4 Verify

| # | Test | Expected |
|---|---|---|
| 1 | Point at an empty room | Green: `NO PERSON - GPIO12/13 LOW`. LED off |
| 2 | Walk into frame | Box within ~0.5s. Red: `PERSON CONFIRMED - GPIO13 HIGH`. **LED on** |
| 3 | Walk out of frame | After ~1s, back to green. **LED off** |
| 4 | Press `Q` | Window closes, `Program closed safely.`, LED off |

The terminal prints a line each time the state flips:

```
YOLO person in frame: True
ESP32: PERSON ON -> GPIO13 HIGH
YOLO person in frame: False
ESP32: PERSON OFF -> GPIO12/13 LOW
```

### If the box appears but the LED does not light

The AI half works; the GPIO half does not. Isolate it:

```bash
curl http://YOUR_ESP32_IP/person_on
```

- **Returns `{"ok":true,...}` and the LED lights** → the ESP32 is fine; a firewall or proxy is
  blocking Python
- **Returns JSON but no light** → wiring. Check LED polarity (long leg = anode = positive) and the
  220Ω resistor
- **Times out** → wrong IP, or the ESP32 crashed. Check the Serial Monitor

## 3.5 Controls

| Key | Action |
|---|---|
| **`Q`** | Quit cleanly — GPIO forced LOW, window closed, sockets released |
| **`Ctrl+C`** | Emergency stop — also forces GPIO LOW |

> Always quit with **`Q`**. Closing the window with ✕ leaves the process running in the background,
> holding the stream open, and the next run will fail to connect.

---

# Architecture — The Nine Steps

## Why the work is split across two devices

| | ESP32-CAM | Your computer |
|---|---|---|
| RAM | ~520 KB internal + 4 MB PSRAM | 8,000,000 KB |
| CPU | 240 MHz, 2 cores, no vector unit | 3 GHz+, 4–16 cores, AVX/NEON |
| Floating-point math | Software-emulated, very slow | Hardware, billions of ops/sec |
| Power draw | 0.5 W | 15–60 W |
| Can it sit on a robot arm? | ✅ | ❌ |
| Can it run YOLO11n? | ❌ | ✅ |

YOLO11n performs roughly **6.5 billion floating-point operations per image**. On the ESP32 that
would take minutes per frame, if the model even fit in memory — which it does not.

- **ESP32 = the senses and the muscles.** It sees (camera) and it acts (GPIO).
- **Computer = the brain.** It thinks.
- **Wi-Fi = the nervous system.** It carries signals both ways.

---

### Step 1 — Light hits the sensor

*ESP32-CAM, OV2640 sensor · continuous*

Millions of photodiodes convert photons into electrical charge, measured and turned into numbers.
The `esp_camera` driver has configured the sensor to output **QVGA** (320 × 240), compress to
**JPEG** on the sensor's own hardware, at quality level 15.

> **Why compress on the sensor?** A raw 320×240 RGB frame is 320 × 240 × 3 = **230,400 bytes**. As
> JPEG at quality 15 it is roughly **10,000–20,000 bytes** — about **15× smaller**. Wi-Fi cannot
> carry 230 KB per frame at 20 FPS; it can easily carry 15 KB.

**If this fails:** `esp_camera_fb_get()` returns null, the sketch prints `Camera capture failed`.
Usually power or a loose ribbon cable.

### Step 2 — A JPEG frame lands in a buffer

```c
camera_fb_t *frame = esp_camera_fb_get();
// frame->buf  → pointer to the JPEG bytes
// frame->len  → how many bytes
```

Every JPEG begins with the two bytes `FF D8` (*Start of Image*, SOI) and ends with `FF D9` (*End of
Image*, EOI). **Remember those four hex digits** — Python uses them in Step 4 to find frame
boundaries.

With `fb_count = 1` there is exactly one buffer. It must be handed back with
`esp_camera_fb_return(frame)` before the next capture, or the camera stalls.

### Step 3 — The frame is pushed onto the MJPEG stream

*ESP32 stream server, port 81 · `multipart/x-mixed-replace`*

**MJPEG** (Motion JPEG) is the simplest possible video format: a sequence of complete JPEG images
with a text separator between them. No inter-frame compression at all — every frame is standalone.

| | MJPEG | H.264 |
|---|---|---|
| Bandwidth | High | Low |
| Encoder complexity | Trivial | Very high |
| Latency | Very low | Higher (needs buffering) |
| Any frame decodable alone | ✅ | ❌ (needs keyframes) |
| Runs on an ESP32 | ✅ | ❌ |

The response is one HTTP reply that **never ends**:

```
Content-Type: multipart/x-mixed-replace;boundary=frame
```

Then, forever:

```
--frame
Content-Type: image/jpeg
Content-Length: 14238
<blank line>
[FF D8 ...14238 bytes of JPEG... FF D9]
<CRLF>
--frame
Content-Type: image/jpeg
...
```

**If this fails:** the loop exits when `httpd_resp_send_chunk()` errors — which is exactly what
happens when the client disconnects. That is not a bug; it is how the handler knows to stop.

### Step 4 — Python receives and reassembles frames

*Computer, `camera_reader()` background thread*

This is subtler than it looks. TCP delivers a **byte stream**, not messages. The 4096-byte chunks
that arrive do **not** line up with frame boundaries — one chunk might contain the tail of frame 7,
all of frame 8, and the head of frame 9.

```python
jpeg_buffer += chunk                                     # accumulate

jpg_start = jpeg_buffer.find(b"\xff\xd8")                # find SOI
jpg_end   = jpeg_buffer.find(b"\xff\xd9", jpg_start + 2) # find EOI after it

if jpg_start != -1 and jpg_end != -1:
    jpg = jpeg_buffer[jpg_start : jpg_end + 2]           # one complete JPEG
    jpeg_buffer = jpeg_buffer[jpg_end + 2:]              # keep the remainder
```

It never parses the `--frame` boundary text or `Content-Length`. It simply scans for JPEG markers.
This is more forgiving of malformed headers, which cheap camera firmware produces often.

```python
frame = cv2.imdecode(np.frombuffer(jpg, dtype=np.uint8), cv2.IMREAD_COLOR)
```

| Call | Produces |
|---|---|
| `np.frombuffer(jpg, np.uint8)` | A flat array of 14,238 raw bytes — still compressed |
| `cv2.imdecode(..., IMREAD_COLOR)` | A `240 × 320 × 3` array of pixel values — a real image |

> [!NOTE]
> **The array is BGR, not RGB.** OpenCV stores colour channels Blue-Green-Red for historical
> reasons. This surprises everyone once. Ultralytics expects BGR too, so nothing needs converting
> here — but if you pass a frame to a library expecting RGB, call
> `cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)`.

**The critical line:**

```python
with frame_lock:
    latest_frame = frame       # OVERWRITE. The previous frame is thrown away.
    latest_frame_id += 1
```

**Old frames are deliberately discarded.** If the camera produces 20 FPS and YOLO handles 6, a queue
would grow by 14 frames per second forever and the display would fall further behind reality every
second. By keeping only the newest frame, latency stays constant no matter how slow inference is.
You lose frames; you do not lose *time*.

### Step 5 — YOLO runs inference

*Computer, main loop · typically 50–200 ms on CPU*

```python
results = model(
    current_frame,
    imgsz=320,
    conf=0.25,
    classes=[0],       # person only
    verbose=False,
    device="cpu"
)
```

Internally, in order:

1. **Preprocess** — the 320×240 frame is letterboxed to 320×320, pixel values scaled from 0–255 to
   0.0–1.0, rearranged from `(H, W, C)` to `(1, C, H, W)`.
2. **Forward pass** — the tensor flows through YOLO11n's convolutional layers. Early layers find
   edges and textures; later layers combine those into shapes and eventually object concepts.
3. **Head output** — thousands of candidate boxes, each with coordinates, an objectness score, and
   80 class scores.
4. **Filter by confidence** — anything below 0.25 is dropped.
5. **Filter by class** — only class 0 survives, because of `classes=[0]`.
6. **NMS** (Non-Maximum Suppression) — overlapping boxes describing the same person are merged,
   keeping the highest-scoring one.

### Step 6 — A raw yes/no is produced

```python
boxes = results[0].boxes
person_detected_this_frame = boxes is not None and len(boxes) > 0
```

The entire detection decision is "did any box survive?" Because YOLO was already told to report only
class 0, any box at all means a person.

This is the **raw** answer. Honest but jittery: a person turning sideways, walking behind a chair,
or moving through a shadow can vanish for a frame or two.

### Step 7 — Debouncing turns the raw answer into a stable one

```
Raw YOLO:     F  T  T  T  F  T  T  F  F  F  F  F  F  F  F  T  T
seen count:   0  1  2  3  0  1  2  0  0  0  0  0  0  0  0  1  2
missing:      1  0  0  0  1  0  0  1  2  3  4  5  6  7  8  0  0
Stable out:   F  F  T  T  T  T  T  T  T  T  T  T  T  T  F  F  T
                    ▲                                   ▲     ▲
                    2 hits → ON            8 misses → OFF    ON again
```

**Asymmetric on purpose:**

| | Threshold | Reasoning |
|---|---|---|
| Turn **ON** | 2 frames | Fast response. A single false positive is filtered, but a real person triggers within ~0.3s |
| Turn **OFF** | 8 frames | Slow release. Brief occlusions, a turned head, or a dropped frame will not switch the output off |

This is **hysteresis** — the same principle a thermostat uses so it does not click on and off every
few seconds around the setpoint.

### Step 8 — An HTTP command is sent (only if the answer changed)

```python
if not force and person_present == last_person_state:
    return                      # nothing changed — send nothing
```

At 10 FPS with someone standing still, a naive implementation would fire **600 HTTP requests per
minute** all saying the same thing. That floods the ESP32's small TCP stack and can crash it.

Instead a request is sent **only on a transition**. Standing still for an hour = zero requests.

```python
response = control_session.get(url, timeout=2)
response.raise_for_status()
last_person_state = person_present   # updated ONLY after success
```

Note the ordering: `last_person_state` is updated *after* the ESP32 confirms. If the request fails,
the stored state stays stale, so the next frame retries automatically. A small but genuinely
important robustness detail.

### Step 9 — The ESP32 drives the pin

```c
digitalWrite(GPIO_PERSON_LED, personDetected ? HIGH : LOW);  // pin 13
digitalWrite(GPIO_ALWAYS_LOW, LOW);                          // pin 12
```

`digitalWrite(13, HIGH)` connects the pin internally to 3.3V. Current flows through your resistor
and LED to ground. **The LED lights.**

```json
{"ok":true,"gpio13":"HIGH","gpio12":"LOW"}
```

**Total end-to-end latency: roughly 150–400 ms** on a typical laptop.

---

## Why Two Servers on Two Ports

### The naive approach (broken)

```
Port 80: /stream, /person_on, /person_off      ← all on one server
```

The ESP32's `esp_http_server` handles a limited number of concurrent connections, and the stream
handler **never returns** — it loops forever sending frames.

```
Time →
Client A: GET /stream ────────────────────────────────────────────► (forever)
Client B: GET /person_on  ░░░░░░ queued ░░░░░░░░░░░░░ never served
```

### This project's approach (correct)

```
Port 80  → controlServer  : /person_on, /person_off   (fast, returns immediately)
Port 81  → streamServer   : /stream                    (slow, never returns)
```

```c
controlConfig.server_port = 80;   controlConfig.ctrl_port = 32768;
streamConfig.server_port  = 81;   streamConfig.ctrl_port  = 32769;
```

> [!IMPORTANT]
> **The `ctrl_port` values must also differ.** `ctrl_port` is an internal UDP port the HTTP server
> uses to talk to itself for shutdown signalling. Both default to 32768. If you leave them identical,
> the **second `httpd_start()` silently fails** and you get `Camera stream server failed` in the
> Serial Monitor. This is a genuinely obscure trap.

Result: video can be saturating the link and a GPIO command still completes in ~5 ms.

---

## The Threading Model

A **thread** is an independent line of execution inside one program. This project uses two.

```
┌─── Main thread ────────────────────┐   ┌─── Reader thread (daemon) ────────┐
│                                    │   │                                    │
│ load YOLO model                    │   │ open streaming HTTP connection     │
│ start reader thread ───────────────┼──►│                                    │
│                                    │   │ loop forever:                      │
│ loop forever:                      │   │   read 4096 bytes                  │
│   read latest_frame (under lock)   │◄──┼──  find FFD8 ... FFD9              │
│   if same id as last → skip        │   │   decode JPEG                      │
│   run YOLO                         │   │   write latest_frame (under lock)  │
│   debounce                         │   │   latest_frame_id += 1             │
│   maybe send HTTP                  │   │                                    │
│   draw + imshow + waitKey          │   │ on network error:                  │
│                                    │   │   sleep 2s, reconnect              │
└────────────────────────────────────┘   └────────────────────────────────────┘
```

**Why threads are necessary.** Reading from a network socket **blocks** — the code stops and waits.
YOLO inference also blocks, for 50–200 ms. In a single thread these alternate, and every millisecond
spent on inference is a millisecond not spent draining the socket. The TCP buffer fills, frames back
up, and latency grows without bound.

**The lock.**

```python
frame_lock = threading.Lock()

with frame_lock:
    latest_frame = frame
```

Two threads writing and reading the same variable at the same moment can produce a **race
condition** — the main thread could read a half-updated value. A `Lock` guarantees only one thread is
inside the protected block at a time. It is held for as short a time as possible (just the
assignment / the `.copy()`), because anything longer would make the threads wait on each other and
defeat the purpose.

**The frame ID.**

```python
if current_frame is None or current_frame_id == last_processed_frame_id:
    continue      # nothing new; do not waste inference on a duplicate
```

Without this, a fast computer runs YOLO repeatedly on the *same* frame while waiting for the next —
burning CPU and inflating the FPS counter with meaningless work.

**`daemon=True`.**

```python
reader_thread = threading.Thread(target=camera_reader, daemon=True)
```

A **daemon thread** does not keep the program alive. When the main thread exits, daemons are killed
immediately. Without this flag, pressing `Q` would close the window but the process would hang
forever waiting for the reader's infinite loop.

---

## Timing — Where the Milliseconds Go

Typical mid-range laptop, `imgsz=320`, CPU inference:

| Stage | Time | Notes |
|---|---|---|
| Camera capture + JPEG encode | 30–60 ms | Fixed by the ESP32 |
| Wi-Fi transfer | 5–30 ms | Depends on signal strength |
| JPEG decode (`cv2.imdecode`) | 1–3 ms | Very fast |
| **YOLO inference** | **50–200 ms** | **The bottleneck** |
| Debounce logic | <0.1 ms | Trivial arithmetic |
| HTTP command (on change only) | 5–20 ms | Only on transitions |
| Draw + display | 5–15 ms | `plot()` + `imshow()` |
| **Total per frame** | **~100–300 ms** | ≈ 3–10 FPS |

Plus the debounce delay: 2 frames to turn on. At 8 FPS that adds ~250 ms.
**Total person-enters-frame → LED-on: roughly 250–500 ms.**

### Data formats along the way

| Stage | Format | Approx. size |
|---|---|---|
| Photons on the sensor | analog charge | — |
| Sensor output | raw pixel data | 230,400 bytes |
| After hardware JPEG encode | compressed JPEG | ~15,000 bytes |
| In `jpeg_buffer` | Python `bytes` | ~15,000 bytes |
| After `np.frombuffer` | `uint8` array, shape `(15000,)` | 15,000 bytes |
| After `cv2.imdecode` | `uint8` array, shape `(240, 320, 3)` | 230,400 bytes |
| YOLO preprocessed | `float32` tensor, shape `(1, 3, 320, 320)` | 1,228,800 bytes |
| YOLO output | boxes: `(N, 6)` — x1,y1,x2,y2,conf,cls | a few hundred bytes |
| Decision | one Python `bool` | 1 bit of meaning |
| On the wire back | `GET /person_on HTTP/1.1` | ~60 bytes |
| Final effect | 3.3V on a pin | — |

**230,400 bytes of pixels compressed down to a single yes/no.** That is what object detection is.

### State machine

```
                   ┌─────────────────────────────────────┐
                   │                                     │
                   ▼                                     │
        ╔═══════════════════╗                            │
        ║  NO PERSON        ║   2 consecutive             │
        ║  GPIO13 = LOW     ║   detections                │
        ║  banner: green    ║ ────────────────────────►┐  │
        ╚═══════════════════╝                          │  │
                   ▲                                   ▼  │
                   │                      ╔═══════════════════╗
                   │   8 consecutive      ║  PERSON CONFIRMED ║
                   └──────────────────────║  GPIO13 = HIGH    ║
                       misses             ║  banner: red      ║
                                          ╚═══════════════════╝

  Special transitions:
  • Program start → forced to NO PERSON (send_person_command(False, force=True))
  • Program exit  → forced to NO PERSON (finally block, even after a crash)
  • HTTP failure  → last_person_state unchanged → automatic retry next frame
```

The `force=True` calls at start and exit matter: without them, a crashed program could leave a relay
energised or a lock disengaged indefinitely. **Always fail to the safe state.**

### Failure modes and what recovers them

| Failure | Detected by | Recovery |
|---|---|---|
| Wi-Fi drops mid-stream | `requests.RequestException` in reader | Sleep 2s, reconnect. Automatic, indefinite |
| ESP32 reboots | Same as above | Automatic once the ESP32 rejoins Wi-Fi |
| One corrupt JPEG | `cv2.imdecode` returns `None` | Frame skipped, loop continues |
| A control request fails | `raise_for_status()` raises | `last_person_state` untouched → retried next frame |
| Camera capture fails | `esp_camera_fb_get()` returns null | Handler returns `ESP_FAIL`; client reconnects |
| YOLO throws unexpectedly | `except Exception` in main | Exits **via `finally`**, forcing GPIO LOW |
| User presses Ctrl+C | `KeyboardInterrupt` | Same clean shutdown path |
| ESP32 IP changed | Connection timeout | ❌ **Not automatic.** Update `ESP_IP`, or set a DHCP reservation |

The one gap is a changed IP address. Fixing that properly means adding mDNS (`esp32cam.local`).

---

# ESP32 Code Walkthrough

Every block gets three questions answered: **Why this exists** · **What happens internally** ·
**How this affects the project**.

## 1. The libraries

```cpp
#include <Arduino.h>
#include <WiFi.h>
#include "esp_camera.h"
#include "esp_http_server.h"
```

`#include` tells the compiler: "before compiling my code, paste in the contents of this other file."

| Library | Provides | From |
|---|---|---|
| `Arduino.h` | `pinMode()`, `digitalWrite()`, `Serial`, `delay()`, `HIGH`/`LOW` | Arduino core |
| `WiFi.h` | `WiFi.begin()`, `WiFi.status()`, `WiFi.localIP()` | ESP32 Arduino core |
| `esp_camera.h` | `esp_camera_init()`, `esp_camera_fb_get()`, `camera_config_t` | Espressif camera driver |
| `esp_http_server.h` | `httpd_start()`, `httpd_register_uri_handler()`, `httpd_resp_send_chunk()` | ESP-IDF HTTP server |

`<angle brackets>` search the system/library include paths; `"quotes"` search the sketch folder first.

The last two come from **ESP-IDF**, Espressif's native C framework. The Arduino core for ESP32 is a
friendly layer built *on top of* ESP-IDF, which is why you can mix Arduino-style `digitalWrite()`
with raw IDF-style `httpd_start()` in the same file. This is normal and supported.

**If you see `esp_camera.h: No such file or directory`,** you have selected a non-ESP32 board.

## 2. Wi-Fi settings

```cpp
const char *WIFI_SSID     = SECRET_WIFI_SSID;
const char *WIFI_PASSWORD = SECRET_WIFI_PASSWORD;
```

`const char *` means "a pointer to characters that must not be modified" — the C way of storing text.
`const` lets the compiler place the string in flash memory instead of precious RAM.

> [!CAUTION]
> **Never write literal credentials here.** If they go public with the repo, your Wi-Fi password is
> permanently in Git history. Bots scan GitHub for exactly this pattern within minutes of a push.
> Use `secrets.h` as shown in [§1.4](#14-add-your-wi-fi-credentials-safely).

## 3. GPIO settings

```cpp
constexpr int GPIO_PERSON_LED = 13;
constexpr int GPIO_ALWAYS_LOW = 12;
```

**GPIO** = *General Purpose Input/Output* — a physical pin the chip can drive to 3.3V (HIGH) or 0V
(LOW), or read a voltage from.

`constexpr` means "compute this at compile time" — zero runtime cost, unlike a variable, and
type-checked, unlike `#define`.

Naming them instead of scattering `13` and `12` through the code means changing a pin is a one-line
edit, and every use site explains itself.

See the [GPIO12 strapping-pin warning](#b-run-mode-normal-operation) and the
[full pin table](#pins-you-cannot-use-on-an-esp32-cam).

## 4. The camera pin map

```cpp
#define PWDN_GPIO_NUM     32     #define Y9_GPIO_NUM  35
#define RESET_GPIO_NUM    -1     #define Y8_GPIO_NUM  34
#define XCLK_GPIO_NUM      0     #define Y7_GPIO_NUM  39
#define SIOD_GPIO_NUM     26     #define Y6_GPIO_NUM  36
#define SIOC_GPIO_NUM     27     #define Y5_GPIO_NUM  21
                                 #define Y4_GPIO_NUM  19
#define VSYNC_GPIO_NUM    25     #define Y3_GPIO_NUM  18
#define HREF_GPIO_NUM     23     #define Y2_GPIO_NUM   5
#define PCLK_GPIO_NUM     22
```

The OV2640 is soldered to the board with fixed traces. The software has to be told which chip pin
each camera signal is wired to. **This map is specific to the AI Thinker module** — a TTGO or M5Stack
board has different numbers and this map will produce `Camera init failed`.

| Signal | Full name | Role |
|---|---|---|
| `PWDN` | Power Down | Pull HIGH to put the sensor to sleep |
| `RESET` | Reset | Hardware reset. **`-1` means "not connected"** — the AI Thinker ties it internally |
| `XCLK` | External Clock | The 20 MHz clock the ESP32 *generates* for the sensor. The camera has no oscillator of its own |
| `SIOD` / `SIOC` | SCCB Data / Clock | The I²C-like control bus used to configure the sensor |
| `Y2`–`Y9` | Pixel data bus | 8 parallel wires carrying one byte of pixel data per clock tick |
| `VSYNC` | Vertical Sync | Pulses once per complete frame |
| `HREF` | Horizontal Reference | HIGH while a row of valid pixels is being sent |
| `PCLK` | Pixel Clock | One tick per pixel byte — tells the ESP32 exactly when to sample the bus |

At 20 MHz XCLK, pixel data streams in far too fast for the CPU to read pin by pin. The ESP32 uses its
**I²S peripheral in camera mode** plus **DMA** (Direct Memory Access) to shovel bytes straight into
RAM without CPU involvement. This is why `esp_camera_fb_get()` feels instant — the frame was already
captured in the background.

> Switching boards? Copy the correct block from `camera_pins.h` in
> **File → Examples → ESP32 → Camera → CameraWebServer**.

## 5. The two server handles

```cpp
httpd_handle_t controlServer = nullptr;
httpd_handle_t streamServer  = nullptr;
```

A **handle** is an opaque token identifying a running object. You do not look inside it; you pass it
back to library functions that need to know *which* server you mean.

`nullptr` is C++'s "points to nothing" value. Initialising to `nullptr` means that if `httpd_start()`
fails, the handle is provably invalid rather than containing garbage.

## 6. `setPersonOutput()` — driving the pin

```cpp
void setPersonOutput(bool personDetected) {
  digitalWrite(GPIO_PERSON_LED, personDetected ? HIGH : LOW);
  digitalWrite(GPIO_ALWAYS_LOW, LOW);

  if (personDetected) Serial.println("GPIO13 HIGH: person detected");
  else                Serial.println("GPIO13 LOW: no person");
}
```

**Why it exists.** Both request handlers do the same physical work. One function means the behaviour
is defined in exactly one place — add a buzzer later and you add it once.

`personDetected ? HIGH : LOW` is the **ternary operator** — compact `if/else` that produces a value.

`digitalWrite(pin, HIGH)` sets a bit in the GPIO output register; the pin's driver connects it to the
3.3V rail. Physically this takes nanoseconds.

GPIO12 is re-driven LOW on **every** call. Slightly redundant, but a cheap guarantee: if electrical
noise ever flipped that pin, the next command corrects it. On a strapping pin, belt-and-braces is
justified.

**If you remove it:**
- Remove the `digitalWrite` calls → nothing physical happens, ever. The API still returns success,
  which makes debugging maddening.
- Remove the `Serial.println` calls → code works identically, but you lose all visibility. Keep them.

## 7. `sendJson()` — replying to Python

```cpp
void sendJson(httpd_req_t *request, const char *json) {
  httpd_resp_set_type(request, "application/json");
  httpd_resp_set_hdr(request, "Access-Control-Allow-Origin", "*");
  httpd_resp_sendstr(request, json);
}
```

Every HTTP request needs a response. Without one, the client waits until it times out. A tiny JSON
reply lets Python's `raise_for_status()` confirm the command landed.

| Line | Effect |
|---|---|
| `httpd_resp_set_type(..., "application/json")` | Sets `Content-Type` so the client knows how to parse the body |
| `httpd_resp_set_hdr(..., "Access-Control-Allow-Origin", "*")` | The **CORS** header. Allows JavaScript on any page to call this endpoint |
| `httpd_resp_sendstr(...)` | Sends the body and completes the response |

**CORS** = *Cross-Origin Resource Sharing*. Browsers block a page served from one domain from calling
a different domain unless that server explicitly permits it. `*` means "anyone may call me."

Python does not read the JSON body — only the status code. But the CORS header means you can later
build a browser dashboard that toggles GPIO with a button, with no extra firmware work.

**If you remove `httpd_resp_sendstr`:** Python hangs for its 2-second timeout on every command. GPIO
still switches, but throughput collapses.

## 8. `personOnHandler()` / `personOffHandler()`

```cpp
esp_err_t personOnHandler(httpd_req_t *request) {
  setPersonOutput(true);
  sendJson(request, "{\"ok\":true,\"gpio13\":\"HIGH\",\"gpio12\":\"LOW\"}");
  return ESP_OK;
}

esp_err_t personOffHandler(httpd_req_t *request) {
  setPersonOutput(false);
  sendJson(request, "{\"ok\":true,\"gpio13\":\"LOW\",\"gpio12\":\"LOW\"}");
  return ESP_OK;
}
```

These are **URI handlers** — callbacks the HTTP server invokes when a matching request arrives.

**The full sequence** when Python calls `http://192.168.0.172/person_on`:

```
1. TCP connection opens on port 80
2. Bytes arrive:  GET /person_on HTTP/1.1\r\nHost: 192.168.0.172\r\n\r\n
3. httpd parses the request line and extracts the URI "/person_on"
4. httpd searches its registered URI table for a match
5. Match found → personOnHandler(request) is called
6. setPersonOutput(true)  → GPIO13 goes HIGH        ← the actual work
7. sendJson(...)          → response written to the socket
8. return ESP_OK          → httpd keeps the connection alive for reuse
```

Steps 6–8 take well under a millisecond.

**The escaped quotes.** In C++, `"` ends a string, so a literal quote inside one must be written
`\"`. What actually goes over the wire is `{"ok":true}`.

**The return value.**

| Return | Meaning |
|---|---|
| `ESP_OK` | Handled successfully; keep the connection open (HTTP keep-alive) |
| `ESP_FAIL` | Something went wrong; **close the socket** |

Returning `ESP_OK` is what lets Python's `requests.Session` reuse one TCP connection for every
command — meaningfully faster than reconnecting each time.

**Test them without Python:**

```bash
curl http://192.168.0.172/person_on
curl http://192.168.0.172/person_off
```

This is the fastest way to separate "the AI is broken" from "the wiring is broken."

**Adding your own endpoint:**

```cpp
esp_err_t statusHandler(httpd_req_t *request) {
  char json[128];
  snprintf(json, sizeof(json),
           "{\"uptime_ms\":%lu,\"rssi\":%d,\"heap\":%u}",
           millis(), WiFi.RSSI(), ESP.getFreeHeap());
  sendJson(request, json);
  return ESP_OK;
}
```

Register it in `startHttpServers()` the same way as the others. `/status` now reports uptime, Wi-Fi
signal strength, and free memory — genuinely useful for debugging.

## 9. `streamHandler()` — the video engine

The most complex function in the file. Take it in pieces.

### 9.1 Setting the content type

```cpp
esp_err_t result = httpd_resp_set_type(
  request, "multipart/x-mixed-replace;boundary=frame");
```

This single header turns an ordinary HTTP response into a video stream.
`multipart/x-mixed-replace` tells the client: *"I will send a sequence of parts. Each new part
replaces the previous one."* `boundary=frame` declares that the literal text `--frame` separates them.

**Remove it** and the browser downloads a file of concatenated binary garbage instead of showing
video. Python's marker-scanning happens to still work; a browser will not.

### 9.2 The infinite loop

```cpp
while (result == ESP_OK) {
  camera_fb_t *frame = esp_camera_fb_get();
  if (!frame) { Serial.println("Camera capture failed"); return ESP_FAIL; }
  ...
}
```

Video is not a single response — it is an endless one. The handler intentionally never returns while
the client is connected.

```c
typedef struct {
    uint8_t  *buf;       // pointer to the JPEG bytes
    size_t    len;       // number of bytes
    size_t    width;     // 320
    size_t    height;    // 240
    pixformat_t format;  // PIXFORMAT_JPEG
    struct timeval timestamp;
} camera_fb_t;
```

`fb` = **frame buffer**. The DMA engine filled it in the background; this call just hands you the
pointer.

**How the loop ends.** `result` is reassigned by every `httpd_resp_send_chunk()` call. When the client
disconnects, the socket write fails, `result` becomes non-`ESP_OK`, and the loop exits naturally.
That is how the ESP32 knows you closed Python.

### 9.3 Building the part header

```cpp
char header[100];

int headerLength = snprintf(
  header, sizeof(header),
  "--frame\r\n"
  "Content-Type: image/jpeg\r\n"
  "Content-Length: %u\r\n\r\n",
  frame->len
);
```

- `char header[100]` reserves 100 bytes on the stack.
- `snprintf` formats into that buffer and — crucially — **will not write past 100 bytes**. Plain
  `sprintf` has no such limit and is a classic buffer-overflow bug. Always use `snprintf`.
- `%u` is substituted with `frame->len` as an unsigned integer.
- `\r\n` is CRLF. **HTTP requires CRLF, not just `\n`.**
- The **double** `\r\n\r\n` marks the boundary between headers and body. Mandatory in HTTP.
- Adjacent string literals in C++ are concatenated at compile time, which is why the format string
  can be split across four lines for readability.

### 9.4 Sending three chunks

```cpp
result = httpd_resp_send_chunk(request, header, headerLength);

if (result == ESP_OK)
  result = httpd_resp_send_chunk(request,
             reinterpret_cast<const char *>(frame->buf), frame->len);

if (result == ESP_OK)
  result = httpd_resp_send_chunk(request, "\r\n", 2);
```

Each frame is transmitted as three pieces: header, JPEG data, trailing CRLF.

**Chunked transfer encoding** is the HTTP mechanism for sending a body of unknown total length. Since
the stream never ends, its length is unknowable — chunking is the only option.

`reinterpret_cast<const char *>` converts a `uint8_t*` (byte pointer) to a `char*` because that is
what the API signature demands. The bytes are unchanged; only the compiler's type view changes. It is
C++'s explicit, searchable way of saying "I know what I'm doing here."

Each send is guarded by `if (result == ESP_OK)` so that once a write fails, the remaining writes are
skipped and the loop exits promptly instead of hammering a dead socket.

### 9.5 Returning the buffer — the critical line

```cpp
esp_camera_fb_return(frame);
```

> [!CAUTION]
> **This is the most important single line in the file.**
>
> The camera has a small fixed pool of frame buffers (`fb_count = 1`, so exactly one). Until you hand
> the buffer back, the driver has nowhere to put the next frame.
>
> **Forget this call and the stream delivers exactly one frame and then freezes forever.** It is the
> single most common ESP32-CAM bug, and it produces a symptom — "the first frame shows, then
> nothing" — that looks like a network problem.

Note it sits **outside** the `if (result == ESP_OK)` guards, so the buffer is returned even when a
send fails. That is deliberate and correct.

### 9.6 The complete function

```cpp
esp_err_t streamHandler(httpd_req_t *request) {
  esp_err_t result = httpd_resp_set_type(
    request, "multipart/x-mixed-replace;boundary=frame");

  while (result == ESP_OK) {
    camera_fb_t *frame = esp_camera_fb_get();
    if (!frame) { Serial.println("Camera capture failed"); return ESP_FAIL; }

    char header[100];
    int headerLength = snprintf(header, sizeof(header),
      "--frame\r\nContent-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n",
      frame->len);

    result = httpd_resp_send_chunk(request, header, headerLength);
    if (result == ESP_OK)
      result = httpd_resp_send_chunk(request,
                 reinterpret_cast<const char *>(frame->buf), frame->len);
    if (result == ESP_OK)
      result = httpd_resp_send_chunk(request, "\r\n", 2);

    esp_camera_fb_return(frame);      // ← always
  }
  return result;
}
```

This function determines your maximum frame rate. At QVGA with quality 15 the ESP32 sustains roughly
15–25 FPS on good Wi-Fi. Raise the resolution and it drops sharply.

## 10. `startHttpServers()`

```cpp
httpd_config_t controlConfig = HTTPD_DEFAULT_CONFIG();
controlConfig.server_port = 80;
controlConfig.ctrl_port   = 32768;
```

`HTTPD_DEFAULT_CONFIG()` is a macro producing a struct pre-filled with sane defaults (stack size,
task priority, max URI handlers, socket limits). You override only what matters.

| Field | Value | Meaning |
|---|---|---|
| `server_port` | 80 | The TCP port clients connect to. 80 is HTTP's default, so no `:80` needed in URLs |
| `ctrl_port` | 32768 | An **internal** UDP port the server uses to signal itself |

**Registering URI handlers:**

```cpp
httpd_uri_t personOnRoute = {};
personOnRoute.uri     = "/person_on";
personOnRoute.method  = HTTP_GET;
personOnRoute.handler = personOnHandler;
```

`= {}` **zero-initialises the whole struct.** This matters: `httpd_uri_t` also has a `user_ctx` field,
and leaving it as uninitialised stack garbage can crash the server. Always zero-init.

| Field | Purpose |
|---|---|
| `.uri` | The path to match, exactly. `/person_on` will not match `/person_on/` |
| `.method` | Which HTTP verb. `HTTP_GET` — the same verb a browser uses |
| `.handler` | A **function pointer** — the address of the code to run on a match |

A function pointer is just the memory address where a function's instructions begin. Assigning
`personOnHandler` (no parentheses) stores its address rather than calling it.

**Starting and checking:**

```cpp
if (httpd_start(&controlServer, &controlConfig) == ESP_OK) {
  httpd_register_uri_handler(controlServer, &personOnRoute);
  httpd_register_uri_handler(controlServer, &personOffRoute);
  Serial.println("GPIO control server started on port 80");
} else {
  Serial.println("GPIO control server failed");
}
```

`httpd_start()` creates a FreeRTOS task, opens a listening socket, and writes the handle into
`controlServer`. The `&` passes the *address* so the function can modify it — C's way of returning a
second value.

Handlers can only be registered **after** the server exists, hence the nesting inside the `if`.

**Why check the return value?** Startup can fail — out of memory, port in use, duplicate `ctrl_port`.
Silent failure leaves you debugging Python for an hour over a firmware problem.

The stream server is identical in structure: `server_port = 81`, `ctrl_port = 32769`,
`uri = "/stream"`.

## 11. `setup()` — the boot sequence

Runs exactly once, immediately after reset. Order matters.

### 11.1 Serial first

```cpp
Serial.begin(115200);
```

**Why first?** So every subsequent step can report success or failure. If camera init fails and
Serial has not started, you get no message at all — just a dead board.

### 11.2 Pins to a safe state

```cpp
pinMode(GPIO_PERSON_LED, OUTPUT);
pinMode(GPIO_ALWAYS_LOW, OUTPUT);
setPersonOutput(false);
```

`pinMode(pin, OUTPUT)` switches the pin's driver on. Before this call the pin **floats** — its
voltage is undefined and can drift, which on a relay input can mean random clicking.

**Why before Wi-Fi and the camera?** Because those steps can take seconds or hang. **Always establish
the safe output state first.** With a relay wired here, this ordering is the difference between a
brief flicker and several seconds of an energised load on every reboot.

### 11.3 Camera configuration

```cpp
camera_config_t cameraConfig = {};     // zero-fill so no field contains garbage

cameraConfig.ledc_channel = LEDC_CHANNEL_0;
cameraConfig.ledc_timer   = LEDC_TIMER_0;
```

**LEDC** is the ESP32's PWM peripheral, normally used for dimming LEDs. Here it is repurposed to
generate the 20 MHz `XCLK` the camera needs. The camera has no crystal of its own — the ESP32 is its
heartbeat.

> If you use `analogWrite()` or `ledcWrite()` elsewhere, avoid channel 0 and timer 0 or you will
> disturb the camera clock.

```cpp
cameraConfig.pin_d0 = Y2_GPIO_NUM;   // ... through d7 = Y9
```

Note the deliberate off-by-one naming: the driver calls them `d0`–`d7` while the OV2640 datasheet
calls them `Y2`–`Y9`. Same eight wires.

```cpp
cameraConfig.xclk_freq_hz = 20000000;   // 20 MHz
cameraConfig.pixel_format = PIXFORMAT_JPEG;
cameraConfig.frame_size   = FRAMESIZE_QVGA;   // 320 x 240
cameraConfig.jpeg_quality = 15;
cameraConfig.fb_count     = 1;
```

| Setting | Value | Trade-off |
|---|---|---|
| `xclk_freq_hz` | 20 MHz | The OV2640's standard rate. Lowering to 10 MHz sometimes fixes unstable boards at the cost of frame rate |
| `pixel_format` | `PIXFORMAT_JPEG` | **Compression happens on the sensor.** `PIXFORMAT_RGB565` would produce 150 KB per frame instead of ~15 KB |
| `frame_size` | `FRAMESIZE_QVGA` | Matches `YOLO_IMAGE_SIZE=320` on the Python side, so nothing is wasted resizing |
| `jpeg_quality` | 15 | **Counter-intuitive: lower = better.** Range 10–63 |
| `fb_count` | 1 | One buffer. Set to 2 **only** if PSRAM is present — double-buffering roughly doubles FPS |

**Frame size options:**

| Constant | Pixels | Notes |
|---|---|---|
| `FRAMESIZE_QQVGA` | 160×120 | Fastest, too small for reliable detection |
| **`FRAMESIZE_QVGA`** | **320×240** | **Used here. The sweet spot** |
| `FRAMESIZE_VGA` | 640×480 | Better detail, roughly half the FPS |
| `FRAMESIZE_SVGA` | 800×600 | Needs PSRAM |
| `FRAMESIZE_UXGA` | 1600×1200 | Full 2 MP. Far too slow for streaming |

### 11.4 Initialising the camera

```cpp
if (esp_camera_init(&cameraConfig) != ESP_OK) {
  Serial.println("Camera initialization failed");
  while (true) { delay(1000); }
}
```

**Why halt on failure?** Everything downstream depends on the camera. Continuing would give you a
board that connects to Wi-Fi, serves a stream URL, and returns nothing — far more confusing than an
honest stop.

`while (true) { delay(1000); }` is a deliberate infinite loop. `delay()` yields to FreeRTOS so the
watchdog stays happy and the board does not reboot-loop.

| Cause | Fix |
|---|---|
| Insufficient power (the #1 cause) | 5V @ ≥1A external supply |
| Ribbon cable not fully seated | Lift the connector latch, reinsert, close it |
| Wrong board selected | Tools → Board → **AI Thinker ESP32-CAM** |
| Wrong pin map for your board | Copy the correct map from `camera_pins.h` |
| Dead camera module | Try another OV2640 |

### 11.5 Connecting to Wi-Fi

```cpp
WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
Serial.print("Connecting to Wi-Fi");

while (WiFi.status() != WL_CONNECTED) {
  delay(500);
  Serial.print(".");
}
```

`WiFi.begin()` returns **immediately** — it only starts the process. The loop polls until
`WL_CONNECTED`, printing a dot every 500 ms.

> [!WARNING]
> **This loop has no timeout.** With wrong credentials the board prints dots forever with no other
> symptom. A production improvement:
>
> ```cpp
> unsigned long start = millis();
> while (WiFi.status() != WL_CONNECTED) {
>   delay(500);
>   Serial.print(".");
>   if (millis() - start > 20000) {          // 20 seconds
>     Serial.println("\nWi-Fi failed. Restarting...");
>     ESP.restart();
>   }
> }
> ```

### 11.6 Printing the IP address

```cpp
Serial.print("ESP32 IP: ");
Serial.println(WiFi.localIP());
```

`WiFi.localIP()` returns the address your router assigned via DHCP. **This is the number you copy
into `ESP_IP` in the Python script.**

### 11.7 🟡 A real bug to watch for

```cpp
Serial.println("Camera stream:");
Serial.println("http://192.168.0.172:81/stream");   // ← hardcoded!
```

**The problem.** If your router hands out a different address — which it will, on a different network
or after a lease expires — the Serial Monitor confidently prints the wrong URL while
`WiFi.localIP()` two lines above printed the right one. That contradiction has cost people hours.

**The fix:**

```cpp
String ip = WiFi.localIP().toString();

Serial.println();
Serial.println("Camera stream:  http://" + ip + ":81/stream");
Serial.println("LED ON test:    http://" + ip + "/person_on");
Serial.println("LED OFF test:   http://" + ip + "/person_off");
```

## 12. `loop()` — the empty loop

```cpp
void loop() {
  // Both HTTP servers run automatically.
}
```

Arduino requires a `loop()` to exist. It is empty because the ESP32 runs **FreeRTOS**, a real-time
operating system with a preemptive scheduler. `httpd_start()` created background *tasks* the
scheduler runs independently of `loop()`. On a classic Arduino Uno there is no scheduler and
everything must be driven from `loop()`; on the ESP32 that is not true.

`loop()` still runs thousands of times per second doing nothing. The Arduino core calls `yield()`
between iterations, letting the scheduler switch to the Wi-Fi and HTTP tasks.

**Do not put blocking code here.** A `delay(1000)` will not stop the servers (separate tasks) but it
holds the loop task's stack and can starve other work on core 1.

**Good uses:**

```cpp
void loop() {
  // Auto-reconnect if Wi-Fi drops
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("Wi-Fi lost. Reconnecting...");
    WiFi.reconnect();
    delay(5000);
  }

  // Heartbeat every 30 seconds
  static unsigned long lastBeat = 0;
  if (millis() - lastBeat > 30000) {
    lastBeat = millis();
    Serial.printf("Uptime %lus | Heap %u | RSSI %d\n",
                  millis() / 1000, ESP.getFreeHeap(), WiFi.RSSI());
  }
}
```

That `millis()`-based pattern is **non-blocking** — it checks the clock instead of stopping. Prefer
it to `delay()` in anything that must stay responsive.

## 13. Known issues and suggested improvements

| # | Issue | Severity | Fix |
|---|---|---|---|
| 1 | Wi-Fi credentials committed in source | 🔴 High | Move to `secrets.h`, add to `.gitignore` |
| 2 | Hardcoded IP in the Serial output | 🟡 Medium | Use `WiFi.localIP().toString()` |
| 3 | Wi-Fi connect loop has no timeout | 🟡 Medium | Add a 20s timeout + `ESP.restart()` |
| 4 | No auto-reconnect if Wi-Fi drops | 🟡 Medium | Add the `WiFi.reconnect()` block to `loop()` |
| 5 | Control endpoints have no authentication | 🟡 Medium | Require a token: `/person_on?key=SECRET` |
| 6 | PSRAM is never detected or used | 🟢 Low | `if (psramFound()) { fb_count = 2; }` for ~2× FPS |
| 7 | No `/status` endpoint | 🟢 Low | Add one; extremely useful for debugging |
| 8 | Trailing space in the SSID literal | 🟢 Low | Verify it is intentional |
| 9 | No watchdog on the stream handler | 🟢 Low | A stuck camera could hang the stream task |

**Detect PSRAM and double-buffer:**

```cpp
if (psramFound()) {
  cameraConfig.frame_size   = FRAMESIZE_VGA;
  cameraConfig.jpeg_quality = 12;
  cameraConfig.fb_count     = 2;
  Serial.println("PSRAM found: VGA, double-buffered");
} else {
  cameraConfig.frame_size   = FRAMESIZE_QVGA;
  cameraConfig.jpeg_quality = 15;
  cameraConfig.fb_count     = 1;
  Serial.println("No PSRAM: QVGA, single buffer");
}
```

**Add a shared-secret check:**

```cpp
bool isAuthorised(httpd_req_t *req) {
  char query[64];
  if (httpd_req_get_url_query_str(req, query, sizeof(query)) != ESP_OK) return false;
  char key[32];
  if (httpd_query_key_value(query, "key", key, sizeof(key)) != ESP_OK) return false;
  return strcmp(key, SECRET_API_KEY) == 0;
}
```

Guard each handler with it. Python's URLs become `.../person_on?key=YOUR_KEY`. Not real security over
plain HTTP, but it stops casual tampering on a shared network.

## Firmware quick reference

| Function | Runs when | Purpose |
|---|---|---|
| `setup()` | Once at boot | Serial → pins → camera → Wi-Fi → servers |
| `loop()` | Forever after setup | Empty; servers run as FreeRTOS tasks |
| `setPersonOutput(bool)` | On each control request | Drives GPIO13, forces GPIO12 LOW |
| `sendJson()` | On each control request | Writes the JSON response |
| `personOnHandler()` | `GET /person_on` (port 80) | GPIO13 HIGH |
| `personOffHandler()` | `GET /person_off` (port 80) | GPIO13 LOW |
| `streamHandler()` | `GET /stream` (port 81) | Infinite MJPEG loop |
| `startHttpServers()` | Once, from `setup()` | Creates both servers, registers routes |

## API reference

| Method | Endpoint | Port | Response | Effect |
|---|---|---|---|---|
| `GET` | `/person_on` | 80 | `{"ok":true,"gpio13":"HIGH","gpio12":"LOW"}` | GPIO13 HIGH |
| `GET` | `/person_off` | 80 | `{"ok":true,"gpio13":"LOW","gpio12":"LOW"}` | GPIO13 LOW |
| `GET` | `/stream` | 81 | `multipart/x-mixed-replace` | Endless MJPEG |

---

# Python Code Walkthrough

Every block gets three questions answered: **What it does** · **Why it exists** ·
**What happens if removed**.

## Block 0 — Python crash course for C++ people

If you have only written Arduino C++, here is the whole translation table:

| Concept | C++ | Python |
|---|---|---|
| Variable declaration | `int x = 5;` | `x = 5` |
| Statement end | `;` | newline |
| Blocks | `{ }` | **indentation** (4 spaces) |
| Comment | `// text` | `# text` |
| Function | `void f(int a) { }` | `def f(a):` |
| Boolean | `true` / `false` | `True` / `False` |
| Null | `nullptr` | `None` |
| Print | `Serial.println(x)` | `print(x)` |
| String format | `sprintf(buf, "%d", n)` | `f"{n}"` |
| For loop | `for (int i=0;i<10;i++)` | `for i in range(10):` |
| Array | `int a[10];` | `a = []` (grows freely) |
| Include | `#include <lib.h>` | `import lib` |

**Indentation is syntax in Python.** Getting it wrong is a compile error, not a style issue.

**f-strings** are worth learning immediately:

```python
name = "ESP32"
print(f"Connecting to {name} at {ESP_IP}")   # the f prefix enables {} substitution
```

## Block 1 — Imports

```python
import threading
import time

import cv2
import numpy as np
import requests
from ultralytics import YOLO
```

| Import | Provides |
|---|---|
| `threading` | `Thread`, `Lock`, `Event` — running code in parallel |
| `time` | `time.time()`, `time.sleep()` — timing and delays |
| `cv2` | OpenCV — decode JPEG, display window, draw text |
| `numpy as np` | Fast arrays. `as np` is a universal convention |
| `requests` | HTTP client |
| `YOLO` | The model class from `ultralytics` |

`from ultralytics import YOLO` imports one name directly, so you write `YOLO(...)` rather than
`ultralytics.YOLO(...)`.

## Block 2 — Network configuration

```python
ESP_IP = "192.168.0.172"

STREAM_URL     = f"http://{ESP_IP}:81/stream"
PERSON_ON_URL  = f"http://{ESP_IP}/person_on"
PERSON_OFF_URL = f"http://{ESP_IP}/person_off"
```

**One source of truth.** Change `ESP_IP` and all three URLs follow. If the IP were repeated three
times, you would update two and spend twenty minutes wondering why the LED stopped working.

Note the ports: `:81` for video, nothing (so `:80`) for control.

## Block 3 — YOLO settings

```python
MODEL_NAME      = "yolo11n.pt"
YOLO_IMAGE_SIZE = 320
YOLO_CONFIDENCE = 0.25
PERSON_CLASS_ID = 0
```

**`MODEL_NAME`** — which model file to load. The `n` means "nano", the smallest.

| Model | Size | Params | Speed (CPU) | Accuracy |
|---|---|---|---|---|
| `yolo11n.pt` | 5 MB | 2.6 M | Fastest | Good |
| `yolo11s.pt` | 19 MB | 9.4 M | ~2× slower | Better |
| `yolo11m.pt` | 40 MB | 20 M | ~5× slower | Very good |
| `yolo11l.pt` | 50 MB | 25 M | ~8× slower | Excellent |
| `yolo11x.pt` | 110 MB | 57 M | ~15× slower | Best |

For person detection on a laptop CPU, **nano is the right call.** People are large, distinctive
objects; the bigger models mostly help with small or unusual classes.

The file downloads automatically on first use. `.pt` = PyTorch weights.

**`YOLO_IMAGE_SIZE = 320`** — every frame is resized to 320×320 before inference. Inference cost
scales roughly with the **square** of this number: 640 is ~4× slower than 320.

**`YOLO_CONFIDENCE = 0.25`** — minimum certainty (0.0–1.0) for a detection to count.

| Value | Effect |
|---|---|
| 0.10 | Catches almost everything, including shadows and coat racks |
| **0.25** | **Default. Good balance** |
| 0.50 | Only confident detections. Misses partially visible people |
| 0.75 | Very strict. Misses a lot |

**`PERSON_CLASS_ID = 0`** — COCO class 0 is `person`. See the [full class list](#the-80-coco-classes).

## Block 4 — Debounce settings

```python
PERSON_ON_FRAMES  = 2
PERSON_OFF_FRAMES = 8
```

See [Step 7 of the architecture](#step-7--debouncing-turns-the-raw-answer-into-a-stable-one) for the
full worked trace.

## Block 5 — Global state

```python
latest_frame      = None
latest_frame_id   = 0
frame_lock        = threading.Lock()
stop_event        = threading.Event()
last_person_state = None

stream_session  = requests.Session()
control_session = requests.Session()
stream_session.trust_env  = False
control_session.trust_env = False
```

| Variable | Purpose |
|---|---|
| `latest_frame` | The newest decoded image. `None` until the first frame arrives |
| `latest_frame_id` | A counter so the main loop can tell "new frame" from "same frame" |
| `frame_lock` | Prevents two threads touching `latest_frame` simultaneously |
| `stop_event` | A thread-safe flag. `stop_event.set()` tells the reader to stop |
| `last_person_state` | What we last told the ESP32. `None` forces the first send |

**Why `requests.Session()`?** A `Session` keeps the TCP connection open between requests. Without
it, every command does a full DNS lookup + TCP handshake — roughly 3× slower.

**`trust_env = False` is the important one.** By default `requests` reads `HTTP_PROXY` /
`HTTPS_PROXY` environment variables and routes through them. On a corporate laptop, or if you have
ever used a VPN, that means your local `192.168.x.x` requests get sent to an external proxy that
cannot reach your ESP32. Symptom: mysterious timeouts that work fine in a browser.

`trust_env = False` says "ignore all that, connect directly." **Removing this line is a genuine
source of hours-long debugging sessions.**

## Block 6 — `send_person_command()`

```python
def send_person_command(person_present, force=False):
    global last_person_state

    if not force and person_present == last_person_state:
        return

    url = PERSON_ON_URL if person_present else PERSON_OFF_URL

    try:
        response = control_session.get(url, timeout=2)
        response.raise_for_status()
        last_person_state = person_present
        print("ESP32: PERSON ON -> GPIO13 HIGH" if person_present
              else "ESP32: PERSON OFF -> GPIO12/13 LOW")
    except requests.RequestException as error:
        print(f"ESP32 command failed: {error}")
```

**`global last_person_state`** — without this line, assigning to `last_person_state` inside the
function would create a *new local variable* and the outer one would never change. Python requires
you to declare the intent explicitly. A classic beginner trap.

**The early return** is the whole point of the function: send nothing when nothing changed.

**`force=True`** bypasses that check. Used exactly twice: at startup and at shutdown, to guarantee
the ESP32 is in a known state.

**`timeout=2`** — give up after 2 seconds. **Never make a network call without a timeout.** The
default is *infinite*, which means one unreachable device hangs your whole program forever.

**`raise_for_status()`** turns an HTTP error code (404, 500) into a Python exception. Without it, a
404 would look like success.

**Order matters:** `last_person_state` is updated *after* the request succeeds. If the ESP32 is
briefly unreachable, the state stays stale and the next frame retries automatically.

## Block 7 — `update_stable_person_state()`

```python
def update_stable_person_state(person_detected_this_frame):
    global person_seen_frames, person_missing_frames, stable_person_state

    if person_detected_this_frame:
        person_seen_frames += 1
        person_missing_frames = 0
    else:
        person_missing_frames += 1
        person_seen_frames = 0

    if not stable_person_state and person_seen_frames >= PERSON_ON_FRAMES:
        stable_person_state = True
    elif stable_person_state and person_missing_frames >= PERSON_OFF_FRAMES:
        stable_person_state = False

    return stable_person_state
```

Note that the counters **reset each other**. Detection zeroes the missing counter and vice versa, so
they count *consecutive* frames, not totals. Alternating T/F/T/F never accumulates to a threshold —
which is exactly right, because that pattern means "uncertain", not "present".

### Worked example at 10 FPS

| Frame | Raw | seen | missing | Stable | Action |
|---|---|---|---|---|---|
| 1 | F | 0 | 1 | F | — |
| 2 | T | 1 | 0 | F | not yet (need 2) |
| 3 | T | 2 | 0 | **T** | **HTTP: person_on** |
| 4 | T | 3 | 0 | T | no send (unchanged) |
| 5 | F | 0 | 1 | T | still on |
| 6 | T | 1 | 0 | T | still on (glitch absorbed) |
| 7–13 | F ×7 | 0 | 1…7 | T | still on |
| 14 | F | 0 | 8 | **F** | **HTTP: person_off** |

Frames 5–6 show the debouncer earning its keep: a one-frame dropout is absorbed entirely.

### Time-based alternative

Frame-based debouncing means the timing changes with your FPS. At 20 FPS, 8 frames = 0.4s; at 3 FPS,
8 frames = 2.7s. If you want consistent timing:

```python
PERSON_OFF_SECONDS = 1.0
last_seen_time = 0.0

def update_stable_person_state(detected):
    global last_seen_time, stable_person_state
    now = time.time()
    if detected:
        last_seen_time = now
        stable_person_state = True
    elif now - last_seen_time > PERSON_OFF_SECONDS:
        stable_person_state = False
    return stable_person_state
```

## Block 8 — `camera_reader()`

```python
def camera_reader():
    global latest_frame, latest_frame_id

    while not stop_event.is_set():
        try:
            with stream_session.get(STREAM_URL, stream=True, timeout=(5, 10)) as response:
                response.raise_for_status()
                print("ESP32 camera stream connected.")
                jpeg_buffer = b""

                for chunk in response.iter_content(chunk_size=4096):
                    if stop_event.is_set():
                        return

                    jpeg_buffer += chunk

                    start = jpeg_buffer.find(b"\xff\xd8")
                    end   = jpeg_buffer.find(b"\xff\xd9", start + 2)

                    if start != -1 and end != -1:
                        jpg = jpeg_buffer[start:end + 2]
                        jpeg_buffer = jpeg_buffer[end + 2:]

                        frame = cv2.imdecode(
                            np.frombuffer(jpg, dtype=np.uint8), cv2.IMREAD_COLOR)

                        if frame is not None:
                            with frame_lock:
                                latest_frame = frame
                                latest_frame_id += 1

        except requests.RequestException as error:
            print(f"Stream error: {error}. Reconnecting in 2s...")
            time.sleep(2)
```

**`while not stop_event.is_set()`** — the outer loop reconnects forever until told to stop.

**`stream=True`** is essential. Without it, `requests` tries to download the *entire* response into
memory before returning — and the stream never ends, so it would hang forever and eventually run out
of RAM.

**`timeout=(5, 10)`** is a tuple: 5 seconds to establish the connection, 10 seconds of silence before
giving up. A single number would apply to both.

**`with ... as response`** guarantees the connection is closed even if an exception is raised —
Python's equivalent of RAII.

**`b""`** is an empty **bytes** object, not a string. Network data is raw bytes; `""` would be text.

**`chunk_size=4096`** — read 4 KB at a time. Smaller means more Python loop overhead; larger means
more latency before a complete frame is spotted.

**The marker scan** is explained in
[Step 4 of the architecture](#step-4--python-receives-and-reassembles-frames).

**`if frame is not None`** — `imdecode` returns `None` on corrupt data. Without this check, a single
glitched JPEG crashes the thread and the video stops permanently.

**The reconnect loop.** `except requests.RequestException` catches every network error `requests` can
raise. Sleep 2 seconds, then the outer `while` tries again. This is what makes the system survive
router reboots and Wi-Fi dropouts unattended.

## Block 9 — Startup

```python
print("Loading YOLO model...")
model = YOLO(MODEL_NAME)

send_person_command(False, force=True)

print("Connecting to ESP32 camera stream...")
reader_thread = threading.Thread(target=camera_reader, daemon=True)
reader_thread.start()

print("Press Q in the video window to stop.")
```

**Order matters:**

1. **Load the model first.** It takes 1–3 seconds. Doing it before the stream starts means no frames
   pile up during loading.
2. **Force GPIO LOW.** If a previous run crashed with the LED on, this resets it. Never assume the
   hardware is in a known state.
3. **Start the reader thread.** `daemon=True` so it dies with the program.

## Block 10 — The main loop

```python
frame_count = 0
fps = 0.0
fps_timer = time.time()
last_processed_frame_id = -1

try:
    while True:
        with frame_lock:
            current_frame = None if latest_frame is None else latest_frame.copy()
            current_frame_id = latest_frame_id

        if current_frame is None or current_frame_id == last_processed_frame_id:
            if cv2.waitKey(1) & 0xFF == ord("q"):
                break
            continue

        last_processed_frame_id = current_frame_id
        ...
```

**`latest_frame.copy()`** — this is not paranoia. Without the copy, the main thread holds a
*reference* to the same NumPy array the reader thread is about to overwrite. YOLO could then be
reading pixels that change underneath it mid-inference. The copy costs ~0.2 ms and eliminates a
whole class of impossible-to-reproduce bugs.

**The lock is released immediately** after the copy. Holding it during inference would block the
reader for 200 ms per frame and defeat the entire threading design.

**`cv2.waitKey(1)` in the skip path** — even when there is no new frame, the window's event loop must
be pumped or the OS marks it "not responding" and it turns grey.

**`& 0xFF`** masks to the low 8 bits. On some platforms `waitKey` returns extra high bits.
`ord("q")` gives the ASCII code for `q`.

### Inference and the visualisation

```python
        results = model(
            current_frame,
            imgsz=YOLO_IMAGE_SIZE,
            conf=YOLO_CONFIDENCE,
            classes=[PERSON_CLASS_ID],
            verbose=False,
            device="cpu"
        )

        boxes = results[0].boxes
        person_detected_this_frame = boxes is not None and len(boxes) > 0

        stable = update_stable_person_state(person_detected_this_frame)
        send_person_command(stable)

        annotated = results[0].plot()
```

**`verbose=False`** — without it, Ultralytics prints a line per frame. At 10 FPS your terminal
becomes unreadable and the printing itself costs FPS.

**`classes=[PERSON_CLASS_ID]`** — filtering happens *inside* NMS, so it is genuinely faster, not just
a post-filter.

**`results[0]`** — the API supports batching multiple images. You passed one, so you want index 0.

**`.plot()`** returns a **new** annotated BGR array with boxes, labels, and confidence scores drawn.
It does not modify your original.

```python
        if stable:
            label, colour = "PERSON CONFIRMED - GPIO13 HIGH", (0, 0, 255)
        else:
            label, colour = "NO PERSON - GPIO12/13 LOW", (0, 255, 0)

        cv2.putText(annotated, label, (10, 25),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.6, colour, 2)
```

> [!WARNING]
> **OpenCV colours are BGR, not RGB.**
>
> | Tuple | Colour |
> |---|---|
> | `(0, 0, 255)` | **Red** |
> | `(0, 255, 0)` | Green |
> | `(255, 0, 0)` | **Blue** |
> | `(0, 255, 255)` | Yellow |
> | `(255, 255, 255)` | White |
>
> If your red text comes out blue, this is why.

`cv2.putText(image, text, origin, font, scale, colour, thickness)` — note that `origin` is the
**bottom-left** corner of the text, so `(10, 25)` puts it near the top of the image, not below it.

### FPS calculation

```python
        frame_count += 1
        if frame_count >= 10:
            now = time.time()
            fps = frame_count / (now - fps_timer)
            fps_timer = now
            frame_count = 0

        cv2.putText(annotated, f"FPS: {fps:.1f}", (10, 50),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.6, (255, 255, 0), 2)

        cv2.imshow("ESP32-CAM YOLO Person Detection", annotated)

        if cv2.waitKey(1) & 0xFF == ord("q"):
            break
```

Averaging over 10 frames smooths the reading. Per-frame timing jumps between 4 and 12 and tells you
nothing. `{fps:.1f}` formats to one decimal place.

### The `finally` block

```python
except KeyboardInterrupt:
    print("\nInterrupted by user.")
finally:
    stop_event.set()
    send_person_command(False, force=True)
    cv2.destroyAllWindows()
    stream_session.close()
    control_session.close()
    print("Program closed safely.")
```

**`finally` runs no matter what** — normal exit, `Q`, Ctrl+C, or an unhandled exception three
functions deep.

**This is the safety net.** Without it, a crash could leave GPIO13 HIGH indefinitely — a relay stuck
closed, a lock stuck open, a light stuck on. **Always force outputs to a safe state in `finally`.**

Order: stop the reader first, then reset hardware, then release GUI and network resources.

---

## Modification Recipes

Ready-to-paste changes for the most common extensions.

### Detect multiple classes

```python
CLASSES_TO_DETECT = [0, 2, 16]        # person, car, dog

results = model(frame, classes=CLASSES_TO_DETECT, ...)

detected = set()
for box in results[0].boxes:
    detected.add(int(box.cls[0]))

if 0 in detected:  send_person_command(True)
if 2 in detected:  send_car_command(True)     # add a matching ESP32 endpoint
```

### Trigger only inside a zone

```python
ZONE = (100, 50, 250, 200)        # x1, y1, x2, y2

def in_zone(box, zone):
    x1, y1, x2, y2 = box.xyxy[0].tolist()
    cx, cy = (x1 + x2) / 2, (y1 + y2) / 2
    zx1, zy1, zx2, zy2 = zone
    return zx1 <= cx <= zx2 and zy1 <= cy <= zy2

person_in_zone = any(in_zone(b, ZONE) for b in (boxes or []))

cv2.rectangle(annotated, ZONE[:2], ZONE[2:], (255, 0, 255), 2)   # draw it
```

### Trigger only on N or more people

```python
MIN_PEOPLE = 3
person_detected_this_frame = boxes is not None and len(boxes) >= MIN_PEOPLE
```

### Save a snapshot on each detection

```python
from datetime import datetime

previous_stable = stable_person_state
stable = update_stable_person_state(person_detected_this_frame)

if stable and not previous_stable:      # OFF → ON transition only
    filename = f"detect_{datetime.now():%Y%m%d_%H%M%S}.jpg"
    cv2.imwrite(filename, annotated)
    print(f"Saved {filename}")
```

### Log detections to CSV

```python
import csv

log = open("detections.csv", "a", newline="")
writer = csv.writer(log)
writer.writerow(["timestamp", "event", "count"])

if stable != previous_stable:
    writer.writerow([datetime.now().isoformat(),
                     "ENTER" if stable else "EXIT",
                     len(boxes) if boxes is not None else 0])
    log.flush()
```

Close it in the `finally` block.

### Headless mode (no window, ~20–30% more FPS)

```python
SHOW_WINDOW = False

if SHOW_WINDOW:
    cv2.imshow("ESP32-CAM YOLO Person Detection", annotated)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break
else:
    time.sleep(0.001)      # yield to the reader thread
```

Skipping `.plot()` as well saves even more — that call is not free.

---

# YOLO for Absolute Beginners

## Where YOLO sits

```
┌─────────────────────────────────────────────────────────┐
│  ARTIFICIAL INTELLIGENCE                                │
│  Any machine doing something that seems intelligent     │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  MACHINE LEARNING                                 │  │
│  │  Systems that learn patterns from data instead    │  │
│  │  of being explicitly programmed                   │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  DEEP LEARNING                              │  │  │
│  │  │  ML using many-layered neural networks      │  │  │
│  │  │                                             │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │
│  │  │  │  COMPUTER VISION                      │  │  │  │
│  │  │  │  Deep learning applied to images      │  │  │  │
│  │  │  │                                       │  │  │  │
│  │  │  │      ┌─────────────────────────┐      │  │  │  │
│  │  │  │      │  OBJECT DETECTION       │      │  │  │  │
│  │  │  │      │  → YOLO lives here      │      │  │  │  │
│  │  │  │      └─────────────────────────┘      │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Machine learning vs traditional programming

| Traditional | Machine learning |
|---|---|
| You write the rules | You provide examples |
| Input + rules → output | Input + output → **rules** |
| `if brightness > 200: bright` | Show 10,000 labelled images; the machine derives the rule |
| Works for defined logic | Works for fuzzy perception |

Consider: write rules to detect a person in an image. Head shape? People wear hats, turn away, get
cropped by the frame edge. Skin tone? Varies enormously, plus clothing. Two legs? Sitting people
exist. Height? Depends on distance.

**Nobody has ever succeeded at writing those rules by hand.** Machine learning sidesteps the problem:
show the system a million labelled photos and let it work out its own internal features.

## Neural networks in one page

A **neural network** is loosely inspired by biological neurons.

```
Input          Hidden layers                Output
──────         ─────────────                ──────

pixel 1 ──┐    ┌──○──┐    ┌──○──┐
pixel 2 ──┼───►│  ○  │───►│  ○  │───►  person: 0.94
pixel 3 ──┼───►│  ○  │───►│  ○  │───►  car:    0.02
   ...    │    │  ○  │    │  ○  │───►  dog:    0.01
pixel N ──┘    └──○──┘    └──○──┘
```

Each connection has a **weight** — a number saying how strongly that input matters. Each neuron sums
its weighted inputs, applies a nonlinear function, and passes the result on.

**Training** = showing the network millions of examples and nudging every weight slightly whenever
the output is wrong. Repeat billions of times. That is it, conceptually.

YOLO11n has **2.6 million weights**, learned from ~200,000 labelled images.

### Images are just numbers

A greyscale pixel is one number, 0 (black) to 255 (white). A colour pixel is three (red, green,
blue). A 320×240 colour image is a **320 × 240 × 3 grid = 230,400 numbers**.

```
Pixel at (10, 20) = [58, 142, 201]   ← in OpenCV: Blue, Green, Red
```

The neural network never sees an "image". It sees a large array of numbers.

### Convolution — the key idea for vision

A **convolutional layer** slides a small window (a "kernel", typically 3×3) across the image, looking
for one specific local pattern. Early layers learn edges and colour gradients. Middle layers combine
edges into corners, curves, textures. Late layers combine those into eyes, wheels, limbs. The final
layers combine *those* into whole-object concepts.

Nobody programs this hierarchy. It emerges from training.

## Three tasks that get confused

| Task | Question answered | Output |
|---|---|---|
| **Classification** | "What is in this image?" | One label: `dog` |
| **Detection** | "What is in it, and *where*?" | Labels + boxes: `dog at (120,80)-(340,290)` |
| **Segmentation** | "Which exact pixels belong to it?" | A per-pixel mask |

This project uses **detection**. It needs the "where" so it can draw boxes and, if you extend it,
check zones.

## What YOLO means

**YOLO = You Only Look Once.**

Before YOLO (2015), detectors worked in two stages: propose ~2000 candidate regions, then classify
each one. That is 2000 network passes per image — seconds per frame.

YOLO's insight: **divide the image into a grid and predict all boxes and classes in a single forward
pass.** One pass, not 2000. This is what made real-time detection possible.

| Version | Year | Note |
|---|---|---|
| YOLOv1 | 2015 | The original single-pass idea |
| YOLOv3 | 2018 | Multi-scale — detects small objects far better |
| YOLOv5 | 2020 | Ultralytics' PyTorch rewrite; huge adoption |
| YOLOv8 | 2023 | Unified detect/segment/pose/classify API |
| **YOLO11** | **2024** | **Used here.** Better accuracy at lower parameter counts |

## Bounding boxes

A box is four numbers. Two conventions exist:

| Format | Meaning | Used by |
|---|---|---|
| `xyxy` | `x1, y1, x2, y2` — top-left and bottom-right corners | Ultralytics output |
| `xywh` | `centre_x, centre_y, width, height` | YOLO label files |

```
(0,0)────────────────────► X
  │
  │      (x1,y1)
  │        ┌─────────┐
  │        │ person  │
  │        │  0.94   │
  │        └─────────┘
  │              (x2,y2)
  ▼ Y
```

**Origin is the top-left** in image coordinates, and Y increases *downwards*. This trips up anyone
coming from maths.

```python
for box in results[0].boxes:
    x1, y1, x2, y2 = box.xyxy[0].tolist()
    confidence     = float(box.conf[0])
    class_id       = int(box.cls[0])
```

## Confidence scores

A number 0.0–1.0 meaning "how sure the model is". Roughly:

| Score | Interpretation |
|---|---|
| 0.90+ | Textbook example — clear, unobstructed, well-lit |
| 0.70–0.90 | Confident. Normal for a person facing the camera |
| 0.40–0.70 | Probably right. Partially visible, small, or poorly lit |
| 0.25–0.40 | Uncertain. Could be a coat on a chair |
| below 0.25 | Discarded by `conf=0.25` |

**It is not a probability of correctness.** It is the model's internal score, and models can be
confidently wrong.

## NMS and IoU

After the forward pass, YOLO has produced thousands of overlapping candidate boxes around each real
object. **Non-Maximum Suppression** cleans that up:

1. Sort all boxes by confidence.
2. Take the highest-scoring one; keep it.
3. Delete every remaining box that overlaps it too much.
4. Repeat with the next highest.

"Too much" is measured by **IoU** (*Intersection over Union*):

```
IoU = area of overlap / area of union

┌────────┐
│   A    │        IoU ≈ 0.0   completely separate
└────────┘  ┌────────┐
            │   B    │
            └────────┘

┌────────┐
│  A ┌───┼────┐     IoU ≈ 0.3   partial overlap
└────┼───┘ B  │
     └────────┘

┌────────┐
│  A/B   │        IoU ≈ 1.0   identical
└────────┘
```

The default NMS threshold is 0.7. Ultralytics does this automatically — you never call it yourself.

## COCO and the 80 classes

**COCO** = *Common Objects in Context* — a Microsoft dataset of ~330,000 images with ~1.5 million
labelled object instances across **80 categories**. YOLO11's pretrained weights are trained on it.

**Person is class 0** because it is the most common and most-requested category, and COCO ordered its
classes roughly by prevalence.

### The 80 COCO classes

| ID | Name | ID | Name | ID | Name | ID | Name |
|---|---|---|---|---|---|---|---|
| 0 | person | 20 | elephant | 40 | wine glass | 60 | dining table |
| 1 | bicycle | 21 | bear | 41 | cup | 61 | toilet |
| 2 | car | 22 | zebra | 42 | fork | 62 | tv |
| 3 | motorcycle | 23 | giraffe | 43 | knife | 63 | laptop |
| 4 | airplane | 24 | backpack | 44 | spoon | 64 | mouse |
| 5 | bus | 25 | umbrella | 45 | bowl | 65 | remote |
| 6 | train | 26 | handbag | 46 | banana | 66 | keyboard |
| 7 | truck | 27 | tie | 47 | apple | 67 | cell phone |
| 8 | boat | 28 | suitcase | 48 | sandwich | 68 | microwave |
| 9 | traffic light | 29 | frisbee | 49 | orange | 69 | oven |
| 10 | fire hydrant | 30 | skis | 50 | broccoli | 70 | toaster |
| 11 | stop sign | 31 | snowboard | 51 | carrot | 71 | sink |
| 12 | parking meter | 32 | sports ball | 52 | hot dog | 72 | refrigerator |
| 13 | bench | 33 | kite | 53 | pizza | 73 | book |
| 14 | bird | 34 | baseball bat | 54 | donut | 74 | clock |
| 15 | cat | 35 | baseball glove | 55 | cake | 75 | vase |
| 16 | dog | 36 | skateboard | 56 | chair | 76 | scissors |
| 17 | horse | 37 | surfboard | 57 | couch | 77 | teddy bear |
| 18 | sheep | 38 | tennis racket | 58 | potted plant | 78 | hair drier |
| 19 | cow | 39 | bottle | 59 | bed | 79 | toothbrush |

To detect something else, change `PERSON_CLASS_ID` to that number. To detect several, pass a list:
`classes=[0, 2, 16]`.

## Precision, recall, mAP

| Metric | Question | Formula |
|---|---|---|
| **Precision** | Of everything I flagged, how much was right? | TP / (TP + FP) |
| **Recall** | Of everything real, how much did I find? | TP / (TP + FN) |
| **mAP50** | Average precision at IoU ≥ 0.5 | The headline number |
| **mAP50-95** | Averaged over IoU 0.5 → 0.95 | The strict number |

They trade off. Lower `conf` → higher recall, lower precision. Raise it → the reverse.

For a security alarm you want **high recall** (never miss a real intruder). For an automated door
lock you want **high precision** (never open for a coat rack).

## What YOLO cannot do

| Cannot | Why |
|---|---|
| Identify *who* a person is | That is face recognition, a different task |
| Detect objects not in its 80 classes | Needs [custom training](#training-yolo-on-your-own-objects) |
| Reliably see very small objects | A person 10 pixels tall is mostly noise |
| See in near-total darkness | It needs light; use an IR camera + IR-trained model |
| Track identity across frames | Detection is per-frame. That is object *tracking* |
| Measure real-world distance | A monocular camera has no depth information |
| Tell a real person from a photo of one | It sees pixels, not physics |

## Learning more

| Resource | Good for |
|---|---|
| [Ultralytics docs](https://docs.ultralytics.com/) | Every API option, worked examples |
| [3Blue1Brown — Neural Networks](https://www.youtube.com/playlist?list=PLZHQObOWTQDNU6R1_67000Dx_ZCJB-3pi) | The best visual intuition anywhere |
| [CS231n](http://cs231n.stanford.edu/) | Stanford's convolutional networks course |
| [Roboflow blog](https://blog.roboflow.com/) | Practical training and dataset advice |
| [COCO explorer](https://cocodataset.org/#explore) | Browse the dataset YOLO learned from |

---

# Training YOLO on Your Own Objects

## Do you actually need to?

```
Is your object one of the 80 COCO classes?
├── YES → Just change PERSON_CLASS_ID. Done in 10 seconds.
└── NO
    └── Is it visually distinctive (unique shape or colour)?
        ├── NO, it's subtle → you will need 1000+ images. Consider a
        │                     physical marker (ArUco) instead — vastly easier
        └── YES → Custom training is worth it. Continue below.
```

> **The ArUco shortcut.** If you control the object — a robot, a box, a tool — sticking an ArUco
> marker on it gives you 100% reliable detection with zero training, in about 20 lines of OpenCV.
> Custom training is for objects you cannot modify.

## Time and cost budget

| Stage | Time | Cost |
|---|---|---|
| Collect 200–500 images | 1–3 hours | Free |
| Annotate them | 2–6 hours | Free (or ~$0.05/image outsourced) |
| Split + configure | 15 min | Free |
| Train (Google Colab free T4) | 1–3 hours | Free |
| Validate and iterate | 1–2 hours | Free |
| **Total** | **~1 weekend** | **$0** |

Annotation is the bottleneck. Everything else is fast.

## Stage 1 — Collect images

**From your ESP32-CAM directly** (best — same lens, same lighting, same distortion):

```python
import cv2, numpy as np, requests, time, os

ESP_IP = "192.168.0.172"
os.makedirs("dataset/raw", exist_ok=True)

session = requests.Session()
session.trust_env = False
response = session.get(f"http://{ESP_IP}:81/stream", stream=True, timeout=(5, 10))

buffer = b""
saved = 0
last_save = 0

for chunk in response.iter_content(4096):
    buffer += chunk
    start = buffer.find(b"\xff\xd8")
    end   = buffer.find(b"\xff\xd9", start + 2)

    if start != -1 and end != -1:
        jpg = buffer[start:end + 2]
        buffer = buffer[end + 2:]

        if time.time() - last_save > 0.5:          # 2 images/sec
            frame = cv2.imdecode(np.frombuffer(jpg, np.uint8), cv2.IMREAD_COLOR)
            cv2.imwrite(f"dataset/raw/img_{saved:04d}.jpg", frame)
            saved += 1
            last_save = time.time()
            print(f"Saved {saved}", end="\r")

        if saved >= 300:
            break
```

**Vary everything:** angle, distance, lighting, background, partial occlusion, and include some
frames with **no** target object at all (negative examples — they matter more than people expect).

| Target | Images needed |
|---|---|
| Proof of concept | 100–200 |
| Usable | 300–500 |
| Good | 1000+ |
| Production | 5000+ |

## Stage 2 — Annotate

| Tool | Type | Notes |
|---|---|---|
| [Roboflow](https://roboflow.com/) | Web | Easiest. Free tier, auto-augmentation, direct YOLO export |
| [CVAT](https://www.cvat.ai/) | Web/self-host | Powerful, free, steeper learning curve |
| [LabelImg](https://github.com/HumanSignal/labelImg) | Desktop | Simple, offline, YOLO format native |
| [Label Studio](https://labelstud.io/) | Both | Handles many data types |

**YOLO label format.** One `.txt` per image, same base name:

```
img_0001.jpg  →  img_0001.txt
```

```
0 0.512 0.634 0.180 0.427
│   │     │     │     └─ height (normalised 0–1)
│   │     │     └─────── width
│   │     └───────────── centre Y
│   └─────────────────── centre X
└─────────────────────── class ID
```

**All values are normalised 0–1** relative to image dimensions — so labels survive resizing. An empty
`.txt` means "no objects in this image", which is valid and useful.

**Annotation rules that decide whether training works:**

1. **Box tightly.** Touch the object's edges, no margin.
2. **Label every instance.** One missed object teaches the model that object is background.
3. **Include partially visible ones.** Box the visible part only.
4. **Be consistent.** If you include the handle on a mug in one image, do it in all of them.
5. **Never guess.** If you cannot tell what it is, skip the image.

## Stage 3 — Split the data

```
dataset/
├── images/
│   ├── train/       70%
│   ├── val/         20%
│   └── test/        10%
└── labels/
    ├── train/
    ├── val/
    └── test/
```

| Split | Purpose |
|---|---|
| **train** | The model learns from these |
| **val** | Checked after each epoch to detect overfitting. The model never learns from them |
| **test** | Touched once, at the very end, for an honest final number |

```python
import os, random, shutil

random.seed(42)
images = sorted(f for f in os.listdir("dataset/raw") if f.endswith(".jpg"))
random.shuffle(images)

n = len(images)
splits = {
    "train": images[:int(0.7*n)],
    "val":   images[int(0.7*n):int(0.9*n)],
    "test":  images[int(0.9*n):],
}

for split, files in splits.items():
    os.makedirs(f"dataset/images/{split}", exist_ok=True)
    os.makedirs(f"dataset/labels/{split}", exist_ok=True)
    for f in files:
        shutil.copy(f"dataset/raw/{f}", f"dataset/images/{split}/{f}")
        label = f.replace(".jpg", ".txt")
        src = f"dataset/raw/{label}"
        if os.path.exists(src):
            shutil.copy(src, f"dataset/labels/{split}/{label}")
    print(f"{split}: {len(files)}")
```

**`random.seed(42)`** makes the split reproducible — rerun it and you get the same division.

## Stage 4 — `data.yaml`

```yaml
path: /absolute/path/to/dataset
train: images/train
val: images/val
test: images/test

nc: 2
names:
  0: bottle_cap
  1: screw
```

`nc` must equal the number of entries in `names`, and your label files must only use IDs `0` to
`nc-1`.

## Stage 5 — Train

```bash
yolo train \
  model=yolo11n.pt \
  data=/path/to/data.yaml \
  epochs=100 \
  imgsz=320 \
  batch=16 \
  patience=20 \
  project=runs \
  name=my_detector
```

| Argument | Meaning |
|---|---|
| `model=yolo11n.pt` | Start from COCO-pretrained weights, not random. **Transfer learning** |
| `epochs=100` | One epoch = one full pass through the training set |
| `imgsz=320` | Match your deployment size, or accuracy will differ |
| `batch=16` | Images processed together. Lower it if you run out of memory |
| `patience=20` | Stop early if 20 epochs pass with no improvement |

**Transfer learning is why this works with 300 images instead of 300,000.** The pretrained model
already knows edges, textures, and shapes from COCO. You are only teaching it a new final mapping.

### Reading the output

```
      Epoch    box_loss    cls_loss    dfl_loss   Instances
      1/100       1.847       3.201       1.412         142
     50/100       0.612       0.428       0.891         138
    100/100       0.398       0.201       0.798         141

                 Class    Images  Instances   Box(P      R    mAP50  mAP50-95)
                   all        60        187   0.891  0.856    0.912     0.674
```

| Signal | Meaning |
|---|---|
| All losses trending down | ✅ Learning normally |
| Train loss ↓ but val loss ↑ | ⚠️ **Overfitting** — memorising instead of generalising |
| Losses flat from epoch 1 | ❌ Bad labels, wrong `data.yaml` path, or learning rate too low |
| mAP50 > 0.8 | ✅ Good for a custom dataset |
| mAP50 < 0.5 | ❌ Needs more or better data |

Outputs land in `runs/my_detector/weights/`:
- `best.pt` — highest validation mAP. **Use this one.**
- `last.pt` — the final epoch. For resuming training.

### Free GPU on Google Colab

```python
# Runtime → Change runtime type → T4 GPU
!pip install ultralytics
from google.colab import drive; drive.mount('/content/drive')
!yolo train model=yolo11n.pt data=/content/drive/MyDrive/dataset/data.yaml epochs=100 imgsz=320
!cp -r runs /content/drive/MyDrive/
```

A T4 trains ~20× faster than a laptop CPU. Sessions disconnect after ~12 hours of inactivity, so save
to Drive.

## Stage 6 — Use it

```python
MODEL_NAME      = "runs/my_detector/weights/best.pt"
PERSON_CLASS_ID = 0        # now means YOUR class 0
```

Everything else in the script is unchanged.

## Diagnosing a disappointing model

| Symptom | Likely cause | Fix |
|---|---|---|
| High train mAP, low val mAP | Overfitting | More data, more augmentation, fewer epochs |
| Both low | Underfitting | Train longer, bigger model, check labels |
| Misses small objects | `imgsz` too low | Train and infer at 640 |
| False positives on background | Not enough negative images | Add images with no target |
| Works indoors, fails outdoors | Training data lacked variety | Collect in both conditions |
| Good on your photos, bad on the ESP32 | Domain gap | **Collect training images from the ESP32 itself** |

## Twelve common mistakes

1. Too few images (under 100 rarely works)
2. All images from one angle or one lighting condition
3. Loose bounding boxes
4. Forgetting to label some instances
5. Inconsistent class definitions between sessions
6. No negative examples
7. Training at 640 but deploying at 320
8. Using `last.pt` instead of `best.pt`
9. Validation images that also appear in training (leakage → fake-good numbers)
10. Wrong `path` in `data.yaml` (must be absolute)
11. `nc` not matching `names`
12. Training on stock photos, deploying on a cheap camera

## Four worked project ideas

| Project | Classes | Images | Difficulty | Notes |
|---|---|---|---|---|
| **Bottle detector** | 1 | 200 | ⭐ Easy | Distinctive shape. Great first project |
| **Car / license plate** | 2 | 500 | ⭐⭐ Medium | Plates need `imgsz=640` — they are small |
| **Face mask compliance** | 2 | 800 | ⭐⭐⭐ Hard | Faces vary enormously |
| **Robot / marker tracking** | 1–3 | 300 | ⭐⭐ Medium | Consider ArUco markers instead — far easier |

## Export formats

```bash
yolo export model=runs/my_detector/weights/best.pt format=onnx
```

| Format | Use case | Speedup |
|---|---|---|
| `onnx` | Cross-platform CPU | 2–3× |
| `openvino` | Intel CPUs specifically | 3–4× |
| `engine` | NVIDIA TensorRT | 5–10× (GPU) |
| `coreml` | Apple devices | Native |
| `tflite` | Mobile / Raspberry Pi | Varies |

---

# Configuration Reference

## Python side

| Setting | Default | What it does | Change it when… |
|---|---|---|---|
| `ESP_IP` | `"192.168.0.172"` | Your ESP32's address | **Always** — must match your board |
| `MODEL_NAME` | `"yolo11n.pt"` | Which YOLO model to load | More accuracy → `"yolo11s.pt"` |
| `YOLO_IMAGE_SIZE` | `320` | Resize before inference | Low FPS → `256`. Missing distant people → `416` |
| `YOLO_CONFIDENCE` | `0.25` | Minimum certainty | False alarms → `0.40`. Missing real people → `0.15` |
| `PERSON_CLASS_ID` | `0` | COCO class 0 = person | Detecting something else |
| `PERSON_ON_FRAMES` | `2` | Detections before GPIO HIGH | Triggering on noise → `4` |
| `PERSON_OFF_FRAMES` | `8` | Misses before GPIO LOW | Lingering too long → `4` |

## ESP32 side

| Setting | Default | What it does |
|---|---|---|
| `GPIO_PERSON_LED` | `13` | The pin that goes HIGH on detection |
| `GPIO_ALWAYS_LOW` | `12` | Held LOW — reserved / safety pin |
| `cameraConfig.frame_size` | `FRAMESIZE_QVGA` (320×240) | Larger = better quality, lower FPS |
| `cameraConfig.jpeg_quality` | `15` | **Lower number = higher quality.** Range 10–63 |
| `cameraConfig.fb_count` | `1` | Frame buffers. `2` **only** if PSRAM is detected |
| `cameraConfig.xclk_freq_hz` | `20000000` | Sensor clock. 10 MHz sometimes fixes unstable boards |

---

# Performance Tuning

| Machine | FPS at `imgsz=320`, CPU | Usable? |
|---|---|---|
| Modern laptop (Apple Silicon / recent Intel/AMD) | 10–20 | ✅ Smooth |
| Mid-range 4-core laptop | 5–10 | ✅ Fine for person detection |
| Older dual-core | 2–4 | ⚠️ Laggy but functional |
| Any NVIDIA GPU (CUDA) | 30–60+ | ✅ Excellent |

**The ESP32-CAM itself caps out at roughly 15–25 FPS at QVGA**, so past ~20 FPS the camera, not your
CPU, is the bottleneck.

## Free 2–3× speedup on CPU — export to ONNX

```bash
yolo export model=yolo11n.pt format=onnx
```
```python
MODEL_NAME = "yolo11n.onnx"
```

**ONNX Runtime** is an optimised inference engine. It fuses layers and uses better CPU kernels than
PyTorch's general-purpose path. No GPU required, no accuracy loss.

On Intel CPUs, `format=openvino` is often faster still.

## Using a GPU

**NVIDIA (CUDA):**

```bash
pip uninstall torch torchvision -y
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
```
```python
device="0"     # or device=0
```

**Apple Silicon (MPS):**

```python
device="mps"
```

Verify with:

```bash
python -c "import torch; print(torch.cuda.is_available())"
python -c "import torch; print(torch.backends.mps.is_available())"
```

## Other levers

| Change | Effect |
|---|---|
| `YOLO_IMAGE_SIZE` 320 → 256 | ~1.5× faster, slightly worse at distance |
| Skip `.plot()`, run headless | 20–30% faster |
| Close Chrome and other apps | Often the biggest single win on a 4-core laptop |
| Plug the laptop in | Battery-saver profiles throttle the CPU hard |
| Move both devices to 5 GHz Wi-Fi | Lower transfer latency (the ESP32 stays on 2.4 GHz, but your laptop's link to the router improves) |
| `yolo11n` → `yolo11s` | ~2× *slower*, noticeably more accurate |

---

# Troubleshooting

## The 60-second diagnostic ladder

Work down this list. Each step isolates one layer.

```
1. Is the ESP32 powered and printing to Serial?          → no: power / wiring
2. Does Serial show an IP address?                        → no: Wi-Fi (§A6)
3. Does  ping <ESP_IP>  respond?                          → no: network (§B)
4. Does the stream open in a browser at :81/stream?       → no: camera / stream (§A1, §B)
5. Does  curl http://<ESP_IP>/person_on  light the LED?   → no: GPIO / wiring (§C)
6. Does  python -c "import cv2, ultralytics"  work?       → no: environment (§D)
7. Does the Python script connect?                        → no: runtime (§E)
```

Or just run the bundled diagnostic:

```bash
python python/diagnose.py
```

---

## §A — ESP32 problems

### A1. `Camera initialization failed`

**Symptom:** Serial prints this and then nothing.

| Cause | Fix |
|---|---|
| Insufficient power (**most common**) | External 5V @ ≥1A. Do not power from the FTDI's 3.3V pin |
| Ribbon cable not seated | Lift the black connector latch, reinsert fully, close the latch |
| Wrong board selected | Tools → Board → **AI Thinker ESP32-CAM** |
| Wrong pin map | Only if you changed boards — copy from `camera_pins.h` |
| Dead camera module | Try another OV2640 |

### A2. `Brownout detector was triggered` / reboot loop

The supply voltage dipped below the chip's minimum. **This is always power.**

- Use a 5V @ 1A+ supply, not USB from a hub
- Shorter, thicker power wires — thin jumpers have real resistance
- Add a 470 µF electrolytic capacitor across 5V and GND close to the board
- Confirm the FTDI jumper is on **5V**, not 3.3V

### A3. Reboot loop with nothing obviously wrong

Check whether anything is attached to **GPIO12**. It is a strapping pin — HIGH at reset selects 1.8V
flash and the board will not boot. Disconnect everything from GPIO12.

### A4. `Failed to connect to ESP32`

| Cause | Fix |
|---|---|
| Not in flash mode | `IO0` → `GND`, press RST, *then* click Upload |
| TX/RX not crossed | ESP `U0T` → adapter `RX`; ESP `U0R` → adapter `TX` |
| Wrong port selected | Tools → Port |
| Charge-only USB cable | Try another cable |
| Upload speed too high | Drop to `115200` |
| Missing driver | Install CP2102 / CH340 / FTDI drivers |

### A5. `text section exceeds available space in board`

**Tools → Partition Scheme → Huge APP (3MB No OTA/1MB SPIFFS)**

### A6. Wi-Fi never connects (endless dots)

1. Password typo — check character by character
2. **Trailing space in the SSID** — `"MyWiFi "` ≠ `"MyWiFi"`
3. Network is 5 GHz only — the ESP32 has a 2.4 GHz radio
4. Weak signal — move closer to the router
5. MAC filtering enabled on the router
6. Insufficient power — the radio spikes current when connecting
7. Captive portal (hotel/university Wi-Fi with a login page) — use a phone hotspot

### A7. Serial Monitor shows gibberish (`⸮⸮⸮@⸮`)

Baud rate mismatch. Set the Serial Monitor to **115200**.

### A8. `Camera stream server failed`

The second `httpd_start()` failed. Almost always because both servers use the same `ctrl_port`.
Confirm 32768 and 32769 are different.

---

## §B — Network and streaming

### B1. Browser cannot open `:81/stream`

| Check | How |
|---|---|
| Correct IP | Re-read the Serial Monitor now, not from memory |
| Same network | Both devices on the same router and subnet |
| Port included | `http://192.168.0.172:81/stream` — the `:81` is required |
| AP isolation off | Some routers block client-to-client traffic. Look for "AP isolation" or "client isolation" in the router admin |
| Guest network | Guest networks almost always isolate clients. Use the main network |

### B2. Browser works but Python does not

**The browser tab is still open.** The ESP32 serves one stream client at a time. Close every tab
pointing at `:81/stream`.

Second most common: a proxy. Confirm `trust_env = False` is set on both sessions.

### B3. Stream freezes after one frame

Missing `esp_camera_fb_return(frame)` in the stream handler.

### B4. Stream is very laggy

- Move closer to the router
- Lower `jpeg_quality` (higher number = smaller files)
- Check for 2.4 GHz congestion — microwaves, Bluetooth, neighbours' networks
- Reduce `frame_size`

### B5. Detection stopped working after weeks

The ESP32's DHCP lease expired and the router gave it a new IP. Check the Serial Monitor and update
`ESP_IP`. Set a **DHCP reservation** in your router to make it permanent.

---

## §C — GPIO and hardware

### C1. Box appears but the LED does not light

```bash
curl http://YOUR_ESP32_IP/person_on
```

| Result | Meaning |
|---|---|
| JSON + LED lights | ESP32 is fine. Firewall or proxy is blocking Python |
| JSON, no light | Wiring. Check polarity and the resistor |
| Timeout | Wrong IP, or the ESP32 crashed |

### C2. LED never lights, even with curl

- **Polarity.** The long leg (anode) goes toward the resistor and GPIO13; the short leg (cathode) to
  GND. Reversed, an LED simply does not conduct
- **Resistor.** 220Ω is right. 10kΩ makes it almost invisible
- **Wrong pin.** Confirm you are on GPIO13, not GPIO12
- **Dead LED.** Test it directly across 3.3V and GND with the resistor

### C3. LED flickers rapidly

Debouncing is not working. Confirm `PERSON_ON_FRAMES` and `PERSON_OFF_FRAMES` are set and that
`send_person_command` has the early-return check.

### C4. Relay clicks randomly at boot

`pinMode()` and the initial `setPersonOutput(false)` must run **before** the camera and Wi-Fi init.
Also: most relay modules are **active-LOW** — LOW energises them. If yours is, invert the logic.

---

## §D — Python environment

### D1. `'python' is not recognized` / `command not found`

PATH problem. Reinstall Python with **"Add python.exe to PATH"** ticked, or use `python3`.
**Open a new terminal afterwards.**

### D2. `pip is not recognized`

```bash
python -m ensurepip --upgrade
python -m pip install ...      # this form always works
```

### D3. VS Code shows red squiggles under imports but the code runs

Wrong interpreter selected. `Ctrl+Shift+P` → **Python: Select Interpreter** → pick the `('venv')` one.

### D4. `running scripts is disabled on this system`

PowerShell as Administrator, once:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### D5. `ERROR: Could not find a version that satisfies the requirement torch`

You are on Python 3.13+. Install Python 3.12 and recreate the venv.

### D6. `ModuleNotFoundError: No module named 'cv2'`

The venv was not active when you installed. Check for `(venv)` in your prompt, activate, reinstall.

### D7. Install is extremely slow or fails partway

PyTorch is 1–2 GB. On a slow connection:

```bash
pip install --timeout 120 -r requirements.txt
```

Or install the CPU-only PyTorch build, which is much smaller:

```bash
pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
pip install ultralytics opencv-python numpy requests
```

### D8. Firewall blocking Python

Windows Defender often prompts on first run. If you clicked "Cancel", Python is blocked:
**Windows Security → Firewall & network protection → Allow an app through firewall** → find Python →
tick **Private**.

---

## §E — Runtime errors

### E1. Black, grey, or frozen OpenCV window

| Cause | Fix |
|---|---|
| `opencv-python-headless` installed | `pip uninstall opencv-python-headless -y && pip install opencv-python` |
| `waitKey()` not called | It must run every loop iteration, including the skip path |
| No frames arriving | Check the stream in a browser first |
| WSL / headless Linux | No display server. Use headless mode, or configure X forwarding |

### E2. Connection timeout on start

Work down the [diagnostic ladder](#the-60-second-diagnostic-ladder). Most likely: wrong `ESP_IP`, or
a browser tab holding the stream.

### E3. `cv2.error: !_src.empty() in function 'imdecode'`

Corrupt JPEG data. The `if frame is not None` check should catch it — confirm it is present.

### E4. `RuntimeError: Couldn't load custom C++ ops`

PyTorch and torchvision versions do not match:

```bash
pip uninstall torch torchvision -y
pip install torch torchvision
```

### E5. Program hangs on Ctrl+C

The reader thread is not a daemon. Confirm `daemon=True`.

### E6. `AttributeError: 'NoneType' object has no attribute 'copy'`

`latest_frame` is still `None` — no frame has arrived yet. The
`if current_frame is None ... continue` guard should prevent this.

---

## §F — Performance

### F1. Low FPS

See [Performance Tuning](#performance-tuning). Quick wins in order:
close other apps → plug in the laptop → `imgsz=256` → export to ONNX → use a GPU.

### F2. High latency (video lags behind reality)

If latency *grows over time*, frames are queueing — confirm the reader overwrites `latest_frame`
rather than appending to a list.

### F3. Memory usage climbing

`jpeg_buffer` is not being trimmed. Confirm `jpeg_buffer = jpeg_buffer[end + 2:]` runs after every
extracted frame.

---

## §G — Detection quality

### G1. Not detecting obvious people

- Lower `YOLO_CONFIDENCE` to `0.15`
- Raise `YOLO_IMAGE_SIZE` to `416` or `640`
- Improve lighting — YOLO needs to see
- Try `yolo11s.pt`
- Check the camera is in focus (the OV2640 lens screws in and out)

### G2. False positives on furniture, coats, shadows

- Raise `YOLO_CONFIDENCE` to `0.40`
- Raise `PERSON_ON_FRAMES` to `4`
- Reposition the camera to exclude the offending object

### G3. Detection flickers on and off

Raise `PERSON_OFF_FRAMES`. Or switch to the
[time-based debounce](#time-based-alternative) so behaviour is independent of FPS.

### G4. Works close up, fails at a distance

Distant people occupy few pixels. Raise `YOLO_IMAGE_SIZE` to `640` and `frame_size` to
`FRAMESIZE_VGA`. Expect roughly half the FPS.

---

# Security Notes

> [!WARNING]
> Three things about this project are **not production-safe**. Know exactly what they are.

**1. Wi-Fi credentials must never be committed to Git.**
Putting your SSID and password directly in the `.ino` means anyone who clones your public repo gets
your home Wi-Fi password — permanently, because Git keeps history forever. Use `esp32/secrets.h`
(already in `.gitignore`) and commit only `secrets.example.h`. **If you have already pushed real
credentials, change your Wi-Fi password now** — deleting the file is not enough.

**2. The HTTP endpoints have no authentication.**
Anyone on your local network can `curl http://<esp-ip>/person_on` and toggle your GPIO. Fine for a
lab bench. Not fine for a door lock. Add a
[shared token](#13-known-issues-and-suggested-improvements), or keep the device on an isolated VLAN.

**3. The video stream is unencrypted HTTP.**
Anyone on the same network can watch it. Do not point this camera anywhere private on an untrusted
network.

**Before every push:** confirm `git status` shows no `secrets.h`, no `venv/`, and no `*.pt`.

---

# Glossary

**AGPL-3.0** — The licence on Ultralytics' YOLO weights. More restrictive than MIT, with specific
obligations for network-deployed and commercial use.

**Baud rate** — Signal changes per second on a serial link. Both ends must agree (here: 115200).

**BGR** — Blue-Green-Red channel order. OpenCV's convention, unlike almost everything else's RGB.

**Bounding box** — Four numbers describing a rectangle around a detected object.

**Brownout** — A supply voltage dip below the chip's minimum, causing a reset.

**Chunked transfer encoding** — HTTP mechanism for sending a body of unknown total length.

**COCO** — *Common Objects in Context*. The 80-class dataset YOLO11 is pretrained on.

**Confidence** — The model's internal 0.0–1.0 score for a detection. Not a probability of being right.

**Convolution** — Sliding a small filter across an image to detect a local pattern. The core
operation of vision networks.

**CORS** — *Cross-Origin Resource Sharing*. Browser rule about which sites may call which servers.

**CRLF** — Carriage Return + Line Feed (`\r\n`). HTTP's required line ending.

**Daemon thread** — A thread that does not keep the program alive; killed when the main thread exits.

**DHCP** — The protocol by which your router assigns IP addresses. Leases expire, so addresses change.

**DMA** — *Direct Memory Access*. Hardware moving data into RAM without CPU involvement.

**Epoch** — One complete pass through the training dataset.

**ESP-IDF** — Espressif's native C framework. The Arduino ESP32 core is built on top of it.

**f-string** — Python's `f"text {variable}"` formatting syntax.

**Frame buffer (`fb`)** — Memory holding one captured image. Must be returned to the driver after use.

**FreeRTOS** — The real-time operating system running on the ESP32, providing preemptive tasks.

**GPIO** — *General Purpose Input/Output*. A pin the chip can drive HIGH/LOW or read.

**Hysteresis** — Different thresholds for switching on and off, preventing rapid oscillation.

**Inference** — Running a trained model on new data to get a prediction.

**IoU** — *Intersection over Union*. Overlap measure between two boxes, 0.0 to 1.0.

**JPEG** — Compressed image format. Starts with bytes `FF D8`, ends with `FF D9`.

**LEDC** — The ESP32's PWM peripheral. Repurposed here to generate the camera's 20 MHz clock.

**MJPEG** — *Motion JPEG*. Video as a sequence of complete, independent JPEG frames.

**mAP** — *mean Average Precision*. The standard object-detection accuracy metric.

**NMS** — *Non-Maximum Suppression*. Merges overlapping boxes describing the same object.

**NumPy `ndarray`** — A fast multi-dimensional array. How images are represented in Python.

**ONNX** — *Open Neural Network Exchange*. A portable model format with an optimised runtime.

**OV2640** — The 2-megapixel camera sensor on the ESP32-CAM.

**Overfitting** — When a model memorises its training data instead of learning general patterns.

**PATH** — The list of folders your OS searches when you type a command.

**PSRAM** — External RAM on some ESP32 modules. Enables higher resolutions and double-buffering.

**QVGA** — 320 × 240 pixels.

**Race condition** — A bug where the outcome depends on unpredictable thread timing.

**Session (`requests`)** — A reusable HTTP connection pool. Much faster than one-off requests.

**Strapping pin** — A pin read once at boot to configure chip behaviour. GPIO0 and GPIO12 here.

**Tensor** — A multi-dimensional array, the fundamental data type in PyTorch.

**Transfer learning** — Starting from pretrained weights instead of random ones. Why 300 images can
be enough.

**venv** — Virtual environment. An isolated per-project Python installation.

**YOLO** — *You Only Look Once*. A single-pass object detection architecture.

---

# FAQ

<details>
<summary><b>Why can't the ESP32 run YOLO by itself?</b></summary>

It has ~520 KB of internal RAM and no floating-point neural accelerator. YOLO11n needs tens of
megabytes of RAM and billions of arithmetic operations per frame. It does not fit. Tiny models *can*
run on an ESP32 (see ESP-DL or TensorFlow Lite Micro), but they are far less accurate and usually
limited to one narrow task like face detection.
</details>

<details>
<summary><b>Does this work over the internet, or only on my local Wi-Fi?</b></summary>

Local network only, by design. Python connects to a private IP like `192.168.x.x`, unreachable from
outside your router. Remote access needs port forwarding or a VPN — and you should **not** expose an
unauthenticated camera to the public internet.
</details>

<details>
<summary><b>Do I need an NVIDIA GPU?</b></summary>

No. The script sets `device="cpu"` and uses the smallest YOLO11 model at reduced resolution precisely
so it runs on ordinary laptops. A GPU makes it faster and changes nothing else.
</details>

<details>
<summary><b>Why does it detect people but not my dog / bottle / chair?</b></summary>

One line: `classes=[PERSON_CLASS_ID]`. YOLO is told to report only class 0. Delete that argument and
it reports all 80 COCO classes.
</details>

<details>
<summary><b>Why two frames to turn ON but eight to turn OFF?</b></summary>

Asymmetric hysteresis. Turning on fast feels responsive. Turning off slowly means a single missed
frame — which happens constantly when someone turns sideways or is briefly occluded — does not make
the LED flicker.
</details>

<details>
<summary><b>The FPS is only 3. Is something broken?</b></summary>

Probably not. 3 FPS is normal for an older CPU. Lower `YOLO_IMAGE_SIZE` to `256`, close Chrome, and
make sure you are not on a battery-saver power profile.
</details>

<details>
<summary><b>Can I connect two ESP32-CAMs?</b></summary>

Yes, but you need one reader thread and one YOLO call per camera, and your FPS will roughly halve.
Get one working perfectly first.
</details>

<details>
<summary><b>Can I use a different board (M5Stack, TTGO, ESP32-S3-EYE)?</b></summary>

Yes — but the `#define ..._GPIO_NUM` block is specific to the AI Thinker pin map. Replace it with
your board's map. `camera_pins.h` from the official Arduino ESP32 examples has maps for most boards.
</details>

<details>
<summary><b>Where does yolo11n.pt get downloaded to?</b></summary>

Your current working directory, on first run. ~5 MB. It is in `.gitignore` — do not commit model
weights.
</details>

<details>
<summary><b>Is my video being sent to a company or the cloud?</b></summary>

No. Everything runs locally. The only internet access is the one-time model download from
Ultralytics' servers.
</details>

<details>
<summary><b>Can I run this on a Raspberry Pi instead of a laptop?</b></summary>

Yes. A Pi 4 or 5 manages 3–8 FPS at `imgsz=320`. Export to ONNX first for a meaningful speedup. A Pi
Zero is too slow to be useful.
</details>

---

# Future Improvements

- [ ] **Multi-class output** — one GPIO pin per detected class
- [ ] **Person counting** — GPIO HIGH only when *N or more* people are present
- [ ] **Zone detection** — trigger only inside a drawn region
- [ ] **Recording** — auto-save a clip on each detection
- [ ] **Telegram / MQTT alerts** — push a notification with a snapshot
- [ ] **Web dashboard** — Flask or FastAPI UI showing the stream and detection log
- [ ] **On-device inference** — port a tiny model to ESP-DL so the ESP32 works standalone
- [ ] **ONNX / OpenVINO export** — 2–3× CPU speedup with no GPU
- [ ] **Object tracking** — persistent IDs so the same person is not re-counted
- [ ] **Custom-trained model** — see [Training](#training-yolo-on-your-own-objects)
- [ ] **mDNS auto-discovery** — `esp32cam.local`, so you never hunt for the IP again
- [ ] **Authentication token** on the control endpoints
- [ ] **Wi-Fi timeout + auto-reconnect** in the firmware
- [ ] **PSRAM detection** for double-buffering

---

# Replacing the Placeholders

| Placeholder | Appears in | Replace with |
|---|---|---|
| `YOUR_ESP32_CODE.ino` | This README, `esp32/` | e.g. `person_detector.ino` |
| `YOUR_YOLO_SCRIPT.py` | This README, `python/` | e.g. `detect_person.py` |
| `YOUR_ESP32_IP` | This README, commands | e.g. `192.168.0.172` |
| `YOUR_WIFI_NAME` | `secrets.example.h` | Your SSID |
| `YOUR_WIFI_PASSWORD` | `secrets.example.h` | Your password |

## Replace them all at once

**macOS / Linux:**

```bash
cd esp32-cam-yolo11-person-detection

grep -rl "YOUR_ESP32_CODE.ino" . | xargs sed -i '' 's/YOUR_ESP32_CODE\.ino/person_detector.ino/g'
grep -rl "YOUR_YOLO_SCRIPT.py"  . | xargs sed -i '' 's/YOUR_YOLO_SCRIPT\.py/detect_person.py/g'
```

> On **Linux**, drop the `''` after `-i`: `sed -i 's/.../.../g'`

**Windows PowerShell:**

```powershell
Get-ChildItem -Recurse -Include *.md,*.txt |
  ForEach-Object {
    (Get-Content $_) `
      -replace 'YOUR_ESP32_CODE\.ino','person_detector.ino' `
      -replace 'YOUR_YOLO_SCRIPT\.py','detect_person.py' |
    Set-Content $_
  }
```

**Rename the actual files** (use `git mv` if already tracked, so history is preserved):

```bash
git mv esp32/YOUR_ESP32_CODE.ino esp32/person_detector.ino
git mv python/YOUR_YOLO_SCRIPT.py python/detect_person.py
```

> [!IMPORTANT]
> **Arduino requires the sketch folder name to match the `.ino` filename.** If you rename the sketch
> to `person_detector.ino`, Arduino IDE will want it in a folder called `person_detector/`. Either
> accept the IDE's offer to create that folder, or keep the sketch in its own subfolder:
> `esp32/person_detector/person_detector.ino`.

**Verify nothing was missed:**

```bash
grep -r "YOUR_" . --exclude-dir=.git --exclude-dir=venv
```

Should return nothing except this section.

---

# Appendix — Supporting Files

Copy these into your repository. They are the only other files you need.

## `requirements.txt`

```
# ESP32-CAM + YOLO11 Person Detection - Python dependencies
#
# Install with:   pip install -r requirements.txt
# Python 3.10-3.12 required. 3.13+ may not have PyTorch wheels yet.

# YOLO11 object detection. Pulls in torch + torchvision automatically (~1-2 GB).
ultralytics>=8.3.0

# Image decoding and display.
# NEVER install opencv-python-headless alongside this - imshow() will break.
opencv-python>=4.8.0

# Fast numeric arrays. Images are arrays.
numpy>=1.24.0

# HTTP client - the video stream and the GPIO commands.
requests>=2.31.0

# --- Optional: NVIDIA GPU ---
# The default torch is CPU-only. For a CUDA build:
#   pip uninstall torch torchvision -y
#   pip install torch torchvision --index-url https://download.pytorch.org/whl/cu121
# Then set device="0" in the script.

# --- Optional: smaller CPU-only install ---
#   pip install torch torchvision --index-url https://download.pytorch.org/whl/cpu
```

## `.gitignore`

```gitignore
# ============ SECRETS - NEVER COMMIT ============
esp32/secrets.h
*.secret
.env

# ============ PYTHON ============
venv/
env/
ENV/
.venv/
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
dist/
*.egg-info/

# ============ MODEL WEIGHTS ============
# Re-downloadable. Do not bloat the repo.
*.pt
*.onnx
*.engine
*.tflite
*.mlmodel
!requirements.txt

# ============ TRAINING OUTPUT ============
runs/
datasets/
dataset/
wandb/

# ============ RECORDINGS / CAPTURES ============
*.avi
*.mp4
*.mkv
detect_*.jpg
detections.csv

# ============ ARDUINO ============
build/
*.hex
*.elf
*.bin
.vscode/arduino.json
.vscode/c_cpp_properties.json

# ============ EDITORS ============
.vscode/*
!.vscode/settings.json
.idea/
*.swp
*.swo
*~

# ============ OS ============
.DS_Store
Thumbs.db
desktop.ini

# ============ LOGS ============
*.log
```

> [!WARNING]
> **`.gitignore` only affects files Git is not already tracking.** If you committed `secrets.h`
> before adding the line, adding it changes nothing:
> ```bash
> git rm --cached esp32/secrets.h
> git commit -m "Remove secrets from tracking"
> ```
> And if it was ever pushed publicly: **change your Wi-Fi password.**

## `esp32/secrets.example.h`

```cpp
// ============================================================
// secrets.example.h
//
//   1. Copy this file and rename the copy to  secrets.h
//   2. Fill in your real Wi-Fi credentials in secrets.h
//   3. Never commit secrets.h  (it is already in .gitignore)
//
// This example file IS committed, so other people know what
// they need to create.
// ============================================================

#pragma once

// The ESP32 radio is 2.4 GHz ONLY. It cannot see a 5 GHz network.
// Watch for trailing spaces - "MyWiFi " and "MyWiFi" are different
// networks as far as the ESP32 is concerned.
// Captive-portal networks (hotel, university) will not work.

#define SECRET_WIFI_SSID      "YOUR_WIFI_NAME"
#define SECRET_WIFI_PASSWORD  "YOUR_WIFI_PASSWORD"

// Optional: shared secret for the GPIO control endpoints.
// The endpoints have no authentication by default. If you implement
// the token check, call them as:  /person_on?key=YOUR_TOKEN
#define SECRET_API_KEY        "change-me"
```

## `.vscode/settings.json` (optional)

```json
{
  "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python",
  "python.terminal.activateEnvironment": true
}
```

On Windows: `"${workspaceFolder}/venv/Scripts/python.exe"`

## Command cheat sheet

```bash
# --- Environment ---
python -m venv venv                  # create
venv\Scripts\activate                # activate (Windows)
source venv/bin/activate             # activate (macOS/Linux)
deactivate                           # leave

# --- Packages ---
pip install -r requirements.txt      # install everything
pip list                             # show installed
pip freeze > requirements.txt        # record exact versions
pip install --upgrade ultralytics    # update YOLO

# --- Run ---
python python/YOUR_YOLO_SCRIPT.py    # <PLACEHOLDER>

# --- Diagnostics ---
python --version
pip --version
python -c "import cv2; print(cv2.__version__)"
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
ping YOUR_ESP32_IP
curl http://YOUR_ESP32_IP/person_on
curl http://YOUR_ESP32_IP/person_off
curl -I http://YOUR_ESP32_IP:81/stream

# --- YOLO CLI ---
yolo export model=yolo11n.pt format=onnx
yolo train model=yolo11n.pt data=data.yaml epochs=100 imgsz=320
yolo predict model=yolo11n.pt source=0        # test with your webcam
```

---

# Contributing

Contributions are welcome — especially documentation fixes, since the whole point is that a beginner
can follow this.

1. Fork the repository
2. `git checkout -b feature/your-idea`
3. `git commit -m "Add: your idea"`
4. `git push origin feature/your-idea`
5. Open a Pull Request describing **what** changed and **why**

**Before you push:** confirm `git status` shows no `secrets.h`, no `venv/`, and no `*.pt` files.

Found a step that did not work for you? Open an issue with your OS, Python version, and the exact
error text.

---

# License

Released under the **MIT License**. Use, modify, and distribute freely, including commercially.
Provided without warranty of any kind.

> [!NOTE]
> The **YOLO11 model weights** from Ultralytics are distributed under **AGPL-3.0**, which is separate
> from and more restrictive than MIT, with specific obligations for network-deployed and commercial
> use. Review [Ultralytics licensing](https://www.ultralytics.com/license) before shipping a product.

---

# Acknowledgements

- **[Ultralytics](https://github.com/ultralytics/ultralytics)** — the YOLO11 models and training framework
- **[Espressif](https://github.com/espressif/arduino-esp32)** — ESP32 Arduino core and `esp_camera` driver
- **[OpenCV](https://opencv.org/)** — image decoding and display
- **[COCO dataset](https://cocodataset.org/)** — the 80-class dataset YOLO11 is pretrained on

---

<div align="center">

**If this helped you, a ⭐ costs nothing and helps other beginners find it.**

</div>
