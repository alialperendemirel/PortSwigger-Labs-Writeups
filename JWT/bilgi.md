🛡️ JWT Security & Exploitation Cheat Sheet

JSON Web Token (JWT), sistemler arasında güvenli veri transferi sağlayan, Base64Url ile kodlanmış bir bilet sistemidir.
🧩 JWT Anatomisi

Bir JWT üç parçadan oluşur (noktalarla ayrılır):

    Header (Kırmızı): Algoritma ve tip bilgisi (alg, typ).

    Payload (Mor): Kullanıcı verileri ve yetkiler (Claims: sub, admin, exp, aud).

    Signature (Mavi): Biletin sahteliğini önleyen mühür.

    ⚠️ Unutma: Payload şifreli değildir! Sadece Base64 ile kodlanmıştır. Hassas veri asla buraya konulmamalıdır.

🚀 Temel Zafiyetler ve Saldırı Vektörleri
Zafiyet	Açıklama	Saldırı Tekniği
Sensitive Info Disclosure	Hassas verilerin Payload içinde saklanması.	jwt.io ile deşifre et ve oku.
Signature Bypass	Sunucunun imzayı kontrol etmemesi.	İmzayı sil ve yetkiyi değiştir.
None Algorithm	alg: none kabul edilmesi.	Header'da algoritmayı none yap, imzayı sil.
Weak Secret	Zayıf parola ile imzalama.	hashcat veya john ile parolayı kır.
Algo Confusion	RS256'dan HS256'ya düşürme.	Public Key'i şifre olarak kullan.
Exp Issues	exp değerinin olmaması/çok uzun olması.	Eski token'ı tekrar kullan (Replay).
Audience Relay	aud kontrolü yapılmaması.	X servisi biletini Y servisinde kullan.
🛠️ Güvenlik İçin En İyi Uygulamalar (The Fix)

    Güçlü Parolalar: HS256 için uzun ve rastgele "Secret" kullanın.

    İmza Kontrolü: verify_signature parametresini asla False yapmayın.

    Algoritma Kilidi: Kabul edilecek algoritmaları sınırlayın: algorithms=["HS256"].

    Kısa Ömür: exp (son kullanma tarihi) mutlaka ekleyin ve kısa tutun.
