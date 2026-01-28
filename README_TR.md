# 🏠 Smart Curtain Control (HomeKit & ESP32-C3)

Bu proje, bir **ESP32-C3** ve **28BYJ-48 step motor** kullanarak standart bir stor perdeyi akıllı hale getirir. **HomeSpan** kütüphanesi sayesinde herhangi bir köprü (bridge) cihazına ihtiyaç duymadan doğrudan Apple Home (HomeKit) ile çalışır.



## 🌟 Öne Çıkan Özellikler (Key Features)

* **Yerel HomeKit Desteği:** Siri veya Ev uygulaması üzerinden doğrudan kontrol.
* **Tap-to-Run:** Fiziksel butonlara tek bir kısa dokunuşla perdeyi tamamen açar veya kapatır.
* **Hold-to-Move:** Hassas konumlandırma için butona basılı tuttuğunuz sürece hareket eder.
* **Kalibrasyon Modu:** 3 saniyelik çift buton basışıyla "Home" (0%) konumunu kolayca ayarlar.
* **Hafıza (Persistence):** Elektrik kesilse bile mevcut konumunu ve kalibrasyon verilerini NVS (flash) üzerinde saklar.
* **Sessiz ve Hassas:** Half-step (yarım adım) sürüş tekniği ile daha sarsıntısız hareket.

---

## 🛠 Donanım ve Pin Bağlantıları (Hardware)

| Bileşen (Component) | Pin (ESP32-C3 Mini) |
| :--- | :--- |
| **Stepper IN1** | GPIO 3 |
| **Stepper IN2** | GPIO 2 |
| **Stepper IN3** | GPIO 1 |
| **Stepper IN4** | GPIO 0 |
| **Open Button** | GPIO 20 (GND'ye çekili) |
| **Close Button** | GPIO 21 (GND'ye çekili) |
| **Status/Calib LED** | GPIO 8 (Active-Low) |

---

## 🚀 Kullanım ve Kalibrasyon (Usage & Calibration)

### Fiziksel Butonlar
* **Kısa Dokunuş:** Uç noktaya kadar otomatik sürüşü başlatır. Durdurmak için tekrar dokunun.
* **Basılı Tutma:** Elinizi çekene kadar hareket devam eder.
* **3sn Çift Basış:** **Kalibrasyon Modu**'nu açar/kapatır.

### Nasıl Kalibre Edilir?
1.  Her iki butona 3 saniye basılı tutarak Kalibrasyon Moduna girin (LED yanacaktır).
2.  Butonları kullanarak perdeyi tam açık (Home - 0%) olmasını istediğiniz noktaya getirin.
3.  Tekrar her iki butona 3 saniye basılı tutarak kaydedin ve çıkın. Bu nokta artık 0 noktasıdır.

---

## 💻 Kurulum (Installation)

1.  **Arduino IDE** ve **ESP32 Core** yüklü olduğundan emin olun.
2.  **HomeSpan (v2.1.3+)** kütüphanesini kütüphane yöneticisinden kurun.
3.  Kod içerisindeki `STEPS_PER_METER` değerini kendi perdenizin boyuna göre güncelleyin.
4.  Kodu ESP32-C3 cihazınıza yükleyin.
5.  Seri Monitörü (115200 baud) açın ve `?` yazarak HomeKit kurulum kodunu veya QR kodu alın.
6.  Apple Ev uygulamasından şu kodla cihazı ekleyin: **`538-27-491`**

---

## ⚖️ Lisans (License)

Bu proje **Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)** ile lisanslanmıştır.

* **BY (Atıf):** Selim Burak Kul ismi belirtilmelidir.
* **NC (Gayri-Ticari):** Bu proje ticari amaçlarla kullanılamaz, satılamaz.
* **SA (Aynı Lisansla Paylaş):** Bu proje üzerinde yapılan geliştirmeler yine aynı lisansla paylaşılmalıdır.

---

Selim Burak Kul - 2025
