URL-based access control bypass (X-Original-URL Başlığı ile Erişim Kontrolünü Atlatma)

    Zafiyetin Mantığı: Sistemin ön uç (front-end / güvenlik duvarı) ile arka uç (back-end) sunucusu arasındaki iletişim ve yorumlama kopukluğu. Ön uç mekanizması sadece isteğin ilk satırındaki yola (GET /) bakarak geçiş izni verirken, arka ucun gidilecek asıl sayfayı belirlemek için X-Original-URL (veya X-Rewrite-URL) gibi özel HTTP başlıklarına (header) körü körüne güvenmesi.

    Sömürü (Exploitation):

        /admin paneline doğrudan girilmeye çalışıldığında ön uç sistemi bunu tespit edip engeller (Block).

        Burp Suite üzerinden istek GET / HTTP/2 olarak değiştirilir. Böylece ön uç "Sadece ana sayfaya gidiliyor" diyerek geçişe izin verir.

        Arka uca asıl hedefi göstermek için HTTP başlıklarına X-Original-URL: /admin/delete satırı eklenir.

        Kritik Adım: İşlemin gerçekleşmesi için gereken parametre (username=carlos) asıl isteğin ilk satırına eklenir. Nihai istek GET /?username=carlos HTTP/2 şeklinde sunucuya gönderilerek arka uçtaki silme işlemi tetiklenir.

    Alınacak Ders (Mitigation): Uygulama mimarisinde, ön uç ve arka uç sistemleri gelen istekleri aynı şekilde (tutarlı) yorumlamalıdır. Geçerli ve çok özel bir mimari sebep yoksa, arka uç sistemleri X-Original-URL veya X-Rewrite-URL gibi uygulamanın yönlendirmesini manipüle edebilecek başlıkları kesinlikle kabul etmemelidir.


    X-Original-URL veya X-Rewrite-URL
