# Paket P6 — Deploy aq-service ke Kubernetes (SS-302)

Prasyarat: cluster k3d dari tugas persiapan P6 (`k3d cluster create ss302`) dan image `aq-service` dari LKM 5.

```bash
kubectl config current-context        # harus k3d-ss302
kubectl get nodes                     # status Ready
```

## 1. Siapkan image (pilih salah satu)

**A. Dari GHCR (image private).** Ganti `OWNER` di `k8s/02-deployment.yaml`, lalu:

```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl -n ss302 create secret docker-registry ghcr-cred \
  --docker-server=ghcr.io --docker-username=<username-github> --docker-password=$CR_PAT
```

**B. Tanpa registry** (Wi-Fi lambat atau GHCR bermasalah). Ubah baris `image:` menjadi `aq-service:0.1.0`, lalu:

```bash
docker build -t aq-service:0.1.0 <folder-starter-P5>
k3d image import aq-service:0.1.0 -c ss302
```

## 2. Deploy dan amati

```bash
kubectl apply -f k8s/00-namespace.yaml -f k8s/01-configmap.yaml \
              -f k8s/02-deployment.yaml -f k8s/03-service.yaml
kubectl -n ss302 get deploy,rs,pods,svc -o wide
kubectl -n ss302 describe pod <nama-pod>      # lihat Events bila ada error
kubectl -n ss302 logs deploy/aq-service
```

## 3. Akses layanan

```bash
kubectl -n ss302 port-forward svc/aq-service 8080:80     # biarkan berjalan
# terminal lain:
curl localhost:8080/health
```

## 4. Self-healing dan scaling manual

```bash
kubectl -n ss302 delete pod <nama-pod>
kubectl -n ss302 get pods -w                  # Ctrl+C setelah pod pengganti Ready
kubectl -n ss302 scale deploy/aq-service --replicas=3
```

## 5. Uji state: dua replika, data di memori

Kirim satu bacaan lewat Service, lalu baca daftar bacaan delapan kali:

```bash
kubectl -n ss302 exec deploy/aq-service -- python -c "import urllib.request as u,json;r=u.Request('http://aq-service/readings',data=json.dumps({'sensor_id':'aq-001','pm25':80}).encode(),headers={'Content-Type':'application/json'});u.urlopen(r);print([len(json.load(u.urlopen('http://aq-service/readings'))) for _ in range(8)])"
```

Catat hasilnya dan jelaskan mengapa angkanya tidak selalu sama.

## 6. Rolling update yang gagal, lalu rollback

```bash
kubectl -n ss302 set image deploy/aq-service aq-service=aq-service:9.9.9   # tag yang tidak ada
kubectl -n ss302 rollout status deploy/aq-service --timeout=60s
kubectl -n ss302 get pods                     # pod lama tetap Running? mengapa?
kubectl -n ss302 rollout history deploy/aq-service
kubectl -n ss302 rollout undo deploy/aq-service
```

## 7. Autoscaling (HPA)

k3s sudah menyertakan metrics-server.

```bash
kubectl -n ss302 scale deploy/aq-service --replicas=2
kubectl apply -f k8s/04-hpa.yaml
kubectl -n ss302 top pods
# beban selama 2 menit:
kubectl -n ss302 exec deploy/aq-service -- python -c "import urllib.request as u,time;e=time.time()+120;[u.urlopen('http://aq-service/readings').read() for _ in iter(lambda: time.time()<e, False)]"
# terminal lain:
kubectl -n ss302 get hpa -w
```

Generator beban berjalan di salah satu pod layanan, jadi pod itu ikut sibuk. Ini cukup untuk demo, tetapi bukan uji beban yang sebenarnya.

## 8. Bersihkan

```bash
kubectl delete namespace ss302
```
