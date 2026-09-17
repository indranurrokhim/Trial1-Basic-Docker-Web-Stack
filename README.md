# Trial 1 --- Basic Docker Web Stack

Project ini adalah project belajar Docker untuk memahami cara
menjalankan aplikasi PHP menggunakan beberapa container yang saling
terhubung.

Stack yang digunakan:

-   **PHP 8.4-FPM** sebagai application runtime
-   **Nginx** sebagai web server
-   **MariaDB 11** sebagai database
-   **phpMyAdmin** sebagai database management interface
-   **Composer 2** tersedia di container PHP

## 1. Struktur Project

``` text
trial1/
├── docker/
│   ├── nginx/
│   │   └── default.conf
│   └── php/
│       └── Dockerfile
├── docs/
├── src/
│   ├── index.php
│   └── test.php
├── .gitignore
├── docker-compose.yml
└── README.md
```

### Fungsi masing-masing bagian

  -----------------------------------------------------------------------
  File / Directory                    Fungsi
  ----------------------------------- -----------------------------------
  `docker/`                           Menyimpan konfigurasi Docker

  `docker/php/Dockerfile`             Membuat image PHP-FPM custom

  `docker/nginx/default.conf`         Konfigurasi Nginx dan koneksi ke
                                      PHP-FPM

  `src/`                              Source code aplikasi PHP

  `src/index.php`                     Halaman utama untuk testing PHP

  `src/test.php`                      File testing PHP

  `docs/`                             Dokumentasi pembelajaran project

  `docker-compose.yml`                Mendefinisikan seluruh
                                      service/container

  `.gitignore`                        File yang tidak perlu disimpan di
                                      Git

  `README.md`                         Dokumentasi project
  -----------------------------------------------------------------------

> Isi `docs/`, `.gitignore`, dan `src/test.php` tidak termasuk dalam
> file yang diperiksa pada saat dokumentasi ini dibuat, sehingga README
> ini tidak mengasumsikan isinya.

## 2. Arsitektur

Project ini menggunakan 4 service:

``` text
Browser
   │
   │ HTTP :8080
   ▼
┌──────────────┐
│    Nginx     │
│     web      │
└──────┬───────┘
       │ FastCGI :9000
       ▼
┌──────────────┐
│  PHP-FPM     │
│     app      │
└──────┬───────┘
       │
       │ MySQL/MariaDB
       ▼
┌──────────────┐
│   MariaDB    │
│      db      │
└──────────────┘

Browser
   │
   │ HTTP :8081
   ▼
┌──────────────┐
│  phpMyAdmin  │
└──────┬───────┘
       │
       ▼
     MariaDB
```

Hal penting dari arsitektur ini adalah **Nginx dan PHP-FPM berada di
container berbeda**.

Nginx menerima HTTP request, kemudian meneruskan request PHP ke service
`app` melalui:

``` text
app:9000
```

Konfigurasi tersebut terdapat di `docker/nginx/default.conf`.

## 3. Docker Compose

`docker-compose.yml` mendefinisikan empat service:

### `app`

``` yaml
app:
  build:
    context: .
    dockerfile: docker/php/Dockerfile
  volumes:
    - ./src:/var/www/html
```

`app` menggunakan Dockerfile sendiri untuk membuat PHP 8.4-FPM.

Source code lokal:

``` text
./src
```

di-mount ke:

``` text
/var/www/html
```

di dalam container.

### `web`

``` yaml
web:
  image: nginx:latest
  ports:
    - "8080:80"
```

Nginx menggunakan official image `nginx:latest`.

Port:

``` text
Host :8080 → Container :80
```

Konfigurasi Nginx dari host juga di-mount ke container:

``` text
./docker/nginx/default.conf
→ /etc/nginx/conf.d/default.conf
```

### `db`

``` yaml
db:
  image: mariadb:11
```

Database menggunakan MariaDB 11.

Credential untuk project belajar:

``` text
Database : trial1
Username : trial1
Password : trial1
Root     : root
```

Data database disimpan pada named volume:

``` yaml
db_data:
  /var/lib/mysql
```

Dengan demikian, data database tidak hilang hanya karena container `db`
dibuat ulang.

> Credential di atas hanya cocok untuk project belajar. Untuk
> production, gunakan secret/environment variable yang aman dan jangan
> commit credential ke Git.

