# PortSwigger Lab: Business Logic Vulnerability — Email Truncation ile Yetki Yükseltme

## Zafiyet Özeti
Uygulama, kullanıcıyı `dontwannacry.com` domain'ine ait mail adresine göre admin yetkisi veriyor. Ancak email adresi veritabanına kaydedilirken **255 karakterde kesiliyor (truncate)**, mail sunucusu ise adresi **tam haliyle** kullanıyor. Bu tutarsızlık, sahte bir `@dontwannacry.com` adresiyle admin paneline erişim sağlıyor.

## Adımlar

### 1. Keşif
- Burp'te **Target > Site map** üzerinden hedef domain'e sağ tıklayıp **Engagement tools > Discover content** ile içerik keşfi yapılır.
- `/admin` path'i bulunur. Erişilmeye çalışıldığında, hata mesajı `DontWannaCry` çalışanlarının erişebildiğini belirtir.
- Kayıt sayfasında, `DontWannaCry` çalışanlarının şirket emaili kullanması gerektiği yazılıdır.

### 2. Truncation Testi
- Lab banner'ından email client açılır, kendi benzersiz ID'niz not alınır (`YOUR-EMAIL-ID`).
- 200+ karakterlik bir email ile kayıt olunur:
  ```
  aaaa...aaa@YOUR-EMAIL-ID.web-security-academy.net
  ```
- Onay maili gelir, link tıklanır, giriş yapılır.
- **My Account** sayfasında email adresinin **255 karaktere kesildiği** görülür → uygulama email'i DB'ye yazarken sondan kesiyor.

### 3. Saldırı — Truncation'ı İstismar Etme
Amaç: `dontwannacry.com` ifadesinin son harfi (`m`) tam olarak **255. karakter** olacak şekilde adres kurgulamak. Böylece 255'ten sonrası (gerçek domain) kesilir ve geriye sahte biçimde geçerli görünen bir `@dontwannacry.com` adresi kalır.

```
[a'lardan oluşan dolgu (239 karakter)]@dontwannacry.com.YOUR-EMAIL-ID.web-security-academy.net
```

**Karakter hesaplaması:**
- `dontwannacry.com` = 16 karakter → pozisyon 240–255 arasına oturmalı
- Bu nedenle `@` işareti dahil dolgu kısmı = 239 karakter olmalı

### 4. Doğrulama ve Bitiriş
- Bu adresle yeni bir hesapla kayıt olunur.
- Mail sunucusu tam adresi kullandığı için onay maili gerçek adrese ulaşır (email client'ta görünür), link tıklanır.
- Giriş yapılır, **My Account** sayfasında email'in artık `...@dontwannacry.com` ile bittiği (gerçek domain'in kesilip atıldığı) görülür.
- `/admin` paneline erişim sağlanır.
- `carlos` kullanıcısı silinerek lab tamamlanır.

## Kök Neden
İki farklı sistem bileşeni aynı veriyi farklı işliyor:

| Bileşen | Davranış |
|---|---|
| Mail sunucusu (SMTP) | Adresi hiç kesmeden, tam haliyle kullanır → mail gerçek adrese ulaşır |
| Uygulama / Veritabanı | Adresi `VARCHAR(255)` gibi bir sütun sınırına göre **sessizce keser** (son kısım atılır) |

Yetkilendirme kontrolü, kesilmiş (DB'deki) adrese bakarak yapıldığından, saldırgan gerçek domain'i 256. karakterden sonrasına gizleyerek attırabiliyor ve sahte biçimde yetkili bir domain'e sahipmiş gibi görünebiliyor.

## Önlem
- Email doğrulama ve DB kaydı **aynı normalizasyon/validasyon adımından** geçmeli.
- Sütun uzunluk sınırı aşıldığında **sessizce kesme yerine hata döndürülmeli**.
- Yetki kontrolleri, kullanıcıdan alınan ham veri yerine tutarlı şekilde doğrulanmış veriye dayanmalı.
