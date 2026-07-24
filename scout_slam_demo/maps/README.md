# Maps

Generated mapping artifacts belong in this directory.

```text
scout_3d.pcd   optimized 3D point-cloud map
scout_2d.pgm   height-projected 2D occupancy image
scout_2d.yaml  map_server metadata for scout_2d.pgm
```

Do not create the PGM with `map_saver` from raw Velodyne points. First save the
optimized map from the 3D SLAM backend as PCD, then run the height-filtered
`pointcloud_to_2dmap` conversion. The concrete commands will be added with the
mapping launch after those packages are installed.
