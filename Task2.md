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

#### 6. Setup Script /init
Script ini yang pertama kali jalan. Tugasnya sekarang: mount sistem file, baca mode warna, dan panggil getty supaya kita bisa login.

```bash
cat > myramdisk/init << 'EOF'
#!/bin/sh
# 1. Mount sistem file virtual
/bin/mount -t proc none /proc
/bin/mount -t sysfs none /sys
/bin/mount -t devtmpfs none /dev

# 2. Set Hostname
/bin/hostname sentinelos

# 3. Baca mode warna dari kernel command line (GRUB)
MODE=$(cat /proc/cmdline | tr ' ' '\n' | grep '^mode=' | cut -d= -f2)
[ -z "$MODE" ] && MODE=1

case "$MODE" in
    1) COLOR='\[\e[1;32m\]'; HOST_COLOR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
    2) COLOR='\[\e[1;33m\]'; HOST_COLOR='\[\e[1;36m\]'; LABEL="WATCH"   ;;
    3) COLOR='\[\e[1;31m\]'; HOST_COLOR='\[\e[1;35m\]'; LABEL="SECURE"  ;;
esac

# 4. Buat /etc/profile secara dinamis (Supaya NOTICE muncul tiap LOGIN)
cat > /etc/profile << PROFILE
export PATH=/bin:/sbin
export TERM=linux
export SENTINELOSMODE="${LABEL}"
PS1='${COLOR}[\u${HOST_COLOR}@\h\[\e[0m\] \W${COLOR}]\[\e[0m\]\$ '
export PS1
# Tampilkan file NOTICE setiap kali user login
[ -f /log/NOTICE ] && cat /log/NOTICE
PROFILE

# 5. Jalankan prompt login selamanya
while true; do
    /bin/getty -L tty1 115200 vt100
    sleep 1
done
EOF

chmod +x myramdisk/init
```

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

cat > myramdisk/init << 'EOF'
#!/bin/sh
# 1. Mount sistem file virtual
/bin/mount -t proc none /proc
/bin/mount -t sysfs none /sys
/bin/mount -t devtmpfs none /dev

# 2. Set Hostname
/bin/hostname sentinelos

# 3. Baca mode warna dari kernel command line (GRUB)
MODE=$(cat /proc/cmdline | tr ' ' '\n' | grep '^mode=' | cut -d= -f2)
[ -z "$MODE" ] && MODE=1

case "$MODE" in
    1) COLOR='\[\e[1;32m\]'; HOST_COLOR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
    2) COLOR='\[\e[1;33m\]'; HOST_COLOR='\[\e[1;36m\]'; LABEL="WATCH"   ;;
    3) COLOR='\[\e[1;31m\]'; HOST_COLOR='\[\e[1;35m\]'; LABEL="SECURE"  ;;
esac

# 4. Buat /etc/profile secara dinamis (Supaya NOTICE muncul tiap LOGIN)
cat > /etc/profile << PROFILE
export PATH=/bin:/sbin
export TERM=linux
export SENTINELOSMODE="${LABEL}"
PS1='${COLOR}[\u${HOST_COLOR}@\h\[\e[0m\] \W${COLOR}]\[\e[0m\]\$ '
export PS1
# Tampilkan file NOTICE setiap kali user login
[ -f /log/NOTICE ] && cat /log/NOTICE
PROFILE

# 5. Jalankan prompt login selamanya
while true; do
    /bin/getty -L tty1 115200 vt100
    sleep 1
done
EOF

