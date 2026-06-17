# K1_Tutorial
Documentation for getting started with the Booster K1 humanoid robot — hardware overview, operating modes, SSH configuration, ROS 2 development, and the full simulation / RL stack. Written as a reference for researchers and developers picking up work on the K1.

**Guides:**
- [Setup & Orientation](Setup_and_Orientation.md) — hardware overview, operating modes, safety, power on/off, SSH, and ROS 2 verification
- [Simulation Setup](Simulation_Setup.md) — Isaac Sim / Isaac Lab (training) and MuJoCo (sim2sim), plus cloning the Booster repos
- [Head Camera — Live Video Feed](Head_Camera.md) — stream the head camera to any workstation over HTTP (the reliable `/booster_video_stream` path)


# Environment Setup

Run `install_all.sh` **once** on a fresh Ubuntu 22.04 machine to set up the base environment.
It takes approximately 20–40 minutes depending on your internet speed.

---

## Minimum Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Ubuntu 22.04 LTS | Ubuntu 22.04 LTS |
| RAM | 16 GB | 32 GB |
| GPU | NVIDIA CC 7.0+ (GTX 1080 / RTX 20xx) | RTX 3060 or better |
| VRAM | 6 GB | 8 GB+ |
| Disk space | 50 GB free | 100 GB free |
| Network | Gigabit Ethernet port | Gigabit Ethernet port |
| CPU | 4 cores | 8+ cores |

> **No NVIDIA GPU?** The script installs CPU-only PyTorch automatically.
> Most workflows (SSH, ROS 2, vision, voice) work fine without a GPU.
> RL / Isaac Lab training requires a GPU, but you can still run the
> pre-trained MuJoCo policy on CPU (slowly).

---

## How to Run

```bash
# 1. Download the script
wget https://raw.githubusercontent.com/Gavinw575/K1_Robot_Project/master/course/setup/install_all.sh

# 2. Make it executable
chmod +x install_all.sh

# 3. Run it (takes 20-40 minutes)
bash install_all.sh

# 4. Reload your shell environment
source ~/.bashrc
```

Or clone the whole repo first:

```bash
git clone https://github.com/Gavinw575/K1_Robot_Project ~/K1_Robot_Project
bash ~/K1_Robot_Project/course/setup/install_all.sh
source ~/.bashrc
```

---

## What Gets Installed

| Step | Packages | Purpose |
|------|----------|---------|
| 1 | `openssh-client`, `sshpass`, `net-tools`, `ffmpeg`, `portaudio19-dev`, `git` | SSH & Linux access |
| 1 | `ros-humble-desktop`, `ros-humble-rmw-fastrtps-cpp` | ROS 2 stack |
| 2 | `ros-humble-rviz2`, `ros-humble-tf2-tools`, `ros-humble-xacro` | Visualization |
| 3 | `booster_robotics_sdk` (built from source), `booster_robotics_sdk_ros2` | SDK & motion control |
| 4 | `ultralytics` (YOLOv8), `opencv-python`, `numpy` | YOLO vision |
| 5 | `openai-whisper`, `sounddevice`, `groq`, `piper-tts`, Piper danny-low model | Voice pipeline |
| 6 | `matplotlib`, `pandas` | Sensor fusion |
| 7 | `filterpy` (Kalman filter) | Ball tracking |
| 8 | `torch`, `torchvision`, `transformers`, `accelerate` | Reinforcement learning |
| 9 | `mujoco`, `booster_deploy` | Simulation (sim2sim) |
| 10 | FastDDS XML config, `~/.bashrc` environment variables | DDS / robot comms |
| † | Isaac Sim + Isaac Lab (`isaacsim`, `IsaacLab`) | RL training |

> **†** Isaac Sim + Isaac Lab are **not** installed by `install_all.sh` — they're large and GPU-only.
> Set them up separately via **[Simulation_Setup.md](Simulation_Setup.md)**. MuJoCo (step 9) is the
> lightweight sim2sim engine and *is* installed by the base script.

**Disk usage (approximate):**
- ROS2 Humble desktop: ~2.5 GB
- PyTorch (CUDA): ~5 GB
- Whisper base model (downloaded on first use): ~150 MB
- YOLOv8n model (downloaded on first use): ~6 MB
- Piper danny-low voice: ~60 MB
- MuJoCo: ~0.1 GB
- Booster SDK + deploy: ~500 MB
- **Total: ~10–12 GB**

**Simulation / RL training stack** (installed separately via [Simulation_Setup.md](Simulation_Setup.md)):
- Isaac Sim + Isaac Lab (incl. extension cache): ~20–30 GB

---

## Simulation Setup

For setting up **Isaac Sim / Isaac Lab** (training) and **MuJoCo** (sim2sim) for the K1 — plus
cloning the Booster repos (`booster_assets`, `booster_train`, `booster_deploy`, `booster_gym`,
`booster_robotics_sdk`) — see **[Simulation_Setup.md](Simulation_Setup.md)**.

---

## Network Setup

The K1 robot communicates over a **wired Ethernet** connection only.

