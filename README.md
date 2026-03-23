# 🌱 AGROVER MR 25.26 — Autonomous Agricultural Rover

> **Design, evolution and integration of an autonomous rover for precision agricultural monitoring and action**

**ISEN4 MR Project — JUNIA** | September 2025 – April 2026 | 

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Team](#-team)
- [System Architecture](#-system-architecture)
- [Hardware Requirements](#-hardware-requirements)
- [Installation & Setup](#-installation--setup)
  - [1. Arduino IDE + OpenCR](#1-arduino-ide--opencr)
  - [2. Dynamixel Motor Configuration](#2-dynamixel-motor-configuration)
  - [3. Raspberry Pi + ROS2](#3-raspberry-pi--ros2)
  - [4. Web Interface (Flask HMI)](#4-web-interface-flask-hmi)
- [Low-Level Control (OpenCR)](#-low-level-control-opencr)
  - [Keyboard Teleoperation](#keyboard-teleoperation)
  - [Sensor Readings](#sensor-readings)
  - [Pump Control](#pump-control)
- [High-Level Control (Raspberry Pi + ROS2)](#-high-level-control-raspberry-pi--ros2)
  - [System Startup](#system-startup)
  - [Autonomous Navigation (SLAM)](#autonomous-navigation-slam)
  - [ROS2 Teleoperation](#ros2-teleoperation)
- [Human-Machine Interface (HMI Web)](#-human-machine-interface-hmi-web)
- [Use Cases](#-use-cases)
- [Code Structure](#-code-structure)
- [Communication Protocols](#-communication-protocols)
- [Troubleshooting](#-troubleshooting)
- [Contact](#-contact)

---

## 🤖 Project Overview

AGROVER is a 4-wheel autonomous agricultural rover developed as part of the ISEN4 MR project at JUNIA. It enables farmers to remotely monitor their fields by collecting environmental data (temperature, humidity) and performing targeted actions (spraying).

### Achieved Objectives

- ✅ Autonomous navigation with obstacle avoidance (LiDAR + SLAM)
- ✅ Secure teleoperation via Wi-Fi web interface
- ✅ Real-time environmental measurements (DHT11, encoders)
- ✅ Targeted spraying via peristaltic pump
- ✅ Web supervision interface (map, measurements, video, commands)
- ✅ CSV/GeoJSON data export
- ✅ Fail-safe mode (return to safe point on connection loss)


## 🏗 System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    WEB HMI (Flask)                          │
│              PC / Tablet via Wi-Fi                          │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTP / WebSocket
┌──────────────────────▼──────────────────────────────────────┐
│                  RASPBERRY PI 4                             │
│              ROS2 Foxy + ROS Bridge                         │
│         Navigation2 | SLAM | Flask Server                   │
│    ┌──────────┐  ┌──────────┐  ┌──────────────────────┐    │
│    │  LiDAR   │  │  Camera  │  │   SQLite / GeoJSON   │    │
│    │  (USB)   │  │  Pi v2   │  │    Data Storage      │    │
│    └──────────┘  └──────────┘  └──────────────────────┘    │
└──────────────────────┬──────────────────────────────────────┘
                       │ UART (115200 bps)
┌──────────────────────▼──────────────────────────────────────┐
│                    OPENCR 1.0                               │
│            Real-time low-level control                      │
│  ┌──────────────┐  ┌───────────┐  ┌──────────────────────┐ │
│  │ Dynamixel    │  │  DHT11    │  │  Peristaltic Pump    │ │
│  │ XL430-W250   │  │  PIN 7    │  │  Kamoer NKP (relay) │ │
│  │ ID: 1,2,3,4  │  │  T° / HR  │  │  Targeted spraying  │ │
│  └──────────────┘  └───────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Operating Modes

| Mode | Description |
|------|-------------|
| **MANUAL** | Teleoperation from HMI, limited speeds |
| **AUTONOMOUS** | Local + global navigation, obstacle avoidance, planned objectives |
| **SAFETY (Fail-safe)** | Emergency stop, geofencing, return to safe point |

---

## 🔧 Hardware Requirements

### Main Components

| Component | Reference | Role |
|-----------|-----------|------|
| Onboard Computer | Raspberry Pi 4 (4 GB RAM) | High-level control, ROS2, HMI |
| Control Board | OpenCR 1.0 (ROBOTIS) | Low-level motor and sensor control |
| Motors | Dynamixel XL430-W250 x4 | 4-wheel propulsion (daisy chain) |
| LiDAR | 2D LiDAR (USB) | Obstacle detection, SLAM mapping |
| Camera | Raspberry Pi Camera v2 | Vision, visual diagnosis |
| T°/HR Sensor | DHT11 | Air temperature and humidity |
| Pump | Kamoer NKP-DC-S10B (12V) | Targeted spraying |
| Battery | Li-Po 11.1V 5000 mAh | System power supply |
| Chassis | Modular metal (3D printed parts) | Mechanical structure |

### Sensor Wiring on OpenCR

```
DHT11    : VCC → 3.3V  |  GND → GND  |  DATA → PIN 7
Pump     : Signal → PIN 6 (via 12V relay)  |  VCC → 12V  |  GND → GND
Dynamixel: TTL ports (daisy chain ID 1→3, ID 2→4)
LiDAR    : USB → Raspberry Pi
Camera   : CSI ribbon → Raspberry Pi
```

### Motor Daisy Chain Configuration

```
OpenCR TTL Port 1 → Motor ID 1 (front left)  → Motor ID 3 (rear left)
OpenCR TTL Port 2 → Motor ID 2 (front right) → Motor ID 4 (rear right)
```

> **What is a Daisy Chain?**
> A wiring scheme where motors are connected in series on the same bus. The signal travels from the OpenCR to the first motor, then continues to the second. Each motor **must have a unique ID** for this to work correctly.

---

## 🚀 Installation & Setup

### 1. Arduino IDE + OpenCR

**Prerequisite:** Arduino IDE 1.8.19 (classic installer, **NOT** the Windows Store version)

> ⚠️ The Windows Store version stores files in OneDrive, which causes conflicts when installing third-party boards. Always use the classic Windows Installer.

Add this URL in Arduino IDE (`File → Preferences → Additional Boards Manager URLs`):
```
https://raw.githubusercontent.com/ROBOTIS-GIT/OpenCR/master/arduino/opencr_release/package_opencr_index.json
```

Then: `Tools → Boards Manager → Search "OpenCR" → Install`

Select board and port:
```
Tools → Board → OpenCR
Tools → Port  → COM7 (or whichever USB port appears)
```

**Required Libraries** (`Sketch → Include Library → Manage Libraries`):
- `DynamixelSDK` — ROBOTIS
- `DHT sensor library` — Adafruit
- `Adafruit Unified Sensor` — Adafruit

### 2. Dynamixel Motor Configuration

Each motor must have a **unique ID**. Connect **one motor at a time** and use the following code:

```cpp
#include <DynamixelSDK.h>

#define PROTOCOL_VERSION   2.0
#define BAUDRATE           1000000
#define DEVICE_NAME        "7"      // COM7 → "7"
#define ADDR_ID            7
#define ADDR_TORQUE_ENABLE 64

dynamixel::PortHandler   *portHandler;
dynamixel::PacketHandler *packetHandler;

void setup() {
  Serial.begin(57600);
  portHandler   = dynamixel::PortHandler::getPortHandler(DEVICE_NAME);
  packetHandler = dynamixel::PacketHandler::getPacketHandler(PROTOCOL_VERSION);
  portHandler->openPort();
  portHandler->setBaudRate(BAUDRATE);

  // Disable torque before changing ID
  packetHandler->write1ByteTxRx(portHandler, 1, ADDR_TORQUE_ENABLE, 0);

  // Change ID: first argument = current ID, last argument = new ID
  uint8_t error = 0;
  int result = packetHandler->write1ByteTxRx(portHandler, 1, ADDR_ID, 3, &error);
  if (result == COMM_SUCCESS)
    Serial.println("ID changed successfully!");
  else
    Serial.println("Failed to change ID.");
}
void loop() {}
```

**Final Motor IDs:**

| Motor | Position | ID |
|-------|----------|----|
| Front left  | Direct on OpenCR TTL 1 | 1 |
| Front right | Direct on OpenCR TTL 2 | 2 |
| Rear left   | Daisy chain from ID 1  | 3 |
| Rear right  | Daisy chain from ID 2  | 4 |

### 3. Raspberry Pi + ROS2

```bash
# Connect via SSH
ssh ubuntu@<RASPBERRY_PI_IP>

# Source ROS2 environment
source /opt/ros/foxy/setup.bash
source ~/agrover_ws/install/setup.bash

# Launch the full system
ros2 launch agrover_bringup agrover.launch.py
```

### 4. Web Interface (Flask HMI)

```bash
# On the Raspberry Pi
cd ~/agrover_ws/src/agrover_ihm
python3 app.py

# Access from any PC or tablet on the same Wi-Fi network
# Open browser → http://<RASPBERRY_PI_IP>:5000
```

---

## 🔌 Low-Level Control (OpenCR)

### Keyboard Teleoperation

Upload the code to the OpenCR and open the Serial Monitor at **57600 baud**.

```cpp
#include <DynamixelSDK.h>
#include <DHT.h>

#define PROTOCOL_VERSION    2.0
#define BAUDRATE            1000000
#define DEVICE_NAME         "7"
#define ADDR_OPERATING_MODE 11
#define ADDR_TORQUE_ENABLE  64
#define ADDR_GOAL_VELOCITY  104
#define ADDR_PRESENT_VELOCITY 128
#define ADDR_PRESENT_POSITION 132

#define DHT_PIN  7
#define DHT_TYPE DHT11
DHT dht(DHT_PIN, DHT_TYPE);

#define PUMP_PIN 6

// Motor IDs
#define DXL_FL 1   // Front left
#define DXL_FR 2   // Front right
#define DXL_RL 3   // Rear left
#define DXL_RR 4   // Rear right
#define SPEED  200

dynamixel::PortHandler   *portHandler;
dynamixel::PacketHandler *packetHandler;

void setVelocity(int fl, int fr, int rl, int rr) {
  packetHandler->write4ByteTxRx(portHandler, DXL_FL, ADDR_GOAL_VELOCITY, fl);
  packetHandler->write4ByteTxRx(portHandler, DXL_FR, ADDR_GOAL_VELOCITY, fr);
  packetHandler->write4ByteTxRx(portHandler, DXL_RL, ADDR_GOAL_VELOCITY, rl);
  packetHandler->write4ByteTxRx(portHandler, DXL_RR, ADDR_GOAL_VELOCITY, rr);
}

// Right-side motors use negative values (mirror-mounted)
void forward()   { setVelocity( SPEED, -SPEED,  SPEED, -SPEED); }
void backward()  { setVelocity(-SPEED,  SPEED, -SPEED,  SPEED); }
void turnLeft()  { setVelocity(-SPEED, -SPEED, -SPEED, -SPEED); }
void turnRight() { setVelocity( SPEED,  SPEED,  SPEED,  SPEED); }
void stopAll()   { setVelocity(0, 0, 0, 0); }

void readDHT() {
  float temp = dht.readTemperature();
  float hum  = dht.readHumidity();
  if (isnan(temp) || isnan(hum)) { Serial.println("DHT11 read error"); return; }
  Serial.print("Temp: "); Serial.print(temp); Serial.print(" C  |  Humidity: ");
  Serial.print(hum); Serial.println(" %");
}

void readEncoders() {
  int32_t vfl, vfr, vrl, vrr, pfl, pfr, prl, prr;
  packetHandler->read4ByteTxRx(portHandler, DXL_FL, ADDR_PRESENT_VELOCITY, (uint32_t*)&vfl);
  packetHandler->read4ByteTxRx(portHandler, DXL_FR, ADDR_PRESENT_VELOCITY, (uint32_t*)&vfr);
  packetHandler->read4ByteTxRx(portHandler, DXL_RL, ADDR_PRESENT_VELOCITY, (uint32_t*)&vrl);
  packetHandler->read4ByteTxRx(portHandler, DXL_RR, ADDR_PRESENT_VELOCITY, (uint32_t*)&vrr);
  packetHandler->read4ByteTxRx(portHandler, DXL_FL, ADDR_PRESENT_POSITION, (uint32_t*)&pfl);
  packetHandler->read4ByteTxRx(portHandler, DXL_FR, ADDR_PRESENT_POSITION, (uint32_t*)&pfr);
  packetHandler->read4ByteTxRx(portHandler, DXL_RL, ADDR_PRESENT_POSITION, (uint32_t*)&prl);
  packetHandler->read4ByteTxRx(portHandler, DXL_RR, ADDR_PRESENT_POSITION, (uint32_t*)&prr);
  Serial.print("Velocity FL:"); Serial.print(vfl); Serial.print(" FR:"); Serial.print(vfr);
  Serial.print(" RL:"); Serial.print(vrl); Serial.print(" RR:"); Serial.println(vrr);
  Serial.print("Position FL:"); Serial.print(pfl); Serial.print(" FR:"); Serial.print(pfr);
  Serial.print(" RL:"); Serial.print(prl); Serial.print(" RR:"); Serial.println(prr);
}

void setup() {
  Serial.begin(57600);
  dht.begin();
  pinMode(PUMP_PIN, OUTPUT);
  digitalWrite(PUMP_PIN, LOW);

  portHandler   = dynamixel::PortHandler::getPortHandler(DEVICE_NAME);
  packetHandler = dynamixel::PacketHandler::getPacketHandler(PROTOCOL_VERSION);
  portHandler->openPort();
  portHandler->setBaudRate(BAUDRATE);

  uint8_t ids[] = {DXL_FL, DXL_FR, DXL_RL, DXL_RR};
  for (int i = 0; i < 4; i++) {
    packetHandler->write1ByteTxRx(portHandler, ids[i], ADDR_TORQUE_ENABLE, 0);
    packetHandler->write1ByteTxRx(portHandler, ids[i], ADDR_OPERATING_MODE, 1);
    packetHandler->write1ByteTxRx(portHandler, ids[i], ADDR_TORQUE_ENABLE, 1);
  }
  Serial.println("AGROVER READY");
  Serial.println("z=forward s=backward q=left d=right x=stop");
  Serial.println("t=DHT11  e=encoders  p=pump ON  o=pump OFF");
}

void loop() {
  static unsigned long lastRead = 0;
  if (millis() - lastRead >= 5000) {
    readDHT();
    readEncoders();
    lastRead = millis();
  }
  if (Serial.available()) {
    char cmd = Serial.read();
    switch(cmd) {
      case 'z': forward();                                             break;
      case 's': backward();                                            break;
      case 'q': turnLeft();                                            break;
      case 'd': turnRight();                                           break;
      case 'x': stopAll();                                             break;
      case 't': readDHT();                                             break;
      case 'e': readEncoders();                                        break;
      case 'p': digitalWrite(PUMP_PIN, HIGH); Serial.println("Pump ON");  break;
      case 'o': digitalWrite(PUMP_PIN, LOW);  Serial.println("Pump OFF"); break;
    }
  }
}
```

**Keyboard Commands:**

| Key | Action |
|-----|--------|
| `z` | Move forward |
| `s` | Move backward |
| `q` | Turn left |
| `d` | Turn right |
| `x` | Stop all motors |
| `t` | Read DHT11 sensor |
| `e` | Read motor encoders |
| `p` | Pump ON |
| `o` | Pump OFF |

### Sensor Readings

| Sensor | Pin | Library | Precision |
|--------|-----|---------|-----------|
| DHT11 (T°) | PIN 7 | Adafruit DHT | ±2°C |
| DHT11 (HR) | PIN 7 | Adafruit DHT | ±5% RH |
| Encoders (velocity) | Register 128 | DynamixelSDK | — |
| Encoders (position) | Register 132 | DynamixelSDK | — |

### Pump Control

The Kamoer NKP 12V peristaltic pump is controlled via a **12V relay** on PIN 6.

**Spraying circuit:**
```
Reservoir → Inlet tube → PUMP → Outlet tube → Spray nozzle
                            ↑
              OpenCR PIN 6 → 12V Relay → 12V Power
```

---

## 🧠 High-Level Control (Raspberry Pi + ROS2)

### System Startup

```bash
# Terminal 1 — Rover bringup (motors, sensors, LiDAR, camera)
ros2 launch agrover_bringup agrover.launch.py

# Terminal 2 — Autonomous navigation with pre-built map
ros2 launch agrover_navigation navigation2.launch.py map:=~/maps/field.yaml

# Terminal 3 — Web HMI server
cd ~/agrover_ws/src/agrover_ihm && python3 app.py
```

### Autonomous Navigation (SLAM)

```bash
# Start SLAM mapping mode
ros2 launch agrover_slam slam_toolbox.launch.py

# Save the map once mapping is complete
ros2 run nav2_map_server map_saver_cli -f ~/maps/field

# Export map to GeoJSON format
python3 ~/agrover_ws/scripts/map_to_geojson.py ~/maps/field.yaml
```

**Main ROS2 Topics:**

| Topic | Message Type | Description |
|-------|-------------|-------------|
| `/cmd_vel` | geometry_msgs/Twist | Velocity commands |
| `/odom` | nav_msgs/Odometry | Rover odometry |
| `/scan` | sensor_msgs/LaserScan | LiDAR point cloud |
| `/sensors/dht11` | std_msgs/Float32MultiArray | Temperature & humidity |
| `/pump/control` | std_msgs/Bool | Pump ON/OFF command |
| `/camera/image_raw` | sensor_msgs/Image | Camera stream |
| `/map` | nav_msgs/OccupancyGrid | Navigation map |
| `/battery_state` | sensor_msgs/BatteryState | Battery level & status |

### ROS2 Teleoperation

```bash
export ROS_DOMAIN_ID=30
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

---

## 🖥 Human-Machine Interface (HMI Web)

```
URL: http://<RASPBERRY_PI_IP>:5000
```

Compatible with any modern browser (PC, tablet, smartphone). No installation required.

---

## 📋 Use Cases

| Use Case | Description | Acceptance Criteria |
|----------|-------------|---------------------|
| UC1 — Field Mapping | SLAM-based autonomous area mapping | Consistent map, zero collisions, ≤ 30 min |
| UC2 — Measurement Mission | Navigate to waypoints, acquire T°/humidity | All measurements timestamped, full log |
| UC3 — Targeted Spraying | Navigate to point, activate pump | Flow achieved, zone ≤ 20 cm, no leaks |
| UC4 — Teleoperation | Manual HMI control with video feedback | Latency < 1 s, e-stop functional |
| UC5 — Fail-safe | Detect Wi-Fi loss, return to safe point | Triggered < 2 s, incident logged |

---

## 📁 Code Structure

```
agrover_ws/
├── src/
│   ├── agrover_bringup/          # Full system launch files
│   ├── agrover_control/          # ROS2 motor control nodes
│   │   ├── opencr_bridge.py      # UART OpenCR ↔ ROS2 bridge
│   │   └── motor_controller.py   # 4-motor Dynamixel controller
│   ├── agrover_sensors/          # ROS2 sensor nodes
│   │   ├── dht11_publisher.py    # DHT11 data publisher
│   │   └── pump_controller.py    # Peristaltic pump controller
│   ├── agrover_navigation/       # Autonomous navigation stack
│   ├── agrover_ihm/              # Flask web HMI
│   └── agrover_slam/             # SLAM parameter files
├── arduino/
│   ├── rover_teleop/             # Low-level keyboard teleoperation
│   ├── rover_sensors/            # Sensor reading code
│   └── rover_full/               # Complete OpenCR firmware
├── maps/                         # Saved GeoJSON field maps
├── data/                         # Mission CSV data exports
└── docs/                         # Technical documentation
```

---

## 🔗 Communication Protocols

| Interface | Protocol | Baud Rate | Data Exchanged |
|-----------|----------|-----------|----------------|
| Raspberry Pi ↔ OpenCR | UART | 115,200 bps | Motor commands, sensor feedback |
| OpenCR ↔ Dynamixel | TTL Half-Duplex | 1,000,000 bps | Speed/position setpoints, encoders |
| Raspberry Pi ↔ HMI | HTTP / WebSocket | Wi-Fi | Telemetry, commands, video, map |
| Pump ↔ OpenCR | PWM 5V → 12V Relay | — | ON/OFF command |

---

## 🛠 Troubleshooting

### Motors do not respond
- Verify 12V power supply connected to OpenCR POWER port
- Check Dynamixel JST 3-pin connectors are firmly seated
- Use **Dynamixel Wizard 2.0** for full diagnostics

### Two motors have the same ID
- Connect **one motor at a time**, use the ID-change sketch (register address 7)
- Required unique IDs: **1, 2, 3, 4**

### DHT11 shows "Read Error"
- Wiring: DATA → PIN 7, VCC → 3.3V, GND → GND
- Wait 2 seconds after startup

### SSH connection fails
- Verify same Wi-Fi network — find IP with `hostname -I` on Raspberry Pi

### ROS2 topics not visible
```bash
export ROS_DOMAIN_ID=30
ros2 topic list
ros2 doctor
```



### 👤 Vinny Juniors — Embedded & Control Lead

📧 vinnyjuniors.ngon@student.junia.com
📍 Lille (59000), France
📞 +33 7 80 84 90 92
🔗 [linkedin.com/in/vinny-ngon-708253342](https://linkedin.com/in/vinny-ngon-708253342)

---

*Project carried out as part of the Mechatronics & Robotics program — JUNIA ISEN Lille*
*September 2025 – April 2026*