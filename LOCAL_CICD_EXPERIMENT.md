# Eksperimen CI/CD Lokal — Latihan End-to-End (GitHub → Jenkins → Docker → Kubernetes)

> Tujuan: memahami *feel* rantai CI/CD lengkap di laptop sendiri **sebelum** menyentuh
> repo/infra asli. Kamu akan mereplika versi mini dari pipeline `Jenkinsfile.dev`:
> **push ke GitHub → Jenkins mendeteksi → build image → push ke registry lokal →
> deploy ke Kubernetes lokal → cek aplikasi jalan**.
>
> Environment yang diasumsikan: **Ubuntu 22.04 LTS**, Docker sudah terpasang.
> Semua langkah pakai **project dummy terpisah**, bukan repo asli.

---

## 0. Peta Konsep — apa yang sedang kamu latih

```
[Laptop kamu]
   kamu push commit
        │
        ▼
[GitHub repo personal]  ← "sumber kebenaran" kode
        │  (Jenkins menarik / poll — BUKAN GitHub yang push, karena Jenkins lokal internal)
        ▼
[Jenkins lokal]  ← otak CI/CD: baca Jenkinsfile, jalankan tahap-tahap
        │
        ├─► docker build  → bikin image dari Dockerfile
        │
        ├─► docker push   → simpan image ke [Registry lokal]
        │
        └─► kubectl ...   → suruh [Kubernetes lokal] pakai image baru
                                    │
                                    ▼
                            [Pod aplikasi jalan] → buka di browser
```

Lima komponen yang harus hidup: **GitHub repo**, **Jenkins**, **Docker**, **Registry**,
**Kubernetes**. Bagian tersulit di dunia nyata (firewall, SSH ke node, secret di disk)
kita sederhanakan supaya fokus ke *alur*-nya dulu.

---

## 1. Cek dulu: minikube atau kubernetes "asli"?

Kamu lupa yang mana yang terpasang. Jalankan ini untuk memastikan:

```bash
# Apakah minikube ada?
which minikube && minikube version || echo "minikube TIDAK ada"

# Apakah kubectl ada, dan menunjuk ke cluster apa?
which kubectl && kubectl config current-context || echo "kubectl belum terkonfigurasi"

# Lihat semua context yang terdaftar
kubectl config get-contexts 2>/dev/null

# Apakah ada cluster yang benar-benar jalan?
kubectl get nodes 2>/dev/null || echo "belum ada cluster aktif"
```

Interpretasi hasil:

| Yang muncul | Artinya |
|---|---|
| `minikube version` keluar angka | Kamu punya **minikube** |
| context bernama `minikube` | Cluster aktifmu minikube |
| context `docker-desktop` | Kubernetes bawaan Docker Desktop |
| context `kind-...` | Pakai **kind** (Kubernetes in Docker) |
| `microk8s` di `which` | Pakai **MicroK8s** (umum di Ubuntu via snap) |
| semua kosong | Belum ada cluster — ikuti langkah 2 |

> **Rekomendasi untuk latihan ini: pakai `minikube`.** Paling gampang, punya registry
> add-on bawaan, dan mudah di-reset (`minikube delete`) kalau eksperimenmu kacau —
> tanpa risiko ke apa pun. Panduan di bawah pakai minikube. Kalau kamu sudah nyaman
> dengan MicroK8s/kind, konsepnya sama, hanya beda perintah.

---

## 2. Menyiapkan Kubernetes lokal (minikube)

### 2.1 Install (kalau belum ada)

```bash
# minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

# kubectl (klien untuk bicara ke cluster)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl
```

### 2.2 Nyalakan cluster + registry lokal

```bash
# start pakai driver docker
minikube start --driver=docker

# aktifkan registry internal minikube (pengganti registry-local.intra.cmk.co.id versi asli)
minikube addons enable registry

# verifikasi
kubectl get nodes            # harus muncul 1 node "Ready"
```

