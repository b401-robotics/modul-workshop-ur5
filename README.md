# Modul Workshop UR5

Reposeitory modul workshop dan simulasi untuk robot arm **Universal Robots UR5** menggunakan **URSim**, **driver ROS 2 untuk UR**, dan **Foxglove Studio** untuk visualisasi.

---

## Daftar Isi

- [Gambaran Umum](#gambaran-umum)
- [Requirement](#requirement)
- [Struktur Repositori](#struktur-repositori)
- [Memulai](#memulai)
  - [1. Clone Repositori](#1-clone-repositori)
  - [2. Build dan Jalankan Stack](#2-build-dan-jalankan-stack)
  - [3. Akses Interface URSim](#3-akses-interface-ursim)
  - [4. Visualisasi dengan Foxglove Studio](#4-visualisasi-dengan-foxglove-studio)
- [Referensi Port](#referensi-port)
- [Menghentikan Stack](#menghentikan-stack)
- [Troubleshooting](#troubleshooting)
- [[Teknis] Arsitektur](#arsitektur)
- [[Teknis] Docker Services](#docker-services)

---

## Gambaran Umum

Repositori ini menyediakan stack simulasi yang sepenuhnya terkontainerisasi sehingga kamu dapat:

- Menjalankan **simulasi robot arm UR5** menggunakan image Docker resmi URSim CB3 (tanpa memerlukan robot fisik).
- Menghubungkan **driver ROS 2 Humble** (`ur_robot_driver`) ke simulator melalui protokol standar UR RTDE.
- Melakukan streaming semua topik ROS 2 (joint states, TF, dll.) ke **Foxglove Studio** melalui WebSocket bridge untuk visualisasi secara real-time.

Semua layanan berjalan di dalam container Docker pada jaringan bridge bersama, sehingga tidak diperlukan instalasi ROS 2 tambahan di host.

---

## Requirement

Sebelum memulai, pastikan perangkat lunak berikut sudah terinstal di mesin host:

| Kebutuhan | Versi | Catatan |
|---|---|---|
| [Docker Engine](https://docs.docker.com/engine/install/) | ≥ 24.x | Dengan BuildKit diaktifkan |
| [Docker Compose](https://docs.docker.com/compose/install/) | ≥ 2.x | (plugin `docker compose` atau standalone) |
| [Foxglove Studio](https://foxglove.dev/) | Terbaru | Aplikasi desktop atau [web app](https://app.foxglove.dev) |

---

## Struktur Repositori

```
modul-workshop-ur5/
├── ursim_simulation/
|   ├── Modul Workshop UR5.pdf
|   └── readme.md
├── ursim_simulation/
│   ├── Dockerfile            # Image ROS 2 Humble dengan UR driver & Foxglove bridge
│   └── docker-compose.yaml   
└── README.md
```

---

## Memulai

### 1. Clone Repositori

```bash
git clone https://github.com/<your-org>/modul-workshop-ur5.git
cd modul-workshop-ur5
```

### 2. Build dan Jalankan Stack

Masuk ke direktori simulasi dan jalankan semua layanan:

```bash
cd ursim_simulation
docker compose up --build
```

Docker Compose akan melakukan:
1. Mengunduh image `universalrobots/ursim_cb3:latest`.
2. Membangun image ROS 2 kustom dari `Dockerfile` lokal.
3. Menjalankan container **URSim** dan menunggu hingga health check berhasil (port TCP 30002 dapat dijangkau).
4. Meluncurkan **driver ROS 2 UR**, menunggu 15 detik untuk handshake RTDE, lalu menjalankan `joint_state_broadcaster` dan `scaled_joint_trajectory_controller`.
5. Menjalankan **Foxglove bridge** setelah driver aktif.

> **Pertama kali menjalankan** akan memakan beberapa menit untuk mengunduh image dan membangun image ROS. Eksekusi berikutnya akan jauh lebih cepat.

Untuk menjalankan dalam mode *detached* (latar belakang):

```bash
docker compose up --build -d
```

Untuk memantau log layanan tertentu:

```bash
docker compose logs -f ur_driver
docker compose logs -f foxglove_bridge
docker compose logs -f ursim
```

### 3. Akses Interface URSim

Setelah container `ursim` berjalan, buka browser dan kunjungi:

```
http://localhost:6080/vnc.html
```

Ini akan membuka **noVNC** remote desktop yang menampilkan GUI Teach Pendant UR (Polyscope). Tekan tombol connect untuk mengakses GUI URSim. Kamu dapat:

- Memuat dan menjalankan program UR.
- Menggerakkan sendi robot secara manual.
- Mengaktifkan / menonaktifkan robot dari tab **Run**.

### 4. Visualisasi dengan Foxglove Studio

1. Buka **Foxglove Studio** (desktop atau [web](https://app.foxglove.dev)).
2. Klik **Open Connection**.
3. Pilih **Foxglove WebSocket** dan masukkan:
   ```
   ws://localhost:8765
   ```
4. Klik **Open**.

Semua topik ROS 2 yang aktif akan muncul di panel kiri. Topik yang berguna antara lain:

| Topik | Tipe | Deskripsi |
|---|---|---|
| `/joint_states` | `sensor_msgs/JointState` | Posisi, kecepatan, dan gaya sendi saat ini |
| `/tf` | `tf2_msgs/TFMessage` | Transformasi frame koordinat robot |
| `/robot_description` | `std_msgs/String` | URDF robot UR5 |

Tambahkan panel **3D** di Foxglove dan atur **Display Frame** ke `base` untuk melihat model robot dirender secara real-time.

---

## Referensi Port

| Port | Protokol | Layanan | Deskripsi |
|---|---|---|---|
| `6080` | HTTP | URSim | GUI Teach Pendant berbasis browser (noVNC) |
| `29999` | TCP | URSim | UR Dashboard Server |
| `30001` | TCP | URSim | UR Primary Interface |
| `30002` | TCP | URSim | UR RTDE (Real-Time Data Exchange) |
| `30003` | TCP | URSim | UR Real-Time Interface |
| `8765` | WebSocket | Foxglove Bridge | Streaming topik ROS 2 ke Foxglove Studio |

---

## Menghentikan Stack

Untuk menghentikan semua container yang berjalan:

```bash
docker compose down
```

Untuk menghentikan **dan menghapus** semua container, jaringan, serta volume (reset penuh):

```bash
docker compose down -v
```

---

## Troubleshooting

### URSim tidak dapat dijangkau / driver gagal terhubung

- Tunggu hingga health check `ursim` berhasil. Driver bergantung padanya, tetapi Polyscope dapat memakan waktu 60–90 detik untuk booting penuh.
- Periksa log URSim: `docker compose logs ursim`
- Pastikan robot sudah **dinyalakan** dan berada dalam mode **Remote Control** di GUI Polyscope pada `http://localhost:6080`.

### Foxglove Studio tidak menampilkan topik

- Pastikan container `foxglove_bridge` berjalan: `docker compose ps`
- Periksa log bridge untuk error: `docker compose logs foxglove_bridge`
- Verifikasi URL koneksi adalah `ws://localhost:8765`.
- Pastikan `ur_driver` sudah sepenuhnya aktif sebelum bridge (sudah diatur via `depends_on`).

### Controller gagal dijalankan

- Driver membutuhkan handshake RTDE agar berhasil terlebih dahulu. Jika robot belum sepenuhnya aktif, jeda 15 detik mungkin tidak cukup.
- Restart layanan driver saja: `docker compose restart ur_driver`

### Port sudah digunakan

- Hentikan proses ROS 2 atau Foxglove lain di host yang menggunakan port yang sama.
- Periksa dengan: `ss -tulpn | grep -E '6080|8765|3000[1-3]|29999'`

---

## Arsitektur

```
┌────────────────────────────────────────────────────────────────┐
│                    Docker Network: ros_robot_net               │
│                                                                │
│  ┌──────────────┐     RTDE/TCP      ┌──────────────────────┐   │
│  │   ursim_cb3  │ ◄──────────────── │   ur_ros2_driver     │   │
│  │  (URSim CB3) │   30001-30003     │  (ur_robot_driver +  │   │
│  │              │   29999           │   controller_manager)│   │
│  └──────┬───────┘                   └──────────┬───────────┘   │
│         │                                      │ /joint_states │
│  noVNC  │ :6080                                │ /tf, dll.     │
│         │                               ┌──────▼───────────┐   │
│         │                               │  foxglove_bridge │   │
│         │                               │  WebSocket :8765 │   │
│         │                               └──────────────────┘   │
└─────────┼──────────────────────────────────────┼───────────────┘
          │ Host                                 │ Host
     http://localhost:6080               ws://localhost:8765
        (GUI URSim)                       (Foxglove Studio)
```

---

## Docker Services

### `ursim` — Simulator URSim CB3

| Parameter | Nilai |
|---|---|
| Image | `universalrobots/ursim_cb3:latest` |
| Model Robot | `UR5` |
| Akses GUI | `http://localhost:6080` (noVNC) |
| Port RTDE | `30002` |

Program UR disimpan secara persisten dalam volume Docker bernama (`ursim_programs`).

---

### `ur_driver` — Driver ROS 2 UR Robot

Dibangun dari [Dockerfile](ursim_simulation/Dockerfile) lokal berbasis `ros:humble`.

Paket yang terinstal:
- `ros-humble-ur-robot-driver` — Driver ROS 2 resmi UR
- `ros-humble-ur-simulation-gz` — Dukungan simulasi Gazebo

Saat dijalankan:
1. Meluncurkan `ur_robot_driver` dengan `robot_ip:=ursim` (diselesaikan melalui DNS Docker).
2. Menunggu 15 detik untuk handshake RTDE.
3. Menjalankan controller `joint_state_broadcaster`.
4. Menjalankan controller `scaled_joint_trajectory_controller`.

---

### `foxglove_bridge` — Foxglove WebSocket Bridge

Juga dibangun dari Dockerfile lokal.

Paket yang terinstal:
- `ros-humble-foxglove-bridge` — Bridge WebSocket ROS 2 ↔ Foxglove

Mengekspos semua topik ROS 2 melalui WebSocket di port `8765` untuk dikonsumsi oleh Foxglove Studio.

---

## Lisensi

Lihat [LICENSE](LICENSE) untuk detail selengkapnya.
