## Task 1 — Terraform

### 1a: NACL Fixes
1. NACL itu (Stateless)
Beda sama Security Group yang ingat riwayat koneksi (stateful), NACL itu enggak. Tiap ada paket data masuk atau keluar, dia bakal ngecek ulang dari nol tanpa peduli sebelumnya udah pernah komunikasi atau belum.

2. Dropped Silently
Gara-gara pakai fitur VPC CNI, IP pod berubah pakai rentang 100.64.0.0/16. Padahal, aturan di NACL cuma ngizinin IP node (10.10.64.0/20). Alhasil, paket dari pod langsung dibuang begitu saja tanpa suara, dan server enggak ngasih tahu si pengirim kalau paketnya ditolak (enggak ada balasan kayak TCP RST).

3. Buat Aplikasi Hanging & Solusinya
Karena enggak ada kabar kalau paketnya ditolak, aplikasi di dalam pod jadi nunggu terus (hanging) sampai timeout. Inilah yang bikin masalah kemarin susah ditebak.
Solusinya: Tinggal tambahin aturan di NACL supaya IP pod (100.64.0.0/16) diizinkan lewat untuk port database (5432), Redis (6379), dan port jalur pulang (1024-65535).

### 1b RDS Module — Bugs Fixed
Memperbaiki urutan resource terraform. Blok ingress dan egress yang nempel langsung di dalam resource utama dipisah keluar menjadi resource independen, yaitu aws_security_group_rule (rds_ingress_proxy dan rds_proxy_egress_rds). Dengan begitu Terraform bisa baca urutan yang sesuai.

### 1b: RDS Proxy — Design Decisions
* **Connection Pooling & Scalability:** Mengonfigurasi `aws_db_proxy.app` di dalam *protected subnets* untuk mengelola lonjakan koneksi bersamaan (*concurrency*) dari aplikasi, mencegah *connection exhaustion* pada database utama.
* **Secret Management & Security:** Mengintegrasikan autentikasi RDS Proxy menggunakan AWS Secrets Manager (`auth_scheme = "SECRETS"`) dan mewajibkan komunikasi terenkripsi melalui `require_tls = true`.

## Task 2 — Kubernetes

### 2a: NodePool Design Decisions
* **Instance Family Selection:** 
  * `helios-ai-app` menggunakan keluarga *Compute/Memory-optimized* (`c`, `m`) untuk memastikan latensi rendah pada API Gateway dan penanganan *throughput* tinggi untuk kebutuhan LLM.
  * `helios-ai-db` (atau *cache/stateful pool*) menggunakan keluarga *Memory-optimized* (`m`, `r`) guna mengakomodasi kebutuhan penyimpanan *in-memory* yang besar.
* **Spot vs. On-Demand Strategy:** 
  * NodePool aplikasi diatur menggunakan kombinasi *Spot* dan *On-Demand* untuk menyeimbangkan efisiensi biaya operasional (*cost-efficiency*) dan ketersediaan layanan.
  * NodePool database/cache dikunci menggunakan kapasitas *On-Demand* murni untuk menjamin stabilitas dan mencegah gangguan operasional pada komponen kritikal.
* **Disruption / Consolidation Policy:** Mengaktifkan `consolidationPolicy: WhenEmptyOrUnderutilized` dengan waktu tunggu pendek (`consolidateAfter: 30s` / `60s`) agar Karpenter dapat secara agresif mendaur ulang node yang kurang optimal dan mengemas pod ke node lain secara efisien guna memangkas biaya cloud.
* **Taints & Tolerations:** Menerapkan *taints* khusus (`workload-type=stateful-cache:NoSchedule`) pada *NodePool* database/cache untuk memastikan hanya *workload* terkait yang memiliki *tolerations* sesuai dapat dijadwalkan ke node tersebut, menjaga isolasi keamanan yang baik.

