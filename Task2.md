# Task 2 — SentinelOS

## A. Mempersiapkan Kernel Linux

### Deskripsi

Pada poin A, dilakukan pengunduhan dan kompilasi **Linux kernel versi 6.1.1** dengan konfigurasi minimal untuk arsitektur x86_64. Kernel dikonfigurasi menggunakan `make tinyconfig` sebagai basis kemudian ditambahkan fitur-fitur esensial melalui `make menuconfig`. Output akhir adalah file `bzImage`.

### Langkah-langkah & Potongan Kode

#### 1. Siapkan Direktori Kerja

```bash
mkdir -p osboot
cd osboot
```

- Semua proses build kernel dan rootfs dilakukan di dalam folder `osboot` ini.

#### 2. Download dan Ekstrak Kernel Linux

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.1.tar.xz
tar -xvf linux-6.1.1.tar.xz
cd linux-6.1.1
```

- Kernel 6.1.1 diunduh langsung dari `kernel.org` sebagai sumber resmi
- Diekstrak dengan `tar -xvf` karena format `.tar.xz`.

#### 3. Konfigurasi Kernel

```bash
make tinyconfig
make menuconfig
```

- `make tinyconfig` : menghasilkan konfigurasi minimal sebagai titik awal (lebih kecil dari `defconfig`)
- `make menuconfig` : membuka antarmuka TUI untuk mengaktifkan fitur-fitur yang dibutuhkan secara manual.

Di dalam `menuconfig`, aktifkan opsi-opsi berikut:

Setelah selesai konfigurasi, simpan dan keluar dari menuconfig.

#### 4. Kompilasi Kernel

```bash
make -j$(nproc)
```

- `-j$(nproc)` : menggunakan seluruh core CPU yang tersedia agar kompilasi lebih cepat
- Proses ini membutuhkan waktu **30–60 menit** tergantung spesifikasi mesin.

#### 5. Copy bzImage ke Direktori osboot

```bash
cp arch/x86/boot/bzImage ..
cd ..
```

- File `bzImage` hasil kompilasi ada di `arch/x86/boot/`
- Di-copy ke direktori `osboot` untuk digunakan di langkah selanjutnya.

### Screenshot
<img width="1562" height="672" alt="image" src="https://github.com/user-attachments/assets/4ca0e177-35f3-4cc7-87cf-e41e1b230747" />
<img width="1562" height="817" alt="image" src="https://github.com/user-attachments/assets/88cc45ca-df50-4b18-abde-f23fae81412b" />
<img width="1562" height="817" alt="image" src="https://github.com/user-attachments/assets/34b56b7a-dc4f-4653-b4a8-cd325a83fbb9" />
<img width="1562" height="542" alt="image" src="https://github.com/user-attachments/assets/2c64f547-5d28-4fad-9fdd-c202078062f2" />
<img width="1081" height="814" alt="image" src="https://github.com/user-attachments/assets/11751d4a-4d31-4183-87f5-1dc76f087be0" />
<img width="1207" height="700" alt="image" src="https://github.com/user-attachments/assets/f6ee5273-8298-4dd7-8386-a20ab4b65460" />



### Kode Penuh

```bash
# Dari direktori osboot/
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.1.tar.xz
tar -xvf linux-6.1.1.tar.xz
cd linux-6.1.1

make tinyconfig
make menuconfig
# (aktifkan opsi-opsi sesuai tabel di atas, lalu simpan)