> **Analogi ke pipeline asli:** `minikube` = cluster kube di kantor,
> `addons registry` = `registry-local.intra.cmk.co.id`. Bedanya semua di laptopmu.

---

## 3. Menyiapkan project dummy + repo GitHub personal

Kita **tidak** pakai aplikasi berat seperti project asli. Cukup web sederhana supaya
fokus ke alur, bukan ke waktu build.

### 3.1 Buat folder project dummy (di luar repo asli)

```bash
mkdir -p ~/cicd-lab && cd ~/cicd-lab
git init -b main
```

### 3.2 Aplikasi mini + Dockerfile

`index.html`:
```bash
cat > index.html <<'EOF'
<!doctype html>
<html><body style="font-family:sans-serif;text-align:center;margin-top:15%">
  <h1>CI/CD Lab</h1>
  <p>Versi: <b>__VERSION__</b></p>
</body></html>
EOF
```

`Dockerfile` (super ringan, pakai nginx):
```bash
cat > Dockerfile <<'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
```

### 3.3 Manifest Kubernetes (versi mini dari Deployment asli)

Di pipeline asli, Deployment **sudah ada** di cluster dan Jenkins hanya `set image`.
Untuk latihan, kita buat dulu Deployment + Service-nya sekali:

`k8s.yaml`:
```bash
cat > k8s.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cicd-lab-development          # tiru pola "<project>-<env>"
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels: { app: cicd-lab-development }
  template:
    metadata:
      labels: { app: cicd-lab-development }
    spec:
      containers:
        - name: cicd-lab-development
          image: localhost:5000/cicd-lab:development  # akan diganti Jenkins
          ports: [ { containerPort: 80 } ]
---
apiVersion: v1
kind: Service
metadata:
  name: cicd-lab-development
  namespace: default
spec:
  selector: { app: cicd-lab-development }
  ports: [ { port: 80, targetPort: 80 } ]
EOF
```

### 3.4 Jenkinsfile mini (meniru tahap-tahap Jenkinsfile.dev)

`Jenkinsfile`:
```bash
cat > Jenkinsfile <<'EOF'
pipeline {
    agent any
    options { timestamps(); disableConcurrentBuilds() }
    environment {
        IMAGE_NAME  = 'cicd-lab'
        REGISTRY    = 'localhost:5000/cicd-lab'   // registry lokal (port-forward minikube)
        DEPLOYMENT  = 'cicd-lab-development'
        NAMESPACE   = 'default'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }                // Jenkins yang ambil kode (bukan git clone manual)
        }
        stage('Build & Push Image') {
            steps {
                script {
                    def sha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.IMAGE_TAG  = "dev-${sha}"
                    env.FULL_IMAGE = "${env.REGISTRY}:${env.IMAGE_TAG}"
                    sh """
                        sed -i "s/__VERSION__/${env.IMAGE_TAG}/" index.html
                        docker build -t ${env.FULL_IMAGE} .
                        docker push ${env.FULL_IMAGE}
                        docker tag ${env.FULL_IMAGE} ${env.REGISTRY}:development
                        docker push ${env.REGISTRY}:development
                    """
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl -n ${NAMESPACE} set image deployment/${DEPLOYMENT} \
                        ${DEPLOYMENT}=${env.FULL_IMAGE}
                    kubectl -n ${NAMESPACE} rollout restart deployment/${DEPLOYMENT}
                    kubectl -n ${NAMESPACE} rollout status deployment/${DEPLOYMENT} --timeout=120s
                """
            }
        }
    }
    post {
        success { echo "OK deployed ${env.FULL_IMAGE}" }
        failure { echo "GAGAL — cek log stage di atas" }
    }
}
EOF
```

> Perhatikan: struktur stage-nya **sama polanya** dengan `Jenkinsfile.dev` asli
> (Checkout → Build & Push → Deploy + rollout status), hanya disederhanakan dan
> tanpa SSH ke node karena Jenkins & kube ada di mesin yang sama.

### 3.5 Push ke GitHub personal

