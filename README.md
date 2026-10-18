# VLSI Tools Installation Guide --- Ubuntu on WSL

Panduan instalasi dan konfigurasi tool VLSI/IC design pada **Ubuntu
melalui Windows Subsystem for Linux (WSL)**. Panduan ini merangkum
proses yang dilakukan, mulai dari menyiapkan lingkungan Linux, memasang
Nix dan OpenLane 2, hingga menghubungkan PDK SkyWater SKY130 dengan
Magic dan Xschem.

> **Status hasil pengujian pada setup ini**
>
> -   OpenLane 2: smoke test berhasil mencapai `78/78`.
> -   SKY130A dan SKY130B: direktori PDK tersedia melalui Volare.
> -   Xschem: konfigurasi SKY130A berhasil dimuat; log menampilkan
>     `SKYWATER_MODELS` dan `SKYWATER_STDCELLS`.
> -   Magic: executable dan file `sky130A.magicrc` tersedia; tetap
>     verifikasi pesan startup untuk memastikan teknologi berhasil
>     dimuat.
> -   Icarus Verilog: terpasang dan diuji dengan testbench yang
>     menghasilkan VCD.
> -   Netgen dan IRSIM: executable berhasil dijalankan sebelumnya.
> -   KLayout: file konfigurasi teknologinya ada di direktori PDK;
>     instalasi aplikasi mandiri perlu diverifikasi terpisah.

------------------------------------------------------------------------

## Daftar Isi