make -j$(nproc)
cp arch/x86/boot/bzImage ..
cd ..
```

---

## B. Membangun Sistem File Root Awal

### Deskripsi

Pada poin B, dibangun **root filesystem (rootfs)** yang menjadi lingkungan dasar SentinelOS. Rootfs berisi struktur direktori sistem, binary BusyBox, device nodes minimal, dan file notifikasi `/log/NOTICE`.

### Langkah-langkah & Potongan Kode

#### 1. Masuk ke Mode Superuser

```bash
sudo bash
```

Seluruh proses pembuatan rootfs dilakukan sebagai root agar operasi `mknod`, `chown`, dan pengaturan permission berjalan tanpa hambatan.

Di Garuda Linux (Arch-based), BusyBox biasanya berada di `/usr/bin/busybox`.

#### 2. Buat Struktur Direktori Rootfs

```bash
mkdir -p myramdisk/{bin,dev,etc,proc,sys,tmp,vault,watch,log,root}
mkdir -p myramdisk/home/{guardian,observer}
```

Struktur yang terbentuk:

```
myramdisk/
├── bin/          # BusyBox dan applet-appletnya
├── dev/          # Device nodes
├── etc/          # Konfigurasi sistem
├── proc/         # Mount point procfs
├── sys/          # Mount point sysfs
├── tmp/          # File sementara
├── vault/        # Direktori eksklusif root
├── watch/        # Direktori eksklusif guardian
├── log/          # Log dan notifikasi sistem
├── root/         # Home root
└── home/
    ├── guardian/ # Home guardian
    └── observer/ # Home observer
```

#### 3. Salin File Device ke Direktori `dev`

```bash
cp -a /dev/null    myramdisk/dev
cp -a /dev/tty*    myramdisk/dev
cp -a /dev/zero    myramdisk/dev
cp -a /dev/console myramdisk/dev
```

- `/dev/null` : membuang output yang tidak diperlukan
- `/dev/tty*` : terminal virtual yang dibutuhkan untuk login dan shell
- `/dev/zero` : sumber byte nol
- `/dev/console` : konsol sistem utama.

#### 4. Salin dan Install BusyBox

```bash
cp /usr/bin/busybox myramdisk/bin
cd myramdisk/bin
./busybox --install .
cd ../..
```

- `./busybox --install .` : membuat symlink untuk semua applet BusyBox (ls, cat, mount, login, getty, dll) ke dalam direktori `bin`
- Satu binary BusyBox menggantikan puluhan perintah Unix standar.

#### 5. Buat File `/log/NOTICE`

```bash
cat > myramdisk/log/NOTICE << 'EOF'
=======================================================
  SENTINELOS - SISTEM OPERASI SENTINEL
=======================================================

PEMBERITAHUAN KEAMANAN:
  Akses ke sistem ini hanya diperbolehkan untuk
  pengguna yang berwenang. Seluruh aktivitas dicatat
  dan dipantau oleh administrator sistem.

DIREKTORI TERBATAS:
  /vault  - Hanya dapat diakses oleh root
  /watch  - Hanya dapat diakses oleh guardian
  /home   - Setiap user hanya dapat mengakses home-nya

Hubungi administrator jika ada masalah.
                     -- SentinelOS Security Team
=======================================================
EOF
```

File ini akan ditampilkan otomatis ke setiap user saat login melalui `/etc/profile`.

### Kode Penuh

```bash
sudo bash

mkdir -p myramdisk/{bin,dev,etc,proc,sys,tmp,vault,watch,log,root}
mkdir -p myramdisk/home/{guardian,observer}

cp -a /dev/null    myramdisk/dev
cp -a /dev/tty*    myramdisk/dev
cp -a /dev/zero    myramdisk/dev
cp -a /dev/console myramdisk/dev

cp /usr/bin/busybox myramdisk/bin
cd myramdisk/bin
./busybox --install .
cd ../..

cat > myramdisk/log/NOTICE << 'EOF'
=======================================================
  SENTINELOS - SISTEM OPERASI SENTINEL
=======================================================

PEMBERITAHUAN KEAMANAN:
  Akses ke sistem ini hanya diperbolehkan untuk
  pengguna yang berwenang. Seluruh aktivitas dicatat
  dan dipantau oleh administrator sistem.

DIREKTORI TERBATAS:
  /vault  - Hanya dapat diakses oleh root
  /watch  - Hanya dapat diakses oleh guardian
  /home   - Setiap user hanya dapat mengakses home-nya

Hubungi administrator jika ada masalah.
                     -- SentinelOS Security Team
