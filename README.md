# Menjalankan Binary Linux di Termux (Android)

Panduan untuk menjalankan aplikasi/CLI yang "unsupported" di Termux — terutama yang
distribusikan sebagai binary Linux native, bukan paket npm/python murni.

Diperbarui: 2026-10-01 · Tested di Termux, aarch64

---

## 1. Kenapa binary Linux biasa gagal di Termux

Dua sebab yang berbeda, tapi sering ketuker:

### 1.1 Android bukan Linux untuk purposes kita

Android kernel = Linux. Tapi userspace-nya **bionic**, bukan glibc.

| | Termux | Debian/Ubuntu | Alpine |
|---|---|---|---|
| libc | bionic | glibc | musl |
| Dynamic linker | `/system/bin/linker64` | `/lib/ld-linux-aarch64.so.1` | `/lib/ld-musl-aarch64.so.1` |

Binary yang di-build untuk Debian/Ubuntu **butuh glibc** untuk startup. Di
Termux file itu tidak ada, sehingga:

```
$ ./some-binary
bash: ./some-binary: cannot execute: required file not found
```

Pesan "required file not found" itu menyesatkan — filenya **ada**, yang hilang
adalah *interpreter* yang diminta ELF-nya.

Cek dulu:

```bash
ls /lib/ld-linux-aarch64.so.1 2>/dev/null || echo "no glibc → bionic"
file ./some-binary        # lihat baris "interpreter" dan "for GNU/Linux"
```

### 1.2 Manajer paket menolak based on `os` field

Banyak paket npm/PyPI/Fyne punya metadata `os: ["darwin","linux","win32"]`.
npm baca `process.platform` = `android`, bukan `linux`, lalu gagal install:

```
npm error code EBADPLATFORM
npm error notsup Unsupported platform: freebuff@0.2.5: wanted {"os":"darwin,linux,win32"}
```

`--force` loloskan **install**, tapi tidak melewati runtime — binary-nya
masih glibc dan masih tidak bisa jalan.

> **Penting:** `--force` itu hanya bagian 1 dari 2. Kalau install berhasil tapi
> jalan masih gagal, itu bukan "install gagal" — itu masalah libc, Section 1.1.

### 1.3 Launcher yang download binary di runtime

Pola yang sering dipakai CLI modern (freebuff, codebuff,ripgrep versi tertentu,
dsb.): paket npm-nya cuma **launcher tipis**, lalu saat dijalankan ia download
binary asli dari server. Di situ ada lapis kedua:

- Metadata paket menyatakan Dukungan platform tidak lengkap
- Launcher-nya sendiri punya daftar target terpisah, dan bisa tidak punya
  entry untuk android walau paketnya lolos `--force`

Gejalanya:

```
$ freebuff --version
❌ Failed to download freebuff: Unsupported platform: android arm64
```

Di `launcher.js` freebuff, target cuma: `linux-x64`, `linux-x64-baseline`,
`linux-arm64`, `darwin-x64`, `darwin-arm64`, `win32-x64`, plus baseline.
Tidak ada `android-arm64`.

---

## 2. Solusinya: glibc-runner

**glibc-runner** (dari proyek Termux-Pacman) menyediakan userland glibc di atas
kernel Android **secara native** — tanpa proot, tanpa virtualisasi, tanpa root.

Bedanya dengan proot:

| | glibc-runner | proot-distro |
|---|---|---|
| Cara kerja | Exec ELF glibc dengan loader/library glibc milik Termux | Userland virtual via `ptrace` |
| Overhead | Sangat rendah,~|native |.signifikan |
| Peroesan | Butuh paket glibc saja | Butuh rootfs penuh (~500MB-1GB) |
| Kesesuaian | Binary statis & glibc statis | Apa pun yang relies ke sistem |

Untuk kasus kita (binary glibc) glibc-runner **jauh lebih tepat**.

### 2.1 Install

```bash
pkg install glibc-repo -y     # repo glibc
pkg install glibc-runner -y   # runner-nya (alias: grun)
```

Verifikasi:

```bash
grun --help
ls $PREFIX/glibc/lib/ld-linux-aarch64.so.1   # harus ada
```

Kalau output `glibc-runner: command not found`, `apt update` dulu lalu ulangi
`pkg install`.

### 2.2 Pakai

