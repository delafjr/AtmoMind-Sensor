# Suhu Detector V2

Sistem monitoring suhu, kelembapan, dan intensitas cahaya berbasis ESP32 dengan dashboard web. Data sensor dikirim ke Supabase setiap 5 menit (rata-rata), dan dapat dipantau secara remote melalui dashboard Flask.

Project ini dibuat untuk memenuhi tugas mata kuliah **Artificial Intelligence (AI)** dengan pendekatan pengumpulan data lingkungan.

## Versi Tersedia

Project ini menyediakan 2 konfigurasi hardware berbeda:

1. **DHT11 Version** (`sketch_DHT11/`) - Menggunakan sensor DHT11 untuk membaca suhu dan kelembapan
2. **Thermocouple Version** (`sketch_Thermocouple/sketch/`) - Menggunakan sensor Thermocouple MAX6675 untuk pembacaan suhu akurat (tanpa kelembapan)

## Stack

- **Hardware**: ESP32 + TFT ILI9341 + Touchscreen + LDR + (DHT11 atau Thermocouple)
- **Database**: Supabase (PostgreSQL)
- **Dashboard**: Python Flask
- **Protokol**: HTTP REST API

## Struktur Project

```
atmomind_hardware/
├── sketch_DHT11/
│   ├── sketch.ino          # Kode ESP32 dengan DHT11
│   └── secrets.h.example   # Template secrets
├── sketch_Thermocouple/
│   └── sketch/
│       ├── sketch.ino      # Kode ESP32 dengan MAX6675
│       └── secrets.h.example
├── dashboard.py            # Dashboard Flask
├── requirements.txt
├── .env.example                    
├── .gitignore
└── README.md
```

## Hardware

### Komponen Utama

- ESP32 Dev Board
- TFT LCD ILI9341 (320x240, Touchscreen)
- Modul LDR (intensitas cahaya)
- **Pilih salah satu sensor suhu:**
  - DHT11 (suhu & kelembapan)
  - Thermocouple MAX6675 (suhu akurat 0-1024°C)

### Wiring - Versi DHT11

| Komponen | Pin Komponen | Pin ESP32 |
|----------|-------------|-----------|
| TFT CS   | CS          | D5 (GPIO 5) |
| TFT DC   | DC          | D21 (GPIO 21) |
| TFT RST  | RST         | D22 (GPIO 22) |
| Touch CS | T_CS        | D15 (GPIO 15) |
| DHT11    | VCC         | 3V3       |
| DHT11    | DATA        | D27 (GPIO 27) |
| DHT11    | GND         | GND       |
| LDR      | VCC         | 3V3       |
| LDR      | AO          | D34 (GPIO 34) |
| LDR      | GND         | GND       |

### Wiring - Versi Thermocouple

| Komponen | Pin Komponen | Pin ESP32 |
|----------|-------------|-----------|
| TFT CS   | CS          | D5 (GPIO 5) |
| TFT DC   | DC          | D21 (GPIO 21) |
| TFT RST  | RST         | D22 (GPIO 22) |
| Touch CS | T_CS        | D15 (GPIO 15) |
| MAX6675  | VCC         | 3V3       |
| MAX6675  | SCK         | D26 (GPIO 26) |
| MAX6675  | CS          | D27 (GPIO 27) |
| MAX6675  | SO          | D25 (GPIO 25) |
| MAX6675  | GND         | GND       |
| LDR      | VCC         | 3V3       |
| LDR      | AO          | D34 (GPIO 34) |
| LDR      | GND         | GND       |

> Kondisi gelap/terang dihitung dari nilai AO dengan threshold `2400`.

## Setup

### 1. Supabase

Buat tabel di Supabase dengan SQL berikut:

```sql
create table sensor_data (
  id bigint generated always as identity primary key,
  created_at timestamptz default now(),
  suhu float,
  kelembapan float,
  cahaya float,
  kondisi text
);
```

Aktifkan Row Level Security dan tambahkan policy insert untuk `anon` role.

### 2. ESP32

Install library berikut di Arduino IDE:

**Untuk semua versi:**
- Adafruit GFX Library
- Adafruit ILI9341
- XPT2046_Touchscreen
- ArduinoJson
- Board: ESP32 Dev Module

**Tambahan untuk DHT11:**
- DHT sensor library (Adafruit)

**Tambahan untuk Thermocouple:**
- max6675 library

Pilih salah satu folder sesuai sensor yang digunakan, lalu buat file `secrets.h` dari template:

**Untuk DHT11** (`sketch_DHT11/secrets.h`):
```cpp
#define WIFI_SSID    "nama_wifi"
#define WIFI_PASS    "password_wifi"
#define SUPABASE_URL "Your_URL_Database"
#define SUPABASE_KEY "Your_key"
```

**Untuk Thermocouple** (`sketch_Thermocouple/sketch/secrets.h`):
```cpp
#define WIFI_SSID    "nama_wifi"
#define WIFI_PASS    "password_wifi"
#define SUPABASE_URL "Your_URL_Database"
#define SUPABASE_KEY "Your_key"
#define SUPABASE_TABLE "sensor_data"
```

Upload sketch yang sesuai ke ESP32.

### 3. Dashboard

Install dependencies:

```bash
py -m pip install flask requests python-dotenv
```

Buat file `.env` di root project:

```
SUPABASE_URL=Your_URL_Database
SUPABASE_KEY=Your_Key
```

Jalankan dashboard:

```bash
py dashboard.py
```

Buka browser: `http://localhost:5000/dashboard`

## Cara Kerja

ESP32 membaca sensor setiap 2 detik dan mengakumulasi nilainya. Setiap 150 tick (5 menit), rata-rata dihitung dan dikirim ke Supabase via HTTP POST. Dashboard Flask membaca data dari Supabase dan menampilkan grafik suhu, kelembapan, dan intensitas cahaya secara otomatis refresh tiap 10 detik.

```
ESP32 → (setiap 5 menit) → Supabase → (dibaca oleh) → Dashboard Flask
```

### Mode Tampilan

Sistem memiliki 2 mode tampilan yang dapat diakses dengan menyentuh layar:

1. **Mode 1 - Typography UI**: Menampilkan angka besar suhu dan kelembapan
2. **Mode 2 - Developer Dashboard**: Menampilkan status WiFi, database, dan sensor detail

Mode 2 akan otomatis kembali ke Mode 1 setelah 10 detik.

## Perbedaan Versi

| Fitur | DHT11 | Thermocouple |
|-------|-------|--------------|
| Suhu | ✅ 0-50°C | ✅ 0-1024°C |
| Kelembapan | ✅ 20-90% | ❌ Tidak ada |
| Akurasi Suhu | ±2°C | ±0.25°C |
| Response Time | ~2s | ~0.25s |
| Use Case | Ruangan biasa | Aplikasi panas tinggi |

## Catatan

- ESP32 hanya support WiFi 2.4GHz
- GPIO 34 adalah input-only, cocok untuk baca analog LDR
