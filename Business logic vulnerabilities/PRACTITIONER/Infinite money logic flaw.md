# PortSwigger Lab: Infinite Money Logic Flaw (Sonsuz Para Mantık Hatası)

## 🎯 Hedef
E-ticaret sistemindeki indirim kuponu ve hediye kartı süreçleri arasındaki mantık hatasını sömürmek; Burp Suite Makro (Macro) otomasyonu kullanarak hesap bakiyesini sonsuz döngüyle katlamak ve **"Lightweight l33t leather jacket"** ürününü satın almak.

---

## 💡 Zafiyetin Mantığı (Vulnerability Logic)
Yazılımcı, sisteme tanımlanan `%30 hoş geldin indirim kuponunun` (`SIGNUP30`), mağaza kredisi/nakit yerine geçen **Hediye Kartı (Gift Card)** satın alımlarında kullanılmasını engellememiştir.

**Matematiksel Döngü:**
1. Kullanıcı $10 değerindeki Hediye Kartını sepetine ekler.
2. `%30` indirim kuponunu uygular ve kartı **$7** karşılığında satın alır.
3. Satın aldığı hediye kartını kendi hesabında aktif eder (Redeem) ve hesabına **$10** nakit yüklenir.
4. Sonuç: Tek bir döngüde **+$3 net kâr** elde edilir. Bu işlem otomatize edilirse bakiye sınırsızca artırılabilir.

---

## 🛠️ Çözüm Adımları (Exploit & Automation Steps)

### 1. Manuel Keşif
- Bültene (newsletter) kayıt olarak `SIGNUP30` kupon kodunu al.
- $10'lık hediye kartını alıp kuponla uygulayarak bakiye artış döngüsünü doğrula.

### 2. Burp Suite Makro Kurulumu (Automation)
- **Settings > Sessions > Session Handling Rules** alanına git ve yeni bir kural ekle.
- **Scope** sekmesinden tüm URL'leri (`Include all URLs`) dahil et.
- **Details > Rule Actions** kısmından **Run a macro** seçeneğini ekle ve şu 5 isteği sırasıyla kaydet:
  1. `POST /cart` (Sepete kart ekleme)
  2. `POST /cart/coupon` (Kupon uygulama)
  3. `POST /cart/checkout` (Ödeme yapma)
  4. `GET /cart/order-confirmation` (Kodun üretildiği onay sayfası)
  5. `POST /gift-card` (Kodu hesaba yükleme)

### 3. Parametre Köprüsü ve Regex (Parameter Handling)
- 4. istek olan `GET /cart/order-confirmation` seçilip **Configure item** denir. Sayfanın altındaki hediye kodu seçilerek `gift-card` adında bir özel parametre (custom parameter) tanımlanır.
- 5. istek olan `POST /gift-card` seçilip, buradaki `gift-card` parametresinin bir önceki (4. istek) yanıttan türetilmesi (`derived from prior response`) söylenir.

### 4. Saldırıyı Tetikleme (Intruder)
- `GET /my-account` isteğini **Intruder**'a gönder. Attack tipi: `Sniper`.
- Payload tipi olarak **Null Payloads** seç ve ceket fiyatına ulaşmak için `412` adet istek üretmesini söyle.
- **Resource Pool** sekmesinden eş zamanlı istek sayısını (`Maximum concurrent requests`) **1** olarak ayarla (İsteklerin sırasının bozulmaması için bu kritik adımdır).
- Saldırıyı başlat. Bakiye $1200 üzerine çıktığında mağazadan ceketi satın alarak labı tamamla.

---

## 🛡️ Güvenli Kodlama (Remediation)
- **Ürün Kategorizasyonu:** İndirim kuponlarının uygulanabileceği ürün kategorileri backend tarafında sıkı şekilde filtrelenmelidir. Hediye kartları, nakit muadili (currency) olduğu için promosyon ve indirim kodlarından tamamen muaf tutulmalıdır.
- **Hız Sınırı (Rate Limiting) & Bot Engelleme:** Kısa sürede ardı ardına yapılan sepet, ödeme ve kupon işlemlerine karşı sunucu tarafında koruma (Rate limiting veya Captcha) mekanizmaları kurulmalıdır.
