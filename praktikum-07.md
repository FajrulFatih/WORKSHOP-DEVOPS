# Praktikum 07: Monitoring Resource dengan Prometheus, cAdvisor, Node Exporter, dan Grafana

> **Mata Kuliah:** Workshop DevOps  
> **Dosen Pengampu:** Dr. Ferry Astika Saputra, S.T., M.Sc.  
> **Kelompok:** 5  
> **Anggota:**
> - Muhammad Fajrul Fatih Abul ‘Ilmi (3124600040)
> - Nur Aini Agusthina (3124600050)
> - Abdullah Falich (3124600059)
> 
> **Program Studi:** D4 Teknik Informatika, Departemen Teknik Informatika dan Komputer  
> **Institusi:** Politeknik Elektronika Negeri Surabaya (PENS) - 2026

---

## PRAKTIKUM

![Screenshot 1](uploads/praktikum-07/image-01.png)

### Menyiapkan Direktori

```bash
mkdir -p ~/docker-lab/bab-7/{prometheus/rules,prometheus/tests,grafana/provisioning/datasources,grafana/provisioning/dashboards,grafana/dashboards,scripts,reports}
cd ~/docker-lab/bab-7

```

Struktur file:

```text
bab-7/
├── .env.example
├── .env
├── .gitignore
├── compose.yaml
├── prometheus/
│   ├── prometheus.yml
│   ├── rules/
│   │   ├── recording-rules.yml
│   │   └── alert-rules.yml
│   └── tests/
│       └── rules.test.yml
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/datasources.yml
│   │   └── dashboards/provider.yml
│   └── dashboards/
│       └── bab-7-overview.json
├── scripts/
│   ├── verify-stack.sh
│   └── connect-bab6.sh
└── reports/

```

#### File .env.example

> **Nama file:** `.env.example`  
> **Lokasi:** `~/docker-lab/bab-7/.env.example`  
> **Buka editor:** `nano .env.example`  
```dotenv
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=grafanaadmin123
GRAFANA_DB_USER=grafana_reader
GRAFANA_DB_PASSWORD=grafanareaderpass123
POSTGRES_DB=labdb
```

Aktifkan konfigurasi lokal:

```dotenv
cp .env.example .env
chmod 600 .env
```

Credential tersebut bersifat sintetis. Pada lingkungan produksi, gunakan secret manager, TLS, rotasi credential, dan kontrol akses berdasarkan peran.

#### File .gitignore

> **Nama file:** `.gitignore`  
> **Lokasi:** `~/docker-lab/bab-7/.gitignore`  
> **Buka editor:** `nano .gitignore`  
```gitignore
.env
reports/*
!reports/.gitkeep
*.log

```

#### File compose.yaml

> **Nama file:** `compose.yaml`  
> **Lokasi:** `~/docker-lab/bab-7/compose.yaml`  
> **Buka editor:** `nano compose.yaml`  
> **Petunjuk:** `image dipin ke release stabil yang diverifikasi pada 22 Agustus 2026. Pin digest tetap disarankan sebelum pelaksanaan kelas.`  
```yaml
services:
  prometheus:
    image: prom/prometheus:v3.14.0
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=7d
      - --web.console.libraries=/usr/share/prometheus/console_libraries
      - --web.console.templates=/usr/share/prometheus/consoles
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus:ro
      - prom-data:/prometheus
    networks:
      - monitoring-net
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  grafana:
    image: grafana/grafana:13.1.4
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_AUTH_ANONYMOUS_ENABLED: "false"
      GRAFANA_DB_USER: ${GRAFANA_DB_USER}
      GRAFANA_DB_PASSWORD: ${GRAFANA_DB_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    networks:
      - monitoring-net
    depends_on:
      - prometheus
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.12.1
    command:
      - --path.rootfs=/host
      - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/containers/storage/.+)($|/)
    pid: host
    volumes:
      - /:/host:ro,rslave
    networks:
      - monitoring-net
    read_only: true
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  cadvisor:
    image: ghcr.io/google/cadvisor:v0.60.5
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    networks:
      - monitoring-net
    restart: unless-stopped

  load-generator:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - while :; do :; done
    cpus: 0.50
    mem_limit: 64m
    profiles:
      - load
    networks:
      - monitoring-net
    read_only: true
    security_opt:
      - no-new-privileges:true

volumes:
  prom-data:
  grafana-data:

networks:
  monitoring-net:
    name: bab7-monitoring-net
    driver: bridge
```

privileged: true pada cAdvisor merupakan pengecualian berisiko karena collector membaca metadata host dan runtime container. Konfigurasi ini hanya untuk laptop lab terisolasi. Pada lingkungan produksi, gunakan integrasi monitoring yang sesuai platform dan kurangi privilege berdasarkan hasil pengujian nyata.

#### File prometheus/prometheus.yml

> **Nama file:** `prometheus.yml`  
> **Lokasi:** `~/docker-lab/bab-7/prometheus/prometheus.yml`  
> **Buka editor:** `nano prometheus/prometheus.yml`  
```bash
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    environment: lab
    chapter: "7"

rule_files:
  - /etc/prometheus/rules/recording-rules.yml
  - /etc/prometheus/rules/alert-rules.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - prometheus:9090

  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100

  - job_name: cadvisor
    static_configs:
      - targets:
          - cadvisor:8080
```

Timeout dibuat lebih pendek daripada interval scrape. Nilai 15 detik sesuai untuk observasi laboratorium, tetapi bukan angka universal untuk lingkungan produksi karena resolusi yang lebih rapat meningkatkan trafik, CPU, dan kebutuhan storage.

#### File prometheus/rules/recording-rules.yml

> **Nama file:** `recording-rules.yml`  
> **Lokasi:** `~/docker-lab/bab-7/prometheus/rules/recording-rules.yml`  
> **Buka editor:** `nano prometheus/rules/recording-rules.yml`  
```yaml
groups:
  - name: bab7-resource-recording
    interval: 15s
    rules:
      - record: instance:node_cpu_utilisation:ratio_rate5m
        expr: |
          1 - avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      - record: instance:node_memory_utilisation:ratio
        expr: |
          1 - (
            node_memory_MemAvailable_bytes
            /
            node_memory_MemTotal_bytes
          )

      - record: instance:node_filesystem_avail:ratio
        expr: |
          node_filesystem_avail_bytes{
            fstype!~"tmpfs|overlay|squashfs|nsfs"
          }
          /
          node_filesystem_size_bytes{
            fstype!~"tmpfs|overlay|squashfs|nsfs"
          }

      - record: container:cpu_usage_cores:rate1m
        expr: |
          sum by (name) (
            rate(container_cpu_usage_seconds_total{name!=""}[1m])
          )

      - record: container:memory_working_set_bytes:sum
        expr: |
          sum by (name) (
            container_memory_working_set_bytes{name!=""}
          )
```

Recording rule mengurangi pengulangan query dashboard. Nama rule menyatakan agregasi, rasio, dan jendela waktu agar semantiknya tidak ambigu.

#### File prometheus/rules/alert-rules.yml

