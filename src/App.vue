<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// ==========================================
// 1. Definisi Tipe & Interface (TypeScript)
// ==========================================

/**
 * Interface struktur objek data cuaca per jam yang dirender pada tabel
 */
export interface WeatherRecord {
  time: string           // Format: DD/MM/YYYY HH:mm
  tempCelsius: number    // Suhu dalam Celcius
  tempFahrenheit: number // Suhu dalam Fahrenheit ((C * 9/5) + 32)
  tempKelvin: number     // Suhu dalam Kelvin (C + 273.15)
}

/**
 * Interface struktur response dari Open-Meteo API
 */
interface OpenMeteoHourlyResponse {
  latitude: number
  longitude: number
  generationtime_ms: number
  utc_offset_seconds: number
  timezone: string
  timezone_abbreviation: string
  elevation: number
  hourly_units: {
    time: string
    temperature_2m: string
  }
  hourly: {
    time: string[]
    temperature_2m: number[]
  }
}

// ==========================================
// 2. Reactive State
// ==========================================

const API_ENDPOINT = 'https://api.open-meteo.com/v1/forecast?latitude=-6.2&longitude=106.8&hourly=temperature_2m'

const rawHourlyData = ref<{ time: string[]; temperature_2m: number[] } | null>(null)
const apiMetadata = ref<{ latitude: number; longitude: number; elevation: number; timezone: string } | null>(null)
const isLoading = ref<boolean>(true)
const errorMessage = ref<string | null>(null)
const lastUpdated = ref<string>('')

// Filter & Pagination state
const searchQuery = ref<string>('')
const selectedDateFilter = ref<string>('all')
const currentPage = ref<number>(1)
const itemsPerPage = ref<number>(12)

// ==========================================
// 3. Logika Transformasi Data & Helper
// ==========================================

/**
 * Mengubah string ISO format (misal: "2026-10-27T00:00") menjadi format "DD/MM/YYYY HH:mm"
 */
function formatIsoToReadable(isoString: string): string {
  try {
    const [datePart, timePart = '00:00'] = isoString.split('T')
    const [year, month, day] = datePart.split('-')
    if (year && month && day) {
      return `${day}/${month}/${year} ${timePart.slice(0, 5)}`
    }
  } catch {
    // fallback jika parsing gagal
  }
  return isoString
}

/**
 * Computed property untuk memetakan response array dari API menjadi array WeatherRecord
 */
const weatherRecords = computed<WeatherRecord[]>(() => {
  if (!rawHourlyData.value || !rawHourlyData.value.time) {
    return []
  }

  const { time, temperature_2m } = rawHourlyData.value

  return time.map((isoTime, index) => {
    const celsius = temperature_2m[index] ?? 0

    // Konversi Celcius ke Fahrenheit: (C * 9/5) + 32, dibulatkan 2 angka desimal
    const fahrenheit = Number(((celsius * 9) / 5 + 32).toFixed(2))

    // Konversi Celcius ke Kelvin: C + 273.15, dibulatkan 2 angka desimal
    const kelvin = Number((celsius + 273.15).toFixed(2))

    return {
      time: formatIsoToReadable(isoTime),
      tempCelsius: celsius,
      tempFahrenheit: fahrenheit,
      tempKelvin: kelvin,
    }
  })
})

/**
 * Daftar tanggal unik untuk filter berdasarkan hari
 */
const availableDates = computed<string[]>(() => {
  const dates = new Set<string>()
  weatherRecords.value.forEach((item) => {
    const dateOnly = item.time.split(' ')[0]
    if (dateOnly) dates.add(dateOnly)
  })
  return Array.from(dates)
})

/**
 * Filter data berdasarkan pencarian teks dan filter tanggal
 */