=======================================================
EOF
```
### Screenshoot
<img width="1207" height="963" alt="image" src="https://github.com/user-attachments/assets/a29d0a61-200c-4f09-9e14-070646684bfd" />


---

## C. Mengimplementasikan Sistem Multi-User dan Keamanan

### Deskripsi

Pada poin C, diimplementasikan sistem autentikasi dengan **3 user** yang masing-masing dilindungi password terenkripsi **MD5**. Setiap user hanya dapat mengakses direktori yang sesuai dengan perannya. Password dienkripsi menggunakan `openssl` dan disimpan di `/etc/passwd`. Permission direktori diatur dengan `chmod` dan `chown`.

### Langkah-langkah & Potongan Kode

#### 1. Generate Password Hash MD5

Sebelum membuat file `passwd`, generate dulu hash MD5 untuk masing-masing password menggunakan `openssl`:

```bash
openssl passwd -1 SkyLord
openssl passwd -1 Guard123
openssl passwd -1 EyeOpen
```

- `openssl passwd -1` : menghasilkan hash MD5 dengan format `$1$<salt>$<hash>`
- Salt di-generate secara acak setiap kali perintah dijalankan, sehingga hash-nya selalu berbeda meski password sama — namun tetap valid saat autentikasi

#### 2. Buat File `/etc/passwd`

```bash
cat > myramdisk/etc/passwd << 'EOF'
root:$1$/C.1cnJa$.SRkwopmH1fMXFaj0VQ9n0:0:0:root:/root:/bin/sh
guardian:$1$tHWp7bLy$3e5VCucLS6/3DytS9wzmh/:1000:1000:Guardian User:/home/guardian:/bin/sh
observer:$1$wygrdZ5P$oHyIfsvYmP3zngADjmAN90:1001:1001:Observer User:/home/observer:/bin/sh
EOF
```

> **Catatan:** Ganti hash di atas dengan hasil `openssl passwd` yang kamu jalankan sendiri, karena salt-nya berbeda setiap kali.

Format setiap baris `/etc/passwd`:

```
username : hash_password : UID : GID : komentar : home_dir : shell
```

| Field | root | guardian | observer |
|-------|------|----------|----------|
| UID | 0 | 1000 | 1001 |
| GID | 0 | 1000 | 1001 |
| Home | `/root` | `/home/guardian` | `/home/observer` |
| Shell | `/bin/sh` | `/bin/sh` | `/bin/sh` |

#### 3. Buat File `/etc/group`

```bash
cat > myramdisk/etc/group << 'EOF'
root:x:0:root
bin:x:1:root
sys:x:2:root
tty:x:5:root,guardian,observer
disk:x:6:root
wheel:x:10:root,guardian
users:x:100:guardian,observer
guardian:x:1000:guardian
observer:x:1001:observer
EOF
```

- Group `tty` berisi ketiga user agar semua bisa mengakses terminal
- Group `wheel` hanya untuk `root` dan `guardian` (privilege lebih tinggi)
- `guardian` dan `observer` masuk group `users` sebagai grup umum.

#### 4. Mengatur Permission Direktori sesuai Role

```bash
# /vault -> hanya root (rwx------)
chmod 700 myramdisk/vault
chown 0:0 myramdisk/vault

# /watch -> hanya guardian
chmod 700 myramdisk/watch
chown 1000:1000 myramdisk/watch

# /home/guardian -> hanya guardian
chmod 700 myramdisk/home/guardian
chown 1000:1000 myramdisk/home/guardian

# /home/observer -> hanya observer
chmod 700 myramdisk/home/observer
chown 1001:1001 myramdisk/home/observer

# /root -> hanya root
chmod 700 myramdisk/root
chown 0:0 myramdisk/root