chmod +x myramdisk/init
```
### Screenshoot
<img width="1207" height="963" alt="image" src="https://github.com/user-attachments/assets/a29d0a61-200c-4f09-9e14-070646684bfd" />


---

## C. Mengimplementasikan Sistem Multi-User dan Keamanan

### Deskripsi

Pada poin C, diimplementasikan sistem autentikasi tiga user dengan password terenkripsi **MD5**. Setiap user memiliki akses terbatas ke direktori tertentu sesuai perannya.

### Langkah-langkah & Potongan Kode

#### 1. Generate Password Hash MD5

```bash
openssl passwd -1 SkyLord
openssl passwd -1 Guard123
openssl passwd -1 EyeOpen
```

- `openssl passwd -1` : menghasilkan hash MD5 dengan format `$1$salt$hash`
- Contoh output:

```
root     : $1$tLdbnIuD$wWE8YP5l0yxKTyWNEXadu0
guardian : $1$TKyBqwW0$z0IRW8tW4dc9tkbBqNU7U/
observer : $1$8xKAwKIz$Q2vjKZFtzenad1g77avyc0
```

Salin masing-masing hasil hash untuk dimasukkan ke file `passwd` di langkah berikutnya.

#### 2. Buat File `/etc/passwd`

```bash
cat > myramdisk/etc/passwd << 'EOF'
root:$1$tLdbnIuD$wWE8YP5l0yxKTyWNEXadu0:0:0:root:/root:/bin/sh
guardian:$1$TKyBqwW0$z0IRW8tW4dc9tkbBqNU7U/:1000:1000:Guardian User:/home/guardian:/bin/sh
observer:$1$8xKAwKIz$Q2vjKZFtzenad1g77avyc0:1001:1001:Observer User:/home/observer:/bin/sh
EOF
```

> **Penting:** Ganti hash di atas dengan hasil generate `openssl passwd` dari langkah 1, karena salt-nya berbeda setiap kali dijalankan.

Format field: `username:hash_password:UID:GID:komentar:home_dir:shell`

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

#### 4. Mengatur Permission Direktori sesuai Role

```bash
# /vault -> root only (rwx------)
chmod 700 myramdisk/vault
chown 0:0 myramdisk/vault

# /watch -> guardian only
chmod 700 myramdisk/watch
chown 1000:1000 myramdisk/watch

# /home/guardian -> guardian only
chmod 700 myramdisk/home/guardian
chown 1000:1000 myramdisk/home/guardian

# /home/observer -> observer only
chmod 700 myramdisk/home/observer
chown 1001:1001 myramdisk/home/observer

# /root -> root only
chmod 700 myramdisk/root
chown 0:0 myramdisk/root

# /log -> semua bisa baca
chmod 755 myramdisk/log
```

| Direktori | Owner | Permission | Akses |
|-----------|-------|-----------|-------|
| `/vault` | root:root | `700` | root saja |
| `/watch` | guardian:guardian | `700` | guardian saja |
| `/home/guardian` | guardian:guardian | `700` | guardian saja |
| `/home/observer` | observer:observer | `700` | observer saja |
| `/root` | root:root | `700` | root saja |
| `/log` | root:root | `755` | semua bisa baca |

### Kode Penuh

```bash
# Generate hash (jalankan satu per satu, salin hasilnya)
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
tty:x:5:root,guardian,observer
wheel:x:10:root,guardian
users:x:100:guardian,observer
guardian:x:1000:guardian
observer:x:1001:observer
EOF

# Set permission
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

Pada poin D, dibuat **init script** yang berjalan sebagai PID 1 saat kernel selesai loading. Init script me-mount filesystem virtual, membaca parameter `mode` dari kernel cmdline untuk menentukan warna prompt, lalu menjalankan `getty` untuk proses login. Kemudian initramfs dikemas dan diuji dengan QEMU.

### Langkah-langkah & Potongan Kode

#### 1. Buat File `init`

