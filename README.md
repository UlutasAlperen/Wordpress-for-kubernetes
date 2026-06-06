# WordPress on Minikube

Minikube üzerinde WordPress çalıştırmak için yazdığım Kubernetes manifestleri. Production benzeri kararlarla kurdum, her kararın bir sebebi var.

## Mimari

- **MySQL StatefulSet + Headless Service** — Veritabanı stateful bir şey, Deployment kullanmak mantıklı değil. StatefulSet ile pod'lar stable identity'ye sahip oluyor, replay'e uygun sıralı başlatma var. Headless Service de DNS üzerinden doğrudan pod adreslemesi yapıyor.
- **WordPress (php-fpm) + Nginx sidecar** — Aynı pod içinde iki container. Nginx static dosyaları serve ediyor, PHP isteklerini FastCGI üzerinden unix socket'ten WordPress container'ına yolluyor. Ayrı pod arasında network hop olmuyor, volume paylaşımı da kolay.
- **PHP-FPM Unix Socket** — `wordpress-php-conf-configmap` ile PHP-FPM'i TCP yerine unix socket'e (`/var/run/php-fpm/wordpress.sock`) dinlemeye ayarladım. Socket dosyasını `emptyDir` volume'ü ile iki container arasında paylaştırıyorum. TCP localhost:9000 yerine unix socket same-pod iletişiminde daha hızlı.
- **Gateway API (Envoy Gateway)** — İngress yerine Gateway API kullandım. Daha expressive, role-based bir model sunuyor. Cluster admin Gateway oluşturuyor, uygulama geliştiricisi sadece route yazıyor.
- **PersistentVolumeClaim** — WordPress dosyaları pod restart'ta kaybolmamalı. 5Gi PVC ile `/var/www/html` path'ini kalıcı hale getirdim. MySQL için de StatefulSet'in `volumeClaimTemplates` özelliğini kullandım, pod silinse bile data duruyor.
- **ConfigMap / Secret ayrımı** — Şifreler Secret'ta, konfigürasyon değerleri ConfigMap'de. DB_PASSWORD base64 encoded halde tutuluyor tabi base64 encryption değil ama en azından manifest'te plain text durmuyor.
- **Init Container** — WordPress PV'sine `www-data` (uid 33) kullanıcısı yazabilsin diye `securityContext: supplementalGroups: [33]` ve init container ile `chown -R 33:33 /var/www/html` yapıyorum. Parcel mount sonrası root'a ait oluyor dizin, WordPress bunu yazamıyor yoksa.
- **RollingUpdate** — WordPress Deployment'ta `maxSurge: 25%` ve `maxUnavailable: 25%` set ettim. Yeni bir sürüm çıkınca hepsini birden düşürmüyor.

## Tasarım Kararları

### WordPress-Apache Image yerine Nginx Sidecar (Caddy versiyonu da gelecek)

WordPress'in resmi image'ı Apache ile geliyor, tek container'da hem PHP hem web server çalışıyor. Ben `wordpress:php8.5-fpm` image'ını Nginx sidecar ile kullandım.

**Neden?**

