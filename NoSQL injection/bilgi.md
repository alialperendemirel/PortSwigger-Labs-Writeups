# TryHackMe: NoSQL Injection (MongoDB) Kısa Notlar ve Cheat Sheet

Bu belge, MongoDB gibi NoSQL veritabanlarında karşılaşılan enjeksiyon zafiyetlerini ve sömürü yöntemlerini özetler.

## 🧠 Temel Farklılıklar (SQL vs NoSQL)
NoSQL veritabanlarında SQL sorguları (`SELECT`, `WHERE`) veya tablolar yoktur. Veriler **JSON/BSON** formatında tutulur.
* **Tablo (Table)** = `Collection` (Koleksiyon)
* **Satır (Row)** = `Document` (Doküman)
* **Saldırı Mantığı:** Metin zincirini kırmak (`' OR 1=1`) yerine, dizilerin (array) içine NoSQL Operatörleri (`$ne`, `$regex` vb.) enjekte edilir.

---

## 💥 1. Authentication Bypass (Kimlik Doğrulama Atlama)
Kullanıcı adı ve şifre bilinmeden sisteme giriş yapma tekniğidir.

### `$ne` (Not Equal - Eşit Değil) Operatörü
Hedef sisteme "Şifresi boş (veya rastgele bir değer) OLMAYAN ilk kullanıcıyı getir" komutunu göndererek genellikle `admin` hesabına sızmayı sağlar.
* **Normal İstek:** `user=admin&pass=1234`
* **Zehirli Payload (Burp Suite):** ```http
  user[$ne]=gecersiz&pass[$ne]=gecersiz
