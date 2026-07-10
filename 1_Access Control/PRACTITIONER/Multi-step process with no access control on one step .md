# PortSwigger Labs: Broken Access Control - Vertical Privilege Escalation

Bu laboratuvarda, web uygulamalarında sıkça karşılaşılan **Hatalı Erişim Kontrolü (Broken Access Control)** zafiyetini sömürerek, düşük yetkili bir kullanıcının oturum bilgileriyle (Session Cookie) dikey yetki yükseltme (Vertical Privilege Escalation) işlemini gerçekleştirdim.

## 🎯 Hedef
Düşük yetkili (non-admin) bir kullanıcı hesabı üzerinden, normal şartlarda sadece yöneticinin (admin) erişebilmesi ve tetikleyebilmesi gereken "Kullanıcı Rolü Değiştirme" fonksiyonunu çalıştırmak.

## 🛠️ İzlenen Adımlar & Mantığı

### 1. Yönetici İsteklerinin Analizi
* İlk olarak **Admin** kimlik bilgileriyle sisteme giriş yapıldı.
* Yönetici panelinden `carlos` adlı kullanıcının rolünü yükseltme (promote) işlemi tetiklendi.
* Bu işlem gerçekleştirilirken arka planda dönen HTTP isteği **Burp Suite** aracılığıyla yakalandı ve üzerinde değişiklik yapabilmek amacıyla **Repeater** modülüne gönderildi.

### 2. Düşük Yetkili Oturumunun Elde Edilmesi
* Tarayıcının çerezlerinin karışmaması için gizli sekme (Incognito Window) açılarak bu kez **düşük yetkili (non-admin)** kullanıcı hesabı ile giriş yapıldı.
* Bu kullanıcıya ait güncel ve geçerli `session` çerezi (cookie) kopyalandı.

### 3. İstek Manipülasyonu ve Yetki Atlatma (Bypass)
* Burp Repeater modülünde bekletilen "yönetici yetkili" HTTP isteği üzerinde iki kritik değişiklik yapıldı:
  1. `Cookie: session=...` kısmına, gizli sekmeden alınan **düşük yetkili kullanıcının çerezi** yapıştırıldı.
  2. Terfi ettirilecek kullanıcı adı parametresi (örneğin `username=carlos`), **kendi düşük yetkili kullanıcı adımızla** değiştirildi.
* Manipüle edilmiş bu istek sunucuya tekrar gönderildi (Replay).

## 📊 Sonuç ve Güvenlik Açığı Analizi

Sunucu, gelen isteğin içeriğini (terfi ettirme fonksiyonunu) ve istek çerezindeki kullanıcının (bizim kullanıcımız) bunu yapmaya izni olup olmadığını **sunucu tarafında (Server-side) doğrulamadı**. 

Sadece geçerli bir oturum çerezi gördüğü için isteği kabul etti ve düşük yetkili kullanıcımız başarıyla yönetici (admin) rolüne yükseltildi.

### 🔒 Çözüm Önerisi (Remediation)
Bu tür zafiyetlerin önüne geçmek için:
* **Erişim Kontrol Listeleri (ACL):** Sunucu, hassas bir işlem içeren her HTTP isteğinde, isteği yapan oturum sahibinin (Session) o işlemi yapmaya yetkili olup olmadığını her seferinde arka planda (Server-side) kontrol etmelidir.
* Sadece URL'e veya arayüze erişimi kısıtlamak yeterli değildir; fonksiyon bazlı roller sıkı bir şekilde doğrulanmalıdır.
