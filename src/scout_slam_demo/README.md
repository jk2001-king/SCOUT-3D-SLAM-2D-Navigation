# scout_slam_demo

Scout V2와 Velodyne VLP-16을 HDL Graph SLAM, AMCL, `move_base`에 연결하는 프로젝트 통합 패키지이다.

전체 시스템 설명과 설치 절차는 저장소 [메인 README](../../README.md)를 참고한다.

## Launch files

| Launch | 역할 |
|---|---|
| `velodyne_test.launch` | VLP-16 단독 PointCloud 및 RViz 점검 |
| `hardware_3d.launch` | Scout + Velodyne + URDF/TF 점검 |
| `mapping_3d.launch` | Fast GICP odometry + HDL Graph SLAM 3D 매핑 |
| `navigation_2d.launch` | map_server + AMCL + move_base 2D Navigation |

## 주요 하드웨어 인자

```bash
roslaunch scout_slam_demo hardware_3d.launch \
  scout_port:=/dev/serial/by-id/<SCOUT_DEVICE> \
  velodyne_ip:=192.168.10.202 \
  velodyne_port:=2369 \
  velodyne_x:=0.34 velodyne_y:=0.0 velodyne_z:=0.185 \
  velodyne_roll:=0.0 velodyne_pitch:=0.0 velodyne_yaw:=0.0
```

## Mapping

```bash
roslaunch scout_slam_demo mapping_3d.launch
```

매핑 중 TF 소유권:

```text
map -> odom                 hdl_graph_slam
odom -> base_link           scan_matching_odometry
base_link -> velodyne       static transform
```

Scout wheel odometry는 TF 없이 `/scout_odom`으로 보존된다.

## Navigation

```bash
roslaunch scout_slam_demo navigation_2d.launch
```

다른 지도 선택:

```bash
roslaunch scout_slam_demo navigation_2d.launch \
  map_file:=$(rospack find scout_slam_demo)/maps/demo_2d/map.yaml
```

Navigation 중 TF 소유권:

```text
map -> odom                 AMCL
odom -> base_link           Scout driver
base_link -> velodyne       static transform
```

매핑과 Navigation launch를 동시에 실행하지 않는다.

## Configuration

| 파일 | 역할 |
|---|---|
| `config/amcl.yaml` | Particle filter 및 sensor/odometry model |
| `config/costmap_common.yaml` | Footprint, LaserScan, inflation 공통 설정 |
| `config/global_costmap.yaml` | Static/global obstacle costmap |
| `config/local_costmap.yaml` | Rolling local obstacle costmap |
| `config/move_base.yaml` | Planner/controller 주기 및 recovery |
| `config/dwa_local_planner.yaml` | 실주행 검증 DWA 속도·평가 파라미터 |

상세 운용 절차:

- [환경 설정](../../docs/SETUP.md)
- [3D 매핑 및 지도 변환](../../docs/MAPPING.md)
- [2D Navigation](../../docs/NAVIGATION.md)
- [문제 해결](../../docs/TROUBLESHOOTING.md)
