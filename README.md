default(1).conf
File
dockerfile(1)
File
index.php
File
docker-compose(1).yml
File
dokumentasi project belajar docker trial1.
buatkan readme.md dari project ini.
cek struktur filenya dan isi nya dulu.


Siap, Pak Indra. Saya sudah cek isi file yang tersedia dan struktur project dari screenshot.

Saya dokumentasikan berdasarkan kondisi Trial1 yang sekarang, termasuk:

Struktur docker/, src/, dan docs/
docker-compose.yml
docker/php/Dockerfile
docker/nginx/default.conf
src/index.php
Arsitektur Nginx → PHP-FPM → MariaDB
Konsep Docker network dan service name (app, db)
Bind mount vs named volume
Command docker compose
Development workflow
Penjelasan kapan perlu build --no-cache
Next step menuju Laravel dan production
Catatan mana yang masih khusus untuk learning project

Contohnya, saya jelaskan bahwa konfigurasi Nginx saat ini meneruskan PHP ke app:9000, sesuai konfigurasi actual project.

Docker Compose juga saya dokumentasikan sesuai konfigurasi actual, termasuk port 8080, MariaDB 11, volume db_data, dan phpMyAdmin 8081.

File index.php memang saat ini hanya menggunakan phpinfo(), jadi README tidak mengarang fungsi aplikasi yang belum ada.

README siap dipakai:

Download README.md 

README.md
Dokumen