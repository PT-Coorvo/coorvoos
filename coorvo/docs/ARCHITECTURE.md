# Arsitektur CoorvoOS

CoorvoOS dibangun di atas **FreeBSD src 16.0-CURRENT** (`sys/conf/newvers.sh`:
`REVISION="16.0"`, `BRANCH="CURRENT"`). Dokumen ini memetakan struktur source
tree, cara sistem dibangun, dan alur kerja dari boot sampai login, sebagai
dasar untuk kustomisasi CoorvoOS.

## 1. Identitas dan ukuran repo

| Item | Nilai |
|---|---|
| Upstream | FreeBSD src (`main`), commit terakhir 2026-10-20 |
| File tracked | ±114.640 |
| Riwayat commit | ±314.000 |
| Bahasa utama | C (±22.6k `.c`, ±21.9k `.h`), C++ (±6k, mayoritas LLVM/Clang), shell/ksh (test & rc), Device Tree (`.dts/.dtsi`), assembly (`.S`), Lua (boot loader) |

Jumlah file per direktori utama:

| Direktori | File | Direktori | File |
|---|---|---|---|
| contrib | 37.202 | usr.bin | 3.327 |
| sys | 36.693 | usr.sbin | 2.895 |
| crypto | 10.400 | tests | 2.455 |
| lib | 5.721 | cddl | 2.237 |
| tools | 3.930 | sbin | 1.305 |
| share | 3.845 | bin | 1.157 |

Kode milik FreeBSD sendiri ada di `sys/`, `lib/`, `bin/`, `sbin/`, `usr.*`,
`libexec/`, `stand/`. `contrib/`, `crypto/`, dan `cddl/contrib` berisi
software pihak ketiga yang di-*vendor* (LLVM, OpenSSL, OpenSSH, OpenZFS,
Lua, libarchive, dll.).

## 2. Peta direktori top-level

| Direktori | Isi | Terpasang ke |
|---|---|---|
| `bin/` | Perintah dasar (sh, ls, cp, ps, ...) | `/bin` |
| `sbin/` | Perintah sistem (init, mount, ifconfig, pfctl, fsck, zfsbootcfg, ...) | `/sbin` |
| `usr.bin/` | Perintah user (login, grep, uname, vi, ...) | `/usr/bin` |
| `usr.sbin/` | Perintah admin (bsdinstall, pkg, bhyve, jail, ...) | `/usr/sbin` |
| `lib/` | Library sistem (libc, libsys, libutil, libc++, libpam, libcasper, ...) | `/lib`, `/usr/lib` |
| `libexec/` | Program yang dipanggil program lain (rc, rtld-elf, getty, ...) | `/libexec`, `/usr/libexec`, `/etc/rc.d` |
| `etc/` | Template `/etc` (master.passwd, group, mtree) | `/etc` |
| `share/` | Data bersama: man page, `share/mk` (framework make), locale, keymap | `/usr/share` |
| `include/` | Header sistem userland | `/usr/include` |
| `stand/` | Boot loader (EFI, BIOS, Lua menu, libsa) | `/boot` |
| `sys/` | Kernel dan modul | `/boot/kernel` |
| `contrib/`, `crypto/`, `cddl/` | Software pihak ketiga + glue build | — |
| `secure/`, `kerberos5/`, `krb5/` | Glue build untuk komponen kriptografi & Kerberos | — |
| `rescue/` | `/rescue`: binary statis untuk pemulihan | `/rescue` |
| `packages/` | Definisi paket **pkgbase** (sistem dasar sebagai paket pkg) | — |
| `release/` | Pembuatan ISO, memstick, VM & cloud image | — |
| `tests/` | Kerangka test suite (ATF/Kyua) | `/usr/tests` |
| `tools/` | Alat bantu dev & build (`tools/build/make.py`), tidak ikut terpasang | — |
| `targets/` | Dukungan eksperimental `DIRDEPS_BUILD` | — |

## 3. Kernel (`sys/`)

### 3.1 Subsistem inti

| Direktori | Fungsi | File kunci |
|---|---|---|
| `kern/` | Inti kernel: proses/thread, scheduler, syscall, sinyal, VFS, IPC, locking | `init_main.c` (start kernel), `kern_fork.c`, `sched_ule.c` (scheduler default), `sched_4bsd.c`, `syscalls.master` (tabel syscall), `vfs_*.c` (lapisan VFS), `subr_witness.c` (deteksi lock order) |
| `vm/` | Virtual memory: map, object, page, pageout, swap, alokator UMA | `vm_map.c`, `vm_fault.c`, `vm_page.c`, `vm_pageout.c`, `uma_core.c` |
| `sys/` | Header kernel (dipakai juga userland) | `param.h` (`__FreeBSD_version`), `sysctl.h` |
| `libkern/` | Fungsi mirip libc untuk kernel | |
| `security/` | MAC framework, audit | |
| `ddb/`, `gdb/` | Debugger kernel interaktif & stub GDB remote | |

