🛡️ Lab: File path traversal, simple case
📝 Açıklama

Bu laboratuvarda, ürün görsellerini ekrana getiren sistemin dosya yolu parametrelerini yeterince filtrelememesi sonucu oluşan Path Traversal zafiyeti sömürülmüştür. Amaç, sunucunun kök dizinindeki hassas bir dosya olan /etc/passwd içeriğini okumaktır.
🎯 Çözüm Adımları

    İnceleme: Bir ürün görseline sağ tıklayıp "Resmi yeni sekmede aç" diyerek veya Burp Suite ile isteği yakalayarak görselin /image?filename=6.jpg gibi bir parametre ile çağrıldığı tespit edildi.

    Manipülasyon: Uygulamanın dosyaları /var/www/images/ gibi bir dizinden çektiği varsayılarak, dizin ağacında yukarı çıkmak için ../ (parent directory) ifadesi kullanıldı.

    Payload: filename parametresi şu şekilde değiştirildi:
    ../../../etc/passwd

    Sonuç: Sunucu, verilen yol tarifini takip ederek görseller dizinden çıkıp sistemin /etc/passwd dosyasına ulaştı ve içeriğini HTTP yanıtı olarak döndürdü.

💡 Temel Mantık

Path Traversal, kullanıcıdan alınan girdiyle (dosya adı gibi) sunucu tarafında bir dosya yolu oluşturulurken, girdinin kontrol edilmemesidir. ../ karakterleri, işletim sistemine "bir üst dizine git" komutu verir. Eğer uygulama bu karakterleri temizlemezse (sanitization), saldırgan sunucu üzerindeki yetkisi dahilindeki her dosyaya erişebilir.