- **Performans** — Nginx static dosya serve etmede Apache'den hızlı. PHP olmayan istekler doğrudan Nginx'ten karşılanıyor, PHP process'lerini meşgul etmiyor.
- **Esnek konfigürasyon** — FastCGI caching, rate limiting, rewrite kuralları Nginx'te daha rahat yazılıyor. Apache `.htaccess` her istekte yeniden okunuyor, Nginx bir kere compile ediyor.
- **Kaynak izolasyonu** — PHP-FPM ve Nginx ayrı process'ler, bellek ve CPU limitlerini container bazında ayrı set edebiliyorum.
- **Sidecar avantajı** — Aynı pod içinde oldukları için unix socket ile haberleşiyorlar (ağ latency'si sıfır), lifecycle aynı, volume paylaşımı doğal.

### MySQL Deployment yerine StatefulSet

İlk başta MySQL için de Deployment kullanmayı düşünüyordum ama StatefulSet'in stateful uygulamalar için kritik olduğunu gördüm.

**Neden?**

- **Stable network identity** — StatefulSet pod'ları sıralı isimlendirme alıyor (`mysql-database-0`, `mysql-database-1`). Headless Service ile `'mysql-database-0.mysql-service` şeklinde doğrudan adreslenebiliyorlar. Deployment'ta pod isimleri her restart'ta değişiyor.
- **Persistent volume guarantee** — `volumeClaimTemplates` ile her replica için ayrı PVC otomatik oluşturuluyor. Pod silindiğinde bile volume korunuyor. Deployment'ta farklı node'a schedule edilirse veri kaybı riski var.
- **Ordered scaling** — Replica sayısını artırırken sıralı başlatma ve kapatma garanti ediliyor. MySQL replication kurarken önce primary, sonra replica başlamalı.
- **İleride replica ekleme** — Şu an `replicas: 1` ama StatefulSet ile replica eklemek güvenli. Deployment ile replica artırınca her pod aynı volume'ı paylaşmak zorunda kalırdı.

## Gereksinimler

| Yazılım | Minimum Versiyon | Kontrol |
|---|---|---|
| Minikube | v1.30+ | `minikube version` |
| kubectl | v1.28+ | `kubectl version` |
| Envoy Gateway | v1.3.0 | Kurulum adımında kurulacak |

**Donanım önerisi:** Minikube için en az 2 CPU, 4GB RAM.

## Kullanılan Teknolojiler

| Bileşen | Versiyon |
|---|---|
| Kubernetes | Minikube |
| MySQL | 8.0 |
| WordPress | php8.5-fpm |
| Nginx | latest |
| Envoy Gateway | gateway.networking.k8s.io/v1 |

## Nasıl Çalıştırılır

### 1. Minikube'ı Başlat

```bash
minikube start
```

### 2. Envoy Gateway'i Kur

Gateway API için Envoy Gateway controller'ını plain manifest ile kuruyoruz:

```bash
kubectl apply -f https://github.com/envoyproxy/gateway/releases/download/v1.3.0/install.yaml
```

Envoy Gateway'in hazır olmasını bekle:

```bash
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

### 3. Uygulama Manifestlerini Sırayla Apply Et

```bash
kubectl apply -f mysql-configmap.yaml
kubectl apply -f wordpress-secrets.yaml
kubectl apply -f mysql-headless-service.yaml
kubectl apply -f mysql-statefulset.yaml
kubectl apply -f wordpress-deployment-configmap.yaml
kubectl apply -f wordpress-php-conf-configmap.yaml
kubectl apply -f nginx-configmap.yaml
kubectl apply -f wordpress-pvc.yaml
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml
kubectl apply -f gateway-class.yaml
kubectl apply -f gateway.yaml
kubectl apply -f wordpress-httproute.yaml
```

ya da

```bash
kubectl apply -f .
```

### 4. Pod'ların Hazır Olmasını Bekle

```bash
kubectl get pods -w
```

Tüm pod'lar `Running` ve `READY 1/1` olana kadar bekle.

### 5. Minikube Tunnel'ı Başlat

LoadBalancer Service'lerine external IP atanması için ayrı bir terminalde çalıştır:

```bash
minikube tunnel --bind-address="127.0.0.1" -c
```

`/etc/hosts` dosyasına ekle:

```
127.0.0.1 deneme.com
```

### 6. Tarayıcıdan Eriş

```bash
kubectl get httproute
```

HTTPRouteStatus'teki adresi tarayıcıda aç. Alternatif olarak port-forward:

```bash
kubectl port-forward svc/wordpress-service 8080:80
```

Tarayıcıda `http://localhost:8080` adresine git.

## Temizlik

Kaynakları temizlemek için:

```bash
kubectl delete -f .
```

Envoy Gateway'i kaldırmak için:

```bash
kubectl delete -f https://github.com/envoyproxy/gateway/releases/download/v1.3.0/install.yaml
```

Minikube'ı durdurmak veya silmek için:

```bash
minikube stop
# veya tamamen silmek için
minikube delete
```

## Minikube Harici Multi-Node Cluster İçin Yapılması Gerekenler

1. **StorageClass RWX destekli olmalı** — yoksa multi-node çalışmaz
2. **WordPress media için object storage ekle** — multi-pod tutarlılık için
3. **Pod anti-affinity ekle** — HA için
4. **MySQL replication** — SPOF giderme
5. **HPA + PDB** — ölçekleme ve güvenlik için