Buat repo kosong di `github.com/<username-kamu>/cicd-lab` (private boleh), lalu:

```bash
git add .
git commit -m "chore: init cicd lab"
git remote add origin git@github.com:<username-kamu>/cicd-lab.git
git push -u origin main
```

> Kalau `git push` via SSH gagal, kamu perlu daftarkan SSH key personal ke
> **GitHub → Settings → SSH and GPG keys** (ini key akun-mu, beda dengan deploy key).
> Tes: `ssh -T git@github.com`.

---

## 4. Menyiapkan Jenkins lokal

Cara termudah: jalankan Jenkins sebagai container Docker yang **bisa memakai Docker &
kubectl milik host**.

### 4.1 Jalankan Jenkins

```bash
docker run -d --name jenkins-lab \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $HOME/.kube:/root/.kube \
  -v $HOME/.minikube:$HOME/.minikube \
  --group-add $(getent group docker | cut -d: -f3) \
  jenkins/jenkins:lts

# ambil password admin awal
docker exec jenkins-lab cat /var/jenkins_home/secrets/initialAdminPassword
```

Penjelasan mount penting:
- `docker.sock` → supaya `docker build` di dalam Jenkins memakai Docker host.
- `~/.kube` + `~/.minikube` → supaya `kubectl` di Jenkins bisa bicara ke cluster lokalmu.

### 4.2 Setup awal via browser

1. Buka `http://localhost:8080`, tempel password admin di atas.
2. Pilih **Install suggested plugins**.
3. Buat user admin.
4. Pastikan plugin **Git** dan **Pipeline** terpasang (biasanya sudah dari suggested).

### 4.3 Pasang `docker` CLI & `kubectl` di dalam kontainer Jenkins

Container `jenkins/jenkins:lts` belum punya `docker`/`kubectl` binary:

```bash
docker exec -u root jenkins-lab bash -c '
  apt-get update &&
  apt-get install -y docker.io curl &&
  curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" &&
  install -m0755 kubectl /usr/local/bin/kubectl && rm kubectl
'
# verifikasi
docker exec jenkins-lab docker version --format '{{.Server.Version}}'
docker exec jenkins-lab kubectl get nodes
```

Kalau `kubectl get nodes` dari dalam Jenkins berhasil → jembatan Jenkins→kube sudah nyambung.

---

## 5. Menyambungkan registry lokal

Supaya `docker push localhost:5000/...` masuk ke registry minikube, buka port-forward:

```bash
# jalankan di terminal terpisah, biarkan hidup
kubectl port-forward --namespace kube-system service/registry 5000:80
```

Test manual sekali:
```bash
docker pull hello-world
docker tag hello-world localhost:5000/hello:test
docker push localhost:5000/hello:test    # kalau sukses, registry siap
```

> **Analogi asli:** `localhost:5000` = `registry-local.intra.cmk.co.id`.

---

## 6. Membuat Deployment awal di kube (sekali saja)

Pipeline hanya melakukan `set image` pada deployment yang **sudah ada**. Jadi buat dulu:

```bash
cd ~/cicd-lab
# build & push image awal supaya deployment punya sesuatu untuk dijalankan
docker build -t localhost:5000/cicd-lab:development .
docker push localhost:5000/cicd-lab:development
kubectl apply -f k8s.yaml
kubectl rollout status deployment/cicd-lab-development
```

Cek aplikasi:
```bash
kubectl port-forward service/cicd-lab-development 8081:80
# buka http://localhost:8081 → harus muncul "CI/CD Lab"
```

---

## 7. Membuat Job di Jenkins (Poll SCM — meniru skenario asli GHEC + Jenkins internal)

Karena di dunia nyata Jenkins-mu internal & GitHub di cloud, kita latih pola **Poll SCM**
(Jenkins menarik, bukan webhook masuk).

