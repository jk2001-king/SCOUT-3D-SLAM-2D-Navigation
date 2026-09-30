# 2D Navigation

## 구성과 TF

Navigation 단계에서는 3D SLAM을 종료하고 다음 구성만 사용합니다.

```text
Scout driver + wheel odometry
Velodyne VLP-16 + ring LaserScan
map_server + AMCL
move_base + NavFn + DWA
```

TF 소유권:

```text
map -> odom                 AMCL
odom -> base_link           Scout driver
base_link -> velodyne       static transform
```

## 실행

먼저 매핑 및 하드웨어 점검 launch가 모두 종료됐는지 확인합니다.

```bash
sudo ip addr replace 192.168.10.102/24 dev eth0
sudo ip link set eth0 up

cd ~/jung_ws
source devel/setup.bash
roslaunch scout_slam_demo navigation_2d.launch
```

다른 지도:

```bash
roslaunch scout_slam_demo navigation_2d.launch \
  map_file:=$(rospack find scout_slam_demo)/maps/demo_2d/map.yaml
```

## 이동 전 점검

```bash
rostopic hz /velodyne_points
rostopic hz /scan
rostopic hz /odom
rostopic info /cmd_vel

rosrun tf tf_echo map odom
rosrun tf tf_echo odom base_link
rosrun tf tf_echo map base_link
```

정상 기준:

- `/scan`: 약 10 Hz
- `/odom`: 약 50 Hz
- `/cmd_vel`: Scout base node가 subscriber
- `map -> odom -> base_link -> velodyne`: 끊김 없이 연결
- LaserScan이 지도 벽과 대체로 일치
- Local/Global Costmap에 장애물이 정상 표시

## RViz 운용 순서

1. `2D Pose Estimate`로 실제 로봇 위치와 방향을 지정합니다.
2. 로봇을 조금 움직여 AMCL particle이 수렴하는지 확인합니다.
3. `2D Nav Goal`로 목표 위치와 최종 방향을 설정합니다.
4. Global Plan, Local Plan, costmap, `/cmd_vel`을 관찰합니다.
5. 목표 취소 시 즉시 정지하는지 확인합니다.

첫 목표가 끝난 뒤 Pose Estimate를 다시 주지 않고 다음 목표를 설정할 수 있습니다. 실제 시험에서 직진, 좌·우회전, 좁은 통로 및 목표 도착 후 자세 정렬을 연속 수행했습니다.

## 검증된 DWA 설정

| 파라미터 | 값 | 목적 |
|---|---:|---|
| `max_vel_x` | `0.22 m/s` | 실내 시연용 최대 속도 |
| `min_vel_x` | `0.05 m/s` | 모터가 반응하지 않는 극저속 제거 |
| `max_vel_theta` | `0.40 rad/s` | 회전 속도 제한 |
| `sim_time` | `1.5 s` | 가까운 코너를 미리 평가 |
| `vth_samples` | `50` | 충분한 회전 후보 검사 |
| `path_distance_bias` | `48.0` | global path 추종 강화 |
| `twirling_scale` | `0.20` | 불필요한 회전과 코너링의 균형 |

이 값들은 실주행에 성공한 최종 설정입니다. 환경 변화에 대한 근거 없이 여러 값을 동시에 수정하지 않는 것을 권장합니다.

## 센서와 footprint

Costmap은 VLP-16 ring 8의 `/scan`을 사용합니다.

```text
sensor frame: velodyne
obstacle range: 8.0 m
raytrace range: 10.0 m
```

차체 footprint는 앞뒤 ±0.47 m, 좌우 ±0.35 m, padding 0.03 m입니다. 실제 차체나 장착물이 더 크면 값을 늘려야 합니다.

## 안전 시험 순서

1. 리모컨과 비상정지를 확인합니다.
2. 가까운 직선 목표를 낮은 속도로 시험합니다.
3. 완만한 회전과 최종 자세 정렬을 시험합니다.
4. 충분한 여유가 있을 때만 좁은 통로를 시험합니다.
5. 사람이 있는 공간에서는 자동 주행을 시작하지 않습니다.
