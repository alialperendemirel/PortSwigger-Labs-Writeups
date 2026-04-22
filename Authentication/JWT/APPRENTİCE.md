Harika bir ilerleme! JWT zafiyetleri mülakatlarda ve gerçek dünya sızma testlerinde (pentest) çok sık karşına çıkacak. GitHub repon için önceki formata uygun, kısa, net ve hemen hatırlamanı sağlayacak bir özet hazırladım.

Bunu doğrudan kopyalayıp GitHub'daki Markdown (Readme) dosyana yapıştırabilirsin:
🛡️ PortSwigger: JWT (JSON Web Token) Zafiyetleri

Bu repo, web uygulamalarının oturum yönetiminde kullanılan JWT (JSON Web Token) mekanizmalarındaki mimari ve konfigürasyon hatalarını barındırmaktadır.
1. JWT authentication bypass via unverified signature (Doğrulanmayan İmza)

    Zafiyetin Mantığı: Sunucunun, gelen JWT'nin imzasını (3. parça) hiç kontrol etmemesi. Sadece ortadaki Payload (Yük) kısmına bakarak kullanıcıya körü körüne güvenmesi.

    Sömürü (Exploitation): 1. Token'ın 2. parçası (Payload) Base64 ile çözülür (Decode).
    2. İçerideki "sub": "wiener" (veya standart kullanıcı) değeri, "sub": "administrator" olarak değiştirilir.
    3. Yeni değer tekrar Base64 ile kodlanır (Encode) ve token'ın ortasına yerleştirilerek istek gönderilir. İmza bozulsa bile sunucu kontrol etmediği için sahte kimlik kabul edilir.

    Alınacak Ders (Mitigation): JWT kütüphaneleri kullanılırken token'lar sadece okunmamalı (decode), mutlaka sunucunun gizli anahtarı (Secret Key) ile imza doğrulaması (verify) yapılmalıdır.

2. JWT authentication bypass via flawed signature verification (alg: none Zafiyeti)

    Zafiyetin Mantığı: Sunucunun imza kontrolü yapması, ancak JWT başlığında (Header) belirtilen şifreleme algoritmasına koşulsuz güvenmesi. Başlıkta alg: none (Algoritma yok) yazıldığında, sistemin imza sormaktan vazgeçmesi.

    Sömürü (Exploitation):

        Header (1. Parça): Algoritma kısmı "alg": "none" olarak değiştirilir. (Sunucuya "imza yok" denilir).

        Payload (2. Parça): "sub": "administrator" olarak yetki yükseltilir.

        Signature (3. Parça): İmzanın tamamı silinir ancak token'ın en sonundaki nokta (.) bırakılır! (Örnek Format: Sahte_Header.Sahte_Payload.)

    Alınacak Ders (Mitigation): JWT doğrulaması yapılırken, dışarıdan gelen algoritma bilgisine (özellikle none değerine) asla güvenilmemelidir. Sunucu tarafında sadece izin verilen güvenli algoritmaları barındıran katı bir Beyaz Liste (Whitelist) uygulanmalıdır.
