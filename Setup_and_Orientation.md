# K1 Setup & Orientation

Make sure you have read through the README and installed the base environment if you run into issues.
**Reference:** [Booster K1 Manual](https://booster.feishu.cn/wiki/E3q5wF5SnitXZgkY18Uc8odBnXb) — read through and keep handy as a reference while working with the K1.

---

## 1. Getting to Know the K1

### 1.1 Compute Platform (Jetson Orin NX)

| Spec | Value |
|------|-------|
| Processor | 6-core ARM Cortex-A78AE @ 2GHz |
| Memory | 8GB unified (CPU + GPU share) |
| GPU | 1024 CUDA cores + 32 Tensor cores |
| AI Performance | 117 TOPS |
| Storage | 512GB |
| Wired Network | Gigabit Ethernet |
| Wireless | Wi-Fi 6, Bluetooth 5.2 |
| Audio | Microphone array + speaker |

<img width="180" height="167" alt="image" src="https://github.com/user-attachments/assets/58b9d24d-e750-40b1-bb15-1afa11741867" />

### 1.2 Robot Specs

| Parameter | Value |
|-----------|-------|
| Height | 0.95 m |
| Weight | ~19.5 kg |
| Total DoFs | 22 (6 per leg, 4 per arm, 2 head) |
| Walking Speed | 1.1 m/s |
| Turning Speed | 1.5 rad/s |
| Battery | 5 Ah, ~1h 10min runtime, ≤2h charge |
| Camera | Depth camera |
--- 

<img width="468" height="533" alt="Image" src="https://github.com/user-attachments/assets/5b44dfe5-54c6-4384-ab54-25cf4f4241fa" />

---

<!-- Add: joint diagram with IDs, DOF labels, and zero position photo -->
<img width="446" height="444" alt="Image" src="https://github.com/user-attachments/assets/fca833c1-0f62-4666-8352-189486aa851c" />

<img width="201" height="225" alt="Image" src="https://github.com/user-attachments/assets/39986fa1-0ece-41f8-95a7-96b3e43aabf0" />

---

## 2. Operating Modes

The K1 operates in **Modes** and can perform different actions depending on the active mode. Modes can switch between each other but with constraints, DAMP can only go to WALK by passing through PREP first.

### DAMP Mode
- Joints are limp and resist position changes but don't hold or move actively
- Robot **cannot stand** — must be supported or seated
- Safe default: protects the robot and operator
- Can switch to: **PREP** only

### PREP Mode
- Robot assumes and holds a standing posture; joints strongly resist movement
- Can stand on its own but **will not balance itself** — do not push it
- Can switch to: **all other modes**

### WALK Mode
- Full locomotion: omni-walk, rotate, step, stand, move head
- More resilient than PREP — will actively try to recover balance if pushed
- Can switch to: **all other modes**
- ⚠️ **Make sure the robot is in PREP and STANDING firmly before switching to WALK**

### PROTECT Mode
- Triggered automatically on errors (joint limit exceeded, fall detected)
- Joints behave the same as DAMP
- Can attempt soft restart by re-entering DAMP mode

---

## 3. Safety

- K1 powers on in DAMP — support the robot before entering PREP
- **DO NOT lift K1 while in WALK mode** — risk of injury to you and the robot
- **DO NOT touch any part of K1 during WALK mode** except the handle
- Clear all ground obstacles before operating
- Never leave battery plugged in unsupervised

---

## 4. Power On / Off

### Powering On
1. Confirm K1 is resting safely and battery is fully charged and installed
2. Hold power button for 3 seconds — light will turn on
3. Hold the robot still while the Inertial Measurement Unit (IMU) initializes; a tone will play when ready
4. Enter PREP (while supporting or laying down): remote **LT + Start**, or press the **STAND** button on the back
5. Enter WALK: remote **RT + A**, or press the **WALK** button

### Powering Off
1. Return robot to **PREP mode**
2. Either lay it down on the floor or hold the handle while entering **DAMP mode**
3. Once resting safely, hold power button until the robot shuts down
4. Remove battery and charge it for the next session (Do not leave plugged in overnight)

---

## 5. Controls

- **Joystick controls:** [Booster Wiki](https://booster.feishu.cn/wiki/E3q5wF5SnitXZgkY18Uc8odBnXb#Ae7zdQ0jVoNBrHxULRLcVjHMnSd)
- **Mobile app:** Booster app available on iOS and Android
- Anything beyond the remote control or the mobile app requires the **Booster SDK**, which runs on Ubuntu 22.04 — see the [README](README.md) for environment setup

---

## 6. SSH Setup

SSH (Secure Shell) is a network protocol that lets you remotely access and control another computer (or robot) over a network. It works over either Wi-Fi or Ethernet, giving you a terminal session on the remote machine instead of needing physical access.
When you run a command such as `ssh booster@192.168.10.102` you authenticate with the remote machine — using a password or a key pair — and are placed into that machine's shell session. From then on, any commands you run execute on that machine.

### 6.1 Configure Your Workstation's Ethernet
1. Confirm the Ethernet cable is connected
2. Open network settings and find manual/static addressing
3. Set IP address: `192.168.10.10` (Allows for easier SSH)
4. Set subnet mask: `255.255.255.0`
5. Leave gateway blank
6. Apply

### 6.2 SSH into the Robot
```bash
ssh booster@192.168.10.102
```

<!-- Add: screenshot of successful SSH session -->

---

## 7. Verifying ROS 2

The K1 runs a **ROS 2 Humble** stack. Here we confirm it's running on the robot itself.

```bash
# After SSH-ing in:
source /opt/ros/humble/setup.bash
ros2 topic list
```

<!-- Add: screenshot of ros2 topic list output on the robot -->

