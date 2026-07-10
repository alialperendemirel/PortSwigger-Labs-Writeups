# PortSwigger Lab: Password Reset Broken Logic — Missing current-password Check

## Zafiyet Özeti
`POST /my-account/change-password` endpoint'i, `current-password` parametresi hiç gönderilmediğinde bile şifre değişikliğine izin veriyor. Ayrıca şifresi değiştirilecek kullanıcı, sunucu tarafında oturum bilgisinden değil, **istemciden gelen `username` parametresinden** belirleniyor. Bu iki hata birleşince, herhangi bir kullanıcı adı için şifre resetlemek mümkün hale geliyor.

## Adımlar

### 1. Kayıtlı İstek İncelemesi
- Giriş yapılır, **My Account** sayfasından şifre değiştirilir.
- Burp'te oluşan `POST /my-account/change-password` isteği **Repeater**'a gönderilir.

### 2. current-password Kontrolünün Atlatılması
- İstekten `current-password` parametresi **tamamen çıkarılır**.
- İstek tekrar gönderilir → sunucu, mevcut şifre doğrulaması yapmadan şifre değişikliğini kabul eder.

### 3. username Parametresinin İstismarı
- İstekteki `username` parametresi `administrator` olarak değiştirilir.
- İstek gönderilir → sunucu, hangi hesabın şifresinin değiştiği kontrolünü oturumdan değil, bu parametreden yaptığı için **admin hesabının şifresi** kendi belirlediğiniz değere ayarlanır.

### 4. Doğrulama ve Bitiriş
- Çıkış yapılır.
- `administrator` kullanıcı adı ve yeni belirlenen şifreyle giriş yapılır → başarılı.
- Admin paneline girilip `carlos` kullanıcısı silinerek lab tamamlanır.

## Kök Neden
- **Eksik yetkilendirme kontrolü:** Sunucu, işlemi yapan kullanıcının kimliğini oturumdan (session) değil, istemcinin gönderdiği `username` parametresinden alıyor.
- **Eksik doğrulama:** `current-password` parametresi zorunlu değil; sunucu bu alan gönderilmediğinde isteği yine de işliyor.

## Önlem
- Şifre değiştirilecek kullanıcı, **her zaman oturumdaki kimlikten** belirlenmeli; istemciden gelen `username` parametresine asla güvenilmemeli.
- `current-password` alanı **sunucu tarafında zorunlu** kılınmalı ve eksikse istek reddedilmeli.
