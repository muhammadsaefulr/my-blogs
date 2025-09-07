---
title: Deploy Service Go Ke Vps Dengan Resource Kecil
published: 2025-09-07
tags: [CI/CD, Blogging, Deployment, Docker, Go]
category: Guides
draft: false
---

# Deploy Service Go ke VPS dengan Resource Terbatas menggunakan Docker dan GitHub Actions, Emang Bisa?

Sebagian developer sangat ingin membuat sesuatu, tapi kadang terkendala dengan resource server yang terbatas seperti dengan spesifikasi server 1GB Ram 1 Core CPU, betul kan? hehe.

Di sini, aku akan coba membahas cara mengakali keterbatasan tersebut agar service yang dibuat tetap bisa dijalankan di VPS, **tanpa harus membebani server** saat build. Caranya adalah dengan memanfaatkan **Docker Hub** untuk menyimpan image, dan **GitHub Actions** untuk otomatisasi build & push image.

<div align="center">
  <img src="/src/assets/images/content/Capture-Github-Actions.png" alt="Github Actions Screenshot" width="600"/>
</div>

Dengan pendekatan ini:

* Build service dilakukan di GitHub Actions (bukan di VPS), sehingga VPS tidak perlu resource besar.
* VPS hanya melakukan **pull image** terbaru dan menjalankannya menggunakan Docker.
* Proses deployment menjadi lebih cepat dan konsisten.

Blog ini akan memandumu langkah demi langkah, mulai dari setup project dan workflow dengan github action, membuat repository untuk meyimpan hasil build image di docker hub, sampai setting up worfklow untuk pulling image dari docker hub ke VPS.

tapi sebelum mulai, kamu mungkin perlu tau kenapa harus menggunakan github action? dan apa aja sih kekurangannya menggunakan github action ini?

## Kenapa Menggunakan GitHub Actions?

- Terintegrasi langsung dengan GitHub: Tidak perlu layanan CI/CD pihak ketiga. Semua bisa dijalankan langsung dari repo kamu, jadi kamu tidak perlu mengubah kodenya lagi di berbagai platform.

- Automatisasi penuh: Bisa otomatis build, test, push image, deploy ke server atau cloud.

- Workflow yang fleksibel: Bisa menentukan event trigger, branch, atau manual dispatch.

- Gratis untuk repositori publik: Menyediakan kuota cukup untuk project hobby mu atau sekedar untuk projek skala kecil dan menegah.

- Mudah dibagikan: Workflow bisa digunakan ulang atau dibagi ke repo lain jika orang tesebut memiliki akses ke repo mu.

- Banyak action dan tool yang tersedia: Github Telah Menyediakan Marketplace untuk berbagai tool dan action yang bisa digunakan.

- Notifikasi & monitoring: Bisa menerima notifikasi sukses/gagal langsung ke email yang terdaftar dari github mu dan memonitor log build.

## Kekurangan Menggunakan GitHub Actions

- Kuota terbatas untuk private repo: Gratisan untuk private repo punya batasan menit eksekusi per bulan.

- Kurang fleksibel untuk server internal: Deploy ke server di network lokal kadang perlu setup tambahan (SSH, firewall, dsb).

- Debugging bisa tricky: Log Actions cukup jelas, tapi jika ada error di step tertentu, kadang perlu trial & error.

- Tergantung GitHub: Kalau GitHub down atau ada maintenance, workflow akan berhenti sementara.

### Apa Yang Diperlukan Untuk Tutorial Kali Ini?

- Repository github yang sudah memiliki dockerfile / docker compose

- Akun Docker Hub

- Dan server / vps untuk deployment service

### Langsung Gass Kita Coba