```bash
cat > myramdisk/init << 'EOF'
#!/bin/sh
/bin/mount -t proc none /proc
/bin/mount -t sysfs none /sys
/bin/mount -t devtmpfs none /dev

/bin/hostname sentinelos

# Baca mode dari kernel cmdline
MODE=$(cat /proc/cmdline | tr ' ' '\n' | grep '^mode=' | cut -d= -f2)
[ -z "$MODE" ] && MODE=1

case "$MODE" in
    1) COLOR='\[\e[1;32m\]'; HOST_COLOR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
    2) COLOR='\[\e[1;33m\]'; HOST_COLOR='\[\e[1;36m\]'; LABEL="WATCH"   ;;
    3) COLOR='\[\e[1;31m\]'; HOST_COLOR='\[\e[1;35m\]'; LABEL="SECURE"  ;;
    *) COLOR='\[\e[1;32m\]'; HOST_COLOR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
esac

cat > /etc/profile << PROFILE
export PATH=/bin:/sbin
export TERM=linux
export SENTINELOSMODE="${LABEL}"
PS1='${COLOR}[\u${HOST_COLOR}@\h\[\e[0m\] \W${COLOR}]\[\e[0m\]\$ '
export PS1
[ -f /log/NOTICE ] && cat /log/NOTICE
PROFILE

while true; do
    /bin/getty -L tty1 115200 vt100
    sleep 1
done
EOF
```

Penjelasan isi init script:

- `mount -t proc none /proc` : mount procfs agar `/proc/cmdline` dan perintah seperti `ps` bisa berjalan
- `mount -t sysfs none /sys` : mount sysfs untuk interaksi dengan kernel
- `mount -t devtmpfs none /dev` : mount devtmpfs agar device nodes tersedia secara dinamis
- `hostname sentinelos` : set hostname sistem
- Blok `case "$MODE"` : menentukan kode warna ANSI berdasarkan parameter `mode=` dari kernel cmdline
- `/etc/profile` : ditulis secara dinamis sehingga warna prompt sesuai mode yang dipilih saat boot
- `/bin/getty -L tty1 115200 vt100` : menjalankan login prompt di terminal tty1

Kode warna ANSI yang digunakan:

| Mode | Label | Warna prompt | Kode ANSI |
|------|-------|-------------|-----------|
| 1 | DEFAULT | Hijau | `\e[1;32m` |
| 2 | WATCH | Kuning | `\e[1;33m` |
| 3 | SECURE | Merah | `\e[1;31m` |

#### 2. Berikan Izin Eksekusi pada File `init`

```bash
chmod +x myramdisk/init
```

Kernel Linux mengharuskan file `/init` bisa dieksekusi, jika tidak kernel akan panic.

#### 3. Salin Script `sysinfo` ke `/bin`

```bash
cat > myramdisk/bin/sysinfo << 'EOF'
#!/bin/sh
CYAN='\033[1;36m'
GREEN='\033[1;32m'
WHITE='\033[1;37m'
RESET='\033[0m'
SEP="${CYAN}=================================================${RESET}"

SYS_HOSTNAME=$(hostname 2>/dev/null || echo "unknown")
SYS_DATE=$(date "+%A, %d %B %Y %H:%M:%S")
SYS_UPTIME=$(uptime | sed 's/.*up /up /' | cut -d',' -f1-2)
SYS_USER=$(whoami)
SYS_HOME="$HOME"
SYS_PROCS=$(ps | wc -l)
SYS_PROCS=$((SYS_PROCS - 1))
SYS_MEM=$(free | awk '/^Mem:/{printf "Total: %dK | Used: %dK | Free: %dK",$2,$3,$4}')

echo ""
printf "${SEP}\n"
printf "  ${WHITE}SentinelOS - System Information${RESET}\n"
printf "${SEP}\n"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Hostname"       "$SYS_HOSTNAME"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Date & Time"    "$SYS_DATE"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Uptime"         "$SYS_UPTIME"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Current User"   "$SYS_USER"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Home Directory" "$SYS_HOME"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Running Procs"  "$SYS_PROCS processes"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Memory"         "$SYS_MEM"
printf "${SEP}\n"
echo ""
EOF

chmod 755 myramdisk/bin/sysinfo
```

#### 4. Buat Initramfs

```bash
cd myramdisk
find . | cpio -oHnewc | gzip > ../myramdisk.gz
cd ..
```

- `find .` : mencantumkan semua file di dalam `myramdisk/`
- `cpio -oHnewc` : membuat arsip CPIO format `newc` (satu-satunya format yang dikenali kernel Linux untuk initramfs)
- `gzip` : kompresi output
- Hasilnya `myramdisk.gz` di direktori `osboot/`.

