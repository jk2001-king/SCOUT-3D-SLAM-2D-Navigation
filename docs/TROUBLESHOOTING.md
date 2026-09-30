# 문제 해결

## Velodyne PointCloud가 나오지 않음

```bash
ip -br addr show eth0
sudo tcpdump -ni eth0 udp port 2369 -c 10
rostopic hz /velodyne_points
```

Jetson 주소, sensor IP, UDP port를 확인하고 다른 Velodyne launch가 같은 port를 사용 중이지 않은지 확인한다.

## Scout serial port를 열 수 없음

```bash
ls -l /dev/serial/by-id/
groups
```

launch의 `scout_port`와 실제 장치가 일치해야 하며 사용자가 `dialout` 그룹에 포함되어야 한다. 다른 Scout driver가 장치를 먼저 열고 있지 않은지도 확인한다.

## TF가 튀거나 지도가 흔들림

매핑과 Navigation launch를 동시에 실행하지 않는다.

```bash
rosrun tf2_tools view_frames.py
rosrun tf tf_echo map base_link
```

매핑에서는 scan matching이 `odom -> base_link`를, Navigation에서는 Scout driver가 같은 edge를 단독으로 발행해야 한다. 같은 TF edge를 둘 이상의 노드가 발행하면 위치가 튑니다.

## 3D 지도의 벽이 여러 겹으로 갈라짐

- `base_link -> velodyne` 장착 TF를 다시 측정한다.
- 급회전, 바퀴 미끄러짐, 고속 주행을 피한다.
- 이미 지나간 장소로 복귀해 loop closure를 유도한다.
- graph 최적화가 끝나기 전에 저장하지 않는다.

## 경로는 생기지만 움직이지 않음

```bash
rostopic echo /cmd_vel
rostopic info /cmd_vel
```

실제 발생했던 원인은 DWA가 `linear.x ≈ 0.0013 m/s`만 선택해 Scout 모터가 반응하지 못한 경우였다. `min_vel_x`와 `min_vel_trans`를 `0.05 m/s`로 맞춰 해결했다.

현재 설정에서도 움직이지 않으면 subscriber 연결, Scout 운전 모드, emergency stop, footprint가 장애물 내부에 있는지 확인한다.

## 한 방향으로 계속 회전함

```bash
rostopic echo /cmd_vel/angular/z
rostopic echo /odom/twist/twist/angular/z
rosrun tf tf_echo map base_link
```

명령, odometry, TF yaw 부호가 실제 회전과 모두 일치하면 motor/TF 부호 문제가 아니라 local planner 선택 문제이다. 목표 화살표의 최종 방향도 확인한다.

## 코너에서 회전하지 못함

- Local Plan이 코너 방향으로 생성되는지 확인한다.
- `sim_time`이 짧으면 가까운 코너를 평가하지 못한다.
- `twirling_scale`이 크면 필요한 회전도 과하게 벌점 처리한다.
- `vth_samples`가 작으면 적절한 회전 후보가 부족한다.

검증된 값은 `sim_time=1.5`, `vth_samples=50`, `twirling_scale=0.20`이다.

## AMCL 위치가 지도와 맞지 않음

1. RViz `2D Pose Estimate`의 위치와 화살표 방향을 정확히 지정한다.
2. `/scan`이 지도 벽과 겹치는지 확인한다.
3. TF 전체 체인이 연결되는지 확인한다.
4. 현재 환경에서 생성한 PGM/YAML인지 확인한다.

## 변환 지도에 바닥이나 천장이 포함됨

- 바닥이 남으면 `--min_height`를 높이다.
- 낮은 장애물이 사라지면 `--min_height`를 낮춥니다.
- 천장이 남으면 `--max_height`를 낮춥니다.
- 센서 TF를 변경했다면 높이 범위도 다시 측정한다.

## 빠른 상태 점검

```bash
rostopic hz /velodyne_points
rostopic hz /scan
rostopic hz /odom
rostopic info /cmd_vel
rosrun tf tf_echo map base_link
rosrun tf tf_echo base_link velodyne
```
