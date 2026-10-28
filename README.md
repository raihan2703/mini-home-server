# 🏠 Mini Home Server — STB Amlogic S905X

[![Armbian](https://img.shields.io/badge/OS-Armbian%20Linux-red?logo=armbian&logoColor=white)](https://www.armbian.com/)
[![SoC](https://img.shields.io/badge/SoC-Amlogic%20S905X-blue)]()
[![Docker](https://img.shields.io/badge/Docker-Engine%20%26%20Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Storage](https://img.shields.io/badge/Storage-eMMC%20%2B%20SanDisk%2032GB-green)]()

Repositori ini berisi dokumentasi lengkap, panduan setup, dan konfigurasi `docker-compose.yml` untuk membangun **Mini Home Server** berdaya rendah menggunakan perangkat STB Android TV berbasis chip **Amlogic S905X** yang telah di-flash dengan sistem operasi **Armbian Linux**.

---

## 📋 Latar Belakang & Spesifikasi Sistem

* **Perangkat:** STB Android TV (Amlogic S905X, RAM 1GB / 2GB)
* **Sistem Operasi:** Armbian Linux (Debian / Ubuntu aarch64)
* **Akses Terminal:** SSH sebagai pengguna `root`
* **Kondisi Storage Internal:** eMMC internal bawaan sangat terbatas (hanya tersisa sekitar `< 5 - 6 GB` yang dapat digunakan)
* **Storage Eksternal:** Flashdisk SanDisk 32GB via USB (mount point: `/mnt/sandisk32G`, filesystem: `ext4`)
* **Tantangan Utama:** Service seperti **Immich** memerlukan penyimpanan besar untuk backup foto/video, sementara Docker secara default menyimpan semua images, container layers, dan volume di internal storage (`/var/lib/docker`). Dokumen ini mencakup mitigasi lengkap untuk memindahkan penyimpanan Docker dan data backup Immich ke media eksternal SanDisk 32GB.

---

## 🛠️ Panduan Langkah Demi Langkah

---

### Langkah 1: Instalasi Docker & Docker Compose

Masuk ke terminal STB via SSH sebagai user `root`, lalu jalankan baris perintah berikut secara berurutan:

```bash
# 1. Update list paket dan perbarui paket sistem yang usang
apt update && apt upgrade -y

# 2. Download dan eksekusi script instalasi resmi dari Docker
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh

# 3. Pasang plugin resmi Docker Compose (Compose V2)
apt install -y docker-compose-plugin

# 4. Aktifkan dan jalankan daemon Docker secara otomatis saat booting
systemctl enable --now docker
```

Verifikasi hasil instalasi:
```bash
docker --version && docker compose version
```

---

### Langkah 2: Menyiapkan Folder Server & `docker-compose.yml`

Seluruh file konfigurasi layanan diatur di dalam direktori kerja `/root/server`:

```bash
# Buat folder server jika belum tersedia
mkdir -p /root/server

# Masuk ke dalam direktori server
cd /root/server
```

**Menyiapkan file `docker-compose.yml`:**
1. Jika kamu meng-clone repository ini ke STB, file `docker-compose.yml` sudah otomatis tersedia di dalam folder.
2. Jika kamu setup manual atau mengunggah file baru, buat file tersebut menggunakan editor nano:
   ```bash
   nano docker-compose.yml
   ```
   *(Tempelkan skrip konfigurasi compose, lalu simpan dengan menekan `Ctrl + O`, `Enter`, dan keluar dengan `Ctrl + X`)*.

---

### Langkah 3: Menjalankan Container

Setelah file `docker-compose.yml` tersedia di direktori `/root/server`, jalankan seluruh service di latar belakang (detached mode):

```bash
docker compose up -d
```

Periksa apakah seluruh container berjalan dengan normal:
```bash
docker compose ps
```

---

## 🚨 Mitigasi Masalah Storage: Pemindahan Docker & Immich ke Flashdisk SanDisk 32GB

### Masalah yang Dihadapi:
Saat server pertama kali dipasang, Docker berjalan normal (*healthy*) langsung di storage internal eMMC tanpa ada storage tambahan yang tertancap. Namun, ketika menjalankan service berat seperti **Immich**, kapasitas eMMC internal (< 5-6 GB) akan langsung penuh karena:
1. Docker menyimpan image layers dan cache di `/var/lib/docker`.
2. Immich menyimpan seluruh file hasil backup galeri foto/video handphone beserta thumbnail dan transcode-nya.

### Solusi:
Memindahkan direktori kerja Docker (`data-root`) beserta folder library upload Immich ke Flashdisk SanDisk 32GB (`/mnt/sandisk32G`).

---

### Bagian A: Persiapan & Mounting Flashdisk SanDisk 32GB Otomatis

Tancapkan Flashdisk SanDisk 32GB ke port USB STB, lalu jalankan langkah-langkah berikut:

1. **Cek partisi flashdisk yang terdeteksi:**
   ```bash
   lsblk
   ```
   > Perintah ini digunakan untuk melihat daftar seluruh media penyimpanan dan partisi yang terdeteksi oleh sistem (misalnya Flashdisk SanDisk terdeteksi sebagai `/dev/sda1`).

2. **Format partisi flashdisk ke filesystem Linux:**
   ```bash
   mkfs.ext4 -F /dev/sda1
   ```
   > Perintah ini digunakan untuk memformat partisi flashdisk ke filesystem `ext4` agar mendukung penuh hak akses Linux dan kompatibel dengan sistem perizinan Docker.

3. **Buat direktori titik pengait (mount point):**
   ```bash
   mkdir -p /mnt/sandisk32G
   ```
   > Perintah ini digunakan untuk membuat folder pengait baru di `/mnt/sandisk32G` sebagai titik masuk penyimpanan flashdisk.

4. **Ambil UUID unik partisi flashdisk:**
   ```bash
   blkid /dev/sda1
   ```
   > Perintah ini digunakan untuk menampilkan kode UUID unik dari flashdisk agar proses pengaitan partisi tidak tertukar meskipun urutan port USB berubah setelah restart.

5. **Daftarkan partisi ke fstab:**
   *(Ganti `PASTE_UUID_DISINI` dengan kode UUID hasil dari perintah nomor 4)*
   ```bash
   echo 'UUID=PASTE_UUID_DISINI /mnt/sandisk32G ext4 defaults,noatime 0 2' >> /etc/fstab
   ```
   > Perintah ini digunakan untuk mendaftarkan konfigurasi mounting flashdisk ke dalam file `/etc/fstab` agar partisi otomatis terpasang ke `/mnt/sandisk32G` setiap kali STB booting.

6. **Pasang partisi langsung tanpa restart:**
   ```bash
   mount -a
   ```
   > Perintah ini digunakan untuk menjalankan mounting seluruh partisi yang ada di fstab secara langsung tanpa perlu melakukan restart perangkat.

---

### Bagian B: Migrasi Data Docker yang Sedang Berjalan ke Flashdisk

Lakukan pemindahan data Docker yang sebelumnya aktif di internal ke flashdisk:

1. **Hentikan service Docker:**
   ```bash
   systemctl stop docker docker.socket
   ```
   > Perintah ini digunakan untuk menghentikan total service dan socket Docker agar tidak ada aktivitas baca-tulis selama proses penyalinan file berlangsung.

2. **Salin seluruh data Docker ke flashdisk:**
   ```bash
   rsync -avP /var/lib/docker/ /mnt/sandisk32G/docker/
   ```
   > Perintah ini digunakan untuk menyalin seluruh images, containers, dan volumes dari folder Docker lama ke flashdisk dengan tetap mempertahankan integritas hak akses file.

3. **Cadangkan direktori lama sebagai arsip:**
   ```bash
   mv /var/lib/docker /var/lib/docker.bak
   ```
   > Perintah ini digunakan untuk mengubah nama folder Docker lama menjadi file cadangan sementara guna memastikan data asli aman sebelum dihapus permanen.

4. **Arahkan `data-root` Docker ke flashdisk:**
   Jalankan perintah ini untuk membuat konfigurasi `/etc/docker/daemon.json`:
   ```bash
   cat <<EOF > /etc/docker/daemon.json
   {
     "data-root": "/mnt/sandisk32G/docker"
   }
   EOF
   ```
   > Perintah ini digunakan untuk mengonfigurasi direktori penyimpanan utama Docker agar berpindah ke flashdisk di folder `/mnt/sandisk32G/docker`.

5. **Jalankan kembali Docker:**
   ```bash
   systemctl start docker
   ```
   > Perintah ini digunakan untuk menyalakan kembali engine Docker menggunakan konfigurasi direktori penyimpanan yang baru di flashdisk.

6. **Verifikasi lokasi `data-root` baru:**
   ```bash
   docker info | grep "Docker Root Dir"
   ```
   > Perintah ini digunakan untuk memverifikasi dan memastikan bahwa path penyimpanan Docker telah berhasil berpindah ke `/mnt/sandisk32G/docker`.

7. **Hapus cadangan lama:**
   ```bash
   rm -rf /var/lib/docker.bak
   ```
   > Perintah ini digunakan untuk menghapus folder cadangan lama di penyimpanan internal eMMC sehingga kapasitas internal STB langsung lega kembali.

---

### Bagian C: Konfigurasi Penyimpanan Immich & Struktur Folder

Agar file backup foto Immich tersimpan langsung di flashdisk, pastikan file konfigurasi `.env` di direktori `/root/server` sudah sesuai:

1. **Verifikasi nilai `UPLOAD_LOCATION` pada file `.env`:**
   ```bash
   cat /root/server/.env | grep UPLOAD_LOCATION
   ```
   > Perintah ini digunakan untuk memastikan variabel path upload mengarah ke flashdisk dengan output:
   ```env
   UPLOAD_LOCATION=/mnt/sandisk32G/immich-app/library
   ```

2. **Buat direktori library tersebut jika belum ada:**
   ```bash
   mkdir -p /mnt/sandisk32G/immich-app/library
   ```
   > Perintah ini digunakan untuk menyiapkan folder target di flashdisk sebelum container Immich dinyalakan.

3. **Jalankan kembali Docker Compose:**
   ```bash
   cd /root/server && docker compose up -d
   ```
   > Perintah ini digunakan untuk menerapkan konfigurasi penyimpanan baru dan menyalakan kembali seluruh container Immich.

---

### 📂 Anatomi Struktur Folder Storage Immich di Flashdisk

Setelah container Immich berjalan dengan path `/mnt/sandisk32G/immich-app/library`, sistem secara otomatis menyusun direktori penyimpanan media sebagai berikut:

```text
/mnt/sandisk32G/immich-app/library
├── library/
│   └── <admin-user-id>/
│       └── 2026/
│           └── 2026-08-07/
│               └── IMG_0001.jpg
├── upload/         (Folder temporary / bilik upload user saat transfer data berlangsung)
├── thumbs/         (Thumbnail resolusi kecil untuk mempercepat loading foto di aplikasi)
└── profile/        (Foto avatar/profil pengguna Immich)
```

**Penjelasan Alur Penyimpanan:**
* **`library/`**: Lokasi penyimpanan permanen file foto/video yang telah berhasil diindeks, terkelompokkan secara terstruktur berdasarkan ID Pengguna (`<user-id>`), Tahun, dan Tanggal pengambilan media.
* **`upload/`**: Direktori penampung sementara (*staging area/bilik transfer*) saat handphone sedang mengunggah file foto sebelum diproses dan dipindahkan ke `library/`.
* **`thumbs/`**: Menyimpan thumbnail hasil kompresi agar aplikasi mobile Immich dapat membuka galeri dengan cepat dan hemat bandwidth/RAM STB.
* **`profile/`**: Menyimpan foto profil masing-masing akun pengguna yang terdaftar di Immich server.

---

## 📊 Pemantauan Sistem & Maintenance Rutin

Gunakan beberapa command di bawah ini untuk memantau performa mini server:

* **Melihat sisa kapasitas storage eMMC dan Flashdisk SanDisk:**
  ```bash
  df -h
  ```
* **Melihat penggunaan RAM, CPU, dan resource per kontainer:**
  ```bash
  docker stats
  ```
* **Melihat beban kerja sistem dan suhu STB:**
  ```bash
  htop
  # atau cek suhu SoC langsung:
  armbianmonitor -m
  ```
