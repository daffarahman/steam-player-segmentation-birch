# Case Method BIRCH: Segmentasi Pemain Steam

## Tujuan tugas

Mengidentifikasi segmen pemain Steam berdasarkan ukuran *library* dan pola waktu bermain yang tercatat, menentukan fitur yang paling membedakan segmen, lalu merumuskan strategi untuk mendorong keterlibatan pemain berdasarkan profil tiap segmen. Pengelompokan pemain memakai BIRCH.

| Komponen | Definisi dalam kasus Steam |
| --- | --- |
| Permasalahan A | Statistik agregat menyamarkan perbedaan cara pemain menggunakan *library* mereka. Tim produk membutuhkan dasar untuk memilih strategi penemuan *game* atau keterlibatan yang sesuai untuk kelompok pemain yang berbeda. |
| Informasi B | Segmen pemain yang terbentuk dari ukuran *library*, jumlah *game* dimainkan, total jam bermain, proporsi *library* yang dimainkan, dan konsentrasi waktu bermain. |
| Keputusan | Menentukan strategi yang layak diuji untuk tiap segmen dan indikator keberhasilannya. |
| Unit analisis | Satu profil pemain yang diringkas dari catatan interaksi antara pemain dan *game*. |

> Segmen pemain Steam apa yang terbentuk dari pola kepemilikan *game* dan waktu bermain yang tercatat, fitur apa yang paling membedakannya, dan strategi keterlibatan apa yang layak diuji untuk setiap segmen?

**Luaran akhir** berupa laporan analisis kasus dan rekomendasi strategi yang mengaitkan setiap usulan dengan bukti dari profil klaster.

## Dasar analisis

Implementasi tersedia di [notebook segmentasi pemain Steam](birch-steam.ipynb). Notebook membaca `steam-200k.csv` dari direktori kerja kernel. Gunakan keluaran eksekusi untuk menentukan konfigurasi terpilih, jumlah klaster akhir, nilai metrik, dan profil setiap segmen.

## Menjalankan notebook dengan uv