# /log -> semua user bisa baca
chmod 755 myramdisk/log
```

- `chmod 700` berarti hanya owner yang bisa baca, tulis, dan masuk direktori tersebut (rwx------). User lain tidak bisa sama sekali
- `chown UID:GID` mengatur kepemilikan direktori ke user yang sesuai.

Ringkasan permission:

| Direktori | Owner | Permission | Dapat Diakses Oleh |
|-----------|-------|------------|-------------------|
| `/vault` | root:root | `700` | root saja |
| `/watch` | guardian:guardian | `700` | guardian saja |
| `/home/guardian` | guardian:guardian | `700` | guardian saja |
| `/home/observer` | observer:observer | `700` | observer saja |
| `/root` | root:root | `700` | root saja |
| `/log` | root:root | `755` | semua user |

### Screenshot
<img width="661" height="211" alt="image" src="https://github.com/user-attachments/assets/9c4e9321-7f38-4235-b8c7-8041ee5e626f" />
<img width="1249" height="171" alt="image" src="https://github.com/user-attachments/assets/74655624-2bb1-42d7-a163-f7f929354011" />
<img width="1352" height="203" alt="image" src="https://github.com/user-attachments/assets/1c38b0d3-6837-40a7-93fa-cc1db94c173f" />



### Kode Penuh

```bash
# Generate hash MD5 (jalankan satu per satu, salin hasilnya)
openssl passwd -1 SkyLord
openssl passwd -1 Guard123
openssl passwd -1 EyeOpen

# Buat /etc/passwd (ganti hash dengan hasil generate di atas)
cat > myramdisk/etc/passwd << 'EOF'
root:<hash_root>:0:0:root:/root:/bin/sh
guardian:<hash_guardian>:1000:1000:Guardian User:/home/guardian:/bin/sh
observer:<hash_observer>:1001:1001:Observer User:/home/observer:/bin/sh
EOF

# Buat /etc/group
cat > myramdisk/etc/group << 'EOF'
root:x:0:root
bin:x:1:root
sys:x:2:root
tty:x:5:root,guardian,observer
disk:x:6:root
wheel:x:10:root,guardian
users:x:100:guardian,observer
guardian:x:1000:guardian
observer:x:1001:observer
EOF

# Set permission direktori
chmod 700 myramdisk/vault
chmod 700 myramdisk/watch
chmod 700 myramdisk/home/guardian
chmod 700 myramdisk/home/observer
chmod 700 myramdisk/root
chmod 755 myramdisk/log

chown 0:0    myramdisk/vault
chown 1000:1000 myramdisk/watch
chown 1000:1000 myramdisk/home/guardian
chown 1001:1001 myramdisk/home/observer
chown 0:0    myramdisk/root
```

---

## D. Uji Coba Boot dan Customization Terminal

### Deskripsi

Pada poin D, dibuat **init script** sebagai program pertama yang dijalankan kernel (PID 1). Init script me-mount filesystem virtual yang dibutuhkan sistem, membaca parameter `mode` dari kernel cmdline untuk menentukan warna prompt, menulis `/etc/profile` secara dinamis, lalu menjalankan `getty` sebagai login prompt. Setelah itu, rootfs dikemas menjadi `initramfs` dan diuji dengan QEMU.

### Langkah-langkah & Potongan Kode

#### 1. Buat File `init`

```bash
cat > myramdisk/init << 'EOF'
#!/bin/sh
/bin/mount -t proc none /proc
/bin/mount -t sysfs none /sys
/bin/mount -t devtmpfs none /dev

/bin/hostname sentinelos

# Baca parameter mode dari kernel cmdline
MODE=$(cat /proc/cmdline | tr ' ' '\n' | grep '^mode=' | cut -d= -f2)
[ -z "$MODE" ] && MODE=1

# Tentukan warna prompt berdasarkan mode
case "$MODE" in
    1)
        USER_CLR='\[\e[1;32m\]'
        HOST_CLR='\[\e[1;34m\]'
        LABEL="DEFAULT"
        ;;
    2)
        USER_CLR='\[\e[1;33m\]'
        HOST_CLR='\[\e[1;36m\]'
        LABEL="WATCH"
        ;;
    3)
        USER_CLR='\[\e[1;31m\]'
        HOST_CLR='\[\e[1;35m\]'
        LABEL="SECURE"
        ;;
    *)
        USER_CLR='\[\e[1;32m\]'
        HOST_CLR='\[\e[1;34m\]'
        LABEL="DEFAULT"
        ;;
