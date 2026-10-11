# Peta File dan Paket

## Peta paket inti (checkpoint Phase 1)

### Command/Script
```bat
git ls-files src/*/package.xml src/*/*/package.xml
```

### Output
Paket yang relevan dengan chassis/sensor/odometri/nav/mapping/localization:
- `controller` (ament_python) — kinematika chassis + node odometri + init_pose.
- `ros_robot_controller` (ament_python) — jembatan board/STM32.
- `ros_robot_controller_msgs` (ament_cmake) — pesan motor/servo/IMU.
- `servo_controller` (ament_python) + `servo_controller_msgs` (ament_cmake).
- `kinematics` (ament_python) + `kinematics_msgs` (ament_cmake).
- `navigation` (ament_python) — launch Nav2 (tanpa executable).
- `slam` (ament_python) — launch SLAM + `map_save`.
- `peripherals` (ament_python) — sensor/teleop/joystick.
- `app`, `bringup`, `calibration`, `example`, `interfaces` (ament_cmake).
- `simulations/landerpi_description` — URDF/mesh (tank/mecanum/acker/arm/gripper).

### Kesimpulan
Peta paket lengkap tersedia di `sources/Doc/file_inventory.txt` (926 entri; tidak dibuat ulang).
Paket inti Phase 1–2 teridentifikasi. Hanya `interfaces`, `*_msgs`, dan
`simulations/landerpi_description` yang ber-`build_type` ament_cmake; sisanya ament_python.
`TERBUKTI DARI SOURCE`.

## Entry point — verifikasi lintas paket

### Command/Script
```bat
grep -n "console_scripts" src/<pkg>/setup.py
```

### Output
- `controller`: `odom_publisher`, `init_pose`.
- `kinematics`: `search_kinematics_solutions`.
- `ros_robot_controller`: `ros_robot_controller`.
- `servo_controller`: `servo_controller`.
- `slam`: `map_save`.
- `peripherals`: `joystick_control`, `tf_broadcaster_imu`, `teleop_key_control`.
- `app`: `lidar_controller`, `line_following`, `object_tracking`, `ar_app`, `hand_trajectory`, `hand_gesture`.
- `bringup`: `startup_check`.
- `calibration`: `calibrate_linear`, `calibrate_angular`.
- `navigation`: **tidak ada** console_scripts (hanya launch + config + rviz).
- `sdk`: **tidak ada** console_scripts (library `Board`).

### Kesimpulan
Entry point yang terdaftar/terpasang hanya muncul setelah `colcon build`. `navigation` tidak
memiliki node Python; seluruh fungsinya via Nav2 + launch/config. `TERBUKTI DARI SOURCE`.

## Dependency: deklarasi vs import (checkpoint Phase 1)

### Command/Script
```bat
grep -n "depend" src/driver/controller/package.xml src/navigation/package.xml src/slam/package.xml src/peripherals/package.xml
grep -n "^import \|^from " src/driver/controller/controller/odom_publisher_node.py
```

### Output
- `package.xml` paket ament_python terkait hanya memuat `test_depend`
  (`ament_copyright`, `ament_flake8`, `ament_pep257`, `python3-pytest`) — **tanpa** `<depend>`/`<exec_depend>`.
- Import aktual `odom_publisher_node.py` (baris 6–14): `rclpy`, `std_srvs`, `nav_msgs`,
  `controller.ackermann`/`controller.mecanum`, `ros_robot_controller_msgs`, `geometry_msgs`.
- Import `ros_robot_controller_node.py`: `rclpy`, `std_srvs`, `sensor_msgs`, `std_msgs`,
  `ros_robot_controller.ros_robot_controller_sdk`, `ros_robot_controller_msgs`, `yaml` (PyYAML).
- Import `ros_robot_controller_sdk.py`: `serial` (pyserial) — **tidak dideklarasikan** di mana pun.
- `*_msgs` ament_cmake mendeklarasikan `<depend>std_msgs</depend>` (+ geometry_msgs untuk kinematics_msgs).

### Kesimpulan
- **Dependency yang dideklarasikan:** hampir tidak ada untuk paket Python (hanya test_depend).
- **Dependency yang diimpor langsung:** `rclpy`, `std_srvs`, `nav_msgs`, `geometry_msgs`,
  `sensor_msgs`, `std_msgs`, `PyYAML`, `pyserial`, paket lokal (`controller`, `ros_robot_controller`,
  `*_msgs`).
- **Dependency transitif:** `imu_calib`, `imu_complementary_filter`, `rf2o_laser_odometry`,
  `robot_localization`, `joy`, `nav2_*`, `slam_toolbox`/`rtabmap` — digunakan via launch, bukan import.
- **Belum dapat dipastikan:** versi paket dan apakah semua terpasang di robot sekolah.
- **Implikasi:** deklarasi manifest tidak dapat dipakai untuk menyimpulkan fitur aktif. `TERBUKTI DARI SOURCE` untuk import; `BELUM TERVERIFIKASI` untuk instalasi runtime.

## Daftar file controller

### Command/Script
```bat
dir /s /b src\driver\controller
```

### Output
Terlihat antara lain:
- `controller\mecanum.py`
- `controller\ackermann.py`
- `controller\odom_publisher_node.py`
- `controller\init_pose.py`
- `launch\controller.launch.py`
- `launch\odom_publisher.launch.py`
- `config\calibrate_params.yaml`
- `config\ekf.yaml`
- `config\init_pose.yaml`
- `setup.py`, `package.xml`

### Kesimpulan
Paket controller mencakup kinematika chassis, node odometry, konfigurasi estimasi gerak, dan launch file. `odom_publisher_node.py` merupakan file prioritas untuk analisis berikutnya.

## Entry point

### Command/Script
```bat
type src\driver\controller\setup.py
```

### Output
```python
'odom_publisher = controller.odom_publisher_node:main',
'init_pose = controller.init_pose:main'
```

### Kesimpulan
Paket mendaftarkan executable ROS 2 `odom_publisher` dan `init_pose` ketika dibangun/diinstal pada lingkungan yang sesuai.

## Launch controller

### Command/Script
```bat
type src\driver\controller\launch\controller.launch.py
```

### Output
Launch menyertakan odometry publisher, IMU filter, node EKF `robot_localization`, dan servo controller. Default yang terlihat mencakup `enable_odom=true`, `use_sim_time=false`, `odom`, `base_footprint`, dan `imu_link`. Pada node EKF terdapat remapping `cmd_vel` ke `controller/cmd_vel`.

### Kesimpulan
Launch file mengorkestrasi beberapa komponen. Remapping pada node EKF tidak cukup untuk membuktikan semua jalur perintah gerak; cek node dan launch terkait.