### 1. Connect the cable
Plug an Ethernet cable between your workstation's wired port and the robot
(directly, or via an unmanaged switch on the same subnet).

### 2. Set a static IP on your workstation

**Via GUI (NetworkManager):**
1. Open Settings → Network → Wired → gear icon
2. IPv4 tab → Method: Manual
3. Address: `192.168.10.10` | Netmask: `255.255.255.0` | Gateway: (leave blank)
4. Apply and toggle the connection off/on

**Via terminal:**
```bash
# Find your wired interface name
ip addr show | grep -E "^[0-9]+: e"

# Set static IP (replace eno1 with your interface name)
sudo nmcli connection modify "Wired connection 1" \
    ipv4.method manual \
    ipv4.addresses 192.168.10.10/24
sudo nmcli connection up "Wired connection 1"
```

**Verify:**
```bash
ip addr show   # should show 192.168.10.10 on the wired interface
ping -c 3 192.168.10.102   # should get replies from the robot
```

### 3. SSH into the robot
```bash
ssh booster@192.168.10.102
# Password: 123456
```

---

## Verification Checklist

Run these after `source ~/.bashrc` to confirm everything is working:

```bash
# ROS2
ros2 --version
# Expected: ros2cli 0.18.x

# FastDDS config
echo $RMW_IMPLEMENTATION
# Expected: rmw_fastrtps_cpp

echo $FASTRTPS_DEFAULT_PROFILES_FILE
# Expected: /home/<you>/robots/k1/config/fastdds_k1.xml

# Python packages
python3 -c "import ultralytics; print('YOLO OK')"
python3 -c "import whisper; print('Whisper OK')"
python3 -c "import sounddevice; print('Audio OK')"
python3 -c "import groq; print('Groq OK')"
python3 -c "import torch; print('PyTorch', torch.__version__, '| CUDA:', torch.cuda.is_available())"
python3 -c "import mujoco; print('MuJoCo OK')"

# Piper voice model
ls ~/robots/shared/models/piper_voices/
# Expected: en_US-danny-low.onnx  en_US-danny-low.onnx.json

# Booster SDK
python3 -c "import booster_robotics_sdk; print('Booster SDK OK')"

# Robot connectivity (robot must be powered on and connected)
ping -c 2 192.168.10.102
ssh booster@192.168.10.102 "ros2 topic list | head -5"
```

**Simulation / RL stack** — Isaac Sim + Isaac Lab live in their own `isaaclab` conda env (not the
base `~/.bashrc` shell), so verify them there:

```bash
conda activate isaaclab
cd ~/booster/IsaacLab && ./isaaclab.sh -p scripts/tutorials/00_sim/create_empty.py   # window opens, exits clean
```

Full simulation checklist: [Simulation_Setup.md › Verification Checklist](Simulation_Setup.md#verification-checklist).

---

## Troubleshooting

**`sudo apt install ros-humble-desktop` fails — package not found**
The ROS2 apt repository wasn't added. Run:
```bash
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
    -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
    http://packages.ros.org/ros2/ubuntu jammy main" \
    | sudo tee /etc/apt/sources.list.d/ros2.list
sudo apt update && sudo apt install ros-humble-desktop
```

**`booster_robotics_sdk` build fails — cmake error**
Install build dependencies:
```bash
sudo apt install -y cmake build-essential libboost-all-dev
```

**`import booster_robotics_sdk` fails after build**
The Python bindings may not be on the path. Add:
```bash
export PYTHONPATH=~/booster_robotics_sdk/build/lib:$PYTHONPATH
```

**`ping 192.168.10.102` — no reply**
Check: (1) robot is powered on and showing green LED, (2) Ethernet cable is plugged in at both ends, (3) your workstation IP is `192.168.10.x` — run `ip addr show` and look for `192.168.10.10`.

**`ssh booster@192.168.10.102` — connection refused**
The SSH daemon on the robot may not have started yet. Wait 60 seconds after the green LED appears and try again.

**PyTorch CUDA not available (`torch.cuda.is_available()` returns False)**
Ensure the NVIDIA driver is installed: `nvidia-smi`. If it shows an error, install the driver:
```bash
sudo ubuntu-drivers autoinstall
sudo reboot
```

**`import mujoco` fails, or the viewer is a black window / `GL` error**
MuJoCo needs system GL libraries. Install them and, on a headless machine, pick a backend:
```bash
sudo apt install -y libglfw3 libegl1 libosmesa6
export MUJOCO_GL=egl   # or 'osmesa' for CPU-only / no display
```

**Isaac Sim / Isaac Lab won't install or import**
Isaac Sim is a large, GPU-only install handled **separately** from `install_all.sh` (see
[Simulation_Setup.md](Simulation_Setup.md)). It needs an RTX GPU with RT cores, driver ≥ 580, and
GLIBC ≥ 2.35 (Ubuntu 22.04+; check `ldd --version`). Most import errors are an Isaac Sim ↔ Isaac Lab
version mismatch — see [Simulation_Setup.md › Troubleshooting](Simulation_Setup.md#troubleshooting).
