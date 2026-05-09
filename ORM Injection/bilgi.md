# TryHackMe: ORM Injection - Cheat Sheet

Bu belge, modern web uygulamalarında (Laravel, Django, Spring vb.) Veritabanı ile Yazılım arasındaki köprü olan ORM (Object-Relational Mapping) katmanındaki zafiyetlerin tespitini, sömürülmesini ve güvenli kodlama pratiklerini özetler.

## 🧠 1. ORM Nedir ve Neden Kullanılır?
ORM, nesne yönelimli programlama (OOP) dilleri ile ilişkisel veritabanları (SQL) arasında veri dönüşümü sağlayan bir "tercüman"dır. Geliştiricilerin SQL yazmadan, kendi dillerindeki objelerle (Örn: `User::find(1)`) veritabanı işlemi yapmasını sağlar.

**Popüler Frameworkler ve ORM'leri:**
* **PHP (Laravel):** Eloquent
* **PHP (Symfony):** Doctrine
* **Ruby (Ruby on Rails):** Active Record
* **Python (Django):** Django ORM
* **Python:** SQLAlchemy
* **Java (Spring):** Hibernate
* **C# (.NET):** Entity Framework

---

## ⚔️ 2. SQL Injection vs ORM Injection
* **SQL Injection:** Doğrudan veritabanını hedefler (`SELECT * FROM...`). WAF'lar (Güvenlik Duvarları) tarafından kolayca tespit edilebilir.
* **ORM Injection:** ORM framework'ünün sorgu oluşturma mantığını hedefler. Girdi, ORM fonksiyonlarını manipüle ettiği için oluşan SQL yapısı farklıdır ve WAF'ları atlatmak çok daha kolaydır.

---

## 🕵️‍♂️ 3. Keşif ve Tespit (Fingerprinting & Testing)
Black-box bir testte arka planda hangi ORM'in çalıştığını anlamak için:
* **Cookie İsimleri:** Örn: `laravel_session` (Laravel/Eloquent kullanıldığını gösterir).
* **Hata Mesajları:** Girdi alanına `'` veya `1'` yazılarak ORM çökertilir. Dönen hata sayfasındaki yollar (Örn: `DOCUMENT_ROOT: /var/www/html/public`) veya `SQLSTATE[42000]` hataları altyapıyı ele verir.

---

## 💥 4. İstismar (Exploitation) - Zayıf Kodlama (Weak Implementation)
Geliştiricilerin ORM'in güvenliğini devre dışı bırakan **"Raw (Çıplak)"** metodları kullanmasından kaynaklanır.

**Tehlikeli Metotlar:**
* **Laravel:** `whereRaw()`, `DB::raw()`
* **Django:** `extra()`, `raw()`
* **Node.js (Sequelize):** `sequelize.query()`

**Sömürü Örneği (Laravel - `whereRaw`):**
* **Zafiyetli Kod:** `$users = Admins::whereRaw("email = '$email'")->get();`
* **Payload:** `1' OR '1'='1`
* **Arka Planda Oluşan SQL:** `SELECT * FROM users WHERE email = '1' OR '1'='1'`
* **Sonuç:** Authentication Bypass veya tüm tablonun sızdırılması (Data Dump).

---

## 🪄 5. İleri Seviye İstismar - Paket Zafiyetleri (Vulnerable Packages)
Bazen kod doğru yazılsa da kullanılan ORM paketinin kendisi hatalı olabilir (Örn: Laravel Spatie Query Builder). Özellikle `ORDER BY` gibi normalde SQLi yapılması imkansıza yakın olan yerlerde ORM'in kendi özellikleri silah olarak kullanılır.

**Sömürü Örneği (JSON Operatörü ile ORDER BY Bypass):**
* **Senaryo:** `.../query_users?sort=name` (İsme göre sırala ve `LIMIT 2` ile sadece 2 kişi getir).
* **Amaç:** `LIMIT 2`'yi kırıp tüm veritabanını çekmek.
* **Silah:** MySQL'deki JSON okuma operatörü olan `->` işaretini kullanarak Laravel'in arka planda açtığı parantezleri kırmak.
* **Zehirli Payload:** `name->"%27)) LIMIT 100#`
* **URL Encode Edilmiş Hali:** `name-%3E%22%27))%20LIMIT%20100%23`
* **Oluşan Nihai SQL:** `SELECT * FROM users ORDER BY json_unquote(json_extract(name, '$.""')) LIMIT 100#"')) ASC LIMIT 2`
* **Sonuç:** Orijinal `LIMIT 2` yorum satırına (`#`) düşer, bizim `LIMIT 100` komutumuz çalışır.

---

## 🛡️ 6. Savunma ve Güvenli Kodlama (Mitigation)
ORM Injection'ı önlemenin temel kuralları:
1. **Parametreli Sorgular (Parameterized Queries):** `whereRaw()` gibi çıplak fonksiyonlar asla kullanılmamalı. Bunun yerine ORM'in standart güvenli fonksiyonları (Örn: `where('email', $email)`) kullanılmalıdır.
2. **Çift Taraflı Validasyon:** Girdiler hem istemci (Client) hem de sunucu (Server) tarafında doğrulanmalıdır.
3. **Allowlisting (Beyaz Liste):** Özellikle sıralama (`sort`) gibi parametreler kullanıcıdan direkt metin olarak alınmamalı, sadece sunucuda önceden tanımlanmış değerlere (Örn: `['name', 'age', 'date']`) izin verilmelidir.
