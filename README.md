# Kinesis Lambda.07 ROS 2 Workspace

This repository is the onboarding workspace for Kinesis CTP users working with
the Force Dimension Lambda.07 haptic device in ROS 2.

It provides:

- `.repos` manifests for the Kinesis forks of the Force Dimension ROS 2 stack;
- a Lambda.07-specific bringup package with 7-DOF controller configuration;
- an RViz view for the device end-effector pose, orientation frames, and clutch;
- a udev helper so the SDK and ROS node can access the USB device without `sudo`;
- guidance for using the Box-hosted Force Dimension SDK media without committing
  vendor binaries to git.

## Repository Layout

```text
.
├── lambda07_kinesis_https.repos  # read-only/simple clone manifest
├── lambda07_kinesis_ssh.repos    # contributor manifest for Kinesis members
├── scripts/                      # host setup helpers
└── src/lambda07_bringup/         # Lambda.07 launch, controller, URDF, RViz config
```

The upstream ROS 2 driver stack is
`ICube-Robotics/forcedimension_ros2`. The Kinesis workspace imports Kinesis
forks of:

- `KinesisCTP/forcedimension_ros2`
- `KinesisCTP/fd_sdk_vendor`

The vendor package can wrap the Force Dimension SDK extracted from the original
Lambda.07 USB media by passing `-DFD_SDK_ROOT=<path-to-sdk-3.17.7>`. Kinesis
will provide that USB media through a Box shared folder for Linux and Windows
drivers, manual SDK demos, PDFs, and offline lab setup. If `FD_SDK_ROOT` is not
provided, the vendor package falls back to downloading the configured SDK version.

## Host Target

Use Ubuntu 24.04 with ROS 2 Jazzy for the most conservative path. The upstream
driver documents Jazzy and Humble testing, and Jazzy is the ROS 2 LTS release
for Ubuntu 24.04.

This workstation also has ROS 2 Kilted available. Kilted may work, but treat it
as a validation target rather than the onboarding default until the stack is
tested there.

## Quick Start

Install ROS 2 Jazzy and common workspace tools, then clone this workspace:

```bash
git clone https://github.com/KinesisCTP/lambda-07.git ~/lambda-07_ws
cd ~/lambda-07_ws
```

Import the ROS 2 dependencies:

```bash
vcs import src < lambda07_kinesis_https.repos
```

Install package dependencies and build:

```bash
source /opt/ros/jazzy/setup.bash
rosdep update
rosdep install --from-paths src --ignore-src -r -y
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release --symlink-install
source install/setup.bash
```

## USB Access Without sudo

The Lambda.07 appears on Linux as:

```text
1451:040c Force Dimension lambda.x
```

Install the udev rule:

```bash
sudo ./scripts/install_udev_rules.sh
```

Unplug and reconnect the Lambda.07, then verify the SDK demo works without
`sudo`. If you are using the Box-hosted USB media:

```bash
cd ~/Lambda.07_USB/Linux/x86_64/sdk-3.17.7/bin
./HapticDesk
```

If the device still requires root, check your active session and permissions:

```bash
lsusb | grep -i '1451:040c'
getfacl /dev/bus/usb/<bus>/<device>
```

## RViz Smoke Test

Use fake hardware first to verify the ROS graph, robot description, controllers,
and RViz config without touching the physical haptic device:

```bash
ros2 launch lambda07_bringup lambda07.launch.py use_fake_hardware:=true
```

For the real device:

```bash
ros2 launch lambda07_bringup lambda07.launch.py
```

Expected topics include:

```bash
ros2 topic echo /lambda07/joint_states
ros2 topic echo /lambda07/ee_pose
ros2 topic echo /lambda07/fd_clutch
```

The device wrench command topic is:

```text
/lambda07/fd_controller/commands
```

Start with fake hardware and very small commands. Do not publish force commands
to the real device until the workspace has been checked by an operator familiar
with the Lambda.07 and the Force Dimension SDK examples.

## Contributor Setup

Users with write access to the Kinesis organization should configure GitHub SSH
access, then import using the SSH manifest:

```bash
vcs import src < lambda07_kinesis_ssh.repos
```

Recommended remote layout inside imported source repos:

```text
origin   -> git@github.com:KinesisCTP/<repo>.git
upstream -> git@github.com:ICube-Robotics/<repo>.git
```

Use feature branches for changes:

```bash
cd src/forcedimension_ros2
git switch -c feature/lambda07-validation
```

## Current Validation Notes

- Local SDK media found at `~/Lambda.07_USB/Linux/x86_64/sdk-3.17.7`.
- Local SDK demos detect and run the physical Lambda.07 when started with
  `sudo`.
- `lsusb` reports `1451:040c Force Dimension lambda.x`.
- The upstream ROS 2 stack is generic Force Dimension hardware through
  `ros2_control`, but its README only lists Omega and Falcon as tested devices.
- Lambda.07-specific launch and controller settings still need real-device
  validation.

