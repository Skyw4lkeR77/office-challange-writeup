# Office Writeup

Challenge: **Office**  
Author: **lunashci**  
Flag: `CBC{b3w4r3_h1dd3n_m4cr0s}`

## Ringkasan

File utama challenge adalah `Game.xlsm`, yaitu workbook Excel macro-enabled. Spreadsheet-nya terlihat seperti game tebak gaji, tetapi data penting disembunyikan di sheet `Data`, tepat di kolom terakhir Excel (`XFD`) dan baris-baris besar dekat batas bawah worksheet.

Alur payload-nya berlapis:

1. Ambil string tersembunyi dari workbook.
2. Decode PowerShell yang dipecah menjadi banyak potongan Base64 UTF-16LE.
3. Decrypt payload AES-CBC dengan key yang berasal dari hostname atau username.
4. Load .NET assembly dari workbook.
5. Reverse fungsi hash custom pada assembly untuk mendapatkan registry value.
6. Registry value menjadi prefix flag, sedangkan PNG hasil decrypt berisi suffix flag.

## 1. Cek Struktur Workbook

File `.xlsm` pada dasarnya adalah ZIP. Isi workbook bisa dicek dengan:

```bash
python Game/parse_cells.py
```

Cell menarik yang ditemukan:

```text
Sheet2 / Data
XFD1048560..XFD1048563 -> blob Base64 untuk assembly.dll
XFD1048572             -> FB11FE0C146FAC
XFD1048573             -> encrypted PowerShell stage
XFD1048574             -> encrypted PowerShell stage
XFD1048576             -> A045A54E5737EF
```

Sheet `Data` sengaja menaruh payload di koordinat ekstrem agar tidak terlihat saat workbook dibuka normal.

## 2. Decode Stage PowerShell

Beberapa payload PowerShell disimpan sebagai array potongan Base64. Setelah digabung, payload didecode sebagai UTF-16LE.

Contoh bentuk payload:

```powershell
$Sd886 = @("JABUAFIAbwBlAEMAUgBiACAAIAA9", ...)
$IVVvPHr = [System.Text.Encoding]::Unicode.GetString(
    [Convert]::FromBase64String([string]::Join("", $Sd886))
)
Invoke-Expression $IVVvPHr
```

Stage berikutnya melakukan gzip decompress, lalu `Add-Type` untuk memuat class .NET bernama `XJJfQh0HMY`.

## 3. Reverse Hostname

Script PowerShell mengecek hostname dengan fungsi `LsXxaQ`:

```text
expectedHash = A045A54E5737EF
seed         = 0xDEADBEEF
```

Fungsi ini memproses tiap byte dengan rotasi bit, XOR seed, lalu update seed:

```python
b = ror8(ord(c), 3)
b ^= seed & 0xFF
b = rol8(b, 5)
seed = seed * 0x6C078965 + 0x12345678
```

Karena operasinya reversible, hash `A045A54E5737EF` bisa dibalik menjadi:

```text
WORK-PC
```

## 4. Reverse Username

Setelah hostname valid, payload membaca cell `XFD1048572`:

```text
FB11FE0C146FAC
```

Nilai ini adalah hasil `Pia2wRPUo4iX(hostname, 0xCAFEBEBE)`, jadi validasi hostname kedua juga cocok.

Untuk stage username, hash target dibentuk secara obfuscated:

```powershell
"{3}{1}{0}{2}{4}" -f "35BFC","E36E","2","FDD","8"
```

Hasil gabungannya:

```text
FDDE36E35BFC28
```

Membalik `Pia2wRPUo4iX` dengan seed/state yang benar menghasilkan username:

```text
Fischer
```

## 5. Decrypt .NET Assembly

Workbook menyimpan empat potongan Base64 di:

```text
XFD1048560
XFD1048561
XFD1048562
XFD1048563
```

Potongan ini digabung lalu didecrypt dengan AES-CBC:

```text
key = SHA256("WORK-PC")
iv  = 16 null bytes
```

Plaintext hasil decrypt adalah PE/.NET assembly valid, disimpan sebagai:

```text
assembly.dll
```

Assembly ini memiliki type:

```text
nksCTGRr
```

Method penting:

```text
emcbDe4wAa -> main logic
REcPQ3X    -> custom hash
fsGiWrV    -> XOR stream decrypt
xSVe3t0nj  -> byte array compare
```

## 6. Analisis DLL

Method `emcbDe4wAa` mengambil:

```text
hostname = WORK-PC
username = Fischer
marker   = CBC2026
```

Lalu ia membentuk string:

```text
WORK-PC:Fischer:CBC2026:<registry_value>
```

String ini di-hash memakai `REcPQ3X` dan dibandingkan dengan `target_hash.bin`:

```text
9b43ab9b696f18d3dd37aceae160108515c88f2be8e24b62221380d77ab97c4bb6fcca76
```

Target hash panjangnya 36 byte, atau 9 block x 4 byte.

## 7. Membalik REcPQ3X

`REcPQ3X` memproses input per 4 byte little-endian:

```python
mixed = rol32(((block ^ jd) + l_wc) & 0xFFFFFFFF, 11)
jd = mixed
```

Dengan:

```text
l_wc = uint32_le("WORK")
jd   = uint32_le("Fisc")
```

Karena operasi ini reversible, setiap block target bisa dibalik:

```python
block = ((ror32(mixed, 11) - l_wc) & 0xFFFFFFFF) ^ jd
jd = mixed
```

Hasil pembalikan `target_hash.bin`:

```text
WORK-PC:Fischer:CBC2026:CBC{b3w4r3\x00\x00
```

Jadi nilai registry yang dicari adalah:

```text
CBC{b3w4r3
```

Ini adalah bagian pertama flag.

## 8. Decrypt PNG

Setelah registry value benar, DLL menghitung:

```python
reg_hash = REcPQ3X(b"CBC{b3w4r3", l_wc, jd)
seed = uint32_le(reg_hash[:4])
```

Seed tersebut dipakai oleh `fsGiWrV` untuk decrypt blob:

```python
for i in range(len(data)):
    data[i] ^= seed & 0xFF
    seed = (seed * 0x6C078965 + 0x12345678) & 0xFFFFFFFF
```

Hasil decrypt adalah PNG valid. Saat dibuka, gambar menampilkan:

```text
_h1dd3n_m4cr0s}
```

## 9. Gabungkan Flag

Bagian pertama berasal dari registry value:

```text
CBC{b3w4r3
```

Bagian kedua berasal dari PNG:

```text
_h1dd3n_m4cr0s}
```

Maka flag final:

```text
CBC{b3w4r3_h1dd3n_m4cr0s}
```

## Solver

Solver final ada di:

```text
solve_final.py
```

Jalankan:

```bash
python Game/solve_final.py
```

Output:

```text
Recovered registry value: CBC{b3w4r3
Recovered PNG: C:\projekan\hack-web2\Game\flag_recovered.png
FLAG: CBC{b3w4r3_h1dd3n_m4cr0s}
```
