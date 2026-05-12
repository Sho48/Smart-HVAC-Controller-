# Smart-HVAC-Controller-
A commercially viable Industrial IoT Smart HVAC Controller using the ESP32 platform. The project integrates embedded systems engineering, RTOS multi-tasking, interrupt handling, ADC integration, structured C programming, industrial safety logic, cybersecurity awareness, and commercial IoT deployment strategy


#include "DHTesp.h"
#include <esp_task_wdt.h>

// =====================================================
//                     PIN DEFINITIONS
// =====================================================

const int DHT_PIN            = 15;
const int POT_PIN            = 34;
const int RELAY_PIN          = 18;
const int BUZZER_PIN         = 19;

const int MANUAL_BUTTON_PIN  = 4;    // Manual override
const int EMERGENCY_BTN_PIN  = 27;   // Emergency stop

const int STATUS_LED_PIN     = 2;    // Status LED

// =====================================================
//                     DHT SENSOR
// =====================================================

DHTesp dht;

// =====================================================
//                SHARED DATA STRUCTURE
// =====================================================

typedef struct {

  float currentTemp;
  float filteredTemp;

  int adcValue;

  bool emergencyStop;
  bool manualOverride;
  bool sensorFault;

  unsigned long lastSensorUpdateMs;

} SharedData;

SharedData shared;
SharedData *sharedPtr = &shared;

// =====================================================
//                    RTOS / MUTEX
// =====================================================

SemaphoreHandle_t dataMutex;

// =====================================================
//                 INTERRUPT / DEBOUNCE
// =====================================================

volatile unsigned long lastInterruptTime = 0;

// =====================================================
//                    WATCHDOG
// =====================================================

const int WDT_TIMEOUT_SEC = 5;

// =====================================================
//              ADC CALIBRATION FUNCTION
// =====================================================

float calibrateAdcToTemp(int adcRaw) {

  float voltage = (adcRaw / 4095.0f) * 3.3f;

  float a = 30.0f;
  float b = 5.0f;

  return a * voltage + b;
}

// =====================================================
//                   EMERGENCY ISR
// =====================================================

void IRAM_ATTR emergencyISR() {

  unsigned long interruptTime = millis();

  if (interruptTime - lastInterruptTime > 200) {

    shared.emergencyStop = true;

    lastInterruptTime = interruptTime;
  }
}

// =====================================================
//               MANUAL OVERRIDE ISR
// =====================================================

void IRAM_ATTR manualOverrideISR() {

  unsigned long interruptTime = millis();

  if (interruptTime - lastInterruptTime > 200) {

    shared.manualOverride = !shared.manualOverride;

    lastInterruptTime = interruptTime;
  }
}

// =====================================================
//                    SENSOR TASK
// =====================================================

