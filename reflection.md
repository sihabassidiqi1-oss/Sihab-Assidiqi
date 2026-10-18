# Refleksi dan Jawaban Soal Analisis - Pertemuan 2

> Catatan: nomor dan urutan soal disesuaikan dengan modul bagian 69.

## 1. Siapa yang login dan akun mana yang aktif?
Perintah `aws sts get-caller-identity` menunjukkan principal `user/cc-p2-sihab`,
yaitu IAM user, bukan root. Account ID disensor dalam laporan.
Memakai IAM user menjaga agar akun root tidak dipakai untuk pekerjaan harian.

## 2. Komponen jaringan apa yang dibutuhkan agar VM bisa diakses dari internet?
Di AWS, subnet baru tidak otomatis punya jalur ke internet (berbeda dengan Azure).
Dibutuhkan: VPC, subnet, Internet Gateway yang di-attach ke VPC, route table
dengan default route 0.0.0.0/0 ke IGW yang diasosiasikan ke subnet, public IP pada
instance, dan Security Group yang mengizinkan port yang dipakai.

## 3. Apa fungsi Security Group dan apa yang terjadi saat port belum dibuka?
Security Group adalah firewall virtual pada level instance. Saat port 80 belum
dibuka, `curl http://127.0.0.1/` di dalam VM berhasil (HTTP 200), tetapi
`curl` dan browser dari laptop timeout (`ERR_CONNECTION_TIMED_OUT`).
Artinya paket dari luar dibuang oleh Security Group sebelum sampai ke NGINX.
Setelah rule port 80 ditambahkan, akses dari luar berhasil. Setelah rule dicabut,
akses timeout lagi. Evidence: `09-nginx.txt`, `09-nginx-before.png`, `09-nginx-after.png`.

## 4. Bagaimana membedakan masalah di service dan masalah di jaringan?
Uji dari dua sisi. Dari dalam VM (`systemctl status nginx`, `ss -tulpen`,
`curl localhost`) membuktikan service sehat dan listen di port 80. Dari luar yang timeout
menunjukkan masalah ada di jalur jaringan atau firewall, bukan di aplikasi.
Timeout (tanpa balasan) biasanya menandakan paket diblokir, sedangkan
"connection refused" menandakan service tidak listen.

## 5. Kenapa SSH dibatasi ke satu IP dan bagaimana key dikelola?
Membuka port 22 ke 0.0.0.0/0 membuat server diserang scan dan brute force dari seluruh
internet. Rule dibatasi `<IP-publik-PC>/32` (least privilege). Autentikasi memakai key
ed25519: hanya public key yang diunggah ke AWS, private key tetap di PC, tidak masuk Git,
dan dihapus setelah praktikum selesai. Konsekuensinya, saat IP PC berubah, SSH timeout
sampai rule diperbarui (lihat soal 7).

## 6. Apa yang terjadi pada biaya saat VM di-stop?
Saat di-stop, biaya komputasi instance berhenti, tetapi volume EBS (8 GB gp2)
tetap ada dan tetap ditagih. Evidence `12-stop-volume.txt` menunjukkan instance
`stopped` sementara volume masih `in-use`/attached. Public IP dinamis juga berubah
setelah stop/start (54.169.245.139 menjadi 54.151.135.213 lalu 52.221.228.194).
Biaya benar-benar nol hanya jika resource di-terminate/dihapus. Volume di praktikum
ini `DeleteOnTermination=True`, sehingga ikut terhapus saat instance di-terminate.
Elastic IP dan public IPv4 juga bisa ditagih jika dibiarkan.

## 7. Ceritakan satu kasus troubleshooting (Masalah, Evidence, Hipotesis, Uji, Perbaikan, Verifikasi)
- **Masalah:** SSH ke VM timeout setelah berpindah jaringan.
- **Evidence:** `ssh: connect to host ... port 22: Connection timed out`.
- **Hipotesis:** rule Security Group port 22 tidak mengizinkan IP publik PC yang baru.
- **Uji:** membandingkan IP PC (`checkip.amazonaws.com`) dengan rule SG: berbeda.
- **Perbaikan:** mencabut rule lama dan menambah rule untuk IP baru dengan /32.
- **Verifikasi:** SSH berhasil, `whoami` menampilkan `ubuntu`.
Evidence lengkap: `10-troubleshoot.txt`. Pelajaran: kegagalan koneksi sebaiknya dilacak
dari bukti (IP, rule, log) dan bukan ditebak.

## 8. Hubungkan praktikum dengan lima karakteristik cloud menurut NIST
- **On-demand self-service:** VPC sampai VM dibuat sendiri lewat CLI tanpa menghubungi penyedia.
- **Broad network access:** server diakses lewat internet standar (SSH dan HTTP) dari PC.
- **Resource pooling:** instance berjalan di infrastruktur AWS bersama yang dibagi antarpelanggan;
  lokasinya hanya dipilih di tingkat region dan AZ.
- **Rapid elasticity:** VM dapat dibuat, di-stop, dan dihapus dalam hitungan menit,
  serta jenis dan jumlahnya bisa diubah sesuai kebutuhan.
- **Measured service:** pemakaian dan biaya terukur (jam instance, EBS, IPv4), sehingga
  perlu cleanup agar tidak ada tagihan.

## 9. Apa hasil observasi IAM dan apakah izinnya terlalu luas?
Principal: `user/cc-p2-sihab`. Policy: `AmazonEC2FullAccess`. Perintah
`iam list-attached-user-policies` ditolak (AccessDenied), dan `ssm get-parameter`
juga ditolak saat mengambil AMI, sehingga AMI dicari dengan `ec2 describe-images`.
Scope: seluruh aksi EC2/VPC di akun, tidak mencakup IAM atau SSM. Ini lebih sempit dari
`AdministratorAccess`, tetapi untuk produksi masih bisa dipersempit (misalnya dibatasi
region atau resource ber-tag). Penolakan tersebut menunjukkan prinsip least privilege bekerja.
Evidence: `11-iam.txt`.

## 10. Mengapa cleanup harus berurutan dan bagaimana membuktikannya?
Resource saling bergantung, sehingga urutannya: terminate instance, hapus key pair dan
Security Group, disassociate dan hapus route table, detach dan hapus Internet Gateway,
hapus subnet, terakhir hapus VPC. Menghapus VPC lebih dulu akan gagal dengan
`DependencyViolation`. Pembuktian lewat CLI (`describe-vpcs`, `describe-instances`,
`describe-volumes`, `describe-addresses` semuanya kosong) dan Console (instance berstatus
`Terminated`). Evidence: `14-cleanup.txt`, `14-cleanup-ec2.png`.

## Refleksi pribadi
Kesulitan terbesar adalah memahami bahwa server, jaringan, dan firewall adalah
komponen terpisah yang harus dihubungkan sendiri. Kesalahan kecil (variabel shell yang
hilang saat terminal ditutup, IP publik yang berubah, izin IAM yang tidak mencukupi)
mengajarkan pentingnya menyimpan state (`vars.sh`), memeriksa kondisi sebelum lanjut,
dan membaca pesan error dengan teliti. Praktik ini memperjelas bahwa cloud mudah
dibuat tetapi harus dikelola, diamankan, dan dibersihkan dengan disiplin.