### 2b: KEDA ScaledObject — Bugs Fixed
* **Fallback Configuration (Bug 1):** Menambahkan blok fallback agar ketika backend metrics (Mimir/Prometheus) mengalami gangguan, KEDA tidak melakukan *scale-down* ke 0 yang dapat menyebabkan *outage* total, melainkan mempertahankan jumlah replika minimum yang aman (replicas: 2).
* **In-Cluster Metrics Endpoint (Bug 2):** Mengubah `serverAddress` dari URL publik eksternal (https://prometheus.helios-ai.io) ke layanan internal klaster Mimir (http://mimir-svc.monitoring.svc.cluster.local:9009/prometheus) guna mengurangi latensi jaringan, memastikan keterjangkauan (*reachability*), dan mencegah kebocoran metrik ke luar.
* **Per-Replica Threshold Adjustment (Bug 3):** Memperbaiki nilai *threshold* pada metrik *in-flight requests* dari "1000" menjadi "10" agar penskalaan aktif secara responsif sesuai kapasitas aman per *pod* (~10 Req bersamaan).
* **Malformed PromQL Query Fix (Bug 4):** Memperbaiki query PromQL P99 latency yang sebelumnya menggunakan metrik mentah menjadi penggunaan fungsi agregasi `rate()` dengan jendela waktu 5 menit (`histogram_quantile(0.99, sum(rate(litellm_request_duration_seconds_bucket{...}[5m])) by (le))`).
* **CPU Utilization Threshold Fix (Bug 5):** Memperbaiki nilai *threshold* utilitas CPU dari `"200"` (200%, yang mustahil dicapai pada single core) menjadi `"70"` untuk merepresentasikan utilitas normal sebesar 70%.

## Task 3 — Incident

### 3a: Root Cause Analysis
Insiden ini disebabkan oleh empat akar masalah independen yang terjadi secara bersamaan di lapisan infrastruktur:

- Root Cause 1: Kegagalan Koneksi Database (helios-api & helios-proxy)

 Pod API mengalami timeout saat mencoba terhubung ke RDS Proxy (ETIMEDOUT 10.10.16.45:5432) dan gagal terkumpul (PrismaClientInitializationError), menghasilkan error 503 Service Unavailable.

Evidence:

Log aplikasi: Error: connect ETIMEDOUT 10.10.16.45:5432 dan All DB connection attempts exhausted. Returning 503.

AWS CLI RDS Proxy output: "VpcSubnetIds": ["subnet-0web1aaa", "subnet-0web2bbb", "subnet-0web3ccc"] — RDS Proxy salah ditempatkan di web subnets (publik) alih-alih protected subnets, sehingga terisolasi atau salah rute dari jaringan internal pod.

- Root Cause 2: Pemblokiran Trafik Pod oleh Network ACL (NACL) Protected Subnets

 Trafik jaringan dari pod EKS ke layanan backend di protected subnets (RDS/Redis) diblokir secara diam-diam (silent drop) oleh aturan NACL.

Evidence:

IP Pod: 100.64.4.17 (berasal dari rentang VPC CNI prefix delegation 100.64.0.0/16).

AWS NACL output: Aturan ingress (RuleNumber 100 & 110) hanya mengizinkan CIDR block node EKS (10.10.64.0/20), tidak memasukkan CIDR block pod (100.64.0.0/16). Karena NACL bersifat stateless, paket dari IP pod langsung didrop tanpa TCP RST.

- Root Cause 3: Kegagalan Koneksi Worker ke Redis (Valkey) Akibat Konflik TLS

 helios-worker gagal terhubung ke ElastiCache Valkey (ECONNREFUSED pada port 6379).

Evidence:

Log aplikasi: Redis URL: redis://master.helios-ai-prod-valkey.abc.apse3.cache.amazonaws.com:6379 dan Job queue unavailable — worker entering idle state.

AWS ElastiCache output: "TransitEncryptionEnabled": true dan "TransitEncryptionMode": "required".

Kubectl describe pod: REDIS_SSL: "false". Aplikasi mencoba melakukan koneksi plaintext (non-TLS) ke cluster Redis yang mewajibkan enkripsi transit (TLS required), sehingga koneksi langsung ditolak.

- Root Cause 4: Kegagalan LLM Gateway Upstream Connection

 Pod LLM Gateway mengalami downstream/upstream reset karena tidak dapat berkomunikasi dengan dependensi backend (API/DB) yang sedang lumpuh.

Evidence: Log aplikasi: upstream connect error or disconnect/reset before headers. reset reason: connection failure.

### 3b: Remediation Plan
1. Perbaikan Konfigurasi RDS Proxy (Terraform / modules/rds/main.tf)
Ubah penempatan subnet IDs pada resource aws_db_proxy.app dari web subnets ke protected subnets (sudah dilakukan di soal 1b)

2. Perbaikan NACL untuk VPC CNI Prefix Delegation (Terraform / modules/networking/nacl.tf atau AWS CLI)
Tambahkan aturan ingress dan egress pada NACL protected subnet untuk mengizinkan rentang IP Pod (100.64.0.0/16)

# Perintah AWS CLI untuk menambah rule ingress pod CIDR (Port 5432 & 6379)
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxxx --ingress --rule-number 90 --protocol tcp --port-range From=5432,To=5432 --rule-action allow --cidr-block 100.64.0.0/16
aws ec2 create-network-acl-entry --network-acl-id acl-xxxxxx --ingress --rule-number 95 --protocol tcp --port-range From=6379,To=6379 --rule-action allow --cidr-block 100.64.0.0/16

3. Perbaikan Environment Variable Worker Redis (Kubernetes Deployment Manifest)
Ubah konfigurasi REDIS_SSL pada deployment helios-worker dan helios-api menjadi "true" agar sesuai dengan kebijakan ElastiCache:

env:
  - name: REDIS_URL
    value: "rediss://master.helios-ai-prod-valkey.abc.apse3.cache.amazonaws.com:6379" # Gunakan rediss:// untuk TLS
  - name: REDIS_SSL
    value: "true" # Perbaiki dari "false" menjadi "true"

### 3c: Prevention
1. IaC Enforcement & Guardrails (Terraform Compliance):

Terapkan policy-as-code (menggunakan OPA/Conftest atau Checkov) di pipeline CI/CD untuk memvalidasi bahwa resource database, RDS Proxy, dan cache tidak boleh ditarik ke dalam web/public subnets, serta wajib berada di protected/isolated subnets.

2. Automated CI Validation for Network Policies:

Buat unit test Terraform otomatis yang memvalidasi aturan NACL dan Security Group agar selalu menyertakan blok CIDR 100.64.0.0/16 (VPC CNI prefix delegation) untuk mencegah pemotongan trafik secara tidak sengaja.

3. Monitoring & Proactive Alerting:

Pasang alert berbasis Prometheus/Alertmanager yang mendeteksi lonjakan error koneksi database (ETIMEDOUT / ECONNREFUSED) serta ketidakcocokan konfigurasi TLS (seperti percobaan plaintext ke endpoint ElastiCache berkemampuan TLS) sebelum berdampak pada outage layanan end-user.
