PortSwigger Access Control & IDOR Labs - Apprentice Level

Bu depo, PortSwigger Web Security Academy'deki Access Control (Erişim Kontrolü) zafiyetlerinin çözüm adımlarını ve arka planda yatan yazılım/mantık hatalarını içermektedir. Temel kural: İstemciden (client) gelen hiçbir veriye güvenme, yetki kontrolünü her zaman sunucuda (backend) yap.
1. Unprotected admin functionality

    Zafiyetin Mantığı: Geliştirici, admin panelini sadece arayüzde (frontend) normal kullanıcılardan gizlemiş. Ancak backend tarafında, ilgili URL dizinine (/admin) istek atıldığında isteği yapanın yetkisini (session/role) kontrol eden bir middleware veya doğrulama mekanizması koymayı unutmuş.

    Nasıl Sömürülür: Tarayıcının adres çubuğuna doğrudan /admin yazılarak admin paneline erişilir ve ilgili işlemler (kullanıcı silme vb.) gerçekleştirilir.

    Çıkarılan Ders: Butonları gizlemek güvenlik sağlamaz. Her uç noktada (endpoint) yetki doğrulaması yapılmalıdır.

2. Unprotected admin functionality with unpredictable URL

    Zafiyetin Mantığı: Geliştirici bu kez yetki kontrolü koymamakla kalmamış, "kimse bulamaz" diyerek admin panelinin URL'ini karmaşıklaştırmış (Security by Obscurity - Gizleyerek Güvenlik). Ancak frontend kodlarına (JavaScript dosyaları veya HTML yorum satırları) bu URL'i statik olarak sızdırmış.

    Nasıl Sömürülür: Sayfa kaynağı (Ctrl+U) veya Geliştirici Araçları (F12) incelenir. JavaScript kodlarının içinde gizlenmiş olan karmaşık admin URL'i (örn: /admin-xg54df) bulunur ve doğrudan bu adrese gidilir.

3. User role controlled by request parameter

    Zafiyetin Mantığı: Uygulama, kullanıcının admin olup olmadığını sunucu tarafında (örneğin veritabanında veya güvenli bir session içinde) tutmak yerine, istemciye gönderdiği bir çerezde (Cookie: Admin=false) tutuyor.

    Nasıl Sömürülür: Burp Suite ile araya girilir. Giden HTTP isteklerindeki Cookie başlığında yer alan değer Admin=true olarak değiştirilerek sunucuya gönderilir. Sunucu bu değere güvendiği için kullanıcıyı admin olarak tanır.

4. User role can be modified in user profile

    Zafiyetin Mantığı: "Mass Assignment" (Toplu Atama) zafiyeti. Kullanıcı profilini güncellerken gönderilen JSON veya form verileri, backend'de filtrelenmeden doğrudan veritabanı modeline aktarılıyor. Yazılımcı, kullanıcının sadece "email" alanını göndereceğini varsaymış.

    Nasıl Sömürülür: E-posta güncelleme isteği Burp Suite'te yakalanır. JSON gövdesine yazılımcının beklemediği yetki parametresi eklenir: {"email":"test@test.com", "roleid":2}. Sunucu bunu doğrudan işlediği için yetki yükseltilmiş olur.

5. User ID controlled by request parameter

    Zafiyetin Mantığı: Klasik IDOR (Insecure Direct Object Reference). Veritabanından bir veri çekilirken URL'de giden ?id=wiener parametresi baz alınıyor. Ancak backend, "Bu veriyi isteyen oturum (session), bu ID'nin gerçek sahibi mi?" diye kontrol etmiyor.

    Nasıl Sömürülür: Hedef URL'deki id parametresi, hedef kullanıcının ID'si ile (örneğin ?id=carlos) değiştirilerek istek (GET) gönderilir ve başkasının verilerine doğrudan erişilir.

6. User ID controlled by request parameter, with unpredictable user IDs

    Zafiyetin Mantığı: Sistemde tahmin edilemeyen (UUID/GUID gibi) karmaşık kullanıcı kimlikleri (ID) kullanılmış. Ancak bu ID'ler, uygulamanın başka bir yerinde (örneğin blog yazılarındaki yazar profil linklerinde) herkese açık şekilde ifşa ediliyor.

    Nasıl Sömürülür: Hedef kullanıcının paylaştığı bir blog yazısına tıklanır ve URL'den o kullanıcının karmaşık ID'si kopyalanır. Ardından kendi profil sayfamıza giderken URL'deki ID kısmı, hedef kullanıcının kopyaladığımız karmaşık ID'si ile değiştirilir.

7. User ID controlled by request parameter with data leakage in redirect

    Zafiyetin Mantığı: Uygulama IDOR zafiyetine sahip ancak sayfayı direkt göstermek yerine, bir hata yakalayıp kullanıcıyı login sayfasına yönlendiriyor (302 Redirect). Buradaki ölümcül hata: Backend yönlendirme komutunu verirken, aynı HTTP yanıtının gövdesine (Body) kullanıcının hassas verilerini de ekleyip gönderiyor.

    Nasıl Sömürülür: Tarayıcı, 302 yönlendirmesini görünce hemen başka sayfaya geçer ve veriyi göremeyiz. Ancak istek Burp Suite üzerinden yapılırsa (Repeater), tarayıcının atladığı o ilk yanıt (Response) yakalanır ve gövdesindeki (body) sızdırılan veriler (API Key vb.) okunur.

8. User ID controlled by request parameter with password disclosure

    Zafiyetin Mantığı: IDOR kullanılarak başka bir kullanıcının profil sayfasına erişiliyor. Buradaki asıl büyük güvenlik zafiyeti, geliştiricinin şifreleri sayfada maskelemeden gizli bir HTML etiketi (<input type="hidden">) içinde açık metin (plaintext) olarak tutmasıdır.

    Nasıl Sömürülür: Kendi hesabımızdan hedef kullanıcının sayfasına (?id=carlos) IDOR ile geçiş yapılır. Sayfa yüklendikten sonra kaynak kodları incelenir veya sayfadaki şifre alanının "type" değeri Inspect (Öğe'yi İncele) ile text yapılarak hedefin şifresi ele geçirilir.

9. Insecure direct object references

    Zafiyetin Mantığı: Web uygulaması; faturalar, sohbet kayıtları veya resimler gibi statik dosyaları sunarken tahmin edilebilir numaralandırmalar (/download?transcript=1.txt) kullanıyor. Erişim kontrolü mekanizması bu statik dosyaların indirilme rotasına entegre edilmemiş.

    Nasıl Sömürülür: Kendi indirdiğimiz bir dosyanın URL'sindeki numara, ardışık sayı denemeleriyle (Burp Intruder kullanılarak veya manuel olarak 2.txt, 3.txt şeklinde) değiştirilerek diğer kullanıcılara ait özel statik dosyalar indirilir.
