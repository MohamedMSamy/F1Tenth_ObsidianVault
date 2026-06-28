## AutoDRIVE RoboRacer Sim Setup — WSL2 + Docker Desktop + NVIDIA GPU

### System: Windows + WSL2 (Ubuntu 22.04) + Docker Desktop + NVIDIA RTX 4070

### ONE-TIME SETUP (only needs to be done once per machine)

#### 1. Prerequisites

- Docker Desktop installed, with WSL2 integration enabled
- NVIDIA Windows driver installed (`nvidia-smi` works in WSL2 terminal)

#### 2. Docker Desktop network setting (critical fix)

In Docker Desktop → **Settings → Resources → Network**:

- ✅ Check **"Enable host networking"**
- Click **Apply & Restart**

(Without this, `--network=host` containers cannot be reached from the WSL2 host — this was the root cause of the "Disconnected" simulator issue.)

#### 3. Clone the repo

bash

```bash
git clone https://github.com/AutoDRIVE-Ecosystem/AutoDRIVE-RoboRacer-Sim-Racing.git
cd AutoDRIVE-RoboRacer-Sim-Racing
```

#### 4. Build the devkit (ROS2/API) Docker image

bash

```bash
docker build --tag autodriveecosystem/autodrive_roboracer_api:latest -f autodrive_devkit.Dockerfile .
```

#### 5. Install GPU-capable Vulkan driver on the WSL2 **host** (not in Docker)

The stock Ubuntu 22.04 Mesa driver can't use your GPU through WSL2's translation layer. Fix:

bash

```bash
sudo add-apt-repository -y --remove ppa:kisak/kisak-mesa   # in case it was added; jammy is unsupported there
sudo add-apt-repository -y ppa:kisak/turtle                 # jammy IS supported here
sudo apt update
sudo apt install --only-upgrade mesa-vulkan-drivers libgl1-mesa-dri
sudo apt install -y vulkan-tools
```

Verify it worked:

bash

```bash
export XDG_RUNTIME_DIR=/tmp
vulkaninfo --summary
```

You should see a `GPU0` entry with `deviceType = PHYSICAL_DEVICE_TYPE_DISCRETE_GPU` and your real GPU name with `driverName = Dozen`.

**Note:** We do NOT run the simulator inside Docker — Docker's container (Ubuntu 18.04 base) can't easily get this same fix. Instead, **the simulator runs directly on the WSL2 host**, using the simulator files already present in the cloned repo (`autodrive_simulator/`). No separate Docker image build is needed for the simulator.

---

### EVERY TIME YOU WANT TO RUN THE SIMULATOR (3 terminals)

#### Terminal 1 — Devkit/API container (ROS2 bridge)

bash

```bash
docker run --name autodrive_roboracer_api --rm -it --entrypoint /bin/bash --network=host --ipc=host -v /tmp/.X11-unix:/tmp/.X11-unix:rw --env DISPLAY --gpus all autodriveecosystem/autodrive_roboracer_api:latest
```

Inside the container:

bash

```bash
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 launch autodrive_roboracer bringup_headless.launch.py
```

Leave this running. You should see:

```
[INFO] [autodrive_bridge-1]: process started with pid [XX]
```

#### Terminal 2 — Simulator (runs directly on WSL2 host, NOT in Docker)

bash

```bash
export XDG_RUNTIME_DIR=/tmp
export VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/dzn_icd.x86_64.json
cd ~/AutoDRIVE-RoboRacer-Sim-Racing/autodrive_simulator
./"AutoDRIVE Simulator.x86_64"
```

Wait for the full 3D scene (car + track) to appear in the window.

#### Step: Connect the simulator to the bridge

In the simulator window's left panel:

- IP Address: `127.0.0.1`
- Port Number: `4567`
- Click the **"Disconnected"** button/icon → it should flip to **"Connected"**

#### Terminal 3 — Send control commands

bash

```bash
docker exec -it autodrive_roboracer_api bash
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 topic pub /autodrive/roboracer_1/throttle_command std_msgs/msg/Float32 "data: 0.3" -r 10
```

For steering, open a **4th terminal** (so throttle keeps publishing):

bash

```bash
docker exec -it autodrive_roboracer_api bash
source /opt/ros/humble/setup.bash
source install/setup.bash
ros2 topic pub /autodrive/roboracer_1/steering_command std_msgs/msg/Float32 "data: 0.3" -r 10
```

---

### Key topics available

```
/autodrive/roboracer_1/throttle_command   (std_msgs/msg/Float32)
/autodrive/roboracer_1/steering_command   (std_msgs/msg/Float32)
/autodrive/roboracer_1/odom
/autodrive/roboracer_1/imu
/autodrive/roboracer_1/lidar
/autodrive/roboracer_1/front_camera
/autodrive/reset_command
```

---

### Shutdown / cleanup

bash

```bash
# Ctrl+C in Terminal 1 and any ros2 topic pub terminals
pkill -f "AutoDRIVE Simulator"
docker stop autodrive_roboracer_api
```

---

### Things that did NOT work (avoid repeating these dead ends)

- Running the **simulator** inside a Docker container in WSL2 — its Ubuntu 18.04 base can't get a working GPU Vulkan driver easily.
- `-force-glcore` flag — this build has no OpenGL fallback compiled in; always segfaults/fails.
- `ppa:kisak/kisak-mesa` (the "fresh" PPA) — dropped Jammy (22.04) support; use `ppa:kisak/turtle` instead.
- Forgetting to enable **"Enable host networking"** in Docker Desktop — without it, `--network=host` containers are invisible to the WSL2 host's `localhost`, which is exactly why the simulator showed "Disconnected" even though everything else was correctly configured.