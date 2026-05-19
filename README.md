# SynapCart Core
*Yapay Zeka Destekli Akıllı Alışveriş Asistanı*

---

## Proje Hakkında
SynapCart, kullanıcıların alışveriş deneyimini kişiselleştiren, ürün fiyat karşılaştırmaları yapan ve kullanıcı tercihlerine göre akıllı öneriler sunan yapay zeka destekli bir alışveriş asistanıdır. Proje, ölçeklenebilir ve modüler bir mimari üzerine inşa edilmiştir.

## Mimari Yapı
Projemiz üç ana bileşenden oluşmaktadır:

* **Mobil Uygulama (mobile/):** Jetpack Compose ve Kotlin kullanılarak geliştirilmiş modern Android arayüzü.
* **Backend (backend/):** LLM (Large Language Model) ajanları ile entegre, yüksek performanslı FastAPI altyapısı.
* **Web Portalı (web/):** Next.js tabanlı, hesap yönetimi ve şifre sıfırlama süreçleri için geliştirilmiş arayüz.

## Canlı Servisler
* **Backend Instance:** [SynapCart API](https://synapcart-backend.onrender.com)
* **Web Portal:** [SynapCart Web](https://synapcart-web.vercel.app)

## Kurulum
Projenin tüm modüllerini tek seferde çekmek için:
```bash
git submodule update --init --recursive
```