#### 5. Uji Boot dengan QEMU (Mode 1)

```bash
cd osboot
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
| `-smp 2` | 2 vCPU virtual |
| `-m 256` | RAM 256 MB |
| `-display curses` | Output teks di terminal (tanpa GUI) |
| `-vga std` | VGA standar 80x25 |
| `-kernel bzImage` | Kernel yang digunakan |
| `-initrd myramdisk.gz` | Root filesystem yang dimuat |
| `-append "mode=1"` | Parameter yang di-pass ke kernel (untuk warna prompt) |

Jika berhasil, akan muncul prompt login:
```
sentinelos login:
```
Login dengan `root` dan password `SkyLord`.

Untuk keluar dari QEMU: tekan **Ctrl+A** lalu **X**.

Untuk mengulangi boot:
```bash
pkill -f qemu
# lalu jalankan kembali perintah qemu di atas
```

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
    1) COLOR='\[\e[1;32m\]'; HOST_COLOR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
    2) COLOR='\[\e[1;33m\]'; HOST_COLOR='\[\e[1;36m\]'; LABEL="WATCH"   ;;
    3) COLOR='\[\e[1;31m\]'; HOST_COLOR='\[\e[1;35m\]'; LABEL="SECURE"  ;;
    *) COLOR='\[\e[1;32m\]'; HOST_COLOR='\[\e[1;34m\]'; LABEL="DEFAULT" ;;
esac

cat > /etc/profile << PROFILE
export PATH=/bin:/sbin
export TERM=linux
PS1='${COLOR}[\u${HOST_COLOR}@\h\[\e[0m\] \W${COLOR}]\[\e[0m\]\$ '
export PS1
[ -f /log/NOTICE ] && cat /log/NOTICE
PROFILE

while true; do
    /bin/getty -L tty1 115200 vt100
    sleep 1
done
EOF

chmod +x myramdisk/init

# Pack menjadi initramfs
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

Pada poin E, SentinelOS dikemas menjadi **ISO bootable** menggunakan GRUB sebagai bootloader. GRUB dikonfigurasi dengan **3 menu** yang masing-masing melewatkan parameter `mode` berbeda ke kernel.

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

#### 2. Salin Kernel dan Initramfs ke Direktori ISO

```bash
cp bzImage    mylinuxiso/boot/
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

- `set timeout=5` : GRUB menunggu 5 detik sebelum boot otomatis ke pilihan default
- `set default=0` : Mode 1 dipilih secara default (indeks mulai dari 0)
- Setiap `menuentry` mem-pass parameter `mode=X` yang berbeda ke kernel
- Parameter ini dibaca oleh `/init` melalui `/proc/cmdline` untuk menentukan warna prompt.

Alur warna berdasarkan pilihan menu:

| Menu | Parameter Kernel | Warna Prompt |
|------|-----------------|-------------|
| Mode 1 | `mode=1` | Hijau & Biru |
| Mode 2 | `mode=2` | Kuning & Cyan |
| Mode 3 | `mode=3` | Merah & Magenta |

#### 4. Buat ISO Bootable dengan grub-mkrescue

```bash
grub-mkrescue -o mylinux.iso mylinuxiso
```

- `grub-mkrescue` : tool resmi GRUB untuk membuat ISO bootable lengkap dengan bootloader untuk BIOS dan UEFI
- `-o mylinux.iso` : nama file ISO output
- `mylinuxiso` : direktori sumber yang berisi boot files.

### Kode Penuh

```bash
mkdir -p mylinuxiso/boot/grub

cp bzImage      mylinuxiso/boot/
cp myramdisk.gz mylinuxiso/boot/

cat > mylinuxiso/boot/grub/grub.cfg << 'EOF'
set timeout=5
set default=0

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

grub-mkrescue -o mylinux.iso mylinuxiso
```

---

## F. Membuat System Info Script

### Deskripsi

