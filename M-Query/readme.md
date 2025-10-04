🌐 Languages: [English](#english-version) | [Bahasa Indonesia](#indonesian-version)

---

<a name="english-version"></a>
# 📊 YouTube Channel Details with Power BI + YouTube API

---
This project demonstrates how to integrate **YouTube Channel data** into **Power BI** directly using the **YouTube Data API v3**.

This Power Query M script retrieves detailed information such as channel name, description, subscriber count, and profile image for a predefined list of channels. Additionally, there is a script that fetches recent video details (title, description, thumbnail, likes, comments, views) from these channels.

---

## 🚀 How to Use

1.  **Get Your API Key:**
    * Create a new project in **[Google Cloud Console](https://console.cloud.google.com)**.
    * Enable the **YouTube Data API v3** for that project.
    * Create credentials and copy your **API Key**.

2.  **Set Up Parameters in Power BI (Recommended):**
    * Open **Power Query Editor** in Power BI (`Home > Transform data`).
    * Go to `Manage Parameters > New Parameter`.
    * Create a parameter named `Key_API_YouTube` (Type: Text) and enter your YouTube API Key as its value.
    * Create another parameter named `ChannelIDs_List` (Type: Text) and enter a comma-separated list of Channel IDs you want to track (e.g., `UCoIiiHof6BJ85PLuLkuxuhw,UCn7Ohm5Dv7uKdODpjVo0xSQ`).
    * (Optional) Create another parameter named `MaxVideosPerChannel` (Type: Number) and set a default value (e.g., `10`) to control how many recent videos are retrieved per channel.

3.  **Run the Query:**
    * In Power Query Editor, create a **New Blank Query**.
    * Open the **Advanced Editor**.
    * Copy one of the **M Scripts** below and paste it into the editor.
    * If you did not set up parameters, replace the placeholders for `API_KEY` and the `ChannelIDs` list, as well as `MaxResults`, directly in the script.
    * Click **Done** and then **Close & Apply** to load the data.

---
## 🔑 Notes & Configuration

1.  **API Quota:**
    * The YouTube Data API has a **free daily usage quota** (typically 10,000 units per day).
    * Each call to `channels.list` with `part=snippet,statistics` consumes **3 quota units**. So, tracking 10 channels will consume 30 quota units each time the data is refreshed.
    * Each call to `search.list` with `part=snippet` consumes **100 quota units**.
    * Each call to `videos.list` with `part=statistics` consumes **2 quota units**.
    * Consider your quota usage when determining `MaxVideosPerChannel` and the number of channels being tracked.

2.  **Channel ID:**
    * Ensure you are using the unique **Channel ID** (which usually starts with "UC..."), not a username (@handle) or custom URL.

---
## 🆔 How to Find a YouTube Channel ID

Ensure you are using the unique **Channel ID** (which always starts with "UC"), not a username (@handle) or custom URL. The easiest and most accurate way to get it is as follows:

1.  **Go to the YouTube Channel Page**
   
    Visit the main page of the channel you want to track.

3.  **Click the "Share channel" Button**
   
    Below the channel name and above the Subscribe button, click the "more" (or three dots `⋮`) icon to open the menu, then select **"Share channel"**.

5.  **Copy Channel ID**

    A pop-up window will appear. Instead of copying the link, click the **"Copy Channel ID"** button. This will copy the unique 24-character ID directly to your clipboard.

7.  **Use the Copied ID**
   
    Paste the copied ID into the `ChannelIDs_List` parameter in Power BI.

---
## M Script

### 1. M Script: Get Channel Details
This version uses Power BI parameters `Key_API_YouTube` and `ChannelIDs_List` for more flexibility and security without having to edit the code.

```m
let
    // --- CONFIGURATION (Retrieving from Power BI Parameters) ---
    API_KEY = Key_API_YouTube,
    ChannelIDs_Text = ChannelIDs_List,

    // --- MAIN PROCESS ---
    // Split the Channel IDs text into a list
    ChannelIDs_List = Text.Split(ChannelIDs_Text, ","),

    // --- FUNCTION TO GET CHANNEL DETAILS ---
    GetChannelDetails = (channel_id as text) =>
    let
        // Trim any potential spaces in the channel ID
        CleanedID = Text.Trim(channel_id),
        URL = "[https://www.googleapis.com/youtube/v3/channels?part=snippet,statistics&id=](https://www.googleapis.com/youtube/v3/channels?part=snippet,statistics&id=)" & CleanedID & "&key=" & API_KEY,
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
    
    // Apply the function to each ID in the list
    ChannelDetails = List.Transform(ChannelIDs_List, each GetChannelDetails(_)),
    TableFromRecords = Table.FromRecords(ChannelDetails),
    ChangedType = Table.TransformColumnTypes(TableFromRecords,{{"Subscribers", Int64.Type}})
in
    ChangedType
```

---
**2. M Script: Get Latest Video Details**
This script retrieves a list of recent videos from the specified channels, along with statistics such as likes, comments, and views.
```
let
    // --- CONFIGURATION (Retrieving from Power BI Parameters) ---
    API_KEY = Key_API_YouTube,
    ChannelIDs_Text = ChannelIDs_List,
    MaxResults = MaxVideosPerChannel, // Number of latest videos per channel to retrieve

    // --- MAIN PROCESS ---
    // Split the Channel IDs text into a list
    ChannelIDs_List = Text.Split(ChannelIDs_Text, ","),

    // --- FUNCTION TO GET VIDEO DETAILS (PART 1: SNIPPET) ---
    GetVideoDetails = (channel_id as text) =>
    let
        CleanedID = Text.Trim(channel_id),
        URL = "[https://www.googleapis.com/youtube/v3/search?part=snippet&channelId=](https://www.googleapis.com/youtube/v3/search?part=snippet&channelId=)" & CleanedID & "&order=date&type=video&maxResults=" & Number.ToText(MaxResults) & "&key=" & API_KEY,
        Source = Json.Document(Web.Contents(URL)),
        Items = Source[items],
        Videos = List.Transform(Items, each [
            ChannelID = CleanedID, // Add ChannelID for later join
            ChannelName = _[snippet][channelTitle],
            VideoTitle = _[snippet][title],
            PublishedAt = _[snippet][publishedAt],
            VideoDescription = _[snippet][description],
            VideoThumbnail = _[snippet][thumbnails][high][url],
            VideoID = _[id][videoId],
            VideoLink = "[https://www.youtube.com/watch?v=](https://www.youtube.com/watch?v=)" & _[id][videoId]
        ])
    in
        Videos,
    
    // --- FUNCTION TO GET VIDEO STATISTICS (PART 2: STATISTICS) ---
    GetVideoStatistics = (video_id as text) =>
    let
        URL = "[https://www.googleapis.com/youtube/v3/videos?part=statistics&id=](https://www.googleapis.com/youtube/v3/videos?part=statistics&id=)" & video_id & "&key=" & API_KEY,
        Source = Json.Document(Web.Contents(URL)),
        // Ensure items exist before accessing them
        Statistics = if List.Count(Source[items]) > 0 then Source[items]{0}[statistics] else null,
        Likes = if Statistics <> null and Record.HasFields(Statistics, {"likeCount"}) then Number.FromText(Statistics[likeCount]) else 0,
        Comments = if Statistics <> null and Record.HasFields(Statistics, {"commentCount"}) then Number.FromText(Statistics[commentCount]) else 0,
        Views = if Statistics <> null and Record.HasFields(Statistics, {"viewCount"}) then Number.FromText(Statistics[viewCount]) else 0
    in
        [Likes = Likes, Comments = Comments, Views = Views],
    
    // --- COMBINE ALL DATA ---
    // Retrieve video details for all specified channels
    AllVideoSnippets = List.Combine(List.Transform(ChannelIDs_List, each GetVideoDetails(_))),

    // Extract VideoIDs from each video snippet
    AllVideoIDs = List.Transform(AllVideoSnippets, each _[VideoID]),
    
    // Retrieve statistics for each VideoID
    AllVideoStats = List.Transform(AllVideoIDs, each GetVideoStatistics(_)),

    // Combine video snippets with their statistics by index
    MergedVideoList = List.Zip({AllVideoSnippets, AllVideoStats}),
    CombinedVideos = List.Transform(MergedVideoList, each Record.Combine(_)),

    // Convert to table and change data types
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
## 📌 M Script Explanation

This script operates in three main stages: retrieving basic video information, fetching video statistics, and then combining both into a complete table.

1. `GetVideoDetails` Function (Retrieving Basic Video Info)
   
This function is responsible for finding the latest videos from each specified channel.
   - **Data Extraction:**
     - **ChannelName:** Retrieves the name of the channel from the video.
     - **VideoTitle:** Retrieves the specific title of each video.
     - **PublishedAt:** Retrieves the date and time the video was uploaded.
     - **VideoDescription:** Retrieves the description text found below the video.
     - **VideoThumbnail:** Retrieves the URL of the high-resolution video thumbnail image.
     - **VideoID:** Retrieves the unique ID of the video, which will be used to fetch its statistics.
     - **VideoLink:** Creates a complete, clickable YouTube URL to watch the video.

3. `GetVideoStatistics` Function (Retrieving Video Statistics)
   
After obtaining the `VideoID` from the first function, this function is called for each video to retrieve its engagement data.
   - **Data Extraction:**
     - **Likes:** Retrieves the total number of likes on the video.
     - **Comments:** Retrieves the total number of comments.
     - **Views:** Retrieves the total number of video views.

4. Combination and Table Creation Process
   
This is the main part where all the collected data is consolidated.

   - **Data Collection:** First, the script runs the `GetVideoDetails` function for all channels and combines the results into one large list of all videos.
   - **Statistics Retrieval:** Next, the script extracts the `VideoID` from each video in the list, then calls the `GetVideoStatistics` function for each ID to get the likes, comments, and views data.
   - **Table Merging:** The `List.Zip` and `Record.Combine` functions are used to "match" the basic data of each video with its statistics, forming a single complete row of data.
   - **Final Table:** Finally, all the complete data is converted into a table format, and its data types are adjusted (e.g., numbers for Likes/Views and datetime for PublishedAt) to be ready for analysis and visualization.

---
## 🙍 About Me  

Hi, I'm **Ahmad Zaki Amani** 👋  

✨ I'm passionate about **Data Analytics** and **Business Intelligence**, focusing on building dashboards, creating data visualizations, and turning raw data into actionable insights.  

💡 This project is part of my portfolio, where I showcase skills in:  
- Data visualization & storytelling  
- Dashboard design (Power BI, Tableau)  
- Data transformation & analysis  
- Business Intelligence solutions  

📫 Let’s connect and collaborate!  

[![Gmail](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:ahmadzaki27.az@gmail.com) 
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ahmad-zaki-amani-ab091635b/)  

---
🌐 Languages: [English](#english-version) | [Bahasa Indonesia](#indonesian-version)

---
<a name="indonesian-version"></a>
# 📊 Detail Channel YouTube dengan Power BI + YouTube API

---
Proyek ini mendemonstrasikan cara mengintegrasikan **data Channel YouTube** ke dalam **Power BI** menggunakan **YouTube Data API v3** secara langsung.

Skrip Power Query M ini mengambil informasi terperinci seperti nama channel, deskripsi, jumlah subscriber, dan gambar profil untuk daftar channel yang telah ditentukan.

---

## 🚀 Cara Menggunakan

1. D**apatkan Kunci API Anda:**
   - Buat proyek baru di **[Google Cloud Console](https://console.cloud.google.com).**
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
   - Setiap panggilan ke `channels.list` dengan `part=snippet,statistics` menghabiskan **3 unit kuota**. Jadi, melacak 10 channel akan menghabiskan **30 unit kuota** setiap kali data diperbarui.
   - Setiap panggilan ke `search.list` dengan `part=snippet` menghabiskan **100 unit kuota**.
   - Setiap panggilan ke `videos.list` dengan `part=statistics` menghabiskan **2 unit kuota**.
   - Perhitungkan penggunaan kuota Anda saat menentukan `MaxVideosPerChannel` dan jumlah channel yang dilacak.
     
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
---
## 📌 Penjelasan M Script

Skrip ini bekerja dalam tiga tahap utama: mengambil informasi dasar video, mengambil statistik video, dan kemudian menggabungkan keduanya menjadi satu tabel yang utuh.

### 1. Fungsi `GetVideoDetails` (Mengambil Info Dasar Video)
Fungsi ini bertanggung jawab untuk mencari video-video terbaru dari setiap channel yang ditentukan.

* **Ekstraksi Data:**
    * **ChannelName:** Mengambil nama channel dari video.
    * **VideoTitle:** Mengambil judul spesifik dari setiap video.
    * **PublishedAt:** Mengambil tanggal dan waktu video tersebut diunggah.
    * **VideoDescription:** Mengambil teks deskripsi yang ada di bawah video.
    * **VideoThumbnail:** Mengambil URL gambar thumbnail video dalam resolusi tinggi.
    * **VideoID:** Mengambil ID unik dari video, yang akan digunakan untuk mencari statistiknya.
    * **VideoLink:** Membuat URL YouTube lengkap yang bisa diklik untuk menonton video tersebut.

### 2. Fungsi `GetVideoStatistics` (Mengambil Statistik Video)
Setelah mendapatkan `VideoID` dari fungsi pertama, fungsi ini dipanggil untuk setiap video guna mengambil data engagement-nya.

* **Ekstraksi Data:**
    * **Likes:** Mengambil jumlah "suka" (likes) pada video.
    * **Comments:** Mengambil jumlah total komentar.
    * **Views:** Mengambil jumlah total penayangan (views) video.

### 3. Proses Penggabungan dan Pembuatan Tabel
Ini adalah bagian utama di mana semua data yang telah dikumpulkan disatukan.

* **Pengumpulan Data:** Pertama, skrip menjalankan fungsi `GetVideoDetails` untuk semua channel dan menggabungkan hasilnya menjadi satu daftar besar berisi semua video.

* **Pengambilan Statistik:** Selanjutnya, skrip mengambil `VideoID` dari setiap video dalam daftar tersebut, lalu memanggil fungsi `GetVideoStatistics` untuk masing-masing ID guna mendapatkan data likes, comments, dan views.

* **Penggabungan Tabel:** Fungsi `List.Zip` dan `Record.Combine` digunakan untuk "mencocokkan" data dasar setiap video dengan data statistiknya, sehingga menjadi satu baris data yang lengkap.

* **Tabel Akhir:** Terakhir, semua data yang sudah lengkap diubah menjadi format tabel dan tipe datanya disesuaikan (misalnya, angka untuk Likes/Views dan tanggal untuk PublishedAt) agar siap untuk dianalisis dan divisualisasikan.
  
---

## 🙍 Tentang Saya

Halo, saya **Ahmad Zaki Amani** 👋

✨ Saya memiliki ketertarikan besar pada bidang **Data Analytics** dan **Business Intelligence**, khususnya dalam membangun dashboard, membuat visualisasi data, dan mengubah data mentah menjadi insight yang bermanfaat.

💡 Proyek ini merupakan bagian dari portofolio saya, yang menampilkan keterampilan dalam:

* Visualisasi data & storytelling
* Perancangan dashboard (Power BI, Tableau)
* Transformasi & analisis data
* Solusi Business Intelligence

📫 Mari terhubung dan berkolaborasi!

[![Gmail](https://img.shields.io/badge/Gmail-D14836?style=for-the-badge\&logo=gmail\&logoColor=white)](mailto:ahmadzaki27.az@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge\&logo=linkedin\&logoColor=white)](https://www.linkedin.com/in/ahmad-zaki-amani-ab091635b/)

---