esac

# Tulis /etc/profile secara dinamis sesuai mode
cat > /etc/profile << PROFILE
export PATH=/bin:/sbin
export TERM=linux
export SENTINELOSMODE="${LABEL}"
PS1='${USER_CLR}[\u${HOST_CLR}@\h\[\e[0m\] \W${USER_CLR}]\[\e[0m\]\$ '
export PS1
[ -f /log/NOTICE ] && cat /log/NOTICE
PROFILE

# Jalankan login prompt di tty1
while true; do
    /bin/getty -L tty1 115200 vt100
    sleep 1
done
EOF
```

Penjelasan setiap bagian:

- `mount -t proc none /proc` : mount procfs, wajib ada agar `/proc/cmdline` bisa dibaca dan perintah seperti `ps` berjalan
- `mount -t sysfs none /sys` : mount sysfs untuk komunikasi kernel-userspace
- `mount -t devtmpfs none /dev` : mount devtmpfs agar device nodes tersedia otomatis tanpa perlu mknod manual
- `hostname sentinelos` : set hostname sistem
- `cat /proc/cmdline | tr ' ' '\n' | grep '^mode=' | cut -d= -f2` : mengambil nilai `mode` dari parameter kernel, misalnya jika boot dengan `-append "mode=2"` maka `MODE=2`
- Blok `case "$MODE"` : menentukan kode warna ANSI dan label sesuai mode
- `/etc/profile` ditulis dinamis setiap boot sehingga warna prompt selalu sesuai mode
- `while true; do /bin/getty ... done` : loop agar getty selalu restart jika user logout, sehingga login prompt selalu muncul kembali

Kode warna ANSI yang digunakan:

| Kode ANSI | Warna | Dipakai di Mode |
|-----------|-------|----------------|
| `\e[1;32m` | Hijau terang | 1 (user) |
| `\e[1;34m` | Biru terang | 1 (host) |
| `\e[1;33m` | Kuning terang | 2 (user) |
| `\e[1;36m` | Cyan terang | 2 (host) |
| `\e[1;31m` | Merah terang | 3 (user) |
| `\e[1;35m` | Magenta terang | 3 (host) |

#### 2. Berikan Izin Eksekusi pada File `init`

```bash
chmod +x myramdisk/init
```

Kernel Linux **wajib** menemukan file `/init` yang executable. Jika tidak, kernel akan panic saat booting.

#### 3. Buat Initramfs

```bash
cd myramdisk
find . | cpio -oHnewc | gzip > ../myramdisk.gz
cd ..
```

- `find .` : mencantumkan semua file dan direktori di dalam `myramdisk/`
- `cpio -oHnewc` : membuat arsip CPIO dengan format `newc` — satu-satunya format yang dikenali kernel Linux untuk initramfs
- `gzip` : mengompresi arsip sehingga lebih kecil dan cepat dimuat ke memori
- Output: file `myramdisk.gz` di direktori `osboot/`

#### 4. Jalankan QEMU untuk Test Boot

```bash
qemu-system-x86_64 \
  -smp 2 \
  -m 256 \
  -display curses \
  -vga std \
  -kernel bzImage \
  -initrd myramdisk.gz \
  -append "mode=1"
```

| Opsi | Fungsi |
|------|--------|
| `-smp 2` | 2 virtual CPU |
| `-m 256` | RAM 256 MB |
| `-display curses` | Output teks di terminal, tanpa jendela GUI |
| `-vga std` | VGA standar resolusi 80x25 |
| `-kernel bzImage` | File kernel yang digunakan |
| `-initrd myramdisk.gz` | File initramfs yang dimuat |
| `-append "mode=1"` | Parameter kernel — dibaca oleh `/init` untuk menentukan warna prompt |

Jika boot berhasil, akan muncul:

```
sentinelos login:
```

Login dengan:
- User: `root` → password: `SkyLord`
- User: `guardian` → password: `Guard123`
- User: `observer` → password: `EyeOpen`

Untuk keluar dari QEMU: tekan **Ctrl+A** lalu **X**.

Untuk mengulang boot:

```bash
pkill -f qemu
qemu-system-x86_64 -smp 2 -m 256 -display curses -vga std \
  -kernel bzImage -initrd myramdisk.gz -append "mode=1"
```

### Screenshot
<img width="1552" height="905" alt="image" src="https://github.com/user-attachments/assets/f18c9ad6-7586-4d9a-89fb-61e41f015044" />


### Kode Penuh

```bash
# Buat init script
cat > myramdisk/init << 'EOF'
#!/bin/sh
/bin/mount -t proc none /proc
/bin/mount -t sysfs none /sys
/bin/mount -t devtmpfs none /dev
/bin/hostname sentinelos

MODE=$(cat /proc/cmdline | tr ' ' '\n' | grep '^mode=' | cut -d= -f2)
[ -z "$MODE" ] && MODE=1

case "$MODE" in
    1) USER_CLR='\[\e[1;32m\]'; HOST_CLR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
    2) USER_CLR='\[\e[1;33m\]'; HOST_CLR='\[\e[1;36m\]'; LABEL="WATCH"   ;;
    3) USER_CLR='\[\e[1;31m\]'; HOST_CLR='\[\e[1;35m\]'; LABEL="SECURE"  ;;
    *) USER_CLR='\[\e[1;32m\]'; HOST_CLR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
esac

cat > /etc/profile << PROFILE
export PATH=/bin:/sbin
export TERM=linux
export SENTINELOSMODE="${LABEL}"
PS1='${USER_CLR}[\u${HOST_CLR}@\h\[\e[0m\] \W${USER_CLR}]\[\e[0m\]\$ '
export PS1
[ -f /log/NOTICE ] && cat /log/NOTICE
PROFILE

while true; do
    /bin/getty -L tty1 115200 vt100
    sleep 1
done
EOF

# Beri izin eksekusi
chmod +x myramdisk/init

# Pack jadi initramfs
cd myramdisk
find . | cpio -oHnewc | gzip > ../myramdisk.gz
cd ..

# Test QEMU
qemu-system-x86_64 \
  -smp 2 -m 256 \
  -display curses \
  -vga std \
  -kernel bzImage \
  -initrd myramdisk.gz \
  -append "mode=1"
```

---

## E. Membuat ISO Bootable dengan Multi-Mode GRUB

### Deskripsi

Pada poin E, SentinelOS dikemas menjadi **ISO bootable** menggunakan GRUB sebagai bootloader. GRUB dikonfigurasi dengan **3 menu** yang masing-masing melewatkan nilai `mode` berbeda ke kernel. Nilai ini dibaca oleh `/init` untuk mengubah warna prompt secara otomatis sesuai pilihan user di menu GRUB.

### Langkah-langkah & Potongan Kode

#### 1. Buat Struktur Direktori ISO

```bash
mkdir -p mylinuxiso/boot/grub
```

Struktur yang terbentuk:

```
mylinuxiso/
└── boot/
    ├── bzImage
    ├── myramdisk.gz
    └── grub/
        └── grub.cfg
```

#### 2. Salin Kernel dan Initramfs

```bash
cp bzImage      mylinuxiso/boot/
cp myramdisk.gz mylinuxiso/boot/
```

#### 3. Buat File Konfigurasi GRUB

```bash
cat > mylinuxiso/boot/grub/grub.cfg << 'EOF'
set timeout=5
set default=0

set color_normal=white/black
set color_highlight=black/white

menuentry "SentinelOS - Mode 1 (Default - Hijau & Biru)" {
    linux  /boot/bzImage mode=1 quiet
    initrd /boot/myramdisk.gz
}

menuentry "SentinelOS - Mode 2 (Watch - Kuning & Cyan)" {
    linux  /boot/bzImage mode=2 quiet
    initrd /boot/myramdisk.gz
}

menuentry "SentinelOS - Mode 3 (Secure - Merah & Magenta)" {
    linux  /boot/bzImage mode=3 quiet
    initrd /boot/myramdisk.gz
}
EOF
```

Penjelasan konfigurasi:

- `set timeout=5` : GRUB menunggu 5 detik sebelum otomatis memilih menu default
- `set default=0` : menu pertama (Mode 1) dipilih secara default jika user tidak memilih
- `linux /boot/bzImage mode=1 quiet` : memuat kernel dengan parameter `mode=1` yang akan dibaca `/init`
- `initrd /boot/myramdisk.gz` : memuat root filesystem ke memori
- `quiet` : menyembunyikan pesan kernel saat booting

Alur warna berdasarkan pilihan menu GRUB:

```
GRUB Menu
├── Pilih Mode 1  →  kernel param: mode=1  →  /init baca MODE=1  →  prompt Hijau & Biru
├── Pilih Mode 2  →  kernel param: mode=2  →  /init baca MODE=2  →  prompt Kuning & Cyan
└── Pilih Mode 3  →  kernel param: mode=3  →  /init baca MODE=3  →  prompt Merah & Magenta
```

| Mode | Parameter | Warna User | Warna Host |
|------|-----------|------------|------------|
| 1 | `mode=1` | Hijau (`\e[1;32m`) | Biru (`\e[1;34m`) |
| 2 | `mode=2` | Kuning (`\e[1;33m`) | Cyan (`\e[1;36m`) |
| 3 | `mode=3` | Merah (`\e[1;31m`) | Magenta (`\e[1;35m`) |

#### 4. Buat ISO dengan grub-mkrescue

```bash
grub-mkrescue -o mylinux.iso mylinuxiso
```

- `grub-mkrescue` : tool resmi GRUB untuk membuat ISO bootable dengan bootloader lengkap (BIOS & UEFI)
- `-o mylinux.iso` : nama file ISO output
- `mylinuxiso` : folder sumber yang berisi semua file boot

#### 5. Uji ISO dengan QEMU

```bash
qemu-system-x86_64 \
  -smp 2 \
  -m 256 \
  -display curses \
  -vga std \
  -cdrom mylinux.iso
```

- `-cdrom mylinux.iso` : menggantikan opsi `-kernel` dan `-initrd`, sistem di-boot dari ISO layaknya CD fisik
- GRUB menu akan muncul, pilih salah satu mode
- Setelah memilih, kernel dan initramfs dimuat, lalu muncul login prompt.

Untuk mengulang:

```bash
pkill -f qemu
qemu-system-x86_64 -smp 2 -m 256 -display curses -vga std -cdrom mylinux.iso
```
### Screenshot
<img width="1554" height="828" alt="image" src="https://github.com/user-attachments/assets/51d5a9c3-9f62-4724-be5b-a4228eb0bf09" />



### Kode Penuh

```bash
# Buat struktur ISO
mkdir -p mylinuxiso/boot/grub

# Copy file kernel dan initramfs
cp bzImage      mylinuxiso/boot/
cp myramdisk.gz mylinuxiso/boot/

# Buat grub.cfg
cat > mylinuxiso/boot/grub/grub.cfg << 'EOF'
set timeout=5
set default=0

set color_normal=white/black
set color_highlight=black/white

menuentry "SentinelOS - Mode 1 (Default - Hijau & Biru)" {
    linux  /boot/bzImage mode=1 quiet
    initrd /boot/myramdisk.gz
}

menuentry "SentinelOS - Mode 2 (Watch - Kuning & Cyan)" {
    linux  /boot/bzImage mode=2 quiet
    initrd /boot/myramdisk.gz
}

menuentry "SentinelOS - Mode 3 (Secure - Merah & Magenta)" {
    linux  /boot/bzImage mode=3 quiet
    initrd /boot/myramdisk.gz
}
EOF

# Buat ISO
grub-mkrescue -o mylinux.iso mylinuxiso

# Test dengan QEMU
qemu-system-x86_64 \
  -smp 2 -m 256 \
  -display curses \
  -vga std \
  -cdrom mylinux.iso
```

---

## F. Membuat System Info Script

### Deskripsi

Pada poin F, dibuat script `/bin/sysinfo` yang bisa dijalankan oleh **semua user** tanpa argumen dari direktori manapun. Script menampilkan 7 informasi sistem secara terformat dengan warna menggunakan kode ANSI.

### Langkah-langkah & Potongan Kode

#### 1. Buat file sysinfo di dalam folder bin:
```sh
nano bin/sysinfo
```

#### 2. Menulis script

```sh
#!/bin/sh

echo "=========================================="
echo "          SENTINELOS SYSTEM INFO          "
echo "=========================================="
echo "Hostname       : $(hostname)"
echo "Current Time   : $(date)"
echo "Uptime         : $(uptime | awk -F, '{print $1}')"
echo "Current User   : $(whoami)"
echo "Home Directory : $HOME"
echo "Running Procs  : $(ps | wc -l)"
echo "Free Memory    : $(free -h | grep Mem | awk '{print $4}')"
echo "=========================================="
```
#### 3. Kasih izin eksekusi:
```sh
chmod +x bin/sysinfo
```

#### 4. Membungkus ulang ramdisk-nya (Balik jadi file .gz):
```sh
find . | cpio -o -H newc | gzip > ../myramdisk.gz
```

#### 5. Finishing :
```sh
cp myramdisk.gz mylinuxiso/boot/
grub-mkrescue -o sentinelos.iso mylinuxiso
```

Setelah login ke SentinelOS (via QEMU atau ISO), jalankan:

```sh
sysinfo
```

### Screenshot
<img width="1550" height="913" alt="image" src="https://github.com/user-attachments/assets/870fd4f0-7b7b-4458-a72a-f7ca9e50d882" />


### Kode Penuh

```bash
nano bin/sysinfo

#!/bin/sh

echo "=========================================="
echo "          SENTINELOS SYSTEM INFO          "
echo "=========================================="
echo "Hostname       : $(hostname)"
echo "Current Time   : $(date)"
echo "Uptime         : $(uptime | awk -F, '{print $1}')"
echo "Current User   : $(whoami)"
echo "Home Directory : $HOME"
echo "Running Procs  : $(ps | wc -l)"
echo "Free Memory    : $(free -h | grep Mem | awk '{print $4}')"
echo "=========================================="

chmod +x bin/sysinfo

find . | cpio -o -H newc | gzip > ../myramdisk.gz

cp myramdisk.gz mylinuxiso/boot/
grub-mkrescue -o sentinelos.iso mylinuxiso

sysinfo
```

---

## Struktur Akhir Folder `osboot`

Setelah semua poin C–F selesai, struktur folder `osboot` akan terlihat seperti ini:

```
osboot/
├── bzImage                    ← hasil Task A
├── myramdisk.gz               ← hasil Task D (pack initramfs)
├── mylinux.iso                ← hasil Task E
├── linux-6.1.1/               ← source kernel
└── myramdisk/                 ← rootfs (Task B–F)
    ├── init                   ← Task D (PID 1)
    ├── bin/
    │   ├── busybox
    │   ├── sh → busybox
    │   ├── getty → busybox
    │   ├── login → busybox
    │   ├── sysinfo            ← Task F
    │   └── ... (applet lainnya)
    ├── dev/
    │   ├── null
    │   ├── tty*
    │   ├── zero
    │   └── console
    ├── etc/
    │   ├── passwd             ← Task C
    │   └── group              ← Task C
    ├── proc/
    ├── sys/
    ├── tmp/
    ├── vault/                 ← chmod 700, owner root       (Task C)
    ├── watch/                 ← chmod 700, owner guardian   (Task C)
    ├── log/
    │   └── NOTICE             ← Task B
    ├── root/                  ← chmod 700, owner root       (Task C)
    └── home/
        ├── guardian/          ← chmod 700, owner guardian   (Task C)
        └── observer/          ← chmod 700, owner observer   (Task C)
```
