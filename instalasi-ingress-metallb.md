# 🌐 Cara Install MetalLB & NGINX Ingress Controller

### Panduan Lengkap instalasi MetalLB dan Ingress untuk akses aplikasi dari Luar Cluster!

---

## 📋 Nice To Have Cnocepts

### Penjelasan Sederhana

Bayangkan Anda sudah punya **aplikasi website** yang jalan di Kubernetes. Tapi masalahnya:
- Aplikasi hanya bisa diakses dari dalam cluster
- Kalau mau akses dari laptop, harus `kubectl port-forward` (ribet!)
- Kalau mau buka website dari HP atau komputer lain, tidak bisa

**Solusinya butuh 2 komponen:**

#### 1. **NGINX Ingress Controller**

**Analogi Sederhana:**
- Tanpa Ingress: Setiap aplikasi butuh IP sendiri (boros!)
- Dengan Ingress: **1 IP bisa untuk banyak aplikasi**, diatur berdasarkan domain name

#### 2. **MetalLB**

**Analogi Sederhana:**
- Tanpa MetalLB: Aplikasi Anda seperti rumah tanpa nomor, orang tidak tahu mau kirim surat ke mana
- Dengan MetalLB: Aplikasi dapat IP address sendiri (misal: 192.168.2.240), jadi semua orang di jaringan yang sama bisa akses

**Singkatnya:**
- **MetalLB** → Kasih IP address ke Ingress Controller
- **Ingress Controller** → Terima semua request dan arahkan ke aplikasi yang tepat berdasarkan domain

---

## 🎯 Apa yang Akan Anda Dapatkan?

Setelah mengikuti panduan ini:

✅ MetalLB terinstall dan memberikan IP External secara otomatis  
✅ NGINX Ingress Controller terinstall  
✅ Aplikasi bisa diakses dari browser dengan domain (misal: `http://myapp.local`)  
✅ Tidak perlu port-forward lagi!  
✅ Bisa hosting banyak aplikasi dengan 1 IP address saja  

---

## ⏱️ Berapa Lama Prosesnya?

- **Waktu:** 20-30 menit
- **Tingkat Kesulitan:** Mudah-Menengah
- **Yang Dibutuhkan:**
  - Cluster Kubernetes yang sudah jalan
  - Akses kubectl ke cluster
  - Tahu range IP address di jaringan Anda

---

## 📋 Persiapan: Cari IP Address yang Kosong

**SANGAT PENTING!** Sebelum install MetalLB, kita harus cari IP address yang:
- Masih kosong (tidak dipakai device lain)
- Dalam satu jaringan dengan cluster Kubernetes
- Tidak akan di-assign otomatis oleh router (di luar DHCP range)

### Cara Cek IP yang Kosong

Misalnya cluster ada di jaringan `192.168.2.x`:

**Langkah 1:** Cek IP range yang aman

Cari IP yang aman dan tidak digunakan oleh device lain. Di dokumentasi ini ktia memakai IP di range belakang, misalnya: `192.168.2.240` - `192.168.2.250`.

**Langkah 2:** Test apakah IP masih kosong

Dari server master, ping satu per satu:

```bash
ping -c 2 192.168.2.240
ping -c 2 192.168.2.241
ping -c 2 192.168.2.242
ping -c 2 192.168.2.243
ping -c 2 192.168.2.244
ping -c 2 192.168.2.245
ping -c 2 192.168.2.246
ping -c 2 192.168.2.247
ping -c 2 192.168.2.248
ping -c 2 192.168.2.249
ping -c 2 192.168.2.250
```

**Output yang bagus (IP kosong):**
```
From 192.168.2.104 icmp_seq=1 Destination Host Unreachable
```

**Output yang jelek (IP sudah dipakai):**
```
64 bytes from 192.168.2.240: icmp_seq=1 ttl=64 time=0.5 ms
```

📝 **CATAT:** Range IP yang kosong, misalnya: `192.168.2.240-192.168.2.250`

⚠️ **Kalau semua IP di range 240-250 sudah terpakai, coba range lain:** `192.168.2.230-192.168.2.239` atau `192.168.2.210-192.168.2.220`

---

## 📋 Langkah-Langkah Instalasi

### LANGKAH 1: Masuk ke Server Master

1. Buka **Terminal**
2. Login SSH ke node master:

```bash
ssh username@192.168.x.x
```

✅ **Sudah masuk? Lanjut!**

---

### LANGKAH 2: Install MetalLB

MetalLB adalah komponen pertama yang harus diinstall.

#### Langkah 2.1: Install MetalLB

