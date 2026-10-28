---
layout: page
title: "GuardSense: Building an IoT Intruder Detector"
permalink: /guardsense-cookbook.html
---

Welcome to **GuardSense** — a hands-on project series where you'll go from your very first blinking LED to a fully automated intruder-detection system that emails you an alert the moment motion is detected. Along the way you'll also build a pulse oximeter to measure heart rate and blood oxygen.

> **Before you start:** Complete the **ESP32 Development Environment Setup (Prerequisites)** guide first — it covers installing the Arduino IDE, ESP32 board support, the USB driver, and required libraries.

## Table of Contents
- [Overview](#overview)
- [Project 1: Hello Hardware (LED Blink)](#project-1-hello-hardware-led-blink)
  - [What is ESP32?](#what-is-esp32)
  - [Objective](#objective)
  - [Blink Code](#blink-code)
  - [Uploading the Sketch](#uploading-the-sketch)
- [Project 2: Intruder Detection with an IR Sensor](#project-2-intruder-detection-with-an-ir-sensor)
  - [Components Required for Intruder Detection](#components-required-for-intruder-detection)
  - [IR Sensor Circuit and Connections](#ir-sensor-circuit-and-connections)
  - [Reading the Raw IR Sensor Value](#reading-the-raw-ir-sensor-value)
  - [Implementing the Intruder Detection Logic](#implementing-the-intruder-detection-logic)
- [Project 3: Automating Alerts with n8n and Gmail](#project-3-automating-alerts-with-n8n-and-gmail)
  - [Required Libraries for Alerting](#required-libraries-for-alerting)
  - [ESP32 Alert Code](#esp32-alert-code)
  - [Setting Up Your n8n Account](#setting-up-your-n8n-account)
  - [Building the n8n Workflow](#building-the-n8n-workflow)
  - [Testing the Complete System](#testing-the-complete-system)
- [Project 4: Smart Pulse Oximeter (Heart Rate and SpO2)](#project-4-smart-pulse-oximeter-heart-rate-and-spo2)
  - [Components Required for the Pulse Oximeter](#components-required-for-the-pulse-oximeter)
  - [MAX30102 Circuit and Connections](#max30102-circuit-and-connections)
  - [Installing the MAX3010x Library](#installing-the-max3010x-library)
  - [Pulse Oximeter Arduino Sketch](#pulse-oximeter-arduino-sketch)
  - [Upload and Test](#upload-and-test)
- [Project Files](#project-files)

---

## Overview

Just like **'Hello World'** in programming, in the world of hardware our first program is usually an LED blink — the **'Hello Hardware'** exercise. From there, you'll build up to a real **intruder detection system** using an IR sensor, automate alerts with **n8n** and **Gmail**, and finally build a **pulse oximeter** to monitor heart rate and SpO2.

---

## Project 1: Hello Hardware (LED Blink)

### What is ESP32?

The **ESP32** is a powerful microcontroller with **Wi-Fi** and **Bluetooth** capabilities. Today we're starting simple — it has an onboard LED connected to a GPIO pin, and that's what we'll be working with.

### Objective

The goal is to write a small program that turns the onboard LED on and off at regular intervals. This helps you understand how to control pins and gives hands-on experience with uploading code to the board.

### Blink Code

Open a new sketch in Arduino IDE, remove the existing boilerplate code, and paste the following:

```cpp
const int LED_PIN = 1;

void setup() {
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_PIN, HIGH);
  delay(1000);
  digitalWrite(LED_PIN, LOW);
  delay(1000);
}
```

### Uploading the Sketch

**Step 0:** Connect your ESP32 to the laptop via a micro-USB cable.

**Step 1:** Select your board before uploading the code. On the top dropdown menu, click **"Select other board and port..."**

![Select board and port menu](images/guardsense-hellohw-select-board-port.png)

Search for your ESP32 board model as shown below:

![Select ESP32 board model in Arduino IDE 2](images/guardsense-hellohw-select-esp32-board-model.png)

Select the ESP32 board model you're using and the COM port — in this example, the **ESP32 DEV Module**. Click **OK** when done.

**Step 2:** Click the **Upload** button. After a few seconds, the upload should complete.

![Arduino IDE 2 upload success](images/guardsense-hellohw-upload-success.png)

> **Output:** The ESP32 on-board LED should be blinking every second.

---

## Project 2: Intruder Detection with an IR Sensor

### Components Required for Intruder Detection

- **ESP32 Dev Board**
- **IR Obstacle Detection Sensor (FC-51)**
- **Jumper Wires**

### IR Sensor Circuit and Connections

| IR Sensor | Connects to ESP32 |
|---|---|
| VCC | 3V3 |
| GND | GND |
| OUT | GPIO 14 |

![IR sensor circuit diagram](images/guardsense-intruder-circuit-diagram.png)

- Connect the **VCC** pin of the sensor module to the **3.3V** pin on the ESP32.
- Connect the **GND** pin of the sensor module to the **GND** pin on the ESP32.
- Connect the **OUT** pin of the sensor module to the **GPIO14** pin on the ESP32 — this is the pin that reads the sensor's output.

### Reading the Raw IR Sensor Value

```cpp
const int IR_SENSOR_PIN = 14; // Pin connected to IR sensor output

void setup() {
  Serial.begin(115200);
  pinMode(IR_SENSOR_PIN, INPUT);
  Serial.println("IR Proximity Sensor Reading:");
}

void loop() {
  int sensorValue = digitalRead(IR_SENSOR_PIN);
  Serial.print("Sensor Output: ");
  Serial.println(sensorValue);
  delay(200); // Delay between readings
}
```

Upload the code to the ESP32 and observe the value by placing your hand in front of the sensor.

### Implementing the Intruder Detection Logic

```cpp
const int IR_SENSOR_PIN = 14; // Pin connected to IR sensor output

void setup() {
  Serial.begin(115200);
  pinMode(IR_SENSOR_PIN, INPUT);
  Serial.println("IR Proximity Sensor Reading:");
}

void loop() {
  int sensorValue = digitalRead(IR_SENSOR_PIN);
  Serial.print("Sensor Output: ");
  Serial.println(sensorValue);

  // Check if the sensor has detected something.
  if (sensorValue == LOW) {
    // If the value is LOW, it means an intruder is detected.
    Serial.println("INTRUDER DETECTED!");
  } else {
    // Otherwise, the area is clear.
    Serial.println("ALL CLEAR");
  }

  delay(200); // Delay between readings
}
```

Upload the code to the ESP32 and observe the **Serial Monitor** by placing your hand in front of the sensor.

---

## Project 3: Automating Alerts with n8n and Gmail

Now that you've learned how to detect intruders, let's make the project more interesting by **automating alerts** using **n8n**.

### Required Libraries for Alerting

- **ArduinoJson**: creates and parses JSON objects, used to format the alert message.
- **WiFi** and **HTTPClient**: standard libraries for managing Wi-Fi connections and HTTP requests on the ESP32 — usually pre-installed with the ESP32 board manager.

![Required libraries for alerting](images/guardsense-n8n-required-libraries.png)

Paste the following code in a new sketch file.

> **Note:** Change the `WIFI_SSID`, `WIFI_PASS`, and `WEBHOOK_URL` in the code below.

### ESP32 Alert Code

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

// WiFi credentials
const char* WIFI_SSID = "WIFI_NAME";
const char* WIFI_PASS = "WIFI_PASSWORD";

// Webhook URL
const char* WEBHOOK_URL = "ADD_URL_FROM_N8N_HERE";

// Hardware pins
const int IR_SENSOR_PIN = 14;
const int LED_PIN = 2;

// Timing variables
bool lastSensorState = HIGH;
unsigned long lastDetectionTime = 0;
unsigned long debounceDelay = 1000; // 1 second debounce

void setup() {
  Serial.begin(115200);

  // Setup pins
  pinMode(IR_SENSOR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);

  // Connect to WiFi
  connectToWiFi();

  Serial.println("Intruder Detection System Ready!");

  // Read initial sensor state
  lastSensorState = digitalRead(IR_SENSOR_PIN);
}

void loop() {
  // Read sensor
  bool currentSensorState = digitalRead(IR_SENSOR_PIN);

  // Check if obstacle detected (HIGH to LOW change)
  if (currentSensorState == LOW && lastSensorState == HIGH) {
    // Check if enough time has passed since last detection
    if (millis() - lastDetectionTime > debounceDelay) {
      Serial.println("Motion detected!");
      digitalWrite(LED_PIN, HIGH); // Turn on LED

      // Send alert
      if (sendAlert()) {
        Serial.println("✅ Alert sent successfully");
      } else {
        Serial.println("⚠️ Failed to send alert");
      }

      lastDetectionTime = millis();
      digitalWrite(LED_PIN, LOW); // Turn off LED
    }
  }

  lastSensorState = currentSensorState;
  delay(100);
}

void connectToWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASS);
  Serial.print("Connecting to WiFi");

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Connected! IP: ");
  Serial.println(WiFi.localIP());
}

bool sendAlert() {
  if (WiFi.status() != WL_CONNECTED) {
    return false;
  }

  HTTPClient http;
  http.begin(WEBHOOK_URL);
  http.addHeader("Content-Type", "application/json");

  // Create JSON payload
  StaticJsonDocument<200> jsonDoc;
  jsonDoc["message"] = "🚨 Intruder detected! 🚨";
  jsonDoc["device_id"] = "ESP32_Sensor";
  jsonDoc["location"] = "NIAT_ROBOTICS_LAB";

  String jsonString;
  serializeJson(jsonDoc, jsonString);

  int httpResponseCode = http.POST(jsonString);
  http.end();

  return (httpResponseCode == 200 || httpResponseCode == 201);
}
```

### Setting Up Your n8n Account

Visit **[n8n.io](http://n8n.io)**, click **"Get Started"**, create an account, then select **"Start 14-day free trial."**

![n8n signup - start free trial](images/guardsense-n8n-signup-start-trial.png)

A brief questionnaire will appear asking about the recipient and purpose. Choose **"only for me,"** then your field (e.g., "Engineering"), followed by **"Education,"** and so on — or click **OK** to skip most questions.

![n8n onboarding questionnaire](images/guardsense-n8n-onboarding-questionnaire.png)

You can skip inviting team members.

![Skip inviting team members](images/guardsense-n8n-skip-invite-team.png)

You're now ready to automate your first workflow. Click **"Start automating."**

![Start automating in n8n](images/guardsense-n8n-start-automating.png)

### Building the n8n Workflow

Click **"create workflow"** in the top right to begin.

![Create workflow button](images/guardsense-n8n-create-workflow-button.png)

Click **"add first step"** (the plus icon) to begin designing your automation flow.

![Add first step](images/guardsense-n8n-add-first-step.png)

Search for **"webhook"** — the lightning bolt icon indicates it's a **trigger node**, which initiates the workflow.

![Search for webhook node](images/guardsense-n8n-search-webhook-node.png)

Change the HTTP method from **GET** to **POST** — this lets an external device "post" information to the URL. Change the path from the random string to something readable, like `ESP/intruder`.

![Configure webhook: POST method](images/guardsense-n8n-webhook-config-post-method.png)

Click the **test URL** to copy it. Open your Arduino sketch, find the line `const char* WEBHOOK_URL = "<your_link_here>"`, and replace it with the copied URL (also update `WIFI_SSID` and `WIFI_PASS`). Upload the sketch with the hardware connections already in place, then click **"listen for test event."**

![Copy the webhook test URL](images/guardsense-n8n-webhook-test-url.png)

After an intruder is detected, the schema appears on the right panel of the screen.

![Webhook schema output](images/guardsense-n8n-webhook-schema-output.png)

Press **Esc** or click **"Back to canvas"** to return to the workflow area.

Click the **plus** symbol in front of the webhook trigger node to add another node. Search for the **Date and time** node, which provides the exact date and time for the email.

![Search for the Date and time node](images/guardsense-n8n-search-datetime-node.png)

Select the **"Get current date"** sub-option and click **"execute step."** The node's output appears in the **OUTPUT** panel on the right.

![Execute the Date and time step](images/guardsense-n8n-datetime-execute-step.png)

Add a new node and search for the **Gmail** node. From the available options, select **"Send a message."**

![Gmail: Send a message node](images/guardsense-n8n-gmail-send-message-node.png)

Click the **Credential** input box and select **"Create new credential."**

![Create new Gmail credential](images/guardsense-n8n-gmail-create-credential.png)

Sign in with your Google account.

![Sign in with Google](images/guardsense-n8n-google-signin.png)

After signing in, close the signup tab by clicking **X**, then return to the node.

![Close signup tab](images/guardsense-n8n-close-signup-tab.png)

Fill in the required fields — specify the recipient in the **"To"** field, add your preferred subject, and set the email type to **TEXT**.

![Configure Gmail fields](images/guardsense-n8n-gmail-fields-config.png)

To format your message, type **"Date & time"** and **"Device ID"** in the message box, then click **"Execute previous nodes"** on the left side of the panel.

![Format the Gmail message](images/guardsense-n8n-gmail-message-format.png)

Drop the relevant schema fields into the message box to get the specific input.

![Drop schema into message box](images/guardsense-n8n-gmail-drop-schema.png)

Customize this input as needed — expand the ESP32 webhook node and drag-and-drop the message and device ID fields from the provided schema.

![Customize Gmail message fields](images/guardsense-n8n-gmail-customize-fields.png)

After testing and confirming your workflow functions correctly, go back to your ESP32 code and replace the test URL with the **production URL** in the webhook configuration.

![Replace with production webhook URL](images/guardsense-n8n-webhook-production-url.png)

Click **"Back to canvas,"** then **"Save"** in the top right corner, and activate the switch to keep the workflow running.

![Save and activate the workflow](images/guardsense-n8n-save-activate-workflow.png)

### Testing the Complete System

Your n8n workflow is now set up. Click **"Execute workflow"** to see intruder detection in action.

![Execute workflow test](images/guardsense-n8n-execute-workflow-test.png)

If an intruder is detected, you'll receive a similar email:

![Intruder alert email](images/guardsense-n8n-intruder-alert-email.png)

---

## Project 4: Smart Pulse Oximeter (Heart Rate and SpO2)

### Components Required for the Pulse Oximeter

- **ESP32 Dev board**
- **MAX30102 Sensor**
- **Jumper Wires** (M-F and F-F)
- **Bread Board**
- **Micro USB cable**

### MAX30102 Circuit and Connections

| MAX30102 pin | Connects to ESP32 |
|---|---|
| VIN | 3V3 (preferred). If the board has a 3.3–5V VIN/LDO, 3V3 is ideal. |
| GND | GND |
| SDA | GPIO 21 (SDA) |
| SCL | GPIO 22 (SCL) |

### Installing the MAX3010x Library

Open **Tools → Manage Libraries** and install:

1. Click the **Library Manager** icon on the left sidebar.
2. Type **"MAX3010x"** in the search box to filter results.
3. Look for **SparkFun MAX3010x Pulse and Proximity Sensor Library** by **SparkFun Electronics**.
4. Click **Install** to add it to your Arduino IDE.

![Install MAX3010x library](images/guardsense-pulseox-install-max3010x-library.png)

### Pulse Oximeter Arduino Sketch

Copy and paste this code into a new Arduino sketch:

```cpp
#include <Wire.h>
#include "MAX30105.h"
#include "spo2_algorithm.h"

MAX30105 particleSensor;

uint32_t irBuffer[100];
uint32_t redBuffer[100];
int32_t bufferLength;
int32_t spo2;
int8_t validSPO2;
int32_t heartRate;
int8_t validHeartRate;

byte pulseLED = 2;
byte readLED = 4;

unsigned long lastPrintTime = 0; // for HR & SpO2 print interval

void setup()
{
  Serial.begin(115200);
  pinMode(pulseLED, OUTPUT);
  pinMode(readLED, OUTPUT);

  if (!particleSensor.begin(Wire, I2C_SPEED_FAST))
  {
    Serial.println(F("MAX30105 was not found. Please check wiring/power."));
    while (1);
  }

  Serial.println(F("Attach sensor to finger with rubber band. Starting..."));

  byte ledBrightness = 100;
  byte sampleAverage = 4;
  byte ledMode = 2;
  byte sampleRate = 400;
  int pulseWidth = 411;
  int adcRange = 4096;

  particleSensor.setup(ledBrightness, sampleAverage, ledMode, sampleRate, pulseWidth, adcRange);

  bufferLength = 100;

  for (byte i = 0; i < bufferLength; i++)
  {
    while (particleSensor.available() == false)
      particleSensor.check();

    redBuffer[i] = particleSensor.getRed();
    irBuffer[i] = particleSensor.getIR();
    particleSensor.nextSample();
  }

  maxim_heart_rate_and_oxygen_saturation(
    irBuffer, bufferLength, redBuffer,
    &spo2, &validSPO2, &heartRate, &validHeartRate);
}

void loop()
{
  for (byte i = 25; i < 100; i++)
  {
    redBuffer[i - 25] = redBuffer[i];
    irBuffer[i - 25] = irBuffer[i];
  }

  for (byte i = 75; i < 100; i++)
  {
    while (particleSensor.available() == false)
      particleSensor.check();

    digitalWrite(readLED, !digitalRead(readLED));

    redBuffer[i] = particleSensor.getRed();
    irBuffer[i] = particleSensor.getIR();
    particleSensor.nextSample();

    // 👉 LINE 1: For Serial Plotter (ONLY IR + Red numbers)
    // Serial.print(irBuffer[i]);
    // Serial.print(" ");
    // Serial.println(redBuffer[i]);

    // 👉 LINE 2: For Serial Monitor (HR & SpO2 with text) every 2 sec
    if (millis() - lastPrintTime >= 2000) {
      Serial.print("HR = ");
      Serial.print(heartRate);
      Serial.print(" bpm, SpO2 = ");
      Serial.print(spo2);
      Serial.println(" %");
      lastPrintTime = millis();
    }

    delay(10);
  }

  maxim_heart_rate_and_oxygen_saturation(
    irBuffer, bufferLength, redBuffer,
    &spo2, &validSPO2, &heartRate, &validHeartRate);
}
```

### Upload and Test

1. Connect the ESP32 via USB.
2. Go to **Tools → Board/Port** and select your board and port.
3. Click **Upload**.
4. Open **Serial Monitor** at **115200** baud.
5. Place a finger **lightly** on the sensor (LEDs under the fingertip). After ~5–10 seconds you should see valid **HR** and **SpO2** values.

**Graphical Visualization:** click the **Serial Plotter** icon in the top-right corner of the IDE to visualize the readings.

> **Expected:** Heart Rate is typically **50–100 bpm** at rest, and SpO2 is **94–100%** for healthy individuals at sea level.

---

## Project Files

The following Drive link contains the `.ino` files for all four projects above:

[GuardSense Project Files (Google Drive)](https://drive.google.com/drive/folders/1Xgn3-47aWrxyXiMcQZI5E-3C1US_2t6s?usp=sharing)
