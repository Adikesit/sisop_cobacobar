# Task 2 — SentinelOS

## Deskripsi Sistem

**SentinelOS** adalah sistem operasi sederhana berbasis Linux kernel 6.1.1 yang dibangun dari awal (*from scratch*). Sistem ini mendukung multi-user dengan sistem keamanan berbasis password terenkripsi MD5, prompt terminal berwarna berdasarkan mode boot, dan dapat dikemas sebagai ISO bootable dengan menu GRUB multi-mode.

### Struktur Direktori Proyek

```
osboot/
├── build.sh                  # Master build script
├── bzImage                   # Output kernel (setelah build)
├── initramfs.cpio.gz         # Output initramfs (setelah build)
├── sentinelos.iso            # Output ISO bootable (setelah build)
├── grub.cfg                  # Konfigurasi GRUB multi-mode
├── kernel.config             # Konfigurasi kernel minimal
├── rootfs/                   # Root filesystem
│   ├── bin/                  # BusyBox dan applet-appletnya
│   ├── dev/                  # Device nodes
│   ├── etc/                  # Konfigurasi sistem (passwd, shadow, dll)
│   ├── proc/, sys/, tmp/     # Virtual filesystem mountpoints
│   ├── vault/                # Direktori eksklusif root
│   ├── watch/                # Direktori eksklusif guardian
│   ├── log/NOTICE            # File notifikasi sistem
│   ├── home/guardian/        # Home guardian
│   ├── home/observer/        # Home observer
│   └── root/                 # Home root
├── rootfs_template/
│   ├── init                  # Init script (PID 1)
│   └── sysinfo               # System info script
└── scripts/
    ├── build_rootfs.sh       # Task B
    ├── setup_users.sh        # Task C
    ├── build_initramfs.sh    # Task D
    └── build_iso.sh          # Task E
```

### Cara Build & Menjalankan

```bash
# Install dependensi (Ubuntu/Debian)
sudo apt-get install build-essential libncurses-dev bison flex \
  libssl-dev libelf-dev bc xorriso grub-pc-bin grub-common \
  grub-efi-amd64-bin mtools qemu-system-x86

# Jalankan master build script dari folder osboot/
chmod +x build.sh
./build.sh

# Test boot QEMU (tanpa ISO)
qemu-system-x86_64 -kernel bzImage \
  -initrd initramfs.cpio.gz \
  -append "console=ttyS0 console=tty1 mode=1" \
  -nographic -m 256M

# Test boot dari ISO
qemu-system-x86_64 -cdrom sentinelos.iso -m 256M
```

---

## A. Mempersiapkan Kernel Linux

### Deskripsi

Pada poin A, dilakukan pengunduhan dan kompilasi **Linux kernel versi 6.1.1** dengan konfigurasi minimal untuk arsitektur x86_64. Kernel dikonfigurasi dengan fitur-fitur esensial yang dibutuhkan SentinelOS. Output akhirnya adalah file `bzImage`.

### Langkah-langkah & Potongan Kode

#### 1. Mengunduh dan Mengekstrak Source Kernel

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.1.tar.xz
tar -xf linux-6.1.1.tar.xz
cd linux-6.1.1
```

- Kernel diunduh langsung dari `kernel.org` sebagai sumber resmi
- Diekstrak menggunakan `tar -xf` karena format `.tar.xz`.

#### 2. Konfigurasi Kernel Minimal

```bash
make defconfig
```

`defconfig` menghasilkan konfigurasi default yang valid untuk arsitektur x86_64 sebagai titik awal, kemudian fitur-fitur esensial diaktifkan secara eksplisit:

```bash
scripts/config --enable CONFIG_64BIT
scripts/config --enable CONFIG_MULTIUSER
scripts/config --enable CONFIG_BINFMT_SCRIPT
scripts/config --enable CONFIG_SERIAL_8250
scripts/config --enable CONFIG_SERIAL_8250_CONSOLE
scripts/config --enable CONFIG_TTY
scripts/config --enable CONFIG_VT
scripts/config --enable CONFIG_VT_CONSOLE
scripts/config --enable CONFIG_DEVTMPFS
scripts/config --enable CONFIG_DEVTMPFS_MOUNT
scripts/config --enable CONFIG_TMPFS
scripts/config --enable CONFIG_PROC_FS
scripts/config --enable CONFIG_SYSFS
scripts/config --enable CONFIG_BLK_DEV_INITRD
scripts/config --enable CONFIG_RD_GZIP
```

| Config | Fungsi |
| ------ | ------ |
| `CONFIG_64BIT` | Mendukung arsitektur 64-bit |
| `CONFIG_MULTIUSER` | Dukungan multiple user dengan UID/GID |
| `CONFIG_BINFMT_SCRIPT` | Menjalankan script shell (`#!/bin/sh`) |
| `CONFIG_SERIAL_8250_CONSOLE` | Serial console untuk QEMU `-nographic` |
| `CONFIG_DEVTMPFS` | Otomatis membuat device nodes di `/dev` |
| `CONFIG_BLK_DEV_INITRD` | Dukungan initramfs/initrd |

