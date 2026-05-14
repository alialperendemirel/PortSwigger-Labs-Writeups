# Cross-Site Request Forgery (CSRF) - Teknik Özet

## 🔍 CSRF Nedir?
**CSRF (Cross-Site Request Forgery)**, bir saldırganın, kurbanın halihazırda oturum açmış (authenticated) olduğu güvenilir bir web sitesinde, onun rızası ve haberi olmadan istem dışı işlemler yapmasını sağlayan bir web güvenlik zafiyetidir.

Temel olarak saldırgan, kurbanın tarayıcısını bir **kukla** gibi kullanarak hedefe sahte istekler gönderir. XSS'ten farklı olarak veri okuyamaz (Blind Attack), ancak kullanıcının yetkilerini kullanarak durum değiştiren (state-changing) eylemler gerçekleştirir.

## ⚙️ Nasıl Çalışır? (Mantık)
Saldırı, tarayıcıların belirli bir domaine yapılan isteklere o domainin çerezlerini (cookies) otomatik olarak eklemesi prensibine dayanır.

1. **Saldırgan**, hedef uygulamanın istek yapısını (örn: şifre değiştirme veya para transferi) analiz eder ve zararlı bir link veya gizli form (payload) hazırlar.
2. **Kurban**, hedef sitede oturumu açıkken (Session Cookie tarayıcıdayken), saldırganın hazırladığı bu tuzağa (örneğin e-postadaki sahte bir kedi resmine) tıklar.
3. **Tarayıcı**, kurbanın oturum çerezlerini de pakete ekleyerek isteği arka planda hedef sunucuya gönderir.
4. **Sunucu**, çerezleri gördüğü için isteğin yetkili kullanıcıdan geldiğini zannederek işlemi onaylar.

## 🚀 Kritik Saldırı Vektörleri
* **Traditional (Geleneksel) CSRF:** Görünmez (`hidden`) input'lara sahip formların JavaScript (`document.forms[0].submit()`) ile otomatik olarak postalanması (Auto-submit).
* **AJAX / XMLHttpRequest:** Sayfa yenilenmesine gerek kalmadan arka planda sessizce yapılan API çağrıları. Sunucudaki hatalı `Access-Control-Allow-Origin: *` (CORS) konfigürasyonlarıyla birleştiğinde tespiti çok zordur.
* **Hidden Link / Image Exploitation:** `<img src="http://banka.com/transfer?amount=1000" width="0" height="0">` gibi etiketlerle, tarayıcının resmi yüklemeye çalışırken arka planda GET isteği atmasını sağlama (Zero-click attack).
* **Token Forgery (Token Sahteciliği):** Geliştiricilerin Base64 vb. zayıf algoritmalarla (örn: kullanıcı ID'si veya hesap numarası tabanlı) ürettiği tahmin edilebilir (predictable) Anti-CSRF token'larını tersine mühendislikle kırıp bypass etme.
* **SameSite Cookie Chaining (Bypass):** `SameSite=Lax` olan çerezlerin normalde POST isteklerinde gönderilmemesi kuralını aşmak için tarayıcı özelliklerini sömürme. Örneğin, Chrome'un "son 2 dakikada güncellenen çerezleri POST isteklerinde de gönderme" istisnasını kullanmak için; kurbana önce `logout.php`'yi tetikletip çerezi tazeletmek, hemen 1 saniye sonra `POST` isteği fırlatarak kalkanı delmek.

## 🛡️ Savunma Yöntemleri (Mitigation)
1. **Anti-CSRF Tokens:** Her form ve durum değiştiren istek için sunucu tarafında benzersiz (unique), karmaşık (unpredictable) ve oturuma özel token'lar üretilip doğrulanmalıdır.
2. **SameSite Cookie Özelliği:** Tarayıcının çerezleri yönetebilmesi için çerezlere `SameSite=Strict` (sadece aynı siteden gelen isteklerde geçerli) veya `SameSite=Lax` özelliği eklenmelidir.
3. **Double-Submit Cookie Patern'i:** Token'ın hem bir çerez (cookie) hem de form parametresi/header olarak aynı anda gönderilmesi ve sunucuda bu iki değerin eşleştiğinin teyit edilmesi.
4. **Referer ve Origin Kontrolü:** HTTP istek başlıklarındaki `Referer` ve `Origin` değerlerinin sunucu tarafında kontrol edilerek sadece güvenilir kaynaklardan gelen isteklerin işleme alınması.
5. **Kritik İşlemlerde Ek Doğrulama (Defense in Depth):** Şifre değiştirme veya para transferi gibi yüksek riskli işlemlerde kullanıcıdan eski şifreyi tekrar istemek, SMS Onayı (MFA) veya **CAPTCHA** kullanmak CSRF'i kesin olarak durdurur.
