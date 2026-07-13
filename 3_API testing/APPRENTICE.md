# PortSwigger Academy - Exploiting an API Endpoint Using Documentation

**Zorluk Derecesi:** Apprentice (Başlangıç)  
**Kategori:** API Testing  
**Amaç:** Gizlenmiş veya unutulmuş API dokümantasyonunu bularak hedef kullanıcıyı (`carlos`) sistemden silmek.

---

## 🎯 Labın Mantığı ve Öğrenilen Ders (Takeaway)
Geliştiriciler, API'lerin nasıl kullanılacağını (hangi parametreleri aldığını, hangi HTTP metotlarıyla çalıştığını) takım arkadaşlarına göstermek için **Swagger, OpenAPI** gibi araçlarla dokümantasyon oluştururlar. 

Bazen bu dokümantasyon sayfaları canlı (production) ortamda yayında unutulur veya `/api`, `/swagger`, `/v1/docs` gibi tahmin edilebilir dizinlerde açık bırakılır. Saldırgan olarak bu dokümantasyonu bulduğumuzda, API'nin tüm haritası ve gizli özellikleri (örneğin kullanıcı silme yetkisi gibi) elimize geçmiş olur.

---

## 🛠️ Çözüm Adımları (Step-by-Step Walkthrough)

### 1. Keşif ve Trafik Analizi
1. Uygulamaya verilen bilgilerle giriş yapıldı: `wiener:peter`.
2. Profil sayfasına gidilerek e-posta adresi güncellendi. Bu işlem, arka planda API ile nasıl iletişim kurulduğunu tetiklemek için yapıldı.
3. **Burp Suite > Proxy > HTTP history** sekmesinden profil güncellenirken giden istek yakalandı:
   * **Metot:** `PATCH`
   * **Endpoint:** `/api/user/wiener`

### 2. API Endpoint Analizi ve Dizin Değiştirme (Fuzzing)
Yakaladığımız istek **Burp Repeater**'a gönderildi (`Ctrl + R`) ve adım adım geriye doğru dizinler kurcalandı:

* **Deneme 1:** `/api/user/wiener` -> `wiener` kullanıcısının bilgilerini döndürüyor.
* **Deneme 2:** `/api/user` -> Kullanıcı adı belirtilmediği için hata döndü.
* **Deneme 3:** En kök API dizinine inildi: **`/api`** 

> **Sonuç:** `/api` dizinine istek atıldığında, uygulamanın tüm API dokümantasyonunun (metotlar, endpointler ve parametreler) JSON veya HTML formatında geri döndüğü tespit edildi.

### 3. Sömürü (Exploit) Aşaması
1. Dönen dokümantasyon içeriği Burp'te sağ tıklanıp **"Show response in browser"** denilerek tarayıcıda açıldı.
2. Dokümantasyon incelendiğinde, kullanıcı silmeye yarayan gizli bir API ucu olduğu görüldü:
   * **Metot:** `DELETE`
   * **Endpoint:** `/api/user/{username}`
3. İnteraktif dokümantasyon alanı veya doğrudan Repeater kullanılarak ilgili endpoint'e `carlos` parametresi gönderildi:
   * **İstek:** `DELETE /api/user/carlos`

Kullanıcı başarıyla silindi ve lab tamamlandı.

---

## 🛡️ Güvenli Kodlama Notu (Remediation)
* **Dokümantasyon Erişimi:** API dokümantasyonları canlı (production) ortamlarda dış dünyaya açık olmamalıdır. IP kısıtlaması (whitelist) veya güçlü bir kimlik doğrulama (auth) arkasına alınmalıdır.
* **Yetki Kontrolü (BOLA/IDOR):** Bir kullanıcı `DELETE` isteği atarak başka bir kullanıcıyı silememelidir. Sunucu tarafında isteği atan kişinin (`wiener`) bu işlemi yapmaya yetkili olup olmadığı (`RBAC - Role Based Access Control`) her istekte kontrol edilmelidir.
