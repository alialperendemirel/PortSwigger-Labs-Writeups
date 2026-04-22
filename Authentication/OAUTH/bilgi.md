🛡️ OAuth 2.0 / 2.1 - Security & Exploitation Cheat Sheet

OAuth (Open Authorization), kullanıcıların şifrelerini üçüncü taraf uygulamalarla paylaşmadan, bu uygulamalara kısıtlı erişim (Token/Bilet) yetkisi vermesini sağlayan bir yetkilendirme protokolüdür.
🎟️ OAuth Aktörleri ve Terimler

    Resource Owner: Kullanıcı (Hesabın asıl sahibi).

    Client: Kullanıcının verilerine erişmek isteyen aracı uygulama.

    Authorization Server: Kullanıcıyı doğrulayan ve bilet (Token) üreten sunucu.

    Access Token: Korumalı kaynaklara erişmek için kullanılan "Vale Anahtarı".

    Redirect URI: İşlem bitince kullanıcının ve biletin geri gönderileceği adres.

    State: CSRF saldırılarını önleyen rastgele güvenlik fişi.

🚀 Temel İzin Türleri (Grant Types)

    Authorization Code Flow (En Güvenlisi): Tarayıcıya sadece geçici bir "Kod" döner. Arka planda sunucular bu kodu Access Token ile değiştirir.

    Implicit Flow (Tehlikeli/Kaldırıldı): Access Token doğrudan URL içinde (Fragment # ile) tarayıcıya gönderilir.

    Client Credentials: Sunucudan sunucuya iletişim için kullanılır (Kullanıcı dahil olmaz).

💥 Kritik Zafiyetler ve İstismar (Exploitation)
Zafiyet Adı	Hatalı Yapılandırma	Saldırı Vektörü (Nasıl Hacklenir?)


Open Redirect (Token Hijacking)	redirect_uri adresinin sunucu tarafından tam olarak (exact match) doğrulanmaması.	redirect_uri saldırganın kontrolündeki bir alt alan adına (subdomain) yönlendirilir. Kurban giriş yaptığında "Kod", saldırganın sunucusuna düşer.


CSRF / Account Linking	state parametresinin kullanılmaması veya kontrol edilmemesi.	Saldırgan kendi onay kodunu (code) kurbana bir link ile gönderir. Tıklayan kurbanın hesabı, saldırganın profiline zorla bağlanır (Senkronizasyon çalınması).



Implicit Flow XSS Sızıntısı	Token'ın doğrudan URL (#access_token=...) üzerinden gönderilmesi.	Hedef sitede basit bir XSS zafiyeti varsa, JavaScript kodu URL'deki token'ı okur ve saldırganın sunucusuna (örn: görünmez bir resim <img> isteğiyle) sızdırır.
🛡️ OAuth 2.1 ve Güvenlik Çözümleri (The Fix)

Sektör, OAuth 2.0'daki bu açıkları kapatmak için OAuth 2.1 standartlarını getirdi:

    ❌ Implicit Grant Yasaklandı: Token'lar asla URL'de taşınamaz.

    🛡️ PKCE Zorunlu Hale Geldi: Public istemciler (Mobil/SPA) için Authorization Code akışına PKCE (Proof Key for Code Exchange) şifrelemesi eklendi.

    ✅ State Parametresi Zorunlu: CSRF koruması standart bir zorunluluk oldu.

    🔒 Sıkı Yönlendirme Kontrolü: redirect_uri adresleri karakteri karakterine (exact string matching) eşleşmek zorundadır.