Yang pertama pastikan kamu udah buat akun untuk docker hub, kalo belum bisa buatnya klik [disini](https://hub.docker.com/signup) ketika sudah berada di dashboard cari bagian repository dan klik `Create Repository` dan isikan nama repository nya lalu klik `Create Repository`.

setelah itu kita harus membuat secret docker hub, klik pada bagian avatar -? di bagian atas -> `Settings` -> `Secrets` dan klik `Personal Access Tokens` lalu klik `New Token` dan isikan nama token nya dan token yang dihasilkan, klik `Create Token` setelah token terbuat copy secret nya.

### Setting Up Environtment repo
Yang kedua disini kita harus set env secret di github, dari halaman projek repository mu klik `Settings And Variable` -> `Actions` -> `Repository Secrets` dan klik `New repository secret` lalu isikan secret berikut sesuai dengan yang kamu punya:

| Secret Name | Deskripsi |
|-------------------|------------------------------------------------------------|
| `DOCKER_USERNAME` | Username Docker Hub kamu |
| `DOCKER_SECRET` | Token / password Docker Hub (disarankan gunakan token) |
| `VPS_HOST` | IP atau hostname VPS |
| `VPS_PORT` | Port SSH untuk login VPS (contoh: `22`) |
| `VPS_USER` | User SSH untuk login VPS (contoh: `root`) |
| `VPS_SSH_KEY` | Private key untuk SSH (disimpan di GitHub Secrets) |

### Setting Up Workflow
Disini kita buat dulu workflow di github action untuk membuild docker compose kita untuk pushing result build image nya ke docker hub, dan pulling image ke instance/vps pastiin dulu kalo repo mu udah ada docker-compose.yml nya yaaa~

dari tampilan repositori mu sekarang klik `Actions` dibagian atas kanan -> `New Workflow` dan isikan kode  berikut:

**build.yml**
```yaml
name: Build and Push to Docker Hub

on:
  push:
    branches: ["main"]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: "1.24.2"

      - name: Cache Go modules
        uses: actions/cache@v3
        with:
          path: |
            ~/.cache/go-build
            ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-

      - name: Install dependencies
        run: go mod tidy

      - name: Log in to Docker Hub
        run: echo "${{ secrets.DOCKER_SECRET }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

      - name: Build Docker image
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/namarepo-dockerhubmu:latest .

      - name: Push Docker image
        run: docker push ${{ secrets.DOCKER_USERNAME }}/namarepo-dockerhubmu:latest

```

kode diatas adalah template workflow yang kita gunakan untuk build service kita ke docker image di instance github action yang nantinya akan di push ke docker hub sebagai media penyimpanan image dan share image antar platform.

oiya sebelum lanjut ke kode dibawah, pastikan kamu udah setting up repository project mu ke vps/instance dengan directory yang udah kamu tentukan dan pastikan juga kamu udah menginstall docker dan docker compose di server mu.

**pull.yml**

```yaml
name: Deploy via SSH

on:
  workflow_run:
    workflows: ["Build and Push to Docker Hub"]
    types:
      - completed

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          port: ${{ secrets.VPS_PORT }}
          script: |
            cd /root/yourprojectrepo

            # Pull Latest Code From Github
            git pull origin main

            # Pull latest image from Docker Hub
            docker compose pull

            # Stop and remove running containers
            docker compose down

            # Rebuild and start services with latest image
            docker compose up -d --remove-orphans
```

Seperti sebelumnya, kode diatas adalah template workflow yang kita gunakan untuk pulling image terbaru dari docker hub ke instance VPS yang nantinya akan di run dengan docker compose.

### Finishing Up

Nah akhirnya kita udah sampai ke proses terakhir, disini kita bisa coba update repo mu untuk melihat hasil build dan deployment yang udah kita buat tadi sambil lihat docker logs nya jikalau ada yang belum sesuai bisa di perbaiki kembali :)

Kalau semua step udah jalan mulus, berarti kamu sekarang udah punya alur **CI/CD sederhana** yang bisa dipakai bahkan di VPS dengan resource terbatas 🎉. 

Langkah selanjutnya, kamu bisa:
- Menambahkan service lain ke dalam `docker-compose.yml`.
- Setup notifikasi (misalnya ke Discord/Telegram) untuk tahu kalau deploy sukses/gagal.
- Optimasi Docker image supaya lebih kecil dan cepat ditarik di server.

Dengan cara ini, kamu nggak perlu lagi build langsung di VPS, cukup push code ke GitHub dan biarkan workflow yang urus deploymentmu :)

Akhir kata, Jikalau ada kesalahan atau ada yang ingin di tanyakan pada blog ini jangan ragu untuk menghubungi ku via dm di social media yang udah ku tampilkan, sekian :)
