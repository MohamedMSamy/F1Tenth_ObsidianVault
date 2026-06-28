
**Every time you want to work, follow this exact order:**

---

**Step 1 — Start the devkit container (WSL terminal)**

```bash
docker run --name autodrive_roboracer_api --rm -it \
  --entrypoint /bin/bash \
  --ipc=host \
  -p 4567:4567 \
  --privileged --gpus all \
  autodriveecosystem/autodrive_roboracer_api:v1
```

---

**Step 2 — Launch the bridge (inside the container)**

```bash
source /opt/ros/humble/setup.bash
source /home/autodrive_devkit/install/setup.bash
ros2 launch autodrive_roboracer bringup_headless.launch.py
```

Wait until you see:

```
[autodrive_bridge-1]: process started with pid [XX]
```

**Keep this terminal open.**

---

**Step 3 — Launch the Windows simulator**

Open the simulator `.exe` on Windows.

In the UI set:

- IP Address: `127.0.0.1`
- Port: `4567`
- Click **Autonomous**

It should auto-connect to the bridge.

⚠️ **Bridge must be running BEFORE you open the simulator or it won't connect.**

---

**Step 4 — Test connection (new WSL terminal)**

```bash
docker exec -it autodrive_roboracer_api bash
source /opt/ros/humble/setup.bash
source /home/autodrive_devkit/install/setup.bash
ros2 topic list
```

If you see `/autodrive/roboracer_1/lidar`, `/autodrive/roboracer_1/imu` etc. — you're fully connected.

---

**Step 5 — Test teleop (optional)**

```bash
ros2 run autodrive_roboracer teleop_keyboard
```

Click on the terminal window to give it focus, then use the keyboard to drive.

---

**If it doesn't connect:**

- Did you start the bridge before the sim? → Close sim, restart sim
- Is port 4567 open? → Run in PowerShell: `Test-NetConnection -ComputerName 127.0.0.1 -Port 4567` → Should say `TcpTestSucceeded: True`
- Is the bridge actually running? → Run `ss -tlnp | grep 4567` inside the container → Should show `0.0.0.0:4567 LISTEN`