#### 3. Finalisasi Konfigurasi dan Kompilasi

```bash
make olddefconfig
make -j$(nproc) bzImage
```

- `make olddefconfig` : menerapkan semua perubahan config dan mengisi nilai default untuk opsi baru
- `-j$(nproc)` : menggunakan seluruh core CPU agar kompilasi lebih cepat
- Output `bzImage` ada di `arch/x86/boot/bzImage`.

```bash
cp arch/x86/boot/bzImage ../bzImage
```

### Kode Penuh

```bash
# Download
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.1.1.tar.xz
tar -xf linux-6.1.1.tar.xz
cd linux-6.1.1

# Konfigurasi
make defconfig
scripts/config --enable CONFIG_64BIT
scripts/config --enable CONFIG_MULTIUSER
scripts/config --enable CONFIG_BINFMT_SCRIPT
scripts/config --enable CONFIG_SERIAL_8250
scripts/config --enable CONFIG_SERIAL_8250_CONSOLE
scripts/config --enable CONFIG_TTY
scripts/config --enable CONFIG_VT
scripts/config --enable CONFIG_VT_CONSOLE
scripts/config --enable CONFIG_UNIX98_PTYS
scripts/config --enable CONFIG_DEVTMPFS
scripts/config --enable CONFIG_DEVTMPFS_MOUNT
scripts/config --enable CONFIG_TMPFS
scripts/config --enable CONFIG_PROC_FS
scripts/config --enable CONFIG_SYSFS
scripts/config --enable CONFIG_BLK_DEV_INITRD
scripts/config --enable CONFIG_RD_GZIP
scripts/config --enable CONFIG_PRINTK

# Kompilasi
make olddefconfig
make -j$(nproc) bzImage

# Copy output
cp arch/x86/boot/bzImage ../bzImage
```

---

## B. Membangun Sistem File Root Awal

### Deskripsi

Pada poin B, dibangun **root filesystem (rootfs)** yang menjadi lingkungan dasar SentinelOS. Rootfs berisi seluruh struktur direktori sistem, binary BusyBox yang berfungsi sebagai pengganti utilitas Linux standar, device nodes minimal, dan file notifikasi `/log/NOTICE`.

### Langkah-langkah & Potongan Kode

#### 1. Membuat Struktur Direktori

```bash
ROOTFS="./rootfs"

mkdir -p "$ROOTFS"/{bin,dev,etc,proc,sys,tmp,vault,watch,log}
mkdir -p "$ROOTFS"/home/{guardian,observer}
mkdir -p "$ROOTFS"/root
```

Struktur direktori mengikuti standar FHS (Filesystem Hierarchy Standard) yang disederhanakan:

| Direktori | Fungsi |
| --------- | ------ |
| `/bin` | Binary dan applet BusyBox |
| `/dev` | Device nodes (null, tty, console, dll) |
| `/etc` | Konfigurasi sistem |
| `/proc` | Virtual filesystem proses kernel |
| `/sys` | Virtual filesystem sysfs |
| `/tmp` | File sementara |
| `/vault` | Direktori rahasia khusus root |
| `/watch` | Direktori pemantauan khusus guardian |
| `/log` | Log dan notifikasi sistem |
| `/home/guardian` | Home directory guardian |
| `/home/observer` | Home directory observer |
| `/root` | Home directory root |

#### 2. Mengunduh dan Menginstal BusyBox

```bash
wget -O busybox \
    "https://busybox.net/downloads/binaries/1.35.0-x86_64-linux-musl/busybox"
chmod +x busybox
cp busybox "$ROOTFS/bin/busybox"
```

