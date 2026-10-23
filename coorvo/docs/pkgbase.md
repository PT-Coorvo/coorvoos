# CoorvoOS pkgbase

Base system CoorvoOS (kernel, libc, `/bin`, `/etc`, dst.) dipecah menjadi
paket `pkg`, sehingga sistem terpasang bisa di-update dengan `pkg upgrade`
alih-alih `buildworld`/`installworld`.

| Hal | Nilai |
|---|---|
| Nama paket | `CoorvoOS-*` (mis. `CoorvoOS-runtime`, `CoorvoOS-kernel-generic`, `CoorvoOS-set-base`) |
| Diatur oleh | `PKG_NAME_PREFIX`, `PKG_MAINTAINER`, `PKG_WWW` di `Makefile.inc1` |
| Nama repo | `CoorvoOS-base` (`/etc/pkg/CoorvoOS.conf`, sumber: `usr.sbin/pkg/CoorvoOS.conf`) |
| Verifikasi | `signature_type: "pubkey"`, kunci `/usr/share/keys/pkg/coorvo-repo.pub` |
| ABI | tetap `FreeBSD:16:amd64` (sengaja, agar paket ports FreeBSD tetap cocok) |
| Repo ports | `FreeBSD-ports` di `/etc/pkg/FreeBSD.conf`, tetap aktif |

`FreeBSD-base` di `/etc/pkg/FreeBSD.conf` sudah `enabled: no` dari upstream.
File itu sengaja tidak diubah supaya sinkronisasi upstream tidak bentrok.

## 1. Kunci signing

Private key **hanya** ada di mesin build dan cadangan aman (password manager
tim). Jangan pernah di-commit; `.gitignore` sudah menolak `repo.key`,
`coorvo-keys/`, dan `share/keys/pkg/*.key|*.pem`.

```sh
mkdir -p /root/coorvo-keys && chmod 700 /root/coorvo-keys
openssl genrsa -out /root/coorvo-keys/repo.key 4096
chmod 600 /root/coorvo-keys/repo.key
openssl rsa -in /root/coorvo-keys/repo.key -pubout \
    -out /usr/src/share/keys/pkg/coorvo-repo.pub
```

Commit `share/keys/pkg/coorvo-repo.pub` ke repo. Selama file itu belum ada,
build tetap jalan tetapi kunci tidak terpasang ke `/usr/share/keys/pkg/`
(lihat `share/keys/pkg/Makefile`), sehingga repo `CoorvoOS-base` tidak bisa
diverifikasi.

Kalau private key hilang atau bocor, buat kunci baru, commit public key baru,
dan setiap mesin harus menerima public key baru itu secara manual
(lewat installworld atau menyalin file). Karena itu cadangkan `repo.key`.

## 2. Build dan signing paket

Di mesin build (VM CoorvoOS), dari `/usr/src`:

```sh
cd /usr/src && git pull
nohup sh -c 'make -j8 buildworld buildkernel && \
    make packages PKG_REPO_SIGNING_KEY=/root/coorvo-keys/repo.key' \
    > /root/build.log 2>&1 &
tail -f /root/build.log
```

`nohup sh -c '...'` dipakai agar kedua langkah terlindung dari logout;
`nohup a && b &` hanya melindungi `a`.

Build berikutnya cukup `make update-packages PKG_REPO_SIGNING_KEY=...`:
paket yang isinya tidak berubah dipertahankan dari versi sebelumnya
(target `real-update-packages` di `Makefile.inc1`).

Paket yang sudah ada bisa ditandatangani ulang tanpa membangun ulang:

```sh
make sign-packages PKG_REPO_SIGNING_KEY=/root/coorvo-keys/repo.key
```

## 3. Struktur repo yang dihasilkan

`REPODIR` default `${OBJROOT}repo`, yaitu `/usr/obj/usr/src/repo`:

