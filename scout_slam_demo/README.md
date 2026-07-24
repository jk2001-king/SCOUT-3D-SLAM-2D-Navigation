# Scout VLP-16 Mapping and Navigation

ROS Noetic integration for a Scout V2 and Velodyne VLP-16. This package owns
only the project launch files, parameters, RViz layouts, and generated maps.
The existing Scout and Velodyne driver workspaces remain separate.

## Current status

The VLP-16 standalone path is verified:

```bash
cd ~/jung_ws
source devel/setup.bash
roslaunch scout_slam_demo velodyne_test.launch
```

Expected output is `/velodyne_points` in frame `velodyne`. The saved RViz
layout is `config/velodyne.rviz`.

The sensor settings currently in use are:

```text
sensor IP:       192.168.10.202
Jetson eth0 IP:  192.168.10.102/24
destination UDP: 2369
```

The measured static transform from `base_link` to `velodyne` is:

```text
x=0.34  y=0.0  z=0.185
roll=0.0  pitch=0.0  yaw=0.0
```

## Start after a reboot

Power the Scout and Velodyne, connect Ethernet and the Scout USB serial cable,
then configure the Velodyne network address:

```bash
sudo ip addr add 192.168.10.102/24 dev eth0 2>/dev/null || true
ip -br addr show eth0
```

Load the workspace in every new terminal:

```bash
cd ~/jung_ws
source devel/setup.bash
```

No `catkin init` or rebuild is needed after an ordinary reboot.

## Standalone Velodyne check

```bash
roslaunch scout_slam_demo velodyne_test.launch
```

In another sourced terminal:

```bash
rostopic hz /velodyne_points
rostopic echo -n 1 /velodyne_points/header
```

Stop this launch with Ctrl-C before starting the combined hardware launch.

## Scout and Velodyne check

Place the robot in a clear area and keep the remote and emergency stop ready:

```bash
roslaunch scout_slam_demo hardware_3d.launch
```

Check the required data and TF tree from another sourced terminal:

```bash
rostopic hz /velodyne_points
rostopic hz /odom
rostopic info /cmd_vel
rosrun tf tf_echo odom base_link
rosrun tf tf_echo base_link velodyne
```

Expected tree:

```text
odom -> base_link -> velodyne
```

To tune the measured transform without editing the launch file:

```bash
roslaunch scout_slam_demo hardware_3d.launch \
  velodyne_x:=0.34 velodyne_y:=0.0 velodyne_z:=0.185 \
  velodyne_roll:=0.0 velodyne_pitch:=0.0 velodyne_yaw:=0.0
```

Use `base_link` as RViz Fixed Frame. The floor should be level and appear near
`z=-0.235 m`. Restart the launch after changing a static transform. Adjust
translation in 0.005-0.01 m increments and angles in 0.0087 rad increments.

## 3D mapping

The selected pipeline does not use RTAB-Map:

```text
/velodyne_points + Scout motion
  -> hdl_graph_slam
  -> maps/scout_3d.pcd
  -> pointcloud_to_2dmap height projection
  -> maps/scout_2d.pgm + maps/scout_2d.yaml
  -> map_server + AMCL + move_base
```

The mapping stack and ARM64 build fixes are installed in this workspace. Stop
`velodyne_test.launch` and `hardware_3d.launch` before starting mapping:

```bash
cd ~/jung_ws
source devel/setup.bash
roslaunch scout_slam_demo mapping_3d.launch
```

The mapping launch publishes the Scout wheel odometry as `/scout_odom` without
its TF. LiDAR scan matching exclusively owns `/odom` and `odom -> base_link`,
which prevents competing TF publishers.

Verify the mapping outputs from another sourced terminal:

```bash
rostopic hz /filtered_points
rostopic hz /odom
rostopic hz /hdl_graph_slam/map_points
rosrun tf tf_echo map base_link
```

Keep the robot still until the first map appears. Then drive slowly, avoid
wheel slip and abrupt turns, and return near the starting area to provide a
loop closure.

## Save PCD and generate PGM/YAML

While mapping remains running, save the optimized 3D map:

```bash
rosservice call /hdl_graph_slam/save_map \
"utm: false
resolution: 0.05
destination: '/home/ros/jung_ws/src/scout_slam_demo/maps/scout_3d.pcd'"
```

After the service reports `success: True`, convert an obstacle-height slice to
a 2D occupancy map:

```bash
cd ~/jung_ws
tools/pointcloud_to_2dmap_build/pointcloud_to_2dmap \
  src/scout_slam_demo/maps/scout_3d.pcd \
  src/scout_slam_demo/maps/scout_2d \
  --resolution 0.05 \
  --map_width 2048 \
  --map_height 2048 \
  --min_height -0.10 \
  --max_height 1.50 \
  --crop_margin 2.0 \
  --unknown_border 0.50
```

This creates `maps/scout_2d/map.pgm` and `maps/scout_2d/map.yaml`. The initial
height limits exclude the floor at approximately `z=-0.235 m`; tune them after
inspecting the generated PCD and occupancy image.

Do not use the navigation parameter files until the Velodyne-based AMCL and
obstacle-source launch is added.

## Rebuild after package changes

```bash
cd ~/jung_ws
source /opt/ros/noetic/setup.bash
source ~/scout/devel/setup.bash
catkin build
source devel/setup.bash
```