```bash
grun ./binary --version      # bungkus satu binary
grun -s                     # shell glibc penuh
grun gcc -v                 # toolchain glibc
grun --configure ./binary   # patch binary supaya jalan tanpa dibungkus
```

Semua argumen setelah nama binary diteruskan ke binary tersebut.

---

## 3. Resep: CLI yang distribusikan sebagai launcher npm

Contoh kasus freebuff. Contoh ini berlaku umum untuk CLI yangATEGORICAL
"download binary saat runtime".

### 3.1 Ambil binary aslinya

Cari URL download di `launcher.js` / `index.js` paket:

```bash
npm pack freebuff && tar -xzf freebuff-0.2.5.tgz
grep -n -E "https?://|DOWNLOAD_URL" package/launcher.js
# https://codebuff.com/api/releases/download/<version>/<file>
```

Format URL-nya:

```
https://codebuff.com/api/releases/download/0.2.5/freebuff-linux-arm64.tar.gz
```

Pola umum file name: `<nama>-<platform>-<arch>.tar.gz`, dengan `platform`
menggunakan `linux` (bukan `android`) untuk Termux — karena target arm64-nya
memang dibangun untuk glibc.

```bash
mkdir -p ~/.local/share/freebuff && cd ~/.local/share/freebuff
curl -sLO "https://codebuff.com/api/releases/download/0.2.5/freebuff-linux-arm64.tar.gz"
```

### 3.2 WAJIB verifikasi checksum

Paket npm biasanya menyertakan `binaryChecksums` di `package.json`. **Lakukan
ini — Anda sedang menjalankan binary strangers dari internet.**

```bash
# ambil checksum yang diharapkan dari package.json
npm view freebuff binaryChecksums
# { 'linux-arm64': '9c7b6d0a74c68150cc21194f9961e09b1ea5c67200be0ac8ca35cb9d865a007c' }

sha256sum freebuff-linux-arm64.tar.gz   # harus sama persis
```

Kalau tidak cocok: hapus file, jangan jalan. Tidak ada masalah di sini —
launcher upstream Hobbyist/upstream sudah apkostume checksum, tapi tetap
verifikasi karena Anda download manual di jalur yang tidak diverifikasi.

> Cara lain: `glibc-runner` juga punya mode verifikasi checksum? Tidak — checksum
> upstream. Yang bisa diandalkan: hash dari `package.json` plus `npm audit signatures`.

### 3.3 Ekstrak

```bash
tar -xzf freebuff-linux-arm64.tar.gz
rm freebuff-linux-arm64.tar.gz
ls -la        # freebuff, tree-sitter.wasm, dll
```

Perhatikan file **`*.wasm`** atau aset lain di samping binary — jangan dibuang,
CLI sering membutuhkannya di runtime. Untuk freebuff ada `tree-sitter.wasm`.

### 3.4 Tes dulu sebelum commit

```bash
file ./freebuff                              # konfirmasi ELF arm64, interpreter glibc
./freebuff --version                         # expect: cannot execute (masih expected)
glibc-runner ./freebuff --version            # expect: nomor versi sungguhan
```

Kalau langkah terakhir masih gagal, Section 5.

### 3.5 Bikin wrapper supaya `nama-perintah` jalan biasa

Jangan selalu mengetik `grun ./binary` tiap kali — user harus ingat. Ganti
symlink di PATH dengan wrapper script.

Hapus symlink lama dari npm dulu:

```bash
npm uninstall -g freebuff     # bersihkan symlink npm
```

Lalu tulis wrapper:

```bash
mkdir -p ~/.local/bin
cat > ~/.local/bin/freebuff <<'EOF'
#!/data/data/com.termux/files/usr/bin/bash
# freebuff launcher for Termux.
# The upstream npm package only ships linux/darwin/win32 targets, but its
# binary is a glibc-linked ELF. Termux is Android/bionic, so we run it through
# glibc-runner, which supplies a glibc userland on the native Android kernel.
set -euo pipefail

FREEBUFF_BIN="$HOME/.local/share/freebuff/freebuff"

if [ ! -x "$FREEBUFF_BIN" ]; then
  echo "freebuff: binary not found at $FREEBUFF_BIN" >&2
  exit 1
fi

exec glibc-runner "$FREEBUFF_BIN" "$@"
EOF
chmod +x ~/.local/bin/freebuff
```

