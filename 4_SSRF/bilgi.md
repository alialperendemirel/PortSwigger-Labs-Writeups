# Web Security 0x16 | Server-Side Request Forgery (SSRF) Nedir? - Video Notları ve Özeti

Bu doküman, Mehmet İnce'nin **"Web Security 0x16 | Server-Side Request Forgery Nedir ?"** başlıklı yayını kapsamında ele alınan teknik detaylar, anlatılan zafiyet mantıkları, çözülen laboratuvarlar ve gerçek dünya senaryoları esas alınarak hazırlanmıştır [1-59].

## 1. Server-Side Request Forgery (SSRF) Nedir?

SSRF (Sunucu Taraflı İstek Sahteciliği), modern web uygulamalarının karmaşık yapılarından ve dış dünyayla (mikroservisler, üçüncü parti API'lar vb.) kurdukları etkileşimlerden beslenen kritik bir zafiyettir [11, 12, 14].

* **Temel Mantık:** Uygulamanın backend (arka uç) sunucusunun, dışarıdan (istemciden) aldığı girdilere (input) dayanarak başka bir kaynağa (internal veya external) istek göndermesi ve bu giden isteğin saldırgan tarafından manipüle edilmesidir [14, 16, 17].
* **İstemci gibi Davranan Sunucu:** Normalde bir istemci (client) sunucuya istek gönderirken, SSRF senaryolarında uygulama sunucusu kendisi bir istemci gibi davranarak başka bir iç veya dış kaynağa talep oluşturur [14, 25].
* **CSRF ile Farkı:** CSRF (Client-Side / Cross-Site Request Forgery) ile isim benzerliği dışında hiçbir alakası yoktur [13]. CSRF istemcinin tarayıcısını istismar ederken, SSRF doğrudan sunucunun (server-side) istek gönderme yeteneğini suistimal eder [13, 14].

## 2. SSRF Türleri

SSRF zafiyetleri temel olarak iki ana başlıkta ele alınır [21]:

* **Basic (Görünür) SSRF:** Sunucunun istek gönderdiği kaynaktan dönen yanıt (response) doğrudan kullanıcıya/saldırgana gösterilir [20, 21]. Saldırgan iç ağdaki servisleri doğrudan bu yanıtlar üzerinden analiz edebilir [21, 24].
* **Blind (Kör) SSRF:** Sunucu isteği gönderir ancak dönen yanıtı istemciye göstermez [20, 21]. Bu durumda zafiyetin tespiti ve sömürülmesi için genellikle dışarıya açık bir sunucuya (örneğin Burp Collaborator) tetikleme (out-of-band) yaptırılması veya DNS/HTTP sorgularının izlenmesi gerekir [20, 35, 36].

## 3. İç Ağ Tespiti ve Cloud (Bulut) Altyapı Tehditleri

SSRF'in en tehlikeli yönlerinden biri, dış dünyadan erişilemeyen iç sistemlere (Localhost, Redis, Memcached, Elastic Search, veri tabanları) sunucu üzerinden erişim imkanı tanımasıdır [16, 17, 31].

### Cloud Metadata İstismarı
* Uygulama AWS, Google Cloud veya Azure gibi bir bulut altyapısında çalışıyorsa, SSRF aracılığıyla bulut sağlayıcılarının sunduğu dahili Metadata IP adresine (`169.254.169.254`) istek gönderilebilir [26, 27, 55].
* Bu adres üzerinden sunucuya ait IAM rolleri, AWS Access Key'leri, GCloud service account token'ları gibi kritik kimlik bilgileri sızdırılabilir [27, 28].
* Saldırganlar bu bilgileri ele geçirerek dikey veya yatay yetki yükseltme (vertical/horizontal privilege escalation) gerçekleştirebilirler [27].

## 4. PortSwigger Web Security Academy - Çözülen Laboratuvarlar

Yayında PortSwigger Web Security Academy üzerinde yer alan çeşitli SSRF senaryoları pratik olarak çözülmüştür [19]:

### Lab 1: SSRF Against the Local Server (Yerel Sunucuya Karşı SSRF)
* **Senaryo:** Ürün detay sayfasındaki stok kontrol özelliği (`checkStock`), arka planda bir stok API'sine istek atmaktadır [23, 24].
* **Çözüm:** `stockApi` parametresi manipüle edilerek yerel adrese (`http://127.0.0.1/admin`) yönlendirilmiş ve normal şartlarda dışarıya kapalı olan yönetim paneline erişilerek Carlos kullanıcısı silinmiştir [24].

### Lab 2: SSRF Against Another Backend System (Başka Bir Dahili Sisteme Karşı SSRF)
* **Senaryo:** Dahili bir IP bloğunda (`192.168.0.x`) ve belirli bir portta (`8080`) çalışan admin arayüzü mevcuttur [31, 32].
* **Çözüm:** Burp Intruder kullanılarak IP adresinin son hanesi taranmış (1-255 arası) [33, 34]. Yanıtlardaki durum kodları (not found - 404, internal server error - 500 vb.) incelenerek aktif olan IP tespit edilmiş ve bu IP üzerinden admin paneline erişilerek işlem tamamlanmıştır [34, 35].

### Lab 3: Blind SSRF with Out-of-Band Detection
* **Senaryo:** Uygulama, kullanıcının `Referer` başlığında (header) gönderdiği URL adresine istek atmakta ancak yanıtı göstermemektedir [35].
* **Çözüm:** `Referer` başlığına bir Burp Collaborator adresi girilerek sunucunun dışarıya doğru DNS ve HTTP istekleri yapması sağlanmış ve kör zafiyet doğrulanmıştır [35, 36].

### Lab 4: SSRF with Open Redirect Bypass (Open Redirect ile SSRF Engellerini Aşma)
* **Senaryo:** `stockApi` parametresi üzerinde sadece belirli URL kalıplarına izin veren bir filtreleme (white-listing) mevcuttur [38, 39].
* **Çözüm:** Uygulama içerisindeki başka bir parametrede (path tabanlı yönlendirme yapan) bulunan Open Redirect zafiyeti tespit edilmiştir [40]. Stok API'sine yasal olarak kabul edilen iç adres verildikten sonra bu adres Open Redirect zafiyetini tetikleyecek şekilde manipüle edilmiş, sunucu `302 Redirection` (Yönlendirme) kodunu takip ederek hedef yerel adrese erişmiştir [40, 41].

### Lab 5: Blind SSRF with Shellshock Exploitation
* **Senaryo:** İç ağda eski bir CGI web sunucusu çalışmaktadır ve bu sunucu üzerinde Shellshock (bash) zafiyeti mevcuttur [47].
* **Çözüm:** Hedef iç ağ taranırken, giden isteklerin `User-Agent` başlığına Shellshock exploit kodu yerleştirilmiştir [45, 47]. Sunucu tetiklendiğinde `whoami` (veya işletim sistemi bilgisi) çıktısını dışarıdaki bir DNS/Collaborator adresine alt alan adı (subdomain) olarak ekleyerek (exfiltration) sızdırması sağlanmıştır [45, 46, 47].

## 5. Gerçek Dünya Örneği: Dropbox (HelloSign) SSRF Zafiyeti

Yayında incelenen gerçek bir bug bounty raporu, zafiyetin gerçek dünyadaki etkisini açıkça göstermektedir [53, 54]:

* **Özellik:** Kullanıcıların Dropbox, Google Drive gibi platformlardan doküman import etmesini sağlayan bir yapı mevcuttur [54].
* **Aşma Yöntemi:** Uygulama doğrudan `169.254.169.254` (metadata) adresine erişimi engellemektedir [55]. Saldırgan, parametreler arasındaki `service_type` (OneDrive, Dropbox, Evernote belirteçleri) değerlerini manipüle ederek korumaları atlatmış ve redirect (yönlendirme) tekniği sayesinde metadata adresine başarıyla erişerek hassas verileri sızdırmıştır [54, 55].

## 6. SSRF Engelleme ve Güvenlik Önlemleri (Hardening)

SSRF zafiyetlerini tamamen kod bazlı doğrulamalar (Blacklist/Whitelist) ile engellemek her zaman yeterli olmayabilir [29, 37]. En güvenilir yöntemler tasarımsal sertleştirme (Hardening) adımlarıdır [29]:

* **Ağ İzolasyonu:** SSRF yapma ihtiyacı olan servisler (örneğin dosya indiren, harici API'lar ile konuşan Docker imajları) izole edilmeli, dahili ağdaki hassas sistemlerle (veritabanı, redis vb.) iletişim kurması ağ katmanında engellenmelidir [29].
* **DNS Çözümleme Kontrolleri (Anti-SSRF):** Alınan URL'lerin DNS çözümlemesi yapıldıktan sonra elde edilen IP adresinin özel/yerel (private IP) aralıkta olup olmadığı doğrulanmalıdır.
* **Yönlendirmeleri Kısıtlama:** Sunucunun HTTP `302` yönlendirmelerini körü körüne takip etmesi engellenmeli veya yönlendirme yapılan yeni adresler de sıkı bir filtrelemeden geçirilmelidir [37].