Ketik perintah ini:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

**Penjelasan:** Perintah ini akan mendownload dan menginstall semua komponen MetalLB.

**Output:**
Akan muncul banyak baris seperti:
```
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io created
...
deployment.apps/controller created
daemonset.apps/speaker created
```

**Tunggu sampai selesai!** Sekitar 10-15 detik (jika internet lancar ya).

#### Langkah 2.2: Verifikasi Instalasi MetalLB

Cek apakah semua pods MetalLB sudah running:

```bash
kubectl get pods -n metallb-system
```

**Output awal (masih proses):**
```
NAME                          READY   STATUS              RESTARTS   AGE
controller-xxxxxxxxxx-xxxxx   0/1     ContainerCreating   0          20s
speaker-xxxxx                 0/1     Init:0/1            0          20s
speaker-yyyyy                 0/1     Init:0/1            0          20s
speaker-zzzzz                 0/1     Init:0/1            0          20s
```

**Tunggu 1-2 menit**, lalu ketik perintah yang sama lagi:

```bash
kubectl get pods -n metallb-system
```

**Output yang diharapkan (semua Running):**
```
NAME                          READY   STATUS    RESTARTS   AGE
controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
speaker-xxxxx                 1/1     Running   0          2m
speaker-yyyyy                 1/1     Running   0          2m
speaker-zzzzz                 1/1     Running   0          2m
```

**Catatan:** Jumlah pods `speaker` akan sama dengan jumlah nodes Anda (1 speaker per node).

✅ **Semua sudah `Running`? Lanjut ke konfigurasi!**

---

#### Langkah 2.3: Konfigurasi IP Pool MetalLB

Sekarang kita akan kasih tahu MetalLB: "Kamu boleh pakai IP range ini untuk dibagikan ke aplikasi."

##### Langkah 2.3.1: Buat File Konfigurasi

Buat file baru bernama `metallb-ip-pool.yaml`:

```bash
nano metallb-ip-pool.yaml
```

##### Langkah 2.3.2: Isi File Konfigurasi

Copy-paste konfigurasi di bawah ini ke dalam file:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.2.240-192.168.2.250 # sesuaikan dengan IP Pool yang ingin di assign ke MetalLB yaa
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2adv
  namespace: metallb-system
```

**⚠️ PENTING:** Ubah IP range `192.168.2.240-192.168.2.250` sesuai dengan range IP kosong yang Anda temukan di bagian Persiapan!

**Penjelasan:**
- **IPAddressPool:** Mendefinisikan range IP yang boleh dipakai MetalLB
- **L2Advertisement:** Mengumumkan IP ke jaringan lokal (Layer 2) agar bisa diakses dari device lain

**Simpan file:**
- Tekan `Ctrl + X`
- Ketik `Y`
- Tekan `Enter`

##### Langkah 2.3.3: Apply Konfigurasi

```bash
kubectl apply -f metallb-ip-pool.yaml
```

**Output:**
```
ipaddresspool.metallb.io/default-pool created
l2advertisement.metallb.io/l2adv created
```

##### Langkah 2.3.4: Verifikasi IP Pool

Cek apakah IP pool sudah terdaftar:

```bash
kubectl get ipaddresspool -n metallb-system
```

**Output:**
```
NAME           AUTO ASSIGN   AVOID BUGGY IPS   ADDRESSES
default-pool   true          false             ["192.168.2.240-192.168.2.250"]
```

✅ **IP Pool sudah terdaftar! MetalLB siap memberikan IP address!**

---

### LANGKAH 3: Install NGINX Ingress Controller

Setelah MetalLB siap, kita install NGINX Ingress Controller.

#### Langkah 3.1: Install NGINX Ingress

Ketik perintah ini:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
```

**Penjelasan:** Perintah ini menginstall NGINX Ingress Controller yang sudah dikonfigurasi untuk bare metal cluster (bukan cloud).

**Output:**
Akan muncul banyak baris seperti:
```
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
...
deployment.apps/ingress-nginx-controller created
```

**Tunggu sampai selesai!** Sekitar 15-20 detik.

#### Langkah 3.2: Verifikasi Instalasi

Cek apakah pods Ingress sudah running:

```bash
kubectl get pods -n ingress-nginx
```

**Tunggu 1-2 menit** sampai semua pods `Running`.