Pada poin F, dibuat script `/bin/sysinfo` yang dapat dijalankan oleh **semua user** dari direktori manapun. Script ini menampilkan informasi sistem secara terformat dan berwarna.

### Langkah-langkah & Potongan Kode

#### 1. Definisi Warna Output

```sh
CYAN='\033[1;36m'
GREEN='\033[1;32m'
WHITE='\033[1;37m'
RESET='\033[0m'
```

Kode warna ANSI menggunakan format `\033[STYLE;COLORm`. `printf` digunakan (bukan `echo`) karena BusyBox `echo` tidak selalu mendukung escape sequence di semua shell.

#### 2. Mengumpulkan Informasi Sistem

```sh
SYS_HOSTNAME=$(hostname)
SYS_DATE=$(date "+%A, %d %B %Y %H:%M:%S")
SYS_UPTIME=$(uptime | sed 's/.*up /up /' | cut -d',' -f1-2)
SYS_USER=$(whoami)
SYS_HOME="$HOME"
SYS_PROCS=$(ps | wc -l)
SYS_PROCS=$((SYS_PROCS - 1))
SYS_MEM=$(free | awk '/^Mem:/{printf "Total: %dK | Used: %dK | Free: %dK",$2,$3,$4}')
```

- `ps | wc -l` : hitung jumlah baris output `ps`, dikurangi 1 untuk header
- `awk '/^Mem:/'` : filter hanya baris memory dari output `free`
- Semua perintah sudah tersedia di BusyBox.

#### 3. Menampilkan Output Terformat

```sh
printf "  ${GREEN}%-18s${RESET}: %s\n" "Hostname" "$SYS_HOSTNAME"
```

- `%-18s` : rata kiri dengan lebar 18 karakter untuk alignment yang rapi
- Setiap baris info dicetak dengan label berwarna hijau dan nilai berwarna default.

#### 4. Pemasangan ke Rootfs dan Permission

Script ini sudah ditulis langsung ke `myramdisk/bin/sysinfo` pada langkah D (bagian pembuatan initramfs). Permission diset dengan:

```bash
chmod 755 myramdisk/bin/sysinfo
```

- `chmod 755` : owner bisa baca/tulis/eksekusi, group dan other bisa baca/eksekusi
- Karena `/bin` ada di `$PATH`, script langsung bisa dipanggil dengan `sysinfo` dari direktori manapun.

Setelah login ke SentinelOS via QEMU atau ISO, jalankan:

```sh
sysinfo
```

Contoh output:

```
=================================================
  SentinelOS - System Information
=================================================
  Hostname          : sentinelos
  Date & Time       : Wednesday, 16 April 2025 14:32:07
  Uptime            : up 3 minutes
  Current User      : guardian
  Home Directory    : /home/guardian
  Running Procs     : 12 processes
  Memory            : Total: 262144K | Used: 18432K | Free: 243712K
=================================================
```

### Kode Penuh

```bash
cat > myramdisk/bin/sysinfo << 'EOF'
#!/bin/sh
CYAN='\033[1;36m'
GREEN='\033[1;32m'
WHITE='\033[1;37m'
RESET='\033[0m'
SEP="${CYAN}=================================================${RESET}"

SYS_HOSTNAME=$(hostname 2>/dev/null || echo "unknown")
SYS_DATE=$(date "+%A, %d %B %Y %H:%M:%S" 2>/dev/null || echo "unknown")
SYS_UPTIME=$(uptime 2>/dev/null | sed 's/.*up /up /' | cut -d',' -f1-2 || echo "unknown")
SYS_USER=$(whoami 2>/dev/null || echo "unknown")
SYS_HOME="$HOME"
SYS_PROCS=$(ps 2>/dev/null | wc -l || echo "0")
SYS_PROCS=$((SYS_PROCS - 1))
SYS_MEM=$(free 2>/dev/null | awk '/^Mem:/{printf "Total: %dK | Used: %dK | Free: %dK",$2,$3,$4}' || echo "unknown")

echo ""
printf "${SEP}\n"
printf "  ${WHITE}SentinelOS - System Information${RESET}\n"
printf "${SEP}\n"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Hostname"       "$SYS_HOSTNAME"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Date & Time"    "$SYS_DATE"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Uptime"         "$SYS_UPTIME"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Current User"   "$SYS_USER"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Home Directory" "$SYS_HOME"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Running Procs"  "$SYS_PROCS processes"
printf "  ${GREEN}%-18s${RESET}: %s\n" "Memory"         "$SYS_MEM"
printf "${SEP}\n"
echo ""
EOF

chmod 755 myramdisk/bin/sysinfo
```

