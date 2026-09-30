# 3D Mapping 및 2D 지도 생성

## 1. 매핑 실행

다른 Scout/Velodyne launch가 실행 중이지 않은지 확인합니다.

```bash
cd ~/jung_ws
source devel/setup.bash
roslaunch scout_slam_demo mapping_3d.launch
```

실행 구성:

- Scout driver: 휠 오도메트리를 `/scout_odom`으로 발행
- Velodyne driver: `/velodyne_points`
- PointCloud prefilter
- Fast GICP scan-matching odometry
- HDL Graph SLAM 및 loop closure
- `map -> odom` publisher
- RViz mapping layout

매핑 중 TF 충돌을 피하기 위해 Scout odometry TF는 끕니다.

```text
map -> odom                 hdl_graph_slam
odom -> base_link           scan_matching_odometry
base_link -> velodyne       static transform
```

## 2. 시작 직후 점검

로봇을 5~10초 정지시킨 상태에서 확인합니다.

```bash
rostopic hz /velodyne_points
rostopic hz /filtered_points
rostopic hz /odom
rostopic hz /scout_odom
rostopic hz /hdl_graph_slam/map_points

rosrun tf tf_echo map base_link
rosrun tf tf_echo odom base_link
rosrun tf tf_echo base_link velodyne
```

| 토픽 | 정상 주기 |
|---|---:|
| `/velodyne_points` | 약 10 Hz |
| `/filtered_points` | 약 10 Hz |
| `/odom` | 약 10 Hz |
| `/scout_odom` | 약 50 Hz |
| `/hdl_graph_slam/map_points` | 약 0.2 Hz |

누적 지도는 첫 graph update 전까지 잠시 발행되지 않을 수 있습니다.

## 3. 매핑 주행

1. 시작 위치에서 5~10초 정지합니다.
2. 보행 속도보다 느리게 주행합니다.
3. 급가속, 급정지, 고속 제자리 회전을 피합니다.
4. 회전은 큰 원을 그리듯 완만하게 수행합니다.
5. 이미 지나간 장소를 다시 통과해 loop closure를 유도합니다.
6. 출발 지점 근처로 복귀한 뒤 약 10초 정지해 graph 최적화를 기다립니다.

벽이 여러 겹으로 벌어지거나 지도가 갑자기 회전하면 저장 전에 odometry, 센서 TF, scan matching 상태를 점검합니다.

## 4. 최적화된 PCD 저장

`mapping_3d.launch`를 실행한 상태에서 호출합니다.

```bash
MAP_DIR="$(rospack find scout_slam_demo)/maps"

rosservice call /hdl_graph_slam/save_map \
"utm: false
resolution: 0.05
destination: '${MAP_DIR}/scout_3d.pcd'"
```

`success: True`를 확인한 뒤:

```bash
ls -lh "${MAP_DIR}/scout_3d.pcd"
head -n 12 "${MAP_DIR}/scout_3d.pcd"
pcl_viewer "${MAP_DIR}/scout_3d.pcd"
```

벽과 기둥이 연속적이고 출발 지점으로 복귀한 부분이 대체로 겹치는지 확인합니다.

## 5. 변환기 빌드

`pointcloud_to_2dmap`은 독립 CMake 프로젝트입니다.

```bash
cd ~/jung_ws
cmake -S src/pointcloud_to_2dmap -B tools/pointcloud_to_2dmap_build
cmake --build tools/pointcloud_to_2dmap_build -j"$(nproc)"
```

## 6. PCD → PGM/YAML

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

생성 결과:

```text
maps/scout_2d/map.pgm
maps/scout_2d/map.yaml
```

기본 TF에서 바닥은 map 기준 약 `z=-0.235 m`입니다.

- 바닥 점이 장애물로 남으면 `min_height`를 높입니다.
- 낮은 장애물이 사라지면 `min_height`를 낮춥니다.
- 천장 점이 섞이면 `max_height`를 낮춥니다.

## 7. 지도 검증

```bash
rosrun map_server map_server \
  ~/jung_ws/src/scout_slam_demo/maps/scout_2d/map.yaml
```

RViz Fixed Frame을 `map`으로 설정하고 `/map`을 표시합니다.

PGM 픽셀 의미:

```text
0     occupied
255   free
128   unknown
```

필요하면 GIMP Pencil로 명백한 이동 물체나 소량의 노이즈만 정리합니다. resize, crop, rotate를 하면 YAML origin과 이미지 관계가 깨지므로 수행하지 않습니다. 실제 문, 벽, 기둥 및 위험 경계를 보기 좋게 만들 목적으로 삭제하지 않습니다.