BusyBox digunakan sebagai binary tunggal yang menyediakan ratusan perintah Linux standar melalui mekanisme symlink (satu binary, banyak nama).

```bash
for applet in sh ash ls cat echo cp mv rm mkdir chmod chown \
    mount umount mknod ps free uptime hostname date login \
    getty init syslogd passwd su whoami id; do
    ln -sf /bin/busybox "$ROOTFS/bin/$applet"
done
```

- Setiap symlink mengarah ke `/bin/busybox`
- Saat dipanggil dengan nama berbeda, BusyBox menjalankan fungsi yang sesuai.

#### 3. Membuat Device Nodes

```bash
mknod -m 666 "$ROOTFS/dev/null"    c 1 3
mknod -m 666 "$ROOTFS/dev/zero"    c 1 5
mknod -m 666 "$ROOTFS/dev/random"  c 1 8
mknod -m 666 "$ROOTFS/dev/urandom" c 1 9
mknod -m 622 "$ROOTFS/dev/console" c 5 1
mknod -m 666 "$ROOTFS/dev/tty"     c 5 0
mknod -m 666 "$ROOTFS/dev/ttyS0"   c 4 64
```

Format `mknod`: `mknod [mode] [path] [type] [major] [minor]`
- `c` = character device
- Major/minor number sesuai standar Linux kernel.

#### 4. Membuat `/log/NOTICE`

```bash
cat > "$ROOTFS/log/NOTICE" << 'EOF'
=======================================================
  SENTINELOS - SISTEM OPERASI SENTINEL
=======================================================

PEMBERITAHUAN KEAMANAN:
  Akses ke sistem ini hanya diperbolehkan untuk
  pengguna yang berwenang. Seluruh aktivitas dicatat
  dan dipantau oleh administrator sistem.
...
EOF
```

File ini akan ditampilkan otomatis kepada setiap user saat login melalui `/etc/profile`.

### Kode Penuh

Lihat [`scripts/build_rootfs.sh`](scripts/build_rootfs.sh)

---

## C. Mengimplementasikan Sistem Multi-User dan Keamanan

### Deskripsi

Pada poin C, diimplementasikan sistem autentikasi tiga user dengan password terenkripsi **MD5**. Setiap user memiliki akses terbatas ke direktori tertentu sesuai perannya (role-based access control sederhana).

### Langkah-langkah & Potongan Kode

#### 1. Generate Password Hash MD5

```bash
hash_root=$(openssl passwd -1 "SkyLord")
hash_guardian=$(openssl passwd -1 "Guard123")
hash_observer=$(openssl passwd -1 "EyeOpen")
```

- `openssl passwd -1` : menghasilkan hash MD5 dengan format `$1$salt$hash`
- Salt digenerate secara acak sehingga hash berbeda setiap kali dijalankan, namun tetap valid untuk autentikasi.

Contoh output:
```
root     : $1$xyz12345$AbCdEfGhIjKlMnOpQrSt..
guardian : $1$abc67890$WxYzAbCdEfGhIjKlMnOp..
observer : $1$def11223$QrStUvWxYzAbCdEfGhIj..
```

#### 2. Membuat `/etc/passwd`

```bash
cat > "$ETC/passwd" << EOF
root:x:0:0:root:/root:/bin/sh
guardian:x:1000:1000:Guardian User:/home/guardian:/bin/sh
observer:x:1001:1001:Observer User:/home/observer:/bin/sh
EOF
```

Format field: `username:x:UID:GID:comment:home:shell`
- `x` pada field password berarti password tersimpan di `/etc/shadow`.

#### 3. Membuat `/etc/shadow`

```bash
cat > "$ETC/shadow" << EOF
root:${hash_root}:19000:0:99999:7:::
guardian:${hash_guardian}:19000:0:99999:7:::
observer:${hash_observer}:19000:0:99999:7:::
EOF
chmod 640 "$ETC/shadow"
```

- `chmod 640` : hanya root yang bisa membaca file shadow
- Format field: `username:hash:lastchange:min:max:warn:inactive:expire:`.

#### 4. Mengatur Permission Direktori sesuai Role

```bash
# /vault -> root only (rwx------)
chmod 700 "$ROOTFS/vault"
chown 0:0 "$ROOTFS/vault"

# /watch -> guardian only
chmod 700 "$ROOTFS/watch"
chown 1000:1000 "$ROOTFS/watch"

# /home/guardian -> guardian only
chmod 700 "$ROOTFS/home/guardian"
chown 1000:1000 "$ROOTFS/home/guardian"

# /home/observer -> observer only
chmod 700 "$ROOTFS/home/observer"
chown 1001:1001 "$ROOTFS/home/observer"

# /root -> root only
chmod 700 "$ROOTFS/root"
chown 0:0 "$ROOTFS/root"
```

