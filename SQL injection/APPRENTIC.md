1. SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

(WHERE bloğundaki SQL enjeksiyonu ile gizli verilerin çekilmesi)

    Olay: Web sitesi ürünleri listelerken sadece yayınlanmış (released = 1) olanları gösteriyor. Biz bu filtreyi kırıp yayınlanmamış ürünleri de listeliyoruz.

    Mantığı (Neden Çalışıyor?): Veritabanı sorgusu normalde "Kategori Hediyeler OLSUN VE Yayınlanmış OLSUN" der. Biz araya girip "VEYA 1=1" (yani her zaman doğru) diyerek sorguyu manipüle ediyoruz.

    Kritik Nokta: OR 1=1 mantığı. Bilgisayar mantığında "1 eşittir 1" ifadesi her zaman TRUE (Doğru) olduğu için, veritabanı filtreyi görmezden gelir ve her şeyi (gizli olanları da) döker.

    Örnek Payload: ' OR 1=1 --

2. SQL injection vulnerability allowing login bypass

(Giriş atlatmaya izin veren SQL enjeksiyonu)

    Olay: Kullanıcı adı ve şifre soran bir giriş panelini, şifreyi bilmeden sadece kullanıcı adını (örneğin administrator) kullanarak geçmek.

    Mantığı (Neden Çalışıyor?): Yazılımın arkadaki sorgusu şöyledir: Kullanıcı adı bu mu? VE Şifre bu mu?. Biz kullanıcı adını yazıp hemen arkasına "Yorum Satırı" (--) işareti koyarız. Bu işaret, veritabanına "Buradan sonrasını okuma/işleme alma" emrini verir. Böylece sorgunun "VE Şifre bu mu?" kısmı silinmiş olur.

    Kritik Nokta: Yorum satırı karakterleri (-- veya #). Bu karakterler sorgunun geri kalanını (şifre kontrolünü) iptal eder.

    Örnek Payload: administrator' --
