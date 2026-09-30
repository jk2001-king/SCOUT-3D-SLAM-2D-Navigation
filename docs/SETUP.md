# 환경 설정 및 빌드

## 검증 환경

```text
NVIDIA Jetson (aarch64)
Ubuntu 20.04
ROS 1 Noetic
AgileX Scout V2
Velodyne VLP-16
```

이 저장소에는 프로젝트 통합 패키지와 SLAM 의존 소스가 포함되어 있다. Scout 드라이버 및 URDF는 별도의 Scout 워크스페이스에서 제공한다고 가정한다.

## 시스템 패키지

```bash
sudo apt update
sudo apt install -y \
  build-essential cmake git python3-catkin-tools python3-rosdep \
  ros-noetic-rviz ros-noetic-tf2-tools \
  ros-noetic-robot-state-publisher ros-noetic-joint-state-publisher \
  ros-noetic-navigation ros-noetic-dwa-local-planner \
  ros-noetic-velodyne ros-noetic-pcl-ros \
  ros-noetic-geodesy ros-noetic-nmea-msgs \
  ros-noetic-libg2o libpcl-dev libopencv-dev libboost-program-options-dev
```

처음 `rosdep`을 사용하는 장비라면 `sudo rosdep init && rosdep update`를 실행한다. 이미 초기화된 장비에서는 반복하지 않는다.

## 워크스페이스 빌드

```bash
source /opt/ros/noetic/setup.bash
source ~/scout/devel/setup.bash

cd ~/jung_ws
catkin init
catkin config --extend ~/scout/devel
rosdep install --from-paths src --ignore-src -r -y
catkin build
source devel/setup.bash
```

검증:

```bash
rospack find scout_slam_demo
rospack find scout_base
rospack find scout_description
rospack find velodyne_pointcloud
rospack find hdl_graph_slam
```

새 터미널에서는 다음 환경을 불러옵니다.

```bash
cd ~/jung_ws
source devel/setup.bash
```

## Velodyne 네트워크

| 항목 | 검증값 |
|---|---|
| Sensor IP | `192.168.10.202` |
| Jetson Ethernet | `192.168.10.102/24` |
| UDP destination port | `2369` |

재부팅 후 한 번 실행한다.

```bash
sudo ip addr replace 192.168.10.102/24 dev eth0
sudo ip link set eth0 up
ip -br addr show eth0
```

패킷 확인:

```bash
sudo tcpdump -ni eth0 udp port 2369 -c 10
```

Velodyne가 ping에 응답하지 않아도 UDP packet이 수신되면 사용할 수 있다.

## Scout serial 장치

기본 launch는 아래 장치 경로를 사용한다.

```text
/dev/serial/by-id/usb-Prolific_Technology_Inc._USB-Serial_Controller_D-if00-port0
```

현재 장비의 경로를 확인한다.

```bash
ls -l /dev/serial/by-id/
```

다르면 launch argument로 지정한다.

```bash
roslaunch scout_slam_demo hardware_3d.launch \
  scout_port:=/dev/serial/by-id/<YOUR_DEVICE>
```

권한 문제가 있으면 `sudo usermod -aG dialout "$USER"` 실행 후 다시 로그인한다.

## 센서 장착 TF

기본값:

```text
base_link -> velodyne
x=0.34, y=0.0, z=0.185
roll=0.0, pitch=0.0, yaw=0.0
```

실제 장착 위치가 다르면 측정 후 launch argument로 조정한다.

```bash
roslaunch scout_slam_demo hardware_3d.launch \
  velodyne_x:=0.34 velodyne_y:=0.0 velodyne_z:=0.185 \
  velodyne_roll:=0.0 velodyne_pitch:=0.0 velodyne_yaw:=0.0
```

각도 단위는 radian이다. RViz에서 바닥이 수평이고 RobotModel과 PointCloud 방향이 일치하는지 확인한다.

## 센서 단독 점검

```bash
roslaunch scout_slam_demo velodyne_test.launch
```

다른 터미널:

```bash
rostopic hz /velodyne_points
rostopic echo -n 1 /velodyne_points/header
```

정상 기준은 약 10 Hz, `frame_id: velodyne`이다. 이 launch를 종료한 뒤 통합 launch를 실행한다.

## 통합 하드웨어 점검

```bash
roslaunch scout_slam_demo hardware_3d.launch
```

```bash
rostopic hz /velodyne_points
rostopic hz /odom
rostopic info /cmd_vel
rosrun tf tf_echo odom base_link
rosrun tf tf_echo base_link velodyne
```

정상 TF:

```text
odom -> base_link -> velodyne
```
