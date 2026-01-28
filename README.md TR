# 🏠 Smart Curtain Control (HomeKit & ESP32-C3)

This project transforms a standard roller blind into a smart curtain using an **ESP32-C3** and a **28BYJ-48 stepper motor**. Powered by the **HomeSpan** library, it works natively with Apple Home (HomeKit) without requiring any additional hubs or bridges.

---

## 🌟 Key Features

* **Native HomeKit Support:** Seamless control via Siri or the Apple Home app.
* **Tap-to-Run:** A single short press on physical buttons fully opens or closes the curtain.
* **Hold-to-Move:** Press and hold for precise manual positioning (Release-to-stop).
* **Calibration Mode:** Easily set the "Home" (0%) position with a 3-second dual-button long press.
* **Persistence:** Saves current position and calibration data to NVS (flash memory), remembering its state after power cycles.
* **Smooth Movement:** Uses a half-step driving sequence for quieter and more precise operation.

---

## 🛠 Hardware & Pinout

The project is built on the **ESP32-C3 Mini** module. The pin connections are as follows:

| Component | Pin (ESP32-C3 Mini) |
| :--- | :--- |
| **Stepper IN1** | GPIO 3 |
| **Stepper IN2** | GPIO 2 |
| **Stepper IN3** | GPIO 1 |
| **Stepper IN4** | GPIO 0 |
| **Open Button** | GPIO 20 (Active-Low / PULLUP) |
| **Close Button** | GPIO 21 (Active-Low / PULLUP) |
| **Status/Calib LED** | GPIO 8 (Active-Low) |

---

## 🚀 Usage & Calibration

### Physical Buttons
* **Short Tap:** Starts moving until the end-point (0% or 100%). Tap the same button again to stop.
* **Long Hold:** Moves as long as the button is held down.
* **3s Dual-Press:** Toggles **Calibration Mode** ON/OFF.

### How to Calibrate?
1. Enter **Calibration Mode** (Hold both buttons for 3 seconds). The LED will turn ON.
2. Use the buttons to move the curtain to your desired **"Fully Open" (Home - 0%)** position.
3. Hold both buttons for 3 seconds again to save and exit. This point is now set as 0%.

---

## 💻 Installation

1. Ensure you have the **ESP32 Core** installed in your **Arduino IDE**.
2. Install the **HomeSpan (v2.1.3+)** library via the Library Manager.
3. Adjust the `STEPS_PER_METER` value in the code to match your curtain's length.
4. Upload the code to your ESP32-C3.
5. Open the Serial Monitor and pair the device with Apple Home using the code: **`538-27-491`**.

---

## ⚖️ License

This project is licensed under the **Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)**.

* **BY (Attribution):** Must give credit to Selim Burak Kul.
* **NC (Non-Commercial):** You may not use this material for commercial purposes or sales.
* **SA (ShareAlike):** If you remix or build upon the material, you must distribute your contributions under the same license.

---

 Selim Burak Kul - 2025