const filteredWeatherRecords = computed<WeatherRecord[]>(() => {
  return weatherRecords.value.filter((record) => {
    const matchesSearch =
      searchQuery.value === '' ||
      record.time.toLowerCase().includes(searchQuery.value.toLowerCase()) ||
      record.tempCelsius.toString().includes(searchQuery.value)

    const matchesDate =
      selectedDateFilter.value === 'all' ||
      record.time.startsWith(selectedDateFilter.value)

    return matchesSearch && matchesDate
  })
})

/**
 * Data untuk halaman aktif saat ini (Pagination)
 */
const paginatedRecords = computed<WeatherRecord[]>(() => {
  const start = (currentPage.value - 1) * itemsPerPage.value
  return filteredWeatherRecords.value.slice(start, start + itemsPerPage.value)
})

const totalPages = computed<number>(() => {
  return Math.ceil(filteredWeatherRecords.value.length / itemsPerPage.value) || 1
})

/**
 * Statistik ringkasan (Suhu Terendah, Tertinggi, Rata-rata)
 */
const summaryStats = computed(() => {
  if (weatherRecords.value.length === 0) return null

  const temps = weatherRecords.value.map((r) => r.tempCelsius)
  const minTemp = Math.min(...temps)
  const maxTemp = Math.max(...temps)
  const avgTemp = Number((temps.reduce((acc, curr) => acc + curr, 0) / temps.length).toFixed(1))
  const currentTemp = weatherRecords.value[0]?.tempCelsius ?? 0

  return {
    min: minTemp,
    max: maxTemp,
    avg: avgTemp,
    current: currentTemp,
  }
})

// ==========================================
// 4. Data Fetching dengan onMounted
// ==========================================

async function fetchWeatherData(): Promise<void> {
  isLoading.value = true
  errorMessage.value = null

  try {
    const response = await fetch(API_ENDPOINT)

    if (!response.ok) {
      throw new Error(`HTTP Error: ${response.status} ${response.statusText}`)
    }

    const data: OpenMeteoHourlyResponse = await response.json()

    if (!data.hourly || !data.hourly.time || !data.hourly.temperature_2m) {
      throw new Error('Format data API tidak sesuai ekspektasi.')
    }

    rawHourlyData.value = {
      time: data.hourly.time,
      temperature_2m: data.hourly.temperature_2m,
    }

    apiMetadata.value = {
      latitude: data.latitude,
      longitude: data.longitude,
      elevation: data.elevation,
      timezone: data.timezone || 'Asia/Jakarta',
    }

    const now = new Date()
    lastUpdated.value = now.toLocaleTimeString('id-ID', {
      hour: '2-digit',
      minute: '2-digit',
      second: '2-digit',
    })
    currentPage.value = 1
  } catch (err: unknown) {
    if (err instanceof Error) {
      errorMessage.value = err.message
    } else {
      errorMessage.value = 'Terjadi kesalahan saat memuat data prakiraan cuaca.'
    }
  } finally {
    isLoading.value = false
  }
}

function handlePageChange(newPage: number): void {
  if (newPage >= 1 && newPage <= totalPages.value) {
    currentPage.value = newPage
  }
}

onMounted(() => {
  fetchWeatherData()
})
</script>