**Output yang diharapkan:**
```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-admission-create-xxxxx        0/1     Completed 0          2m
ingress-nginx-admission-patch-xxxxx         0/1     Completed 0          2m
ingress-nginx-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

**Catatan:** Pods `admission-create` dan `admission-patch` berstatus `Completed` adalah **NORMAL** (bukan error). Yang penting `ingress-nginx-controller` harus `Running`.

#### Langkah 3.3: Cek Service Ingress (The Magic Moment!)

Sekarang cek apakah MetalLB sudah memberikan IP address ke Ingress Controller:

```bash
kubectl get svc -n ingress-nginx
```

**Output yang diharapkan:**
```
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.97.56.159    192.168.2.240   80:32086/TCP,443:30953/TCP   3m
ingress-nginx-controller-admission   ClusterIP      10.108.237.57   <none>          443/TCP                      3m
```

**Lihat kolom `EXTERNAL-IP` pada baris `ingress-nginx-controller`!**

✅ **Kalau muncul IP address (misal: 192.168.2.240) bukan `<pending>`, berarti BERHASIL!**

📝 **CATAT IP ini!** Ini adalah IP address yang akan Anda gunakan untuk akses semua aplikasi.

⚠️ **Kalau masih `<pending>`:**
- Tunggu 1-2 menit lagi
- Ketik perintah `kubectl get svc -n ingress-nginx` lagi
- Kalau masih pending setelah 5 menit, cek kembali konfigurasi MetalLB IP Pool di Langkah 2.3

---

### LANGKAH 4: Setup Aplikasi dengan Ingress

Sekarang kita akan buat contoh aplikasi sederhana dan setup Ingress-nya.

**Catatan:** Kalau Anda sudah punya aplikasi sendiri, skip ke Langkah 4.3.

#### Langkah 4.1: Deploy Aplikasi Contoh (Opsional)

Kita akan deploy aplikasi NGINX sederhana sebagai contoh.

Buat file `demo-app.yaml`:

```bash
nano demo-app.yaml
```

Isi dengan:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo-service
  namespace: demo-app
spec:
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

Simpan (`Ctrl+X`, `Y`, `Enter`), lalu apply:

```bash
kubectl apply -f demo-app.yaml
```

#### Langkah 4.2: Verifikasi Aplikasi

```bash
kubectl get pods -n demo-app
```

Harusnya ada 2 pods nginx-demo yang `Running`.

---

#### Langkah 4.3: Buat Ingress Resource

Sekarang kita buat Ingress untuk mengarahkan traffic dari luar cluster ke aplikasi.

**Ada 2 jenis setup:**

##### OPTION A: Ingress untuk HTTP (Tanpa SSL)

Buat file `ingress-demo.yaml`:

```bash
nano ingress-demo.yaml
```

Isi dengan:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo-app
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-demo-service
            port:
              number: 80
```

**Penjelasan:**
- **host: demo.local** → Domain yang akan digunakan untuk akses aplikasi
- **ingressClassName: nginx** → Memberitahu Kubernetes untuk pakai NGINX Ingress Controller
- **service name: nginx-demo-service** → Nama service aplikasi yang akan menerima traffic (arahkan ke service nginx dari aplikasinya)

##### OPTION B: Ingress untuk HTTPS (Dengan SSL)

Buat file `ingress-demo-https.yaml`:

```bash
nano ingress-demo-https.yaml
```

Isi dengan:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  namespace: demo-app
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - demo-local-https
  rules:
  - host: demo-local-https
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-demo-service
            port:
              number: 443
