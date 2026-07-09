# PortSwigger Lab: Insufficient Workflow Validation (Yetersiz İş Akışı Doğrulaması)

## 🎯 Hedef
Satın alma iş akışındaki adımsal mantık hatasını sömürerek, yeterli bakiyemiz olmamasına rağmen **"Lightweight l33t leather jacket"** (Deri Ceket) ürününü ücretsiz satın almak.

---

## 💡 Zafiyetin Mantığı (Vulnerability Logic)
Uygulama, kullanıcının satın alma adımlarını sadece tarayıcı üzerindeki sırayla (1. Sepete ekle ➡️ 2. Ödeme yap ➡️ 3. Onay sayfasına git) takip edeceğini varsayıyor. 

Ancak backend (sunucu tarafı), en son adımdaki **sipariş onay isteği** geldiğinde, kullanıcının bir önceki "ödeme/bakiye düşme" adımını başarıyla tamamlayıp tamamlamadığını geriye dönük kontrol etmiyor (State/Durum kontrolü eksik). Bu sayede ödeme adımını atlayıp doğrudan onay isteğini tetiklediğimizde, sistem sepetteki pahalı ürünü bedavaya onaylıyor.

---

## 🛠️ Çözüm Adımları (Exploit Steps)

1. **Normal Akışı Analiz Et:**
   - `wiener:peter` bilgileriyle giriş yap.
   - Bakiyenin yettiği ucuz bir ürünü sepete ekle ve normal şekilde satın al.
   - **Burp Suite > Proxy > HTTP History** sekmesinden giden istekleri incele.

2. **Onay İsteğini Yakala:**
   - Siparişi kesin olarak bitiren son isteği bul: `GET /cart/order-confirmation?order-confirmation=true`
   - Bu isteğe sağ tıklayıp **Burp Repeater** sekmesine gönder.

3. **İş Akışını Atlat (Bypass):**
   - Mağazaya dön ve normalde paranın yetmediği pahalı **"Lightweight l33t leather jacket"** ürününü sepete ekle.
   - Tarayıcıdan "Checkout" (Ödeme) butonuna **basma**.
   - Doğrudan **Burp Repeater** sekmesine geç ve az önce yakaladığın onay isteğini `Send` diyerek gönder.

4. **Sonuç:**
   - Sunucu sepete bakıyor (Ceket var), ödeme kontrolü yapmadığı için siparişi onaylıyor. Ceket $0 karşılığında alınmış oluyor ve lab çözülüyor.

---

## 🛡️ Güvenli Kodlama (Remediation)
- Satın alma gibi çok adımlı (multi-step) işlemler için sunucu tarafında sıkı bir **Durum Makinesi (Server-Side State Machine)** kurulmalıdır.
- Kullanıcı `C` (Onay) adımına istek attığında, sunucu o oturum (session) için `B` (Ödeme Başarılı) bayrağının (flag) veri tabanında aktif olup olmadığını kontrol etmelidir.
- İstemcinin (kullanıcının) adımları sırayla takip edeceği varsayımına asla güvenilmemelidir.