> **Nama file:** `alert-rules.yml`  
> **Lokasi:** `~/docker-lab/bab-7/prometheus/rules/alert-rules.yml`  
> **Buka editor:** `nano prometheus/rules/alert-rules.yml`  
```yaml
groups:
  - name: bab7-resource-alerts
    interval: 15s
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: warning
          category: availability
        annotations:
          summary: "Target {{ $labels.instance }} tidak dapat discrape"
          description: "Job {{ $labels.job }} gagal discrape selama sedikitnya satu menit."

      - alert: HostHighCPU
        expr: instance:node_cpu_utilisation:ratio_rate5m > 0.90
        for: 5m
        labels:
          severity: warning
          category: saturation
        annotations:
          summary: "Utilisasi CPU host melampaui 90 persen"
          description: "Nilai saat ini adalah {{ $value | humanizePercentage }}."

      - alert: HostLowAvailableMemory
        expr: instance:node_memory_utilisation:ratio > 0.90
        for: 5m
        labels:
          severity: warning
          category: saturation
        annotations:
          summary: "Memori host yang digunakan melampaui 90 persen"
          description: "Nilai saat ini adalah {{ $value | humanizePercentage }}."

      - alert: ContainerHighCPU
        expr: container:cpu_usage_cores:rate1m > 0.30
        for: 1m
        labels:
          severity: info
          category: saturation
        annotations:
          summary: "Container {{ $labels.name }} memakai CPU tinggi"
          description: "Rata-rata satu menit adalah {{ $value }} core CPU."

      - alert: PrometheusRuleEvaluationFailure
        expr: increase(prometheus_rule_evaluation_failures_total[5m]) > 0
        for: 1m
        labels:
          severity: warning
          category: monitoring
        annotations:
          summary: "Evaluasi rule Prometheus mengalami kegagalan"
          description: "Periksa status rule, ekspresi PromQL, dan log Prometheus."
```

Threshold merupakan parameter eksperimen, bukan standar lingkungan produksi. Baseline, kapasitas host, karakter workload, dan SLO harus menentukan nilai operasional yang sebenarnya.

#### File scripts/generate-logs.sh

> **Nama file:** `generate-logs.sh`  
> **Lokasi:** `~/docker-lab/bab-6/scripts/generate-logs.sh`  
> **Buka editor:** `nano scripts/generate-logs.sh`  
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

docker compose up -d postgres-db fluent-bit

for attempt in {1..30}; do
    if curl -fsS http://127.0.0.1:2020/api/v1/health >/dev/null; then
        break
    fi

    if [[ "$attempt" -eq 30 ]]; then
        echo "[FAIL] Endpoint kesehatan Fluent Bit belum siap." >&2
        exit 1
    fi
    sleep 2
done

docker compose up -d --force-recreate \
    generator-auth \
    generator-api \
    generator-worker

echo "[PASS] Tiga generator telah dijalankan. Tunggu sekitar 25 detik."
echo "[INFO] Pantau dengan: docker compose ps"
```

![Screenshot 2](uploads/praktikum-07/image-02.png)

#### File scripts/test-buffering.sh

> **Nama file:** `test-buffering.sh`  
> **Lokasi:** `~/docker-lab/bab-6/scripts/test-buffering.sh`  
> **Buka editor:** `nano scripts/test-buffering.sh`  
> **Tujuan:** `membuktikan bahwa Fluent Bit menahan event ketika PostgreSQL dihentikan sementara, lalu mengirimkannya setelah backend pulih.`  
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

count_rows() {
    docker compose exec -T postgres-db \
        psql \
        --username "$POSTGRES_USER" \
        --dbname "$POSTGRES_DB" \
        --tuples-only \
        --no-align \
        --command 'SELECT count(*) FROM logs.fluentbit;'
}

before="$(count_rows)"
echo "[INFO] Jumlah awal: $before"

docker compose stop postgres-db

docker compose up -d --force-recreate \
    generator-auth \
    generator-api \
    generator-worker

echo "[INFO] PostgreSQL berhenti; generator berjalan selama 25 detik."
sleep 25

docker compose start postgres-db

for attempt in {1..30}; do
    if docker compose exec -T postgres-db \
        pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB" >/dev/null 2>&1; then
        break
    fi

    if [[ "$attempt" -eq 30 ]]; then
        echo "[FAIL] PostgreSQL tidak kembali sehat." >&2
        exit 1
    fi
    sleep 2
done

for attempt in {1..60}; do
    after="$(count_rows)"
    if (( after > before )); then
        echo "[PASS] Jumlah sesudah pemulihan: $after"
        echo "[INFO] Kenaikan record: $((after - before))"
        exit 0
    fi
    sleep 2
done

echo "[FAIL] Tidak ada record baru setelah PostgreSQL pulih." >&2
echo "[INFO] Periksa: docker compose logs --tail 200 fluent-bit" >&2
exit 1
```

![Screenshot 3](uploads/praktikum-07/image-03.png)

Pengujian ini membuktikan pemulihan pada skala lab, bukan jaminan exactly-once. Retry dapat menghasilkan duplikasi, sedangkan buffer penuh, disk gagal, atau proses dihentikan secara paksa tetap dapat menyebabkan kehilangan event.

> **Nama file:** `.env.example`

> **Lokasi:** `~/docker-lab/bab-7/.env.example`

> **Buka editor:** `nano .env.example`

```dotenv
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=grafanaadmin123
GRAFANA_DB_USER=grafana_reader
GRAFANA_DB_PASSWORD=grafanareaderpass123
POSTGRES_DB=labdb
```

### Aktifkan konfigurasi lokal:

```bash
cp .env.example .env
chmod 600 .env
```

### Credential tersebut bersifat sintetis. Pada lingkungan produksi, gunakan secret manager, TLS, rotasi credential, dan kontrol akses berdasarkan peran.

## Kotak Script 2 — File `.gitignore`

> **Nama file:** `.gitignore`

> **Lokasi:** `~/docker-lab/bab-7/.gitignore`

> **Buka editor:** `nano .gitignore`

```gitignore
.env
reports/*
!reports/.gitkeep
*.log
```

## Kotak Script 3 — File `compose.yaml`

> **Nama file:** `compose.yaml`

> **Lokasi:** `~/docker-lab/bab-7/compose.yaml`

> **Buka editor:** `nano compose.yaml`

> **Petunjuk:** image dipin ke release stabil yang diverifikasi pada 22 Agustus 2026. Pin digest tetap disarankan sebelum pelaksanaan kelas.

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.14.0
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=7d
      - --web.console.libraries=/usr/share/prometheus/console_libraries
      - --web.console.templates=/usr/share/prometheus/consoles
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus:ro
      - prom-data:/prometheus
    networks:
      - monitoring-net
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  grafana:
    image: grafana/grafana:13.1.4
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_AUTH_ANONYMOUS_ENABLED: "false"
      GRAFANA_DB_USER: ${GRAFANA_DB_USER}
      GRAFANA_DB_PASSWORD: ${GRAFANA_DB_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    networks:
      - monitoring-net
    depends_on:
      - prometheus
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.12.1
    command:
      - --path.rootfs=/host
      - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/containers/storage/.+)($|/)
    pid: host
    volumes:
      - /:/host:ro,rslave
    networks:
      - monitoring-net
    read_only: true
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  cadvisor:
    image: ghcr.io/google/cadvisor:v0.60.5
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    networks:
      - monitoring-net
    restart: unless-stopped

  load-generator:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - while :; do :; done
    cpus: 0.50
    mem_limit: 64m
    profiles:
      - load
    networks:
      - monitoring-net
    read_only: true
    security_opt:
      - no-new-privileges:true

volumes:
  prom-data:
  grafana-data:

networks:
  monitoring-net:
    name: bab7-monitoring-net
    driver: bridge
```

### `privileged: true` pada cAdvisor merupakan pengecualian berisiko karena collector membaca metadata host dan runtime container. Konfigurasi ini hanya untuk laptop lab terisolasi. Pada lingkungan produksi, gunakan integrasi monitoring yang sesuai platform dan kurangi privilege berdasarkan hasil pengujian nyata.

## Kotak Script 4 — File `prometheus/prometheus.yml`

> **Nama file:** `prometheus.yml`

> **Lokasi:** `~/docker-lab/bab-7/prometheus/prometheus.yml`

> **Buka editor:** `nano prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    environment: lab
    chapter: "7"

