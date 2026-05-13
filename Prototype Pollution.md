# 🧬 Prototype Pollution (Prototip Kirliliği) - Teknik Özet

## 🔍 Kavramlar ve Temel Mantık
**Prototype Pollution**, JavaScript'in esnek "Prototip Tabanlı Kalıtım" (Prototype-based Inheritance) modelindeki mantık hatalarını istismar eden bir zafiyettir. Diğer dillerin aksine JS'de sınıflar (classes) statik değildir; nesneler özelliklerini atalarından (prototiplerinden) çalışma zamanında (runtime) miras alırlar.

* **Prototip Zinciri:** JS'de bir nesnede olmayan bir özellik çağrıldığında, sistem `__proto__` bağını kullanarak o nesnenin atasına (prototipine) bakar. En tepede ulu ata olan `Object.prototype` bulunur.
* **Zafiyetin Temeli:** Saldırgan, `__proto__` veya `constructor.prototype` parametrelerini kullanarak en tepedeki `Object.prototype`'a dışarıdan yeni bir özellik (property) enjekte ederse, sistemdeki **istisnasız tüm nesneler** bu özelliği anında miras alır (kirlenir).

---

## 💣 Zafiyetin Kaynağı (Vulnerable Functions)
Prototip kirliliği genellikle yazılımcıların dışarıdan gelen (user-controlled) JSON verilerini güvenli olmayan şekilde işlemesiyle ortaya çıkar.

1. **Property Definition by Path (Yola Göre Atama):**
   * Lodash gibi kütüphanelerin eski sürümlerindeki `_.set(obj, path, value)` fonksiyonları.
   * *Tehlike:* Yol (path) olarak `__proto__.isAdmin` gönderildiğinde kontrolsüzce ataya yazılması.
2. **Object Recursive Merge (İç İçe Birleştirme):**
   * İki objeyi birleştiren (eski ayarlar + yeni ayarlar) özel `merge()` fonksiyonlarında anahtar kelimelerin (keys) filtrelenmemesi.
3. **Object Clone (Nesne Kopyalama):**
   * Bir nesneyi derinlemesine (deep clone) kopyalarken `__proto__`'nun da geçerli bir veri gibi işlenip kopyalanması.

---

## 🛠️ Sömürü (Exploitation) Taktikleri ve Payloadlar
Prototip kirliliği tek başına sadece bir "hafıza manipülasyonudur". Etkili olması için uygulamanın mantığına (Business Logic), XSS'e veya RCE'ye bağlanması (Chaining) gerekir.

### 1. Yetki Yükseltme (Privilege Escalation)
Sistemin `isAdmin` veya `role` gibi özelliklerin varsayılan değerlerini kontrol etmediği durumlarda tüm kullanıcılara yetki vermek için kullanılır.
``json
// Payload Örneği (Merge veya JSON Parse işlemlerine yedirilir)
{"__proto__": {"isAdmin": true}}
// veya
{"constructor": {"prototype": {"isAdmin": true}}}

2. Client-Side (DOM) XSS

Tarayıcı tarafında çalışan JS kodlarını zehirleyerek, sayfayı ziyaret eden herkesin tarayıcısında zararlı script çalıştırmak.
JSON

{"__proto__": {"content": "<script>alert('Hacked')</script>"}}

3. Denial of Service (DoS) - Çökertme

Tüm nesnelerin kullandığı yerleşik ana fonksiyonları (örneğin toString) bozarak/ezerek Node.js sunucusunu Type Error ile çökertmek.
JSON

{"__proto__": {"toString": "Crash"}}
// Sistem obj.toString() yapmaya çalıştığında fonksiyon yerine string görecek ve çökecektir.

🛡️ Savunma ve Korunma Yöntemleri (Mitigation)

Bir geliştirici (Developer) olarak bu zafiyetleri önlemek için kod mimarisinde şu adımlar atılmalıdır:

    Atasız Nesneler Yaratmak:

        Nesne oluştururken let obj = {} yerine let obj = Object.create(null); kullanın. Bu, __proto__ bağı olmayan "yetim" bir nesne yaratır, dolayısıyla kirlenemez.

    Prototipi Dondurmak (Immutable Object):

        Uygulama başlarken Object.freeze(Object.prototype); kodunu çalıştırın. Bu, ulu atanın genetiğiyle oynanmasını tamamen engeller.

    Girdi Filtreleme (Input Sanitisation & Validation):

        Dışarıdan gelen verilerde __proto__, constructor ve prototype anahtarlarını (keys) kesinlikle kara listeye alın (drop/delete).

    Bağımlılık Yönetimi (Dependency Management):

        Zafiyet genelde sizin kodunuzda değil, kullandığınız kütüphanelerdedir (lodash, hoek vb.). npm audit ile paketleri sürekli tarayın ve güncelleyin.

    Daha Güvenli Alternatifler Kullanmak:

        __proto__ yerine her zaman Object.getPrototypeOf() kullanın.

        Anahtar-değer (Key-Value) eşleşmeleri için standart Object yerine JavaScript'in yerleşik Map sınıfını kullanmayı tercih edin (Map'ler bu tür kirliliklere karşı dirençlidir).




Burp Suite'in içindeki kendi tarayıcısını (Burp Browser) açtığında sağ üstte DOM Invader adında bir eklenti göreceksin. Bu araç, PortSwigger mühendisleri tarafından özel olarak Prototype Pollution bulmak için geliştirilmiştir.

    DOM Invader'ı aktif et.

    Ayarlarından "Prototype Pollution" tespitini aç.

    Sayfada gezinirken araç sana "Hey, burada bir obje kopyalama hatası var, payload yollayayım mı?" diyecektir.

        