| Direktori | Owner | Permission | Akses |
| --------- | ----- | ---------- | ----- |
| `/vault` | root:root | `700` | root saja |
| `/watch` | guardian:guardian | `700` | guardian saja |
| `/home/guardian` | guardian:guardian | `700` | guardian saja |
| `/home/observer` | observer:observer | `700` | observer saja |
| `/root` | root:root | `700` | root saja |
| `/log` | root:root | `755` | semua bisa baca |

### Kode Penuh

Lihat [`scripts/setup_users.sh`](scripts/setup_users.sh)

---

## D. Uji Coba Boot dan Customization Terminal

### Deskripsi

Pada poin D, dibuat **init script** yang berjalan sebagai PID 1 saat kernel selesai loading. Init script ini me-mount filesystem virtual, membaca parameter `mode` dari kernel cmdline, mengatur warna prompt terminal sesuai mode, lalu menyerahkan kontrol ke BusyBox `init` untuk menjalankan getty/login.

### Langkah-langkah & Potongan Kode

#### 1. Struktur Init Script

```sh
#!/bin/sh
# /init — berjalan sebagai PID 1

# Mount filesystem virtual
mount -t proc  none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
mount -t tmpfs  none /tmp
```

Init script harus me-mount filesystem virtual ini lebih dahulu sebelum apapun, karena `/proc` dibutuhkan untuk membaca cmdline kernel dan banyak perintah lainnya.

#### 2. Membaca Mode dari Kernel Cmdline

```sh
MODE=$(cat /proc/cmdline | tr ' ' '\n' | grep '^mode=' | cut -d= -f2)
[ -z "$MODE" ] && MODE=1
```

- `/proc/cmdline` berisi parameter yang di-pass kernel saat boot, contoh: `console=ttyS0 mode=2`
- `tr ' ' '\n'` : memisahkan tiap parameter ke baris baru
- `grep '^mode='` : mengambil hanya parameter mode
- `cut -d= -f2` : mengambil nilai setelah tanda `=`.

#### 3. Menentukan Warna Prompt Berdasarkan Mode

```sh
case "$MODE" in
    1)
        PS1_COLOR='\[\e[1;32m\]'   # hijau terang (default)
        USER_COLOR='\[\e[1;34m\]'  # biru terang
        MODE_LABEL="DEFAULT"
        ;;
    2)
        PS1_COLOR='\[\e[1;33m\]'   # kuning terang
        USER_COLOR='\[\e[1;36m\]'  # cyan terang
        MODE_LABEL="WATCH"
        ;;
    3)
        PS1_COLOR='\[\e[1;31m\]'   # merah terang
        USER_COLOR='\[\e[1;35m\]'  # magenta terang
        MODE_LABEL="SECURE"
        ;;
esac
```

Kode warna ANSI:

| Kode | Warna |
| ---- | ----- |
| `\e[1;32m` | Hijau terang |
| `\e[1;34m` | Biru terang |
| `\e[1;33m` | Kuning terang |
| `\e[1;36m` | Cyan terang |
| `\e[1;31m` | Merah terang |
| `\e[1;35m` | Magenta terang |

#### 4. Menulis `/etc/profile` secara Dinamis

```sh
cat > /etc/profile << PROFILE_EOF
export PATH=/bin:/sbin:/usr/bin:/usr/sbin
export TERM=linux
export SENTINELOSMODE="${MODE_LABEL}"

PS1='${PS1_COLOR}[\u${USER_COLOR}@\h\[\e[0m\] \W${PS1_COLOR}]\[\e[0m\]\$ '
export PS1

if [ -f /log/NOTICE ]; then
    cat /log/NOTICE
fi
PROFILE_EOF
```

- `/etc/profile` dibaca setiap kali user login melalui shell
- Prompt format: `[username@hostname direktori]$`

#### 5. Konfigurasi `/etc/inittab`

```sh
cat > /etc/inittab << 'EOF'
::sysinit:/etc/init.d/rcS
tty1::respawn:/sbin/getty -L tty1 0 linux
ttyS0::respawn:/sbin/getty -L ttyS0 115200 vt100
::restart:/sbin/init
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
EOF
```