1. Dari direktori proyek, jalankan `uv sync --locked` untuk menyiapkan environment Python 3.13 dari `uv.lock`.
2. Jika `steam-200k.csv` belum tersedia, ekstrak file tersebut dari unduhan Kaggle dan simpan di direktori yang sama dengan notebook. [Dataset Steam Video Games](https://www.kaggle.com/datasets/tamber/steam-video-games)
3. Jalankan `uv run jupyter lab`. Buka `birch-steam.ipynb`, pilih kernel **Python 3 (ipykernel)**, lalu jalankan semua sel. [Panduan Jupyter untuk uv](https://docs.astral.sh/uv/guides/integration/jupyter/)

## Tujuh langkah analisis

### 1. Mencari dan memahami karakteristik dataset

- Dataset Steam Video Games berisi sekitar 200.000 interaksi dari 12.393 pengguna dan 5.155 *game* menurut dokumentasi data. CSV dibaca tanpa header dan kolom kelima dibuang. Pada baris `purchase`, `value=1` adalah penanda *library*. Pada baris `play`, `value` berisi jam bermain. [Dataset Kaggle](https://www.kaggle.com/datasets/tamber/steam-video-games), [dokumentasi data Steam CaseRec](https://github.com/caserec/Datasets-for-Recommender-Systems/blob/master/Processed%20Datasets/Steam/README.md)
- Catat jumlah aktual interaksi, pemain, *game*, dan profil setelah pembersihan. Data historis ini berasal dari akun publik. Analisis terbatas pada aktivitas yang tercatat karena data sesi bertanggal, harga pembelian, dan hasil kampanye tidak tersedia.

### 2. Melakukan preprocessing data

- Notebook menormalkan penulisan kolom teks, mengonversi `value` ke numerik, dan menyaring jenis interaksi. Catatan `purchase` dideduplikasi per pasangan pemain dan *game*. Jika ada beberapa catatan `play` positif untuk pasangan yang sama, notebook mengambil nilai maksimum dengan asumsi catatan tersebut dapat mewakili cuplikan jam kumulatif.
- Jam bermain dihitung untuk *game* yang memiliki penanda `purchase` pada pemain yang sama. Pemain dengan penanda *library* yang tidak memiliki catatan `play` positif yang cocok tetap masuk ke analisis dengan fitur bermain bernilai nol. Verifikasi jumlah catatan bermain yang tidak cocok dan cek nilai kosong pada ID pemain dan judul *game* sebelum agregasi.

| Fitur model | Definisi |
| --- | --- |
| `games_owned` | Jumlah *game* unik dengan penanda `purchase`, termasuk kemungkinan *game* gratis. |
| `games_played` | Jumlah *game* dalam *library* dengan waktu bermain positif yang tercatat. |
| `total_hours` | Total jam bermain tercatat pada *game* dalam *library*. |
| `play_ratio` | Proporsi *game* dalam *library* dengan waktu bermain positif yang tercatat. |
| `focus_ratio` | Pangsa total jam pada *game* yang paling lama dimainkan. Diisi nol ketika tidak ada jam bermain yang tercatat. |

- Terapkan `log1p` pada dua fitur jumlah *game* dan `total_hours`, lalu gunakan `StandardScaler` pada kelima fitur. ID pemain dipakai untuk menghubungkan profil dan label klaster. `max_game_hours` serta `is_no_play_recorded` dipakai untuk perhitungan dan interpretasi.

### 3. Menganalisis pola

- Notebook sudah menyediakan statistik deskriptif. Lengkapi eksplorasi sebelum klastering dengan visualisasi distribusi ukuran *library*, jam bermain, dan kedua rasio, serta catat proporsi profil tanpa waktu bermain yang tercatat.
- Periksa hubungan antara `games_owned`, `games_played`, dan `play_ratio`, karena rasio tersebut dihitung dari dua fitur lainnya. Catat juga bahwa `focus_ratio=0` pada profil tanpa waktu bermain adalah nilai pengganti. Interpretasikan konsentrasi bermain bersama `total_hours` dan persentase profil tanpa catatan bermain.

### 4. Menggunakan algoritma BIRCH

- Notebook menguji `threshold` 0,5, 0,8, dan 1,1 dengan permintaan 2–6 klaster akhir serta `branching_factor=50`. Catat jumlah klaster yang diminta, jumlah yang terbentuk, jumlah subklaster, dan waktu pemodelan. BIRCH dijalankan pada lima fitur terstandarisasi. [Dokumentasi BIRCH scikit-learn](https://scikit-learn.org/stable/modules/generated/sklearn.cluster.Birch.html)

### 5. Mengevaluasi hasil model

- Notebook menghitung Silhouette Score pada sampel tetap hingga 1.000 profil dan Davies–Bouldin Index pada seluruh profil. Konfigurasi diurutkan berdasarkan Silhouette tertinggi, lalu Davies–Bouldin terendah sebagai pemecah seri. Klaster terkecil diprioritaskan beranggotakan minimal 1% dari seluruh profil, jika ada konfigurasi yang memenuhi kriteria tersebut.
- Bandingkan 15 konfigurasi BIRCH pada profil dan fitur yang sama. Laporkan skor, ukuran klaster, dan waktu pemodelan, lalu bandingkan ketiga radius pada jumlah klaster akhir yang terpilih. Gunakan PCA dua dimensi untuk menampilkan profil hasil pengelompokan. [Panduan evaluasi klaster scikit-learn](https://scikit-learn.org/stable/modules/clustering.html#clustering-performance-evaluation)

### 6. Menentukan faktor paling berpengaruh

- Notebook sudah membuat tabel ukuran klaster, median lima fitur pada skala asli, dan persentase pemain tanpa catatan bermain. Gunakan tabel ini untuk memberi nama segmen berdasarkan hasil yang terukur.
- Lengkapi dengan peringkat fitur pembeda. Untuk setiap fitur pada skala model, hitung selisih antara median klaster tertinggi dan terendah, lalu periksa klaster mana yang mendorong selisih tersebut. Jika ingin menguji sensitivitas, jalankan ulang BIRCH tanpa satu fitur terkait, misalnya `games_played` atau `play_ratio`, dan periksa perubahan keanggotaan klaster. Skrip reproduksi laporan juga menguji urutan masukan dan faktor percabangan dengan menjalankan ulang BIRCH, lalu mengukur perubahan keanggotaan klaster. Peringkat fitur menjelaskan perbedaan antarklaster pada data yang dianalisis.

### 7. Memberikan rekomendasi strategi

- Hubungkan setiap rekomendasi dengan profil klaster yang terbentuk, tindakan yang diusulkan, dan indikator keberhasilan yang bisa diukur pada pengujian berikutnya.
- Jika ada segmen dengan *library* besar tetapi proporsi *game* dimainkan rendah, uji saran untuk menemukan kembali *game* dalam *library*. Ukur persentase pemain yang mulai memainkan *game* tersebut selama periode uji. Jika ada segmen dengan jam bermain tinggi yang terkonsentrasi pada satu *game*, uji rekomendasi *game* serupa setelah data genre tersedia, lalu ukur persentase pemain yang mencoba rekomendasi tersebut.

## Struktur laporan akhir

1. **Pendahuluan** memuat permasalahan A, informasi B yang dibutuhkan, dan pertanyaan penelitian.
2. **Dataset dan metode** menjelaskan sumber data, pembentukan profil pemain, preprocessing, eksplorasi, dan konfigurasi BIRCH.
3. **Hasil dan pembahasan** memuat evaluasi antar konfigurasi BIRCH, profil dan ukuran klaster, visualisasi PCA, serta faktor pembeda utama.
4. **Rekomendasi strategi** mengaitkan tiap segmen yang relevan dengan tindakan, indikator, dan cara menguji usulan.
5. **Kesimpulan dan keterbatasan** menjawab pertanyaan penelitian dengan ruang lingkup data historis akun publik dan menjelaskan bukti tambahan yang dibutuhkan untuk menilai dampak strategi.