rule_files:
  - /etc/prometheus/rules/recording-rules.yml
  - /etc/prometheus/rules/alert-rules.yml

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - prometheus:9090

  - job_name: node-exporter
    static_configs:
      - targets:
          - node-exporter:9100

  - job_name: cadvisor
    static_configs:
      - targets:
          - cadvisor:8080
```

### Timeout dibuat lebih pendek daripada interval scrape. Nilai 15 detik sesuai untuk observasi laboratorium, tetapi bukan angka universal untuk lingkungan produksi karena resolusi yang lebih rapat meningkatkan trafik, CPU, dan kebutuhan storage.

## Kotak Script 5 — File `prometheus/rules/recording-rules.yml`

> **Nama file:** `recording-rules.yml`

> **Lokasi:** `~/docker-lab/bab-7/prometheus/rules/recording-rules.yml`

> **Buka editor:** `nano prometheus/rules/recording-rules.yml`

```yaml
groups:
  - name: bab7-resource-recording
    interval: 15s
    rules:
      - record: instance:node_cpu_utilisation:ratio_rate5m
        expr: |
          1 - avg by (instance) (
            rate(node_cpu_seconds_total{mode="idle"}[5m])
          )

      - record: instance:node_memory_utilisation:ratio
        expr: |
          1 - (
            node_memory_MemAvailable_bytes
            /
            node_memory_MemTotal_bytes
          )

      - record: instance:node_filesystem_avail:ratio
        expr: |
          node_filesystem_avail_bytes{
            fstype!~"tmpfs|overlay|squashfs|nsfs"
          }
          /
          node_filesystem_size_bytes{
            fstype!~"tmpfs|overlay|squashfs|nsfs"
          }

      - record: container:cpu_usage_cores:rate1m
        expr: |
          sum by (name) (
            rate(container_cpu_usage_seconds_total{name!=""}[1m])
          )

      - record: container:memory_working_set_bytes:sum
        expr: |
          sum by (name) (
            container_memory_working_set_bytes{name!=""}
          )
```

### Recording rule mengurangi pengulangan query dashboard. Nama rule menyatakan agregasi, rasio, dan jendela waktu agar semantiknya tidak ambigu.

## Kotak Script 6 — File `prometheus/rules/alert-rules.yml`

> **Nama file:** `alert-rules.yml`

> **Lokasi:** `~/docker-lab/bab-7/prometheus/rules/alert-rules.yml`

> **Buka editor:** `nano prometheus/rules/alert-rules.yml`

```yaml
groups:
  - name: bab7-resource-alerts
    interval: 15s
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: warning
          category: availability
        annotations:
          summary: "Target {{ $labels.instance }} tidak dapat discrape"
          description: "Job {{ $labels.job }} gagal discrape selama sedikitnya satu menit."

      - alert: HostHighCPU
        expr: instance:node_cpu_utilisation:ratio_rate5m > 0.90
        for: 5m
        labels:
          severity: warning
          category: saturation
        annotations:
          summary: "Utilisasi CPU host melampaui 90 persen"
          description: "Nilai saat ini adalah {{ $value | humanizePercentage }}."

      - alert: HostLowAvailableMemory
        expr: instance:node_memory_utilisation:ratio > 0.90
        for: 5m
        labels:
          severity: warning
          category: saturation
        annotations:
          summary: "Memori host yang digunakan melampaui 90 persen"
          description: "Nilai saat ini adalah {{ $value | humanizePercentage }}."

      - alert: ContainerHighCPU
        expr: container:cpu_usage_cores:rate1m > 0.30
        for: 1m
        labels:
          severity: info
          category: saturation
        annotations:
          summary: "Container {{ $labels.name }} memakai CPU tinggi"
          description: "Rata-rata satu menit adalah {{ $value }} core CPU."

      - alert: PrometheusRuleEvaluationFailure
        expr: increase(prometheus_rule_evaluation_failures_total[5m]) > 0
        for: 1m
        labels:
          severity: warning
          category: monitoring
        annotations:
          summary: "Evaluasi rule Prometheus mengalami kegagalan"
          description: "Periksa status rule, ekspresi PromQL, dan log Prometheus."
```

### Threshold merupakan parameter eksperimen, bukan standar lingkungan produksi. Baseline, kapasitas host, karakter workload, dan SLO harus menentukan nilai operasional yang sebenarnya.

## Kotak Script 7 — File `prometheus/tests/rules.test.yml`

> **Nama file:** `rules.test.yml`

> **Lokasi:** `~/docker-lab/bab-7/prometheus/tests/rules.test.yml`

> **Buka editor:** `nano prometheus/tests/rules.test.yml`

```yaml
rule_files:
  - /etc/prometheus/rules/alert-rules.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'up{job="node-exporter",instance="node-exporter:9100"}'
        values: '1 0 0 0'

    alert_rule_test:
      - eval_time: 2m
        alertname: TargetDown
        exp_alerts:
          - exp_labels:
              job: node-exporter
              instance: node-exporter:9100
              severity: warning
              category: availability
            exp_annotations:
              summary: "Target node-exporter:9100 tidak dapat discrape"
              description: "Job node-exporter gagal discrape selama sedikitnya satu menit."
```

### Unit test memeriksa logika rule secara deterministik tanpa menunggu kegagalan nyata. Runtime test tetap diperlukan karena unit test tidak membuktikan target, jaringan, atau notifikasi berfungsi.

## Kotak Script 8 — File `grafana/provisioning/datasources/datasources.yml`

> **Nama file:** `datasources.yml`

> **Lokasi:** `~/docker-lab/bab-7/grafana/provisioning/datasources/datasources.yml`

> **Buka editor:** `nano grafana/provisioning/datasources/datasources.yml`

```yaml
apiVersion: 1

deleteDatasources:
  - name: Prometheus Bab 7
    orgId: 1
  - name: PostgreSQL Logs Bab 6
    orgId: 1

datasources:
  - name: Prometheus Bab 7
    uid: prometheus-bab7
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
    jsonData:
      httpMethod: POST
      timeInterval: 15s

  - name: PostgreSQL Logs Bab 6
    uid: postgres-logs-bab6
    type: postgres
    access: proxy
    url: postgres-db:5432
    user: ${GRAFANA_DB_USER}
    editable: false
    secureJsonData:
      password: ${GRAFANA_DB_PASSWORD}
    jsonData:
      database: ${POSTGRES_DB}
      sslmode: disable
      postgresVersion: 1600
      timescaledb: false
```

### Datasource PostgreSQL akan berstatus gagal sampai prosedur integrasi opsional Bab 6 dijalankan. `sslmode: disable` hanya dapat diterima pada jaringan lab lokal; koneksi lingkungan produksi memerlukan verifikasi TLS.

## Kotak Script 9 — File `grafana/provisioning/dashboards/provider.yml`

> **Nama file:** `provider.yml`

> **Lokasi:** `~/docker-lab/bab-7/grafana/provisioning/dashboards/provider.yml`

> **Buka editor:** `nano grafana/provisioning/dashboards/provider.yml`

```yaml
apiVersion: 1

providers:
  - name: DevSecOps Lab
    orgId: 1
    folder: DevSecOps Lab
    type: file
    disableDeletion: false
    allowUiUpdates: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
