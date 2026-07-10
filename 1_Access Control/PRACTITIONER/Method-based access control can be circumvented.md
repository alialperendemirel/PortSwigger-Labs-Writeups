Access Control Bypass via HTTP Method Tampering (Bup Suite Lab Write-up)
📌 Zafiyet Özeti (Vulnerability Summary)

Bu laboratuvarda, "Erişim Kontrolü İhlali" (Broken Access Control) ve "HTTP Metot İstismarı" (HTTP Method Tampering) yöntemleri kullanılarak düşük yetkili bir kullanıcının (Vertical Privilege Escalation), admin paneline erişimi olmadan kendi yetkilerini en üst seviyeye çıkarması (Admin yapması) simüle edilmiştir.

Zafiyetin kök nedeni; arka uçtaki (backend) geliştiricinin, kritik fonksiyonları koruyan güvenlik filtrelerini yalnızca belirli HTTP metotları (POST) için aktif etmesi, diğer metotları (GET) gözden kaçırmasıdır.
🛠️ Teknik Analiz ve Mantıksal Akış (Logical Flow)
Aşama 1: Keşif ve İstek Şablonunun Çıkarılması (Reconnaissance)

    Yapılan İşlem: Admin yetkileriyle giriş yapılarak admin panelinden carlos isimli kullanıcının yetkileri yükseltildi ve bu istek Burp Repeater'a gönderildi.

    MANTIĞI: Gerçek bir siber güvenlik senaryosunda saldırganın elinde admin şifresi bulunmaz. Ancak bu laboratuvarda admin hesabının kullanılmasının tek bir amacı vardır: Sistemin bir kullanıcıyı admin yaparken arka planda sunucuya nasıl bir "emir" (HTTP isteği) gönderdiğini haritalandırmak. Sunucunun hangi uç noktayı (Endpoint: /admin-panel/promote), hangi parametreleri (username=carlos) ve hangi HTTP metodunu (POST) beklediği tespit edilmiştir.

Aşama 2: Yetkilendirme Kontrolünün Test Edilmesi (Authentication Testing)

    Yapılan İşlem: Gizli sekmeden yetkisiz/düşük yetkili bir kullanıcı ile oturum açıldı. Bu kullanıcının oturum anahtarı (Session Cookie), Burp Repeater'daki admin isteğinin içine yerleştirildi. İstek gönderildiğinde sunucudan Unauthorized (Yetkisiz) yanıtı alındı.

    MANTIĞI: Sistem ilk aşamada doğru çalışıyor. Sunucu gelen POST isteğindeki Cookie'ye bakıyor: "Sen admin değilsin, o halde bu POST isteğini işleme almıyorum" diyerek isteği reddediyor. Yani doğrudan kimlik taklidi işe yaramıyor.

Aşama 3: Güvenlik Duvarını Şaşırtma - Metot Değişikliği (HTTP Method Tampering)

    Yapılan İşlem: İstek metodu önce rastgele bir değerle (POSTX), ardından geçerli bir GET metoduyla değiştirildi (Change request method).

    MANTIĞI (Kritik Nokta): Yazılımcılar genellikle güvenlik filtrelerini yazarken şu mantık hatasına düşerler:

        "Admin panelindeki buton sunucuya POST isteği atıyor. O halde ben kod tarafında gelen istek POST ise yetki kontrolü yapayım."

    Biz isteği GET metoduna çevirdiğimizde, sunucu tarafındaki zafiyetli kodun (WAF veya iç mekanizma) yetki kontrol filtresini bypass etmiş (atlatmış) oluruz. Sunucu gelen isteğin GET olduğunu görünce "Bunda yetki aramama gerek yok" diyerek koruma kalkanını indirir.

Aşama 4: Parametre Manipülasyonu ve Gol (Exploitation)

    Yapılan İşlem: GET isteği içerisindeki username=carlos parametresi, kendi düşük yetkili kullanıcı adımızla değiştirildi ve istek sunucuya tekrar fırlatıldı.

    MANTIĞI: Koruma kalkanı (Authentication Check) aşındığı için sunucu artık isteği gönderen kişinin kim olduğuna (admin olup olmadığına) bakmıyor. Sadece gelen emirde ne yazdığına bakıyor: "username parametresindeki kullanıcıyı admin yap." Sunucu bu isteği körü körüne işler ve düşük yetkili kullanıcımız admin rolüne yükselir.

🛑 Kök Neden (Root Cause)

Sistem, "Denetimsiz HTTP Metotları" hatasına sahiptir. Erişim kontrol mekanizmaları, HTTP metodundan bağımsız olarak (GET, POST, PUT, DELETE fark etmeksizin) her kritik istekte kullanıcının oturumunu ve yetkisini doğrulamak zorundadır. Güvenliğin sadece POST metoduna endekslenmesi bu zafiyete yol açmıştır.
🛡️ Çözüm Önerisi (Remediation)

    Metottan Bağımsız Yetkilendirme: /admin/* altındaki tüm uç noktalara (endpoints) gelen isteklerin, HTTP metoduna bakılmaksızın (GET/POST fark etmeksizin) merkezi bir filtre/middleware üzerinden geçmesi ve Role-Based Access Control (RBAC) kontrolüne tabi tutulması gerekir.

    Girdi Doğrulama ve Sıkılaştırma: Sunucu, belirli bir endpoint için yalnızca beklenen HTTP metodunu kabul etmeli (Örn: Yetki yükseltme sadece POST olmalı), GET ile gelen istekleri doğrudan 405 Method Not Allowed hatasıyla reddetmelidir.
