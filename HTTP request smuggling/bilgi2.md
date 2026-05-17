# 🚀 HTTP/2 Request Smuggling - Technical Cheat Sheet & Write-Up

**Zorluk Seviyesi:** Advanced (İleri Seviye)  
**Kategori:** Web Application Security, Protocol Downgrade Attacks  
**Temel Konsept:** HTTP/2'den HTTP/1.1'e çeviri hataları (Downgrade) ve Request Tunneling.

---

## 🌍 1. HTTP/2 ve HTTP/1.1 Arasındaki Temel Farklar
HTTP/2 Request Smuggling zafiyetini anlamak için protokol mimarisindeki değişimi bilmek şarttır:
* **HTTP/1.1 (Metin Tabanlı):** Veriler düz metindir. İsteklerin bitişi `\r\n` (CRLF) karakterleri, `Content-Length` (CL) veya `Transfer-Encoding` (TE) başlıklarıyla belirlenir.
* **HTTP/2 (İkili / Binary):** Veriler çerçeveler (frames) halinde şifrelenir. Her çerçevenin boyutu matematiksel olarak bellidir. CL veya TE başlıklarına ihtiyaç duymaz. İstek satırı yerine `:method`, `:path`, `:authority` gibi sahte başlıklar (pseudo-headers) kullanır.

---

## 💥 2. Zafiyetin Kaynağı: "Sürüm Düşürme" (HTTP/2 Downgrade)
Modern web mimarilerinde Ön Sunucu (Load Balancer / WAF) istemciyle **HTTP/2** konuşurken, Arka Sunucu (Backend) ile eski uyumluluk için **HTTP/1.1** konuşur. 
Ön sunucu, HTTP/2 paketini alıp HTTP/1.1'e çevirirken (Downgrade) esnek kuralları sıkı bir şekilde filtrelemezse **İstek Kaçakçılığı (Request Smuggling)** meydana gelir.

---

## 🥷 3. Temel Saldırı Vektörleri

### A. H2.CL ve H2.TE Saldırıları
Ön sunucu HTTP/2 kullandığı için `Content-Length` veya `Transfer-Encoding` başlıklarını umursamaz. Ancak biz paketin içine bu başlıkları manuel olarak eklersek, ön sunucu bunları HTTP/1.1 formatına çevirip arka sunucuya iletir. Arka sunucu bu sahte boyutları ciddiye alır ve klasik **CL.TE** veya **TE.CL** zafiyetleri tetiklenir.

### B. CRLF Injection (İstek Bölme - Request Splitting)
HTTP/2 binary olduğu için başlıkların (headers) içine her türlü özel karakteri (örn: `\r\n`) kabul eder. 
* Kasıtlı olarak bir başlığa `\r\n` ekleriz (Örn: `Foo: bar\r\nTransfer-Encoding: chunked`).
* Ön sunucu bunu HTTP/1.1'e çevirdiğinde, o `\r\n` karakterleri gerçek bir "Satır Atlama" komutuna dönüşür ve paket ortadan ikiye bölünerek **Kaçak İstek (Smuggled Request)** yaratır.

---

## 🎯 4. Sömürü Senaryoları (Exploitation Tactics)

### Senaryo 1: Request Desync (Ortak Bağlantı Zehirleme)
* **Durum:** Arka sunucu tüm kullanıcılar için tek bir ortak bağlantı (pipe) kullanıyorsa.
* **Saldırı:** H2.CL yöntemiyle arka sunucuyu kandırıp, eksik bir komutu boruda asılı bırakırız. 
* **Etki:** Bizden hemen sonra siteye giren masum bir kullanıcının isteği, bizim asılı kalan komutumuzla birleşir. Kullanıcının oturumunu (Cookie) çalarak bizim belirlediğimiz eylemleri (Örn: Zorla beğeni attırma) gerçekleştirmesini sağlarız.

### Senaryo 2: Request Tunneling (Özel Bağlantı Tünelleme)
* **Durum:** Ön sunucu her kullanıcı için arka sunucuya ayrı bir bağlantı açıyorsa (Kullanıcıları hackleyemeyiz).
* **Saldırı 1 (Internal Header Leaking):** Yankı (Reflection) yapan bir endpoint'e (`/hello?q=`) yarım bir istek kaçırırız. Ön sunucunun araya eklediği gizli başlıklar (Internal Headers), arka sunucu tarafından parametre sanılarak bize ekranda yansıtılır.
* **Saldırı 2 (Bypassing WAF/ACL):** Ön sunucu `/admin` sayfasına girmemizi engelliyordur. İzin verilen `/hello` sayfasına HTTP/2 isteği atarız. Ancak içine CRLF ile böldüğümüz `GET /admin` komutunu gizleriz. Ön sunucu `/hello`'ya gittiğimizi sanırken, arka sunucu gizli komutu okuyup bize Admin panelini verir.

### Senaryo 3: Web Cache Poisoning (Önbellek Zehirlenmesi)
* **Durum:** Ön sunucunun (HAProxy) aktif bir önbellekleme (Caching) mekanizması varsa.
* **Saldırı:** Sunucuya zararlı bir JS dosyası yükleriz. Ardından Request Tunneling ile iki istek birden yollarız: Birinci istek meşru bir dosya (`text.js`), ikinci istek (kaçak olan) zararlı dosyamız. Arka sunucu iki cevap döner. Ön sunucu meşru cevabı bize verirken, zararlı cevabı boruda bekletir. İkinci kez `text.js` istediğimizde, borudaki zararlı dosyayı alır ve "Bu text.js'dir" diyerek **Önbelleğine (Cache)** kaydeder.
* **Etki:** O an siteye giren herkesin tarayıcısı, bizim yüklediğimiz zararlı kodu çalıştırır ve tüm kullanıcıların Cookie'leri (Oturumları) saldırganın sunucusuna düşer.

### Senaryo 4: h2c Smuggling (Protokol Müzakeresi İstismarı)
* **Durum:** Sunucu şifresiz HTTP/2 (h2c) bağlantısını destekliyorsa.
* **Saldırı:** Hedef sunucuya `Upgrade: h2c` başlığı gönderilir. Eğer Ön Sunucu bu güncellemeyi yönetemeyip doğrudan Arka Sunucuya iletirse, Arka Sunucu ile saldırgan arasında doğrudan, şifresiz bir HTTP/2 tüneli kurulur.
* **Etki:** Ön sunucu bu tünelin içinden geçen trafiği göremez. Bu sayede tüm WAF ve Frontend güvenlik kuralları tamamen atlatılır (Bypass). Otomatize etmek için `h2csmuggler` aracı kullanılır.

---

## 🛠️ 5. Çözüm Önerileri (Remediation)
1. **End-to-End HTTP/2:** Mümkünse HTTP/2'den HTTP/1.1'e çeviri yapmaktan kaçının. Ön sunucudan arka sunucuya kadar tüm mimaride HTTP/2 kullanın.
2. **Sıkı Başlık Doğrulaması:** Eğer downgrade işlemi zorunluysa, Ön Sunucunun HTTP/2 isteklerindeki özel karakterleri (özellikle `\r\n`), `Content-Length` ve `Transfer-Encoding` başlıklarını HTTP/1.1'e çevirirken kesinlikle filtrelediğinden/sildiğinden emin olun.
3. **h2c Kapatılması:** Modern mimarilerde "Cleartext HTTP/2" (h2c) desteğini tamamen devre dışı bırakın.
