# SynapCart Core
*Yapay Zeka Destekli Akıllı Alışveriş Asistanı*

---

## Proje Hakkında
SynapCart, kullanıcıların alışveriş deneyimini kişiselleştiren, ürün fiyat karşılaştırmaları yapan ve kullanıcı tercihlerine göre akıllı öneriler sunan yapay zeka destekli bir alışveriş asistanıdır. Proje, ölçeklenebilir ve modüler bir mimari üzerine inşa edilmiştir.

Uygulamayı hemen denemek isterseniz, [Releases sekmesinden güncel debug.apk dosyasını](https://github.com/emreaydindev/synapcart_mobile/releases/tag/v1.0.0_Debug) indirerek cihazınıza kurabilirsiniz. (Debug dosyası Render.com ile çalışan bir backend instance'ına sahip. Ücretsiz bir hosting servisi olduğundan request tepkileri uzun sürebiliyor size önerim backendi lokalde 8001 portu ile derleyip android projesinde /util/Constants.kt dosyasında backend urlsine http://localhost:8001/ yazarak çalıştırmanızdır.)
Ayrıca projemi çalıştırdığım örnek bir video olarak bu linki ziyaret edebilirsiniz: [Youtube Videosu.](https://www.youtube.com/watch?v=l9S_6qSSHvs)

## Mimari Yapı
Projemiz üç ana bileşenden oluşmaktadır:

* **Mobil Uygulama (mobile/):** Jetpack Compose ve Kotlin kullanılarak geliştirilmiş modern Android arayüzü.
* **Backend (backend/):** LLM (Large Language Model) ajanları ile entegre, yüksek performanslı FastAPI altyapısı.
* **Web Portalı (web/):** Next.js tabanlı, hesap yönetimi ve şifre sıfırlama süreçleri için geliştirilmiş arayüz.

## Canlı Servisler
* **Backend Instance:** [SynapCart API](https://synapcart-backend.onrender.com)
* **Web Portal:** [SynapCart Web](https://synapcart-web.vercel.app)
Not: Canlı servisler ücretsiz hosting hizmetleri (Render/Vercel) üzerinde barındırıldığından, ilk açılışta veya uzun süreli boşta kalmalarda yanıt süresi uzayabilir.

## Kurulum
Projenin tüm modüllerini tek seferde çekmek için:
```bash
git submodule update --init --recursive
```