```

**Penjelasan:**
- **host: demo-local-https** → Domain yang akan digunakan untuk akses aplikasi
- **ingressClassName: nginx** → Memberitahu Kubernetes untuk pakai NGINX Ingress Controller
- **nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"** → Memberitahu NGINX bahwa koneksi dari Ingress ke backend service menggunakan protokol HTTPS (bukan HTTP)
- **nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"** → Menonaktifkan verifikasi sertifikat SSL ketika NGINX terhubung ke backend (dipakai jika menggunakan self-signed certificate)
- **nginx.ingress.kubernetes.io/ssl-redirect: "true"** → Memaksa redirect dari HTTP ke HTTPS ketika client mengakses aplikasi

##### Apply Ingress

Pilih salah satu (HTTP atau HTTPS):

**Untuk HTTP:**
```bash
kubectl apply -f ingress-demo.yaml
```

**Untuk HTTPS:**
```bash
kubectl apply -f ingress-demo-https.yaml
```

#### Langkah 4.4: Verifikasi Ingress

```bash
kubectl get ingress -n demo-app
```

**Output:**
```
NAME           CLASS   HOSTS        ADDRESS         PORTS     AGE
demo-ingress   nginx   demo.local   192.168.2.x   80, 443   30s
```

**Lihat kolom `ADDRESS`** → Harus terisi dengan IP Node tempat Ingress Controller berada.

✅ **Ingress sudah terdaftar!**

---

### LANGKAH 5: Akses Aplikasi dari Browser

Sekarang saatnya akses aplikasi dari browser!

#### Langkah 5.1: Setup DNS Lokal (Untuk HTTP dengan domain .local)

Kalau Anda pakai domain custom seperti `demo.local`, Anda perlu tambahkan ke file `/etc/hosts`.

**Di Server Master:**

```bash
sudo nano /etc/hosts
```

Tambahkan baris ini di paling bawah:

```
192.168.2.240  demo.local
```

**⚠️ Ganti IP:** Sesuaikan dengan EXTERNAL-IP Ingress Controller Anda!

Simpan (`Ctrl+X`, `Y`, `Enter`).

**Di Device Windows:**

1. Buka Notepad **sebagai Administrator**
2. File → Open → `C:\Windows\System32\drivers\etc\hosts`
3. Ubah file type dari "Text Documents" ke "All Files"
4. Tambahkan baris:
   ```
   192.168.2.240  demo.local
   ```
5. Save

**Di Device Linux:**

```bash
sudo nano /etc/hosts
```

Tambahkan:
```
192.168.2.240  demo.local
```

#### Langkah 5.2: Buka Browser dan Akses!

**Untuk HTTP:**
```
http://demo.local
```

**Untuk HTTPS:**
```
https://demo-local-https
```

**Kalau muncul halaman "Welcome to nginx!" (atau halaman aplikasi jika langsung test dengan aplikasi yang sudah ada) berarti BERHASIL!** 🎉

---

## 🎨 Cara Menambahkan Aplikasi Lain

Setelah setup ini, Anda bisa hosting banyak aplikasi dengan 1 IP address saja!

### Contoh: Tambah Aplikasi Kedua

**1. Deploy aplikasi kedua** (misal: app2)

**2. Buat Ingress baru** dengan domain berbeda:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app2-ingress
  namespace: app2-namespace
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  ingressClassName: nginx
  rules:
  - host: app2.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

**3. Tambahkan domain ke `/etc/hosts`:**

```
192.168.2.240  app2.local
```

**4. Akses di browser:**
```
http://app2.local
```

**Semua aplikasi akan pakai IP yang sama (192.168.2.240), tapi dipisahkan berdasarkan domain name!**

---

## 😰 Troubleshooting - Kalau Ada Masalah

### Masalah 1: EXTERNAL-IP Ingress Stuck di `<pending>`

**Gejala:** `kubectl get svc -n ingress-nginx` menunjukkan `<pending>` terus

**Penyebab:** MetalLB belum berjalan dengan baik atau IP pool salah

**Solusi:**

1. **Cek pods MetalLB:**
```bash
kubectl get pods -n metallb-system
```
Semua harus `Running`.

2. **Cek IP pool:**
```bash
kubectl get ipaddresspool -n metallb-system
```
Pastikan addresses-nya benar.

3. **Cek log MetalLB controller:**
```bash
kubectl logs -n metallb-system deployment/controller
```
Lihat apakah ada error.

4. **Restart MetalLB:**
```bash
kubectl rollout restart deployment controller -n metallb-system
kubectl rollout restart daemonset speaker -n metallb-system
```

---

### Masalah 2: Tidak Bisa Akses `http://demo.local`

**Gejala:** Browser menampilkan "This site can't be reached"

**Solusi:**

1. **Cek apakah domain sudah di `/etc/hosts`:**
```bash
cat /etc/hosts | grep demo
```
Harus muncul: `192.168.2.240  demo.local`

2. **Test ping domain:**
```bash
ping demo.local
```
Harus resolve ke IP Ingress Controller.

3. **Test ping IP langsung:**
```bash
ping 192.168.2.240
```
Kalau tidak bisa ping, berarti IP tidak reachable dari komputer Anda. Cek apakah IP masih dalam range jaringan yang benar.

4. **Cek firewall:**
```bash
sudo ufw status
```
Pastikan port 80 dan 443 terbuka.

---

### Masalah 3: Aplikasi Menampilkan "404 Not Found"

**Gejala:** Bisa akses domain, tapi muncul halaman 404 dari NGINX

**Penyebab:** Ingress resource tidak cocok dengan service

**Solusi:**

1. **Cek Ingress:**
```bash
kubectl describe ingress demo-ingress -n demo-app
```
Lihat bagian "Backend" - harus mengarah ke service yang benar.

