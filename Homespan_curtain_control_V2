/*
 *  Selim Burak Kul 04.11.2025
 *  In this version, the ability to open and close the curtain with a single click on the buttons has been added.
 *  Smart Curtain (HomeSpan / Apple Home) — ESP32-C3 Mini — Arduino IDE
 *
 * NEW: Tap‑to‑Run for buttons
 *  - Short press (tap) OPEN → runs continuously to 0% (home)
 *  - Short press (tap) CLOSE → runs continuously to 100%
 *  - Tap again the same button to stop.
 *  - Press & hold still works as before (hold‑to‑move). Releasing stops.
 *  - Dual‑button 3 s long‑press still toggles Calibration Mode (unchanged).
 *  - HomeKit behavior unchanged.
 *
 * - Half-step motor drive, 2-button debounce, 3s dual-button calibration
 * - Normal mode: hold-to-move (press→move, release→stop) + Home control enabled
 * - 1 m travel mapped to 0..100%
 * - NVS (Preferences) persistence
 * - HomeKit WindowCovering: CurrentPosition, TargetPosition, PositionState
 *
 * Pins (ESP32-C3 Mini):
 *   ULN2003 IN1..IN4  ←→  MOTOR_PIN1..4  (GPIO 3,2,1,0)
 *   BUTTON_OPEN       : GPIO 21 (active-low to GND)
 *   BUTTON_CLOSE      : GPIO 20 (active-low to GND)
 *   LED_CALIBRATION   : GPIO 8  (active-low)
 *
 * Libs:
 *   - Arduino-ESP32 core
 *   - HomeSpan 2.1.3
 */

#include <Arduino.h>
#include <Preferences.h>
#include <HomeSpan.h>

#define TAG "SMART_CURTAIN"
#define LOGI(tag, fmt, ...)  do { Serial.printf("[I] %s: " fmt "\n", tag, ##__VA_ARGS__); } while(0)
#define LOGW(tag, fmt, ...)  do { Serial.printf("[W] %s: " fmt "\n", tag, ##__VA_ARGS__); } while(0)

// === GPIO ===
#define MOTOR_PIN1 3
#define MOTOR_PIN2 2
#define MOTOR_PIN3 1
#define MOTOR_PIN4 0
#define BUTTON_OPEN 20
#define BUTTON_CLOSE 21
#define LED_CALIBRATION 8  // active-low

// === Hız & Yön Ayarları ===
#define STEP_DELAY_MS 1            // ms tabanlı zamanlama için (kullanmazsan sorun değil)
#define USE_US_TIMING 1            // 1: micros() ile ince zamanlama, 0: millis()
#define STEP_DELAY_US 1200         // 1.2 ms ≈ hızlı ve genelde stabil
#define SEQUENCE_REVERSE 1         // 0: normal, 1: elektriksel adım sırasını ters çevir (yönü değiştirir)

// === Hareket Aralığı ===
#define STEPS_PER_METER 97000      // mekaniklerine göre kalibre et

// === Debounce / Kalibrasyon ===
#define DEBOUNCE_MS 20
#define LONGPRESS_MS 3000

// === Tap‑to‑Run (new) ===
#define TAP_MIN_MS 50               // min tap süresi (debounce üstü)
#define TAP_MAX_MS 500              // max tap süresi (kısa dokunuş)

// === Half-step sequence ===
static const uint8_t HALFSTEP[8][4] = {
  {1,0,0,0}, {1,1,0,0}, {0,1,0,0}, {0,1,1,0},
  {0,0,1,0}, {0,0,1,1}, {0,0,0,1}, {1,0,0,1}
};

// === State ===
static volatile int32_t current_pos = 0;     // steps
static volatile int32_t home_pos    = 0;     // steps
static volatile int32_t target_steps= 0;     // steps (clamped to span)
static volatile int8_t  move_dir    = 0;     // -1,0,+1
static volatile bool    calibration_mode = false;
static bool             manual_drive = false; // buton kaynaklı mı?

// Tap‑to‑Run bookkeeping
static bool     latch_active = false;   // tap ile başlatılan sürekli sürüş aktif mi?
static int8_t   latch_dir    = 0;       // -1 open/UP, +1 close/DOWN
static bool     open_prev = false, close_prev = false; // önceki debounce state
static uint32_t open_press_t = 0, close_press_t = 0;   // basılma zamanı

static uint8_t  step_index    = 0;
static uint32_t last_step_ms  = 0;

// Debounce bookkeeping
static int open_cnt = 0, close_cnt = 0;
static bool open_state = false, close_state = false;
static uint32_t both_press_start = 0;
static bool both_pressed_latched = false;

