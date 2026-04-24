Harika bir laboratuvar daha bitmiş! Bu bölüm, siber güvenlikte sadece web mimarisini değil, bilgisayar bilimlerinin temellerini (hafıza ve veri tiplerini) de bilmenin ne kadar önemli olduğunu gösteren efsanevi bir örnektir.

Buna literatürde "Integer Overflow" (Tam Sayı Taşması) denir. GitHub repon için doğrudan kopyalayabileceğin, net ve profesyonel özetini aşağıda hazırladım:
Low-level logic flaw (Düşük Seviye Mantık Hatası / Integer Overflow)

    Zafiyetin Mantığı: Sunucu tarafında sepetin toplam tutarı hesaplanırken kullanılan veri tipinin kapasite sınırının aşılmasıdır. Arka planda toplam fiyatı tutmak için 32-bit imzalı tam sayı (32-bit signed integer) kullanılmıştır. Bu veri tipinin alabileceği maksimum değer 2,147,483,647'dir (yaklaşık 2.1 milyar). Eğer sepetin toplam fiyatı bu sayıyı aşarsa, sistem bunu hesaplayamaz ve sayı bir anda en dip noktaya, yani devasa bir negatif değere (-2,147,483,648) sarar (Taşma/Overflow).

    Sömürü (Exploitation):

        Sepete tek seferde maksimum 99 adet ürün eklenebildiği tespit edilir.

        Burp Suite Intruder modülündeki Null Payloads (Sonsuz tekrar) özelliği kullanılarak, sepete sürekli 99 adet deri ceket (1337$) ekleme isteği gönderilir. (Sıranın bozulmaması için Resource pool > Maximum concurrent requests: 1 yapılmalıdır).

        Sepetin toplam fiyatı arka planda 32-bit sınırını aştığında aniden negatif bir değere düşer.

        Fiyatın eksi değere düştüğü noktada tam hesaplama yapılarak, sepetin toplam tutarını yönetilebilir bir eksi fiyata (örneğin -$1221) getirecek sayıda ürün eklenir (Örn: 323 kez 99 ceket + 1 kez 47 ceket).

        Sepetin nihai toplamı eksideyken, aradaki açığı kapatıp toplam fiyatı elimizdeki bakiye olan $0 - $100 arasına getirecek başka ucuz bir ürün eklenir. Sepet tutarı bakiyemizin altına düştüğü için sipariş başarıyla tamamlanır.

    Alınacak Ders (Mitigation): Finansal ve matematiksel işlemler tasarlanırken hesaplanacak olası maksimum değerler öngörülmeli ve daha büyük veri tipleri (örneğin 64-bit Long veya BigInteger) kullanılmalıdır. Ek olarak, uygulamanın koduna matematiksel taşmaları yakalayan (overflow validation) korumalar eklenmeli ve bir sepetin toplam tutarının kesinlikle 0'dan küçük olamayacağı (if totalPrice < 0) mantıksal olarak doğrulanmalıdır.
