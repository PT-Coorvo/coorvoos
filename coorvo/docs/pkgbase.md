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

Paket di hasil `comm` tidak ikut di-upgrade karena tidak ada di repo 16.
Contoh nyata di VM uji: `kernel-generic-mmccam`, `kernel-generic-nodebug`,
`kernel-minimal` (masing-masing beserta `-dbg`), dan `lldb-dev` 15.0 sisa
image awal. Biarkan dulu; urus setelah migrasi (langkah 7).

```sh
# 4. Ganti nama dan origin di database (tidak menyentuh file)
pkg set -y -p -n FreeBSD-:CoorvoOS-     # di VM uji: 522 paket
pkg set -y -p -o base/FreeBSD-:base/CoorvoOS-
pkg info -x '^FreeBSD-'                 # harus kosong
pkg check -d -n
```

Di titik ini `pkg check -d -n` melaporkan ratusan
`missing dependency: FreeBSD-*`. Itu wajar: `pkg set -n` hanya mengganti
nama paket, bukan nama dependensinya (lihat di atas). Keluhan ini hilang
sendiri setelah langkah 5, karena paket baru membawa dependensi `CoorvoOS-*`.

```sh
# 4b. Paket yang dikunci (pkg lock) dilewati oleh pkg upgrade
pkg lock -l
pkg unlock -y -g 'CoorvoOS-*'

# 5a. Uji coba. Simpan seluruh output ke file, jangan hanya tail
pkg upgrade -n -f -r CoorvoOS-base -x '^CoorvoOS-' > /root/migrate-dryrun.log 2>&1
grep -E 'to be|Number of' /root/migrate-dryrun.log
```

Yang boleh muncul hanya `to be UPGRADED` dan `to be REINSTALLED`. Kalau ada
yang lain, terutama `to be REMOVED`, hentikan dan jangan lanjut.

```sh
# 5b. Jalankan sebenarnya di background dengan log: sshd ikut diperbarui,
#     sehingga sesi SSH bisa putus di tengah
nohup pkg upgrade -y -f -r CoorvoOS-base -x '^CoorvoOS-' > /root/migrate.log 2>&1 &
tail -f /root/migrate.log
pkg check -d -n      # sekarang hanya paket sisa dari comm yang mengeluh

# 6. Paket baru yang ada di 16 tetapi belum terpasang
pkg install -n -r CoorvoOS-base CoorvoOS-set-base CoorvoOS-set-kernels
```

### Langkah 7: paket sisa, `.pkgnew`, `.pkgsave`

**Paket sisa.** `CoorvoOS-lldb-dev` 15.0 membuat `pkg check -d` mengeluh
(dependensi `FreeBSD-lldb`, `libprivateclang.so.19`). Isinya hanya symlink
`/usr/lib/libprivatelldb.so`, jadi aman dihapus. Paket kernel sisa periksa
dulu dengan `pkg info -l` sebelum memutuskan.

```sh
pkg info -l CoorvoOS-lldb-dev
pkg delete -y CoorvoOS-lldb-dev
```

**`.pkgnew`.** pkg menulis `<file>.pkgnew` dan membiarkan `<file>` apa
adanya kalau file konfigurasi di disk berbeda dari yang tercatat di paket
lama dan merge gagal (`pkg_extract_finalize` di `libpkg/pkg_add.c`). Setelah
installworld, ini biasa terjadi pada `/etc/rc.d/*`, yang dipasang sebagai
`CONFS`. Di VM uji, `/etc/rc.d/os-release.pkgnew` berisi versi FreeBSD,
sedangkan file aktif sudah CoorvoOS. Jadi jangan menilai dari `.pkgnew`;
cocokkan dengan isi paket:

```sh
for f in $(find /etc /boot /usr/local/etc -name '*.pkgnew' 2>/dev/null); do
  if cmp -s "${f%.pkgnew}" "$f"; then echo "IDENTIK: $f"
  else echo "===== BEDA: $f"; diff -u "${f%.pkgnew}" "$f"; fi
done

R=/usr/obj/usr/src/repo/FreeBSD:16:amd64/latest
ls $R | grep '^CoorvoOS-rc-[0-9]'      # nama file paket yang persis
tar -xOf $R/CoorvoOS-rc-<versi>.pkg /etc/rc.d/os-release | grep 'NAME='
```

Tulis nama file paket persis; glob `CoorvoOS-rc-*.pkg` ikut mencocokkan
paket `-dbg`. Kalau isi paket sama dengan file aktif, `.pkgnew` dihapus
saja. Kalau isi paket yang berisi FreeBSD, paketnya dibangun dari tree lama:
bangun ulang paket, jangan hanya menghapus `.pkgnew`.

**`.pkgsave`.** pkg menyimpan file yang ditimpanya sebagai `.pkgsave` (di VM
uji: 271 file). Biarkan sampai sistem terbukti boot normal, lalu:

```sh
find / -xdev -name '*.pkgsave' -delete
```

```sh
# 8. Reboot dan verifikasi
shutdown -r now
uname -v                                   # CoorvoOS ...
uname -s                                   # FreeBSD (disengaja, lihat bawah)
pkg info | grep -c '^CoorvoOS-'            # ratusan
pkg info | grep '^FreeBSD-'                # kosong
pkg upgrade -n -r CoorvoOS-base            # "Your packages are up to date"
```

