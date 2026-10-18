<template>
    <ion-page>
        <ion-header>
            <ion-toolbar color="primary">
                <ion-title>Prakiraan Cuaca Jakarta</ion-title>

                <ion-buttons slot="end">
                    <ion-button @click="ambilData" :disabled="loading" aria-label="Refresh data cuaca">
                        <ion-icon slot="icon-only" :icon="refreshOutline" :class="{ berputar: loading }"/>
                    </ion-button>
                </ion-buttons>
            </ion-toolbar>
        </ion-header>

        <!--halaman-->
        <ion-content class="ion-padding halaman-cuaca">
            <div class="container-cuaca">
                <div class="judul-halaman">
                    <p class="label-lokasi">PRAKIRAAN CUACA JAKARTA</p>
                    <h1>Cuaca Jakarta</h1>
                    <p class="tanggal">{{ tanggalHariIni }}</p>
                </div>
                <!--loading-->
                <div v-if="loading" class="pesan-status">
                    <ion-spinner name="crescent"/>
                    <p>Ambil data cuaca...</p>
                </div>
                <!--tampilan pesan error-->
                <ion-card v-else-if="error" class="kartu-error">
                    <ion-card-content>
                        <ion-icon :icon="alertCircleOutline" />
                        <p>{{ error }}</p>
                        <ion-button @click="ambilData">CobaLagi</ion-button>
                    </ion-card-content>
                </ion-card>

                <!--tampilan data berhasil dimuat-->
                <template v-else-if="dataCuaca.length > 0">
                    <!--suhu di hari ini-->
                    <ion-card class="kartu-suhu">
                        <ion-card-content>
                            <div class="baris-suhu">
                                <div>
                                    <p class="label-suhu">SUHU HARI INI</p>
                                    <h2>{{ suhuTerkini }}</h2>
                                    <p class="satuan">Temperatur udara (°C)</p>
                                </div>
                                <ion-icon :icon="thermometerOutline" class="ikon-suhu"/>
                            </div>
                            <div class="garis-pemisah"></div>
                            <p class="waktu-suhu">
                                <ion-icon :icon="timeOutline"/>{{ keteranganSuhu }}
                            </p>

                            <p class="catatan">Data prakiraan dari Open-Meteo API.</p>
                        </ion-card-content>
                    </ion-card>

                    <!--judul tabel-->
                    <div class="judul-tabel">
                        <div>
                            <h2>Temperatur Per Jam</h2>
                            <p>Waktu Pengukuran Suhu dan Udara</p>
                        </div>
                        <ion-badge color="primary">{{ dataCuaca.length }} data</ion-badge>
                    </div>

                    <!--tabel data-->
                    <ion-card class="kartu-tabel">
                        <ion-card-content class="isi-tabel">
                            <div class="tabel-scroll">
                                <table>
                                    <thead>
                                        <tr>
                                            <th>No.</th>
                                            <th>Time</th>
                                            <th>Temperature_2m (°C)</th>
                                        </tr>
                                    </thead>

                                    <tbody>
                                        <tr v-for="(item, index) in dataDitampilkanlayar" :key="item.time">
                                            <td>{{ index + 1 }}</td>
                                            <td>{{ formatWaktu(item.time) }}</td>
                                            <td>
                                                <span class="nilai-suhu">{{ formatSuhu(item.temperature) }}°C</span>
                                            </td>
                                        </tr>
                                    </tbody>
                                </table>
                            </div>

                            <p class="info-data">Menampilkan {{ dataDitampilkanlayar.length }} dari {{ dataCuaca.length }} data.</p>

                            <!--button lihat semua data-->
                            <ion-button expand="block" fill="outline" @click="tampilkanSemuadata = !tampilkanSemuadata">
                                <ion-icon slot="start" :icon="tampilkanSemuadata ? chevronUpOutline : chevronDownOutline"/>
                                {{ tampilkanSemuadata ? 'Tampilkan 10 Data Saja' : 'Lihat Data Selengkapnya' }}
                            </ion-button>

                        </ion-card-content>
                    </ion-card>

                    <!--info sumber API-->
                    <div class="catatan-bawah">
                        <ion-icon :icon="informationCircleOutline"/>
                        <p>
                            sumber API: <a :href="API_URL" target="_blank" rel="noopener noreferrer">
                                Buka Open-Meteo API
                            </a>
                        </p>
                    </div>
                </template>
            </div>
        </ion-content>
    </ion-page>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import axios from 'axios'