### 3.2 Jaringan

`net/` (inti: ifnet, routing, bridge, VLAN), `netinet/` & `netinet6/`
(TCP/IP v4/v6), `netlink/` (konfigurasi jaringan gaya Linux), `netpfil/`
(firewall **pf**, **ipfw**, ipfilter), `netipsec/`, `netgraph/`,
`net80211/` (Wi-Fi), `netsmb/`, `nfs*/`, `rpc/`, `ofed/` (InfiniBand/RDMA).

### 3.3 Penyimpanan & filesystem

- `geom/` — framework blok modular (partisi GPT/MBR, mirror, enkripsi ELI).
- `cam/` — subsistem SCSI/ATA/NVMe (+ `ctl` target).
- `ufs/` — UFS/FFS (soft updates, journaling).
- `fs/` — filesystem lain (tmpfs, nullfs, fusefs, msdosfs, cd9660, ...).
- ZFS — OpenZFS di `sys/contrib/openzfs`, glue di `sys/cddl`.

### 3.4 Driver dan arsitektur

- `dev/` — ribuan driver perangkat independen-arsitektur.
- Arsitektur yang didukung: `amd64`, `arm64`, `riscv`, `powerpc`, `i386`
  (kernel), `arm` (32-bit); `x86/` untuk kode bersama amd64/i386.
- `dts/` + `sys/contrib/device-tree` — device tree untuk board ARM/RISC-V.
- `compat/` — Linuxulator (menjalankan binary Linux), freebsd32, LinuxKPI
  (port driver grafis/Wi-Fi Linux).
- `xen/` dan `dev/hyperv`, `dev/virtio` — dukungan virtualisasi sebagai guest;
  `usr.sbin/bhyve` + `sys/amd64/vmm` — hypervisor bhyve sebagai host.

### 3.5 Konfigurasi kernel

Konfigurasi ada di `sys/<arch>/conf/`:

- `GENERIC` — kernel rilis standar.
- `MINIMAL` — kernel kecil, fitur lain lewat modul.
- `NOTES` / `LINT` — dokumentasi semua opsi / build coverage.
- `GENERIC-NODEBUG`, `GENERIC-KASAN`, dll. — varian debug/sanitizer.

Kernel kustom CoorvoOS nantinya cukup dibuat sebagai file baru (mis.
`sys/amd64/conf/COORVO`) yang `include GENERIC` lalu menambah/menghapus opsi.

Identitas kernel dibuat saat build oleh `sys/conf/newvers.sh`, yang
menghasilkan `vers.c` berisi `ostype` (`kern.ostype`, `uname -s`), `version`
(`uname -v`, banner boot) dan copyright.

## 4. Alur boot

```
Firmware (UEFI / BIOS)
  └─ stand/efi/loader  atau  stand/i386 (boot0 → gptboot → loader)
       └─ loader membaca /boot/loader.conf (stand/defaults/loader.conf)
       └─ menu Lua: stand/lua/loader.lua → core.lua, menu.lua, drawer.lua
            (judul "Welcome to ...", logo/brand ASCII atau PNG dari stand/images)
       └─ memuat kernel + modul, lompat ke kernel
Kernel: sys/<arch>/... locore → sys/kern/init_main.c (mi_startup, SYSINIT)
  └─ mount root (vfs_mountroot.c) → jalankan /sbin/init (sbin/init)
/sbin/init
  └─ /etc/rc → rcorder atas libexec/rc/rc.d/* (mis. os-release, motd, netif, sshd)
  └─ getty (libexec/getty) → login (usr.bin/login) → tampilkan /var/run/motd
```

## 5. Userland

- **libc** (`lib/libc`) dan **libsys** (`lib/libsys`, wrapper syscall) adalah
  fondasi semua program; `libexec/rtld-elf` adalah dynamic linker.
- **Capsicum/Casper** (`lib/libcapsicum`, `lib/libcasper`) untuk sandboxing.
- **rc system** (`libexec/rc`) — skrip startup berbasis `rcorder` dengan
  dependensi `PROVIDE/REQUIRE/BEFORE`, dikonfigurasi lewat `/etc/rc.conf`.
- **Manajemen paket**: `usr.sbin/pkg` adalah bootstrap pkg(8); repo default di
  `usr.sbin/pkg/FreeBSD.conf.*` (`pkg.FreeBSD.org`). Dengan **pkgbase**
  (`packages/`), sistem dasar sendiri bisa dipasang/di-update sebagai paket.
