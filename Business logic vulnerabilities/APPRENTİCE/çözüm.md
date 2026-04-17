1. Excessive trust in client-side controls (İstemci Tarafındaki Kontrollere Aşırı Güven)

    Zafiyetin Mantığı: Sunucunun, kullanıcının tarayıcısından (istemciden) gelen kritik verilere doğrulamadan güvenmesi. Yazılımcı, arayüzdeki fiyatın değiştirilemeyeceğini varsaymıştır.

    Sömürü (Exploitation): Ürün sepete eklenirken giden POST /cart isteği Burp Suite ile yakalanır. İstek gövdesindeki price (fiyat) parametresi manipüle edilerek çok düşük bir değere (örn: price=1) çekilir.

    Alınacak Ders (Mitigation): Sunucu, asla fiyat gibi kritik verileri istemciden almamalıdır. Kullanıcıdan sadece ürün_id ve miktar alınmalı, fiyat veritabanından (backend) çekilerek hesaplanmalıdır.

2. High-level logic vulnerability (Üst Düzey Mantık Zafiyeti / Negatif Miktar)

    Zafiyetin Mantığı: Uygulamanın, sepete eklenen ürünlerin miktar (quantity) değerinde matematiksel bir sınırlandırma yapmayı unutması. Eksi (-) değerlerin kabul edilmesi.

    Sömürü (Exploitation): Normal yollarla alınamayacak kadar pahalı bir ürün sepete eklenir. Ardından ucuz başka bir ürünün sepete eklenme isteği Burp ile yakalanır ve miktarı negatif bir değer yapılır (örn: quantity=-50). Matematiksel olarak ucuz ürün sepetin toplam tutarından düşer ve pahalı ürün bakiyenin yettiği bir fiyata alınır.

    Alınacak Ders (Mitigation): Kullanıcıdan alınan miktar parametreleri kesinlikle sunucu tarafında doğrulanmalıdır (if quantity > 0). Ayrıca sepet toplamının eksiye veya sıfırın altına düşmesi engellenmelidir.

3. Inconsistent security controls (Tutarsız Güvenlik Kontrolleri)

    Zafiyetin Mantığı: Sistemin farklı bölümlerinde aynı güvenlik kurallarının uygulanmaması. Kayıt (Register) olurken e-posta doğrulayan sistemin, Profil Güncelleme (Update) yaparken doğrulamayı unutması. Ayrıca yetkilendirmenin sadece e-posta uzantısına bakılarak verilmesi.

    Sömürü (Exploitation): Standart, geçerli bir e-posta ile kayıt olunur ve e-posta onaylanır. Hesap ayarlarına gidilip e-posta adresi hedef kurumun uzantısıyla (örn: hacker@dontwannacry.com) değiştirilir. Sistem bu yeni adresi doğrulamadan kabul eder ve uzantıya bakarak doğrudan Admin yetkisi tanımlar.

    Alınacak Ders (Mitigation): Kritik profil güncellemelerinde (özellikle e-posta) mutlaka yeni adrese doğrulama maili gönderilmelidir. Yetkilendirmeler e-posta uzantısıyla değil, veritabanındaki sağlam yetki rolleriyle (Role-Based Access Control) yapılmalıdır.

4. Flawed enforcement of business rules (İş Kurallarının Hatalı Uygulanması)

    Zafiyetin Mantığı: Sistemin, uygulanan kuralların tarihçesini tutmak yerine sadece "bir adım geriyi" hatırlaması.

    Sömürü (Exploitation): İki farklı indirim kuponu (örn: NEWCUST5 ve SIGNUP30) elde edilir. Sepette bu iki kupon art arda dönüşümlü olarak uygulanır (A -> B -> A -> B). Sistem sadece en son girilen kuponu kontrol edip "zaten girilmiş mi" diye baktığı için bu döngüyü yakalayamaz ve sepet tutarı sıfırlanana kadar indirim uygular.

    Alınacak Ders (Mitigation): İş kuralları tekil değişkenler üzerinden değil, diziler (array/list) üzerinden kontrol edilmelidir. Kullanılan tüm kuponlar bir listede tutulmalı ve "Bu kupon listede var mı?" veya "Sepette birden fazla kupon var mı?" şeklinde katı kurallar uygulanmalıdır.
