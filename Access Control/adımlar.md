# İleri Seviye API Zafiyet Test Rehberi (IDOR & Business Logic)

Bu doküman, istek gövdesinde (body) açıkça `user_id` gibi kimlikleyiciler barındırmayan ve yetkilendirmeyi oturum (Cookie/Token) üzerinden yapan karmaşık API uç noktalarında uygulanacak test metodolojilerini içerir.

**Hedef Uç Nokta Tipi:** `POST /api/services/...` gibi JSON gövdesi alan uç noktalar.
**Kullanılacak Araçlar:** Burp Suite (Repeater & Intruder)

---

## 1. Gizli veya Eksik Parametre Ekleme (Parameter Addition & Mass Assignment)

**Mantığı:** Geliştiriciler veritabanı modellerini doğrudan API ile eşleştirmiş olabilir. İstek içerisinde bulunmayan ancak veritabanında var olan bir kimlik parametresi dışarıdan eklendiğinde, sistem oturum bilgisi yerine bu parametreyi işleme alabilir.

**Test Yöntemi:**
JSON objesinin ana dizinine veya iç objelere kurbana ait kimlikleyiciler (ID) eklenmelidir.

**Örnek Payload:**
`json
{
  "user_id": "KURBAN_ID",
  "owner_id": "KURBAN_ID",
  "id": "KURBAN_HEDEF_ID",
  "item": {
    "valid_from": "2026-05-05",
    "default_goal": { ... }
  }
}
Beklenen Zafiyet (Etki): Sistem gönderilen user_id değerini kabul ederse, yetki yükseltme veya veri manipülasyonu (IDOR) gerçekleşir.








2. Dizi (Array) ve Alt Obje Manipülasyonu

Mantığı: İstek gövdesinde gönderilen diziler (array) içindeki her bir obje için yetki kontrolü atlanmış veya sadece dizinin genel yapısı için kontrol yapılmış olabilir.

Test Yöntemi:
Mevcut JSON yapısındaki liste elemanlarına kurbana ait bir id değeri enjekte edilmelidir.

Orijinal Yapı:
"daily_goals": [
  { "day_of_week": "monday", "energy": {"value": "1300"} }
]
Saldırı (Exploit) Yapısı:
"daily_goals": [
  { "id": "KURBAN_HEDEF_ID", "day_of_week": "monday", "energy": {"value": "1300"} }
]
Beklenen Zafiyet (Etki): Yetkisiz bir şekilde başka bir kullanıcıya ait spesifik bir kaydın (örneğin hedeflenen günün kalori limitinin) değiştirilebilmesi.







3. İş Mantığı Zafiyetleri (Business Logic Flaws)

Mantığı: Uygulamanın gelen girdileri mantıksal olarak nasıl işlediği test edilmelidir. Doğrulama (validation) eksiklikleri sistemin çökmesine veya belirlenen kuralların dışına çıkılmasına neden olabilir.

Test Yöntemi:
Parametre değerleri anormal veri tipleri ve sınır değerleri ile değiştirilmelidir.

    Negatif Değer Testi: "energy": {"value":"-5000"} (Sistemin matematiksel hesaplamaları bozması hedeflenir).

    Integer Overflow (Sınır Aşımı): "carbohydrates": 99999999999999999 (Veritabanı sınırı aşılarak hata veya sıfırlanma aranır).

    Veri Tipi Uyuşmazlığı (Type Juggling): "protein": "admin" veya "protein": true (Sistemin farklı veri tiplerine karşı hata verip vermediği kontrol edilir).

Beklenen Zafiyet (Etki): Arka plan (backend) hata loglarının sızması, hatalı hesaplamalar yaptırılması veya mantıksal limitlerin aşılması.









4. HTTP Metot Değiştirme (Verb Tampering)

Mantığı: API üzerindeki yetkilendirme (Authorization) mekanizması sadece belirli HTTP metotları (örneğin sadece POST) için yapılandırılmış, diğerleri unutulmuş veya varsayılan ayarlarda bırakılmış olabilir.

Test Yöntemi:
Burp Suite Repeater sekmesinde orijinal istek metodu farklı metotlarla değiştirilerek sunucu yanıtı incelenmelidir.

    PUT: Kaydı sıfırdan oluşturmak yerine zorla güncellemeyi denemek.

    PATCH: Tüm gövdeyi göndermek yerine sadece tek bir alanı (örneğin sadece kaloriyi) değiştirmeyi denemek.

    GET: Başka bir kullanıcının verilerini sadece URL veya ID ile okumayı denemek.

Beklenen Zafiyet (Etki): Yetkisiz işlemlerin 403 Forbidden veya 401 Unauthorized yerine 200 OK yanıtı alarak gerçekleşmesi.






5. Header Manipülasyonu ile Kimlik Değiştirme

Mantığı: Aracı sunucular (Load Balancers, Reverse Proxies), arka plandaki ana sunucuya ilettikleri özel HTTP başlıkları (headers) üzerinden kimlik doğrulaması yapıyor olabilir.

Test Yöntemi:
HTTP İstek Başlıklarına (Request Headers) sistemde tanınan özel parametreler eklenerek sistem manipüle edilmelidir.

Örnek Header Gönderimleri:
X-User-ID: KURBAN_ID
X-UID: KURBAN_ID
X-Account-ID: KURBAN_ID
X-Forwarded-For: 127.0.0.1
X-Original-URL: /api/services/nutrient-goals/KURBAN_ID
Beklenen Zafiyet (Etki): Sistemin oturum çerezi (Cookie) veya yetkilendirme tokeni yerine, dışarıdan eklenen başlığa (Header) güvenerek kurban hesabında işlem yapması.

Test Standartları ve Kuralları

    Hesap İzolasyonu: Tüm testler, kontrol hesabı (saldırgan) ve kurban hesabı (hedef) olmak üzere kişisel olarak oluşturulmuş iki farklı hesap arasında yapılmalıdır. Kesinlikle başka gerçek kullanıcı verilerine müdahale edilmemelidir.

    Referans Kimlik Toplama: Hedef hesaplara ait ID veya UUID değerleri platform içerisindeki topluluk forumları, ortak paylaşımlar veya profil bağlantılarından edilinmelidir.

    Hata Analizi (Stack Trace): Alınan 500 Internal Server Error yanıtları dikkatle incelenmeli; dönen hatalar üzerinden arka plan mimarisi, kullanılan veritabanı veya framework türü analiz edilmelidir.
