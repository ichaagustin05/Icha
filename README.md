KELOMPOK 4	
Nama Anggota	: 1. Icha Agustin			(09030182327003)
		          2. Marshanda				(09030282327048)
		          3. Atiya Melvina			(09030282327069)
		          4. Ivan Saputra			(09030282327072)
		          5. Citra Putri Firdawati	(09030582327105)
Kelas		: TK5A
Mata Kuliah	: Keamanan Komputer
_____________________________________________________________________________________

# Project
Kotak Sampah Pintar Otomatis 

# Deskripsi Project
Proyek ini mengembangkan kotak sampah otomatis berbasis sensor, kemudian melakukan simulasi serangan DoS untuk melihat bagaimana perangkat IoT bereaksi terhadap beban trafik yang tinggi. Seluruh proses pengujian dilakukan secara legal dalam lingkungan laboratorium dan jaringan tertutup, sehingga aman untuk penelitian.

1. Kotak Sampah Pintar
Pada bagian ini dibuat sebuah perangkat kotak sampah yang dapat membuka dan menutup secara otomatis menggunakan sensor jarak (ultrasonic). Sistem ini dirancang agar lebih higienis dan memudahkan pengguna

2. Pengujiuan Serangan ( DoS )
Sebagai bagian dari analisis keamanan, dilakukan pengujian berupa simulasi gangguan lalu lintas jaringan. Tujuan dari pengujian ini adalah untuk melihat bagaimana sistem bereaksi ketika menerima trafik berlebih.

# Serangan Dos
Serangan DoS (Denial-of-Service) adalah upaya jahat untuk membuat suatu sistem, situs web, atau layanan online menjadi tidak tersedia atau lumpuh bagi pengguna yang sah dengan cara membanjiri HTTP target dengan lalu lintas atau permintaan yang sangat banyak dari satu sumber tunggal, sehingga sumber daya target habis dan fungsionalitasnya terganggu parah hingga tidak bisa diakses sama sekali

# Impack Serangan Dos
- Membuat Web Down
- Wifi Lost
- Sensor Tidak bisa terdeteksi

# Cara Instalasi serangan DoS menggunakan tools hping3 
- Pertama kami menginstall hping3 menggunakan command : sudo apt install hping3 -y
- Kemudian setelah di install kami cek version menggunakan command : hping3 --version 

Cara Menjalankan Serangan DoS Tools hping3 menggunakan command : hping3 -c 20000 -d 120 -S -w 64 -p 80 --flood --rand-source [IP]

# tools untuk nyerangnya : hping3
Hping3 tools untuk menyerang nama serangannya dari hping3 flooding/synflood attack yaitu membanjiri data. 

# mitigasi : evillimiter
mitigasi serangan Dos, menggunakan tools evillimiter sebagai alat untuk memblokir/membatasi bandwitch trafic.

# Alur Kerja evilimiter
- Periksa versi evillimiter dengan command sudo evillimiter --version
- Kemudian masuk ke direktori evillimiter dengan command cd evillimiter
- Untuk menjalankan evillimiter gunakan command evillimiter
- Setelah masuk ke evillimiter (Main) >>> buatkan command scan. Untuk perintah ini akan menampilkan ada beberapa hosts yang aktif atau terhubung ke jaringan yang sama.
- Untuk melihat daftar hosts gunakan command hosts, maka akan muncul daftar hosts aktif atau terhubung beserta ID, IP dan lain - lain. 
- Dari daftar hosts yang ditampilkan, cari IP yang mencurigakan atau terindikasi melakukan serangan.
- Untuk memblokir penyerangkan dengan menggunakan command block (ID/IP), penyerang akan memblokir sehingga tidak dapat mengirimkan paket baru ke ke jaringan.
catatan : Evillimiter hanya membatasi atau memblokir, bukan membersihkan sisa paket serangan.

# pencegahan : Suricata
Suricata adalah sistem deteksi dan pencegahan intrusi (IDS/IPS) yang bersifat open-source dan dirancang untuk memantau lalu lintas jaringan. IPS (Intrusion Prevention System) adalah sistem yang dirancang untuk mendeteksi dan mencegah ancaman atau serangan yang dapat merusak jaringan atau sistem komputer. IPS berfungsi untuk memantau lalu lintas jaringan secara real-time, menganalisis pola lalu lintas tersebut, dan mengambil tindakan pencegahan jika terdeteksi adanya potensi ancaman. Adapun fungsi ips Fungsi IPS Mencegah serangan masuk sebelum menyebabkan kerusakan, Meningkatkan keamanan dengan monitoring real-time, Melindungi dari DoS dengan memblokir trafik tidak sah, Mengurangi risiko kerusakan karena ancaman dihentikan lebih awal, Memenuhi kebijakan keamanan dengan memblokir aktivitas ilegal/berbahaya.

# Cara Kerja Surricata menggunakan IPS (Intrusion Prevention System)
- Memantau dan menganalisis lalu lintas jaringan
- Deteksi Anomali Memeriksa lalu lintas jaringan dan mencari aktivitas yang tidak normal.
- Signature Detection  Mencocokkan pola serangan yang sudah dikenal (misal DoS, SQLi).
- Pencegahan Otomatis Memblokir IP/koneksi yang mencurigakan.

  










