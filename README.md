# Repositori Laporan Praktikum Workshop DevOps

<p align="center">
  <img src="uploads/pens-logo.png" alt="Logo Politeknik Elektronika Negeri Surabaya" width="160" />
</p>

<p align="center">
  <strong>POLITEKNIK ELEKTRONIKA NEGERI SURABAYA</strong><br>
  Departemen Teknik Informatika dan Komputer<br>
  Program Studi Sarjana Terapan Teknik Informatika<br>
  Tahun Ajaran 2026 / 2027
</p>

---

## 📌 Informasi Mata Kuliah & Kelompok

- **Mata Kuliah:** Workshop DevOps (DevSecOps)
- **Dosen Pengampu:** Dr. Ferry Astika Saputra, S.T., M.Sc.
- **Kelompok:** 5 (Lima)

### 👥 Anggota Kelompok:

| No | Nama Mahasiswa | NRP | Peran / Fokus |
|:--:|:---|:---:|:---|
| 1 | **Muhammad Fajrul Fatih Abul ‘Ilmi** | `3124600040` | Mahasiswa D4 Teknik Informatika |
| 2 | **Nur Aini Agusthina** | `3124600050` | Mahasiswa D4 Teknik Informatika |
| 3 | **Abdullah Falich** | `3124600059` | Mahasiswa D4 Teknik Informatika |

---

## 📚 Daftar Isi Laporan Praktikum

Seluruh laporan di bawah ini telah dikonversi secara lengkap ke dalam format GitHub Markdown dengan bukti screenshot, tabel konfigurasi, serta blok kode perintah terstruktur:

| Modul | File Laporan Markdown | Topik Utama & Ruang Lingkup | Folder Gambar |
|:--:|:---|:---|:---:|
| **01** | [Praktikum 01: Fondasi Teoritis dan Kerangka Kerja DevSecOps](praktikum-01.md) | Baseline lab DevSecOps, pemeriksaan versi environment (`docker`, `compose`, `git`, `openssl`), security options Docker. | [`uploads/praktikum-01/`](uploads/praktikum-01/) |
| **02** | [Praktikum 02: Konsep Container dan Instalasi Docker](praktikum-02.md) | Instalasi Docker Engine Linux, privilege non-root (`docker group`), testing image `hello-world`, interaktif Nginx & Ubuntu, serta custom Dockerfile image. | [`uploads/praktikum-02/`](uploads/praktikum-02/) |
| **03** | [Praktikum 03: Docker Network, Volume, Bind Mount, Tmpfs, dan Compose](praktikum-03.md) | User-defined bridge network, manipulasi Docker Named Volume, persistensi data dengan multikontainer Alpine, dan pengujian sinkronisasi volume. | [`uploads/praktikum-03/`](uploads/praktikum-03/) |
| **04** | [Praktikum 04: Web Service Container](praktikum-04.md) | Reverse proxy Nginx dengan TLS self-signed sertifikat, backend Apache HTTPD statis, aplikasi Python Flask dengan healthcheck, Docker Compose stack. | [`uploads/praktikum-04/`](uploads/praktikum-04/) |
| **05** | [Praktikum 05: Database Service di Docker PostgreSQL & pgAdmin](praktikum-05.md) | Orkestrasi database PostgreSQL 16 Alpine, Web UI pgAdmin 4 otomatis (`servers.json`), skrip otomasi backup database (`backup.sh`), skrip uji restore (`restore-test.sh`). | [`uploads/praktikum-05/`](uploads/praktikum-05/) |
| **06** | [Praktikum 06: Centralized Logging dengan Fluent Bit dan PostgreSQL](praktikum-06.md) | Centralized logging pipeline: Log stream container -> Fluent Bit collector -> PostgreSQL JSONB storage (`01-init-logging.sh`, `analysis.sql`), uji retensi buffer recovery (`test-buffering.sh`). | [`uploads/praktikum-06/`](uploads/praktikum-06/) |
| **07** | [Praktikum 07: Monitoring Resource dengan Prometheus, cAdvisor, Node Exporter, dan Grafana](praktikum-07.md) | Full observability monitoring stack: Prometheus time-series scraper, cAdvisor container metrics, Node Exporter host metrics, Grafana dashboard provisioning, unit test rules (`rules.test.yml`), cross-stack Bab 6 bridge (`connect-bab6.sh`). | [`uploads/praktikum-07/`](uploads/praktikum-07/) |
| **Proxmox** | [Praktikum Virtualisasi: Konfigurasi Virtualisasi Proxmox VE](praktikum-proxmox.md) | Topologi hierarkis jaringan lab PENS, identifikasi router core RB1100, switch CSS326, subnetting 10 router kelompok MikroTik RB3011, instalasi VM Ubuntu, pembuatan Linux Container (LXC) di Proxmox VE. | [`uploads/praktikum-proxmox/`](uploads/praktikum-proxmox/) |

---

## 📁 Struktur Direktori Repositori

```text
laporan-github/
├── README.md                      # Index utama dan ringkasan seluruh praktikum
├── praktikum-01.md                # Praktikum 01: Fondasi DevSecOps
├── praktikum-02.md                # Praktikum 02: Konsep Container & Docker
├── praktikum-03.md                # Praktikum 03: Docker Network, Volume, Compose
├── praktikum-04.md                # Praktikum 04: Web Service Container
├── praktikum-05.md                # Praktikum 05: Database PostgreSQL & pgAdmin
├── praktikum-06.md                # Praktikum 06: Centralized Logging Fluent Bit
├── praktikum-07.md                # Praktikum 07: Monitoring Prometheus & Grafana
├── praktikum-proxmox.md           # Praktikum Konfigurasi Virtualisasi Proxmox VE
└── uploads/                       # Direktori penyimpanan aset visual / screenshot
    ├── pens-logo.png              # Logo resmi PENS untuk cover dokumen
    ├── praktikum-01/              # Screenshot bukti praktikum 01
    ├── praktikum-02/              # Screenshot bukti praktikum 02
    ├── praktikum-03/              # Screenshot bukti praktikum 03
    ├── praktikum-04/              # Screenshot bukti praktikum 04
    ├── praktikum-05/              # Screenshot bukti praktikum 05
    ├── praktikum-06/              # Screenshot bukti praktikum 06
    ├── praktikum-07/              # Screenshot bukti praktikum 07
    └── praktikum-proxmox/         # Screenshot bukti konfigurasi Proxmox & WinBox
```

---

## 🛠️ Ringkasan Teknologi yang Digunakan

- **Virtualization & Bare-metal:** Proxmox Virtual Environment (PVE), KVM Virtual Machine, LXC Container, MikroTik RouterOS & SwOS.
- **Containerization & Orchestration:** Docker Engine, Docker Compose V2, Alpine Linux, Nginx, Apache HTTPD, Python Flask.
- **Data Persistence & Management:** PostgreSQL 16, Named Volumes, pgAdmin 4, automated dump/restore shell scripts.
- **Observability & Logging:** Fluent Bit log forwarder, regex parsing, PostgreSQL JSONB storage, Prometheus v3, cAdvisor, Node Exporter, Grafana.
- **Security & Networking:** TLS/SSL OpenSSL self-signed certificates, User-defined bridge networks, RBAC (Role-Based Access Control) PostgreSQL read-only users.

---

> *Laporan ini disusun untuk memenuhi tugas mata kuliah Workshop DevOps, Departemen Teknik Informatika dan Komputer, Politeknik Elektronika Negeri Surabaya.*
