# Kinesis Lambda.07 ROS 2 Workspace

This repository is the onboarding workspace for Kinesis CTP users working with
the Force Dimension Lambda.07 haptic device in ROS 2.

It provides:

- `.repos` manifests for the Kinesis forks of the Force Dimension ROS 2 stack;
- a Lambda.07-specific bringup package for read-only state, pose, and RViz validation;
- an RViz view for the device end-effector pose, orientation frames, and clutch marker;
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

Kinesis provides the original Lambda.07 USB media through this Box shared folder:
https://nyu.box.com/s/pwvzxy0muptwxoucpn8e4xnkkil2vow4

Use that folder for Linux and Windows drivers, manual SDK demos, PDFs, and
offline lab setup.

## Recommended Onboarding Sequence

Before setting up ROS 2, validate the device with the official Force Dimension
SDK media. This separates hardware/driver problems from ROS integration issues.

1. Download the Box folder and extract the package for your operating system.
   On Linux, this repository expects the SDK folder to look like:

   ```text
   ~/Lambda.07_USB/Linux/x86_64/sdk-3.17.7
   ```

2. Read the official Lambda manual and SDK notes from the USB media. For the
   Lambda.07, start with:

   ```text
   ~/Lambda.07_USB/Manuals/lambda.x.pdf
   ```

3. Run the official SDK examples before using ROS. On Linux:

   ```bash
   cd ~/Lambda.07_USB/Linux/x86_64/sdk-3.17.7/bin
   ./HapticInit
   ./Emporium
   ```

   `HapticInit` initializes and checks the device. `Emporium` exercises a wider
   set of haptic capabilities and is a good interactive sanity check. Other
   useful examples in the same folder include `HapticDesk`, `gravity`,
   `autoinit`, and `torques`.

4. If Linux reports a missing shared library when starting an SDK example,
   install the package requested by the error and retry. To inspect unresolved
   runtime dependencies:

   ```bash
   ldd ./HapticInit | grep 'not found'
   ldd ./Emporium | grep 'not found'
   ```

5. If SDK examples only work with `sudo`, install the udev rule below, unplug
   and reconnect the device, then retry the same SDK examples without `sudo`.

After the official SDK examples work, continue with the ROS 2 setup below.

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

If you downloaded or copied the original USB media from Box, point the build at
that extracted SDK instead of letting `fd_sdk_vendor` download one:

```bash
colcon build \
  --cmake-args \
  -DCMAKE_BUILD_TYPE=Release \
  -DFD_SDK_ROOT=$HOME/Lambda.07_USB/Linux/x86_64/sdk-3.17.7 \
  --symlink-install
```

Replace `$HOME/Lambda.07_USB/Linux/x86_64/sdk-3.17.7` with the actual path to
the extracted `sdk-3.17.7` folder on your computer.

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

Force output is not enabled by default and should be treated as experimental.
Do not publish force commands to the real Lambda.07 from this workspace yet. The
read-only state, pose, clutch-button, and RViz paths are the currently validated
onboarding path.

The upstream Force Dimension ROS 2 stack exposes an effort controller path, but
on this Ubuntu 24.04 / ROS 2 Kilted workstation the real Lambda.07 force path
still needs follow-up: controller activation reached the hardware, but testing
hit `std::out_of_range: unordered_map::at` and command-interface activation
failures before any nonzero force command was safely sent. Keep force-command
work behind an explicit development branch until that hardware-interface issue
is isolated.

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
- Local SDK demos detect and run the physical Lambda.07. After installing the
  udev rule, SDK demos and ROS can access the USB device without `sudo` on this
  workstation.
- `lsusb` reports `1451:040c Force Dimension lambda.x`.
- Fake-hardware launch opens RViz successfully.
- Real-device read-only launch connects to the Lambda.07, publishes joint state,
  end-effector pose, and clutch-button state, and displays the simplified RViz
  model.
- Force output is not validated. A force-controller investigation was stopped
  before any nonzero force command was sent to the real device.

