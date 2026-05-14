# 🕷️ TryHackMe - Cross-Site Scripting (XSS) In-Depth Write-Up

Bu write-up, TryHackMe'deki **Cross-Site Scripting (XSS)** odasının detaylı çözümünü ve zafiyetlerin arka planındaki kök nedenleri (Root Causes) içermektedir. Oda, XSS'in temel türlerini (Reflected, Stored, DOM-based) sadece sömürmeyi değil, aynı zamanda güvenli kodlama (Patching) pratikleriyle nasıl engelleneceğini öğretmektedir.

## 📖 Temel Kavramlar ve Mekanikler

XSS saldırılarının kalbinde, modern tarayıcıların **Same-Origin Policy (SOP)** güvenlik mekanizmasını atlatmak yatar. Saldırgan, zararlı JavaScript kodunu doğrudan hedefin (güvenilir sitenin) içine enjekte ettiği için, tarayıcı kodu güvenilir kabul eder ve çalıştırır.

Zafiyetlerin temel nedenleri:
* **Yetersiz Girdi Doğrulaması (Validation) ve Temizleme (Sanitization):** Kullanıcı verisinin filtrelenmeden kabul edilmesi.
* **Çıktı Kodlamasının (Output Encoding) Eksikliği:** Tehlikeli karakterlerin (`<`, `>`, `&`) ekrana basılmadan önce zararsız HTML varlıklarına (`&lt;`, `&gt;`) dönüştürülmemesi.

---

## ⚡ Görev 1: Reflected XSS (Yansıyan XSS) - CVE-2023-38501

Reflected XSS, kullanıcıdan alınan girdinin hiçbir veritabanına kaydedilmeden doğrudan ekrana (HTML içerisine) yansıtılmasıyla oluşur. 

**Hedef Sistem:** Copyparty (Zafiyetli Versiyon)
**Port:** 3923

### Sömürü (Exploitation):
Sistemdeki URL parametresinin yansıma zafiyetini (CVE-2023-38501) kullanarak bir saldırı vektörü oluşturduk. Klasik `<script>` etiketleri engellendiği için, zararlı kodumuzu sahte bir resim dosyası çağırma hatası (`onerror`) üzerinden tetikledik.

**Kullanılan Payload (URL Encoded):**
`?k304=y%0D%0A%0D%0A%3Cimg+src%3Dcopyparty+onerror%3Dalert(1)%3E`

**Payload'ın Decode Edilmiş Hali:**
```html
?k304=y

<img src=copyparty onerror=alert(1)>

Not: %0D%0A (CR/LF - Satır atlama) karakterleri, arka plandaki WAF/Filtre mekanizmasını bozmak (Evasion) amacıyla kullanılmıştır.

Savunma (Remediation):
Reflected XSS'i önlemek için dillerin yerleşik HTML dönüştürme fonksiyonları kullanılmalıdır (Örn: PHP'de htmlspecialchars(), Node.js'de escapeHtml()).
💣 Görev 2: Stored XSS (Kalıcı XSS) - CVE-2021-38757

Stored XSS, en tehlikeli varyanttır. Zararlı payload doğrudan veritabanına kaydedilir ve sayfayı ziyaret eden her kullanıcının tarayıcısında "Zero-Click" (tıklamasız) olarak çalışır.

Hedef Sistem: Hospital Management System (Hastane Yönetim Sistemi)
Port: 80
Sömürü (Exploitation):

Hedef sistemin contact.php dosyasında, iletişim formundan gelen verilerin temizlenmeden (unsanitized) veritabanına kaydedildiğini tespit ettik.

    http://MACHINE_IP adresindeki iletişim (Contact) formuna gidildi.

    Message (Mesaj) kısmına kurbanın oturum çerezlerini (Cookies) çalacak şu payload yerleştirildi:
    HTML

    <script>alert(document.cookie)</script>

3. Hedef kullanıcı (Resepsiyonist / Admin) `admin:admin123` bilgileriyle giriş yapıp mesajları okuduğu anda payload tetiklendi.
4. Ekrana yansıyan uyarı kutusunda hedefin oturum anahtarı olan **`PHPSESSID`** ele geçirildi (Session Hijacking).

---

## 👻 Görev 3: DOM-Based XSS

DOM tabanlı XSS, sunucunun (Backend) hiçbir rol oynamadığı, tamamen tarayıcıda (Client-Side) gerçekleşen bir saldırıdır. 

**Kök Neden (Root Cause):**
Geliştiricinin URL'den (`window.location.search`) aldığı veriyi, filtrelemeden `document.write` veya `innerHTML` gibi tehlikeli (sink) JavaScript fonksiyonlarına göndermesidir. Bu tür saldırılar sunucu loglarında iz bırakmaz.

**Savunma (Remediation):**
Veri doğrudan HTML iskeletine kod olarak işlenmemeli, bunun yerine `textContent` (Sadece düz yazı) özelliği kullanılmalı ve girdi `encodeURIComponent()` ile temizlenmelidir.

---

## 🥷 Görev 4: Bağlamdan Kaçış ve Atlatma (Context & Evasion)

Başarılı bir XSS saldırısı için, payload'un sayfanın neresine düştüğü (Bağlam/Context) hayati önem taşır.

* **HTML Etiket İçi Kaçış:** Eğer kodumuz `<input value="BURADA">` içine düşüyorsa, önce etiketi kırmamız gerekir: `"><script>alert(1)</script>`
* **JavaScript İçi Kaçış:** Eğer bir script bloğunun içindeysek, önce değişkeni kapatıp kodu çalıştırır, kalanı yorum satırı yaparız: `';alert(1)//`

**Filtre Atlatma (Evasion):**
WAF (Web Application Firewall) sistemlerini atlatmak için aralara görünmez "Hex/Unicode" karakterler eklenebilir. Örneğin:
* `&#x09` (Horizontal Tab - Sekme) karakteri, `javascript:` kelimesini bölmek ve filtreyi atlatmak için `jav&#x09;ascript:alert(1)` şeklinde kullanılabilir. Tarayıcı bu boşluğu yok sayarak kodu çalıştırır.

---

## 🏆 Sonuç

Bu laboratuvar, web güvenliğinde "Girdi Doğrulama" ve "Çıktı Kodlama" ilkelerinin ne kadar kritik olduğunu göstermektedir. Geliştiriciler olarak sadece Client-Side doğrulamalara güvenmemeli, "Derinlemesine Savunma (Defense in Depth)" prensibiyle sunucu ve veritabanı seviyelerinde güvenlik kontrollerini uygulamalıyız.