import {
    IonPage, IonHeader, IonToolbar, IonTitle, IonButtons, IonButton, IonIcon, IonContent, IonCard, IonCardContent, IonSpinner, IonBadge
} from '@ionic/vue'
import { refreshOutline, thermometerOutline, timeOutline, alertCircleOutline, chevronDownOutline, chevronUpOutline, informationCircleOutline } from 'ionicons/icons'

const API_URL = 'https://api.open-meteo.com/v1/forecast?latitude=-6.2&longitude=106.8&hourly=temperature_2m'

const dataCuaca = ref([])
const loading = ref(false)
const error = ref('')
const tampilkanSemuadata = ref(false)
const waktuSekarang = ref(new Date())

//tanggal hari ini
const tanggalHariIni = computed(() => {
    return new Intl.DateTimeFormat('id-ID', {
        timeZone: 'Asia/Jakarta',
        weekday: 'long',
        day: 'numeric',
        month: 'long',
        year: 'numeric'
    }).format(waktuSekarang.value)
})

//ambil data menggunakan axios
async function ambilData() {
    loading.value = true
    error.value = ''

    try {
        //panggil URL API 
        const response = await axios.get(API_URL, {
            timeout: 15000
        })

        //ambil dua field dari respon API 
        const waktu = response.data.hourly.time
        const suhu = response.data.hourly.temperature_2m

        //gabungkan waktu & suhu sesuai indeks
        dataCuaca.value = waktu.map((time, index) => ({
            time: time,
            temperature: suhu[index]
        }))

        //kembali 10 data pertama setelah refresh
        tampilkanSemuadata.value = false

        //perbarui waktu aplikasi
        waktuSekarang.value = new Date()

    } catch (err) {
        console.error('Gagal ambil data:', err)

        if (err.code === 'ECONNABORTED') {
            error.value = 'Periksa Internet dan Coba Lagi.'
        } else if (err.response) {
            error.value = 'API bermasalah. Coba Lagi Nanti.'
        } else {
            error.value = 'Gagal ambil data. Periksa Internet Anda.'
        }
    } finally {
        loading.value = false
    }
}

//Tampilkan 10 atau semua data
const dataDitampilkanlayar = computed(() => {
    return tampilkanSemuadata.value ? dataCuaca.value : dataCuaca.value.slice(0, 10)
})

//tentukan suhu berdasarkan jam hari ini
const dataTerkini = computed(() => {
    const bagianWaktu = new Intl.DateTimeFormat('en-GB', {
        timeZone: 'Asia/Jakarta',
        year: 'numeric',
        month: '2-digit',
        day: '2-digit',
        hour: '2-digit',
        hourCycle: 'h23'
    }).formatToParts(waktuSekarang.value)
    const bagian = {}

    bagianWaktu.forEach(item => {
        bagian[item.type] = item.value
    })
    const tanggalJakarta = `${bagian.year}-${bagian.month}-${bagian.day}`

    const batasWaktu = `${tanggalJakarta}T${bagian.hour}:00`
    //pilih data yang tanggalnya sama dengan hari ini
    const dataHariIni = dataCuaca.value.filter(item => item.time.startsWith(tanggalJakarta))
    //cari data dari jam terakhir
    const dataSebelumnya = dataHariIni.filter(item => item.time <= batasWaktu)
    if (dataSebelumnya.length > 0) {
        return dataSebelumnya[dataSebelumnya.length - 1]
    }
    //jika belum ada data di jam sebelumnya
    return dataHariIni[0] || null
})

//Tampilkan Nilai Suhu
const suhuTerkini = computed(() => {
    if (!dataTerkini.value) {
        return '--°C'
    }
    return `${formatSuhu(dataTerkini.value.temperature)}°C`
})

const keteranganSuhu = computed(() => {
    if (!dataTerkini.value) {
        return 'Data hari ini belum tersedia'
    }
    return `Waktu pengukuran: ${formatJam(dataTerkini.value.time)}`
})

//Format waktu & suhu
function formatWaktu(waktu) {
    const [tanggal, jam] = waktu.split('T')
    const [tahun, bulan, hari] = tanggal.split('-')
    return `${hari}-${bulan}-${tahun} ${jam}`
}