1. Jenkins → **New Item** → nama `cicd-lab-dev` → **Pipeline** → OK.
2. Bagian **Pipeline**:
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**
   - Repository URL: `git@github.com:<username-kamu>/cicd-lab.git`
   - Credentials: tambahkan **SSH Username with private key** (username `git`, isi private
     key yang public-nya terdaftar di GitHub). Untuk lab boleh pakai key personal-mu.
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
3. **Build Triggers** → centang **Poll SCM**, schedule: `H/2 * * * *`.
4. Save.

> **Ini momen kunci** yang menjawab kebingunganmu sebelumnya: dengan "Pipeline script
> from SCM" + Poll SCM, Jenkins tahu commit mana yang baru → itulah yang memicu build.
> Bukan `git clone` manual di dalam stage.

---

## 8. Uji end-to-end 🎯

```bash
cd ~/cicd-lab
# ubah sesuatu
sed -i 's/CI\/CD Lab/CI\/CD Lab (update 1)/' index.html
git commit -am "feat: ubah judul"
git push
```

Lalu:
1. Tunggu ≤ 2 menit (atau di Jenkins job klik **Build Now** untuk memaksa).
2. Lihat Jenkins → job `cicd-lab-dev` → build berjalan → cek **Console Output**.
3. Perhatikan tahap: Checkout → Build & Push → Deploy → `rollout status`.
4. Setelah sukses, cek pod baru:
   ```bash
   kubectl get pods -l app=cicd-lab-development
   kubectl rollout history deployment/cicd-lab-development
   ```
5. Refresh `http://localhost:8081` → judul berubah = **rantai penuh berhasil**.

Kalau ini jalan, kamu sudah merasakan persis apa yang dilakukan `Jenkinsfile.dev` asli,
minus kerumitan SSH-ke-node & firewall.

---

## 9. Latihan tambahan (biar makin paham pola aslinya)

- **Simulasi dua environment di satu cluster** (persis concern mentormu): buat
  `k8s.yaml` kedua dengan deployment `cicd-lab-production`, branch `main`→prod dan
  buat branch `dev`→dev. Buat 2 job Jenkins terpisah. Amati bahwa `set image` ke
  `cicd-lab-development` **tidak** menyentuh `cicd-lab-production`. Ini membuktikan
  isolasi lewat *nama deployment*, bukan namespace.
- **Simulasi rollback**: bikin Dockerfile yang sengaja gagal start, lihat
  `rollout status` gagal, lalu jalankan `kubectl rollout undo`.
- **Simulasi migrasi DB gated**: tambah stage yang `kubectl run` pod one-shot
  (`--restart=Never`) yang mencetak sesuatu lalu exit 0/1, tiru pola stage
  "Database Migration".

---

## 10. Membersihkan setelah selesai

```bash
docker rm -f jenkins-lab
docker volume rm jenkins_home
minikube delete            # hapus seluruh cluster lab — aman, tidak ada infra asli
```

---

## Peta padanan Lab → Produksi (asli)

| Di lab ini | Di pipeline asli (`Jenkinsfile.dev`) |
|---|---|
| `minikube` | Cluster Kubernetes kantor (node `172.27.20.70`) |
| `localhost:5000` | `registry-local.intra.cmk.co.id` |
| Jenkins di Docker lokal | Jenkins server internal |
| `checkout scm` (Poll SCM) | Trigger + `git clone` (perlu deploy key GitHub) |
| `kubectl` langsung (satu mesin) | `ssh mimin-kube@172.27.20.70 -> kubectl` |
| `cicd-lab-development` | `new-corp-apps-myapps-development` |
| nginx + index.html | .NET 8 app (Dockerfile multi-stage) |
| tanpa secret | `appsettings.json` di-inject dari `/var/lib/jenkins/workspace/secret/...` |

> Setelah nyaman dengan lab ini, langkah ke repo asli tinggal: pakai **deploy key**
> (bukan key personal), ganti registry & perintah kube pakai **SSH ke node**, dan
> pastikan pemetaan **job ↔ branch ↔ Jenkinsfile** benar (dev→`dev`+`Jenkinsfile.dev`,
> prod→`main`+`Jenkinsfile`).
