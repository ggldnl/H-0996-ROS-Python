# ROS2 Python H-0996 Controller Node

This repository contains a ROS2 python node wrapping the [hexapod controller](https://github.com/ggldnl/Hexapod-Controller.git).

For a complete overview of the project refer to the [main H-0996 repository](https://github.com/ggldnl/Hexapod.git).

## 🛠️ Setup

Prerequisite: having ROS2 installed. I am running Ubuntu Server 24.04 on the RPI. The recommended stable distribution is [Jazzy Jalisco](https://docs.ros.org/en/jazzy/index.html)
which was specifically designed to target Ubuntu 24.04 as its primary platform.

> Note:️ on a fresh Ubuntu install, following the official ROS2 guide, 
> I had problems installing `ros-dev-tools` due to a library version mismatch.
> I solved them with:
> ```bash
> sudo apt install libbz2-1.0=1.0.8-5.1 -y --allow-downgrades
> sudo apt install bzip2 -y
> sudo apt update && sudo apt install ros-dev-tools
> ```

### Install dependencies

```bash
sudo apt install ros-<your-distro>-tf-transformations
```

### Clone the repo

Clone the repo. For simplicity, I will assume the ROS workspace is in the `home` folder. 
A ROS best practice is to put any packages in the workspace into the `src` directory.

```bash
cd ~/ros_ws/src  # use your actual ROS workspace
git clone https://github.com/ggldnl/Hexapod-ROS-Python.git
```

### Install the controller

Building this node will require the [python controller](https://github.com/ggldnl/Hexapod-Controller.git) to be installed as a python package.
The node imports it to open the serial link and provision the board, so the framing and the wire protocol stay in one place.
When `colcon build` installs the node it only knows about packaged code installed for system python. Install the controller with the `serial` extra so it pulls in pyserial for the real UART:

```bash
pip install --break-system-packages "hexapod-controller[serial] @ git+https://github.com/ggldnl/Hexapod-Controller.git"
```

### Build the package 

```bash
cd ~/ros_ws
colcon build --packages-select hexapod_controller
source install/setup.bash
```

### Verify it's working

Run the node:

```bash
ros2 run hexapod_controller hexapod_controller
```

In a second terminal, listen to the topic:

```bash
source ~/ros_ws/install/setup.bash
ros2 topic echo /hexapod/state
```

You should see the status of the robot (e.g. `IDLE`) printed every second on the terminal.

## ⚙️ Configuration

The node ships its own `config/config.yml` with the full robot description (kinematics, gaits, safety limits, hardware wiring and calibration, serial port).
At launch the node loads it and provisions the Servo2040 with it before enabling the board, so editing this file and relaunching is all it takes to change the robot description.
Point the node at a different file with the `config_path` parameter when you need to.

## 🚀 Deploy

Run the node:

```bash
ros2 run hexapod_controller hexapod_controller
```

ROS2 parameters can be overridden at launch time without touching the code:

```bash
ros2 run hexapod_controller hexapod_controller --ros-args -p port:=/dev/ttyAMA0 -p config_path:=/path/to/config.yml
```

## 📷 Camera (optional)

The robot can carry a Raspberry Pi Camera Module 2 / NoIR (IMX219). It is optional, you can skip this section and everything else will work exactly the same.

### One-time setup on the Pi

On Ubuntu Server the camera does not work out of the box: the sensor overlay is not enabled, and the packaged libcamera that ROS uses cannot enumerate the sensor on a Pi 5. `scripts/pi_camera_setup.sh` fixes both. Run it on the Pi as your normal user:

```bash
./scripts/pi_camera_setup.sh
```

It tells you when it needs a reboot. Reboot, run it again, and repeat until it says done (usually one or two reboots). If the packaged libcamera turns out to be broken it builds one from source, which takes 15-25 minutes and happens automatically.

> Note: the script only prepares the environment. The camera stays free until you launch a node.

Check it any time with:

```bash
./scripts/pi_camera_setup.sh status
```

### Launch the robot with the camera

```bash
ros2 launch hexapod_controller hexapod.launch.py camera:=true
```

Without `camera:=true` you get the controller alone. The usual parameters still work:

```bash
ros2 launch hexapod_controller hexapod.launch.py camera:=true port:=/dev/ttyAMA0
```

### Launch the camera alone

```bash
ros2 launch hexapod_controller camera.launch.py
ros2 launch hexapod_controller camera.launch.py camera_width:=1280 camera_height:=720
```

Either way you get `/camera/image_raw`, `/camera/image_raw/compressed` and `/camera/camera_info`.

> Note: only one process can use the camera at a time. If a second one starts it fails with `no cameras available`, which looks like a broken setup but only means the camera is already in use.

To view the feed from another machine, subscribe to `/camera/image_raw/compressed`. The raw topic is roughly 28 MB/s at 640x480, far more than Wi-Fi carries.

```bash
ros2 run rqt_image_view rqt_image_view
```

The `Unable to open camera calibration file` warning on startup is harmless, it just means the camera has no intrinsic calibration yet. Images stream normally.


## 🤝 Contribution

Feel free to contribute by opening issues or submitting pull requests. For further information, check out the [main H-0996 repository](https://github.com/ggldnl/Hexapod). Give a ⭐️ to this project if you liked the content.