```
/usr/obj/usr/src/repo/
└── FreeBSD:16:amd64/            <- ${ABI}
    ├── <PKG_VERSION>/           <- mis. 16.snap<tanggal><jam>
    │   ├── CoorvoOS-runtime-<PKG_VERSION>.pkg
    │   ├── CoorvoOS-kernel-generic-<PKG_VERSION>.pkg
    │   ├── CoorvoOS-set-base-<PKG_VERSION>.pkg
    │   ├── ... (ratusan paket)
    │   ├── meta.conf, packagesite.*, data.*   <- katalog repo (pkg repo)
    └── latest -> <PKG_VERSION>  <- symlink, dibuat oleh real-sign-packages
```

Paket berada langsung di folder versi (tanpa subfolder `All/`). Cek:

```sh
ls -l /usr/obj/usr/src/repo/*/
ls /usr/obj/usr/src/repo/*/latest | grep -c '^CoorvoOS-'
ls /usr/obj/usr/src/repo/*/latest | grep '^FreeBSD-'   # harus kosong
```

URL di `CoorvoOS.conf` (`file:///usr/obj/usr/src/repo/${ABI}/latest`) dan di
installer (`https://pkg.coorvo.com/${ABI}/latest`) mengikuti struktur ini.
Untuk publikasi, salin folder `${ABI}/` ke server apa adanya (termasuk
symlink `latest`), lalu ganti URL di `usr.sbin/pkg/CoorvoOS.conf`.

Di mesin tanpa `/usr/obj/.../repo`, `pkg update` akan melaporkan error untuk
`CoorvoOS-base` sampai URL diganti ke `pkg.coorvo.com`.

## 4. Memindahkan sistem terpasang dari FreeBSD-* ke CoorvoOS-*

Berlaku untuk mesin yang dipasang dengan pkgbase FreeBSD (database `pkg`
berisi `FreeBSD-runtime`, `FreeBSD-kernel-generic`, dst.), lalu dijadikan
CoorvoOS dengan installworld/installkernel dari tree ini.

### Kenapa tidak boleh langsung `pkg install CoorvoOS-*`

Perilaku pkg (diperiksa di source pkg, bukan asumsi):

- `libpkg/pkg_jobs_conflicts.c`: paket remote yang berisi path yang sudah
  dimiliki paket lain yang terpasang dicatat sebagai konflik
  `PKG_CONFLICT_REMOTE_LOCAL`. Solver lalu harus memilih salah satu, yaitu
  **menghapus** paket lama `FreeBSD-*` beserta file-nya (libc, `/bin/sh`,
  `/boot/kernel`) sebelum paket baru terpasang. Proses yang terputus di
  tengah meninggalkan sistem yang tidak bisa boot.
- Set `base`, `base-jail`, `devel`, `lib32` ditandai `vital = true`
  (`release/packages/sets/*.ucl`); pkg menolak menghapus paket vital tanpa
  dipaksa.
- Jangan pernah menjalankan `pkg delete FreeBSD-*`.

### Cara aman: ganti nama di database, lalu pasang ulang di tempat

`pkg set -p -n lama:baru` (`src/set.c`, fungsi `pkgdb_replace`) mengganti
teks `lama` menjadi `baru` pada nama **semua** paket terpasang, **tanpa
menyentuh file**. `-p` berarti ganti substring di mana pun, jadi periksa
dulu bahwa tidak ada paket lain yang namanya mengandung `FreeBSD-` di
tengah. `-n` dan `-o` tidak bisa digabung dalam satu perintah.

Nama pada catatan dependensi **tidak** ikut terganti: query `PKG_SET_DEPNAME`
di `libpkg/pkgdb.c` menulis `glob '?3'` (parameter di dalam tanda kutip),
sehingga tidak pernah cocok. `-o` memang mengganti origin dependensi, tetapi
nama dependensi tetap `FreeBSD-*` sampai `pkg upgrade -f` menulis ulang
setiap paket beserta dependensinya dari repo.