---

## Emulasi Menjalankan File ISO

Setelah ISO berhasil dibuat, uji dengan QEMU:

```bash
qemu-system-x86_64 \
  -smp 2 \
  -m 256 \
  -display curses \
  -vga std \
  -cdrom mylinux.iso
```

- `-cdrom mylinux.iso` : menggantikan `-kernel` dan `-initrd`, sistem di-boot dari ISO
- GRUB menu akan muncul dengan 3 pilihan mode
- Setelah memilih, kernel dan initramfs dimuat, lalu muncul prompt login.

Untuk menjalankan ulang:
```bash
pkill -f qemu
qemu-system-x86_64 -smp 2 -m 256 -display curses -vga std -cdrom mylinux.iso
```

---

## Rangkuman Perintah Lengkap (Cheatsheet)

```bash
# ===== PREREQUISITE (Garuda Linux) =====
sudo pacman -S base-devel bc bison flex ncurses openssl elfutils \
               qemu-system-x86 grub xorriso mtools busybox tmux

# ===== A: KERNEL =====
mkdir -p osboot && cd osboot
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.1.tar.xz
tar -xvf linux-6.1.1.tar.xz && cd linux-6.1.1
make tinyconfig && make menuconfig   # aktifkan opsi sesuai tabel
make -j$(nproc)
cp arch/x86/boot/bzImage .. && cd ..

# ===== B: ROOTFS =====
sudo bash
mkdir -p myramdisk/{bin,dev,etc,proc,sys,tmp,vault,watch,log,root}
mkdir -p myramdisk/home/{guardian,observer}
cp -a /dev/null /dev/tty* /dev/zero /dev/console myramdisk/dev/
cp /usr/bin/busybox myramdisk/bin
cd myramdisk/bin && ./busybox --install . && cd ../..
# buat /log/NOTICE (isi terserah)

# ===== C: MULTI-USER =====
openssl passwd -1 SkyLord    # salin hasilnya
openssl passwd -1 Guard123   # salin hasilnya
openssl passwd -1 EyeOpen    # salin hasilnya
# buat myramdisk/etc/passwd dan /etc/group dengan hash di atas
chmod 700 myramdisk/{vault,watch,home/guardian,home/observer,root}
chown 0:0 myramdisk/vault myramdisk/root
chown 1000:1000 myramdisk/watch myramdisk/home/guardian
chown 1001:1001 myramdisk/home/observer

# ===== D: INIT + QEMU =====
# buat myramdisk/init (lihat kode penuh di atas)
chmod +x myramdisk/init
# buat myramdisk/bin/sysinfo (lihat kode penuh di atas)
chmod 755 myramdisk/bin/sysinfo
cd myramdisk && find . | cpio -oHnewc | gzip > ../myramdisk.gz && cd ..
qemu-system-x86_64 -smp 2 -m 256 -display curses -vga std \
  -kernel bzImage -initrd myramdisk.gz -append "mode=1"

# ===== E: ISO =====
mkdir -p mylinuxiso/boot/grub
cp bzImage myramdisk.gz mylinuxiso/boot/
# buat mylinuxiso/boot/grub/grub.cfg (lihat kode penuh di atas)
grub-mkrescue -o mylinux.iso mylinuxiso
qemu-system-x86_64 -smp 2 -m 256 -display curses -vga std -cdrom mylinux.iso
```
