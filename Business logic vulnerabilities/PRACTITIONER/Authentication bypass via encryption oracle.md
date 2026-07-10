# PortSwigger Lab: Authentication Bypass via Encryption Oracle (Şifreleme Kahini ile Kimlik Doğrulama Atlatma)

## 🎯 Hedef
Uygulama içerisindeki bir mantık hatası nedeniyle dışarıya sızan şifreleme/şifre çözme mekanizmasını (Encryption Oracle) sömürmek; kendi gizli anahtarımızı üretmeden Hex düzeyinde blok manipülasyonu yapmak ve sisteme **Administrator** olarak giriş yaparak `carlos` kullanıcısını silmek.

---

## 💡 Zafiyetin Mantığı (Vulnerability Logic)
Bu zafiyet, iki farklı işlevin (Beni Hatırla çerezi ve Yorum hatası bildirimi) arka planda **aynı şifreleme algoritmasını ve aynı gizli anahtarı (secret key)** kullanmasından kaynaklanır.

1. **Giriş Süreci:** `stay-logged-in` çerezi sunucu tarafından şifrelenir ve çözüldüğünde `username:timestamp` formatını bekler.
2. **Yorum Süreci:** Geçersiz e-posta girildiğinde sistem girdiyi alır, başına sabit bir hata metni ekler (`Invalid email address: `) ve bunu şifreleyerek `notification` çerezi olarak tarayıcıya verir. Tarayıcı sayfayı yenilediğinde bu çerez çözülerek ekrana basılır.

**Kriptografik Engel ve Blok Yönetimi:**
Hata mesajını şifreleyen mekanizmayı kullanarak `administrator:[timestamp]` verisini şifreletebiliriz. Ancak sunucu girdimizin başına otomatik olarak **23 karakterlik (23 bayt)** `Invalid email address: ` önekini eklemektedir. Sunucu tarafında kullanılan şifreleme (AES-CBC vb.) **16 baytlık bloklar** halinde çalışır. 

Şifreli metinden cerrahi bir kesim yapabilmemiz için sileceğimiz alanın 16'nın katı olması şarttır. Bu yüzden matematiksel bir hizalama (Padding) yapmamız gerekir:
* Hedef silinecek boyut: **32 bayt** (16'nın en yakın katı).
* Mevcut önek: **23 bayt**.
* Eklenmesi gereken dolgu (Padding): $32 - 23 = \mathbf{9\text{ bayt}}$.

Girdimizin başına 9 tane önemsiz karakter (`xxxxxxxxx`) eklediğimizde toplam önek boyutu tam 32 bayt (2 tam blok) olur. Böylece şifreli verinin ilk 32 baytını Hex editörle sildiğimizde geriye kalan bloklar sunucu tarafından temiz ve hatasız bir şekilde çözülebilir.

---

## 🛠️ Çözüm Adımları (Exploit Steps)

### 1. Formatı ve Zaman Damgasını Keşfetme (Decryption)
- `wiener:peter` bilgileri ve "Stay logged in" seçeneği aktifken giriş yap. Giden istekteki `stay-logged-in` çerez değerini kopyala.
- Blog sayfasında geçersiz bir e-posta ile yorum yap. Yanıtta dönen `Set-Cookie: notification=...` yapısını içeren istekleri Burp Repeater'a aktar.
- `GET` (Yorum sayfası) isteğindeki `notification` çerezi yerine kendi `stay-logged-in` çerezini yapıştırıp gönder. Çözülen veriden formatı (`username:timestamp`) öğren ve **zaman damgasını (timestamp)** kopyala.

### 2. Matematiksel Hizalama ve Şifreleme (Encryption)
- `POST /post/comment` isteğindeki `email` parametresini şu şekilde manipüle et (Başında tam 9 adet x karakteri mevcut):
  `xxxxxxxxxadministrator:KOPYALANAN_ZAMAN_DAMGASI`
- İsteği gönder ve sunucunun üreterek `Set-Cookie: notification=` içinde verdiği yeni şifreli çerez değerini kopyala.

### 3. Cerrahi Hex Kesimi (Burp Decoder)
- Kopyaladığın yeni şifreli çerezi **Burp Decoder**'a gönder.
- Sırasıyla **Decode as... > URL** ve **Decode as... > Base64** işlemlerini uygula.
- En alttaki görünümü **Hex** moduna al. En baştan başlayarak tam **32 baytlık** alanı fareyle seç, sağ tıkla ve **Delete selected bytes** diyerek sil.
- Kalan temiz veriyi sırasıyla **Encode as... > Base64** ve **Encode as... > URL** adımlarıyla tekrar paketle ve oluşan nihai kodu kopyala.

### 4. Yetki Yükseltme (Exploit)
- Burp Proxy geçmişinden sitenin ana sayfasına giden temiz bir `GET /` isteğini Repeater'a al.
- `Session` çerezini tamamen sil. `stay-logged-in` çerezinin değerine ise Decoder'da manipüle edip ürettiğin o nihai kodu yapıştır.
- İsteği gönderdiğinde sunucu şifreyi çözecek, öndeki çöpler gittiği için doğrudan `administrator:[timestamp]` verisini okuyacaktır. Sayfada Admin Panelinin açıldığını doğrula.
- `/admin/delete?username=carlos` adresini tetikleyerek Carlos kullanıcısını sil ve labı tamamla.

---

## 🛡️ Güvenli Kodlama (Remediation)
- **Anahtar Ayrımı (Key Separation):** Uygulama içindeki farklı işlevler (oturum yönetimi, hata mesajları, kuponlar vb.) asla aynı kriptografik anahtar (Secret Key) ile şifrelenmemelidir.
- **Bütünlük Kontrolü (MAC/AEAD):** Sadece şifreleme (Encryption) yetersizdir. Şifreli metinlerin değiştirilmesini veya kırpılmasını engellemek için **AES-GCM** gibi hem şifreleme hem de veri bütünlüğü doğrulaması (Authenticated Encryption) sunan algoritmalar veya şifreli metne özel **HMAC** imzaları kullanılmalıdır.