<template>
  <div class="min-h-screen bg-slate-50 text-slate-800 antialiased py-8 px-4 sm:px-6 lg:px-8">
    <div class="max-w-6xl mx-auto space-y-6">

      <!-- ========================================== -->
      <!-- HEADER & INFORMASI LOKASI                  -->
      <!-- ========================================== -->
      <header class="bg-gradient-to-r from-blue-600 via-indigo-600 to-sky-600 rounded-2xl shadow-xl shadow-blue-500/10 text-white p-6 sm:p-8">
        <div class="flex flex-col md:flex-row md:items-center md:justify-between gap-4">
          <div class="space-y-2">
            <div class="inline-flex items-center gap-2 px-3 py-1 rounded-full bg-white/15 backdrop-blur-md text-xs font-medium text-sky-100 border border-white/20">
              <span class="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span>
              Live Data • Open-Meteo API
            </div>
            <h1 class="text-2xl sm:text-3xl font-extrabold tracking-tight">
              Prakiraan Cuaca Per Jam
            </h1>
            <p class="text-blue-100 text-sm sm:text-base flex items-center gap-2">
              <svg class="w-4 h-4 shrink-0 text-sky-200" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z" />
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 11a3 3 0 11-6 0 3 3 0 016 0z" />
              </svg>
              Wilayah DKI Jakarta &bull; Koordinat: 
              <span class="font-semibold text-white">Lat -6.2, Long 106.8</span>
              <span v-if="apiMetadata" class="hidden sm:inline text-sky-200">
                (Elevasi: {{ apiMetadata.elevation }}m)
              </span>
            </p>
          </div>

          <!-- Tombol Segarkan / Refresh -->
          <div class="flex items-center gap-3">
            <button
              id="refresh-weather-btn"
              @click="fetchWeatherData"
              :disabled="isLoading"
              class="inline-flex items-center gap-2 px-4 py-2.5 rounded-xl bg-white/10 hover:bg-white/20 active:scale-95 transition-all text-white text-sm font-semibold backdrop-blur-md border border-white/20 shadow-sm disabled:opacity-50 disabled:cursor-not-allowed"
              title="Segarkan data cuaca"
            >
              <svg
                class="w-4 h-4"
                :class="{ 'animate-spin': isLoading }"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
              >
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
              </svg>
              <span>{{ isLoading ? 'Memuat...' : 'Segarkan Data' }}</span>
            </button>
          </div>
        </div>

        <div v-if="lastUpdated" class="mt-4 pt-4 border-t border-white/15 text-xs text-sky-200 flex flex-wrap justify-between items-center gap-2">
          <span>Terakhir diperbarui: <strong class="text-white">{{ lastUpdated }} WIB</strong></span>
          <span>Zona Waktu: <strong class="text-white">{{ apiMetadata?.timezone || 'Asia/Jakarta' }}</strong></span>
        </div>
      </header>

      <!-- ========================================== -->
      <!-- STATISTIK RINGKASAN                        -->
      <!-- ========================================== -->
      <section v-if="summaryStats && !isLoading && !errorMessage" class="grid grid-cols-2 md:grid-cols-4 gap-4">
        <div class="bg-white rounded-xl p-4 shadow-sm border border-slate-200/80">
          <p class="text-xs font-medium text-slate-500 uppercase tracking-wider">Suhu Terkini</p>
          <div class="mt-1 flex items-baseline gap-1">
            <span class="text-2xl font-bold text-slate-900">{{ summaryStats.current }}</span>
            <span class="text-sm font-medium text-slate-500">°C</span>
          </div>
          <span class="inline-block mt-2 text-xs font-semibold px-2 py-0.5 rounded-full bg-blue-100 text-blue-700">
            Jam Pertama
          </span>
        </div>

        <div class="bg-white rounded-xl p-4 shadow-sm border border-slate-200/80">
          <p class="text-xs font-medium text-slate-500 uppercase tracking-wider">Suhu Terendah</p>
          <div class="mt-1 flex items-baseline gap-1">
            <span class="text-2xl font-bold text-sky-600">{{ summaryStats.min }}</span>
            <span class="text-sm font-medium text-slate-500">°C</span>
          </div>
          <span class="inline-block mt-2 text-xs font-semibold px-2 py-0.5 rounded-full bg-sky-100 text-sky-700">
            Minimum Periode
          </span>
        </div>

        <div class="bg-white rounded-xl p-4 shadow-sm border border-slate-200/80">
          <p class="text-xs font-medium text-slate-500 uppercase tracking-wider">Suhu Tertinggi</p>
          <div class="mt-1 flex items-baseline gap-1">
            <span class="text-2xl font-bold text-amber-600">{{ summaryStats.max }}</span>
            <span class="text-sm font-medium text-slate-500">°C</span>
          </div>
          <span class="inline-block mt-2 text-xs font-semibold px-2 py-0.5 rounded-full bg-amber-100 text-amber-700">
            Maksimum Periode
          </span>
        </div>

        <div class="bg-white rounded-xl p-4 shadow-sm border border-slate-200/80">
          <p class="text-xs font-medium text-slate-500 uppercase tracking-wider">Rata-Rata Suhu</p>
          <div class="mt-1 flex items-baseline gap-1">
            <span class="text-2xl font-bold text-indigo-600">{{ summaryStats.avg }}</span>
            <span class="text-sm font-medium text-slate-500">°C</span>
          </div>
          <span class="inline-block mt-2 text-xs font-semibold px-2 py-0.5 rounded-full bg-indigo-100 text-indigo-700">
            Dari {{ weatherRecords.length }} Data Jam
          </span>
        </div>
      </section>

      <!-- ========================================== -->
      <!-- KONTROL PENCARIAN & FILTER                 -->
      <!-- ========================================== -->
      <section v-if="!isLoading && !errorMessage" class="bg-white rounded-xl p-4 shadow-sm border border-slate-200/80 flex flex-col md:flex-row md:items-center justify-between gap-4">
        <div class="flex-1 flex flex-col sm:flex-row gap-3">
          <!-- Kotak Pencarian -->
          <div class="relative flex-1">
            <div class="absolute inset-y-0 left-0 pl-3 flex items-center pointer-events-none text-slate-400">
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z" />
              </svg>
            </div>
            <input
              id="search-input"
              type="text"
              v-model="searchQuery"
              placeholder="Cari waktu (contoh: 28/10, 14:00) atau suhu..."
              class="w-full pl-9 pr-4 py-2 text-sm bg-slate-50 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:bg-white transition-all"
            />
          </div>

          <!-- Filter Tanggal -->
          <div class="w-full sm:w-56">
            <select
              id="date-filter-select"
              v-model="selectedDateFilter"
              class="w-full px-3 py-2 text-sm bg-slate-50 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 focus:bg-white transition-all"
            >
              <option value="all">Semua Tanggal (7 Hari)</option>
              <option v-for="d in availableDates" :key="d" :value="d">
                Tanggal: {{ d }}
              </option>
            </select>
          </div>
        </div>

        <!-- Jumlah baris per halaman -->
        <div class="flex items-center gap-2 text-sm text-slate-500 shrink-0">
          <label for="per-page-select">Tampilkan:</label>
          <select
            id="per-page-select"
            v-model.number="itemsPerPage"
            class="px-2.5 py-1.5 text-sm bg-slate-50 border border-slate-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            <option :value="12">12 data</option>
            <option :value="24">24 data (1 Hari)</option>
            <option :value="48">48 data (2 Hari)</option>
            <option :value="168">Semua (168 data)</option>
          </select>
        </div>
      </section>

      <!-- ========================================== -->
      <!-- STATE: LOADING                             -->
      <!-- ========================================== -->
      <div
        v-if="isLoading"
        class="bg-white rounded-2xl p-12 text-center shadow-sm border border-slate-200/80 flex flex-col items-center justify-center space-y-4"
      >
        <div class="relative">
          <div class="w-14 h-14 rounded-full border-4 border-blue-100 border-t-blue-600 animate-spin"></div>
          <div class="absolute inset-0 flex items-center justify-center text-blue-600">
            <svg class="w-6 h-6 animate-pulse" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 15a4 4 0 004 4h9a5 5 0 10-.1-9.999 5.002 5.002 0 00-9.78 2.096A4.001 4.001 0 003 15z" />
            </svg>
          </div>
        </div>
        <div class="space-y-1">
          <h3 class="text-base font-semibold text-slate-800">Memuat data cuaca...</h3>
          <p class="text-sm text-slate-500">Mengambil prakiraan per jam dari API Open-Meteo untuk Jakarta</p>
        </div>
      </div>

      <!-- ========================================== -->
      <!-- STATE: ERROR                               -->
      <!-- ========================================== -->
      <div
        v-else-if="errorMessage"
        class="bg-red-50 border border-red-200 rounded-2xl p-6 sm:p-8 text-center space-y-4 shadow-sm"
      >
        <div class="w-12 h-12 mx-auto rounded-full bg-red-100 text-red-600 flex items-center justify-center">
          <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z" />
          </svg>
        </div>
        <div class="space-y-1">
          <h3 class="text-lg font-bold text-red-800">Gagal Memuat Data Cuaca</h3>
          <p class="text-sm text-red-600 max-w-md mx-auto">{{ errorMessage }}</p>
        </div>
        <button
          id="retry-fetch-btn"
          @click="fetchWeatherData"
          class="inline-flex items-center gap-2 px-5 py-2.5 rounded-xl bg-red-600 hover:bg-red-700 active:scale-95 transition-all text-white text-sm font-semibold shadow-md shadow-red-500/20"
        >
          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" />
          </svg>
          Coba Lagi
        </button>
      </div>

      <!-- ========================================== -->
      <!-- TABEL DATA CUACA (RESPONSIF)              -->
      <!-- ========================================== -->
      <section v-else class="bg-white rounded-2xl shadow-sm border border-slate-200/80 overflow-hidden">
        
        <!-- Header informasi tabel -->
        <div class="p-4 sm:p-5 border-b border-slate-200/80 flex flex-col sm:flex-row sm:items-center justify-between gap-2">
          <div>
            <h2 class="text-base sm:text-lg font-bold text-slate-800">
              Tabel Prakiraan Suhu
            </h2>
            <p class="text-xs text-slate-500">
              Menampilkan data per jam dengan konversi ke Fahrenheit dan Kelvin
            </p>
          </div>
          <div class="text-xs text-slate-500">
            Total Data: <strong class="text-slate-800">{{ filteredWeatherRecords.length }}</strong> jam
          </div>
        </div>

        <!-- Wrapper dengan overflow-x-auto untuk responsivitas layar kecil -->
        <div class="overflow-x-auto w-full">
          <table class="w-full text-left border-collapse">
            <thead>
              <tr class="bg-slate-50/80 border-b border-slate-200 text-xs font-semibold text-slate-600 uppercase tracking-wider">
                <th scope="col" class="py-3.5 px-4 sm:px-6">Waktu</th>
                <th scope="col" class="py-3.5 px-4 sm:px-6">Suhu (°C)</th>
                <th scope="col" class="py-3.5 px-4 sm:px-6">Suhu (°F)</th>
                <th scope="col" class="py-3.5 px-4 sm:px-6">Suhu (K)</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-slate-100 text-sm">
              <!-- Jika data kosong setelah filter -->
              <tr v-if="paginatedRecords.length === 0">
                <td colspan="4" class="py-12 text-center text-slate-400">
                  <svg class="w-10 h-10 mx-auto text-slate-300 mb-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9.172 16.172a4 4 0 015.656 0M9 10h.01M15 10h.01M21 12a9 9 0 11-18 0 9 9 0 0118 0z" />
                  </svg>
                  Tidak ada data yang sesuai dengan pencarian.
                </td>
              </tr>

              <!-- Baris Data Cuaca dengan Efek Hover -->
              <tr
                v-for="(row, idx) in paginatedRecords"
                :key="row.time + idx"
                class="hover:bg-blue-50/60 transition-colors duration-150 group"
              >
                <!-- Kolom Waktu -->
                <td class="py-3.5 px-4 sm:px-6 whitespace-nowrap text-slate-700 font-medium">
                  <div class="flex items-center gap-2">
                    <svg class="w-4 h-4 text-slate-400 group-hover:text-blue-500 transition-colors" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z" />
                    </svg>
                    <span>{{ row.time }}</span>
                  </div>
                </td>

                <!-- Kolom Suhu Celcius -->
                <td class="py-3.5 px-4 sm:px-6 whitespace-nowrap">
                  <span
                    class="inline-flex items-center px-2.5 py-1 rounded-md font-semibold text-xs transition-colors"
                    :class="{
                      'bg-sky-100 text-sky-800': row.tempCelsius < 26,
                      'bg-emerald-100 text-emerald-800': row.tempCelsius >= 26 && row.tempCelsius <= 31,
                      'bg-amber-100 text-amber-800': row.tempCelsius > 31,
                    }"
                  >
                    {{ row.tempCelsius }} °C
                  </span>
                </td>

                <!-- Kolom Suhu Fahrenheit -->
                <td class="py-3.5 px-4 sm:px-6 whitespace-nowrap text-slate-600 font-mono text-sm">
                  {{ row.tempFahrenheit }} °F
                </td>

                <!-- Kolom Suhu Kelvin -->
                <td class="py-3.5 px-4 sm:px-6 whitespace-nowrap text-slate-600 font-mono text-sm">
                  {{ row.tempKelvin }} K
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- ========================================== -->
        <!-- PAGINATION CONTROLS                        -->
        <!-- ========================================== -->
        <div
          v-if="filteredWeatherRecords.length > 0"
          class="p-4 border-t border-slate-200/80 bg-slate-50/50 flex flex-col sm:flex-row items-center justify-between gap-3 text-sm text-slate-600"
        >
          <div>
            Menampilkan data
            <strong class="text-slate-800">{{ ((currentPage - 1) * itemsPerPage) + 1 }}</strong>
            sampai
            <strong class="text-slate-800">{{ Math.min(currentPage * itemsPerPage, filteredWeatherRecords.length) }}</strong>
            dari
            <strong class="text-slate-800">{{ filteredWeatherRecords.length }}</strong> data
          </div>

          <div class="inline-flex items-center gap-1">
            <button
              id="prev-page-btn"
              @click="handlePageChange(currentPage - 1)"
              :disabled="currentPage === 1"
              class="px-3 py-1.5 rounded-lg border border-slate-300 bg-white text-slate-700 text-xs font-medium hover:bg-slate-100 active:scale-95 transition-all disabled:opacity-40 disabled:cursor-not-allowed"
            >
              Sebelumnya
            </button>

            <span class="px-3 py-1 text-xs font-semibold text-slate-700">
              Halaman {{ currentPage }} dari {{ totalPages }}
            </span>

            <button
              id="next-page-btn"
              @click="handlePageChange(currentPage + 1)"
              :disabled="currentPage === totalPages"
              class="px-3 py-1.5 rounded-lg border border-slate-300 bg-white text-slate-700 text-xs font-medium hover:bg-slate-100 active:scale-95 transition-all disabled:opacity-40 disabled:cursor-not-allowed"
            >
              Selanjutnya
            </button>
          </div>
        </div>

      </section>

      <!-- ========================================== -->
      <!-- FOOTER & KETERANGAN RUMUS                  -->
      <!-- ========================================== -->
      <footer class="rounded-xl p-4 bg-white border border-slate-200/80 text-xs text-slate-500 space-y-2">
        <div class="font-semibold text-slate-700">Catatan Konversi Suhu:</div>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-2 text-slate-600">
          <div class="flex items-center gap-2">
            <span class="w-2 h-2 rounded-full bg-blue-500"></span>
            <span><strong>Fahrenheit (°F):</strong> Rumus: <code>(C &times; 9/5) + 32</code> (dibulatkan 2 desimal)</span>
          </div>
          <div class="flex items-center gap-2">
            <span class="w-2 h-2 rounded-full bg-indigo-500"></span>
            <span><strong>Kelvin (K):</strong> Rumus: <code>C + 273.15</code> (dibulatkan 2 desimal)</span>
          </div>
        </div>
      </footer>

    </div>
  </div>
</template>