Setelah nama di database sama dengan nama di repo, `pkg upgrade -f` memasang
ulang setiap paket sebagai upgrade paket yang sama: file ditimpa di
tempatnya, bukan dihapus dulu.

### Prasyarat

1. **Snapshot VM/ZFS dulu.** Di VMware: matikan VM, ambil snapshot.
   Di ZFS: `bectl create pre-coorvo-pkgbase`.
2. World dan kernel yang berjalan dibangun dari source yang sama dengan
   paket (`uname -v` menunjukkan CoorvoOS).
3. Paket sudah dibuat dan ditandatangani (bagian 2).
4. Public key dan konfigurasi repo sudah ada di sistem. Kalau belum lewat
   installworld:
   ```sh
   install -m 644 /usr/src/share/keys/pkg/coorvo-repo.pub /usr/share/keys/pkg/
   install -m 644 /usr/src/usr.sbin/pkg/CoorvoOS.conf /etc/pkg/
   ```

### Langkah

```sh
# 0. Cadangkan database pkg
cp -a /var/db/pkg /var/db/pkg.before-coorvo

# 1. Repo CoorvoOS bisa dibaca dan tanda tangan valid
pkg update -f -r CoorvoOS-base
pkg rquery -r CoorvoOS-base '%n' | grep -c '^CoorvoOS-'

# 2. Pastikan tidak ada paket non-base yang ikut terganti oleh -p
pkg query '%n' | grep 'FreeBSD-' | grep -v '^FreeBSD-'    # harus kosong
pkg query '%o' | grep 'FreeBSD-' | grep -v '^base/FreeBSD-' # harus kosong

# 3. Bandingkan daftar paket terpasang dengan repo
pkg query '%n' | sed -n 's/^FreeBSD-//p' | sort > /tmp/local.txt
pkg rquery -r CoorvoOS-base '%n' | sed -n 's/^CoorvoOS-//p' | sort > /tmp/remote.txt
comm -23 /tmp/local.txt /tmp/remote.txt   # terpasang tapi tidak ada di repo
```

Paket di hasil `comm` (biasanya paket yang dihapus/diganti nama antara 15.x
dan 16) tidak ikut di-upgrade. Biarkan dulu; setelah migrasi periksa isinya
dengan `pkg info -l CoorvoOS-<nama>` sebelum memutuskan menghapusnya.

```sh
# 4. Ganti nama dan origin di database (tidak menyentuh file)
pkg set -y -p -n FreeBSD-:CoorvoOS-
pkg set -y -p -o base/FreeBSD-:base/CoorvoOS-
pkg info -x '^FreeBSD-'          # harus kosong
pkg check -d -n                  # wajar melaporkan dependensi FreeBSD-* hilang;
                                 # beres setelah langkah 5

# 4b. Paket yang dikunci dilewati oleh pkg upgrade; buka kuncinya
pkg lock -l
pkg unlock -y -g 'CoorvoOS-*'    # hanya kalau daftar di atas tidak kosong

# 5. Uji coba dulu, lalu pasang ulang paket base dari repo CoorvoOS
pkg upgrade -n -f -r CoorvoOS-base -x '^CoorvoOS-'
pkg upgrade -f -r CoorvoOS-base -x '^CoorvoOS-'

# 6. Paket baru yang ada di 16 tetapi belum terpasang
pkg install -n -r CoorvoOS-base CoorvoOS-set-base CoorvoOS-set-kernels
```

Pada langkah 5, periksa hasil `-n` sebelum menjalankan yang sebenarnya:
daftar harus berisi **REINSTALLED/UPGRADED**, bukan **REMOVED** untuk paket
base. Kalau ada paket base yang akan dihapus, hentikan dan kembali ke
snapshot.

