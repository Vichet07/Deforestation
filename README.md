#include <TinyGPS++.h>
#include <SoftwareSerial.h>

SoftwareSerial gsmSerial(10, 11); // RX-pin 10, TX-pin 11 (reversed order)
TinyGPSPlus gps;

const int smokeSensorPin = A3; // Smoke sensor analog pin
const int pirSensorPin = 2; // PIR motion sensor digital pin
const int buzzerPin = 9; // Buzzer digital pin

enum SystemState {
  MONITORING,
  ALERTING,
  SENDING_SMS
};

SystemState currentState = MONITORING;

void setup() {
  pinMode(buzzerPin, OUTPUT); // Set buzzer pin as output
  pinMode(pirSensorPin, INPUT); // Set PIR sensor pin as input
  Serial.begin(9600); // Initialize serial communication
  gsmSerial.begin(9600); // Initialize GSM serial communication
  Serial.println("System initialized");
}

void loop() {
  switch (currentState) {
    case MONITORING:
      // Perform monitoring tasks
      monitorSensors();
      break;
    case ALERTING:
      // Perform alerting tasks
      alert("Some alert message");
      currentState = SENDING_SMS;
      break;
    case SENDING_SMS:
      // Perform SMS sending tasks
      sendSMS("Some SMS message");
      currentState = MONITORING;
      break;
  }
}

void monitorSensors() {
  // Smoke sensor
  int smokeValue = analogRead(smokeSensorPin); // Read smoke sensor value

  if (smokeValue > 500) { // If smoke value exceeds threshold (adjust threshold as needed)
    currentState = ALERTING;
  }

  // Motion sensor
  int pirValue = digitalRead(pirSensorPin); // Read PIR sensor value

  if (pirValue == HIGH) { // If motion is detected
    currentState = ALERTING;
  }

  // GPS
  while (gsmSerial.available()) {
    gps.encode(gsmSerial.read());
  }

  if (gps.location.isUpdated()) {
    Serial.print("\r");
    delay(1000);
    sendSMS("I'm lost! Please help me. I'm here: https://www.google.com/maps/place/" + String(gps.location.lat(), 6) + "," + String(gps.location.lng(), 6)); // Send location SMS alert
    delay(1000);
  }

  delay(100); // Delay for stability
}

void alert(String message) {
  digitalWrite(buzzerPin, HIGH); // Turn on buzzer
  delay(1000); // Wait for 1 second
  digitalWrite(buzzerPin, LOW); // Turn off buzzer

  sendSMS(message); // Send alert via SMS
}

void sendSMS(String message) {
  gsmSerial.println("AT+CMGF=1"); // Set SMS mode to text
  delay(500);
  gsmSerial.print("AT+CMGS=\"+85585892303\"\r");
  delay(500);
  gsmSerial.print(message);
  gsmSerial.write((char)26); // ASCII code for Ctrl+Z
  delay(1000);
  gsmSerial.write(0x0D); // Carriage return
  delay(1000);
}