```

### Dashboard dibuat tidak dapat diedit melalui UI agar sumber konfigurasi tetap jelas. Perubahan dilakukan pada JSON dan dikaji melalui version control.

## Kotak Script 10 — File `grafana/dashboards/bab-7-overview.json`

> **Nama file:** `bab-7-overview.json`

> **Lokasi:** `~/docker-lab/bab-7/grafana/dashboards/bab-7-overview.json`

> **Buka editor:** `nano grafana/dashboards/bab-7-overview.json`

```json
{
  "annotations": {
    "list": []
  },
  "editable": false,
  "graphTooltip": 1,
  "id": null,
  "links": [],
  "panels": [
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus-bab7"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "red",
                "value": null
              },
              {
                "color": "green",
                "value": 3
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 5,
        "w": 6,
        "x": 0,
        "y": 0
      },
      "id": 1,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "targets": [
        {
          "expr": "sum(up)",
          "legendFormat": "target up",
          "refId": "A"
        }
      ],
      "title": "Jumlah Target Up",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus-bab7"
      },
      "fieldConfig": {
        "defaults": {
          "max": 100,
          "min": 0,
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 9,
        "x": 6,
        "y": 0
      },
      "id": 2,
      "options": {
        "legend": {
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "expr": "instance:node_cpu_utilisation:ratio_rate5m * 100",
          "legendFormat": "{{instance}}",
          "refId": "A"
        }
      ],
      "title": "Utilisasi CPU Host",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus-bab7"
      },
      "fieldConfig": {
        "defaults": {
          "max": 100,
          "min": 0,
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 9,
        "x": 15,
        "y": 0
      },
      "id": 3,
      "options": {
        "legend": {
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "expr": "instance:node_memory_utilisation:ratio * 100",
          "legendFormat": "{{instance}}",
          "refId": "A"
        }
      ],
      "title": "Utilisasi Memori Host",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus-bab7"
      },
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "unit": "cores"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 0,
        "y": 8
      },
      "id": 4,
      "options": {
        "legend": {
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "expr": "topk(5, container:cpu_usage_cores:rate1m)",
          "legendFormat": "{{name}}",
          "refId": "A"
        }
      ],
      "title": "Top 5 CPU Container",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "prometheus",
        "uid": "prometheus-bab7"
      },
      "fieldConfig": {
        "defaults": {
          "min": 0,
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 9,
        "w": 12,
        "x": 12,
        "y": 8
      },
      "id": 5,
      "options": {
        "legend": {
          "displayMode": "table",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "multi",
          "sort": "desc"
        }
      },
      "targets": [
        {
          "expr": "topk(5, container:memory_working_set_bytes:sum)",
          "legendFormat": "{{name}}",
          "refId": "A"
        }
      ],
      "title": "Top 5 Memori Container",
      "type": "timeseries"
    }
  ],
  "refresh": "15s",
  "schemaVersion": 42,
  "tags": [
    "devsecops",
    "bab-7",
    "monitoring"
  ],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-30m",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "browser",
  "title": "Bab 7 — Resource Monitoring",
  "uid": "bab7-resource-monitoring",
  "version": 1
}
```

## Kotak Script 11 — File `scripts/verify-stack.sh`

> **Nama file:** `verify-stack.sh`

> **Lokasi:** `~/docker-lab/bab-7/scripts/verify-stack.sh`

> **Buka editor:** `nano scripts/verify-stack.sh`

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "0")/.."

wait_http() {
    local name="$1"
    local url="$2"

    for attempt in {1..30}; do
        if curl -fsS "$url" >/dev/null; then
            echo "[PASS] $name siap: $url"
            return 0
        fi
        sleep 2
    done

    echo "[FAIL] $name tidak siap: $url" >&2
    return 1
}

wait_http "Prometheus" "http://127.0.0.1:9090/-/ready"
wait_http "Grafana" "http://127.0.0.1:3000/api/health"

targets="$(curl -fsS http://127.0.0.1:9090/api/v1/targets)"
up_count="targets" | wc -l | tr -d ' ' || true)"
down_count="targets" | wc -l | tr -d ' ' || true)"

if (( up_count >= 3 && down_count == 0 )); then
    echo "[PASS] Sedikitnya tiga target berstatus up."
else
    echo "[FAIL] Target up=down_count." >&2
    exit 1
fi

for expression in \
    'up' \
    'instance:node_cpu_utilisation:ratio_rate5m' \
    'instance:node_memory_utilisation:ratio' \
    'container_memory_working_set_bytes'; do
    response="$(
        curl -fsS --get \
            --data-urlencode "query=$expression" \
            http://127.0.0.1:9090/api/v1/query
    )"

    if [[ "$response" == *'"status":"success"'* ]]; then
        echo "[PASS] PromQL valid: $expression"
    else
        echo "[FAIL] PromQL gagal: $expression" >&2
        exit 1
    fi
done

docker compose exec -T prometheus \
    promtool check config /etc/prometheus/prometheus.yml

docker compose exec -T prometheus \
    promtool check rules \
    /etc/prometheus/rules/recording-rules.yml \
    /etc/prometheus/rules/alert-rules.yml

echo "[PASS] Verifikasi dasar stack selesai."
```

### Status sukses query belum membuktikan result vector berisi data. Karena itu, mahasiswa tetap harus memeriksa nilai dan label melalui Prometheus atau Grafana.

## Kotak Script 12 — File `scripts/connect-bab6.sh`

> **Nama file:** `connect-bab6.sh`

> **Lokasi:** `~/docker-lab/bab-7/scripts/connect-bab6.sh`

> **Buka editor:** `nano scripts/connect-bab6.sh`

> **Tujuan:** menghubungkan Grafana ke PostgreSQL Bab 6 tanpa memberi akun administrator kepada dashboard.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "0")/.."

bab6_dir="${BAB6_DIR:-../bab-6}"

if [[ ! -f .env || ! -f "$bab6_dir/.env" ]]; then
    echo "[FAIL] File .env Bab 7 atau Bab 6 tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

reader_user="$GRAFANA_DB_USER"
reader_password="$GRAFANA_DB_PASSWORD"

set -a
source "$bab6_dir/.env"
set +a

db_container="$(
    docker compose --project-directory "$bab6_dir" ps -q postgres-db
)"

if [[ -z "$db_container" ]]; then
    echo "[FAIL] Container PostgreSQL Bab 6 tidak berjalan." >&2
    exit 1
fi

if ! docker network inspect bab7-monitoring-net >/dev/null 2>&1; then
    echo "[FAIL] Network Bab 7 belum tersedia. Jalankan stack Bab 7 dahulu." >&2
    exit 1
fi

if ! docker inspect "$db_container" \
    --format '{{json .NetworkSettings.Networks}}' \
    | grep -q 'bab7-monitoring-net'; then
    docker network connect \
        --alias postgres-db \
        bab7-monitoring-net \
        "$db_container"
fi

docker compose --project-directory "$bab6_dir" exec -T postgres-db \
    psql \
    --set=ON_ERROR_STOP=1 \
    --username "$POSTGRES_USER" \
    --dbname "$POSTGRES_DB" \
    --set=reader_user="$reader_user" \
    --set=reader_password="$reader_password" <<'SQL'
SELECT format(
    'CREATE ROLE %I LOGIN PASSWORD %L',
    :'reader_user',
    :'reader_password'
)
WHERE NOT EXISTS (
    SELECT 1 FROM pg_roles WHERE rolname = :'reader_user'
)
\gexec

SELECT format(
    'ALTER ROLE %I PASSWORD %L',
    :'reader_user',
    :'reader_password'
)
\gexec

SELECT format(
    'GRANT CONNECT ON DATABASE %I TO %I',
    current_database(),
    :'reader_user'
)
\gexec

SELECT format('GRANT USAGE ON SCHEMA logs TO %I', :'reader_user')
\gexec

SELECT format(
    'GRANT SELECT ON ALL TABLES IN SCHEMA logs TO %I',
    :'reader_user'
)
\gexec

