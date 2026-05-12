# Smart-HVAC-Controller-
A commercially viable Industrial IoT Smart HVAC Controller using the ESP32 platform. The project integrates embedded systems engineering, RTOS multi-tasking, interrupt handling, ADC integration, structured C programming, industrial safety logic, cybersecurity awareness, and commercial IoT deployment strategy


#include "DHTesp.h"
#include <esp_task_wdt.h>

// -------------------- PINS --------------------
const int DHT_PIN     = 15;
const int POT_PIN     = 34;
const int RELAY_PIN   = 18;
const int BUZZER_PIN  = 19;
const int BUTTON_PIN  = 4;

// -------------------- DHT ---------------------
DHTesp dht;

// -------------------- Shared data struct --------------------
typedef struct {
  float currentTemp;
  float filteredTemp;
  int   adcValue;
  bool  emergencyStop;
  bool  sensorFault;
  unsigned long lastSensorUpdateMs;
} SharedData;

SharedData shared;
SharedData *sharedPtr = &shared;

// -------------------- RTOS / Sync --------------------
SemaphoreHandle_t dataMutex;

// -------------------- Interrupt / debounce --------------------
volatile unsigned long lastInterruptTime = 0;

// -------------------- Watchdog --------------------
const int WDT_TIMEOUT_SEC = 5;

// -------------------- Helper: ADC calibration --------------------
// Example: convert raw ADC to a “calibrated” temperature-like value
// You can explain this equation in the report as a linear calibration model.
float calibrateAdcToTemp(int adcRaw) {
  // 12-bit ADC: 0–4095, 3.3V reference
  float voltage = (adcRaw / 4095.0f) * 3.3f;
  // Dummy linear calibration: T = a*V + b
  // Suppose sensor gives 10 mV/°C with offset.
  float a = 30.0f;   // slope
  float b = 5.0f;    // offset
  return a * voltage + b;
}

// -------------------- ISR --------------------
void IRAM_ATTR emergencyISR() {
  unsigned long interruptTime = millis();
  // Debounce + basic flood protection (ignore presses <200 ms apart)
  if (interruptTime - lastInterruptTime > 200) {
    // Set flag only; actual handling in task (safer)
    shared.emergencyStop = true;
    lastInterruptTime = interruptTime;
  }
}

// ================= SENSOR TASK =================
void sensorTask(void * parameter) {
  for (;;) {
    TempAndHumidity data = dht.getTempAndHumidity();

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    if (isnan(data.temperature) || isnan(data.humidity)) {
      // Anomaly detection: invalid reading
      sharedPtr->sensorFault = true;
    } else {
      sharedPtr->sensorFault = false;
      sharedPtr->currentTemp = data.temperature;
      // Simple exponential smoothing filter
      sharedPtr->filteredTemp = (sharedPtr->filteredTemp * 0.7f) +
                                (sharedPtr->currentTemp * 0.3f);
      sharedPtr->lastSensorUpdateMs = millis();
    }

    xSemaphoreGive(dataMutex);

    vTaskDelay(pdMS_TO_TICKS(2000));
  }
}

// ================= ADC TASK =================
void adcTask(void * parameter) {
  for (;;) {
    int raw = analogRead(POT_PIN);

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    sharedPtr->adcValue = raw;
    xSemaphoreGive(dataMutex);

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// ================= SAFETY TASK =================
void safetyTask(void * parameter) {
  for (;;) {
    float temp;
    bool emergency;
    bool fault;

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    temp      = sharedPtr->filteredTemp;
    emergency = sharedPtr->emergencyStop;
    fault     = sharedPtr->sensorFault;
    xSemaphoreGive(dataMutex);

    // Sensor fault → fail-safe
    if (fault) {
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, HIGH);
      Serial.println("SENSOR FAULT - FAIL SAFE (RELAY OFF, BUZZER ON)");
    }
    // Emergency stop
    else if (emergency) {
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, HIGH);
      Serial.println("EMERGENCY STOP ACTIVATED");
    }
    // Overheat condition
    else if (temp > 40.0f) {
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, HIGH);
      Serial.println("WARNING: OVERHEAT - SHUTDOWN");
    }
    // Cooling mode
    else if (temp > 30.0f) {
      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(BUZZER_PIN, LOW);
      Serial.println("Cooling Fan ON");
    }
    // Normal mode
    else {
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
      Serial.println("System Normal");
    }

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// ================= LOGGING TASK =================
void loggingTask(void * parameter) {
  for (;;) {
    float temp;
    int adc;
    bool fault;
    bool emergency;

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    temp      = sharedPtr->filteredTemp;
    adc       = sharedPtr->adcValue;
    fault     = sharedPtr->sensorFault;
    emergency = sharedPtr->emergencyStop;
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

    Serial.print("Calibrated Temp (from ADC): ");
    Serial.print(calibTemp);
    Serial.println(" C");

    Serial.print("Sensor Fault: ");
    Serial.println(fault ? "YES" : "NO");

    Serial.print("Emergency Stop: ");
    Serial.println(emergency ? "YES" : "NO");
    Serial.println("---------------");

    vTaskDelay(pdMS_TO_TICKS(3000));
  }
}

// ================= WATCHDOG / FAULT TOLERANCE TASK =================
void watchdogTask(void * parameter) {
  for (;;) {
    unsigned long now = millis();
    unsigned long lastUpdate;

    xSemaphoreTake(dataMutex, portMAX_DELAY);
    lastUpdate = sharedPtr->lastSensorUpdateMs;
    xSemaphoreGive(dataMutex);

    // If sensor hasn't updated for > WDT_TIMEOUT_SEC → treat as fault
    if (now - lastUpdate > (unsigned long)WDT_TIMEOUT_SEC * 1000UL) {
      Serial.println("WATCHDOG: SENSOR TIMEOUT - FORCING FAIL-SAFE");
      xSemaphoreTake(dataMutex, portMAX_DELAY);
      sharedPtr->sensorFault = true;
      xSemaphoreGive(dataMutex);
    }

    vTaskDelay(pdMS_TO_TICKS(1000));
  }
}

// ================= SETUP =================
void setup() {
  Serial.begin(115200);

  // Init shared data
  sharedPtr->currentTemp        = 0.0f;
  sharedPtr->filteredTemp       = 0.0f;
  sharedPtr->adcValue           = 0;
  sharedPtr->emergencyStop      = false;
  sharedPtr->sensorFault        = false;
  sharedPtr->lastSensorUpdateMs = millis();

  dht.setup(DHT_PIN, DHTesp::DHT22);

  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  attachInterrupt(digitalPinToInterrupt(BUTTON_PIN), emergencyISR, FALLING);

  // Create mutex
  dataMutex = xSemaphoreCreateMutex();

  // Optional: hardware watchdog for this core (you can mention this in report)
  esp_task_wdt_init(WDT_TIMEOUT_SEC, true);

  // Create tasks with priorities
  xTaskCreate(sensorTask,   "Sensor Task",   4096, NULL, 3, NULL);
  xTaskCreate(adcTask,      "ADC Task",      2048, NULL, 1, NULL);
  xTaskCreate(safetyTask,   "Safety Task",   4096, NULL, 4, NULL);
  xTaskCreate(loggingTask,  "Logging Task",  4096, NULL, 1, NULL);
  xTaskCreate(watchdogTask, "Watchdog Task", 2048, NULL, 2, NULL);

  Serial.println("Industrial HVAC Controller Started");
}

// ================= LOOP =================
void loop() {
  // Empty: all logic in FreeRTOS tasks
}
