# Server-Side Request Forgery (SSRF) - Teknik Özet

## 🔍 SSRF Nedir?
**SSRF (Server-Side Request Forgery)**, saldırganın bir web sunucusunu, sunucunun erişebildiği (ancak saldırganın doğrudan erişemediği) yerel veya uzak kaynaklara istek yapmaya zorladığı bir güvenlik zafiyetidir.

Temel olarak saldırgan, web sunucusunu bir **proxy/vekil** olarak kullanarak güvenlik duvarlarını (Firewall) ve iç ağ korumalarını bypass eder.

## ⚙️ Nasıl Çalışır? (Mantık)
Bir uygulama, kullanıcının verdiği bir URL'den veri çekiyorsa (örneğin: profil resmi yükleme, URL önizleme, PDF oluşturma) ve bu URL'yi doğrulamıyorsa SSRF oluşur.

1.  **Saldırgan**, sunucuya dışarıdan erişilemeyen bir iç adres gönderir (Örn: `http://localhost/admin` veya `http://192.168.1.10`).
2.  **Sunucu**, bu isteği kendi "güvenli" iç ağı üzerinden gerçekleştirir.
3.  **Yanıt**, sunucu tarafından saldırgana geri döndürülür.

## 🚀 Kritik Saldırı Vektörleri
* **Internal Port Scanning:** Dışarıya kapalı olan iç ağdaki cihazların portlarını taramak.
* **Cloud Metadata Access:** AWS, Azure veya Google Cloud sunucularında, sadece sunucu içinden erişilebilen `169.254.169.254` IP adresine istek atarak sistemin gizli API anahtarlarını (credentials) çalmak.
* **Local File Read:** `file:///etc/passwd` gibi protokollerle sunucunun sistem dosyalarını okumak.
* **Blind SSRF:** Sunucunun cevabı ekrana basmadığı durumlarda, saldırganın kendi sunucusuna (OOB - Out-of-band) bağlantı isteği attırarak zafiyeti kanıtlaması.

## 🛡️ Savunma Yöntemleri (Mitigation)
1.  **Allowlist (Beyaz Liste):** Uygulamanın sadece önceden tanımlanmış, güvenilir alan adlarına istek atmasına izin verin.
2.  **Girdi Doğrulama:** Kullanıcıdan gelen URL'lerin protokolünü (sadece `https`), IP adresini (iç ağ blokları yasaklanmalı) ve portunu kontrol edin.
3.  **Ağ Segmantasyonu:** Web sunucusunun iç ağdaki hassas servislere erişimini kısıtlayın.
4.  **Tehlikeli Protokolleri Devre Dışı Bırakın:** `file://`, `gopher://`, `dict://` gibi protokolleri engelleyin.


💡 Küçük Bir İpucu (Pro Tip):

SSRF bulduğunda ilk denemen gereken adres her zaman http://127.0.0.1 veya http://localhost olmalı. Eğer sunucu bulut üzerindeyse (AWS gibi), meşhur http://169.254.169.254/latest/meta-data/ adresini denemek doğrudan "Kritik" (P1) seviye bir bulgu sağlar.
