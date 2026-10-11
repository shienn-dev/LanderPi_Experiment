# Batasan dan Langkah Berikutnya

> Checkpoint Phase 1–2: bagian "gap yang masih tersisa" dan "persiapan Phase 3" diperbarui.
> Phase 3 (audit algoritma navigasi/SLAM/obstacle avoidance) BELUM dimulai.

## Yang sudah didukung oleh source yang diperiksa
- Ada kelas `MecanumChassis` untuk menghitung kecepatan roda.
- `odom_publisher_node.py` menggunakan kelas tersebut, berlangganan `Twist`, dan menerbitkan `MotorsState`.
- Node `ros_robot_controller` mengirim data motor ke `Board.set_motor_speed(data)`.
- Launch controller menyertakan odometry, IMU filter, EKF, dan servo controller.
- Berbagai contoh menerbitkan perintah gerak `Twist`.
- **Baru (checkpoint):** node `odom_publisher` memiliki DUA subscription (`controller/cmd_vel` tanpa
  clamp, dan `cmd_vel` dengan clamp ±0.2); cabang chassis Mecanum/Tank/Ackermann terverifikasi;
  rantai Nav2 `cmd_vel_nav`→`cmd_vel`→`odom_publisher` terverifikasi statis.
- **Baru (checkpoint):** jalur feedback terverifikasi — `odom_raw` (open-loop dari perintah) + `imu`
  (dari `ros_robot_controller` via `imu_calib`+`imu_complementary_filter`) + `odom_rf2o` → EKF →
  `odom`; dikonsumsi Nav2 dan rtabmap.
- **Baru (checkpoint):** paket ament_python inti tidak mendeklarasikan dependency runtime di
  `package.xml` (hanya test_depend).

## Gap Phase 1–2 yang masih tersisa
- **Nilai runtime env `MACHINE_TYPE` dan `need_compile`** — hanya dibaca, tidak di-set di repo.
  Menentukan cabang chassis aktif dan path `compiled`/non-`compiled` pada launch. `BELUM TERVERIFIKASI`.
- **Instalasi dependency runtime** (`rf2o_laser_odometry`, `imu_calib`, `imu_complementary_filter`,
  `robot_localization`, `joy`, `nav2_*`, `slam_toolbox`/`rtabmap`, `pyserial`) di robot sekolah.
- **Resolusi nama topic runtime** (namespace/remapping aktual) dan TF nyata (`odom`→`base_footprint`,
  `imu_link`).
- **Arah/urutan fisik motor** dan kalibrasi (tanda `-motor1,-motor2,motor3,motor4`).
- **Keandalan odometri open-loop** (integrasi perintah, bukan encoder) — perlu validasi fisik.

## Yang belum pasti (dipertahankan)
- Nama topik final setelah namespace/remapping pada robot sekolah.
- Tipe chassis dan parameter yang benar-benar aktif.
- Urutan dan arah motor fisik.
- Protokol board secara rinci.
- Kecocokan implementasi navigasi/obstacle avoidance dengan varian hardware sekolah.

## Persiapan Phase 3 (navigasi otonom & obstacle avoidance)
Prioritas baca berikutnya (statis, tanpa menjalankan robot):
- `src/navigation/config/nav2_params.yaml` (penuh) — parameter controller/planner/costmap.
- `src/navigation/config/nav2_controller_dwb.yaml`, `nav2_controller_teb.yaml`,
  `nav2_controller_teb_ack.yaml` — pemilihan controller per tipe chassis.
- `src/navigation/launch/include/navigation_base.launch.py` & `localization.launch.py` — susunan nav2.
- `src/navigation/config/behavior_trees/*.xml` — logika recovery/backup.
- `src/app/app/lidar_controller.py` — obstacle avoidance/ikuti/patroli berbasis lidar (mode 1/2/3).
- `src/slam/launch/include/slam_base.launch.py` + `src/slam/config/*` — SLAM & map.
- `src/example/example/self_driving/*` dan `line_follow_clean` — obstacle/line avoidance contoh.

## File yang dibaca berikutnya (checkpoint terpenuhi)

### Command/Script
```bat
type src\driver\controller\controller\odom_publisher_node.py
```

### Output yang perlu dicari
Periksa inisialisasi `MecanumChassis`, kondisi pemilihan tipe mesin, `cmd_vel_callback`, publikasi `speeds`, serta callback odometry dan sumber data encoder/IMU.

### Kesimpulan
File ini menjadi prioritas karena menghubungkan perintah gerak dengan kinematika roda. Sesudah alur itu jelas, telusuri algoritma navigasi dan obstacle avoidance yang menerbitkan `Twist`.

### Status (checkpoint Phase 2)
`odom_publisher_node.py` SUDAH dibaca lengkap pada checkpoint ini: inisialisasi `MecanumChassis`
(baris 106/108), pemilihan tipe via `machine_type` (baris 105–108), dua `cmd_vel` callback
(baris 225/240), publikasi `MotorsState` (baris 138/247), dan odometri open-loop `cal_odom_fun`
(baris 291–325). Temuan dirujuk di `03_ALUR_KONTROL_ROS2.md`.
Catatan koreksi: tidak ditemukan pembacaan encoder pada node ini — odometri roda berbasis integrasi
perintah, bukan sensor encoder.

## Keselamatan uji fisik
Uji robot di sekolah dengan pengawasan guru/mentor, area kosong, dan kecepatan rendah. Saat memeriksa arah putaran, angkat roda dari lantai dan amankan robot sebelum menguji gerakan otonom.
