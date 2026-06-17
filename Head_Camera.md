# Head Camera — Live Video Feed

How to get the K1 head camera's live video onto a workstation. The reliable path
is a tiny **HTTP bridge that runs on the robot**: it subscribes to the camera
topic over the robot's onboard ROS 2 stack and serves JPEG frames over plain
HTTP. Any machine then pulls frames with a normal GET — **no ROS 2 needed on the
consumer side**.

> **The one thing that matters:** use the topic **`/booster_video_stream`**.
> It is published by the robot's `booster-video-stream` service, **starts
> automatically on boot**, runs at **~10 fps**, and needs **no unlock call and no
> reboot**. See [What *not* to do](#what-not-to-do) for the dead end to avoid.

---

## Prerequisites

- Robot powered on and reachable (wired `192.168.10.102` — see
  [README › Network Setup](README.md#network-setup); Wi-Fi also works if configured).
- `sshpass` on your workstation (installed by `install_all.sh`).
- The robot already has `python3`, `numpy`, `opencv`, and `rclpy` — nothing to install on it.

---

## Step 1 — Copy the bridge to the robot

The bridge is [`scripts/robot_video_bridge.py`](scripts/robot_video_bridge.py) in this repo.

```bash
scp scripts/robot_video_bridge.py booster@192.168.10.102:/tmp/
# password: 123456
```

## Step 2 — Start the bridge **on the robot**

The bridge must run on the robot (it subscribes to the robot's *local* DDS; a
remote PC cannot subscribe to this camera directly).

```bash
ssh booster@192.168.10.102          # password: 123456

# on the robot:
source /opt/booster/BoosterRos2/install/setup.bash
source /opt/booster/BoosterRos2Interface/install/setup.bash
python3 /tmp/robot_video_bridge.py --topic /booster_video_stream --port 8080
```

You should see:

```
[bridge] subscribed to /booster_video_stream as sensor_msgs/msg/CompressedImage ...
[bridge] HTTP listening on http://0.0.0.0:8080/  (try /frame.jpg, /stream, or /)
```

To leave it running after you log out:

```bash
setsid python3 /tmp/robot_video_bridge.py --topic /booster_video_stream --port 8080 \
  >/tmp/bridge.log 2>&1 </dev/null &
```

## Step 3 — Pull frames from your workstation

```bash
# one latest frame
curl -s http://192.168.10.102:8080/frame.jpg -o frame.jpg && xdg-open frame.jpg

# live status (frame count, size, age)
curl -s http://192.168.10.102:8080/
```

**Endpoints**

| Endpoint | Use |
|----------|-----|
| `GET /frame.jpg` | latest single JPEG — **use this for scripts / vision models** (one GET per frame) |
| `GET /stream` | `multipart/x-mixed-replace` MJPEG — point a **browser** at it for a live view |
| `GET /` or `/status` | plain-text status: topic, type, frame count, size, age |

**In Python (no ROS 2):**

```python
import urllib.request, numpy as np, cv2
data = urllib.request.urlopen("http://192.168.10.102:8080/frame.jpg", timeout=5).read()
bgr = cv2.imdecode(np.frombuffer(data, np.uint8), cv2.IMREAD_COLOR)   # HxWx3 BGR
```

---

## Verify it's actually live (~10 fps)

Frozen frames are the classic failure mode, so confirm the feed is *moving*:

```bash
curl -s http://192.168.10.102:8080/ | grep frames ; sleep 5
curl -s http://192.168.10.102:8080/ | grep frames
```

Healthy state:
- frame count climbing **~10 per second**
- `age_s` **< 0.1** (frames are fresh, not a stuck buffer)
- `size` ≈ **544×306**

---

## What *not* to do

There is a **second** head-camera topic, `/boostercamera/head/raw/rgb` (the raw
MIPI stereo image). **Avoid it for a live feed.** It is throttled at startup
(~0.2–0.67 fps), needs a `StartVisionService` "unlock" call that resets on every
reboot and decays after about an hour, and leads to a fruitless reboot loop.
`/booster_video_stream` has none of these problems — it is the supported,
service-published stream. Always bridge `/booster_video_stream`.

---

## Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `/frame.jpg` → **503 "no frame yet"** | Bridge is up but received no frame. Check the topic has a publisher: `ssh booster@192.168.10.102 'source /opt/booster/BoosterRos2/install/setup.bash; ros2 topic info /booster_video_stream'` → `Publisher count` should be ≥ 1. If 0, the `booster-video-stream` service isn't running (`ps -ef \| grep video-stream`); reboot the robot and retry. |
| status shows **`frames: 0`** | Wrong topic. It **must** be `/booster_video_stream`. Do not use `/boostercamera/...` or `/booster_camera_bridge/...`. |
| **low / decaying fps** | You're on the raw MIPI topic. Switch to `/booster_video_stream` (see [What not to do](#what-not-to-do)). |
| **`rclpy not importable`** on the robot | You didn't `source /opt/booster/BoosterRos2Interface/install/setup.bash` before running the bridge. |
| `curl` from the workstation **hangs / refused** | Hit the robot's IP (`192.168.10.102`), not `localhost`. The bridge binds `0.0.0.0` by default. Confirm you can `ping 192.168.10.102`. |
| frame looks like a **frozen image** | Re-check `age_s` in `/status`. If it climbs, the producer stalled — restart the bridge; if still stale, reboot the robot. |

---

## How it fits together

```
robot: booster-video-stream service ──► /booster_video_stream (CompressedImage, ~10 fps)
            │
            └─► robot_video_bridge.py (subscribes, serves HTTP on :8080)
                    │
   workstation: GET http://192.168.10.102:8080/frame.jpg ──► your script / vision model
```

Because the consumer side is plain HTTP, the same feed works for a browser, a
`curl` loop, a YOLO node, or a remote VLM — none of them need ROS 2 installed.