SELECT format(
    'ALTER DEFAULT PRIVILEGES IN SCHEMA logs GRANT SELECT ON TABLES TO %I',
    :'reader_user'
)
\gexec
SQL

docker compose restart grafana
echo "[PASS] PostgreSQL Bab 6 terhubung dengan role read-only."
```

### Script menggunakan network tambahan dan role baca khusus. Hal ini lebih aman daripada memublikasikan port PostgreSQL atau memberikan credential admin kepada Grafana, tetapi trafik database pada lab ini masih tanpa TLS.

## Kotak Perintah B — Memberi Izin dan Memvalidasi Konfigurasi

```bash
chmod +x scripts/verify-stack.sh scripts/connect-bab6.sh

bash -n scripts/verify-stack.sh
bash -n scripts/connect-bab6.sh
docker compose config --quiet
```

### Validasi JSON dashboard:

```bash
python3 -m json.tool \
  grafana/dashboards/bab-7-overview.json \
  >/dev/null
```

### Validasi konfigurasi dan rule Prometheus sebelum stack dinyalakan:

```bash
docker compose run --rm --no-deps \
  --entrypoint /bin/promtool \
  prometheus \
  check config /etc/prometheus/prometheus.yml

docker compose run --rm --no-deps \
  --entrypoint /bin/promtool \
  prometheus \
  test rules /etc/prometheus/tests/rules.test.yml
```

## Kotak Perintah C — Menjalankan Stack Monitoring

```bash
docker compose pull
docker compose up -d
docker compose ps
./scripts/verify-stack.sh
```

### Antarmuka lokal:

### - Prometheus: `http://127.0.0.1:9090`

### - Grafana: `http://127.0.0.1:3000`

### - Dashboard Grafana: folder **DevSecOps Lab**, dashboard **Bab 7 — Resource Monitoring**

Masuk ke Grafana menggunakan nilai `GRAFANA_ADMIN_USER` dan `GRAFANA_ADMIN_PASSWORD` dari `.env`.

## Kotak Perintah D — Query PromQL Dasar

### Jalankan melalui menu **Prometheus → Graph** atau Grafana Explore:

```promql
up
```

```promql
instance:node_cpu_utilisation:ratio_rate5m * 100
```

```promql
instance:node_memory_utilisation:ratio * 100
```

```promql
topk(5, container:cpu_usage_cores:rate1m)
```

```promql
topk(5, container:memory_working_set_bytes:sum)
```

### `rate()` digunakan pada counter CPU karena nilai counter mentah terus bertambah. Untuk dashboard dan alert yang relatif stabil, `rate()` lebih tepat daripada `irate()`, yang hanya menggunakan dua sampel terakhir dan lebih sensitif terhadap lonjakan.

## Kotak Perintah E — Menguji Beban Container

### Jalankan generator dengan batas setengah core CPU:

```bash
docker compose --profile load up -d load-generator
```

### Amati query berikut selama sedikitnya dua menit:

```promql
container:cpu_usage_cores:rate1m{name=~".*load-generator.*"}
```

```promql
ALERTS{alertname="ContainerHighCPU"}
```

### Hentikan generator setelah evidence diperoleh:

```bash
docker compose --profile load stop load-generator
docker compose --profile load rm -f load-generator
```

## Kotak Perintah F — Negative Test Target Down

```bash
docker compose stop node-exporter
```

### Setelah sedikitnya 75 detik, periksa:

```promql
up{job="node-exporter"}
```

```promql
ALERTS{alertname="TargetDown",alertstate="firing"}
```

### Pulihkan target:

```bash
docker compose start node-exporter
```

### Dalam satu atau dua interval scrape, `up` harus kembali menjadi `1`. Alert akan kembali nonaktif setelah kondisi tidak lagi terpenuhi.

## Kotak Perintah G — Integrasi Opsional Log Bab 6

### Pastikan Bab 6 dan Bab 7 berjalan, lalu jalankan dari direktori Bab 7:

```bash
./scripts/connect-bab6.sh
```

### Pada Grafana Explore, pilih datasource **PostgreSQL Logs Bab 6** dan gunakan query:

```sql
SELECT
    time AS "time",
    tag,
    replace(data ->> 'container_name', '/', '') AS container_name,
    data ->> 'source' AS source,
    data ->> 'log' AS log_message
FROM logs.fluentbit
WHERE $__timeFilter(time)
ORDER BY time DESC
LIMIT 200;
```

### Macro `$__timeFilter(time)` membuat query mengikuti rentang waktu Grafana. Untuk data besar, query harus tetap didukung indeks waktu dan pembatasan baris.

## Tabel Verifikasi PASS/FAIL

### | No. | Skenario | Evidence yang sahih | PASS | FAIL |

### |---:|---|---|---|---|

### | 1 | Validasi Compose | `docker compose config --quiet` | Exit code 0 | Galat YAML atau interpolasi |

### | 2 | Unit test rule | Output `promtool test rules` | `SUCCESS` | Test gagal atau rule tidak termuat |

### | 3 | Kesiapan Prometheus | `/-/ready` | HTTP 200 | Timeout atau non-2xx |

### | 4 | Kesiapan Grafana | `/api/health` | HTTP 200 dan database `ok` | Service belum sehat |

### | 5 | Target scrape | Halaman Targets atau API | 3 target `UP` | Salah satu target `DOWN` |

### | 6 | Host metrics | Query CPU dan memori | Result vector berisi nilai | Query sukses tetapi result kosong |

### | 7 | Container metrics | Query cAdvisor | Nama container dan nilai muncul | cAdvisor kosong/tidak dapat membaca runtime |

### | 8 | Dashboard | Dashboard terprovisi | Lima panel berisi data sesuai unit | Panel `No data` atau unit salah |

### | 9 | Load test | Query CPU load generator | Pemakaian naik secara terukur | Tidak ada perubahan/limit tidak berlaku |

### | 10 | Negative test | `TargetDown` | Pending lalu firing | Alert tidak berubah state |

### | 11 | Recovery | `up{job="node-exporter"}` | Kembali `1` | Tetap `0`/series hilang |

### | 12 | Integrasi Bab 6 | Query Grafana PostgreSQL | Log tampil dengan role read-only | Koneksi atau privilege gagal |

## Troubleshooting dan Analisis Hasil

### | Gejala | Diagnosis yang mungkin | Pemeriksaan | Tindakan korektif |

### |---|---|---|---|

### | cAdvisor gagal pada Windows/WSL 2 | Mount atau kapabilitas Linux VM berbeda | `docker compose logs cadvisor` | Catat keterbatasan platform; uji di host Linux bila pengamatan host penuh diperlukan |

### | Galat mount propagation `rslave` | Docker Desktop tidak mendukung opsi yang sama | Log startup Node Exporter | Ubah sementara `/:/host:ro,rslave` menjadi `/:/host:ro` dan dokumentasikan dampaknya |

### | Target `DOWN` | DNS, service, port, atau exporter gagal | Prometheus Targets; `docker compose logs` | Uji endpoint dari jaringan internal dan periksa nama target |

### | Query sukses tetapi kosong | Metric/label tidak tersedia atau jendela `rate` belum cukup | Cari nama metric tanpa filter; tunggu beberapa scrape | Sesuaikan selector berdasarkan label aktual, jangan mengganti nilai kosong menjadi nol tanpa alasan |

### | CPU host tampak kecil saat load tinggi | Beban 0,5 core dibandingkan banyak core host | Bandingkan CPU container dan CPU host | Jelaskan denominator; jangan menyamakan core usage dengan persentase seluruh host |

