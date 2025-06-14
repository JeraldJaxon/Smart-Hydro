#define BLYNK_TEMPLATE_ID "TMPL3_w8o50zO"
#define BLYNK_TEMPLATE_NAME "SmartHydro"
#define BLYNK_PRINT Serial
#include <WiFi.h>
#include <BlynkSimpleEsp32.h>
// Blynk Credentials
char auth[] = "pmuKOj_kP6UUP0Cqhf2JTy_XIw3KpNFV";
char ssid[] = "JERALD";
char pass[] = "12345678";
// Pin Definitions
const int trigPin = 12, echoPin = 13, relayPin = 14;
const int vibrationSensorPin = 25, soilMoisturePin = 36;
const int flowSensorPin = 27, phSensorPin = 34, tempSensorPin = 35;
// ADC Settings
const float ADC_REF_VOLTAGE = 3.3;
const float ADC_MAX_VALUE = 4095.0;
// Water Level Thresholds
const float lowLevel = 8.0, highLevel = 15.0;
// Flow Sensor Variables
volatile unsigned long pulseCount = 0;
unsigned long previousMillis = 0;
const unsigned long interval = 1000;
float calibrationFactor = 4.5, flowRate = 0.0;
unsigned long totalMilliLitres = 0;
// Pressure Constants
const float maxFlowRate = 30.0, maxPressure = 1.75;
const float flowRateThreshold = 5.0, pressureThreshold = 0.5;
// Soil Moisture Sensor Threshold
const int leakThreshold = 30;
// Function Prototypes
float measureWaterLevel();
void pulseCounter();
float voltageToPH(float voltage);
void autoCalibrateFlowSensor();
void setup()
 {
    Serial.begin(115200);
    delay(1000);
    Blynk.begin(auth, ssid, pass);
    pinMode(trigPin, OUTPUT);
    pinMode(echoPin, INPUT);
    pinMode(relayPin, OUTPUT);
    pinMode(vibrationSensorPin, INPUT);
    pinMode(soilMoisturePin, INPUT);
    pinMode(flowSensorPin, INPUT_PULLUP);
    pinMode(phSensorPin, INPUT);
    pinMode(tempSensorPin, INPUT);
    attachInterrupt(digitalPinToInterrupt(flowSensorPin), pulseCounter, FALLING);
    digitalWrite(relayPin, LOW);
    autoCalibrateFlowSensor();  // Auto-calibrate flow sensor
}
void loop() 
{
    Blynk.run();
// Water Level Monitoring
    float waterLevel = measureWaterLevel();
    Serial.print("Water Level: ");
    Serial.print(waterLevel);
    Serial.println(" cm");
    
    if (waterLevel < lowLevel) 
{
        digitalWrite(relayPin, HIGH);
        Serial.println("Pump OFF");
    } else if (waterLevel > highLevel) 
{
        digitalWrite(relayPin, LOW);
        Serial.println("Pump ON");
    }
    Blynk.virtualWrite(V1, waterLevel);
    // pH Sensor Reading
    int phRaw = analogRead(phSensorPin);
    float phVoltage = (phRaw * ADC_REF_VOLTAGE) / ADC_MAX_VALUE;
    float phValue = voltageToPH(phVoltage);
    Serial.print("pH Value: ");
    Serial.print(phValue, 2);
    Serial.println();
    Blynk.virtualWrite(V2, phValue);
    // Temperature Sensor Reading
    int tempRaw = analogRead(tempSensorPin);
    float tempVoltage = (tempRaw * ADC_REF_VOLTAGE) / ADC_MAX_VALUE;
float temperature = tempVoltage * 13.24;
    Serial.print("Temperature: ");
    Serial.print(temperature, 2);
    Serial.println(" °C");
    Blynk.virtualWrite(V3, temperature);
    // Flow Rate and Pressure Calculation
    unsigned long currentMillis = millis();
    if (currentMillis - previousMillis >= interval) {
        detachInterrupt(digitalPinToInterrupt(flowSensorPin));
        flowRate = ((1000.0 / (currentMillis - previousMillis)) * pulseCount) /  calibrationFactor;
        unsigned long flowMilliLitres = (flowRate / 60) * 1000;
        totalMilliLitres += flowMilliLitres;
        float pressureKPa = (flowRate / maxFlowRate) * maxPressure * 1000;
        Serial.print("Flow Rate: ");
        Serial.print(flowRate, 2);
        Serial.print(" L/min | Total Volume: ");
        Serial.print(totalMilliLitres / 1000.0, 2);
        Serial.print(" L | Pressure: ");
        Serial.print(pressureKPa, 2);
        Serial.println(" kPa");
        Blynk.virtualWrite(V4, flowRate);
        Blynk.virtualWrite(V5, pressureKPa);
        pulseCount = 0;
        previousMillis = currentMillis;
        attachInterrupt(digitalPinToInterrupt(flowSensorPin), pulseCounter,
FALLING);	
    }
    // Soil Moisture Sensor
    int soilAnalog = analogRead(soilMoisturePin);
    int moisture = (100 - ((soilAnalog / ADC_MAX_VALUE) * 100));
    Serial.print("Soil Moisture: ");
    Serial.print(moisture);
    Serial.println("%");
    if (moisture > leakThreshold)
 {
        Serial.println("ALERT: Leakage detected at joints!");
    }
    Blynk.virtualWrite(V6, moisture);
    // Vibration Sensor Detection
    if (digitalRead(vibrationSensorPin))
 {
        Serial.println("Detected Vibration...");
        Blynk.virtualWrite(V7, 1);
    }
 else
 {
        Blynk.virtualWrite(V7, 0);
    }

    delay(1000);
}
// Measure Water Level using Ultrasonic Sensor
float measureWaterLevel()
 {
    digitalWrite(trigPin, HIGH);
    delayMicroseconds(10);
    digitalWrite(trigPin, LOW);
    while (!digitalRead(echoPin));
    unsigned long startTime = micros();
    while (digitalRead(echoPin));
    unsigned long endTime = micros();
    unsigned long pulseDuration = endTime - startTime;
    return (pulseDuration * 0.0343) / 2;
}
// Interrupt Service Routine for Flow Sensor
void IRAM_ATTR pulseCounter() 
{
    pulseCount++;
}
// Convert Voltage to pH Value
float voltageToPH(float voltage)
 {
    return 7.0 + ((2.5 - voltage) * 3.5);
}
// Auto-Calibrate Flow Sensor
void autoCalibrateFlowSensor() 
{
    Serial.println("Calibrating Flow Sensor...");
    unsigned long startMillis = millis();
    pulseCount = 0;
    while (millis() - startMillis < 5000);  // Wait for 5 seconds
    calibrationFactor = (1000.0 / (millis() - startMillis)) * pulseCount;
    Serial.print("New Calibration Factor: ");
    Serial.println(calibrationFactor);
}
