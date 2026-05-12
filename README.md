# Smart-HVAC-Controller-
A commercially viable Industrial IoT Smart HVAC Controller using the ESP32 platform. The project integrates embedded systems engineering, RTOS multi-tasking, interrupt handling, ADC integration, structured C programming, industrial safety logic, cybersecurity awareness, and commercial IoT deployment strategy


#include "DHTesp.h"

DHTesp dht;

// Pins
const int DHT_PIN = 15;
const int POT_PIN = 34;
const int RELAY_PIN = 18;
const int BUZZER_PIN = 19;
const int BUTTON_PIN = 4;

// Shared variables
float currentTemp = 0;
float filteredTemp = 0;
int adcValue = 0;
bool emergencyStop = false;

// Mutex
SemaphoreHandle_t dataMutex;

// Debounce
volatile unsigned long lastInterruptTime = 0;

// Interrupt Service Routine
void IRAM_ATTR emergencyISR() {

  unsigned long interruptTime = millis();

  if(interruptTime - lastInterruptTime > 200) {
    emergencyStop = true;
  }

  lastInterruptTime = interruptTime;
}

// ================= SENSOR TASK =================
void sensorTask(void * parameter) {

  while(true) {

    TempAndHumidity data = dht.getTempAndHumidity();

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    currentTemp = data.temperature;

    // Simple filtering
    filteredTemp = (filteredTemp * 0.7) + (currentTemp * 0.3);

    xSemaphoreGive(dataMutex);

    vTaskDelay(2000 / portTICK_PERIOD_MS);
  }
}

// ================= ADC TASK =================
void adcTask(void * parameter) {

  while(true) {

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    adcValue = analogRead(POT_PIN);

    xSemaphoreGive(dataMutex);

    vTaskDelay(1000 / portTICK_PERIOD_MS);
  }
}

// ================= SAFETY TASK =================
void safetyTask(void * parameter) {

  while(true) {

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    float temp = filteredTemp;

    xSemaphoreGive(dataMutex);

    // Emergency stop
    if(emergencyStop) {

      digitalWrite(RELAY_PIN, LOW);

      digitalWrite(BUZZER_PIN, HIGH);

      Serial.println("EMERGENCY STOP ACTIVATED");

    }

    // Overheat condition
    else if(temp > 40) {

      digitalWrite(RELAY_PIN, LOW);

      digitalWrite(BUZZER_PIN, HIGH);

      Serial.println("WARNING: OVERHEAT");

    }

    // Cooling mode
    else if(temp > 30) {

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

    vTaskDelay(1000 / portTICK_PERIOD_MS);
  }
}

// ================= LOGGING TASK =================
void loggingTask(void * parameter) {

  while(true) {

    xSemaphoreTake(dataMutex, portMAX_DELAY);

    float temp = filteredTemp;
    int adc = adcValue;

    xSemaphoreGive(dataMutex);

    float voltage = (adc / 4095.0) * 3.3;

    Serial.println("---------------");

    Serial.print("Temperature: ");
    Serial.print(temp);
    Serial.println(" C");

    Serial.print("ADC Value: ");
    Serial.println(adc);

    Serial.print("Voltage: ");
    Serial.println(voltage);

    Serial.println("---------------");

    vTaskDelay(3000 / portTICK_PERIOD_MS);
  }
}

// ================= SETUP =================
void setup() {

  Serial.begin(115200);

  dht.setup(DHT_PIN, DHTesp::DHT22);

  pinMode(RELAY_PIN, OUTPUT);

  pinMode(BUZZER_PIN, OUTPUT);

  pinMode(BUTTON_PIN, INPUT_PULLUP);

  attachInterrupt(
    digitalPinToInterrupt(BUTTON_PIN),
    emergencyISR,
    FALLING
  );

  // Create mutex
  dataMutex = xSemaphoreCreateMutex();

  // Create tasks
  xTaskCreate(sensorTask, "Sensor Task", 2048, NULL, 2, NULL);

  xTaskCreate(adcTask, "ADC Task", 2048, NULL, 1, NULL);

  xTaskCreate(safetyTask, "Safety Task", 2048, NULL, 3, NULL);

  xTaskCreate(loggingTask, "Logging Task", 2048, NULL, 1, NULL);

  Serial.println("Industrial HVAC Controller Started");
}

// ================= LOOP =================
void loop() {

}
