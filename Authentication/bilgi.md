📝 Enumeration & Brute Force - Pentest Cheat Sheet

Bu doküman, kimlik doğrulama mekanizmalarındaki mantık hatalarını tespit etme (enumeration) ve zayıf korunan giriş noktalarına kaba kuvvet (brute force) saldırıları düzenleme yöntemlerini özetlemektedir.
1. Kullanıcı Tespiti ve "Geveze" Hatalar (Enumerating Users via Verbose Errors)

    Zafiyetin Mantığı: Güvenli bir sistem, başarısız bir giriş denemesinde sadece "Kullanıcı adı veya şifre hatalı" demelidir. Ancak hatalı yapılandırılmış sistemler, "Böyle bir kullanıcı bulunamadı" veya "Şifre yanlış" gibi farklı hata mesajları (Verbose Errors) döndürür.

    Saldırı Vektörü: Bu farklı hata mesajları analiz edilerek, sistemde gerçekte var olan kullanıcı adlarının bir listesi çıkarılır (User Enumeration).

    Neden Önemlidir? Kaba kuvvet saldırılarında zaman en değerli kaynaktır. Geçerli kullanıcı adları bilindiğinde, saldırı sadece bu hesaplar üzerine yoğunlaştırılarak başarı şansı ve hızı katlanarak artırılır.

2. Zayıf Parola Sıfırlama Mantığını Sömürme (Exploiting Vulnerable Password Reset Logic)

    Zafiyetin Mantığı: Parola sıfırlama mekanizmaları genellikle kullanıcıya bir e-posta veya SMS ile doğrulama kodu (token/OTP) gönderir. Eğer bu kod yeterince karmaşık değilse (örneğin sadece 4 haneli rakamlar) veya deneme sınırı (Rate Limiting) konmamışsa, mekanizma kırılabilir hale gelir.

    Saldırı Vektörü:

        Hedef kullanıcının şifre sıfırlama süreci tetiklenir.

        Burp Suite Intruder veya benzeri araçlarla 0000'dan 9999'a kadar tüm ihtimaller sunucuya hızlıca gönderilir.

        Doğru kod bulunduğunda, hesaba erişim sağlanır ve şifre değiştirilir.

3. HTTP Basic Auth Sömürüsü (Exploiting HTTP Basic Authentication)

    Zafiyetin Mantığı: Eski ama hala yönlendiricilerde (router), yönetici panellerinde ve bazı API'lerde kullanılan bir yöntemdir. Kullanıcı adı ve şifre kullanici:sifre formatında birleştirilir, Base64 ile kodlanır (şifrelenmez!) ve Authorization: Basic <base64_metni> başlığı altında gönderilir.

    Saldırı Vektörü: Bu sistemler genellikle modern CAPTCHA veya "hesap kilitleme" (account lockout) mekanizmalarından yoksundur. Bu durum, Hydra gibi araçlar kullanılarak devasa şifre listeleriyle (Dictionary Attack) durmaksızın saldırı yapılmasına olanak tanır.

4. OSINT ile Kaba Kuvvet Saldırılarını Güçlendirme (OSINT for Wordlists)

    Mantık: Rastgele oluşturulmuş devasa kelime listeleri kullanmak (Örn: rockyou.txt) her zaman işe yaramaz. İnsanlar şifrelerini oluştururken psikolojik olarak çevrelerindeki isimleri, tuttukları takımları, şirket adlarını veya doğum yıllarını kullanmaya eğilimlidir.

    Yöntem: Açık Kaynak İstihbaratı (OSINT) ile hedef kurumun çalışan isimleri, teknoloji yığınları, kuruluş yılları ve yerel terimleri toplanır. Cewl (web sitesinden kelime kazıma) veya Mentalist gibi araçlarla bu veriler harmanlanarak hedefe özel, kısa ama vurucu bir kelime listesi (Custom Wordlist) oluşturulur. Örneğin; SirketAdi2026!.

🛡️ Savunma ve Çözüm Önerileri (The Fix)

    Genel Hata Mesajları Kullanılmalı: Giriş başarısız olduğunda, hatanın kullanıcı adından mı yoksa şifreden mi kaynaklandığı belirtilmemelidir.

    Hız Sınırlandırması (Rate Limiting) Uygulanmalı: Belirli bir IP adresinden veya belirli bir hesaba yönelik art arda yapılan başarısız giriş denemeleri geçici olarak engellenmelidir.

    Hesap Kilitleme (Account Lockout): Çok fazla başarısız denemeden sonra hesap, kullanıcının e-posta onayı veya belirli bir süre geçene kadar kilitlenmelidir.

    Çok Faktörlü Kimlik Doğrulama (MFA): Parola ele geçirilse dahi sisteme girişi engelleyen en etkili yöntemdir.