Preferences prefs;

// === Utils ===
static inline int32_t clamp_i32(int32_t v, int32_t lo, int32_t hi){ return v<lo?lo:(v>hi?hi:v); }
static inline uint8_t steps_to_percent(int32_t steps, int32_t home, int32_t span_steps) {
  if (span_steps<=0) return 0;
  int32_t num = (int64_t)(steps - home) * 100 / span_steps;
  if (num<0) num=0; if (num>100) num=100;
  return (uint8_t)num;
}
static inline int32_t percent_to_steps(uint8_t p, int32_t home, int32_t span){
  if (p>100) p=100;
  int32_t desired = home + ((int64_t)span * p) / 100;
  return clamp_i32(desired, home, home+span);
}

// === IO ===
static void motor_step(uint8_t idx){
  digitalWrite(MOTOR_PIN1, HALFSTEP[idx][0]);
  digitalWrite(MOTOR_PIN2, HALFSTEP[idx][1]);
  digitalWrite(MOTOR_PIN3, HALFSTEP[idx][2]);
  digitalWrite(MOTOR_PIN4, HALFSTEP[idx][3]);
}
static void motor_release(){
  digitalWrite(MOTOR_PIN1, LOW);
  digitalWrite(MOTOR_PIN2, LOW);
  digitalWrite(MOTOR_PIN3, LOW);
  digitalWrite(MOTOR_PIN4, LOW);
}
static void io_init(){
  pinMode(MOTOR_PIN1, OUTPUT);
  pinMode(MOTOR_PIN2, OUTPUT);
  pinMode(MOTOR_PIN3, OUTPUT);
  pinMode(MOTOR_PIN4, OUTPUT);
  pinMode(BUTTON_OPEN,  INPUT_PULLUP);
  pinMode(BUTTON_CLOSE, INPUT_PULLUP);
  pinMode(LED_CALIBRATION, OUTPUT);
  digitalWrite(LED_CALIBRATION, HIGH);  // off
  motor_release();
}

// active-low read + debounce @1ms
static inline bool raw_open_pressed(){  return digitalRead(BUTTON_OPEN)==LOW;  }
static inline bool raw_close_pressed(){ return digitalRead(BUTTON_CLOSE)==LOW; }
static void debounce_buttons(){
  if (raw_open_pressed()){  if (open_cnt  < DEBOUNCE_MS) open_cnt++; }  else { if (open_cnt  > 0) open_cnt--; }
  if (raw_close_pressed()){ if (close_cnt < DEBOUNCE_MS) close_cnt++; } else { if (close_cnt > 0) close_cnt--; }
  open_state  = (open_cnt  >= DEBOUNCE_MS/2);
  close_state = (close_cnt >= DEBOUNCE_MS/2);
}

// === HomeSpan WindowCovering ===
struct CurtainWC : Service::WindowCovering {
  Characteristic::CurrentPosition *current;     // 0..100
  Characteristic::TargetPosition  *target;      // 0..100
  Characteristic::PositionState   *state;       // 0=Decreasing,1=Increasing,2=Stopped
  Characteristic::ObstructionDetected *obst;

  CurtainWC() : Service::WindowCovering(){
    current = new Characteristic::CurrentPosition(0);
    target  = new Characteristic::TargetPosition(0);
    state   = new Characteristic::PositionState(2); // Stopped
    obst    = new Characteristic::ObstructionDetected(false);
    uint8_t p = steps_to_percent(current_pos, home_pos, STEPS_PER_METER);
    current->setVal(p);
    target->setVal(p);
  }

  boolean update() override {
    if (calibration_mode){
      LOGW(TAG, "Ignoring HomeKit target in calibration mode");
      return true;
    }
    uint8_t targetPercent = target->getNewVal();  // 0..100
    target_steps = percent_to_steps(targetPercent, home_pos, STEPS_PER_METER);
    if (target_steps > current_pos)      move_dir = +1;
    else if (target_steps < current_pos) move_dir = -1;
    else                                 move_dir = 0;

    target->setVal(steps_to_percent(target_steps, home_pos, STEPS_PER_METER));
    state->setVal(move_dir>0 ? 1 : (move_dir<0 ? 0 : 2));

    // HomeKit komutu gelince manuel/latch iptal
    manual_drive = false;
    latch_active = false;
    latch_dir = 0;
    return true;
  }

  void loop() override {
    if (move_dir==0)
      state->setVal(2);  // Stopped
  }
} *wc = nullptr;

