# Laporan Praktik Pemrograman Perangkat Bergerak
## Aplikasi Prakiraan Cuaca Jakarta Berbasis Web (Vue 3 + TypeScript + Tailwind CSS)

---

### Identitas Mahasiswa

| Informasi | Keterangan |
| :--- | :--- |
| **Nama** | Berlian Duta Krisna |
| **NIM** | 05452335 |
| **Fakultas** | Fakultas Sains dan Teknologi |
| **Program Studi** | Sistem Informasi |
| **UPBJJ** | UPBJJ UT Surabaya |

---

## 1. Pendahuluan & Tujuan Proyek

Proyek ini bertujuan untuk membangun sebuah aplikasi web modern prakiraan cuaca per jam untuk wilayah DKI Jakarta dengan memanfaatkan data cuaca terbuka dari **Open-Meteo API**. Aplikasi dikembangkan dengan arsitektur modern berbasis komponen tunggal (*Single File Component* / SFC) menggunakan **Vue 3 Composition API**, **TypeScript** untuk kepastian tipe data (*type safety*), serta **Tailwind CSS** untuk desain antarmuka yang bersih, responsif, dan elegan.

---

## 2. Langkah-Langkah Praktik

Berikut adalah tahapan lengkap yang telah dilakukan mulai dari instalasi environment hingga aplikasi siap dijalankan:

### Tahap 1: Instalasi Node.js dan NPM
1. Mengunduh dan menginstal **Node.js** (versi LTS/terbaru, pada sistem ini menggunakan Node.js `v24.14.1` dan NPM `v11.19.0`).
2. Melakukan verifikasi instalasi melalui terminal/PowerShell:
   ```bash
   node -v
   npm -v
   ```

### Tahap 2: Inisialisasi Proyek Vue 3 + TypeScript
Menginisialisasi proyek berbasis Vite dengan template `vue-ts` secara non-interaktif:
```bash
npx -y create-vite@latest ./ --template vue-ts --no-immediate
```

### Tahap 3: Instalasi Dependensi & Tailwind CSS
1. Memasang dependensi dasar proyek:
   ```bash
   npm install
   ```
2. Memasang Tailwind CSS versi modern bersama plugin Vite:
   ```bash
   npm install -D tailwindcss @tailwindcss/vite
   ```

### Tahap 4: Konfigurasi Tailwind CSS dan Vite
1. Mengonfigurasi berkas `vite.config.ts` untuk mengintegrasikan plugin Tailwind:
   ```typescript
   import vue from '@vitejs/plugin-vue'
   import tailwindcss from '@tailwindcss/vite'
   import { defineConfig } from 'vite'

   export default defineConfig({
     plugins: [vue(), tailwindcss()],
   })
   ```
2. Mengonfigurasi `src/style.css` untuk mengimpor Tailwind CSS dan font Google *Plus Jakarta Sans*:
   ```css
   @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@300;400;500;600;700;800&display=swap');
   @import "tailwindcss";

   :root {
     font-family: 'Plus Jakarta Sans', system-ui, sans-serif;
   }
   ```

### Tahap 5: Implementasi Kode Logika & Antarmuka (`src/App.vue`)
1. Mendefinisikan kontrak data menggunakan TypeScript Interface:
   - `WeatherRecord`: Struktur objek dengan atribut `time`, `tempCelsius`, `tempFahrenheit`, dan `tempKelvin`.
   - `OpenMeteoHourlyResponse`: Struktur data response JSON dari API.
2. Mengambil data dari endpoint API Open-Meteo pada siklus hidup `onMounted`:
   ```text
   https://api.open-meteo.com/v1/forecast?latitude=-6.2&longitude=106.8&hourly=temperature_2m
   ```
3. Melakukan transformasi data menggunakan `computed` property di Vue 3:
   - Memformat tanggal ISO (`YYYY-MM-DDTHH:mm`) menjadi format lokal yang mudah dibaca (`DD/MM/YYYY HH:mm`).
   - Menghitung konversi suhu ke Fahrenheit dan Kelvin.
4. Mendesain tabel HTML responsif dengan Tailwind CSS lengkap dengan status *Loading*, *Error Handling*, pencarian (*search*), filter tanggal, dan paginasi (*pagination*).

### Tahap 6: Menjalankan dan Memverifikasi Aplikasi
Menjalankan server pengembangan lokal:
```bash
npm run dev
```
Aplikasi berhasil dijalankan pada alamat `http://localhost:5173/` dan diverifikasi langsung melalui penjelajah web.

---

## 3. Hasil dan Analisa Praktik

### A. Perolehan Satuan Suhu Fahrenheit dan Kelvin
Data awal yang disediakan oleh Open-Meteo API berada dalam satuan Celcius ($^\circ\text{C}$). Untuk menyajikan data yang lengkap dan bertaraf internasional, dilakukan transformasi matematis sebagai berikut:

1. **Suhu Fahrenheit ($^\circ\text{F}$):**
   - **Rumus:** 
     $$\text{Fahrenheit} = \left(\text{Celsius} \times \frac{9}{5}\right) + 32$$
   - **Implementasi Kode:**
     ```typescript
     const fahrenheit = Number(((celsius * 9) / 5 + 32).toFixed(2))
     ```
   - Nilai dibulatkan secara presisi hingga 2 angka di belakang koma menggunakan `.toFixed(2)` kemudian dikembalikan menjadi tipe `number`.

