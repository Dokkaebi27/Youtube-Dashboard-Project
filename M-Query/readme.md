# 📊 YouTube Channel Details with Power BI + YouTube API
Proyek ini mendemonstrasikan cara mengintegrasikan data Channel YouTube ke dalam Power BI menggunakan YouTube Data API v3 secara langsung.

Skrip Power Query M ini mengambil informasi terperinci seperti nama channel, deskripsi, jumlah subscriber, dan gambar profil untuk daftar channel yang telah ditentukan.

🚀 Cara Menggunakan
Dapatkan Kunci API Anda:

Buat proyek baru di Google Cloud Console.

Aktifkan YouTube Data API v3 untuk proyek tersebut.

Buat kredensial dan salin Kunci API Anda.

Siapkan Parameter di Power BI (Direkomendasikan):

Buka Power Query Editor di Power BI (Home > Transform data).

Buka Manage Parameters > New Parameter.

Buat parameter bernama Key (Type: Text) dan masukkan Kunci API YouTube Anda sebagai nilainya.

Buat parameter lain bernama ChannelIDs_Text (Type: Text) dan masukkan daftar ID Channel yang ingin Anda lacak, dipisahkan dengan koma (contoh: UCoIiiHof6BJ85PLuLkuxuhw,UCn7Ohm5Dv7uKdODpjVo0xSQ).

Jalankan Kueri:

Di Power Query Editor, buat Kueri Kosong Baru (New Blank Query).

Buka Advanced Editor.

Salin salah satu Skrip M di bawah ini dan tempelkan ke dalam editor.

Jika Anda tidak mengatur parameter, ganti placeholder API_KEY dan daftar ChannelIDs secara langsung di dalam skrip.

Klik Selesai lalu Tutup & Terapkan untuk memuat data.

🔑 Catatan & Konfigurasi
Kuota API:

YouTube Data API memiliki kuota penggunaan harian gratis (biasanya 10.000 unit per hari).

Setiap panggilan ke channels.list dengan part=snippet,statistics menghabiskan 3 unit kuota. Jadi, melacak 10 channel akan menghabiskan 30 unit kuota setiap kali data diperbarui.

ID Channel:

Pastikan Anda menggunakan ID Channel unik (biasanya diawali dengan UC...), bukan nama pengguna (@handle) atau URL kustom.

Kustomisasi:

Anda dapat mengambil data tambahan (seperti total video atau total views) dengan mengubah parameter part dalam URL API dan menyesuaikan langkah-langkah parsing di dalam skrip.

Skrip M
M Script (Versi Dasar dengan Nilai Hardcode)
Versi ini paling mudah digunakan. Cukup ganti Kunci API dan daftar ID Channel langsung di dalam kode.
```
let
    // --- KONFIGURASI (Mengambil dari Parameter Power BI) ---
    API_KEY = Key,
    ChannelIDs_Text = ChannelIDs_Text,

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
