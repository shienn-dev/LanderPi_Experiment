# Kinematika Mecanum

## Parameter chassis

### Command/Script
```bat
type src\driver\controller\controller\mecanum.py
```

### Output
Kelas didefinisikan sebagai:
```python
class MecanumChassis:
    def __init__(self, wheelbase=0.1368, track_width=0.1410, wheel_diameter=0.065):
```
Pencarian di `odom_publisher_node.py` juga menemukan pemanggilan dengan `track_width=0.1446`, dan diameter roda `0.065` atau `0.068`.

### Kesimpulan
Parameter aktual bergantung pada pemanggilan dan cabang kode. Jangan menganggap nilai default sebagai nilai yang pasti dipakai robot sekolah.

## Konversi kecepatan

### Command/Script
```python
def speed_covert(self, speed):
    return speed / (math.pi * self.wheel_diameter)
```

### Output
Kecepatan linear dibagi keliling roda untuk memperoleh putaran per detik (RPS).

### Kesimpulan
Fungsi mengubah kecepatan linear roda menjadi RPS. Nama `speed_covert` adalah ejaan persis dalam source.

## Penghitungan roda

### Command/Script
```python
def set_velocity(self, linear_x, linear_y, angular_z):
    motor1 = linear_x - linear_y - angular_z * (self.wheelbase + self.track_width) / 2
    motor2 = linear_x + linear_y - angular_z * (self.wheelbase + self.track_width) / 2
    motor3 = linear_x + linear_y + angular_z * (self.wheelbase + self.track_width) / 2
    motor4 = linear_x - linear_y + angular_z * (self.wheelbase + self.track_width) / 2
    v_s = [self.speed_covert(v) for v in [-motor1, -motor2, motor3, motor4]]
```

### Output
Empat nilai kecepatan dihitung dari gerak translasi dan rotasi, dikonversi ke RPS, lalu dimasukkan ke empat pesan `MotorState` dengan ID 1–4 dalam satu pesan `MotorsState`.

### Kesimpulan
`set_velocity()` mengubah perintah gerak chassis menjadi kecepatan roda. Arah fisik aktual perlu diverifikasi terhadap koordinat, pemasangan motor, dan konfigurasi robot.