1.  [Arsitektur lingkungan](#1-arsitektur-lingkungan)
2.  [Persiapan WSL dan Ubuntu](#2-persiapan-wsl-dan-ubuntu)
3.  [Instalasi Nix](#3-instalasi-nix)
4.  [Clone dan jalankan OpenLane 2](#4-clone-dan-jalankan-openlane-2)
5.  [Smoke test OpenLane 2](#5-smoke-test-openlane-2)
6.  [Verifikasi PDK SKY130](#6-verifikasi-pdk-sky130)
7.  [Konfigurasi PDK di `.bashrc`](#7-konfigurasi-pdk-di-bashrc)
8.  [Menjalankan Magic dan Xschem dengan
    SKY130A](#8-menjalankan-magic-dan-xschem-dengan-sky130a)
9.  [Netgen dan IRSIM](#9-netgen-dan-irsim)
10. [Icarus Verilog dan Surfer](#10-icarus-verilog-dan-surfer)
11. [KLayout](#11-klayout)
12. [Troubleshooting](#12-troubleshooting)
13. [Checklist akhir](#13-checklist-akhir)

------------------------------------------------------------------------

## 1. Arsitektur lingkungan

Setup ini menggunakan Windows sebagai host dan Ubuntu sebagai lingkungan
Linux melalui WSL.

  Tool                Kegunaan
  ------------------- -------------------------------------------------------------
  Nix                 Mengelola environment dan dependensi OpenLane 2
  OpenLane 2          Otomatisasi alur desain digital dari RTL hingga layout
  SKY130A / SKY130B   Process Design Kit (PDK) dari SkyWater
  Xschem              Menggambar skematik dan menghasilkan netlist
  Magic               Layout IC dan pemeriksaan DRC
  Netgen              Perbandingan netlist, termasuk alur LVS
  Ngspice             Simulasi rangkaian SPICE
  IRSIM               Simulasi logika digital berbasis switch-level
  KLayout             Melihat dan memeriksa layout
  Icarus Verilog      Kompilasi dan simulasi Verilog
  Surfer              Melihat waveform VCD, termasuk dari simulasi Icarus Verilog

Sebagian tool tersedia melalui instalasi sistem, sebagian merupakan
dependensi atau integrasi PDK, dan OpenLane 2 dijalankan dalam
environment Nix. Karena itu, keberadaan file konfigurasi PDK tidak
selalu berarti executable aplikasinya tersedia pada `PATH`.

## 2. Persiapan WSL dan Ubuntu

Buka PowerShell di Windows untuk memeriksa WSL:

``` powershell
wsl --status
wsl --list --verbose
```

Jika WSL perlu diperbarui:

``` powershell
wsl --update
```

Jika distro mengalami masalah atau perlu dimulai ulang:

``` powershell
wsl --shutdown
```

Kemudian buka kembali Ubuntu. Periksa versi sistem dan ruang memori:

``` bash
lsb_release -a
free -h
```

Pada proses setup ini, lingkungan yang digunakan adalah Ubuntu 26.04.1
LTS di WSL. Versi paket yang tersedia bisa berbeda dari tutorial yang
dibuat untuk Ubuntu versi sebelumnya.

### Catatan PATH Windows

Jika WSL menampilkan peringatan seperti:

``` text
wsl: Failed to translate 'E:\Program Files\bin'
```

periksa apakah entri Windows tersebut masih valid. Jangan menghapus
entri PATH secara sembarangan; koreksi hanya entri yang memang sudah
tidak ada atau tidak diperlukan. Setelah mengubah PATH Windows, tutup
dan buka kembali WSL, atau jalankan `wsl --shutdown` dari PowerShell.

## 3. Instalasi Nix

OpenLane 2 pada tutorial ini menggunakan Nix. Jangan memasang Nix
melalui APT untuk mengikuti setup ini; gunakan installer resmi
Determinate Systems.

Pasang prasyarat:

``` bash
sudo apt update
sudo apt install -y curl
```

Jalankan installer resmi:

``` bash
curl --proto '=https' --tlsv1.2 -sSf \
  -L https://install.determinate.systems/nix | sh -s -- install
```

Ikuti instruksi installer. Jika diminta membuka ulang shell atau
menjalankan perintah untuk memuat environment, ikuti petunjuk yang
ditampilkan.

Verifikasi:

``` bash
nix --version
```

Contoh hasil pada setup ini menunjukkan Nix versi `2.35.3` yang dikelola
oleh Determinate Systems. Versi aktual dapat berubah.

### Cachix (opsional)

Cache binary dapat membantu mengurangi kompilasi dependensi jika
repository OpenLane menggunakan cache tersebut. Cachix bukan pengganti
Nix dan tidak wajib dikonfigurasi jika environment sudah berjalan.

Ikuti instruksi repository OpenLane 2 untuk cache yang direkomendasikan.
Jangan menyalin konfigurasi `trusted-users` dari sumber yang tidak
tepercaya.

## 4. Clone dan jalankan OpenLane 2

Clone repository resmi OpenLane 2:

``` bash
cd ~
git clone https://github.com/efabless/openlane2.git
cd ~/openlane2
```

Jika folder repository sudah ada, jangan clone ulang; cukup masuk ke
foldernya:

``` bash
cd ~/openlane2
```

Jalankan environment pengembangan:

``` bash
nix-shell
```

Tunggu hingga proses build selesai. Pertama kali menjalankan perintah
ini dapat memerlukan waktu lama karena Nix mungkin mengunduh atau
mengompilasi dependensi, termasuk KLayout dan library lainnya.

Jika prompt berubah menjadi bentuk seperti berikut, berarti shell Nix
aktif:

``` text
[nix-shell:~/openlane2]$
```

Periksa OpenLane dari dalam environment tersebut:

``` bash
openlane --help
```

Keluar dari environment Nix:

``` bash
exit
```

> **Penting:** pada setup ini, `openlane` tersedia di dalam `nix-shell`.
> Jika `command -v openlane` tidak menghasilkan apa pun pada terminal
> Ubuntu biasa, itu belum berarti instalasinya gagal.

## 5. Smoke test OpenLane 2

Masuk ke environment OpenLane:

``` bash
cd ~/openlane2
nix-shell
```

Jalankan smoke test dari dalam `nix-shell`:

``` bash
openlane --log-level ERROR --condensed --show-progress-bar --smoke-test
```

Pada setup yang dirangkum di sini, proses selesai pada:

``` text
Classic - Stage 78 - Report Manufacturability
78/78
```

Smoke test tersebut berhasil dijalankan hingga tahap terakhir. Smoke
test juga dapat memicu pengunduhan PDK yang diperlukan oleh flow.

Keluar setelah selesai:

``` bash
exit
```

## 6. Verifikasi PDK SKY130

PDK pada setup ini dikelola melalui Volare. Folder PDK ditemukan pada
struktur berikut:

``` text
~/.volare/volare/sky130/versions/<versi-pdk>/sky130A
~/.volare/volare/sky130/versions/<versi-pdk>/sky130B
```

Cari direktori PDK yang tersedia:

``` bash
find ~/.volare -type d \( -name "sky130A" -o -name "sky130B" \) 2>/dev/null
```

Periksa direktori teknologi SKY130A:

``` bash
find ~/.volare -type d -path "*/sky130A/libs.tech" 2>/dev/null
```

Pada setup ini, versi PDK yang ditemukan berada di bawah:

``` text
~/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/
```

Contoh struktur penting:

``` text
sky130A/
├── libs.ref/
└── libs.tech/
    ├── magic/
    ├── netgen/
    ├── ngspice/
    ├── openlane/
    └── xschem/
```

Periksa isi `libs.tech`:

``` bash
PDK_VERSION_DIR="$HOME/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af"

ls "$PDK_VERSION_DIR/sky130A/libs.tech"
ls "$PDK_VERSION_DIR/sky130A/libs.tech/magic"
ls "$PDK_VERSION_DIR/sky130A/libs.tech/xschem"
```

> **Catatan:** ID versi di atas khusus untuk PDK yang ditemukan pada
> setup ini. Jika Volare mengunduh versi lain, gunakan path hasil
> pencarian aktual, bukan menyalin ID tersebut secara membabi buta.

## 7. Konfigurasi PDK di `.bashrc`

Agar terminal mengetahui lokasi PDK, atur variabel `PDK_ROOT` dan `PDK`.
Untuk setup yang ditemukan dalam tutorial ini, edit file:

``` bash
nano ~/.bashrc
```

Tambahkan konfigurasi berikut. Ganti direktori versi jika hasil
pencarian PDK di komputermu berbeda.

``` bash
# SKY130 PDK managed by Volare
export PDK_ROOT="$HOME/.volare/volare/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af"
export PDK="sky130A"
```

Simpan dengan `Ctrl+O`, tekan `Enter`, lalu keluar dengan `Ctrl+X`. Muat
ulang:

``` bash
source ~/.bashrc
```

Verifikasi:

``` bash
echo "PDK_ROOT=$PDK_ROOT"
echo "PDK=$PDK"
ls "$PDK_ROOT/$PDK/libs.tech"
```

Hasil yang diharapkan adalah path PDK dan nama `sky130A`, disertai
direktori teknologi seperti `magic`, `ngspice`, dan `xschem`.

### Mengganti PDK aktif

Untuk menggunakan SKY130B pada terminal saat ini:

``` bash
export PDK="sky130B"
```

Untuk kembali ke SKY130A:

``` bash
export PDK="sky130A"
```

Pastikan direktori yang dipilih memang tersedia di bawah `$PDK_ROOT`.

### Catatan pemeliharaan

Path di atas mencakup ID versi PDK. Jika Volare memperbarui PDK, path
versi dapat berubah. Setelah memperbarui PDK, cari kembali direktori
`sky130A` dan sesuaikan `PDK_ROOT` bila diperlukan.

## 8. Menjalankan Magic dan Xschem dengan SKY130A

### 8.1 Periksa executable dan file konfigurasi

``` bash
command -v magic
command -v xschem

ls "$PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc"
ls "$PDK_ROOT/$PDK/libs.tech/xschem/xschemrc"
```

Pada setup ini, executable ditemukan di:

``` text
/usr/local/bin/magic
/usr/local/bin/xschem
```

### 8.2 Jalankan Magic dengan konfigurasi SKY130A

``` bash
magic -rcfile "$PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc"
```

Magic akan membuka jendela layout. Perhatikan pesan startup di terminal
untuk memastikan file konfigurasi dibaca tanpa error.

### 8.3 Jalankan Xschem dengan konfigurasi SKY130A

``` bash
xschem --rcfile "$PDK_ROOT/$PDK/libs.tech/xschem/xschemrc"
```

Pada setup ini, log Xschem menampilkan informasi seperti:

``` text
Sourcing .../sky130A/libs.tech/xschem/xschemrc init file
open_pdks installation: using .../versions/...
SKYWATER_MODELS: .../sky130A/libs.tech/combined
SKYWATER_STDCELLS: .../sky130A/libs.ref/sky130_fd_sc_hd/spice
```

Log tersebut menunjukkan bahwa konfigurasi Xschem untuk SKY130A telah
dimuat. Untuk memastikan library simbol muncul, periksa panel library di
dalam aplikasi.

### 8.4 Alias opsional agar lebih praktis

Jika kedua aplikasi bisa dijalankan dengan perintah eksplisit di atas,
alias dapat ditambahkan ke bagian bawah `~/.bashrc`:

``` bash
alias magic='magic -rcfile "$PDK_ROOT/$PDK/libs.tech/magic/$PDK.magicrc"'
alias xschem='xschem --rcfile "$PDK_ROOT/$PDK/libs.tech/xschem/xschemrc"'
```

Aktifkan:

``` bash
source ~/.bashrc
hash -r
```

Periksa apakah alias sudah aktif:

``` bash
type magic
type xschem
```

Jika alias aktif, `type` akan menampilkan definisi alias, bukan hanya
lokasi executable.

Sekarang jalankan:

``` bash
magic
xschem
```

> Alias hanya mempersingkat perintah. Alias tidak memasang tool dan
> tidak menjamin library simbol tersedia; tetap periksa log startup dan
> library di GUI.

### 8.5 Peringatan eksekusi Tcl di Xschem

Xschem dapat menampilkan dialog:

> Allow xschem to execute scripts embedded in schematics?

Ini adalah peringatan keamanan untuk skrip yang tertanam dalam skematik,
bukan bukti bahwa integrasi SKY130A gagal. Pilih **No** jika sumber
skematik belum tepercaya. Pilih **Yes** hanya jika kamu memercayai
sumber dan memahami skripnya. Jangan mengaktifkan eksekusi skrip secara
global hanya untuk menghilangkan dialog.

## 9. Netgen dan IRSIM

Netgen dan IRSIM telah diuji sebelumnya pada setup ini.

### Netgen

Periksa executable:

``` bash
command -v netgen
```

Jika tersedia, jalankan:

``` bash
netgen -batch
```

Pada pengujian sebelumnya, Netgen menampilkan console; peringatan
terkait perintah Tcl dapat muncul. Untuk pengujian LVS yang sebenarnya,
gunakan netlist layout dan skematik yang sesuai, beserta setup file
SKY130 yang benar.

### IRSIM

Periksa executable:

``` bash
command -v irsim
```

Jalankan:

``` bash
irsim
```

Jika muncul pesan seperti `No sim file specified`, itu berarti IRSIM
dibuka tanpa file simulasi. Pesan tersebut bukan pengujian simulasi yang
lengkap. Untuk simulasi, siapkan file netlist dan stimulus yang sesuai.

## 10. Icarus Verilog dan Surfer

Icarus Verilog dipasang melalui APT dan diuji dengan testbench
sederhana. Surfer digunakan untuk melihat waveform VCD.

### 10.1 Instal Icarus Verilog

``` bash
sudo apt update
sudo apt install -y iverilog
```

Verifikasi:

``` bash
iverilog -V
vvp -V
```

### 10.2 Buat proyek sederhana

``` bash
mkdir -p ~/open_source/projects/verilog
cd ~/open_source/projects/verilog
nano test.v
```

Contoh testbench:

``` verilog
`timescale 1ns/1ps

module test;
    reg a, b;
    wire y;

    assign y = a & b;

    initial begin
        $dumpfile("test.vcd");
        $dumpvars(0, test);

        a = 0; b = 0;
        #10 a = 0; b = 1;
        #10 a = 1; b = 0;
        #10 a = 1; b = 1;
        #10 $finish;
    end
endmodule
```

Simpan file, lalu kompilasi dan jalankan:

``` bash
iverilog -o test.out test.v
vvp test.out
```

Output yang diharapkan mencakup pesan bahwa `test.vcd` dibuka dan
simulasi selesai. Periksa file hasil:

``` bash
ls -l test.v test.out test.vcd
```

### 10.3 Lihat waveform di Surfer

Buka file `test.vcd` menggunakan Surfer atau ekstensi Surfer di VS Code.
Pilih scope `test`, lalu tambahkan sinyal `a`, `b`, dan `y` ke waveform.
Jika panel Variables kosong, pastikan scope sudah dipilih dan sinyal
ditambahkan ke tampilan waveform.

## 11. KLayout

PDK SKY130A yang ditemukan memiliki direktori teknologi KLayout:

``` bash
ls "$PDK_ROOT/$PDK/libs.tech/klayout"
```

Namun, adanya direktori tersebut tidak otomatis membuktikan aplikasi
KLayout dapat dijalankan dari terminal.

Pada proses instalasi ini, paket `.deb` yang dicoba sebelumnya mengalami
ketidakcocokan dependensi Python (paket memerlukan `libpython3.12`,
sedangkan lingkungan Ubuntu yang digunakan lebih baru). Jangan memaksa
instalasi paket untuk versi Ubuntu lain tanpa memeriksa dependensinya.

Periksa apakah executable tersedia:

``` bash
command -v klayout
klayout -v
```

Jika tidak ditemukan, pilih paket KLayout yang kompatibel dengan versi
Ubuntu yang digunakan atau ikuti metode instalasi resmi KLayout. Setelah
terpasang, verifikasi bahwa aplikasi dapat dibuka dan konfigurasi
teknologi SKY130 tersedia.

## 12. Troubleshooting

### `openlane: command not found`

Masuk ke repository lalu jalankan Nix shell:

``` bash
cd ~/openlane2
nix-shell
openlane --help
```

OpenLane pada setup ini dijalankan dari environment Nix.

### `nix-shell` lama saat pertama kali dijalankan

Nix dapat mengunduh atau mengompilasi dependensi. Periksa aktivitas CPU
dan memori pada terminal lain:

``` bash
free -h
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head
```

Jangan langsung menghentikan proses hanya karena kompilasi berlangsung
lama.

### WSL mengalami OOM atau berhenti mendadak

Kompilasi paket besar dapat menghabiskan RAM. Periksa log kernel jika
akses tersedia:

``` bash
sudo dmesg -T | grep -iE "out of memory|oom|killed process" | tail -20
```

Jika WSL tidak merespons, dari PowerShell:

``` powershell
wsl --shutdown
```

Buka kembali Ubuntu setelah itu. Sesuaikan alokasi memori/swap WSL jika
masalah berulang, dengan mempertimbangkan RAM fisik komputer.

### `PDK_ROOT` kosong

Muat ulang `.bashrc`:

``` bash
source ~/.bashrc
echo "$PDK_ROOT"
echo "$PDK"
```

Jika masih kosong, periksa apakah variabel telah ditambahkan dan
disimpan dengan benar:

``` bash
tail -n 30 ~/.bashrc
```

### Direktori `sky130A` tidak ditemukan

Cari ulang lokasi PDK:

``` bash
find ~/.volare -type d -name "sky130A" 2>/dev/null
```

Gunakan hasil pencarian aktual untuk menentukan direktori yang memuat
folder `sky130A` dan `sky130B`.

### Xschem terbuka tetapi library SKY130 tidak terlihat

Jalankan dengan file konfigurasi secara eksplisit:

``` bash
xschem --rcfile "$PDK_ROOT/$PDK/libs.tech/xschem/xschemrc"
```

Periksa log startup untuk `SKYWATER_STDCELLS` dan pastikan folder simbol
yang dirujuk benar-benar ada. Berhasil memuat `xschemrc` adalah langkah
penting, tetapi panel library tetap perlu diperiksa.

### Magic terbuka tetapi teknologi belum jelas

Jalankan dengan konfigurasi eksplisit:

``` bash
magic -rcfile "$PDK_ROOT/$PDK/libs.tech/magic/sky130A.magicrc"
```

Periksa output startup. Jangan menyimpulkan teknologi telah termuat
hanya dari tampilan jendela Magic.

### Peringatan skrip Xschem muncul berulang kali

Peringatan ini berhubungan dengan keamanan skrip tertanam. Pilih **No**
untuk sumber yang belum tepercaya. Dialog ini tidak dengan sendirinya
menunjukkan masalah PDK.

## 13. Checklist akhir

-   [ ] Ubuntu berjalan di WSL.
-   [ ] Nix terpasang dan `nix --version` berhasil.
-   [ ] Repository OpenLane 2 tersedia di `~/openlane2`.
-   [ ] `nix-shell` berhasil masuk ke environment OpenLane.
-   [ ] Smoke test OpenLane berhasil selesai.
-   [ ] PDK SKY130A dan/atau SKY130B ditemukan melalui Volare.
-   [ ] `PDK_ROOT` dan `PDK` benar setelah `source ~/.bashrc`.
-   [ ] Magic dapat dijalankan dengan `sky130A.magicrc` tanpa error
    konfigurasi.
-   [ ] Xschem dapat dijalankan dengan `xschemrc` SKY130A dan library
    simbol tersedia.
-   [ ] Netgen dan IRSIM dapat dijalankan.
-   [ ] Icarus Verilog dapat menghasilkan file VCD.
-   [ ] Surfer dapat menampilkan sinyal dari file VCD.
-   [ ] KLayout telah diverifikasi terpisah jika dibutuhkan.

------------------------------------------------------------------------

## Referensi

-   [OpenLane 2 --- GitHub](https://github.com/efabless/openlane2)
-   [OpenLane 2 Documentation](https://openlane2.readthedocs.io/)
-   [Determinate Nix
    Installer](https://github.com/DeterminateSystems/nix-installer)
-   [SkyWater SKY130 PDK](https://github.com/google/skywater-pdk)
-   [Volare --- PDK manager](https://github.com/efabless/volare)
-   [Magic VLSI](http://opencircuitdesign.com/magic/)
-   [Xschem](https://github.com/stefanschippers/xschem)
-   [Netgen](http://opencircuitdesign.com/netgen/)
-   [Icarus Verilog](https://github.com/steveicarus/iverilog)
-   [Surfer waveform viewer](https://surfer-project.org/)

## Catatan

Panduan ini mendokumentasikan hasil pada satu lingkungan WSL. Versi
Ubuntu, Nix, OpenLane, PDK, dan dependensi dapat berubah. Selalu
verifikasi versi dan path aktual sebelum menyalin konfigurasi. Jangan
mengunggah data pribadi, token, atau konfigurasi rahasia ke repository
publik.
