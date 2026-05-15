# 🛡️ Web Security Laboratory: SOP & CORS Exploitation

Bu rapor, tarayıcı güvenlik politikaları (SOP), bu politikaların esnetilmesi (CORS) ve bu mekanizmaların yanlış yapılandırılmasından kaynaklanan kritik zafiyetlerin sömürülmesine dair teknik notlarımı içerir.

---

## 🧱 1. Temel Mekanizmalar: SOP ve CORS

### SOP (Same-Origin Policy) - "Aynı Köken Politikası"
Tarayıcıların içine gömülü olan en temel güvenlik kalkanıdır. Bir sitedeki JavaScript kodunun, başka bir sitedeki hassas verileri okumasını engeller. "Aynı Köken" sayılması için şu üçlünün birebir aynı olması gerekir:
1. **Protokol** (http vs https)
2. **Alan Adı** (site.com vs api.site.com)
3. **Port** (80 vs 8080)

### CORS (Cross-Origin Resource Sharing) - "Vize Sistemi"
SOP'un katı kurallarını esnetmek için icat edilmiş bir izin sistemidir. Sunucu, HTTP yanıt başlıkları (Headers) kullanarak tarayıcıya hangi dış sitelere izin verdiğini bildirir.
* **Kritik Başlık:** `Access-Control-Allow-Origin (ACAO)`
* **Önemli Kural:** Veriyi engelleyen sunucu değildir; sunucu veriyi yollar, **tarayıcı** ACAO başlığına bakarak veriyi JavaScript'ten gizler veya gösterir.

---

## ⚔️ 2. CORS Yanlış Yapılandırma (Misconfiguration) Sömürüleri

Laboratuvar ortamında gerçekleştirilen kritik saldırı vektörleri:

### A. Arbitrary Origin (Gelişigüzel Köken)
**Senaryo:** Sunucu, gelen isteğin `Origin` başlığını hiçbir kontrol yapmadan yanıtın `ACAO` başlığına aynen yansıtır (Echo back).
* **Zafiyet:** Sunucu "Kim gelirse gelsin verimi okuyabilir" der.
* **Sömürü:** Kurban saldırganın sitesine girdiğinde, saldırganın JS kodu kurbanın oturumuyla (çerezleriyle) hedef siteden veriyi çeker ve kendi sunucusuna postalar.

### B. Bad Regex (Hatalı Düzenli İfade)
**Senaryo:** Yazılımcı sadece kendi domain'ine izin vermek ister ama hatalı Regex yazar. Örn: `preg_match('#corssop.thm#', origin)`.
* **Zafiyet:** Başlangıç (`^`) ve bitiş (`$`) sınırları olmadığı için, içinde "corssop.thm" kelimesi geçen her adrese izin verilir.
* **Bypass:** Saldırgan `corssop.thm.evilcors.thm` gibi bir subdomain açarak filtreyi atlatır ve sunucunun güvenini kazanır.

### C. Null Origin Misconfiguration & XSS Chaining
**Senaryo:** Sunucu `Origin: null` değerine güvenir.
* **Zafiyet:** Tarayıcılar; yerel dosyalar veya `data:` URL formatındaki iframe'ler üzerinden atılan istekleri "null" olarak işaretler.
* **Sömürü (Vulnerability Chaining):** 1. Hedefteki bir **XSS** açığı kullanılarak kurbanın tarayıcısına bir kod enjekte edilir.
    2. Bu kod, Base64 formatında bir **iframe** açarak isteği `null` kökenine büründürür.
    3. Sunucu "null" izni verdiği için veriyi teslim eder ve çalınan veri saldırganın `receiver.php` dosyasına sızdırılır.

---

## 🛠️ 3. Teknik Hazırlık ve Exfiltration

Saldırıların başarılı olması için saldırgan tarafında kurulan düzenek:
* **`/etc/hosts`:** Farklı alan adları arası etkileşimi simüle etmek için kurban ve saldırgan domainleri yerel IP'ye yönlendirildi.
* **Exfiltrator Server:** Kurbandan çalınan verileri yakalamak için PHP tabanlı bir `receiver.php` dosyası hazırlandı.
* **`receiver.php` Mantığı:** `php://input` üzerinden gelen POST verilerini yakalayıp `data.txt` dosyasına kaydeder.

---

## 🏁 4. Alınan Kritik Dersler (Key Takeaways)

1. **Görünmezlik:** CORS zafiyetleri istemci taraflı (Client-Side) olduğu için sunucu tarafındaki WAF sistemleri tarafından fark edilemez.
2. **Zincirleme Etkisi:** Basit bir XSS veya hatalı bir CORS kuralı tek başına yıkıcı olmayabilir, ancak birleştiklerinde tam yetkili hesap ele geçirmeye (Account Takeover) yol açabilir.
3. **Regex Tehlikesi:** Alan adı kontrollerinde Regex kullanımı yerine her zaman tam eşleşme (Exact Match) sağlayan Beyaz Listeler (Allowlist) tercih edilmelidir.
4. **Credential Hassasiyeti:** `Access-Control-Allow-Credentials: true` ayarı yapıldığında, `ACAO` başlığında asla `*` (Wildcard) kullanılamayacağı, tarayıcıların bu durumda veriyi blokladığı öğrenildi.
