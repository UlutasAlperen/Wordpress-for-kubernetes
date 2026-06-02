# WordPress on Minikube

Minikube üzerinde WordPress çalıştırmak için yazdığım Kubernetes manifestleri. Amaç sadece "çalışsın" değil, production benzeri kararlarla mimariyi kurmak. Her kararı bir sebebiyle aldım, aşağıda açıklıyorum.

## Mimari

- **MySQL StatefulSet + Headless Service** — Veritabanı stateful bir uygulama, Deployment değil StatefulSet kullanmamın sebebi bu. Her pod yeniden başladığında aynı identity ve volume ile gelmeli. Headless Service ise DNS üzerinden doğrudan pod adreslemesi yapıyor, bu da replica eklemek istediğimde hayati önem taşıyor.
- **WordPress (php-fpm) + Nginx sidecar** — Aynı pod içinde iki container çalışıyor. Nginx static dosyaları serve ediyor, PHP isteklerini FastCGI üzerinden localhost:9000'den WordPress container'ına yönlendiriyor. Ayrı pod'lar yerine sidecar tercih etmemin sebebi: ağ latency'si sıfır, lifecycle aynı, volume paylaşımı kolay.
- **Gateway API (Envoy Gateway)** — Klasik Ingress yerine Gateway API kullandım. Daha expressive, role-based (cluster admin gateway oluşturuyor, app dev sadece route yazıyor) ve ilerisi için daha sağlıklı bir yönetim modeli sunuyor.
- **PersistentVolumeClaim** — WordPress dosyaları pod restart'ta silinmemeli. 5Gi PVC ile `/var/www/html` path'ini kalıcı hale getirdim.
- **ConfigMap / Secret ayrımı** — Şifreler Secret'ta, konfigürasyon değerleri ConfigMap'de. DB_PASSWORD base64 encoded halde tutuluyor. Bu ayrım güvenlik açısından önemsenecek bir detay.

## Tasarım Kararları

### Wordpress-Apache Image  yerine  Nginx Sidecar (caddy icin olan versiyonda gelecek)

WordPress'in resmi `wordpress:latest` image'ı Apache ile geliyor ve tek bir container'da hem PHP hem web server çalıştırıyor. Ben bunun yerine `wordpress:php8.5-fpm` image'ını Nginx sidecar ile kullandım.

**Neden?**

- **Performans** — Nginx static dosya serve etme konusunda Apache'den belirgin şekilde hızlı. PHP olmayan istekler (CSS, JS, görseller) doğrudan Nginx'ten karşılanıyor, PHP process'lerini meşgul etmiyor.
- **Esnek konfigürasyon** — FastCGI caching, rate limiting, custom rewrite kuralları gelişmiş ayarlar Nginx'te çok daha expressive yazılabiliyor. Apache `.htaccess` dosyaları her istekte yeniden okunuyor, Nginx ise bir kere compile ediyor.
- **Kaynak izolasyonu** — PHP-FPM ve Nginx ayrı process'ler olduğu için bellek ve CPU sınırlarını container bazında ayrı ayrı set edebiliyorum. `resources` alanında `wordpress` ve `nginx` container'larına farklı limitler verebiliyorum.
- **Sidecar avantajı** — Aynı pod içinde oldukları için ağ latency'si sıfır (localhost:9000), lifecycle aynı, volume paylaşımı doğal. Ayrı pod'larda olsaydı Service üzerinden git-gel maliyeti olurdu.

### MySQL Deployment yerine StatefulSet

İlk başta MySQL için de Deployment kullanmayı düşünüyordum, ancak araştırınca StatefulSet'in stateful uygulamalar için kritik olduğunu anladım.

**Neden?**

- **Stable network identity** — StatefulSet pod'ları sıralı isimlendirme alıyor (`mysql-database-0`, `mysql-database-1`). Headless Service ile DNS üzerinden `mysql-database-0.mysql-database` şeklinde doğrudan adreslenebiliyorlar. Deployment'ta pod isimleri her restart'ta değişiyor.
- **Persistent volume guarantee** — `volumeClaimTemplates` ile her replica için ayrı bir PVC otomatik oluşturuluyor. Pod silindiğinde bile volume korunuyor. Deployment'ta pod restart'ta farklı bir node'a schedule edilirse veri kaybı riski var.
- **Ordered scaling** — Replica sayısını artırırken sıralı başlatma ve kapatma garanti ediliyor. MySQL replication kurarken bu kritik — önce primary, sonra replica başlamalı.
- **İleride replica ekleme** — Şu an `replicas: 1` ama StatefulSet kullanarak ileride replica eklemek çok daha güvenli. Deployment ile replica artırınca her pod aynı volume'ı paylaşmak zorunda kalırdı.

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
kubectl apply -f nginx-configmap.yaml
kubectl apply -f wordpress-pvc.yaml
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml
kubectl apply -f gateway-class.yaml
kubectl apply -f gateway.yaml
kubectl apply -f wordpress-httproute.yaml
```
yada

```bash
kubectl apply -f .
```

### 4. Pod'ların Hazır Olmasını Bekle

```bash
kubectl get pods -w
```

Tüm pod'lar `Running` ve `READY 1/1` (veya MySQL için `1/1`) olana kadar bekle.

### 5. Minikube Tunnel'ı Başlat

LoadBalancer Service'lerine external IP atanması için ayrı bir terminalde çalıştır test edebilmek icin:

```bash
minikube tunnel --bind-address="127.0.0.1" -c
```
/etc/hosts duzenle domainden gelecek olan istekleri 127.0.0.1 yonlendirsin
        
```
127.0.0.1 deneme.com
```

### 6. Tarayıcıdan Eriş

```bash
kubectl get httproute
```

HTTPRouteStatus bölümündeki adresi alıp tarayıcıda aç. Alternatif olarak port-forward kullanabilirsin:

```bash
kubectl port-forward svc/wordpress-service 8080:80
```

Tarayıcıda test etmek icin `http://localhost:8080` adresine git.

## Temizlik

Kaynakları temizlemek için manifestleri ters sırada sil:

```bash
kubectl delete -f wordpress-httproute.yaml
kubectl delete -f gateway.yaml
kubectl delete -f gateway-class.yaml
kubectl delete -f wordpress-service.yaml
kubectl delete -f wordpress-deployment.yaml
kubectl delete -f wordpress-pvc.yaml
kubectl delete -f nginx-configmap.yaml
kubectl delete -f wordpress-deployment-configmap.yaml
kubectl delete -f mysql-statefulset.yaml
kubectl delete -f mysql-headless-service.yaml
kubectl delete -f wordpress-secrets.yaml
kubectl delete -f mysql-configmap.yaml
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
## Minikube harici multi-node cluster icin yapilacaklar veya yapilabilecekler

1. **StorageClass'i RWX destekli hale getirilmesi zorunlu yoksa calismaz** (en kritik — olmadan multi-node calismaz)
2. **WordPress media icin object storage ekle** (multi-pod tutarlilik)
3. **Pod anti-affinity ekle** (HA)
4. **MySQL replication** (SPOF giderme)
5. **HPA + PDB** (olcekleme ve guvenlik)
