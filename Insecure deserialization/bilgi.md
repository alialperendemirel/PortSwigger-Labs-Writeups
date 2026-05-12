# TryHackMe: Insecure Deserialization (Güvensiz Ters Serileştirme) - Cheat Sheet

Bu belge, OWASP Top 10 listesinin en tehlikeli ve karmaşık zafiyetlerinden biri olan "Insecure Deserialization" (Güvensiz Ters Serileştirme) veya diğer adıyla "Object Injection" (Nesne Enjeksiyonu) açığının temel mantığını, sömürü adımlarını ve savunma yöntemlerini özetler.

## 🧠 1. Temel Kavramlar: Serileştirme Nedir?

Bir yazılımda (Örn: PHP, Java, Python) veriler RAM üzerinde "Nesneler (Objects)" olarak yaşar. Bu nesneleri bir veritabanına kaydetmek veya HTTP üzerinden çerez (cookie) olarak göndermek için düz metne veya bayt (binary) dizisine çevirmek gerekir.

* **Serialization (Serileştirme - Paketleme):** RAM'deki canlı bir objeyi alıp, transfer edilebilmesi için yassı bir metne/pakete dönüştürme işlemidir. (Örn: PHP'de `serialize()`, Python'da `pickle.dumps()`).
* **Deserialization (Ters Serileştirme - Paketi Açma):** Gelen bu yassı metni (paketi) okuyup, içindeki bilgilere göre RAM'de o objeyi tekrar "canlı" hale getirme işlemidir. (Örn: PHP'de `unserialize()`, Python'da `pickle.loads()`).

## ⚠️ 2. Zafiyetin Özü (Neden Ortaya Çıkar?)

Zafiyet, sunucunun kullanıcıdan gelen serileştirilmiş veriye (Örneğin bir oturum çerezine) **körü körüne güvenmesinden** kaynaklanır. Sunucu, `unserialize()` işlemini yaparken paketin içinin değiştirilip değiştirilmediğini kontrol etmez. 

Bu durum saldırgana iki farklı kapı açar:

### ⚔️ Saldırı Tipi A: Veri ve Yetki Manipülasyonu (Data Tampering)
Eğer sistem yetkilendirme kontrolünü sadece çerezin içindeki verilere bakarak yapıyorsa, kullanıcı kendi paketini açıp değerleri değiştirebilir.
* **Orijinal Çerez:** `O:5:"Notes":2:{s:4:"user";s:5:"guest";s:12:"isSubscribed";b:0;}`
* **Saldırı:** Kullanıcı metni Base64 ile çözer, abonelik durumu olan `b:0` (False) değerini `b:1` (True) olarak değiştirir ve tekrar Base64 ile şifreleyip sunucuya yollar.
* **Sonuç:** Yetki Yükseltme (Privilege Escalation) / Kimlik Doğrulama Atlama.

### 💣 Saldırı Tipi B: Object Injection ve RCE (Uzaktan Kod Çalıştırma)
İşte bu açığı ölümcül yapan kısım burasıdır. Saldırgan sadece veriyi değiştirmekle kalmaz, paketin içine **sistemin beklemediği bambaşka bir zararlı obje (Truva Atı)** koyar.

**Nasıl Çalışır? (Magic Methods)**
1. Nesne yönelimli programlamada (OOP), nesneler bellekte canlandığında (veya yok edildiğinde) otomatik olarak çalışan "Sihirli Metotlar" vardır (Örn: PHP'de `__wakeup()`, `__destruct()`).
2. Saldırgan, hedef sistemin kaynak kodunda bu sihirli metotları kullanan ve içinde `exec()`, `system()`, `eval()` gibi tehlikeli fonksiyonlar barındıran zafiyetli bir sınıf (Class) tespit eder (Örn: `MaliciousUserData` sınıfı).
3. Saldırgan, bu zararlı sınıfı ve çalışmasını istediği komutu (Örn: `nc -e /bin/sh`) serileştirip sunucuya yollar.
4. Sunucu `unserialize()` yaptığı saniye bu obje bellekte canlanır, sihirli metot `__wakeup()` otomatik tetiklenir ve sunucu hacklenir.

## 🛠️ 3. Otomasyon ve Kullanılan Araçlar

Gerçek dünyada Laravel, Symfony, Spring gibi devasa framework'lerde bu zararlı objeleri (Truva atlarını) elle yazmak imkansızdır. Bu objelerin uç uca eklenerek RCE oluşturmasına **"Gadget Chain" (Cihaz Zinciri)** denir. Bunun için şu otomatik araçlar kullanılır:

* **PHPGGC (PHP Generic Gadget Chains):** PHP tabanlı sistemlerde (Laravel, CodeIgniter vb.) serileştirme zafiyetlerini sömürmek için otomatik Payload (Gadget Chain) üreten araçtır.
  * *Kullanım Örneği:* `php phpggc Laravel/RCE3 system "whoami"`
* **Ysoserial:** Aynı mantığın Java tabanlı uygulamalar için tasarlanmış, dünyanın en meşhur RCE payload üreticisidir.

## 🛡️ 4. Savunma ve Çözüm Önerileri (Mitigation)

Bir yazılımcı olarak bu zafiyetten korunmanın kesin kuralları:

1. **Güvensiz Serileştirmeden Kaçının:** Veri transferi için dilin kendi serileştirme fonksiyonlarını (`serialize()`, `pickle`) kullanmak yerine, sadece veri taşıyan ve obje çalıştırma yeteneği olmayan **JSON** formatını (`json_encode()`, `json_decode()`) kullanın.
2. **Kullanıcı Girdisine Güvenmeyin:** Eğer mutlaka `unserialize()` kullanılacaksa, paketin bütünlüğünü doğrulamak için kriptografik imzalar (HMAC) kullanılmalıdır.
3. **Tehlikeli Fonksiyonları İzole Edin:** `__wakeup()`, `__destruct()` gibi sihirli metotların içerisinde asla `eval()`, `exec()`, `system()` gibi sistemde komut çalıştıran (OS Command) yapılar bulundurmayın.
