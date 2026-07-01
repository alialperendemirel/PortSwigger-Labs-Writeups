# PortSwigger Labs: Broken Access Control - Bypass via Referer Header

Bu laboratuvarda, sunucu tarafındaki yetkilendirme mekanizmasının, kullanıcının manipüle edebileceği bir HTTP başlığı olan **`Referer`** bilgisine güvenmesi durumunda ortaya çıkan dikey yetki yükseltme (Vertical Privilege Escalation) zafiyetini sömürdüm.

## 🎯 Hedef
Geliştiriciler tarafından konulan `Referer` tabanlı güvensiz erişim kontrolü filtresini atlatarak (bypass), düşük yetkili bir kullanıcıyı yönetici (admin) rolüne yükseltmek.

## 🛠️ İzlenen Adımlar & Mantığı

### 1. Yönetici Senaryosunun İncelenmesi
* **Admin** hesabıyla giriş yapıldı ve kullanıcı terfi ettirme işlemi gerçekleştirildi.
* Bu işlem sırasında arka planda oluşan ve içinde geçerli `Referer: .../admin` başlığı barındıran HTTP isteği yakalanarak **Burp Repeater**'a gönderildi.

### 2. Güvensiz Filtre Kontrolünün Tespiti
* Gizli sekme açılarak **düşük yetkili (non-admin)** kullanıcı ile giriş yapıldı.
* Doğrudan `/admin-roles?username=carlos&action=upgrade` adresine gidilmeye çalışıldığında sunucu `Unauthorized` (Yetkisiz İşlem) hatası döndürdü. Çünkü tarayıcı bu isteğe geçerli bir `Referer` başlığı eklememişti ve sunucu isteğin admin panelinden gelmediğini anladı.

### 3. Referer Filtresinin Atlatılması (Bypass)
* Burp Repeater modülünde halihazırda bekleyen ve içinde `Referer` başlığı bulunan orijinal admin isteği açıldı.
* Bu istek üzerinde şu manipülasyonlar yapıldı:
  1. `Cookie: session=...` değeri, gizli sekmedeki **normal kullanıcının oturum çereziyle** değiştirildi.
  2. Hedef kullanıcı adı, **kendi normal kullanıcı adımızla** değiştirildi.
  3. İstek içindeki `Referer` başlığına (admin paneli adresi) **dokunulmadı**.
* İstek tekrar gönderildiğinde (Replay), sunucu isteğin admin panelinden geldiğini varsaydı ve düşük yetkili kullanıcımızı başarıyla terfi ettirdi.

## 📊 Sonuç ve Güvenlik Açığı Analizi

Bu zafiyetin temel nedeni, **girdilerin yetersiz doğrulanması (Improper Input Validation)** ve **güvenilmeyen istemci verilerine (Client-controlled data) bel bağlanmasıdır**. `Referer` başlığı kullanıcı (saldırgan) tarafından Burp Suite gibi araçlarla tamamen manipüle edilebildiği için, bir güvenlik sınırı olarak asla kullanılmamalıdır.

### 🔒 Çözüm Önerisi (Remediation)
* **Session Tabanlı Yetkilendirme:** Erişim kontrolleri, HTTP başlıkları (Referer vb.) yerine doğrudan sunucu tarafında saklanan güvenli oturum (Session) verilerine ve kullanıcının gerçek rol haritasına (RBAC - Role-Based Access Control) dayanmalıdır.
* İstemci tarafından gelen ve değiştirilebilir olan hiçbir HTTP başlığı (`Referer`, `User-Agent`, `X-Forwarded-For` vb.) kritik iş mantığı (Business Logic) kararlarında tek başına bir güvenlik doğrulayıcısı olmamalıdır.
