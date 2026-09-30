# pointcloud_to_2dmap

PCD의 지정 높이 구간을 2D occupancy map(PGM + YAML)으로 투영하는 독립 CMake 도구이다.

## Build

```bash
cd ~/jung_ws
cmake -S src/pointcloud_to_2dmap -B tools/pointcloud_to_2dmap_build
cmake --build tools/pointcloud_to_2dmap_build -j"$(nproc)"
```

## Usage

```bash
tools/pointcloud_to_2dmap_build/pointcloud_to_2dmap \
  INPUT.pcd OUTPUT_DIRECTORY \
  --resolution 0.05 \
  --map_width 2048 \
  --map_height 2048 \
  --min_height -0.10 \
  --max_height 1.50 \
  --crop_margin 2.0 \
  --unknown_border 0.50
```

| 옵션 | 기본값 | 설명 |
|---|---:|---|
| `--resolution` | `0.1` | pixel당 meter |
| `--map_width` | `1024` | 초기 map width |
| `--map_height` | `1024` | 초기 map height |
| `--min_points_in_pix` | `2` | occupied pixel 최소 점 수 |
| `--max_points_in_pix` | `5` | pixel saturation 점 수 |
| `--min_height` | `0.5` | 포함할 최소 z |
| `--max_height` | `1.0` | 포함할 최대 z |
| `--crop_margin` | `0.0` | occupied 영역 주변 여백(m)으로 자동 crop |
| `--unknown_border` | `0.0` | crop 결과 주변 unknown border 폭(m) |

출력 디렉터리에 `map.pgm`과 상대 이미지 경로를 사용하는 `map.yaml`을 생성한다.

Scout 프로젝트의 검증된 변환 절차와 높이값은 [Mapping 문서](../../docs/MAPPING.md)를 참고한다.