function formatJam(waktu) {
    return `${waktu.split('T')[1]}`
}

function formatSuhu(suhu) {
    if (suhu === null || suhu === undefined) {
        return '--'
    }
    return Number(suhu).toFixed(1)
}

//Memuat data homepage
onMounted(() => {
    ambilData()
})
</script>

<style scoped>
    .halaman-cuaca {
        --background: #f4f7fb;
    }
    .container-cuaca {
        max-width: 850px;
        margin: 0 auto;
        padding-bottom: 24px;
    }

    /* judul */
    .judul-halaman {
        padding: 12px 4px 20px;
    }
    .label-lokasi {
        color: #1769aa;
        font-size: 12px;
        font-weight: 700;
        letter-spacing: 1.5px;
    }
    .judul-halaman h1 {
        color: #172b4d;
        font-size: 28px;
        font-weight: 800;
        margin: 8px 0;
    }
    .tanggal {
        color: #66758a;
        margin: 0;
        text-transform: capitalize;
    }

    /*card suhu*/
    .kartu-suhu {
        margin: 0 0 28px;
        border-radius: 20px;
        background: linear-gradient(135deg, #1769aa, #2549a5);
        color: white;
        box-shadow: 0 8px 24px #174a8b26;
    }
    .kartu-suhu ion-card-content {
        padding: 24px;
    }
    .baris-suhu {
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 12px;
    }
    .label-suhu {
        font-size: 12px;
        font-weight: 700;
        letter-spacing: 1px;
    }
    .satuan,
    .catatan {
        font-size: 13px;
        opacity: 0.85;
    }
    .ikon-suhu {
        font-size: 65px;
        opacity: 0.85;
    }
    .garis-pemisah {
        height: 1px;
        background: #ffffff45;
        margin: 18px 0;
    }
    .waktu-suhu {
        display: flex;
        align-items: center;
        gap: 8px;
        font-size: 14px;
    }

    /*judul tabel*/
    .judul-tabel {
        display: flex;
        justify-content: space-between;
        align-items: center;
        gap: 12px;
        margin-bottom: 12px;
    }
    .judul-tabel h2 {
        color: #172b4d;
        font-size: 19px;
        font-weight: 750;
        margin: 0 0 6px;
    }
    .judul-tabel p {
        color: #66758a;
        font-size: 13px;
        margin: 0;
    }
    /*tabel*/
    .kartu-tabel {
        margin: 0;
        border-radius: 16px;
        box-shadow: 0 4px 18px #102c4d0d;
    }
    .isi-tabel {
        padding: 12px;
    }
    .tabel-scroll {
        width: 100%;
        overflow-x: auto;
    }
    table {
        width: 100%;
        border-collapse: collapse;
        text-align: left;
        font-size: 13px;
        white-space: nowrap;
    }
    thead {
        background: #edf4fc;
        color: #244c79;
    }
    th,
    td {
        padding: 14px 10px;
        border-bottom: 1px solid #e7edf4;
    }
    th {
        font-weight: 700;
    }
    tbody tr:last-child td {
        border-bottom: none;
    }
    .nilai-suhu {
        color: #1769aa;
        font-weight: 750;
    }
    .info-data {
        color: #718096;
        font-size: 12px;
        margin: 16px 0;
    }
    /*sumber data API*/
    .catatan-bawah {
        display: flex;
        align-items: flex-start;
        gap: 8px;
        color: #718096;
        font-size: 12px;
        margin-top: 20px;
    }
    .catatan-bawah ion-icon {
        font-size: 18px;
        flex-shrink: 0;
    }
    .catatan-bawah p {
        margin: 0;
        line-height: 1.6;
    }
    .catatan-bawah a {
        color: #1769aa;
    }
    /*loading*/
    .pesan-status {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 12px;
        padding: 50px 0;
        color: #1769aa;
    }
    /*info error*/
    .kartu-error {
        border-radius: 16px;
        text-align: center;
    }
    .kartu-error ion-icon {
        font-size: 32px;
        color: #c0392b;
    }
    .kartu-error p {
        color: #c0392b;
    }

    /*animasi tombol refresh */
    .berputar {
        animation: putar 1s linear infinite;
    }
    @keyframes putar {
        to {
            transform: rotate(360deg);
        }
    }
</style>