### | Grafana menampilkan `No data` | UID datasource, rule, atau rentang waktu salah | Grafana Explore; log Grafana; Prometheus Rules | Verifikasi datasource `prometheus-bab7`, rule sehat, dan rentang waktu dashboard |

### | Alert tidak firing | Kondisi belum melewati threshold selama durasi `for` | Query ekspresi alert; menu Alerts | Tunggu durasi penuh dan pastikan evaluation interval berjalan |

### | PostgreSQL datasource gagal | Bab 6 belum terhubung, role salah, atau database tidak berjalan | Log Grafana; `docker network inspect`; `\du`/`\dp` | Jalankan `connect-bab6.sh`, verifikasi alias jaringan dan GRANT |

### | Panel container memiliki cardinality tinggi | Label container berubah setiap recreate | Hitung series per label | Gunakan label service stabil dan batasi label yang dipertahankan |

### | Penggunaan disk Prometheus tumbuh | Retensi dan jumlah series terlalu besar | Status TSDB; cardinality; volume | Kurangi cardinality, sesuaikan interval dan retensi berdasarkan kapasitas |

## Catatan Keamanan

### - Endpoint metric dapat mengungkap hostname, mount point, versi, nama service, dan pola beban. Jangan memublikasasikan exporter langsung ke jaringan publik.

### - Akun anonim Grafana dinonaktifkan, tetapi password lab tetap harus diganti bila lingkungan dapat diakses pihak lain.

### - cAdvisor yang privileged memiliki visibilitas luas terhadap host. Hindari menjalankannya pada laptop yang menangani data sensitif tanpa persetujuan dan isolasi yang memadai.

### - `up = 1` hanya membuktikan scrape berhasil; nilai tersebut tidak membuktikan fungsi bisnis sehat.

### - Tidak adanya series berbeda dari nilai nol. Alert dan dashboard harus memperlakukan missing data secara eksplisit.

## Kotak Perintah H — Cleanup

### Menghentikan stack sambil mempertahankan data Prometheus dan Grafana:

```bash
docker compose --profile load down --remove-orphans
```

### Menghapus stack beserta seluruh time series dan konfigurasi lokal Grafana yang tersimpan dalam volume:

```bash
docker compose --profile load down --volumes --remove-orphans
```

### Perintah kedua bersifat destruktif terhadap evidence. Ekspor data atau dashboard yang diperlukan sebelum menghapus volume.

## Latihan Mandiri

### 1. Hitung estimasi penggunaan CPU host tanpa recording rule, lalu bandingkan hasil dan biaya query dengan recording rule.

### 2. Ubah interval scrape menjadi 5 detik dan 30 detik. Ukur dampak terhadap jumlah sampel serta kemampuan melihat spike singkat.

### 3. Buat alert filesystem tersisa di bawah 10 persen dan unit test menggunakan `promtool`.

### 4. Identifikasi label cAdvisor dengan cardinality tertinggi. Jelaskan label mana yang layak dipertahankan atau dipindahkan ke log.

### 5. Tambahkan panel network receive/transmit menggunakan `rate(node_network_receive_bytes_total[5m])` dan `rate(node_network_transmit_bytes_total[5m])`.

### 6. Hubungkan kenaikan CPU container dengan log Bab 6 pada rentang waktu yang sama dan jelaskan apakah korelasi tersebut cukup untuk menyatakan sebab-akibat.

### 7. Rancang SLI dan SLO sederhana untuk service aplikasi; jelaskan mengapa CPU tidak dapat menjadi pengganti langsung untuk availability pengguna.

## Rujukan Utama

### 1. [Prometheus — Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/).

### 2. [Prometheus — Query functions](https://prometheus.io/docs/prometheus/latest/querying/functions/).

### 3. [Prometheus — Recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/).

### 4. [Prometheus — Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/).

### 5. [Grafana — Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/).

### 6. [Grafana — Data sources](https://grafana.com/docs/grafana/latest/datasources/).

### 7. [Prometheus Node Exporter releases](https://github.com/prometheus/node_exporter/releases).

### 8. [Google cAdvisor releases](https://github.com/google/cadvisor/releases).

### Versi perangkat lunak berubah dari waktu ke waktu. Sebelum praktikum kelas, verifikasi tag image, catat `docker version` dan `docker compose version`, jalankan unit test rule, serta simpan digest image yang telah dinyatakan kompatibel.

![Screenshot 4](uploads/praktikum-07/image-04.png)

### Menyiapkan Direktori

```bash
mkdir -p ~/docker-lab/bab-6/{init,fluent-bit,queries,scripts}
cd ~/docker-lab/bab-6

```

Struktur file:

```text
bab-6/
├── .env.example
├── .env
├── .gitignore
├── compose.yaml
├── fluent-bit/
│   └── fluent-bit.conf
├── init/
│   └── 01-init-logging.sh
├── queries/
│   └── analysis.sql
└── scripts/
    ├── generate-logs.sh
    └── test-buffering.sh

```

#### File .env.example

> **Nama file:** `.env.example`  
> **Lokasi:** `~/docker-lab/bab-6/.env.example`  
> **Buka editor:** `nano .env.example`  
> **Tujuan:** `memusatkan parameter lab tanpa menaruh credential aktif pada compose.yaml.`  
![Screenshot 5](uploads/praktikum-07/image-05.png)

Aktifkan konfigurasi lokal:

![Screenshot 6](uploads/praktikum-07/image-06.png)

Nilai tersebut hanya untuk laptop laboratorium. File .env bukan secret manager dan tidak boleh digunakan sebagai pola penyimpanan secret di lingkungan produksi.

#### File .gitignore

> **Nama file:** `.gitignore`  
> **Lokasi:** `~/docker-lab/bab-6/.gitignore`  
> **Buka editor:** `nano .gitignore`  
![Screenshot 7](uploads/praktikum-07/image-07.png)

Volume PostgreSQL dan buffer Fluent Bit dikelola Docker sehingga tidak perlu dimasukkan ke repository.

#### File compose.yaml

> **Nama file:** `compose.yaml`  
> **Lokasi:** `~/docker-lab/bab-6/compose.yaml`  
> **Buka editor:** `nano compose.yaml`  
> **Petunjuk:** `Gunakan spasi, bukan tab. Semua nilai logging.options sengaja ditulis sebagai string.`  
```yaml
services:
  postgres-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      FLUENTBIT_DB_USER: ${FLUENTBIT_DB_USER}
      FLUENTBIT_DB_PASSWORD: ${FLUENTBIT_DB_PASSWORD}
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    networks:
      - logging-net
    healthcheck:
      test:
        - CMD-SHELL
        - "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"
      interval: 5s
      timeout: 5s
      retries: 12
      start_period: 10s
    restart: unless-stopped

  fluent-bit:
    image: fluent/fluent-bit:4.2.8
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      FLUENTBIT_DB_USER: ${FLUENTBIT_DB_USER}
      FLUENTBIT_DB_PASSWORD: ${FLUENTBIT_DB_PASSWORD}
    volumes:
      - ./fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf:ro
      - flb-storage:/var/log/flb-storage
    ports:
      - "127.0.0.1:24224:24224"
      - "127.0.0.1:2020:2020"
    networks:
      - logging-net
    depends_on:
      postgres-db:
        condition: service_healthy
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  generator-auth:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - |
        i=1
        while [ "$$i" -le 20 ]; do
          printf '{"service":"auth","severity":"INFO","event":"login_test","sequence":%s}\n' "$$i"
          i=$$((i + 1))
          sleep 1
        done
    logging:
      driver: fluentd
      options:
        fluentd-address: "127.0.0.1:24224"
        fluentd-async: "true"
        tag: "docker.{{.Name}}"
    depends_on:
      - fluent-bit

  generator-api:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - |
        i=1
        while [ "$$i" -le 20 ]; do
          printf '{"service":"api","severity":"WARN","event":"latency_test","duration_ms":%s}\n' "$$((i * 10))"
          i=$$((i + 1))
          sleep 1
        done
    logging:
      driver: fluentd
      options:
        fluentd-address: "127.0.0.1:24224"
        fluentd-async: "true"
        tag: "docker.{{.Name}}"
    depends_on:
      - fluent-bit

  generator-worker:
    image: alpine:3.20
    command:
      - /bin/sh
      - -c
      - |
        i=1
        while [ "$$i" -le 20 ]; do
          printf '{"service":"worker","severity":"ERROR","event":"retry_test","attempt":%s}\n' "$$i"
          i=$$((i + 1))
          sleep 1
        done
    logging:
      driver: fluentd
      options:
        fluentd-address: "127.0.0.1:24224"
        fluentd-async: "true"
        tag: "docker.{{.Name}}"
    depends_on:
      - fluent-bit

volumes:
  pg-data:
  flb-storage:

networks:
  logging-net:
    driver: bridge
```

