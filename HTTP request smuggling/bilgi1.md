# 🛡️ HTTP/2 Request Smuggling - Technical Notes & Overview

**Zorluk Seviyesi:** Advanced / High  
**Araştırmacı:** James Kettle (PortSwigger)  
**Odaklanılan Zafiyet:** HTTP/2 Downgrade Attacks, Protocol Translation Flaws (Protokol Çeviri Hataları).

---

## 🌍 1. HTTP/2'nin Doğası ve Paradigma Değişimi

HTTP/1.1'in aksine, HTTP/2 **düz metin (plain text) tabanlı değil, ikili (binary) tabanlı** bir protokoldür. Veriler `\r\n` (CRLF) karakterleriyle sınırlandırılmaz; bunun yerine matematiksel olarak boyutları kesin belirlenmiş **Çerçeveler (Frames)** halinde iletilir. 

Bu temel mimari farkı nedeniyle, HTTP/1.1'de kullanılan geleneksel `Content-Length` (CL) ve `Transfer-Encoding` (TE) boyut manipülasyonları doğrudan HTTP/2 üzerinde çalışmaz. Ancak asıl tehlike, sistemlerin entegrasyon noktasında başlar.

## 💥 2. Zafiyetin Kaynağı: "Sürüm Düşürme" (Downgrade) İşlemi

Modern web mimarilerinde genellikle hibrit bir yapı kullanılır:
* **Ön Sunucu (Load Balancer / Reverse Proxy):** Hız ve performans için kullanıcı ile **HTTP/2** üzerinden haberleşir.
* **Arka Sunucu (Back-end):** Eski altyapıları ve uygulamaları desteklemek için genellikle hala **HTTP/1.1** kullanır.

**Zafiyetin Doğuşu:** Ön sunucu, kullanıcıdan aldığı HTTP/2 çerçevelerini açar, okur ve arka sunucuya iletmek için **HTTP/1.1 formatına çevirir (Downgrade).** Eğer ön sunucu, bu çeviri işlemi sırasında HTTP/2'nin esnek kurallarını HTTP/1.1'in katı kurallarına düzgün bir şekilde filtreleyerek aktarmazsa, *HTTP İstek Kaçakçılığı (Request Smuggling)* zafiyeti tetiklenir.

---

## 🥷 3. Temel Sömürü Teknikleri (Attack Vectors)

### A. H2.CL ve H2.TE Saldırıları (Eski Zehir, Yeni Şişe)
Ön sunucu HTTP/2 kullandığı için `Content-Length` veya `Transfer-Encoding` gibi boyut başlıklarına ihtiyaç duymaz ve çoğunlukla bunları filtrelemez. 
* Saldırgan, bu HTTP/1.1 başlıklarını HTTP/2 isteğinin içine gizlice yerleştirir.
* Ön sunucu bu başlıkları HTTP/1.1 formatına çevirip arka sunucuya ilettiğinde, arka sunucu bu sahte başlıkları gerçek sanır.
* Sonuç olarak, klasik **CL.TE** veya **TE.CL** boru hattı (pipeline) zehirlenmesi arka sunucuda başarılı bir şekilde gerçekleşir.

### B. CRLF Injection (İstek Bölme - Request Splitting)
HTTP/1.1'de bir başlığın (header) içine satır atlama karakteri olan `\r\n` yazarsanız, sunucu yapıyı bozduğunuz için anında "400 Bad Request" hatası verir. Ancak **HTTP/2 binary olduğu için başlıkların içindeki `\r\n` karakterlerini sadece sıradan bir veri olarak kabul eder.**
* Saldırgan, bir HTTP/2 başlığının değerine `\r\n` karakterleri enjekte ederek içine tamamen yeni bir HTTP isteği gizler.
* Ön sunucu bunu HTTP/1.1'e çevirdiğinde, arka sunucu o `\r\n` karakterlerini bir satır atlama komutu olarak okur. İsteğin tam ortadan ikiye bölündüğünü sanarak içerideki kaçak isteği (Smuggled Request) çalıştırır.

### C. Sahte Başlık (Pseudo-Headers) Manipülasyonu
HTTP/2'de HTTP metodu (`GET`, `POST`) veya yol (`/admin`) gibi temel bilgiler normal satırlarda değil; `:method`, `:path`, `:authority` gibi özel "pseudo-headers" içinde taşınır.
* Ön sunucunun güvenlik duvarı (WAF) kurallarını atlatmak için bu başlıklar manipüle edilebilir.
* Örneğin, `:path` başlığına normal bir dizin yazılıp, arkasından gelen çeviri (downgrade) hatalarıyla arka sunucunun yetki gerektiren gizli bir API noktasına erişmesi (Bypass) sağlanabilir.

---

## 🎯 4. Etki ve Sonuç (Impact)

HTTP/2 Request Smuggling, hedef sistemdeki WAF ve Ön Sunucu güvenlik kurallarını tamamen kör edebilir. Başarıyla sömürüldüğünde şu yıkıcı sonuçlara yol açar:
1. Masum kullanıcıların hesaplarının ve oturumlarının çalınması (Session Hijacking / Credential Capture).
2. Sitenin önbelleğinin zehirlenerek tüm ziyaretçilere zararlı içerik sunulması (Cache Poisoning).
3. Güvenlik kontrollerinin atlatılarak yetkisiz yönetici (Admin) işlemlerinin yapılması (Privilege Escalation & WAF Bypass).
