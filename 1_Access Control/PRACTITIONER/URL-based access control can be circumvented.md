🎯 Zafiyet Tanımı: Routing Misconfiguration & Reverse Proxy Bypass

Bu laboratuvarda, ön uç (Front-end Proxy/WAF) ile arka uç (Back-end Application Server) sunucuları arasındaki URL ayrıştırma (parsing) ve rota senkronizasyonu uyumsuzluğu istismar edilmiştir.

Uygulama, erişim kontrolü filtrelemesini Front-end üzerinde yaparken; yönlendirme (routing) mantığını Back-end üzerinde X-Original-URL HTTP başlığına (header) güvenerek gerçekleştirmektedir.
🛠️ Adım Adım Saldırı Mantığı (Attack Steps)

    Erişim Engeli (403 Forbidden):
    Tarayıcı üzerinden doğrudan /admin yoluna istek atıldığında, Front-end güvenlik duvarı isteği yakalar ve erişimi engeller.

    Front-end Filtresini Atlatma (Bypass):

        İstek satırındaki (Request Line) URL, herkesin erişimine açık olan ana sayfa (/) olarak değiştirilir.

        Bu sayede Front-end proxy, "İstek zararsız bir sayfaya gidiyor" mantığıyla güvenliği tetiklemez ve isteği Back-end'e iletir.

    Back-end Rota Manipülasyonu (X-Original-URL):

        İsteğe X-Original-URL: /admin başlığı eklenir.

        Back-end sunucu bu başlığı okuduğunda, kullanıcının gitmek istediği asıl hedefin /admin olduğunu varsayar ve admin panelinin içeriğini (200 OK) saldırgana döner.

    Parametreli Aksiyon Tetikleme (User Deletion):

        Hedef fonksiyona (/admin/delete) parametre göndermek için Front-end'i yanıltacak şekilde query string ana isteğe eklenir: GET /?username=carlos

        Gitmek istediğimiz asıl fonksiyon yolu ise başlıkta gizlenir: X-Original-URL: /admin/delete

        Back-end, hafızasındaki username=carlos verisini X-Original-URL'den aldığı silme rotasına paslayarak hedef kullanıcıyı siler.

📌 Çıkarılan Ders (Takeaway)

Güvenlik kararları (Access Control), ön uç ve arka uç arasında paylaştırılmamalıdır. Ön ucun engellediği veya izin verdiği bir rota ile arka ucun işlediği rota tanımı birbiriyle %100 senkronize olmalıdır. X-Original-URL veya X-Rewrite-URL gibi başlıklar üzerinde arka uç sunucusu doğrudan yönlendirme yapmamalıdır.