2. **Cek Service:**
```bash
kubectl get svc -n demo-app
```
Pastikan nama service sama dengan yang di Ingress.

3. **Cek Pods:**
```bash
kubectl get pods -n demo-app
```
Pastikan pods aplikasi berjalan.

4. **Cek log Ingress Controller:**
```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

---

### Masalah 4: HTTPS Tidak Bekerja

**Gejala:** HTTP bisa, tapi HTTPS error

**Solusi:**

1. **Pastikan annotations HTTPS ada:**
```bash
kubectl get ingress demo-ingress -n demo-app -o yaml
```
Cek apakah ada annotations:
- `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`
- `nginx.ingress.kubernetes.io/ssl-redirect: "true"`

2. **Test akses langsung dengan IP:**
```
https://192.168.2.240
```
Kalau bisa, berarti masalah di DNS/domain.

---

### Masalah 5: IP Address Conflict

**Gejala:** Setelah install MetalLB, device lain di jaringan jadi error

**Penyebab:** IP yang dipakai MetalLB ternyata sudah dipakai device lain

**Solusi:**

1. **Cek IP mana yang conflict:**
```bash
arp -a | grep 192.168.2.x
```

2. **Ubah IP pool MetalLB:**
Edit file `metallb-ip-pool.yaml`, ubah range IP-nya.

3. **Apply ulang:**
```bash
kubectl delete -f metallb-ip-pool.yaml
kubectl apply -f metallb-ip-pool.yaml
```

4. **Restart Ingress:**
```bash
kubectl delete pod -n ingress-nginx -l app.kubernetes.io/component=controller
```

---

## 🔧 Perintah Penting untuk Dicatat

```bash
# Cek pods MetalLB
kubectl get pods -n metallb-system

# Cek IP pool MetalLB
kubectl get ipaddresspool -n metallb-system

# Cek service Ingress (lihat EXTERNAL-IP)
kubectl get svc -n ingress-nginx

# Cek pods Ingress
kubectl get pods -n ingress-nginx

# Cek semua Ingress resources
kubectl get ingress -A

# Cek detail Ingress tertentu
kubectl describe ingress <nama-ingress> -n <namespace>

# Cek log Ingress Controller
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Cek log MetalLB
kubectl logs -n metallb-system deployment/controller

# Restart Ingress Controller
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx

# Restart MetalLB
kubectl rollout restart deployment controller -n metallb-system
kubectl rollout restart daemonset speaker -n metallb-system
```

---

## 📚 Konsep Penting yang Perlu Dipahami

### 1. Perbedaan Service Type

**ClusterIP** (Default):
- Hanya bisa diakses dari dalam cluster
- Tidak punya IP eksternal
- Cocok untuk komunikasi antar service

**NodePort**:
- Bisa diakses dari luar via `<IP-Node>:<Port>`
- Port-nya high number (30000-32767)
- Tidak praktis untuk production

**LoadBalancer**:
- Dapat EXTERNAL-IP dari MetalLB
- Paling praktis untuk bare metal cluster
- Ingress Controller pakai type ini

### 2. Cara Kerja Ingress

```
Internet/Browser
    ↓
[EXTERNAL-IP dari MetalLB: 192.168.2.240]
    ↓
[NGINX Ingress Controller]
    ↓ (cek domain name)
    ├─ app1.local → Service app1 → Pods app1
    ├─ app2.local → Service app2 → Pods app2
    └─ app3.local → Service app3 → Pods app3
```

Satu IP, banyak aplikasi, dipisah berdasarkan domain!

### 3. DNS Resolution

**Tanpa DNS Server:**
- Pakai `/etc/hosts` (manual, hanya di komputer Anda)
- Pakai sslip.io (otomatis, tapi domain aneh)

**Dengan DNS Server** (Advanced):
- Setup DNS server sendiri (BIND, CoreDNS)
- Atau pakai router yang support custom DNS

---

## 🎓 Kesimpulan

**Apa yang sudah Anda capai:**

✅ MetalLB terinstall dan memberikan IP pool  
✅ NGINX Ingress Controller terinstall dengan EXTERNAL-IP  
✅ Aplikasi bisa diakses dari browser dengan domain  
✅ Setup untuk HTTP dan HTTPS  
✅ Paham cara kerja MetalLB + Ingress  

**Setup saat ini:**
- **1 IP Address** (dari MetalLB) bisa untuk **banyak aplikasi**
- **Akses mudah** dengan domain name (tidak perlu ingat port)
- **Production-ready** untuk bare metal cluster

---