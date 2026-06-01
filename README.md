# WordPress on Minikube

Minikube üzerinde WordPress çalıştırmak için yazdığım Kubernetes manifestleri. Amaç sadece "çalışsın" değil, production benzeri kararlarla mimariyi kurmak. Her kararı bir sebebiyle aldım, aşağıda açıklıyorum.

## Mimari

- **MySQL StatefulSet + Headless Service** — Veritabanı stateful bir uygulama, Deployment değil StatefulSet kullanmamın sebebi bu. Her pod yeniden başladığında aynı identity ve volume ile gelmeli. Headless Service ise DNS üzerinden doğrudan pod adreslemesi yapıyor, bu da replica eklemek istediğimde hayati önem taşıyor.
- **WordPress (php-fpm) + Nginx sidecar** — Aynı pod içinde iki container çalışıyor. Nginx static dosyaları serve ediyor, PHP isteklerini FastCGI üzerinden localhost:9000'den WordPress container'ına yönlendiriyor. Ayrı pod'lar yerine sidecar tercih etmemin sebebi: ağ latency'si sıfır, lifecycle aynı, volume paylaşımı kolay.
- **Gateway API (Envoy Gateway)** — Klasik Ingress yerine Gateway API kullandım. Daha expressive, role-based (cluster admin gateway oluşturuyor, app dev sadece route yazıyor) ve ilerisi için daha sağlıklı bir yönetim modeli sunuyor.
- **PersistentVolumeClaim** — WordPress dosyaları pod restart'ta silinmemeli. 5Gi PVC ile `/var/www/html` path'ini kalıcı hale getirdim.
- **ConfigMap / Secret ayrımı** — Şifreler Secret'ta, konfigürasyon değerleri ConfigMap'de. DB_PASSWORD base64 encoded halde tutuluyor. Bu ayrım güvenlik açısından önemsenecek bir detay.

## Kullanılan Teknolojiler

| Bileşen | Versiyon |
|---|---|
| Kubernetes | Minikube |
| MySQL | 8.0 |
| WordPress | php8.5-fpm |
| Nginx | latest |
| Envoy Gateway | gateway.networking.k8s.io/v1 |

## Nasıl Çalıştırılır

```bash
minikube start

# Sırayla apply et
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

## Karşılaştığım Zorluklar ve Öğrendiklerim

- **Volume permission sorunu** — WordPress container www-data (uid 33) kullanıcısıyla çalışıyor ama PVC root'a ait mount ediliyor. Bu yüzden init container ile `chown -R 33:33` yapıp yetkiyi düzeltmem gerekti. Başlangıçta pod CrashLoopBackOff'a düşüyordu, logları inceledikten sonra sorunun yetki olduğunu anladım.

- **StatefulSet vs Deployment kararı** — İlk başta MySQL için de Deployment kullanmayı düşünüyordum. Araştırınca anladım ki StatefulSet'in sağladığı stable network identity ve (ordered) scaling kavramları stateful uygulamalar için kritik. Hele replica sayısı 1'den fazla olursa StatefulSet şart.

- **Ingress yerine Gateway API** — Başlangıçta Ingress ile gidecektim ama Gateway API'nin sunduğu structural separation (cluster admin ve app geliştirici rollerinin ayrılması) daha büyük projelerde hayat kurtarıyor. Öğrenme eğrisi biraz dik ama yatırım yapmaya değer.

- **Sidecar pattern** — Nginx ve WordPress FPM'yi ayrı pod'larda çalıştırıp Service ile bağlamak kolay yol. Ama sidecar seçince aynı lifecycle içinde manej edebiliyorum, network hop yok, volume paylaşımı doğal. Debug etmek için `kubectl logs -c` ile ayrı ayrı log alabilmek de cabası.


