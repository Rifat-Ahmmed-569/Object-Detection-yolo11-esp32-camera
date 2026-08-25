# Object-Detection-yolo11-esp32-camera


For GitHub, add a **Prerequisites** section like this:

# Prerequisites

This project streams live video from an ESP32-CAM to a computer and performs real-time object detection using YOLO11 and OpenCV.

## Hardware Requirements

### ESP32-CAM (AI Thinker)

* ESP32-CAM AI Thinker module
* FTDI USB-to-Serial programmer
* Micro USB cable
* Stable 5V power supply

### Computer

* Windows 10/11, Linux, or macOS
* Minimum 4GB RAM (8GB recommended)
* Internet connection (required once to download YOLO model)

---

# ESP32-CAM Setup

## 1. Install Arduino IDE

Download and install Arduino IDE:

[https://www.arduino.cc/en/software](https://www.arduino.cc/en/software)

## 2. Install ESP32 Board Package

Open:

**File → Preferences**

Add the following URL to **Additional Boards Manager URLs**:

```text
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Then:

```text
Tools → Board → Boards Manager
```

Search:

```text
ESP32
```

Install:

```text
esp32 by Espressif Systems
```

---

## 3. Upload CameraWebServer Example

Open:

```text
File → Examples → ESP32 → Camera → CameraWebServer
```

Configure:

```cpp
#define CAMERA_MODEL_AI_THINKER
```

Enter your WiFi credentials:

```cpp
const char* ssid = "YOUR_WIFI_NAME";
const char* password = "YOUR_WIFI_PASSWORD";
```

Before uploading plug out the usb cable hold the reseat button, plug in the esp cam upload the sketch then release the reseat button. Now press the boot button. 

On Serial buad 115200 a ip will pop up then use the ip in your yolo code in vs studio, you can also test camera streaming by running the ip on your browser (but you have to stay in a single connected network)


---

## 4. Verify Camera Stream

After uploading, open Serial Monitor.

You should see:

```text
Camera Ready! Use 'http://192.168.x.x'
```

Open the IP address in your browser.

Test stream:

```text
http://ESP32_IP:81/stream
```

Example:

```text
http://192.168.0.172:81/stream
```

If you can see live video, the ESP32-CAM is configured correctly.

---

# Python Environment Setup

## 1. Install Python

Download Python 3.10+:

[https://www.python.org/downloads/](https://www.python.org/downloads/)

Verify installation:

```bash
python --version
```

Expected:

```text
Python 3.10+
```

---

## 2. Create Virtual Environment (Recommended)

Windows:

```bash
python -m venv venv
venv\Scripts\activate
```

Linux/macOS:

```bash
python3 -m venv venv
source venv/bin/activate
```

---

## 3. Install Required Libraries

Install all dependencies:

```bash
pip install ultralytics opencv-python numpy requests
```

Verify:

```bash
pip list
```

Expected packages:

```text
ultralytics
opencv-python
numpy
requests
torch
torchvision
```

---

# Required Python Imports

```python
import cv2
import requests
import numpy as np
from ultralytics import YOLO
```

Package purposes:

| Package          | Purpose                              |
| ---------------- | ------------------------------------ |
| OpenCV (cv2)     | Image decoding and display           |
| Requests         | Receive ESP32-CAM MJPEG stream       |
| NumPy            | Convert JPEG bytes into image arrays |
| Ultralytics YOLO | AI object detection                  |

---

# YOLO Model

The script automatically downloads:

```python
model = YOLO("yolo11n.pt")
```

YOLO11n is:

* Fast
* Lightweight
* Good for real-time detection
* Suitable for laptops without dedicated GPUs

The model downloads automatically during the first run.

---

# Running the Project

Update the camera IP:

```python
URL = "http://192.168.0.172:81/stream"
```

Run:

```bash
python object_detection.py
```

---

# Controls

| Key | Action       |
| --- | ------------ |
| Q   | Exit program |

---

# What Can Be Detected?

The default YOLO model can detect 80 COCO classes, including:

* Person
* Car
* Bus
* Bicycle
* Motorcycle
* Dog
* Cat
* Bird
* Laptop
* Cell Phone
* Bottle
* Chair
* TV
* Keyboard

and many more.

---

# System Architecture

```text
ESP32-CAM
    │
    │ MJPEG Stream
    ▼
WiFi Network
    │
    ▼
Python Application
    │
    ├── Requests
    ├── OpenCV
    ├── NumPy
    └── YOLO11
    │
    ▼
Object Detection Results
    │
    ▼
Live Display Window
```

---

# Troubleshooting

## Stream Not Opening

Verify:

```text
http://ESP32_IP:81/stream
```

works in a browser.

---

## Connection Timeout

Check:

* ESP32-CAM is powered properly
* Computer and ESP32 are on the same WiFi network
* IP address is correct

---

## YOLO Model Download Error

Run:

```bash
pip install ultralytics --upgrade
```

Ensure internet access is available during the first run.

---

## OpenCV Window Not Appearing

Install:

```bash
pip install opencv-python
```

Avoid using:

```bash
pip install opencv-python-headless
```

for desktop visualization.

---

# Expected Result

After successful setup, the system should:

1. Connect to the ESP32-CAM stream.
2. Receive live video frames.
3. Run YOLO object detection on each frame.
4. Draw bounding boxes and confidence scores.
5. Display detected objects in real time.
6. Exit cleanly when the user presses **Q**.
