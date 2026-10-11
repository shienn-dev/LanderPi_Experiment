# Ringkasan Analisis LanderPi — 10 Oktober 2026

> Pembaruan checkpoint Phase 1–2 (lihat bagian "Status progres Phase 1–2" di bawah).
> Semua temuan ditandai `TERBUKTI DARI SOURCE`, `INFERENSI`, atau `BELUM TERVERIFIKASI`.
> Audit ini bersifat **statis** (baca source). Tidak ada node/launch yang dijalankan, tidak ada
> pengujian runtime, dan tidak ada klaim bahwa robot bergerak.

## Status progres Phase 1–2

### Phase 1 — Repository audit: SELESAI (dengan gap yang ditandai)

Yang berhasil ditutup dan diverifikasi dari source:
- Peta paket + entry point + metadata manifest untuk paket inti (lihat `01_PETA_FILE.md`).
- **Dependency: temuan penting.** Seluruh paket `ament_python` terkait (controller, kinematics,
  ros_robot_controller, servo_controller, navigation, slam, peripherals, app) **tidak
  mendeklarasikan dependency runtime apa pun** di `package.xml` — hanya `test_depend`
  (ament_copyright/flake8/pep257/pytest). Dependency nyata hanya terlihat dari `import` di source.
  → label `TERBUKTI DARI SOURCE`.
- Hubungan launch: `bringup.launch.py` → `controller.launch.py` → `odom_publisher.launch.py` +
  `imu_filter.launch.py` + node `ekf_filter_node` + `servo_controller.launch.py`
  (`TERBUKTI DARI SOURCE`).

Gap Phase 1 yang masih terbuka:
- Nilai runtime environment variable `MACHINE_TYPE` dan `need_compile` **tidak di-set di dalam
  repo** (hanya dibaca via `os.environ[...]`). → `BELUM TERVERIFIKASI`.
- Beberapa paket besar (large_models, large_models_examples, xf_mic_asr_offline, yolov8_detect,
  multi) belum diaudit mendalam karena di luar fokus chassis/nav (lihat `04_LANGKAH_LANJUT.md`).

### Phase 2 — Control-flow tracing: SELESAI untuk jalur mecanum

- Jalur perintah → motor terbukti (2 jalur `cmd_vel`), lihat `03_ALUR_KONTROL_ROS2.md`.
- Jalur feedback odometri terbukti: `odom_raw` (wheel/kinematik) + `imu` → EKF → `odom`.
- Antarmuka Nav2 `cmd_vel_nav` → `cmd_vel` terbukti secara statis.
- Yang tetap `BELUM TERVERIFIKASI`: resolusi topik runtime di robot sekolah (namespace/remapping
  aktual), tipe chassis yang benar-benar aktif, dan arah fisik motor.

### Batas bukti (evidence bounds)

Semua kesimpulan berasal dari pembacaan source statis. Analisis statis **bukan** pengujian
runtime. Eksistensi node/topic/topik yang terdaftar tidak membuktikan node berhasil berjalan.

## Status Git terkini yang telah diperiksa (checkpoint Phase 1–2)

### Command/Script
```bat
git status --short --branch
git branch -vv
git rev-list --left-right --count HEAD...@{u}
```

### Output
```text
## experiment/autonomy...origin/experiment/autonomy
 D sources/images/Doc/ls-files.txt
?? sources/Doc/
```
Divergensi dengan upstream: `0	0` (tidak ada commit ahead/behind).

### Kesimpulan
- **PERBAIKAN terhadap catatan lama:** status **tidak bersih**. Working tree memiliki 1 file
  terhapus (unstaged) `sources/images/Doc/ls-files.txt` dan direktori untracked `sources/Doc/`.
  (`TERBUKTI DARI SOURCE`.)
- Hasil investigasi: `sources/images/Doc/ls-files.txt` adalah file **kosong (0 byte)** yang
  ditambahkan oleh commit HEAD `fa05225`; direktori `sources/images/Doc/` sudah tidak ada di disk
  dan file bernama sama muncul di `sources/Doc/` — indikasi **relokasi direktori**, bukan
  penghapusan tak sengaja. **Belum dipulihkan / belum dihapus** (menunggu persetujuan pengguna).
- `ls-files.txt` kosong **tidak layak** dijadikan dokumentasi/commit.

## Temuan: pencarian jalur kontrol (bukti lama, tetap valid)

### Command/Script
```bat
git grep -n -E "MecanumChassis|set_velocity|set_motor|cmd_vel" -- src
```

### Output
Hasil menunjukkan:
- `src/driver/controller/controller/mecanum.py` berisi kelas `MecanumChassis`.
- `src/driver/controller/controller/odom_publisher_node.py` membuat objek mecanum, berlangganan `controller/cmd_vel`, dan menerbitkan `MotorsState` ke `ros_robot_controller/set_motor`.
- `src/driver/ros_robot_controller/ros_robot_controller/ros_robot_controller_node.py` berlangganan `~/set_motor` dan memanggil `self.board.set_motor_speed(data)`.
- Banyak aplikasi/contoh menerbitkan `Twist` ke `/controller/cmd_vel`.

### Kesimpulan
Jalur kontrol utama sudah ditemukan dari perintah gerak ROS 2 menuju perhitungan kecepatan roda dan API board. Resolusi nama topik aktual masih dapat dipengaruhi namespace/remapping runtime.

## Status Git yang dilaporkan sebelumnya (DIGANTIKAN oleh bagian di atas)

### Command/Script
```bat
git status
git branch -vv
```

### Output
Catatan sesi sebelumnya: repo `D:\Telkom_sch\RobotCeunahh\LanderPi`; `origin` adalah repo eksperimen pribadi dan `upstream` adalah `Hiwonder/LanderPi`; branch `baseline/hiwonder-original` di `e93be9a`, branch `experiment/autonomy` di `fa05225`; status terakhir dilaporkan bersih.

### Kesimpulan
Baseline dan eksperimen dipisahkan. **Catatan "status bersih" sudah tidak berlaku** — lihat bagian
"Status Git terkini" di atas (`D` pada `sources/images/Doc/ls-files.txt` dan `?? sources/Doc/`).
Tetap ulangi `git status` sebelum mengubah kode pada sesi berikutnya.
