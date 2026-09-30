# Maps

이 디렉터리에는 매핑 결과와 Navigation용 지도를 저장합니다.

```text
<name>.pcd            HDL Graph SLAM에서 저장한 최적화 3D 지도
<name>_2d/map.pgm     높이 구간을 투영한 2D occupancy image
<name>_2d/map.yaml    map_server metadata
```

현재 포함된 지도:

- `hit.pcd` + `hit_2d/map.pgm` + `hit_2d/map.yaml`: 기본 Navigation 지도
- `demo_2d/map.pgm` + `demo_2d/map.yaml`: 별도 데모 지도

PCD와 PGM/YAML은 같은 환경의 같은 좌표계를 사용해야 합니다. 새 지도를 추가할 때 YAML의 `image`는 같은 디렉터리의 PGM을 상대경로로 참조해야 합니다.

원본 PointCloud 토픽에 `map_saver`를 직접 사용하지 않습니다. 먼저 HDL Graph SLAM의 `save_map` service로 최적화된 PCD를 저장하고, `pointcloud_to_2dmap`으로 필요한 높이 범위만 PGM/YAML로 변환합니다.

전체 절차는 [3D Mapping 및 2D 지도 생성](../../../docs/MAPPING.md)을 참고하세요.