- `::sysinit:` : jalankan saat boot pertama kali
- `tty1::respawn:` : jalankan getty di tty1; jika mati, hidupkan lagi
- `ttyS0::respawn:` : sama untuk serial console (digunakan QEMU `-nographic`).

#### 6. Build Initramfs dan Test QEMU

```bash
# Build initramfs
cd rootfs
find . | cpio -H newc -o | gzip -9 > ../initramfs.cpio.gz

# Test boot QEMU Mode 1
qemu-system-x86_64 \
  -kernel bzImage \
  -initrd initramfs.cpio.gz \
  -append "console=ttyS0 console=tty1 mode=1" \
  -nographic -m 256M
```

- `cpio -H newc -o` : membuat arsip CPIO format newc (format yang didukung kernel Linux)
- `gzip -9` : kompresi maksimal
- `-nographic` : QEMU tanpa GUI, I/O melalui terminal (serial console).

### Kode Penuh

Lihat [`rootfs_template/init`](rootfs_template/init) dan [`scripts/build_initramfs.sh`](scripts/build_initramfs.sh)

---

## E. Membuat ISO Bootable dengan Multi-Mode GRUB

### Deskripsi

Pada poin E, SentinelOS dikemas menjadi **ISO bootable** yang bisa di-boot dari CDROM/DVDROM menggunakan GRUB sebagai bootloader. GRUB dikonfigurasi dengan **3 menu** yang masing-masing melewatkan parameter `mode` berbeda ke kernel, sehingga warna prompt berubah sesuai pilihan.

### Langkah-langkah & Potongan Kode

#### 1. Struktur ISO

```
iso_root/
└── boot/
    ├── bzImage
    ├── initramfs.cpio.gz
    └── grub/
        └── grub.cfg
```

```bash
mkdir -p iso_root/boot/grub
cp bzImage           iso_root/boot/bzImage
cp initramfs.cpio.gz iso_root/boot/initramfs.cpio.gz
cp grub.cfg          iso_root/boot/grub/grub.cfg
```

#### 2. Konfigurasi GRUB Multi-Mode

```
set default=0
set timeout=10

set color_normal=white/black
set color_highlight=black/white

menuentry "SentinelOS - Mode 1 (Default - Hijau & Biru)" {
    linux  /boot/bzImage console=ttyS0 console=tty1 mode=1 quiet
    initrd /boot/initramfs.cpio.gz
}

menuentry "SentinelOS - Mode 2 (Watch - Kuning & Cyan)" {
    linux  /boot/bzImage console=ttyS0 console=tty1 mode=2 quiet
    initrd /boot/initramfs.cpio.gz
}

menuentry "SentinelOS - Mode 3 (Secure - Merah & Magenta)" {
    linux  /boot/bzImage console=ttyS0 console=tty1 mode=3 quiet
    initrd /boot/initramfs.cpio.gz
}
```

- `set default=0` : Mode 1 dipilih secara default
- `set timeout=10` : GRUB menunggu 10 detik sebelum boot otomatis
- Setiap menuentry mem-pass `mode=X` yang berbeda sebagai kernel parameter
- `console=ttyS0 console=tty1` : output tersedia di keduanya (GUI dan serial).

#### 3. Membuat ISO dengan grub-mkrescue

```bash
grub-mkrescue -o sentinelos.iso iso_root -- -V "SENTINELOS"
```

- `grub-mkrescue` : tool resmi GRUB untuk membuat ISO bootable lengkap dengan bootloader
- `-V "SENTINELOS"` : volume label ISO
- Secara otomatis menyertakan GRUB untuk BIOS (i386-pc) dan UEFI (x86_64-efi).

#### 4. Alur Prompt Berdasarkan Mode

```
Boot ISO → GRUB Menu
    ├── Pilih Mode 1 → kernel param: mode=1
    │   └── /init membaca mode=1 → PS1 hijau & biru
    ├── Pilih Mode 2 → kernel param: mode=2
    │   └── /init membaca mode=2 → PS1 kuning & cyan
    └── Pilih Mode 3 → kernel param: mode=3
        └── /init membaca mode=3 → PS1 merah & magenta
```

