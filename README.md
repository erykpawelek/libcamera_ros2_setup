# Raspberry Pi 5 + Camera Module 3 Setup for ROS 2 Jazzy (Ubuntu 24.04)

This repository provides a comprehensive guide and necessary commands to set up the official Raspberry Pi Camera Module 3 on Raspberry Pi 5 running Ubuntu 24.04 LTS and ROS 2 Jazzy.

**The Issue:**
The standard ROS 2 installation (`sudo apt install ros-jazzy-camera-ros`) installs the upstream version of `libcamera`, which currently lacks full support for the RPi 5 + Camera Module 3 combination on Ubuntu. This results in "no cameras available" errors.

**The Solution:**
We need to manually build the **Raspberry Pi fork of `libcamera`** from source and link it with the `camera_ros` node within a colcon workspace.

**Sources:**
* Based on the official Raspberry Pi libcamera fork: [https://github.com/raspberrypi/libcamera](https://github.com/raspberrypi/libcamera)
* Camera ROS node wrapper: [https://github.com/christianrauch/camera_ros](https://github.com/christianrauch/camera_ros)

---

## 1. System Preparation (On Raspberry Pi)

Update the system and install all necessary build tools, dependencies, and the ROS 2 colcon-meson plugin.

```bash
sudo apt install -y g++ cmake meson ninja-build pkg-config 
sudo apt install -y libboost-dev libglib2.0-dev libgstreamer-plugins-base1.0-dev libgnutls28-dev openssl 
sudo apt install -y libjpeg-dev libpng-dev libtiff-dev 
sudo apt install -y python3-yaml python3-ply pybind11-dev 
sudo apt install -y qt6-base-dev libqt6widgets6 
sudo apt install -y python3-colcon-meson 
```

## 2. Create ROS2 workspace (if you already haven't).

Create standard folder structure for ros2_ws.

```bash
mkdir -p ~/ros2_ws/src
```

## 3. Clone source code.

Clone the specific Raspberry Pi fork of `libcamera` and the ROS 2 camera node.

```bash
cd ~/ros2_ws/src

# 1. Clone the RPi fork of libcamera
git clone https://github.com/raspberrypi/libcamera.git

# 2. Clone the ROS 2 camera node wrapper
git clone https://github.com/christianrauch/camera_ros.git
```

## 4. Building.
Install any remaining ROS dependencies and build the packages. Note that `camera_ros` will automatically link against our local `libcamera` build.

While being in `~/ros2_ws`:

```bash
# Check for missing dependencies
rosdep install --from-paths src --ignore-src -y

# Build the packages (this may take 5-15 minutes on RPi 5)
colcon build --packages-select libcamera camera_ros
```

**Hint:**
Make sure that your hardware setup is correct, CSI camera port is enabled. (It should be by default).

## 5. Running the Camera Node (On Raspberry PI).

After rebooting, connect via SSH and run the node. Always source the local setup file to use the custom-built library.

```bash
cd ~/ros2_ws
source install/setup.bash
ros2 run camera_ros camera_node
```

**Warning:**
After running node you will probably get load of `INFO` and `WARNING` logs. Don't mind them as long as there are is no `FATAL/ERROR`. The easiest test is to just test it using ROS2's `topic list` command.

**Hint:**
You may encounter `ERROR` corresponding to calibration files, but since it's your first test and you haven't created them it isn't a problem.

```bash
ros2 topic list

# Expected result: 
/camera/camera_info
/camera/image_raw
/camera/image_raw/compressed
/parameter_events
/rosout
```
## 6. Viewing image.

To view the video stream on your laptop (connected to the same Wi-Fi):

### 1. Make sure you have installed `rqt_image_view`.

```bash
sudo apt install ros-jazzy-rqt-image-view ros-jazzy-image-transport-plugins
```

### 2. Run the viewer.

```bash
source /opt/ros/jazzy/setup.bash
ros2 run rqt_image_view rqt_image_view
```

### 3. Select topic from dropdown menu (left top corner):

Select `/camera/image_raw/compressed`. 

## 7. Common Issues & Troubleshooting (Viewing image).

### `rqt_image_view` Crashing or Freezing
**Symptoms:** The application closes unexpectedly with `invalid allocator` error or freezes when refreshing the topic list.
**Cause:** This is a known bug in the `rqt_image_view` application in ROS 2 when handling compressed streams during topic switching.
**Workaround:**
1. Restart the application.
2. If it keeps crashing, try selecting the empty row in the topic list (or a different topic) before switching to `/camera/image_raw/compressed`.

### Terminal Errors: `(!buf.empty())`
**Symptoms:** You see repeated red error logs in the terminal like:
`OpenCV(4.6.0) ... error: (-215:Assertion failed) !buf.empty() in function 'imdecode_'`
**Cause:** This is normal when streaming video over Wi-Fi. It indicates that a data packet was lost during transmission (Packet Loss).
**Solution:** **Ignore it.** As long as the video stream in the window looks smooth, the system is working correctly. ROS 2 skips the broken frame and displays the next one.