![Screenshot 8](uploads/praktikum-07/image-08.png)

fluentd-async: "true" membuat Docker mencoba tersambung di latar belakang sehingga kegagalan collector tidak langsung menggagalkan startup container. Buffer driver Docker tetap terbatas dan berbeda dari buffer filesystem Fluent Bit; keduanya tidak menjamin kehilangan data mustahil terjadi.

#### File init/01-init-logging.sh

> **Nama file:** `01-init-logging.sh`  
> **Lokasi:** `~/docker-lab/bab-6/init/01-init-logging.sh`  
> **Buka editor:** `nano init/01-init-logging.sh`  
> **Tujuan:** `membuat objek database dengan nama dan tipe kolom yang sesuai kontrak plugin PostgreSQL Fluent Bit.`  
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

: "${POSTGRES_USER:?POSTGRES_USER belum diatur}"
: "${POSTGRES_DB:?POSTGRES_DB belum diatur}"
: "${FLUENTBIT_DB_USER:?FLUENTBIT_DB_USER belum diatur}"
: "${FLUENTBIT_DB_PASSWORD:?FLUENTBIT_DB_PASSWORD belum diatur}"

psql \
  --set=ON_ERROR_STOP=1 \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --set=fb_user="$FLUENTBIT_DB_USER" \
  --set=fb_password="$FLUENTBIT_DB_PASSWORD" <<'SQL'
SELECT format(
    'CREATE ROLE %I LOGIN PASSWORD %L',
    :'fb_user',
    :'fb_password'
)
WHERE NOT EXISTS (
    SELECT 1 FROM pg_roles WHERE rolname = :'fb_user'
)
\gexec

CREATE SCHEMA IF NOT EXISTS logs;