### `phpmyadmin`

``` yaml
phpmyadmin:
  image: phpmyadmin:latest
  environment:
    PMA_HOST: db
  ports:
    - "8081:80"
```

phpMyAdmin dapat mengakses MariaDB menggunakan nama service Docker:

``` text
db
```

Port:

``` text
Host :8081 → Container :80
```

## 4. Dockerfile PHP

Isi `docker/php/Dockerfile`:

``` dockerfile
FROM composer/composer:2-bin AS composer

FROM php:8.4-fpm
# FROM php:7.2-fpm

COPY --from=composer /composer /usr/bin/composer

RUN docker-php-ext-install pdo_mysql

WORKDIR /var/www/html
```

### Penjelasan

#### PHP-FPM

``` dockerfile
FROM php:8.4-fpm
```

Container menggunakan PHP 8.4 dengan PHP-FPM.

PHP-FPM tidak menerima HTTP secara langsung seperti web server. PHP-FPM
menerima request dari Nginx melalui FastCGI.

#### Composer

``` dockerfile
FROM composer/composer:2-bin AS composer
```

Composer diambil dari image Composer.

Kemudian binary Composer disalin ke image PHP:

``` dockerfile
COPY --from=composer /composer /usr/bin/composer
```

Artinya container `app` nantinya mempunyai command:

``` bash
composer
```

#### MySQL Driver

``` dockerfile
RUN docker-php-ext-install pdo_mysql
```

Extension `pdo_mysql` dipasang agar PHP dapat berkomunikasi dengan
MySQL/MariaDB menggunakan PDO.

#### Working Directory

``` dockerfile
WORKDIR /var/www/html
```

Directory kerja PHP di dalam container adalah:

``` text
/var/www/html
```

## 5. Konfigurasi Nginx

File:

``` text
docker/nginx/default.conf
```

menggunakan:

``` nginx
root /var/www/html;
index index.php index.html;
```

Source code yang dilihat Nginx berasal dari directory tersebut.

Untuk request:

``` text
/
```

Nginx menggunakan:

``` nginx
try_files $uri $uri/ /index.php?$query_string;
```

Sedangkan file PHP diproses menggunakan:

``` nginx
location ~ \.php$ {
    try_files $uri =404;

    fastcgi_pass app:9000;
}
```

`app` bukan IP address. `app` adalah **service name Docker Compose**.

Docker menyediakan internal DNS sehingga container `web` dapat menemukan
container PHP menggunakan:

``` text
app
```

## 6. Source Code

`src/index.php` digunakan sebagai test sederhana:

``` php
<?php
phpinfo();
```

`phpinfo()` menampilkan informasi PHP yang berjalan di dalam container.

Ini berguna untuk memastikan bahwa:

1.  Nginx berjalan.
2.  Request PHP diteruskan ke PHP-FPM.
3.  PHP berjalan di dalam container.
4.  Versi PHP yang aktif dapat diperiksa.
5.  Extension PHP yang ter-install dapat diperiksa.

## 7. Menjalankan Project

Pastikan Docker dan Docker Compose sudah ter-install.

Dari directory project:

``` bash
cd trial1
```

Build image dan menjalankan seluruh service:

``` bash
docker compose up -d --build
```

Periksa container:

``` bash
docker compose ps
```

Melihat log:

``` bash
docker compose logs
```

Log service tertentu:

``` bash
docker compose logs app
docker compose logs web
docker compose logs db
docker compose logs phpmyadmin
```

## 8. Mengakses Project

Web PHP:

``` text
http://localhost:8080
```

phpMyAdmin:

``` text
http://localhost:8081
```

Untuk phpMyAdmin, database server menggunakan:

``` text
Server   : db
Username : trial1
Password : trial1
Database : trial1
```

## 9. Mengecek Container

Melihat container yang sedang berjalan:

``` bash
docker ps
```

Masuk ke container PHP:

``` bash
docker compose exec app bash
```

Kemudian:

``` bash
php -v
composer --version
php -m
```

Keluar:

``` bash
exit
```

Masuk ke MariaDB:

``` bash
docker compose exec db mariadb -u trial1 -p
```

Password:

``` text
trial1
```

## 10. Menghentikan Project

Stop container tanpa menghapus container:

``` bash
docker compose stop
```

Menjalankan kembali:

``` bash
docker compose start
```

