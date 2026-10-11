# Phase 3 Summary — Autonomous Navigation & Obstacle Avoidance

> Audit statis (baca source). Tidak ada node/hardware dijalankan. Label:
> `TERBUKTI DARI SOURCE`, `INFERENSI`, `BELUM TERVERIFIKASI`.
> Referensi: `05`–`08` + Phase 1–2 (`00`–`04`).

## Executive summary

Repository LanderPi menyediakan stack **Nav2 lengkap** (planner, controller, costmap, AMCL, BT,
velocity smoother) plus **slam_toolbox** (SLAM 2D) dan **rtabmap** (SLAM/nav 3D), serta satu
**avoidance reaktif berbasis LiDAR** di level aplikasi. Semua komponenn konfigurasi terbukti ada
di source dan terhubung lewat launch, **tetapi tidak ada bukti runtime** pada robot.

**Tiga temuan paling penting:**
1. **Nav2 menggerakkan mecanum sebagai robot diferensial**: DWB `max_vel_y: 0.0` dan TEB tanpa
   parameter `vy`; AMCL memakai `DifferentialMotionModel`. Keunggulan strafing mecanum **tidak
   dimanfaatkan**. (`TERBUKTI DARI SOURCE`.)
2. **Controller default = TEB** (`use_teb=true`), padahal `teb_local_planner` **tidak dideklarasikan**
   sebagai dependency → risiko gagal load. DWB adalah alternatif bawaan Nav2. (`TERBUKTI DARI SOURCE`.)
3. **Odometri roda `odom_raw` open-loop** (integrasi perintah, bukan encoder) — melanjutkan temuan
   Phase 2. Ini berisiko drift, meski EKF memfusikan IMU (yaw) + `odom_rf2o` bila rf2o aktif.
   (`TERBUKTI DARI SOURCE` untuk mekanisme; `INFERENSI` untuk dampak.)

## Strongest verified findings

| # | Temuan | Bukti (path:line) | Label |
|---|---|---|---|
| 1 | Entry `navigation.launch.py`, default `use_teb=true`, namespace dari `HOST` | `navigation.launch.py:21-23,35-39` | Terbukti |
| 2 | 8 node Nav2 composable + 2 lifecycle manager | `navigation_base.launch.py:28-143`, `localization.launch.py:21-86` | Terbukti |
| 3 | Planner = NavFn (`use_astar:false`), AMCL `DifferentialMotionModel` | `nav2_params.yaml:243,29` | Terbukti |
| 4 | Costmap konsumsi `/scan_raw` (voxel local, obstacle global) | `nav2_params.yaml:149-206` | Terbukti |
| 5 | Controller DWB tanpa lateral (`max_vel_y:0.0`) | `nav2_controller_dwb.yaml:52-54` | Terbukti |
| 6 | TEB default tapi tidak dideklarasikan | `navigation.launch.py:23`; `navigation/package.xml` | Terbukti |
| 7 | BT recovery mecanum: clear+spin+wait (tanpa backup) | `navigate_to_pose_without_backup.xml:22-32` | Terbukti |
| 8 | `cmd_vel_nav → cmd_vel → odom_publisher` | `navigation_base.launch.py:97,134`; `odom_publisher_node.py:144` | Terbukti |
| 9 | SLAM 2D = slam_toolbox konsumsi `scan_raw` | `slam_base.launch.py:51-58`; `slam.yaml:13-19` | Terbukti |
| 10 | TF: `map→odom→base_footprint→base_link→{lidar,imu}` | AMCL/EKF/RSP | Terbukti (statis) |
| 11 | App transport pakai `nav2_simple_commander` | `navigation_transport.py:19,33` | Terbukti |
| 12 | Avoidance reaktif `lidar_controller.py` → `/controller/cmd_vel` | `lidar_controller.py:51,85,178-276` | Terbukti |

## Tabel koneksi: terverifikasi vs belum

| Koneksi | Sumber → Tujuan | Status | Bukti |
|---|---|---|---|
| LiDAR → `scan_raw` | driver → costmap/SLAM/AMCL | **Terverifikasi (statis)** | `ldlidar_LD19.launch.py:30`; `ms200_scan.launch.py:44` |
| `scan_raw` → costmap | topic → layer | **Terverifikasi (statis)** | `nav2_params.yaml:160,198` |
| costmap → planner → controller | layer → path → cmd | **Terverifikasi (statis)** | BT + launch |
| `cmd_vel_nav` → smoother → `cmd_vel` | remap | **Terverifikasi (statis)** | `navigation_base.launch.py:97,134` |
| `cmd_vel` → motor | `odom_publisher` → Board | **Terverifikasi (statis, Phase 1–2)** | `odom_publisher_node.py:144,246`; `ros_robot_controller_node.py:165` |
| `odom_raw` → EKF → `odom` | publisher → fuse → consume | **Terverifikasi (statis)** | `ekf.yaml:80`; `controller.launch.py:76` |
| `/scan_raw` benar-benar mengalir (frame/TF lengkap) | runtime | **BELUM TERVERIFIKASI** | butuh runtime |
| Nav2 mencapai lifecycle `active` | runtime | **BELUM TERVERIFIKASI** | butuh runtime |
| `teb_local_planner`/`nav2_*`/`slam_toolbox`/`rtabmap` terpasang | runtime | **BELUM TERVERIFIKASI** | tidak dideklarasikan |
| `odom_rf2o` aktif | runtime | **BELUM TERVERIFIKASI** | rf2o bukan di controller.launch default |
| obstacle benar ter-marking | runtime | **BELUM TERVERIFIKASI** | butuh runtime |
| Nilai env `MACHINE_TYPE`/`LIDAR_TYPE`/`HOST`/dll | runtime | **BELUM TERVERIFIKASI** | di luar repo |

