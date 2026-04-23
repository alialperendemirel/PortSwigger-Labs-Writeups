Broken brute-force protection, IP block (Kırık Kaba Kuvvet Koruması, IP Engeli)

    Zafiyetin Mantığı: IP tabanlı kaba kuvvet korumasındaki mantık hatası. Sistem 3 yanlış girişte IP'yi engelliyor, ancak o IP'den herhangi bir hesaba başarılı giriş yapıldığında hata sayacını (counter) tamamen sıfırlıyor.

    Sömürü (Exploitation): 1. Burp Suite Intruder modülü açılır ve saldırı tipi Pitchfork olarak seçilir.
    2. Zikzaklı bir Payload listesi hazırlanır. (Örnek: 1. İstek kendi geçerli hesabımız, 2. İstek kurbanın hesabına şifre denemesi, 3. İstek tekrar kendi geçerli hesabımız...)
    3. Kendi geçerli hesabımızla yaptığımız her giriş sistemi kandırarak IP ban sayacımızı sıfırlar. Böylece aralara sıkıştırdığımız kaba kuvvet (brute-force) denemeleri engellenmeden devam eder.
    4. Kritik Adım: İsteklerin sunucuya sırayla gitmesi için Intruder sekmesinden Resource pool -> Maximum concurrent requests: 1 ayarı kesinlikle yapılmalıdır. Aksi halde Burp istekleri aynı anda atar ve IP engeli yenilir.

    Alınacak Ders (Mitigation): Başarılı bir giriş yapıldığında IP'nin genel hata sayacı sıfırlanmamalıdır. Sayacı sıfırlama işlemi sadece giriş yapılan o spesifik kullanıcı hesabı (username) için geçerli olmalıdır. Ayrıca, tek bir IP'den farklı hesaplara sürekli giriş denemeleri yapılıyorsa bu da WAF tarafından tespit edilip engellenmelidir.
