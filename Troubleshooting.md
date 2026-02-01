# Resetting Ubuntu

```
If you prefer staying in a terminal (PowerShell or Command Prompt), you can "unregister" the distribution. This is instantaneous.

1. Open **PowerShell** as Administrator.
    
2. Run this command to see your installed distro name: `wsl --list`
    
3. Run the following to wipe it (replace `Ubuntu` with your distro name if it's different): `wsl --unregister Ubuntu`
    
4. To start over, just type `ubuntu` in PowerShell or launch it from your Start Menu. It will behave as if it’s being installed for the first time, asking you for a **new username and password**.
   
   
   GEMINI RESPONSE
```
so to completely reset ubuntu, i did:
1. `wsl --unregister Ubuntu-22.04`
2. opening ubuntu again from its windows app icon.

# Setting up Docker

after downloading docker and enabling ***WSL Integration***, 

go back to your **Ubuntu-22.04 terminal** (the one you just created). You don't need to install the Docker engine inside Ubuntu itself; the Windows app handles the heavy lifting.
Run this command to see if Ubuntu can "see" Docker:
```
docker --version
```

Then, run a test container to make sure everything is flowing correctly:
```
docker run hello-world
```
If you see a message saying _"Hello from Docker!"_, you are officially set up.
### Permission Denied error

 the classic "Permission Denied." This happens because, even though Docker Desktop is linked, your new Ubuntu user doesn't have the necessary "keys" to talk to the Docker engine yet.

The `docker.sock` file is owned by the `root` user and the `docker` group. Since your user **\[Name]** is brand new, we need to add you to that group.

### The Fix

Run this command in your Ubuntu terminal:
```
sudo usermod -aG docker $USER
```
**Crucial Step:** This change won't take effect until you "log out" and back in. Since this is WSL, the easiest way to do that is to shut down WSL entirely from Windows to force a refresh.

1. **Close** your Ubuntu terminal.
    
2. Open **PowerShell** and run: `wsl --shutdown`
    
3. **Re-open** your Ubuntu terminal.
### Deleting all docker containers for a clean reset

```
docker system prune -a --volumes
```
This will delete all the volumes and containers on docker.

# Rocker "Not on PATH" Warning

Run these two commands in your terminal. The first adds the directory to your configuration file, and the second activates it immediately.

1. **Add to your profile:**
    
    Bash
    
    ```
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
    ```
    
2. **Refresh the terminal:**
    
    Bash
    
    ```
    source ~/.bashrc
    ```
    

---

### Verify it worked

Now, try to see if Ubuntu can find `rocker` without you providing the full path:

Bash

```
which rocker
```

It should return `/home/[USER]/.local/bin/rocker`. If it does, you're all set to run your F1Tenth containers.

# Installing F1Tenth_gym_ros

https://github.com/f1tenth/f1tenth_gym_ros?tab=readme-ov-file

using the rocker steps ***only*** , NOT native, only following  >> With an NVIDIA gpu

# Running the rocker container

To open the container, run:
```
rocker --nvidia --x11 --volume .:/sim_ws/src/f1tenth_gym_ros -- f1tenth_gym_ros
```
MUST BE IN THE Cloned Repo Folder of `f1tenth_gym_ros`
# Launching the Simulation (Inside the container)

Using `tmux`, have 2 panels and run:
	
```
source /opt/ros/foxy/setup.bash && source install/local_setup.bash
```
	
```
ros2 launch f1tenth_gym_ros gym_bridge_launch.py
```
and
```
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```
on the other panel.