## Component reusability table

| Komponen | Kategori | Alasan singkat |
|---|---|---|
| `mecanum.py` kinematika | **A. Reusable** | Matematis, independen hardware |
| SLAM 2D config | **A. Reusable** | Param standar slam_toolbox |
| NavFn/BT/smoother/velocity_smoother config | **A. Reusable** | Param standar Nav2 |
| `lidar_controller.py` (PID reaktif) | **A. Reusable** | Logika threshold/PID |
| `odom_publisher_node` | **B. Adaptasi** | Topik/frame/faktor koreksi |
| costmap + controller config | **B. Adaptasi** | Radius/sumber scan/`vy` |
| URDF/TF | **B. Adaptasi** | Geometri robot sendiri |
| Launch orchestration | **B. Adaptasi** | Env/path/namespace |
| App transport (BasicNavigator) | **B. Adaptasi** | Topik goal; pick/place |
| Board SDK + `ros_robot_controller` | **C. Hardware-specific** | Protokol serial |
| `servo_controller`, kalibrasi | **C. Hardware-specific** | Servo/fisik |
| LiDAR/depth camera driver | **C. Hardware-specific** | Perangkat spesifik |
| Dependency `teb_local_planner`/`nav2_*`/dll | **D. Unknown** | Tak dideklarasikan |
| Env runtime | **D. Unknown** | Tak ada di repo |

## Staged adaptation plan (ringkas)

1. **Identifikasi hardware & lingkungan target** — Pi/OS/distro/sensor/motor. Success: daftar terkonfirmasi.
2. **Verifikasi kontrol motor + konvensi mecanum** — Success: roda berputar sesuai perintah (roda diangkat).
3. **Verifikasi odometri & TF** — Success: `odom` halus, TF utuh. (Rekomendasi: encoder nyata > open-loop.)
4. **Verifikasi sensor & alignment TF** — Success: `scan` stabil, sejajar di RViz.
5. **Bring up Nav2 (peta minimal, param konservatif, `use_teb=false`)** — Success: lifecycle active, goal diproses.
6. **Verifikasi costmap** — Success: obstacle marking/clearing terlihat.
7. **Uji avoidance kecepatan rendah area terkendali** — Success: N rute tanpa kontak, recovery jalan.
8. **Tambah mapping/localization/perilaku lanjut** — Success: peta/lokalisasi stabil, waypoint benar.

Detail success/failure/rollback tiap tahap: `08_REUSE_AND_ADAPTATION.md`.

## Unresolved questions (butuh input pengguna / inspeksi robot)

1. Distro ROS 2 & model Raspberry Pi di robot sekolah (dan robot kustom target)?
2. Nilai runtime `MACHINE_TYPE`, `LIDAR_TYPE`, `DEPTH_CAMERA_TYPE`, `HOST`, `MASTER`, `need_compile`?
3. Apakah `teb_local_planner`, `nav2_*`, `slam_toolbox`, `rtabmap`, `robot_localization`,
   `nav2_simple_commander`, `rf2o_laser_odometry`, `imu_calib`, `imu_complementary_filter` terpasang?
4. Apakah `rf2o_laser_odometry` dijalankan (menentukan kualitas `odom`)?
5. Apakah robot kustom punya encoder roda? (menggantikan odometri open-loop)
6. Apakah strafing (lateral) mecanum ingin dimanfaatkan? (kini dinonaktifkan di Nav2)
7. Geometri fisik robot (radius/inflasi aman)?

## Next steps (setelah verifikasi)
- Bila `teb_local_planner` tidak tersedia → set `use_teb=false` (DWB) sebagai default aman.
- Bila ingin memanfaatkan mecanum: pertimbangkan costmap/controller yang mendukung `vy` (audit lanjut,
  di luar cakupan Phase 3 ini).
- Verifikasi TF & `odom` sebelum menambah SLAM.

## Batas bukti
Seluruh kesimpulan = analisis statis. **Tidak ada** klaim bahwa navigasi/obstacle avoidance
berfungsi di hardware. Status "terverifikasi" di dokumen ini berarti *terbukti ada & terhubung
di source*, bukan *terbukti berjalan*.
