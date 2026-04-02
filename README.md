# 🤖 R.A.S.C.A.L  
### Robotic Autonomous Servant for Courier & Logistics

## 🚀 Overview
RASCAL is an autonomous mobile robot designed for last-mile delivery applications in courier and logistics environments.

The goal of this project is to develop a robot capable of navigating real-world environments, avoiding obstacles, and delivering packages efficiently with minimal human intervention.


https://github.com/user-attachments/assets/97637182-ccf0-4386-8f26-bfa4046b3d35




---

## What This Robot Does

RASCAL is a four-wheeled autonomous ground robot capable of two modes of operation.

**Manual mode** — controlled in real time via a custom mobile app over Bluetooth. Directional commands, speed adjustment, live status feedback.

**Autonomous mode** — given a GPS coordinate via the app, the robot navigates to it independently. It calculates bearing using the Haversine formula, tracks heading error in real time, and corrects course continuously until arrival within a 3-metre threshold.

Both modes were demonstrated in real outdoor conditions. This is not a simulation. The robot moves.

---

## Vision

Most robotics ideas begin in controlled environments. This one didn’t.

RASCAL started as a robot built to operate directly in the real world, moving on actual roads, communicating with a phone, and navigating autonomously to a destination. No simulations, no staged demos — just a system designed and tested in live conditions from the beginning.

This prototype represents the first step toward a broader vision: a platform of companion robots designed for environments where presence matters — homes, hotels, and campuses. The focus is not just on functionality, but on creating systems that feel intuitive, approachable, and reliable in everyday human spaces.

This repository documents Prototype 1 — the foundation on which everything else is being built.

---

## System Architecture

The control stack is split across three layers: perception, decision, and actuation.

