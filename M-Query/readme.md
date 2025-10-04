# 📊 YouTube Channel Details with Power BI + YouTube API

Proyek ini mendemonstrasikan cara mengintegrasikan **data Channel YouTube** ke dalam **Power BI** menggunakan **YouTube Data API v3** secara langsung.

Skrip Power Query M ini mengambil informasi terperinci seperti nama channel, deskripsi, jumlah subscriber, dan gambar profil untuk daftar channel yang telah ditentukan.

---

## 🚀 Cara Menggunakan

1. D**apatkan Kunci API Anda:**
   - Buat proyek baru di [Google Cloud Console](https://console.cloud.google.com).
   - Aktifkan **YouTube Data API v3** untuk proyek tersebut.
   - Buat kredensial dan salin **Kunci API** Anda.

2. **Siapkan Parameter di Power BI (Direkomendasikan):**
   - Buka **Power Query Editor** di Power BI `(Home > Transform data)`.
   - Buka `Manage Parameters > New Parameter`.
   - Buat parameter bernama `Key` (Type: Text) dan masukkan Kunci API YouTube Anda sebagai nilainya.
   - Buat parameter lain bernama `ChannelIDs_Text` (Type: Text) dan masukkan daftar ID Channel yang ingin Anda lacak, dipisahkan dengan koma (contoh: `UCoIiiHof6BJ85PLuLkuxuhw,UCn7Ohm5Dv7uKdODpjVo0xSQ`).

3. **Jalankan Kueri:**
   - Di Power Query Editor, buat **Kueri Kosong Baru** `(New Blank Query)`.
   - Buka **Advanced Editor**.
   - Salin salah satu **M Script** di bawah ini dan tempelkan ke dalam editor.
   - Jika Anda tidak mengatur parameter, ganti placeholder `API_KEY` dan daftar `ChannelIDs` secara langsung di dalam skrip.
   - Klik Selesai lalu **Tutup & Terapkan** untuk memuat data.

## 🔑 Catatan & Konfigurasi

1. **Kuota API:**
   - YouTube Data API memiliki **kuota penggunaan harian gratis** (biasanya 10.000 unit per hari).
   - Setiap panggilan ke `channels.list` dengan `part=snippet,statistics` menghabiskan **3 unit kuota**. Jadi, melacak 10 channel akan menghabiskan 30 unit kuota setiap kali data diperbarui.
   - Setiap panggilan ke `search.list` dengan `part=snippet` menghabiskan 100 unit kuota.
   - Setiap panggilan ke `videos.list` dengan `part=statistics` menghabiskan 2 unit kuota.
   - Perhitungkan penggunaan kuota Anda saat menentukan MaxVideosPerChannel dan jumlah channel yang dilacak.
     
2. **ID Channel:**
   - Pastikan Anda menggunakan **ID Channel** unik (biasanya diawali dengan UC...), bukan nama pengguna (@handle) atau URL kustom.

---
## 🆔 Cara Menemukan ID Channel YouTube

Pastikan Anda menggunakan **ID Channel** unik (yang selalu diawali dengan "UC"), bukan nama pengguna (@handle) atau URL kustom. Cara termudah dan paling akurat untuk mendapatkannya adalah sebagai berikut:

1. **Buka Halaman Channel YouTube**
Kunjungi halaman utama channel yang ingin Anda lacak datanya.

2. **Klik Tombol "Bagikan channel"**
Di bawah nama channel dan di atas tombol Subscribe, klik tulisan "selengkapnya" untuk membuka menu, lalu pilih **"Bagikan channel"**.

3. **Salin ID Channel**
Sebuah jendela pop-up akan muncul. Alih-alih menyalin link, klik tombol **"Salin ID Channel"** (Copy Channel ID). Ini akan menyalin ID unik 24 karakter langsung ke clipboard Anda.

4. **Gunakan ID yang Disalin**
Tempelkan ID yang sudah Anda salin ke dalam parameter `ChannelIDs_List` di Power BI.

---
## M Script

### 1. M Script: Mendapatkan Detail Channel
Versi ini menggunakan parameter Power BI Key_API_YouTube dan ChannelIDs_List agar lebih fleksibel dan aman tanpa harus mengedit kode.

```
let
    // --- KONFIGURASI (Mengambil dari Parameter Power BI) ---
    API_KEY = Key_API_YouTube,
    ChannelIDs_Text = ChannelIDs_List,

    // --- PROSES UTAMA ---
    // Pisahkan teks ID Channel menjadi sebuah list/daftar
    ChannelIDs_List = Text.Split(ChannelIDs_Text, ","),

    // --- FUNGSI UNTUK MENGAMBIL DETAIL CHANNEL ---
    GetChannelDetails = (channel_id as text) =>
    let
        // Trim spasi yang mungkin ada di ID channel
        CleanedID = Text.Trim(channel_id),
        URL = "https://www.googleapis.com/youtube/v3/channels?part=snippet,statistics&id=" & CleanedID & "&key=" & API_KEY,
        Source = Json.Document(Web.Contents(URL)),
        Items = Source[items]{0},
        Snippet = Items[snippet],
        Statistics = Items[statistics],
        ChannelName = Snippet[title],
        ChannelDescription = Snippet[description],
        CustomURL = try Snippet[customUrl] otherwise "N/A",
        ProfileImage = Snippet[thumbnails][high][url],
        Subscribers = Number.FromText(Statistics[subscriberCount])
    in
        [
            ChannelName = ChannelName,
            ChannelDescription = ChannelDescription,
            CustomURL = CustomURL,
            ProfileImage = ProfileImage,
            Subscribers = Subscribers
        ],
    
    // Terapkan fungsi ke setiap ID dalam daftar
    ChannelDetails = List.Transform(ChannelIDs_List, each GetChannelDetails(_)),
    TableFromRecords = Table.FromRecords(ChannelDetails),
    ChangedType = Table.TransformColumnTypes(TableFromRecords,{{"Subscribers", Int64.Type}})
in
    ChangedType
```
---

### 2. M Script: Mendapatkan Detail Video Terbaru
Skrip ini akan mengambil daftar video terbaru dari channel yang ditentukan, beserta statistik seperti jumlah likes, comments, dan views.
```
let
    // --- KONFIGURASI (Mengambil dari Parameter Power BI) ---
    API_KEY = Key_API_YouTube,
    ChannelIDs_Text = ChannelIDs_List,
    MaxResults = MaxVideosPerChannel, // Jumlah video terbaru per channel yang akan diambil

    // --- PROSES UTAMA ---
    // Pisahkan teks ID Channel menjadi sebuah list/daftar
    ChannelIDs_List = Text.Split(ChannelIDs_Text, ","),

    // --- FUNGSI UNTUK MENGAMBIL DETAIL VIDEO (PART 1: SNIPPET) ---
    GetVideoDetails = (channel_id as text) =>
    let
        CleanedID = Text.Trim(channel_id),
        URL = "https://www.googleapis.com/youtube/v3/search?part=snippet&channelId=" & CleanedID & "&order=date&type=video&maxResults=" & Number.ToText(MaxResults) & "&key=" & API_KEY,
        Source = Json.Document(Web.Contents(URL)),
        Items = Source[items],
        Videos = List.Transform(Items, each [
            ChannelID = CleanedID, // Tambahkan ChannelID untuk join nanti
            ChannelName = _[snippet][channelTitle],
            VideoTitle = _[snippet][title],
            PublishedAt = _[snippet][publishedAt],
            VideoDescription = _[snippet][description],
            VideoThumbnail = _[snippet][thumbnails][high][url],
            VideoID = _[id][videoId],
            VideoLink = "https://www.youtube.com/watch?v=" & _[id][videoId]
        ])
    in
        Videos,
    
    // --- FUNGSI UNTUK MENGAMBIL STATISTIK VIDEO (PART 2: STATISTICS) ---
    GetVideoStatistics = (video_id as text) =>
    let
        URL = "https://www.googleapis.com/youtube/v3/videos?part=statistics&id=" & video_id & "&key=" & API_KEY,
        Source = Json.Document(Web.Contents(URL)),
        // Pastikan item ada sebelum mengaksesnya
        Statistics = if List.Count(Source[items]) > 0 then Source[items]{0}[statistics] else null,
        Likes = if Statistics <> null and Record.HasFields(Statistics, {"likeCount"}) then Number.FromText(Statistics[likeCount]) else 0,
        Comments = if Statistics <> null and Record.HasFields(Statistics, {"commentCount"}) then Number.FromText(Statistics[commentCount]) else 0,
        Views = if Statistics <> null and Record.HasFields(Statistics, {"viewCount"}) then Number.FromText(Statistics[viewCount]) else 0
    in
        [Likes = Likes, Comments = Comments, Views = Views],
    
    // --- GABUNGKAN SEMUA DATA ---
    // Ambil detail video untuk semua channel yang ditentukan
    AllVideoSnippets = List.Combine(List.Transform(ChannelIDs_List, each GetVideoDetails(_))),

    // Ekstrak VideoID dari setiap video snippet
    AllVideoIDs = List.Transform(AllVideoSnippets, each _[VideoID]),
    
    // Ambil statistik untuk setiap VideoID
    AllVideoStats = List.Transform(AllVideoIDs, each GetVideoStatistics(_)),

    // Gabungkan snippet video dengan statistiknya berdasarkan indeks
    MergedVideoList = List.Zip({AllVideoSnippets, AllVideoStats}),
    CombinedVideos = List.Transform(MergedVideoList, each Record.Combine(_)),

    // Konversi ke tabel dan ubah tipe data
    TableFromRecords = Table.FromRecords(CombinedVideos),
    #"Changed Type" = Table.TransformColumnTypes(TableFromRecords,{
        {"Likes", Int64.Type},
        {"Comments", Int64.Type},
        {"Views", Int64.Type},
        {"PublishedAt", type datetime}
    })
in
    #"Changed Type"
```

---
## 📌 Penjelasan Skrip Detail Video

Skrip ini bekerja dalam tiga tahap utama: mengambil informasi dasar video, mengambil statistik video, dan kemudian menggabungkan keduanya menjadi satu tabel yang utuh.

1. **Fungsi `GetVideoDetails` (Mengambil Info Dasar Video)**
Fungsi ini bertanggung jawab untuk mencari video-video terbaru dari setiap channel yang ditentukan.
    - **Ekstraksi Data:**
      - **ChannelName:** Mengambil nama channel dari video.
      - **VideoTitle:** Mengambil judul spesifik dari setiap video.
      - **PublishedAt:** Mengambil tanggal dan waktu video tersebut diunggah.
      - **VideoDescription:** Mengambil teks deskripsi yang ada di bawah video.
      - **VideoThumbnail:** Mengambil URL gambar thumbnail video dalam resolusi tinggi.
      - **VideoID:** Mengambil ID unik dari video, yang akan digunakan untuk mencari statistiknya.
      - **VideoLink:** Membuat URL YouTube lengkap yang bisa diklik untuk menonton video tersebut.

2. **Fungsi `GetVideoStatistics` (Mengambil Statistik Video)**
Setelah mendapatkan VideoID dari fungsi pertama, fungsi ini dipanggil untuk setiap video guna mengambil data engagement-nya.
    - **Ekstraksi Data:**
      - **Likes:** Mengambil jumlah "suka" (likes) pada video.
      - **Comments:** Mengambil jumlah total komentar.
      - **Views:** Mengambil jumlah total penayangan (views) video.
        
3. **Proses Penggabungan dan Pembuatan Tabel**
Ini adalah bagian utama di mana semua data yang telah dikumpulkan disatukan.
    - **Pengumpulan Data:** Pertama, skrip menjalankan fungsi `GetVideoDetails` untuk semua channel dan menggabungkan hasilnya menjadi satu daftar besar berisi semua video.
    - **Pengambilan Statistik:** Selanjutnya, skrip mengambil `VideoID` dari setiap video dalam daftar tersebut, lalu memanggil fungsi GetVideoStatistics untuk masing-masing ID guna mendapatkan data likes, comments, dan views.
    - **Penggabungan Tabel:** Fungsi `List.Zip` dan `Record.Combine` digunakan untuk "mencocokkan" data dasar setiap video dengan data statistiknya, sehingga menjadi satu baris data yang lengkap.
    - **Tabel Akhir:** Terakhir, semua data yang sudah lengkap diubah menjadi format tabel dan tipe datanya disesuaikan (misalnya, angka untuk Likes/Views dan tanggal untuk PublishedAt) agar siap untuk dianalisis dan divisualisasikan.
