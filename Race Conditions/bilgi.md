# 🏎️ Race Conditions (Yarış Durumları) - Teknik Özet

## 🔍 Kavramlar ve Temel Mantık
**Race Condition (Yarış Durumu)**, bir uygulamanın birden fazla işlemi veya isteği (thread/process) aynı anda (eşzamanlı olarak) işlemeye çalışırken, işlemlerin sırasının ve zamanlamasının beklenmedik mantık hatalarına yol açması durumudur.

En meşhur ve yaygın türü **TOCTOU (Time of Check to Time of Use - Kontrol Zamanı ile Kullanım Zamanı Arasındaki Fark)** zafiyetidir.

* **Nasıl Çalışır?:** Sunucu önce bir koşulu kontrol eder (Check: "Kullanıcının bakiyesi yeterli mi?"), sonra işlemi uygular (Use: "Bakiyeyi düş ve parayı gönder"). Eğer saldırgan, sunucu bakiyeyi düşmeden hemen önceki o milisaniyelik boşlukta aynı isteği defalarca gönderirse, sunucu her seferinde "Evet bakiye yeterli" deyip işlemi onaylar.

---

## 💣 Sık Karşılaşılan Saldırı Senaryoları (Limit Overruns)

1. **Finansal İşlemler (Double Spending):**
   * Hesapta 100 TL var. Saldırgan aynı milisaniyede 100 TL'lik iki transfer isteği atar. Sistem ikisini de kontrol eder, ikisinde de 100 TL olduğunu görür ve işlemi yapar. Bakiye -100 TL'ye düşer, saldırgan havadan 100 TL yaratmış olur.
2. **Kupon ve Promosyon Suistimali:**
   * "Tek kullanımlık" %50 indirim kuponunu aynı anda 5 farklı iş parçacığı (thread) ile sepete uygulamak. Sepet tutarı 5 kez %50 düşer.
3. **Oylama/Beğeni Hileleri:**
   * Bir kullanıcıya sadece 1 oy verme hakkı varken, eşzamanlı isteklerle bu limiti aşıp 10 oy birden göndermek.
4. **Kimlik Doğrulama (Authentication):**
   * Şifre sıfırlama veya hesap doğrulama kodlarını eşzamanlı göndererek token çakışmaları (collision) yaratmak.

---

## 🛠️ Sömürü (Exploitation) Taktikleri ve Araçlar
Bir Race Condition zafiyetini tetiklemek için isteklerin ağ (network) gecikmesine takılmadan, sunucuya **tam olarak aynı anda** ulaşması gerekir.

* **Burp Suite - Turbo Intruder:** Race Condition sömürüsü için altın standarttır. Geleneksel Intruder'dan çok daha hızlıdır. Özel Python betikleriyle milyonlarca isteği saniyeler içinde gönderir.
* **Burp Suite - Repeater (Grup Gönderimi):** Yeni Burp Suite güncellemelerinde birden fazla Repeater sekmesini gruplayıp, "Send group in parallel (single packet attack)" seçeneği ile tüm istekleri tek bir ağ paketi içinde eşzamanlı gönderebilirsiniz.
* **Curl (Parallel):** Terminal üzerinden paralel istek atmak için.
  * `curl -Z "http://site.com/buy" "http://site.com/buy" "http://site.com/buy"`
* **Python - ThreadPoolExecutor:** `requests` kütüphanesi ve `concurrent.futures` kullanılarak aynı anda 50-100 thread (iş parçacığı) açıp hedefe saldırmak.

---

## 🛡️ Savunma ve Korunma Yöntemleri (Mitigation)
Yazılımcıların yarış durumlarını engellemek için kod mimarisini değiştirmesi gerekir:

1. **Veritabanı Kilitleri (Database Locks):**
   * **Pessimistic Locking:** İşlem yapılan satırı (row) işlem bitene kadar diğer okuma/yazma işlemlerine kapatmak (Örn: SQL'de `SELECT * FROM users WHERE id=1 FOR UPDATE;`).
   * **Optimistic Locking:** Tabloya bir `version` sütunu eklemek. İşlem bitiminde sürüm değişmişse işlemi iptal etmek.
2. **Atomik İşlemler (Atomic Operations):**
   * Kontrol ve güncelleme işlemini tek bir veritabanı sorgusunda birleştirmek.
   * *Yanlış:* Önce bakiyeyi PHP'de çek, kontrol et, sonra güncelle.
   * *Doğru:* `UPDATE accounts SET balance = balance - 100 WHERE id = 1 AND balance >= 100;`
3. **Kuyruk Sistemleri (Message Queues):**
   * İşlemleri aynı anda işlemek yerine (RabbitMQ, Celery, Redis gibi) bir kuyruğa alıp sırayla (FIFO - First In First Out) işlemek.
4. **Mutex / Semaphore:** Kod seviyesinde kritik işlem bloklarına (Critical Section) aynı anda sadece tek bir thread'in girmesine izin vermek.