`uname -s` (`kern.ostype`) tetap `FreeBSD` dengan sengaja: nilai itu
membentuk ABI pkg (`FreeBSD:16:amd64`) dan dipakai build system serta
configure script untuk mengenali platform. Nama produk ada di
`/etc/os-release`, motd, banner login (`gettytab`), loader, dan `uname -v`.

### Rollback

**Hanya kalau ada masalah.** Perintah di bawah membatalkan migrasi dan bukan
bagian dari urutan normal.

- Sebelum langkah 5 (hanya database yang berubah):
  `rm -rf /var/db/pkg && mv /var/db/pkg.before-coorvo /var/db/pkg`.
- Setelah langkah 5: kembali ke snapshot VMware, atau
  `bectl activate pre-coorvo-pkgbase && shutdown -r now`.

Untuk VM uji coba, alternatif paling aman adalah memasang CoorvoOS baru dari
ISO yang sudah berisi paket `CoorvoOS-*`, tanpa migrasi.

## 5. Masalah yang diketahui: makewhatis Signal 11

Di VM uji, `make installworld` dua kali gagal di langkah terakhir
(`share/man`, `make makedb`) dan `make packages` sekali gagal di stageworld,
karena `makewhatis` crash dengan Signal 11. dmesg:
`pid NNNN (mandoc), ... exited on signal 11 (core dumped)`. Percobaan ulang
tanpa perubahan lolos, jadi crash-nya acak. `makewhatis` yang dijalankan
manual dan trigger pkg `mandoc.ucl` tidak pernah crash. Belum ada backtrace.

Yang sudah diperiksa di source:

- Tidak ada perubahan CoorvoOS di `contrib/mandoc`, `usr.bin/mandoc`,
  `lib/libc`, atau `share/man`. Tiga man page yang diubah (`loader.conf.5`,
  `menu.4th.8`, `drawer.lua.8`) hanya berganti kata.
- Selama installworld/stageworld, `makewhatis` yang jalan **bukan**
  `/usr/bin/makewhatis`. `usr.bin/mandoc` selalu dibangun sebagai bootstrap
  tool (daftar `bootstrap-tools` di `Makefile.inc1`) dan dipasang di
  `${WORLDTMP}/legacy/usr/bin`. PATH install (`IMAKEENV`) menaruh
  `STRICTTMPPATH`, yang berisi folder legacy itu, di depan `INSTALLTMP`.
- Di folder legacy, `LINKS` dipasang oleh `tools/install.sh`, yang
  mengubah `-l mr` menjadi symlink, jadi `makewhatis -> mandoc`. Kernel
  menamai proses dari komponen terakhir path setelah symlink diikuti, maka
  dmesg menyebut `(mandoc)`. `/usr/bin/makewhatis` adalah hardlink, sehingga
  crash di sana akan tercatat sebagai `(makewhatis)`.
- Riwayat upstream sejak vendor import 2025-09-26 hanya mengubah tabel teks
  (`st.c`, `lib.in`) dan const untuk C23 (`9f18614d535`, tanpa perubahan
  perilaku). NEWS mandoc tidak mencatat rilis setelah 1.14.6.

Jadi dugaan terkuat: yang crash adalah binary bootstrap di folder legacy.
Ini belum terbukti; perlu backtrace.

```sh
# 0. Binary mana yang ada
L=/usr/obj/usr/src/amd64.amd64/tmp/legacy/usr/bin
ls -li /usr/bin/mandoc /usr/bin/makewhatis
ls -l $L/mandoc $L/makewhatis

# 1. Arahkan core dump
mkdir -p /var/crash
sysctl kern.coredump=1
sysctl kern.corefile=/var/crash/%N.%P.core
ulimit -c unlimited

# 2. Ulangi sampai crash (ganti BIN dan DIR untuk tiap kombinasi:
#    BIN = $L/makewhatis atau /usr/bin/makewhatis,
#    DIR = /usr/share/man atau .../worldstage/usr/share/man)
BIN=$L/makewhatis
DIR=/usr/obj/usr/src/amd64.amd64/worldstage/usr/share/man
i=0
while [ $i -lt 200 ]; do
  i=$((i + 1))
  $BIN $DIR; rc=$?
  if [ $rc -ne 0 ]; then echo "iterasi $i: exit $rc"; break; fi
done
ls -l /var/crash/*.core; dmesg | tail -3

# 3. Backtrace; "image list" menunjukkan path binary yang crash
file /var/crash/<nama>.core
lldb -b -o 'image list' -o 'bt all' -c /var/crash/<nama>.core $BIN
```

Binary legacy tidak di-strip (`tools/install.sh` mengabaikan `-s`), jadi
nama fungsi terlihat di backtrace. Untuk
`/usr/bin/makewhatis`, pasang dulu `CoorvoOS-mandoc-dbg` supaya lldb punya
simbol.

## 6. Yang belum ikut CoorvoOS

Konfigurasi image cloud dan OCI di `release/tools/*.conf` (ec2, gce, azure,
oci-image-*) dan `release/scripts/make-oci-image.sh` masih memakai nama
`FreeBSD-*`/`FreeBSD-base`. Perbarui sebelum membuat image tersebut.
`release/tools/vmimage.subr` (dipakai `make vm-image`) sudah diperbarui.