// === Helpers: Home'a konum/state bildir ===
static void report_position_immediate(){
  if (!wc) return;
  uint8_t p = steps_to_percent(current_pos, home_pos, STEPS_PER_METER);
  wc->current->setVal(p);
  wc->state->setVal(move_dir>0 ? 1 : (move_dir<0 ? 0 : 2));
}
static void report_stop_sync(){
  if (!wc) return;
  uint8_t p = steps_to_percent(current_pos, home_pos, STEPS_PER_METER);
  wc->current->setVal(p);
  wc->target->setVal(p);
  wc->state->setVal(2);  // Stopped
}

// === Tap‑to‑Run helpers ===
static void latch_start(int8_t dir, int32_t target){
  latch_active = true;
  latch_dir = dir;
  manual_drive = false;        // latch manuel değil (tap tetik ama sürekli sürer)
  move_dir = dir;
  target_steps = target;
  LOGI(TAG, "Latch start dir=%d target=%ld", dir, (long)target);
}
static void latch_cancel_and_stop(){
  if (!latch_active) return;
  latch_active = false;
  latch_dir = 0;
  move_dir = 0;
  motor_release();
  report_stop_sync();
  LOGI(TAG, "Latch stop");
}

// === 1 ms application loop ===
static void app_loop_1ms(){
  const int32_t span = STEPS_PER_METER;

  debounce_buttons();
  uint32_t now_ms = millis();

  // edge detect (debounced)
  bool open_rise  = (!open_prev  && open_state);
  bool open_fall  = (open_prev   && !open_state);
  bool close_rise = (!close_prev && close_state);
  bool close_fall = (close_prev  && !close_state);

  if (open_rise)  open_press_t  = now_ms;
  if (close_rise) close_press_t = now_ms;

  // Dual-button: stop + toggle calibration after 3s
  if (open_state && close_state){
    // Any dual-press cancels latch instantly
    latch_cancel_and_stop();
    move_dir = 0;
    motor_release();

    if (!both_pressed_latched){
      if (both_press_start==0) both_press_start = now_ms;
      if (now_ms - both_press_start >= LONGPRESS_MS){
        calibration_mode = !calibration_mode;
        both_pressed_latched = true;
        digitalWrite(LED_CALIBRATION, calibration_mode ? LOW : HIGH);

        if (!calibration_mode){
          // exiting calibration: set home to current; clamp & persist
          home_pos = current_pos;
          int32_t lo = home_pos;
          int32_t hi = home_pos + span;
          current_pos  = clamp_i32(current_pos, lo, hi);
          target_steps = current_pos;
          move_dir = 0;
          manual_drive = false;
          latch_active = false;
          latch_dir = 0;

          prefs.putInt("home", (int)home_pos);
          prefs.putInt("pos",  (int)current_pos);

          report_stop_sync();
          LOGI(TAG, "Calibration exit: home=%ld", (long)home_pos);
        } else {
          LOGI(TAG, "Calibration enter");
        }
      }
    }
  } else {
    both_press_start = 0;
    both_pressed_latched = false;

    if (!calibration_mode){
      // === TAP‑TO‑RUN: tap detection on release ===
      if (open_fall){
        uint32_t dt = now_ms - open_press_t;
        if (dt>=TAP_MIN_MS && dt<=TAP_MAX_MS && !close_state){
          // toggle/open latch
          int32_t lo = home_pos;
          if (latch_active && latch_dir<0){
            latch_cancel_and_stop();
          } else {
            latch_start(-1, lo);
          }
        }
      }
      if (close_fall){
        uint32_t dt = now_ms - close_press_t;
        if (dt>=TAP_MIN_MS && dt<=TAP_MAX_MS && !open_state){
          // toggle/close latch
          int32_t hi = home_pos + span;
          if (latch_active && latch_dir>0){
            latch_cancel_and_stop();
          } else {
            latch_start(+1, hi);
          }
        }
      }

      // === NORMAL MODE: hold-to-move (unchanged) ===
      int32_t lo = home_pos, hi = home_pos + span;

      if (open_state ^ close_state){
        // Any active hold cancels latch immediately
        if (latch_active) latch_cancel_and_stop();

        manual_drive = true;
        if (open_state){
          if (current_pos > lo){ move_dir = -1; target_steps = lo; } else { move_dir = 0; }
        } else { // close_state
          if (current_pos < hi){ move_dir = +1; target_steps = hi; } else { move_dir = 0; }
        }
      } else if (!open_state && !close_state){
        // No button held: if last action was a manual hold, stop (unchanged behavior)
        if (manual_drive){
          move_dir = 0;
          manual_drive = false;
          motor_release();
          report_stop_sync();
        }
        // If latch_active, movement continues under latch control until target/endstop.
        // If HomeKit commanded, movement also continues as before.
      }

    } else {
      // CALIBRATION: free move (no span clamp, no target stop)
      // Any hold cancels latch in calibration
      if (latch_active) latch_cancel_and_stop();

      manual_drive = true;
      if (open_state && !close_state)       move_dir = -1;
      else if (!open_state && close_state)  move_dir = +1;
      else                                   move_dir = 0;
    }
  }

  // === Stepping ===
#if USE_US_TIMING
  static uint32_t last_step_us = 0;
  uint32_t now_us = micros();
  if (move_dir!=0 && (uint32_t)(now_us - last_step_us) >= STEP_DELAY_US){
    last_step_us = now_us;
#else
  if (move_dir!=0 && (now_ms - last_step_ms) >= STEP_DELAY_MS){
    last_step_ms = now_ms;
#endif

#if SEQUENCE_REVERSE
    // Elektriksel sıralamayı ters çevir (yön değişir)
    if (move_dir > 0){ step_index = (step_index + 7) & 7; current_pos++; }
    else              { step_index = (step_index + 1) & 7; current_pos--; }
#else
    if (move_dir > 0){ step_index = (step_index + 1) & 7; current_pos++; }
    else              { step_index = (step_index + 7) & 7; current_pos--; }
#endif
    motor_step(step_index);

    int32_t lo = home_pos, hi = home_pos + span;

    // Hard safety clamp ONLY when not in calibration
    if (!calibration_mode){
      if (current_pos <= lo){ current_pos = lo; move_dir = 0; motor_release(); report_stop_sync(); }
      if (current_pos >= hi){ current_pos = hi; move_dir = 0; motor_release(); report_stop_sync(); }

      // Stop at target (manual, latch veya HomeKit fark etmez)
      if ((move_dir>0 && current_pos >= target_steps) ||
          (move_dir<0 && current_pos <= target_steps)){
        current_pos = clamp_i32(current_pos, lo, hi);
        move_dir = 0;
        motor_release();
        report_stop_sync();           // Home arayüzü “closing/opening” takılmaz
        // Latch bittiğinde bayrağı düşür
        if (latch_active) { latch_active = false; latch_dir = 0; }
      }
    } else {
      // calibration: serbest — clamp yok, target yok
    }

    // Position report (downsample)
    static uint8_t reportDiv=0;
    if (++reportDiv>=10){
      reportDiv=0;
      report_position_immediate();
    }
  }

  // Idle coils off
  if (move_dir==0)
    motor_release();

  // Persist when idle
  static uint32_t last_save = 0;
  if (move_dir==0 && (millis() - last_save) > 1000){
    prefs.putInt("pos", (int)current_pos);
    last_save = millis();
  }

  // update prev states after all logic
  open_prev = open_state;
  close_prev = close_state;
}

// ===== Arduino / HomeSpan =====
void setup(){
  Serial.begin(115200);
  delay(200);

  prefs.begin("storage", false);
  home_pos    = prefs.getInt("home", 0);
  current_pos = prefs.getInt("pos", home_pos);

  io_init();

  // Pairing & QR
  homeSpan.setPairingCode("53827491");  // 8 haneli, "too simple" değil
  homeSpan.setQRID("SBK1");             // 4-char Setup ID (reset sonrası aktif)

  homeSpan.begin(Category::WindowCoverings,"Smart Curtain");

  new SpanAccessory();
    new Service::AccessoryInformation();
      new Characteristic::Identify();
      new Characteristic::Name("Curtain");
      new Characteristic::Manufacturer("SelimBurak");
      new Characteristic::Model("ESP32-C3-28BYJ48");
      new Characteristic::SerialNumber("C3-0002");
      new Characteristic::FirmwareRevision("1.1.0"); // bumped
    new Service::HAPProtocolInformation();
      new Characteristic::Version("1.1.0");
    wc = new CurtainWC();

  Serial.println("HomeKit Setup Code : 53827491");
  Serial.println("QR için: Seri Monitör'de '?' yaz → menüde 'Display QR/Setup Payload'.");
  LOGI(TAG,"Setup done. home=%ld pos=%ld", (long)home_pos, (long)current_pos);
}

void loop(){
  homeSpan.poll();

  static uint32_t last_ms = 0;
  uint32_t now = millis();
  if ((now - last_ms) >= 1){
    last_ms = now;
    app_loop_1ms();
  }
}