![bb7f535c-c80d-4982-a2fe-2df3a2539d58](https://github.com/user-attachments/assets/8c5b2ec1-d08b-46e8-8ef1-87fa795cc6d2)


---

## Navigation Logic

The robot uses GPS-guided navigation with continuous heading correction. No SLAM, no map — pure geometry and real-time feedback.

**Haversine distance:**
```
d = 2R · arcsin(√(sin²(Δlat/2) + cos(lat₁)·cos(lat₂)·sin²(Δlon/2)))
```

**Bearing to target:**
```
θ = atan2(sin(Δlon)·cos(lat₂), cos(lat₁)·sin(lat₂) − sin(lat₁)·cos(lat₂)·cos(Δlon))
```

**Heading correction loop** (1 Hz, GPS-limited):
```
error = bearing_to_target − current_heading  [normalized −180° to +180°]

|error| < 15°  →  moveForward()
error  > 15°   →  turnRight()
error  < −15°  →  turnLeft()
distance < 3m  →  STOP  (waypoint reached)
```

This is intentionally lean — Prototype 2 replaces this with a full ROS navigation stack and LiDAR. But this loop ran in the field, and it works.

---

## Hardware

| Component | Part | Role |
|---|---|---|
| Compute | 32-bit embedded MCU (72MHz, hardware UART) | Navigation logic, motor control, sensor fusion |
| GPS | u-blox NEO-6M | Position fix at ±2.5m CEP, up to 5 Hz |
| Wireless | HC-05 Bluetooth | Bidirectional app ↔ robot command bridge |
| Drive | 4× TT DC gear motors (6V, ~200 RPM) | 4WD differential drive |
| Motor drivers | 2× L298N dual H-bridge | PWM speed + direction control |
| Power | 7.4V 2S Li-Po, 2200mAh | ~1.5 hrs continuous / ~3 hrs delivery duty cycle |
| Regulation | 2× LM2596 buck converters | Isolated 5V rails for compute and sensors |
| Structure | Custom SolidWorks chassis | Designed for modularity and field serviceability |

**Total BOM cost (India-sourced): ₹5,000 – ₹7,000**

Full parts list with suppliers in [`B.O.M`](./B.O.M).

---

## Communication Protocol

Commands flow over Bluetooth UART at 9600 baud.

| Command | Function |
|---|---|
| `FWD` / `BWD` | Move forward / backward |
| `LEFT` / `RIGHT` | In-place tank turn |
| `STOP` | Halt all motors |
| `GOTO:LAT,LON` | Initiate autonomous navigation to coordinate |
| `CLEAR` | Cancel active navigation |
| `SPEED:0–255` | Adjust PWM output |
| `POS` | Request full status dump |

Status is broadcast every 2 seconds in autonomous mode:
```
STATUS:AUTO|LAT,LON|SAT:8|HDG:245.3|WP:LAT,LON|DIST:15.7m
```

---

## Drive Behaviour

4WD tank-drive differential. Turns are in-place. No slip compensation in P1 — that's a P2 problem.

| Command | FL | FR | BL | BR |
|---|---|---|---|---|
| Forward | ↑ | ↑ | ↑ | ↑ |
| Backward | ↓ | ↓ | ↓ | ↓ |
| Turn Left | ↓ | ↑ | ↓ | ↑ |
| Turn Right | ↑ | ↓ | ↑ | ↓ |

Base speed: 200/255 PWM. Turn speed: 180/255. Top speed ~0.5 m/s.

---

## Performance

| Metric | Value |
|---|---|
| Waypoint accuracy | ±3 metres |
| Navigation update rate | 1 Hz (GPS-limited) |
| Bluetooth range | ~10 metres line-of-sight |
| Command latency | < 100ms |
| Operating range | ~1.5 hrs continuous movement |

---

## What Building This Actually Taught Us


https://github.com/user-attachments/assets/ead47679-252b-4ddd-95c1-c096aacddda8


Three things that specs don't tell you:

**GPS noise is not Gaussian.** The fix drifts in bursts. Our 3m arrival threshold exists because of this — not as a design choice, but as an honest acknowledgment of sensor reality.

**Motor symmetry is a myth at low cost.** Four nominally identical motors run at different effective speeds under load. Heading drift accumulates. You correct it in the navigation loop or you don't correct it at all.

**System integration is the real engineering.** The Haversine formula took an hour. Getting it to behave predictably while Bluetooth, GPS UART, and four PWM channels compete for timing cycles took weeks.

These aren't clean solutions. They're constraints we understood — and that understanding is exactly what Prototype 2 is built on.

---

## Prototype 2 — In Progress

| Capability | P1 (This repo) | P2 (In Progress) |
|---|---|---|
| Compute | Embedded MCU | Raspberry Pi 4 |
| Localisation | GPS only | GPS + IMU + wheel odometry |
| Environment sensing | None | RPLidar A1 |
| Mapping | None | SLAM (GMapping / Cartographer) |
| Framework | Bare-metal C++ | ROS 2 |
| Obstacle avoidance | None | Dynamic Window Approach |
| Navigation | Haversine + heading correction | Full ROS Nav Stack |

ROS 2 node graph for P2:
```
/gps_node       ─┐
/lidar_node     ─┤─► /localisation_node ──► /navigation_node ──► /motor_bridge
/imu_node       ─┘
```

---

## Repository Contents

| File | Contents |
|---|---|
| `RASCAL_main` | Full embedded firmware |
| `Control_Architect` | System architecture, state machine, algorithm detail |
| `B.O.M` | Complete bill of materials with India-sourced pricing |
| `Wiring_Schematic` | Pin assignments and power distribution diagram |
| `App_protocol` | Full Bluetooth command and status protocol spec |
| `App_Setup_guide` | Pairing and operation guide |
| `App_readMe` | Mobile app overview |
| `Introducing RASCAL.mp4` | Live prototype demonstration |
| `VibeCoding.mp4` | Build process footage |

---

## Builder

**Hetav Shah** — Mechanical Engineer · Roboticist

Designed and built the full stack end-to-end: mechanical chassis (SolidWorks), embedded firmware (C++), GPS navigation algorithms, sensor integration, and mobile app — independently.

[Portfolio](https://shahhetav.my.canva.site/) · [LinkedIn](https://linkedin.com/in/hetav--shah) · [hetav.ubs@gmail.com](mailto:hetav.ubs@gmail.com)

---

*RASCAL is open hardware. Schematics, firmware, BOM, and full documentation are available in this repository.*