`set -euo pipefail` + check `! -x` itu penting: kalau binary hilang saat
upgrade, user dapat pesan jelas, bukan "command not found" yang membingungkan.

`exec` di baris terakhir penting — mengganti PID, jadi `Ctrl-C` dan signal
propagate dengan benar ke freebuff, bukan ke wrapper.

### 3.6 Verifikasi end-to-end

```bash
cd ~
freebuff --version     # harus cetak versi, tanpa grun
freebuff --help        # harus print usage lengkap
```

---

## 4. Kalau proyeknya menyediakan `-glibc` di Termux

Sebelum download manual, cek dulu apakah Termux sudah punya paket resminya:

```bash
apt-cache search glibc | grep -i "<namapaket>"
apt-cache policy <namapaket>-glibc
```

Termux-Pacman meng-compile ratusan paket glibc (nodejs, python, git, docker,
ffmpeg, imagemagick, dll). Kalau ada, **itu selalu lebih baik** daripada cara
manual — sudah di-patch, di-test, ada di `dpkg`, dan bisa di-update bareng
sistem.

```bash
pkg install python-glibc nodejs-glibc
```

Selesaikan daftarnya di `pkg search` atau
<https://termux-pacman.dev/>. Daftar di `~/.local/share/freebuff` adalah
fallback kalau tidak ada paketnya.

---

## 5. Troubleshooting

### "cannot execute: required file not found"

Linker ELF tidak ditemukan — normal, itu sebabnya kita pakai glibc-runner.
Kalau muncul **setelah** dibungkus `grun`, berarti `grun` juga gagal; cek
Section 5.2.

### glibc-runner: "incorrect flag or binary to run"

Anda membungkus file yang bukan ELF — biasanya skrip JS/Python/shell, bukan binary.
`grun` hanya bisa menjalankan ELF glibc.

```bash
file ./target       # harus "ELF 64-bit ... for GNU/Linux"
head -c 100 ./target | cat -v | head -2   # skrip = lihat shebang #!
```

Contoh nyata: `grun freebuff` gagal karena `freebuff` di PATH saat itu symlink ke
`index.js` (JavaScript) dari paket npm. Bukan `grun` yang salah, tapi yang
dibungkus adalah file yang salah. Download ELF aslinya (Section 3).

### glibc-runner: "shell is not running"

Anda panggil `grun --info` di luar shell glibc. `grun --info` cuma valid dari
dalam `grun -s`.

### Error `GLIBC_2.XX not found`

Butuh versi glibc yang lebih baru dari yang terpasang. Upgrade:

```bash
pkg upgrade glibc-runner
# atau cek versi loader yang Anda punya:
$PREFIX/glibc/lib/ld-linux-aarch64.so.1 --version
```

Kalau memang lebih baru dari yang ada, opsinya: upgrade Termux dari F-Droid/
GitHub, atau build dari sumber (repo: termux-pacman/glibc-packages).

### Missing library (`libfoo.so.1`)

```bash
grun --findlib ./binary       # cari di mana lib-nya
```

Kalau memang tidak ada di $PREFIX/glibc, cari paketnya:

```bash
apt-cache search foo | grep glibc
pkg install libfoo-glibc
```

### DNS / network tidak jalan di dalam glibc-runner

`resolv.conf` Termux ada di root Termux, sedangkan root Android punya `/etc/resolv.conf` sendiri yang read-only,
sedangkan glibc program mungkin cari `/etc/resolv.conf` di root Android yang
readonly. Gejalanya: download GitHub-nya timeout padahal `curl` di Termux jalan.

```bash
grun -s 'cat /etc/resolv.conf'    # lihat apa yang dibaca glibc
```

Kalau kosong/salah, arahkan manual:

```bash
grun -s 'DNS=1.1.1.1 ./binary'
```

### CTRL-C tidak membatalkan operasi

Wrapper tidak pakai `exec`, jadi signal tidak propagate. Pastikan baris terakhir
wrapper adalah `exec ... "$@"`.

### Binary jalan tapi lambat parah

Ini biaya glibc-runner. Kalau ini masalah, pertimbangkan proot-distro dengan
distro glibc — lebih lambat lagi, tapi userland lengkap (Section 6).

---

## 6. Opsi lain, dan kapan memakainya