Langkah 7: pkg menulis `<file>.pkgnew` dan membiarkan `<file>` apa adanya
kalau file konfigurasi di disk berbeda dari yang tercatat di paket lama dan
merge gagal (`pkg_extract_finalize` di `libpkg/pkg_add.c`). Setelah
installworld, ini biasa terjadi pada `/etc/rc.d/*`, yang dipasang sebagai
`CONFS`. Isi `.pkgnew` adalah isi paket baru; kalau di dalamnya masih ada
teks FreeBSD (mis. `NAME=FreeBSD` di `rc.d/os-release`), paket itu dibangun
dari tree lama. Cocokkan dengan paketnya sebelum menghapus:

```sh
# 7. File konfigurasi yang gagal di-merge otomatis
for f in $(find /etc /boot /usr/local/etc -name '*.pkgnew' 2>/dev/null); do
  if cmp -s "${f%.pkgnew}" "$f"; then echo "IDENTIK: $f"
  else echo "===== BEDA: $f"; diff -u "${f%.pkgnew}" "$f"; fi
done
grep '^[[:space:]]*NAME=' /etc/rc.d/os-release*   # harus NAME=CoorvoOS
# IDENTIK: hapus .pkgnew. BEDA: gabungkan manual, lalu hapus .pkgnew.

# 8. Reboot dan verifikasi
shutdown -r now
uname -v                                   # CoorvoOS ...
pkg info | grep -c '^CoorvoOS-'            # ratusan
pkg info | grep '^FreeBSD-'                # kosong
pkg upgrade -n -r CoorvoOS-base            # "Your packages are up to date"
```

### Rollback

- Sebelum langkah 5 (hanya database yang berubah):
  `rm -rf /var/db/pkg && mv /var/db/pkg.before-coorvo /var/db/pkg`.
- Setelah langkah 5: kembali ke snapshot VMware, atau
  `bectl activate pre-coorvo-pkgbase && shutdown -r now`.

Untuk VM uji coba, alternatif paling aman adalah memasang CoorvoOS baru dari
ISO yang sudah berisi paket `CoorvoOS-*`, tanpa migrasi.

## 5. Masalah yang diketahui

`makewhatis` (mandoc) crash dengan Signal 11 saat `make packages`, sudah tiga
kali, dan sempat menggagalkan build. Selidiki sebelum `make release`.

Yang dijalankan adalah `makewhatis` milik **host** (`/usr/bin/makewhatis`,
disalin ke `INSTALLTMP` lewat `ITOOLS` di `Makefile.inc1`), dipanggil target
`makedb` di `share/man/Makefile` pada direktori staging. Jadi crash berasal
dari world yang terpasang di mesin build, bukan dari hasil build baru.
Perubahan man page di branch ini (`loader.conf.5`, `menu.4th.8`,
`drawer.lua.8`) hanya mengganti kata, tidak mengubah struktur mdoc.

Data yang dibutuhkan:

```sh
grep -n -B5 'Signal 11' /root/build.log
# jalankan ulang di direktori staging (WSTAGEDIR = ${OBJTOP}/worldstage)
sysctl kern.coredump=1; ulimit -c unlimited
makewhatis -p /usr/obj/usr/src/amd64.amd64/worldstage/usr/share/man
dmesg | tail          # "pid ... (makewhatis), ... exited on signal 11"
```

Kalau crash bisa diulang, cari file man penyebabnya dengan `makewhatis -t`
per file atau `mandoc -T lint`. Kalau tidak bisa diulang, curigai memori VM
(RAM/swap habis saat `-j8`).

## 6. Yang belum ikut CoorvoOS

Konfigurasi image cloud dan OCI di `release/tools/*.conf` (ec2, gce, azure,
oci-image-*) dan `release/scripts/make-oci-image.sh` masih memakai nama
`FreeBSD-*`/`FreeBSD-base`. Perbarui sebelum membuat image tersebut.
`release/tools/vmimage.subr` (dipakai `make vm-image`) sudah diperbarui.