| Mode | Label | Warna User | Warna Host |
| ---- | ----- | ---------- | ---------- |
| 1 | DEFAULT | Hijau (`\e[1;32m`) | Biru (`\e[1;34m`) |
| 2 | WATCH | Kuning (`\e[1;33m`) | Cyan (`\e[1;36m`) |
| 3 | SECURE | Merah (`\e[1;31m`) | Magenta (`\e[1;35m`) |

#### 5. Test ISO dengan QEMU

```bash
qemu-system-x86_64 -cdrom sentinelos.iso -m 256M
```

### Kode Penuh

Lihat [`grub.cfg`](grub.cfg) dan [`scripts/build_iso.sh`](scripts/build_iso.sh)

---

## F. Membuat System Info Script

### Deskripsi

Pada poin F, dibuat script `/bin/sysinfo` yang dapat dijalankan oleh **semua user** dari direktori manapun. Script ini menampilkan informasi sistem secara terformat dan berwarna meliputi: hostname, tanggal/waktu, uptime, user aktif, jumlah proses yang berjalan, dan sisa memori.

### Langkah-langkah & Potongan Kode

#### 1. Definisi Warna Output

```sh
CYAN='\033[1;36m'
GREEN='\033[1;32m'
YELLOW='\033[1;33m'
WHITE='\033[1;37m'
RESET='\033[0m'
```

Kode `\033[` adalah escape code ANSI yang didukung oleh terminal Linux. `printf` digunakan (bukan `echo`) karena lebih konsisten dalam memproses escape sequence di BusyBox sh.

#### 2. Mengumpulkan Informasi Sistem

```sh
SYS_HOSTNAME=$(hostname 2>/dev/null || echo "unknown")
SYS_DATE=$(date "+%A, %d %B %Y %H:%M:%S" 2>/dev/null)
SYS_UPTIME=$(uptime | sed 's/.*up /up /' | cut -d',' -f1-2)
SYS_USER=$(whoami 2>/dev/null)
SYS_HOME=$(echo "$HOME")
SYS_PROCS=$(ps | wc -l)
SYS_PROCS=$((SYS_PROCS - 1))  # kurangi baris header ps
SYS_MEM=$(free | awk '/^Mem:/ {
    printf "Total: %dK | Used: %dK | Free: %dK", $2, $3, $4
}')
```

- `2>/dev/null || echo "unknown"` : fallback jika perintah gagal
- `ps | wc -l` : hitung baris output ps, dikurangi 1 untuk header
- `awk '/^Mem:/'` : filter baris memory dari output `free`.

#### 3. Menampilkan Output Terformat

```sh
SEP="${CYAN}=================================================${RESET}"

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
```

- `%-18s` : rata kiri dengan lebar 18 karakter untuk alignment yang rapi
- `printf` dipakai karena BusyBox `echo` tidak selalu mendukung `-e`.

#### 4. Pemasangan ke Rootfs dan Permission

```bash
cp rootfs_template/sysinfo rootfs/bin/sysinfo
chmod 755 rootfs/bin/sysinfo  # rwxr-xr-x → semua user bisa execute
```

- `chmod 755` : owner bisa baca/tulis/eksekusi, group dan other bisa baca/eksekusi
- Karena `/bin` ada di `$PATH`, script langsung bisa dipanggil dengan `sysinfo` dari direktori manapun.

Contoh output saat dijalankan:

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

```sh
#!/bin/sh
# /bin/sysinfo

CYAN='\033[1;36m'
GREEN='\033[1;32m'
WHITE='\033[1;37m'
RESET='\033[0m'

SEP="${CYAN}=================================================${RESET}"

SYS_HOSTNAME=$(hostname 2>/dev/null || echo "unknown")
SYS_DATE=$(date "+%A, %d %B %Y %H:%M:%S" 2>/dev/null || echo "unknown")
SYS_UPTIME=$(uptime 2>/dev/null | sed 's/.*up /up /' | cut -d',' -f1-2 || echo "unknown")
SYS_USER=$(whoami 2>/dev/null || echo "unknown")
SYS_HOME=$(echo "$HOME")
SYS_PROCS=$(ps | wc -l 2>/dev/null || echo "0")
SYS_PROCS=$((SYS_PROCS - 1))
SYS_MEM=$(free 2>/dev/null | awk '/^Mem:/ {
    printf "Total: %dK | Used: %dK | Free: %dK", $2, $3, $4
}' || echo "unknown")

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
```

Lihat juga [`rootfs_template/sysinfo`](rootfs_template/sysinfo)