| Teknologi | Cara kerja | Kapan pakai |
|---|---|---|
| **glibc-runner** | Exec ELF glibc dengan glibc Termux | Default. Native, cepat, minim resource. |
| **Paket `-glibc` Termux-Pacman** | Sudah di-compile & patch | Selalu coba ini duluan (Section 4). |
| **proot-distro Debian** | Userland virtual via `ptrace` | Binary butuh `/etc` sistem, systemd, atau userland lengkap. |
| **patchelf + LD_PRELOAD** | Patch ELF: set-interpreter/rpath | Kalau hanya satu binary, tidak mau pasang apa pun. |
| **gcompat** | Shim glibc untuk musl | Shim tipis. Hampir pasti gagal untuk binary besar. |
| **QEMU user-mode** | Emulasi CPU | Hampir tidak pernah perlu; sangat lambat. |

### proot-distro (kalau memang butuh)

```bash
pkg install proot-distro
proot-distro install debian      # rootfs glibc penuh
proot-distro login debian
# di dalam:
apt install -y curl ca-certificates
npm install -g freebuff          # sekarang platform-nya `linux`, bukan `android`
```

Trade-off: rootfs ~500MB-1GB, dan `ptrace` overhead nyata. Tapi kalau programnya
butuh `systemd`, `apt` dengan repo penuh, atau `/etc/passwd` yang masuk akal,
ini satu-satunya jalan yang bersih.

### patchelf (kalau cuma satu binary)

```bash
pkg install patchelf
patchelf --set-interpreter $PREFIX/glibc/lib/ld-linux-aarch64.so.1 ./binary
./binary --version     # tanpa grun
```

Cepat, tapi perlu satu binary perPATH, dan hanya berhasil kalau binary-nya
tidak butuh struktur `/etc` atau `/usr/share` dari glibc. Untuk CLI besar
sering lebih ribet daripada `grun` yang cuma dibungkus.

### gcompat (di Alpine)

```bash
proot-distro login alpine
apk add gcompat
```

`gcompat` cuma mengimplementasikan sebagian kecil API glibc. Binaries kecil
yang cuma butuh `pthread_create`/`libstdc++` bisa jalan; CLI 134MB seperti
freebuff hampir pasti tidak. Tetap tahu ini ada, tapi jangan hope.

---

## 7. Manual update

Launcher upstream tidak jalan di Android, jadi **tidak ada auto-update**.
Upgrade manual:

```bash
# 1. cek versi terbaru + checksum
npm view freebuff version binaryChecksums

# 2. download versi baru ke direktori yang sama
cd ~/.local/share/freebuff
curl -sLO "https://codebuff.com/api/releases/download/<versi>/freebuff-linux-arm64.tar.gz"

# 3. WAJIB verifikasi sebelum extract
sha256sum freebuff-linux-arm64.tar.gz

# 4. backup binary lama, lalu extract
cp freebuff freebuff.bak
tar -xzf freebuff-linux-arm64.tar.gz && rm freebuff-linux-arm64.tar.gz

# 5. tes
freebuff --version
```

Kalau `--version` gagal, kembalikan:

```bash
mv freebuff.bak freebuff
```

Wrapper di Section 3.5 tidak perlu diubah antar versi — dia menunjuk ke path
yang sama.

---

## 8. Checklist sebelum menjalankan binary strangers

- [ ] Checksum SHA256 cocok dengan yang dipublikasikan (**wajib**)
- [ ] `file ./binary` → `ELF 64-bit ... aarch64 ... for GNU/Linux`
- [ ] Download dari URL resmi / repo publik, bukan mirror acak
- [ ] Binary berasal dari project yang aktif & bisa diaudit
- [ ] Sudah `pkg upgrade` Termux ke versi terbaru

Kalau satu saja gagal, berhenti.

---

## 9. TL;DR

```bash
# sekali saja
pkg install glibc-runner -y

# untuk CLI yang metadata-nya salah
npm pack <paket> && tar -xzf <paket>-*.tgz
grep -oE 'https?://[^"]*(download|release)[^"]*' package/*.js
curl -sLO "<download-url>"
sha256sum <file>            # cocokkan dgn binaryChecksums
tar -xzf <file>

# lalu
grun ./binary --version
```

Dan kalau `grun` bilang "incorrect flag or binary to run", cek `file` dulu —
kemungkinan besar yang Anda bungkus skrip, bukan ELF.