void sensorTask(void * parameter) {

  for (;;) {

    TempAndHumidity data = dht.getTempAndHumidity();

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    if (isnan(data.temperature) || isnan(data.humidity)) {

      sharedPtr->sensorFault = true;

    } else {

      sharedPtr->sensorFault = false;

      sharedPtr->currentTemp = data.temperature;

      sharedPtr->filteredTemp =
        (sharedPtr->filteredTemp * 0.7f) +
        (sharedPtr->currentTemp * 0.3f);

      sharedPtr->lastSensorUpdateMs = millis();
    }

    xSemaphoreGive(dataMutex);

    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}

// =====================================================
//                      ADC TASK
// =====================================================

void adcTask(void * parameter) {

  for (;;) {

    int raw = analogRead(POT_PIN);

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    sharedPtr->adcValue = raw;

    xSemaphoreGive(dataMutex);

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// =====================================================
//                    SAFETY TASK
// =====================================================

void safetyTask(void * parameter) {

  for (;;) {

    float temp;
    bool emergency;
    bool fault;
    bool overrideMode;

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    temp         = sharedPtr->filteredTemp;
    emergency    = sharedPtr->emergencyStop;
    fault        = sharedPtr->sensorFault;
    overrideMode = sharedPtr->manualOverride;

    xSemaphoreGive(dataMutex);

    // ================= SENSOR FAULT =================

    if (fault) {

      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(STATUS_LED_PIN, HIGH);

      Serial.println("SENSOR FAULT - FAIL SAFE");
    }

    // ================= EMERGENCY STOP =================

    else if (emergency) {

      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, HIGH);

      // Blinking LED
      digitalWrite(STATUS_LED_PIN,
                   !digitalRead(STATUS_LED_PIN));

      Serial.println("EMERGENCY STOP ACTIVATED");
    }

    // ================= MANUAL OVERRIDE =================

    else if (overrideMode) {

      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(STATUS_LED_PIN, HIGH);

      Serial.println("MANUAL OVERRIDE MODE");
    }

    // ================= OVERHEAT =================

    else if (temp > 40.0f) {

      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(STATUS_LED_PIN, HIGH);

      Serial.println("WARNING: OVERHEAT - SHUTDOWN");
    }

    // ================= COOLING MODE =================

    else if (temp > 30.0f) {

      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(STATUS_LED_PIN, HIGH);

      Serial.println("Cooling Fan ON");
    }

    // ================= NORMAL MODE =================

    else {

      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(STATUS_LED_PIN, LOW);

      Serial.println("System Normal");
    }

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// =====================================================
//                    LOGGING TASK
// =====================================================

void loggingTask(void * parameter) {

  for (;;) {

    float temp;
    int adc;
    bool fault;
    bool emergency;
    bool overrideMode;

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    temp         = sharedPtr->filteredTemp;
    adc          = sharedPtr->adcValue;
    fault        = sharedPtr->sensorFault;
    emergency    = sharedPtr->emergencyStop;
    overrideMode = sharedPtr->manualOverride;

    xSemaphoreGive(dataMutex);

    float voltage = (adc / 4095.0f) * 3.3f;

    float calibTemp = calibrateAdcToTemp(adc);

    Serial.println("---------------");

    Serial.print("Filtered Temp: ");
    Serial.print(temp);
    Serial.println(" C");

    Serial.print("ADC Raw: ");
    Serial.println(adc);

    Serial.print("ADC Voltage: ");
    Serial.print(voltage, 3);
    Serial.println(" V");

    Serial.print("Calibrated Temp: ");
    Serial.print(calibTemp);
    Serial.println(" C");

    Serial.print("Sensor Fault: ");
    Serial.println(fault ? "YES" : "NO");

    Serial.print("Emergency Stop: ");
    Serial.println(emergency ? "YES" : "NO");

    Serial.print("Manual Override: ");
    Serial.println(overrideMode ? "ON" : "OFF");

    Serial.println("---------------");

    vTaskDelay(pdMS_TO_TICKS(3000));
  }
}

// =====================================================
//                  WATCHDOG TASK
// =====================================================

void watchdogTask(void * parameter) {

  for (;;) {

    unsigned long now = millis();

    unsigned long lastUpdate;

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    lastUpdate = sharedPtr->lastSensorUpdateMs;

    xSemaphoreGive(dataMutex);

    if (now - lastUpdate >
        (unsigned long)WDT_TIMEOUT_SEC * 1000UL) {

      Serial.println(
        "WATCHDOG: SENSOR TIMEOUT - FAIL SAFE");

      xSemaphoreTake(dataMutex, portMAX_DELAY);

      sharedPtr->sensorFault = true;

      xSemaphoreGive(dataMutex);
    }

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// =====================================================
//                         SETUP
// =====================================================
void setup() {

  Serial.begin(115200);

  // ================= INIT SHARED DATA =================

  sharedPtr->currentTemp        = 0.0f;
  sharedPtr->filteredTemp       = 0.0f;
  sharedPtr->adcValue           = 0;

  sharedPtr->emergencyStop      = false;
  sharedPtr->manualOverride     = false;
  sharedPtr->sensorFault        = false;

  sharedPtr->lastSensorUpdateMs = millis();

  // ================= DHT SENSOR =================

  dht.setup(DHT_PIN, DHTesp::DHT22);

  // ================= PIN MODES =================

  pinMode(RELAY_PIN, OUTPUT);

  pinMode(BUZZER_PIN, OUTPUT);

  pinMode(STATUS_LED_PIN, OUTPUT);

  pinMode(MANUAL_BUTTON_PIN, INPUT_PULLUP);

  pinMode(EMERGENCY_BTN_PIN, INPUT_PULLUP);

  // ================= INTERRUPTS =================

  attachInterrupt(
    digitalPinToInterrupt(MANUAL_BUTTON_PIN),
    manualOverrideISR,
    FALLING
  );

  attachInterrupt(
    digitalPinToInterrupt(EMERGENCY_BTN_PIN),
    emergencyISR,
    FALLING
  );

  // ================= CREATE MUTEX =================

  dataMutex = xSemaphoreCreateMutex();

  // ================= WATCHDOG =================

  esp_task_wdt_config_t wdt_config = {
    .timeout_ms = WDT_TIMEOUT_SEC * 1000,
    .idle_core_mask = (1 << portNUM_PROCESSORS) - 1,
    .trigger_panic = true
  };

  esp_task_wdt_init(&wdt_config);

  // ================= CREATE TASKS =================

  xTaskCreate(
    sensorTask,
    "Sensor Task",
    4096,
    NULL,
    3,
    NULL
  );

  xTaskCreate(
    adcTask,
    "ADC Task",
    2048,
    NULL,
    1,
    NULL
  );

  xTaskCreate(
    safetyTask,
    "Safety Task",
    4096,
    NULL,
    4,
    NULL
  );

  xTaskCreate(
    loggingTask,
    "Logging Task",
    4096,
    NULL,
    1,
    NULL
  );

  xTaskCreate(
    watchdogTask,
    "Watchdog Task",
    2048,
    NULL,
    2,
    NULL
  );

  Serial.println("Industrial HVAC Controller Started");
}

// =====================================================
//                         LOOP
// =====================================================

void loop() {

  // Empty because FreeRTOS tasks handle everything
}