CREATE TABLE IF NOT EXISTS logs.fluentbit (
    tag  TEXT NOT NULL,
    time TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    data JSONB NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_fluentbit_time
    ON logs.fluentbit (time DESC);

CREATE INDEX IF NOT EXISTS idx_fluentbit_tag
    ON logs.fluentbit (tag);

CREATE INDEX IF NOT EXISTS idx_fluentbit_data_gin
    ON logs.fluentbit USING GIN (data);

CREATE OR REPLACE VIEW logs.recent_logs AS
SELECT
    time,
    tag,

    replace(data ->> 'container_name', '/', '') AS container_name,
    data ->> 'source' AS source,
    left(data ->> 'log', 200) AS log_preview
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 100;

SELECT format(
    'ALTER ROLE %I IN DATABASE %I SET search_path TO logs, public',
    :'fb_user',
    current_database()
)
\gexec

SELECT format('GRANT USAGE, CREATE ON SCHEMA logs TO %I', :'fb_user')
\gexec

SELECT format(
    'GRANT SELECT, INSERT ON TABLE logs.fluentbit TO %I',
    :'fb_user'
)
\gexec

SELECT format('GRANT SELECT ON TABLE logs.recent_logs TO %I', :'fb_user')
\gexec
SQL
```

![Screenshot 9](uploads/praktikum-07/image-09.png)

Hak CREATE pada schema diperlukan karena plugin PostgreSQL dapat melakukan bootstrap tabel. Untuk produksi, proses bootstrap sebaiknya dipisahkan dari akun ingest; setelah schema stabil, evaluasi pencabutan hak DDL dan uji kembali kompatibilitas plugin.

#### File fluent-bit/fluent-bit.conf

> **Nama file:** `fluent-bit.conf`  
> **Lokasi:** `~/docker-lab/bab-6/fluent-bit/fluent-bit.conf`  
> **Buka editor:** `nano fluent-bit/fluent-bit.conf`  
> **Tujuan:** `menerima protokol Forward, menahan backlog pada volume, serta meneruskan record ke PostgreSQL dan stdout.`  
```bash
[SERVICE]
    Flush                     1
    Daemon                    Off
    Log_Level                 info
    storage.path              /var/log/flb-storage
    storage.sync              normal
    storage.checksum          on
    storage.backlog.mem_limit 10M
    HTTP_Server               On
    HTTP_Listen               0.0.0.0
    HTTP_Port                 2020

[INPUT]
    Name          forward
    Listen        0.0.0.0
    Port          24224
    Mem_Buf_Limit 10M
    storage.type  filesystem

[OUTPUT]
    Name                     pgsql
    Match                    *
    Host                     postgres-db
    Port                     5432
    User                     ${FLUENTBIT_DB_USER}
    Password                 ${FLUENTBIT_DB_PASSWORD}
    Database                 ${POSTGRES_DB}
    Table                    fluentbit
    storage.total_limit_size 256M

[OUTPUT]
    Name   stdout
    Match  *
    Format json_lines
```

Keluaran stdout merupakan jalur observasi tambahan untuk praktikum. Pada sistem sibuk, keluaran ini dapat menambah volume log internal dan sebaiknya disesuaikan dengan kebutuhan.

![Screenshot 10](uploads/praktikum-07/image-10.png)

#### File queries/analysis.sql

> **Nama file:** `analysis.sql`  
> **Lokasi:** `~/docker-lab/bab-6/queries/analysis.sql`  
> **Buka editor:** `nano queries/analysis.sql`  
```sql
\pset pager off
\timing on

-- 1. Jumlah event dan rentang waktu.
SELECT
    count(*) AS total_event,
    min(time) AS event_pertama,
    max(time) AS event_terakhir
FROM logs.fluentbit;

-- 2. Distribusi berdasarkan tag Docker.
SELECT
    tag,
    count(*) AS total
FROM logs.fluentbit
GROUP BY tag
ORDER BY total DESC, tag;

-- 3. Metadata terstruktur dari Docker logging driver.
SELECT
    time,
    tag,
    replace(data ->> 'container_name', '/', '') AS container_name,
    data ->> 'source' AS source,
    left(data ->> 'log', 160) AS log_preview
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 15;

-- 4. Pencarian severity pada JSON yang masih berada di field log.
SELECT
    time,
    tag,
    data ->> 'log' AS log_message
FROM logs.fluentbit
WHERE data ->> 'log' ILIKE '%"severity":"ERROR"%'
ORDER BY time DESC;

-- 5. Ringkasan per menit untuk mengamati laju ingest.
SELECT
    date_trunc('minute', time) AS minute_bucket,
    count(*) AS total
FROM logs.fluentbit
GROUP BY minute_bucket
ORDER BY minute_bucket DESC
LIMIT 10;

-- Contoh retensi; tinjau dahulu sebelum menghapus data.
-- DELETE FROM logs.fluentbit
-- WHERE time < CURRENT_TIMESTAMP - INTERVAL '7 days';
```

![Screenshot 11](uploads/praktikum-07/image-11.png)

Pesan aplikasi berbentuk JSON masih disimpan sebagai string pada data.log, sedangkan metadata Docker berada pada objek data. Praktikum lanjutan dapat menambahkan parser Fluent Bit agar isi data.log diangkat menjadi field terstruktur tersendiri.

#### File scripts/generate-logs.sh

> **Nama file:** `generate-logs.sh`  
> **Lokasi:** `~/docker-lab/bab-6/scripts/generate-logs.sh`  
> **Buka editor:** `nano scripts/generate-logs.sh`  
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

docker compose up -d postgres-db fluent-bit

for attempt in {1..30}; do
    if curl -fsS http://127.0.0.1:2020/api/v1/health >/dev/null; then
        break
    fi

    if [[ "$attempt" -eq 30 ]]; then
        echo "[FAIL] Endpoint kesehatan Fluent Bit belum siap." >&2
        exit 1
    fi
    sleep 2
done

docker compose up -d --force-recreate \
    generator-auth \
    generator-api \
    generator-worker

echo "[PASS] Tiga generator telah dijalankan. Tunggu sekitar 25 detik."
echo "[INFO] Pantau dengan: docker compose ps"
```

![Screenshot 12](uploads/praktikum-07/image-12.png)

#### File scripts/test-buffering.sh

> **Nama file:** `test-buffering.sh`  
> **Lokasi:** `~/docker-lab/bab-6/scripts/test-buffering.sh`  
> **Buka editor:** `nano scripts/test-buffering.sh`  
> **Tujuan:** `membuktikan bahwa Fluent Bit menahan event ketika PostgreSQL dihentikan sementara, lalu mengirimkannya setelah backend pulih.`  
```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

count_rows() {
    docker compose exec -T postgres-db \
        psql \
        --username "$POSTGRES_USER" \
        --dbname "$POSTGRES_DB" \
        --tuples-only \
        --no-align \
        --command 'SELECT count(*) FROM logs.fluentbit;'
}

before="$(count_rows)"
echo "[INFO] Jumlah awal: $before"

docker compose stop postgres-db

docker compose up -d --force-recreate \
    generator-auth \
    generator-api \
    generator-worker

echo "[INFO] PostgreSQL berhenti; generator berjalan selama 25 detik."
sleep 25

docker compose start postgres-db

for attempt in {1..30}; do
    if docker compose exec -T postgres-db \
        pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB" >/dev/null 2>&1; then
        break
    fi

    if [[ "$attempt" -eq 30 ]]; then
        echo "[FAIL] PostgreSQL tidak kembali sehat." >&2
        exit 1
    fi
    sleep 2
done

for attempt in {1..60}; do
    after="$(count_rows)"
    if (( after > before )); then
        echo "[PASS] Jumlah sesudah pemulihan: $after"
        echo "[INFO] Kenaikan record: $((after - before))"
        exit 0
    fi
    sleep 2
done

echo "[FAIL] Tidak ada record baru setelah PostgreSQL pulih." >&2
echo "[INFO] Periksa: docker compose logs --tail 200 fluent-bit" >&2
exit 1
```

![Screenshot 13](uploads/praktikum-07/image-13.png)

Pengujian ini membuktikan pemulihan pada skala lab, bukan jaminan exactly-once. Retry dapat menghasilkan duplikasi, sedangkan buffer penuh, disk gagal, atau proses dihentikan secara paksa tetap dapat menyebabkan kehilangan event.

### Memberi Izin Eksekusi dan Memvalidasi Berkas

```bash
chmod +x init/01-init-logging.sh
chmod +x scripts/generate-logs.sh scripts/test-buffering.sh

bash -n init/01-init-logging.sh
bash -n scripts/generate-logs.sh
bash -n scripts/test-buffering.sh
docker compose config --quiet
```

![Screenshot 14](uploads/praktikum-07/image-14.png)

Validasi konfigurasi Fluent Bit tanpa menjalankan seluruh stack:

```bash
docker compose run --rm --no-deps fluent-bit \
  --dry-run \
  --config=/fluent-bit/etc/fluent-bit.conf
```

![Screenshot 15](uploads/praktikum-07/image-15.png)

Jika versi image tidak mengenali --dry-run, validasi dilakukan dengan menjalankan service dan memeriksa log startup:

```bash
docker compose up -d postgres-db fluent-bit
docker compose logs --tail 100 fluent-bit

```

### Menjalankan Pipeline dan Menghasilkan Log

```bash
./scripts/generate-logs.sh
sleep 25
docker compose ps --all
```

![Screenshot 16](uploads/praktikum-07/image-16.png)

Status Exited (0) pada ketiga generator setelah sekitar 20 detik merupakan hasil normal karena proses generator telah selesai. Service PostgreSQL dan Fluent Bit harus tetap Up.

### Memeriksa Monitoring Fluent Bit

```bash
curl -fsS http://127.0.0.1:2020/api/v1/health
curl -fsS http://127.0.0.1:2020/api/v1/metrics
```

![Screenshot 17](uploads/praktikum-07/image-17.png)

Endpoint monitoring hanya diikat ke 127.0.0.1. Jangan membuka endpoint internal ini ke jaringan publik tanpa autentikasi, pembatasan jaringan, dan kebijakan akses yang sesuai.

### Memverifikasi Record PostgreSQL

```bash
set -a
source .env
set +a

docker compose exec -T postgres-db \
  psql \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --file /dev/stdin \
  < queries/analysis.sql
```

![Screenshot 18](uploads/praktikum-07/image-18.png)

Pemeriksaan ringkas terhadap view:

```bash
docker compose exec -T postgres-db \
  psql \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --command 'TABLE logs.recent_logs;'
```

![Screenshot 19](uploads/praktikum-07/image-19.png)

Contoh evidence yang diharapkan (jumlah aktual dapat berbeda akibat waktu startup dan retry):

```bash
           tag            | total
--------------------------+-------
 docker.bab-6-generator-* |    20

 container_name           | source | log_preview
--------------------------+--------+------------------------------------------
 bab-6-generator-worker-1 | stdout | {"service":"worker","severity":...

```

### Menguji Buffering dan Pemulihan

Pastikan pipeline normal telah menghasilkan record, kemudian jalankan:

```bash
./scripts/test-buffering.sh
```

![Screenshot 20](uploads/praktikum-07/image-20.png)

Selama PostgreSQL berhenti, perhatikan retry Fluent Bit dan kondisi buffer:

```bash
docker compose logs --tail 200 fluent-bit
docker compose volume ls
```

![Screenshot 21](uploads/praktikum-07/image-21.png)

Kriteria utama bukan bahwa tidak ada pesan galat, melainkan bahwa kegagalan output terlihat, service collector tetap berjalan, dan jumlah record bertambah setelah PostgreSQL pulih.

Verifikasi tipe data dapat dilakukan dengan:

![Screenshot 22](uploads/praktikum-07/image-22.png)

```bash
docker compose exec -T postgres-db \
  psql \
  --username "$POSTGRES_USER" \
  --dbname "$POSTGRES_DB" \
  --command 'SELECT pg_typeof(data), count(*) FROM logs.fluentbit GROUP BY pg_typeof(data);'

```

### Cleanup

Menghentikan container tanpa menghapus data:

```bash
docker compose down --remove-orphans
```

Menghapus seluruh container dan volume lab (termasuk tabel log serta backlog Fluent Bit):

```bash
docker compose down --volumes --remove-orphans
```

Perintah kedua bersifat destruktif terhadap data praktikum. Gunakan hanya setelah evidence yang diperlukan telah disimpan.