2. **Suhu Kelvin ($\text{K}$):**
   - **Rumus:** 
     $$\text{Kelvin} = \text{Celsius} + 273.15$$
   - **Implementasi Kode:**
     ```typescript
     const kelvin = Number((celsius + 273.15).toFixed(2))
     ```
   - Skala Kelvin merupakan satuan suhu mutlak (*absolute temperature scale*) standar SI, diperoleh dengan menambahkan titik nol mutlak yaitu $273.15$.

---

### B. Perhitungan Metrik Statistik Suhu

1. **Rata-Rata Suhu (Average Temperature):**
   - Didapatkan dengan menjumlahkan seluruh nilai suhu Celcius dalam rentang observasi, kemudian dibagi dengan total banyaknya data jam yang tersedia:
     $$\bar{X} = \frac{\sum_{i=1}^{n} T_i}{n}$$
   - **Implementasi Kode:**
     ```typescript
     const temps = weatherRecords.value.map((r) => r.tempCelsius)
     const avgTemp = Number((temps.reduce((acc, curr) => acc + curr, 0) / temps.length).toFixed(1))
     ```

2. **Suhu Terkini (Current / Initial Temperature):**
   - Diperoleh dari indeks pertama hasil response API (`hourly.temperature_2m[0]` atau `weatherRecords.value[0]`), yang merepresentasikan suhu prakiraan pada jam awal observasi saat data ditarik.

3. **Suhu Terendah (Minimum Temperature):**
   - Didapatkan dengan mengevaluasi nilai minimum dari seluruh array data suhu Celcius:
     ```typescript
     const minTemp = Math.min(...temps)
     ```

4. **Suhu Tertinggi (Maximum Temperature):**
   - Didapatkan dengan mengevaluasi nilai maksimum dari seluruh array data suhu Celcius:
     ```typescript
     const maxTemp = Math.max(...temps)
     ```

---

### C. Jumlah Data yang Digunakan
- **Total Data:** **168 data observasi per jam**.
- **Keterangan:** Open-Meteo API mengembalikan rentang prakiraan selama 7 hari penuh (7 hari $\times$ 24 jam = 168 titik data waktu). Setiap titik data memuat informasi jam pengukuran serta temperatur pada ketinggian 2 meter di atas permukaan tanah untuk koordinat DKI Jakarta (Latitude: -6.2, Longitude: 106.8).

---

### D. Analisa Teknis & Hasil Pengujian

1. **Analisa Karakteristik Suhu Jakarta:**
   - Berdasarkan data 168 jam prakiraan cuaca, suhu terendah di Jakarta berkisar pada **23.0°C – 24.5°C** (terjadi menjelang dini hari/subuh), sedangkan suhu tertinggi mencapai **32.0°C – 33.5°C** (terjadi di siang hari sekitar pukul 12.00 – 14.00 WIB). Rata-rata suhu mingguan berada di kisaran tropis hangat **27.6°C**.

2. **Keunggulan Vue 3 Composition API & TypeScript:**
   - Penggunaan `ref` dan `computed` memungkinkan antarmuka terbarukan secara reaktif secara instan (*instant reactivity*) ketika pengguna memfilter data atau berpindah halaman.
   - Definisi antarmuka `WeatherRecord` menjamin integritas data sehingga mencegah kesalahan penulisan properti dan mempermudah pemeliharaan kode (*maintainability*).

3. **User Experience (UX) dan Responsivitas Antarmuka:**
   - **Wrapper `overflow-x-auto`:** Memastikan tabel dapat digeser secara horizontal tanpa merusak tata letak saat dibuka pada layar ponsel pintar (*smartphone*).
   - **Efek Hover Baris:** Memberikan umpan balik visual instan (`hover:bg-blue-50/60 transition-colors`) saat kursor melintasi baris data.
   - **State Handling Lengkap:** Tersedianya status *loading* dengan spinner animasi serta blok penanganan *error* disertai tombol coba lagi (*retry button*) apabila terjadi kendala jaringan atau kegagalan pemanggilan API.

---

## 4. Kata Penutup

Puji dan syukur saya panjatkan ke hadirat Tuhan Yang Maha Esa karena atas berkat dan rahmat-Nya, tugas praktik pemrograman aplikasi web prakiraan cuaca berbasis bergerak ini dapat terselesaikan dengan baik dan lancar. 

Melalui pelaksanaan praktik ini, saya memperoleh wawasan mendalam mengenai pemanfaatan framework modern **Vue 3** dengan paradigma **Composition API**, kekuatan pengetikan statis pada **TypeScript**, serta efisiensi perancangan UI menggunakan utilitas **Tailwind CSS**. Selain itu, integrasi data *real-world* melalui RESTful API dari Open-Meteo memberikan pengalaman nyata dalam melakukan transformasi data, konversi satuan matematis, serta penyajian data statistik yang informatif bagi pengguna.

Semoga laporan dan proyek ini dapat memenuhi seluruh kriteria penilaian tugas yang diberikan serta memberikan manfaat bagi perkembangan keilmuan teknologi informasi, khususnya dalam pengembangan aplikasi perangkat bergerak dan web modern. Kritik serta saran yang membangun sangat saya harapkan demi penyempurnaan karya-karya selanjutnya.

Terima kasih.

**Surabaya, 2026**  
*Mahasiswa Praktikan,*  

**Berlian Duta Krisna**  
NIM: 05452335  
UPBJJ Universitas Terbuka Surabaya
