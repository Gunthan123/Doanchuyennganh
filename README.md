#define BLYNK_TEMPLATE_ID "TMPL6gJ8-Ay8F"
#define BLYNK_TEMPLATE_NAME "Doanchuyennganh"
#define BLYNK_AUTH_TOKEN "_UcaUICGN9EdZ7ZJ8NxXiRoR_USXf9Ed"
#define BLYNK_PRINT Serial  // Enable Serial printing for Blynk debug information
#include "EmonLib.h"  // Include EmonLib for energy monitoring
#include <EEPROM.h>  // Include EEPROM library for storing data
#include <WiFi.h>  // Include WiFi library for network connectivity
#include<WiFiClient.h>
#include <BlynkSimpleEsp32.h>  // Include Blynk library for ESP32
#include <Wire.h>  // Include Wire library for I2C communication
#include <LiquidCrystal_I2C.h>  // Include LiquidCrystal_I2C library for LCD display
LiquidCrystal_I2C lcd(0x27,20, 4);  // Initialize LCD with I2C address 0x27 and size 16x
// Constants for calibration
const float vCalibration = 42.5;  // Voltage calibration factor
const float currCalibration = 1.80;  // Current calibration factor
 
// Blynk and WiFi credentials
char auth[] = "_UcaUICGN9EdZ7ZJ8NxXiRoR_USXf9Ed";  // Blynk authentication token
char ssid[] = "UTC2";  // WiFi SSID
char pass[] = "";  // WiFi password
// EnergyMonitor instance
EnergyMonitor emon;  // Create an instance of EnergyMonitor
// Timer for regular updates
BlynkTimer timer;  // Create a Blynk timer instance
 
// Variables for energy calculation
float kWh = 0.0;  // Variable to store energy consumed in kWh
const float ratePerkWh = 6.5;  // Cost rate per kWh
unsigned long lastMillis = millis();  // Variable to store last time in milliseconds
// EEPROM addresses for each variable
const int addrKWh = 12;  // EEPROM address for kWh
// Display page variable
int displayPage = 0;  // Variable to track current LCD display page
// Reset button pin
const int resetButtonPin = 4;  // Pin for reset button (change to your button pin)
// Function prototypes
void sendEnergyDataToBlynk();  // Prototype for sending energy data to Blynk
void readEnergyDataFromEEPROM();  // Prototype for reading energy data from EEPROM
void saveEnergyDataToEEPROM();  // Prototype for saving energy data to EEPROM
void updateLCD();  // Prototype for updating LCD display
void changeDisplayPage();  // Prototype for changing LCD display page
void sendBillToTelegram();  // Prototype for sending bill to Telegram
void resetEEPROM();  // Prototype for resetting EEPROM data
 
void setup() {
  WiFi.begin(ssid, pass);  // Start WiFi connection
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);  // Wait for WiFi connection
  }
  Blynk.begin(auth, ssid, pass);  // Start Blynk connection
 
  // Initialize the LCD
  lcd.init();  // Initialize the LCD
  lcd.backlight();  // Turn on LCD backlight
 
  // Initialize EEPROM
  EEPROM.begin(32);  // Initialize EEPROM with 32 bytes of storage
 
  // Initialize the reset button pin
  pinMode(resetButtonPin, INPUT_PULLUP);  // Set reset button pin as input with pull-up resistor
 
  // Read stored data from EEPROM
  readEnergyDataFromEEPROM();  // Read energy data from EEPROM
 
  // Setup voltage and current inputs
  emon.voltage(35, vCalibration, 1.7);  // Configure voltage measurement: input pin, calibration, phase shift
  emon.current(34, currCalibration);  // Configure current measurement: input pin, calibration
 
  // Setup timers
  timer.setInterval(2000L, sendEnergyDataToBlynk);  // Set timer to send energy data to Blynk every 2 seconds
  timer.setInterval(2000L, changeDisplayPage);  // Set timer to change display page every 2 seconds
}
 
void loop() {
  Blynk.run();  // Run Blynk
  timer.run();  // Run timers
 
  // Check if the reset button is pressed
  if (digitalRead(resetButtonPin) == LOW) {  // If reset button is pressed (assuming button press connects to ground)
    delay(200);  // Debounce delay
    resetEEPROM();  // Reset EEPROM data
  }
}
 
void sendEnergyDataToBlynk() {
  emon.calcVI(20, 2000);  // Calculate voltage and current: number of half wavelengths (crossings), time-out
  float Vrms = emon.Vrms;  // Get root mean square voltage
  float Irms = emon.Irms;  // Get root mean square current
  float apparentPower = emon.apparentPower;  // Get apparent power
  // Calculate energy consumed in kWh
  unsigned long currentMillis = millis();  // Get current time in milliseconds
  kWh += apparentPower * (currentMillis - lastMillis) / 3600000000.0;  // Update kWh
  lastMillis = currentMillis;  // Update last time
  // Save the latest values to EEPROM
  saveEnergyDataToEEPROM();  // Save energy data to EEPROM
  // Send data to Blynk
  Blynk.virtualWrite(V2, Vrms);  // Send voltage to Blynk virtual pin V0
  Blynk.virtualWrite(V1, Irms);  // Send current to Blynk virtual pin V1
  Blynk.virtualWrite(V3, apparentPower);  // Send apparent power to Blynk virtual pin V2
  Blynk.virtualWrite(V4, kWh);  // Send energy in kWh to Blynk virtual pin V3
  // Update the LCD with the new values
  updateLCD();  // Update LCD display
}
 
void readEnergyDataFromEEPROM() {
  EEPROM.get(addrKWh, kWh);  // Read kWh from EEPROM
  // Initialize to zero if values are invalid
  if (isnan(kWh)) {
    kWh = 0.0;  // Set kWh to 0 if invalid
    saveEnergyDataToEEPROM();  // Save to EEPROM
  }
}
 
void saveEnergyDataToEEPROM() {
  EEPROM.put(addrKWh, kWh);  // Save kWh to EEPROM
  EEPROM.commit();  // Commit EEPROM changes
}
 
void updateLCD() {
  lcd.clear();  // Clear LCD display
  lcd.setCursor(0,0);
  lcd.print("Giam sat dien NLMT!");
    lcd.setCursor(0, 1);  // Set cursor to first row
    lcd.printf("V:%.fV I: %.f A", emon.Vrms, emon.Irms);  // Print voltage and current
    lcd.setCursor(0, 2);  // Set cursor to second row
    lcd.printf("P: %.f Watt", emon.apparentPower);  // Print power and energy
    lcd.setCursor(0, 3);  // Set cursor to first row
    lcd.printf("Energy: %.2fkWh",kWh);  // Print energy
  }
void changeDisplayPage() {
  displayPage = (displayPage + 1) % 2;  // Change display page
  updateLCD();  // Update LCD display
}
void resetEEPROM() {
  kWh = 0.0;  // Reset kWh to 0
  saveEnergyDataToEEPROM();  // Save to EEPROM
}