- **Installer**: `usr.sbin/bsdinstall` (skrip shell + `bsddialog`); nama OS
  di installer berasal dari variabel `OSNAME` di
  `usr.sbin/bsdinstall/include/Makefile`.
- Jail (`usr.sbin/jail`), bhyve, ZFS boot environment (`sbin/bectl`).

## 6. Build system

- Menggunakan **bmake** dan framework di `share/mk/` (`bsd.prog.mk`,
  `bsd.lib.mk`, `bsd.kmod.mk`, ...).
- Entry point: `Makefile` (top-level) → `Makefile.inc1`.

| Target | Fungsi |
|---|---|
| `make buildworld` | Bootstrap toolchain lalu build seluruh userland |
| `make buildkernel KERNCONF=GENERIC` | Build kernel + modul |
| `make installworld` / `installkernel` | Pasang ke sistem (atau `DESTDIR`) |
| `make packages` | Buat paket pkgbase |
| `make -C release ...` | Buat ISO/memstick/VM image |

- Opsi build diatur lewat `/etc/src.conf` (`WITH_*` / `WITHOUT_*`, lihat
  `src.conf(5)` dan `tools/build/options/`).
- **Build dari Windows**: tidak bisa langsung. Gunakan WSL (Ubuntu) atau VM,
  lalu cross-build dengan `tools/build/make.py`, yang mem-*bootstrap* bmake
  dan alat lain di host non-FreeBSD:

  ```sh
  MAKEOBJDIRPREFIX=$HOME/obj ./tools/build/make.py --bootstrap-toolchain \
      TARGET=amd64 TARGET_ARCH=amd64 -j$(nproc) buildworld buildkernel
  ```

  Catatan: clone repo di filesystem Linux (bukan `/mnt/c`), karena
  filesystem Windows tidak case-sensitive dan lambat untuk build ini.

## 7. Release engineering (`release/`)

- `release/Makefile` — target `cdrom`, `memstick`, `mini-memstick`, `dvdrom`.
- `release/Makefile.vm` — image VM (qcow2, vmdk, vhd, raw); `Makefile.ec2`,
  `Makefile.azure`, `Makefile.gce`, `Makefile.oci`, `Makefile.vagrant`,
  `Makefile.firecracker` — image cloud.
- `release/release.sh` + `release.conf.sample` — build rilis lengkap dalam
  chroot.
- Skrip per arsitektur di `release/<arch>/` (mis. `mkisoimages.sh`).

## 8. Testing

- Test ditulis dengan **ATF** (C, C++, sh) atau GoogleTest, dijalankan dengan
  **Kyua**: `cd /usr/tests && kyua test`.
- Struktur `tests/` mencerminkan source tree; test kernel di `sys/tests` dan
  `tests/sys`.
- `sys/<arch>/conf/LINT` dipakai untuk memastikan semua opsi kernel tetap bisa
  dikompilasi.

## 9. Lisensi

- Mayoritas kode berlisensi **BSD-2-Clause** (lihat `COPYRIGHT`) — boleh
  dipakai dan di-rebrand secara komersial, **asal pemberitahuan hak cipta
  tetap dipertahankan**.
- Komponen **CDDL** (ZFS, DTrace) di `cddl/` dan `sys/cddl`, komponen **GPL**
  di `sys/gnu` — terikat lisensi masing-masing.
- Nama dan logo "FreeBSD" adalah merek dagang The FreeBSD Foundation; produk
  turunan sebaiknya memakai nama/logo sendiri (alasan rebranding CoorvoOS)
  sambil tetap menyatakan "berbasis FreeBSD".

## 10. Titik branding (dasar rebranding CoorvoOS)

| Yang terlihat user | Lokasi |
|---|---|
| Nama & versi kernel (`uname -v`, banner boot) | `sys/conf/newvers.sh` (`TYPE`, `VERSION`) |
| Judul menu boot | `stand/lua/drawer.lua` (`"Welcome to FreeBSD"`) |
| Logo ASCII boot | `stand/lua/drawer.lua` (`fbsd_brand`, `drawer.default_brand`) |
| Logo grafis boot | `stand/images/*.png` |
| Judul installer | `usr.sbin/bsdinstall/include/Makefile` (`OSNAME`) |
| `/etc/os-release` | `libexec/rc/rc.d/os-release` |
| Pesan login | `usr.bin/login/motd.template` |
| Nama image rilis | `release/` |

Yang **tidak boleh** diganti agar kompatibilitas paket dan software pihak
ketiga tetap terjaga: makro `__FreeBSD__` / `__FreeBSD_version`,
`kern.ostype` / `uname -s` (dipakai pkg sebagai ABI `FreeBSD:16:amd64` dan
oleh banyak skrip `configure`), serta repo pkg `FreeBSD.conf`.
