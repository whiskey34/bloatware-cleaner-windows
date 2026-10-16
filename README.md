# DebloatWin

Alat baris perintah (CLI) untuk menghapus aplikasi bawaan yang tidak diperlukan (bloatware) di Windows secara terkendali. Hanya paket yang terdaftar di `catalog.json` yang bisa disentuh, setiap perubahan bisa dilihat dulu lewat mode *dry-run*, dan restore point dibuat otomatis sebelum eksekusi.

> [!WARNING]
> Alat ini mengubah sistem. Uji dulu di VM (misalnya Hyper-V dengan checkpoint), bukan langsung di mesin utama. Selalu jalankan `scan` dan `apply --dry-run` sebelum `apply`.

---

## Daftar Isi

- [Prasyarat](#prasyarat)
- [Build](#build)
- [Cara Menjalankan](#cara-menjalankan)
- [Perintah dan Opsi](#perintah-dan-opsi)
- [Membuat Satu File .exe](#membuat-satu-file-exe)
- [Pengaman Bawaan](#pengaman-bawaan)
- [Batasan](#batasan)
- [Keputusan Desain](#keputusan-desain)
- [Pemecahan Masalah](#pemecahan-masalah)

---

## Prasyarat

- Windows 10 atau Windows 11 (x64 atau ARM64)
- .NET 8 SDK

```cmd
winget install Microsoft.DotNet.SDK.8
```

- Terminal dengan hak **Administrator** (wajib, lihat di bawah)

## Build

Dari folder proyek:

```cmd
dotnet build -c Release
```

Hasil build ada di:

```
bin\Release\net8.0-windows\debloatwin.exe
```

## Cara Menjalankan

### 1. Buka terminal sebagai Administrator

Klik kanan **Command Prompt** atau **Terminal**, lalu pilih **Run as administrator**.

Ini wajib karena manifest aplikasi meminta hak admin. Jika dijalankan dari terminal biasa:

- `debloatwin.exe` akan memicu jendela UAC baru, dan jendela hasilnya langsung tertutup sehingga output tidak bisa dibaca.
- `dotnet run` akan gagal dengan pesan `requires elevation`.

### 2. Pindah ke folder hasil build

```cmd
cd /d <folder-proyek>\bin\Release\net8.0-windows
```

Contoh:

```cmd
cd /d D:\anti-bloatware\bin\Release\net8.0-windows
```

Atau tanpa `cd`, pakai path lengkap:

```cmd
D:\anti-bloatware\bin\Release\net8.0-windows\debloatwin.exe list
```

Alternatif lain, jalankan langsung dari folder proyek (terminal tetap harus Administrator):

```cmd
dotnet run -c Release -- list
```

Tanda `--` memisahkan argumen milik `dotnet` dari argumen milik program.

### 3. Jalankan dengan urutan yang aman

```cmd
debloatwin list
debloatwin scan
debloatwin apply --dry-run
debloatwin apply
```

Mulailah dari `list` lalu `scan`. Keduanya **tidak mengubah apa pun**. Periksa hasil `scan` dengan teliti: jika ada nama paket yang tidak cocok atau item yang tidak terdeteksi, jangan lanjut ke `apply` dulu.

Jika perlu mengembalikan perubahan:

```cmd
debloatwin undo
```

### (Opsional) Tambahkan ke PATH

Agar `debloatwin` bisa dipanggil dari folder mana pun, tambahkan folder hasil build ke variabel lingkungan `PATH`, lalu buka terminal baru.

## Perintah dan Opsi

### Perintah

| Perintah | Fungsi | Mengubah sistem? |
|---|---|---|
| `list` | Menampilkan isi katalog. | Tidak |
| `scan` | Mendeteksi item katalog yang terpasang dan menampilkan rencana. | Tidak |
| `apply --dry-run` | Menampilkan apa yang akan dilakukan `apply`. | Tidak |
| `apply` | Meminta konfirmasi (ketik `YES`), membuat restore point, lalu mengeksekusi. | **Ya** |
| `undo` | Mengembalikan pengaturan registry dan menampilkan daftar aplikasi yang perlu dipasang ulang. | Ya (registry) |

### Opsi

| Opsi | Fungsi |
|---|---|
| `--optional` | Menyertakan item berisiko sedang, seperti Xbox dan Game Bar. |
| `--only <id,id>` | Hanya memproses item tertentu, contoh: `--only clipchamp,tiktok`. |
| `--provisioned` | Juga menghapus paket *provisioned* (agar tidak terpasang untuk akun baru). Default: **mati**. |
| `--yes` | Melewati konfirmasi `YES`. Gunakan dengan hati-hati. |
| `--dry-run` | Hanya menampilkan rencana tanpa eksekusi. |

Contoh:

```cmd
debloatwin apply --only clipchamp,tiktok --dry-run
debloatwin apply --optional
```

## Membuat Satu File .exe

Untuk dibagikan sebagai satu file:

```cmd
dotnet publish -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true
```

Ganti `win-x64` dengan `win-arm64` untuk perangkat ARM. Karena `--self-contained false`, komputer tujuan tetap perlu .NET 8 Runtime.

Hasilnya ada di `bin\Release\net8.0-windows\win-x64\publish\`.

## Pengaman Bawaan

- **Allowlist.** Hanya paket yang ada di `catalog.json` yang bisa disentuh.
- **Daftar terlarang keras.** Microsoft Store, App Installer, Defender, Edge, paket `Microsoft.Windows.*`, runtime, dan komponen sistem sejenis tidak bisa dihapus. Jika ada yang menambahkannya ke katalog, tool menolak berjalan.
- **Paket `NonRemovable` dilewati.**
- **Registry terbatas.** Perubahan hanya di `HKCU\Software`, dan nilai sebelumnya dicatat untuk `undo`.
- **Validasi input.** Semua input ke PowerShell divalidasi ketat sebelum dipakai.
- **Restore point** dibuat otomatis sebelum `apply`.

## Batasan

- **`undo` hanya memulihkan registry sepenuhnya.** Untuk aplikasi, Windows menghapus foldernya, jadi `undo` hanya mencetak daftar aplikasi yang perlu dipasang ulang dari Microsoft Store.
- **Efek registry Start menu bergantung build Windows.** Item `start-promos` bersifat *best effort* dan belum diverifikasi di Windows 11 terbaru.
- **Nama paket belum semuanya terverifikasi.** Beberapa nama (misalnya Disney+, TikTok) belum dicek langsung. Item yang tidak terpasang tidak akan muncul di `scan` dan tidak menimbulkan error.
- **Tidak ada estimasi ruang disk.** Folder `WindowsApps` dibatasi ACL sehingga angkanya tidak bisa diandalkan.

## Keputusan Desain

- **Satu proyek.** Untuk versi pertama, semuanya ada dalam satu proyek dengan file terpisah agar mudah di-build. Bisa dipecah menjadi Core/Catalog/Cli nanti.
- **`--provisioned` default mati** agar perubahan hanya berdampak pada akun saat ini.
- **PowerShell dipanggil lewat `powershell.exe`**, bukan dengan meng-host `System.Management.Automation`. Modul Appx paling andal di Windows PowerShell 5.1, dan cara ini menghindari dependensi tambahan.

## Pemecahan Masalah

**Jendela langsung tertutup setelah dijalankan**
Terminal tidak dibuka sebagai Administrator. Tutup, lalu buka ulang dengan *Run as administrator*.

**`dotnet run` gagal dengan pesan `requires elevation`**
Penyebabnya sama: terminal tidak elevated.

**`'debloatwin' is not recognized...`**
Anda tidak berada di folder hasil build. Gunakan `cd /d` ke folder tersebut, pakai path lengkap, atau tambahkan foldernya ke `PATH`.

**Item yang diharapkan tidak muncul di `scan`**
Aplikasi tersebut mungkin tidak terpasang, atau nama paketnya di katalog berbeda dengan yang ada di sistem. Cek nama sebenarnya dengan:

```powershell
Get-AppxPackage | Select-Object Name, PackageFullName
```

Lalu laporkan lewat Issues.

---

## Kontribusi

Laporan bug, terutama nama paket yang tidak cocok atau perilaku berbeda di build Windows tertentu, sangat diterima. Sertakan output `debloatwin scan` dan versi Windows (`winver`) saat membuat issue.