Atau:

``` bash
docker compose up -d
```

Menghentikan dan menghapus container/network:

``` bash
docker compose down
```

### Menghapus database volume

Hati-hati, command berikut juga menghapus data MariaDB:

``` bash
docker compose down -v
```

Karena project menggunakan:

``` yaml
volumes:
  db_data:
```

maka `-v` akan menghapus named volume tersebut.

## 11. Konsep yang Dipelajari

Project `trial1` digunakan untuk memahami konsep dasar:

### Container

Setiap service berjalan sebagai container terpisah:

``` text
web
app
db
phpmyadmin
```

### Image

Container dibuat berdasarkan image.

Contoh:

``` text
nginx:latest
mariadb:11
phpmyadmin:latest
php:8.4-fpm
```

Untuk `app`, image dibuat sendiri menggunakan Dockerfile.

### Volume

Source code menggunakan bind mount:

``` text
./src:/var/www/html
```

Database menggunakan named volume:

``` text
db_data:/var/lib/mysql
```

Keduanya mempunyai tujuan berbeda:

``` text
Bind Mount
→ source code development

Named Volume
→ persistent application data
```

### Network

Docker Compose membuat network internal untuk service.

Karena itu service dapat berkomunikasi menggunakan nama:

``` text
app
db
```

bukan menggunakan `localhost`.

Contoh:

``` text
Nginx → app:9000
phpMyAdmin → db
```

## 12. Alur Request

Ketika browser membuka:

``` text
http://localhost:8080
```

alur sederhananya:

``` text
Browser
   ↓
Host port 8080
   ↓
Nginx container
   ↓
PHP request
   ↓
app:9000
   ↓
PHP-FPM
   ↓
index.php
```

Jika PHP membutuhkan database, aplikasi PHP nantinya dapat menggunakan:

``` text
db
```

sebagai hostname database.

**Jangan menggunakan `localhost` untuk mengakses MariaDB dari container
PHP**, karena `localhost` berarti container PHP itu sendiri.

## 13. Kenapa Tidak Semua Digabung dalam Satu Container?

Salah satu konsep penting dari project ini adalah separation of
concerns.

Daripada membuat:

``` text
1 container
├── Nginx
├── PHP
└── MariaDB
```

project menggunakan:

``` text
container web
container app
container db
container phpmyadmin
```

Keuntungannya:

-   Setiap service mempunyai tanggung jawab jelas.
-   Image dapat di-update secara terpisah.
-   Scaling lebih mudah.
-   Maintenance lebih mudah.
-   Architecture lebih dekat dengan praktik Docker modern.

## 14. Development Workflow

Untuk development sederhana:

``` text
Windows / VS Code
       │
       │ edit source
       ▼
   ./src
       │
       │ bind mount
       ▼
  PHP container
```

Perubahan file di `src` langsung terlihat oleh container karena
menggunakan:

``` yaml
- ./src:/var/www/html
```

Sehingga untuk perubahan sederhana pada PHP, tidak perlu rebuild image
setiap kali mengubah source code.

Rebuild diperlukan ketika mengubah hal seperti:

``` text
Dockerfile
PHP extension
base image
system package
dependency yang dibangun ke image
```

## 15. Next Step

Setelah konsep dasar `trial1` dipahami, project dapat dikembangkan
bertahap:

1.  PHP terhubung ke MariaDB.
2.  Membuat database/table.
3.  Menggunakan Composer.
4.  Membuat project Laravel.
5.  Memisahkan environment `development` dan `production`.
6.  Menggunakan `.env`.
7.  Mengelola database migration.
8.  Menggunakan Git/GitHub.
9.  Membuat deployment workflow.
10. Menjalankan beberapa aplikasi pada satu Docker host.

------------------------------------------------------------------------

## Catatan

Project ini adalah **learning project**, bukan production configuration.

Beberapa konfigurasi sengaja dibuat sederhana agar konsep Docker lebih
mudah dipelajari, terutama:

-   credential database ditulis langsung di Compose.
-   image menggunakan tag `latest` pada Nginx dan phpMyAdmin.
-   source code menggunakan bind mount.
-   belum menggunakan secrets.
-   belum menggunakan reverse proxy/domain.
-   belum menggunakan HTTPS.
-   belum ada healthcheck.
-   belum ada production-specific configuration.
