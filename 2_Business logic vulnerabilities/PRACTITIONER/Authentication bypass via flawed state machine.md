# PortSwigger Lab: Authentication Bypass via Flawed State Machine (Hatalı Durum Makinesi ile Kimlik Doğrulama Atlatma)

## 🎯 Hedef
Giriş (Login) sürecindeki adımsal durum yönetimi (State Management) hatasını sömürerek, sisteme **Administrator (Admin)** yetkileriyle sızmak ve `carlos` kullanıcısını silmek.

---

## 💡 Zafiyetin Mantığı (Vulnerability Logic)
Bu zafiyet, backend (sunucu) mimarisinin **"Güvenli Varsayılanlar" (Secure Defaults)** ve **"En Düşük Yetki İlkesi" (Principle of Least Privilege)** kurallarını ihlal etmesinden kaynaklanır.

Yazılımcı sistemi tasarlarken şöyle hatalı bir mantık kurmuştur:
1. Kullanıcı başarılı şekilde giriş yaptığında (`POST /login`), sistem kullanıcının rolünü henüz bilmediği için arka planda geçici olarak **en yüksek yetkiyi (Admin)** tanımlar.
2. Ardından kullanıcıyı bir rol seçme sayfasına (`GET /role-selector`) yönlendirir.
3. Kullanıcı bu sayfada bir rol seçtiğinde (örn: Normal Kullanıcı), sunucuya giden istek doğrultusunda kullanıcının yetkisi Admin'den normal kullanıcıya **düşürülür**.

**Zafiyet Noktası:** Sunucu, kullanıcının rol seçme adımını (yani yetki düşürme aşamasını) kesin olarak tamamlayıp tamamlamadığını doğrulamıyor. Eğer saldırgan rol seçme isteğini tarayıcıda engelleyip (Drop edip) doğrudan ana sayfaya zıplarsa, sunucu hafızasındaki rolü "düşürülmemiş" haliyle, yani varsayılan **Admin** olarak bırakıyor.

---

## 🛠️ Çözüm Adımları (Exploit Steps)

1. **Keşif ve Analiz:**
   - `wiener:peter` bilgileriyle giriş yapıldığında sistemin bizi doğrudan ana sayfaya götürmediğini, arada bir rol seçme ekranına (`/role-selector`) yönlendirdiğini gözlemle.
   - Rol seçme ekranındayken doğrudan `/admin` adresine gitmeyi dene. Sistemin basit bir yetki kontrolü yaparak bizi engellediğini gör (Çünkü henüz rolümüz tanımsız/boş).

2. **İş Akışını Araya Girerek Bozma (Interception):**
   - Hesaptan çıkış yap (`Log out`) ve tekrar giriş sayfasına gel.
   - **Burp Suite > Proxy > Intercept** özelliğini `Intercept is on` konumuna getir.
   - Giriş bilgilerini yazıp butonuna bas.

3. **Kritik İsteği Engelleme (Drop):**
   - Burp Suite ekranına düşen ilk istek olan `POST /login` isteğini **Forward** diyerek sunucuya gönder.
   - Hemen ardından gelen ikinci istek olan `GET /role-selector` (Rol seçme ekranını yüklemeye çalışan istek) isteğini gördüğün an **Drop** butonuna basarak çöpe at. Sunucunun bu adımı işletmesini engelle.

4. **Bypass ve Sonuç:**
   - Tarayıcıya geri dön ve doğrudan sitenin ana sayfasına (`/`) git.
   - Sunucu rol düşürme adımını işletemediği için seni varsayılan yetkin olan **Administrator** olarak görecektir. Üst barda Admin Panelinin açıldığını doğrula.
   - Admin paneline girerek `carlos` kullanıcısını sil ve labı tamamla.

---

## 🛡️ Güvenli Kodlama (Remediation)
- **Fail-Secure (Güvenli Başarısızlık):** Bir kullanıcının rolü henüz kesin olarak belirlenmemişse, sistem varsayılan olarak en yüksek yetkiyi (Admin) değil, **en düşük yetkiyi (Misafir/Giriş Yapmamış)** atamalıdır. Yetki, süreç başarıyla tamamlandıkça yükseltilmelidir (Privilege Escalation yerine Privilege De-escalation mantığı kullanılmamalıdır).
- **Sıkı Durum Kontrolü:** Sunucu tarafında çok adımlı süreçler bir State Machine ile sıkıca takip edilmeli, ara adımlar tamamlanmadan ana fonksiyonlara erişim kesinlikle engellenmelidir.
