📝 Session Management (Oturum Yönetimi) - Pentest Cheat Sheet

Bu doküman, web uygulamalarında oturum yönetim mekanizmalarını, yaşam döngüsünü ve sık karşılaşılan yapılandırma hatalarının istismarını (exploitation) özetlemektedir.
1. Temel Kavramlar: Kimlik ve Yetki

    Authentication (Kimlik Doğrulama): Sisteme giriş yapan kişinin kimliğini kanıtlama sürecidir. (Örn: "Ben User1'im ve şifrem bu.")

    Authorization (Yetkilendirme): Kimliği doğrulanmış kişinin sistemde hangi eylemleri yapmaya yetkili olduğunu belirleme sürecidir. (Örn: "User1 bu dosyayı silebilir mi?")

2. Oturum Taşıma Mekanizmaları

    Cookies (Çerezler): Tarayıcı tarafından otomatik olarak yönetilir. Durum bilgisi tutan (stateful) mimarilerde yaygındır. XSS saldırılarından korunmak için HttpOnly ve Secure bayrakları (flags) mutlaka aktif edilmelidir.

    Tokens (Jetonlar - Örn: JWT): İstemci tarafında (Local/Session Storage) saklanır. Modern API'ler ve mobil uygulamalar için uygundur. HTTP isteklerinde Authorization: Bearer başlığı altında sunucuya iletilir.

3. Oturum Yaşam Döngüsü (Session Lifecycle)

Güvenli bir oturum yönetimi şu 4 aşamayı eksiksiz uygulamalıdır:

    Creation (Oluşturma): Oturum kimlikleri (Session ID) karmaşık, tahmin edilemez (randomized) ve yeterli uzunlukta olmalıdır.

    Tracking (Takip): İstekler arası durum bilgisini korumak için token veya çerezler her HTTP isteğiyle güvenli bir şekilde iletilmelidir.

    Expiry (Zaman Aşımı): Sistemde belirli bir süre aktif olunmadığında oturum zaman aşımına uğramalıdır (Excessive Session Lifetimes zafiyetini önlemek için).

    Termination (Sonlandırma): Çıkış (Logout) işlemi yapıldığında, oturum sadece tarayıcıdan silinmemeli, sunucu tarafında da (Server-side) kesin olarak iptal edilmelidir.

4. Sık Karşılaşılan Zafiyetler ve İstismar (Exploitation) Yöntemleri

    İstemci Tarafına Güvenmek (Client-Side Trust & Authorization Bypass):

        Açıklama: Yetki kontrollerinin arka planda (API) yapılmayıp, tarayıcıdaki verilere göre yapılmasıdır.

        Saldırı Vektörü: Tarayıcının Geliştirici Araçları (F12) -> Application -> Local Storage üzerinden kullanıcı verilerine müdahale edilir. Örneğin; role: "student" değeri role: "admin" yapılarak kısıtlı arayüzlere (menüler, butonlar) erişim sağlanır.

    Kimlik Parametrelerinin Manipülasyonu (IDOR):

        Açıklama: Veritabanı sorgularının yeterince izole edilmemesidir.

        Saldırı Vektörü: Local Storage veya çerez içindeki id parametresi değiştirilir (Örn: id: 1 genellikle ilk yönetici hesaptır). Bu sayede sunucu başka bir kullanıcının bilgilerini getirmeye zorlanır.

    İstemci Tarafı Veri Sızıntısı (Client-Side Data Leakage):

        Açıklama: Sunucunun, istemcinin ihtiyacı olandan çok daha fazla veriyi (gizli veriler dahil) ön yüze göndermesidir.

        Saldırı Vektörü: Tarayıcının Network (Ağ) sekmesindeki API istekleri (Fetch/XHR) dinlenir. Arayüzde (ekranda) görünmese bile, sunucudan dönen JSON yanıtlarının (Response) içinde gizli bilgilere (örn: test cevap anahtarları, diğer kullanıcıların mail adresleri) ulaşılabilir.
