
#include <Arduino.h>
#include <ESP32Servo.h>
#include "BluetoothSerial.h" // Bluetooth kütüphanesi eklendi

// Pin Tanımlamaları
#define TRIG_PIN 5
#define ECHO_PIN 4
#define SERVO_PAN 12
#define SERVO_TILT 13
#define BUZZER_PIN 14
#define LED_RED 26
#define LED_YELLOW 27
#define LED_GREEN 25

// Nesne Tanımlamaları
Servo servoPan, servoTilt;
BluetoothSerial SerialBT; // Bluetooth nesnesi

// Değişkenler
bool radarActive = false;
bool trackingMode = false;
float panAngle = 90;
float distanceThreshold1 = 40.0;
float distanceThreshold2 = 80.0;

void setup() {
    Serial.begin(115200);
    
    // Bluetooth Başlatma
    SerialBT.begin("ESP32_Radar_Sistemi"); // Telefondaki görünecek isim
    Serial.println("Bluetooth cihazı hazır: ESP32_Radar_Sistemi");
    
    pinMode(TRIG_PIN, OUTPUT);
    pinMode(ECHO_PIN, INPUT);
    pinMode(BUZZER_PIN, OUTPUT);
    pinMode(LED_RED, OUTPUT);
    pinMode(LED_YELLOW, OUTPUT);
    pinMode(LED_GREEN, OUTPUT);
    
    servoPan.attach(SERVO_PAN);
    servoTilt.attach(SERVO_TILT);
    servoPan.write(90);
    servoTilt.write(90);
    
    // Açılış mesajlarını hem Serial hem Bluetooth'a gönderiyoruz
    sendToAll("\n--- SISTEM HAZIR ---");
    sendToAll("1: Baslat, 2: Takip, 3: Durdur");
}

void loop() {
    // Hem Kablodan (Serial) hem Bluetooth'tan (SerialBT) komut kabul et
    if (Serial.available() || SerialBT.available()) {
        String input = "";
        if (SerialBT.available()) input = SerialBT.readStringUntil('\n');
        else input = Serial.readStringUntil('\n');
        
        input.trim();
        int choice = input.toInt();
        handleMenuChoice(choice);
    }
    
    if (radarActive) {
        if (trackingMode) {
            performTracking();
        } else {
            performScan();
        }
    }
    delay(10);
}

// Yardımcı Fonksiyon: Veriyi hem PC'ye hem Telefona gönderir
void sendToAll(String msg) {
    Serial.println(msg);
    SerialBT.println(msg);
}

float getDistance() {
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    
    long duration = pulseIn(ECHO_PIN, HIGH, 30000);
    float distance = (duration * 0.034) / 2;
    
    if (distance < 2 || distance > 400) distance = 0;
    return distance;
}

void performScan() {
    static int angle = 0;
    static int step = 5;

    servoPan.write(angle);
    delay(50);
    
    float distance = getDistance();
    updateLEDandAlarm(distance);
    
    // Uygulamanın anlayacağı format: acı,mesafe
    SerialBT.print(angle);
    SerialBT.print(",");
    SerialBT.println(distance);
    
    angle += step;
    if (angle >= 180 || angle <= 0) step = -step;
}

void performTracking() {
    float bestDistance = 999;
    float bestAngle = panAngle;
    
    for (int angle = panAngle - 20; angle <= panAngle + 20; angle += 4) {
        if (angle < 0 || angle > 180) continue;
        servoPan.write(angle);
        delay(30);
        float distance = getDistance();
        if (distance > 5 && distance < bestDistance && distance < 150) {
            bestDistance = distance;
            bestAngle = angle;
        }
    }
    
    if (bestDistance < 150) {
        panAngle = bestAngle;
        servoPan.write(panAngle);
        
        // Mesafe bazlı dikey (tilt) hareket
        int tiltPos = map(bestDistance, 10, 150, 40, 90);
        servoTilt.write(tiltPos);
        
        updateLEDandAlarm(bestDistance);
        SerialBT.print("HEDEF: "); SerialBT.print(panAngle);
        SerialBT.print(" derece, "); SerialBT.print(bestDistance); SerialBT.println(" cm");
    }
}

void updateLEDandAlarm(float distance) {
    if (distance <= 0) {
        digitalWrite(LED_GREEN, HIGH);
        digitalWrite(LED_RED, LOW);
        noTone(BUZZER_PIN);
    } else if (distance <= distanceThreshold1) {
        digitalWrite(LED_RED, HIGH);
        digitalWrite(LED_GREEN, LOW);
        tone(BUZZER_PIN, 1000, 100);
    } else {
        digitalWrite(LED_GREEN, HIGH);
        digitalWrite(LED_RED, LOW);
        noTone(BUZZER_PIN);
    }
}

void handleMenuChoice(int choice) {
    switch (choice) {
        case 1:
            radarActive = true;
            trackingMode = false;
            sendToAll("Radar Baslatildi");
            break;
        case 2:
            radarActive = true;
            trackingMode = true;
            sendToAll("Takip Modu Aktif");
            break;
        case 3:
            radarActive = false;
            noTone(BUZZER_PIN);
            sendToAll("Radar Durduruldu");
            break;
    }
}![Uploading krcircle.png…]()
#include <Arduino.h>
#include <ESP32Servo.h>

#define TRIG_PIN 5
#define ECHO_PIN 4
#define SERVO_PAN 12
#define SERVO_TILT 13
#define BUZZER_PIN 14
#define LED_RED 26
#define LED_YELLOW 27
#define LED_GREEN 25

Servo servoPan, servoTilt;
bool radarActive = false;
bool trackingMode = false;
float panAngle = 90;
float distanceThreshold1 = 40.0;
float distanceThreshold2 = 80.0;

void setup() {
    Serial.begin(115200);
    delay(2000);
    
    pinMode(TRIG_PIN, OUTPUT);
    pinMode(ECHO_PIN, INPUT);
    pinMode(BUZZER_PIN, OUTPUT);
    pinMode(LED_RED, OUTPUT);
    pinMode(LED_YELLOW, OUTPUT);
    pinMode(LED_GREEN, OUTPUT);
    
    servoPan.attach(SERVO_PAN);
    servoTilt.attach(SERVO_TILT);
    servoPan.write(90);
    servoTilt.write(90);
    
    displayWelcome();
    displayMenu();
}

void loop() {
    if (Serial.available()) {
        String input = Serial.readStringUntil('\n');
        input.trim();
        int choice = input.toInt();
        handleMenuChoice(choice);
    }
    
    if (radarActive) {
        if (trackingMode) {
            performTracking();
        } else {
            performScan();
        }
    }
    
    delay(10);
}

float getDistance() {
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    
    long duration = pulseIn(ECHO_PIN, HIGH, 30000);
    float distance = (duration * 0.034) / 2;
    
    if (distance < 2 || distance > 400) {
        distance = 0;
    }
    
    return distance;
}

void performScan() {
    for (int angle = 0; angle <= 180; angle += 5) {
        if (!radarActive) break;
        
        servoPan.write(angle);
        delay(50);
        
        float distance = getDistance();
        updateLEDandAlarm(distance);
        
        Serial.print(angle);
        Serial.print(",");
        Serial.println(distance);
    }
    
    for (int angle = 180; angle >= 0; angle -= 5) {
        if (!radarActive) break;
        
        servoPan.write(angle);
        delay(50);
        
        float distance = getDistance();
        updateLEDandAlarm(distance);
        
        Serial.print(angle);
        Serial.print(",");
        Serial.println(distance);
    }
}

void performTracking() {
    float bestDistance = 999;
    float bestAngle = panAngle;
    
    for (int angle = panAngle - 30; angle <= panAngle + 30; angle += 2) {
        if (angle < 0 || angle > 180) continue;
        
        servoPan.write(angle);
        delay(30);
        
        float distance = getDistance();
        
        if (distance > 10 && distance < bestDistance && distance < 200) {
            bestDistance = distance;
            bestAngle = angle;
        }
    }
    
    if (bestDistance < 200) {
        panAngle = bestAngle;
        servoPan.write(panAngle);
        
        if (bestDistance < 50) {
            servoTilt.write(45);
        } else if (bestDistance < 150) {
            servoTilt.write(75);
        } else {
            servoTilt.write(90);
        }
        
        updateLEDandAlarm(bestDistance);
        
        Serial.print("Hedef - Acı: ");
        Serial.print(panAngle);
        Serial.print("° | Mesafe: ");
        Serial.print(bestDistance);
        Serial.println("cm");
    }
    
    delay(100);
}

void updateLEDandAlarm(float distance) {
    if (distance <= 0 || distance > 400) {
        digitalWrite(LED_RED, LOW);
        digitalWrite(LED_YELLOW, LOW);
        digitalWrite(LED_GREEN, HIGH);
        noTone(BUZZER_PIN);
    } 
    else if (distance <= distanceThreshold1) {
        digitalWrite(LED_RED, HIGH);
        digitalWrite(LED_YELLOW, LOW);
        digitalWrite(LED_GREEN, LOW);
        tone(BUZZER_PIN, 1000);
        delay(100);
        noTone(BUZZER_PIN);
    } 
    else if (distance <= distanceThreshold2) {
        digitalWrite(LED_RED, LOW);
        digitalWrite(LED_YELLOW, HIGH);
        digitalWrite(LED_GREEN, LOW);
        tone(BUZZER_PIN, 800);
        delay(200);
        noTone(BUZZER_PIN);
    } 
    else {
        digitalWrite(LED_RED, LOW);
        digitalWrite(LED_YELLOW, LOW);
        digitalWrite(LED_GREEN, HIGH);
        noTone(BUZZER_PIN);
    }
}

void displayWelcome() {
    Serial.println("\n\n");
    Serial.println("╔═══════════════════════════════════════════════════════╗");
    Serial.println("║   ESP32 - 360° RADAR KONTROL SİSTEMİ v1.0           ║");
    Serial.println("║   HCSR04 + Servo Motor + Hedefe Kitlenme            ║");
    Serial.println("╚═══════════════════════════════════════════════════════╝");
    delay(2000);
}

void displayMenu() {
    Serial.println("\n┌─ ANA MENU ──────────────────────────────────────────┐");
    Serial.println("│ 1. Radar Başlat (Tarama Modu)                       │");
    Serial.println("│ 2. Hedefe Kitlenme Modu                             │");
    Serial.println("│ 3. Radarı Durdur                                    │");
    Serial.println("│ 4. Servo Kalibrasyonu                               │");
    Serial.println("│ 5. Mesafe Eşiklerini Ayarla                         │");
    Serial.println("│ 6. Sistem Durumu Göster                             │");
    Serial.println("│ 7. Test Modu                                        │");
    Serial.println("└─────────────────────────────────────────────────────┘");
    Serial.print("Seçiminiz (1-7): ");
}

void handleMenuChoice(int choice) {
    switch (choice) {
        case 1:
            radarActive = true;
            trackingMode = false;
            Serial.println("\n✓ Radar Tarama Modu Başlatıldı!");
            break;
        case 2:
            radarActive = true;
            trackingMode = true;
            Serial.println("\n✓ Hedefe Kitlenme Modu Başlatıldı!");
            break;
        case 3:
            radarActive = false;
            digitalWrite(LED_RED, LOW);
            digitalWrite(LED_YELLOW, LOW);
            digitalWrite(LED_GREEN, LOW);
            noTone(BUZZER_PIN);
            Serial.println("\n✓ Radar Durduruldu!");
            displayMenu();
            break;
        case 4:
            calibrateServo();
            displayMenu();
            break;
        case 5:
            setThresholds();
            displayMenu();
            break;
        case 6:
            showStatus();
            displayMenu();
            break;
        case 7:
            runTests();
            displayMenu();
            break;
        default:
            Serial.println("Geçersiz seçim!");
            displayMenu();
    }
}

void calibrateServo() {
    Serial.println("\n--- SERVO KALİBRASYONU ---");
    Serial.println("Servo 0° konumuna getiriliyor...");
    servoPan.write(0);
    delay(2000);
    Serial.println("Servo 90° konumuna getiriliyor...");
    servoPan.write(90);
    delay(2000);
    Serial.println("Servo 180° konumuna getiriliyor...");
    servoPan.write(180);
    delay(2000);
    servoPan.write(90);
    panAngle = 90;
    Serial.println("✓ Kalibrasyon Tamamlandı!");
}

void setThresholds() {
    Serial.println("\n--- MESAFE EŞİKLERİ AYARLA ---");
    Serial.print("Kirmizi esigini (cm) girin (şu an ");
    Serial.print((int)distanceThreshold1);
    Serial.print("): ");
    while (!Serial.available());
    distanceThreshold1 = Serial.parseFloat();
    Serial.println("\nSari esigini (cm) girin (şu an ");
    Serial.print((int)distanceThreshold2);
    Serial.print("): ");
    while (!Serial.available());
    distanceThreshold2 = Serial.parseFloat();
    Serial.println("✓ Esikler Kaydedildi!");
}

void showStatus() {
    Serial.println("\n╔════════════════════════════════════════╗");
    Serial.println("║      SİSTEM DURUMU                    ║");
    Serial.println("╠════════════════════════════════════════╣");
    Serial.print("║ Radar Durumu: ");
    Serial.println(radarActive ? "AÇIK      ║" : "KAPAL     ║");
    Serial.print("║ Mod: ");
    Serial.println(trackingMode ? "İZLEME    ║" : "TARAMA    ║");
    Serial.print("║ Pan Açısı: ");
    Serial.print((int)panAngle);
    Serial.println("°              ║");
    Serial.print("║ Kirmizi Esigi: ");
    Serial.print((int)distanceThreshold1);
    Serial.println(" cm          ║");
    Serial.print("║ Sari Esigi: ");
    Serial.print((int)distanceThreshold2);
    Serial.println(" cm          ║");
    Serial.println("╚════════════════════════════════════════╝");
}

void runTests() {
    Serial.println("\n--- TEST MODU ---");
    Serial.println("LED Test: Kirmizi...");
    digitalWrite(LED_RED, HIGH);
    delay(1000);
    digitalWrite(LED_RED, LOW);
    Serial.println("LED Test: Sari...");
    digitalWrite(LED_YELLOW, HIGH);
    delay(1000);
    digitalWrite(LED_YELLOW, LOW);
    Serial.println("LED Test: Yeşil...");
    digitalWrite(LED_GREEN, HIGH);
    delay(1000);
    digitalWrite(LED_GREEN, LOW);
    Serial.println("Buzzer Test...");
    tone(BUZZER_PIN, 1000);
    delay(500);
    noTone(BUZZER_PIN);
    Serial.println("Servo Test...");
    servoPan.write(0);
    delay(1000);
    servoPan.write(90);
    delay(1000);
    servoPan.write(180);
    delay(1000);
    servoPan.write(90);
    Serial.println("Mesafe Test (5 olcum)...");
    for (int i = 0; i < 5; i++) {
        float dist = getDistance();
        Serial.print("Olcum ");
        Serial.print(i + 1);
        Serial.print(": ");
        Serial.print(dist);
        Serial.println(" cm");
        delay(500);
    }
    Serial.println("✓ Testler Tamamlandi!");
}1:31:11] 🚀 STARTING STRING DUMPER...
[11:31:11] 🔍 Looking for libanogs.so...
[11:31:11] ✅ libanogs.so FOUND at: 0x7afa81a000
[11:31:11] 🎯 Hooking function at: 0x4DF6CC ->> String Comparison Hook
[11:31:11] ✅ HOOK SUCCESSFUL - Ready to dump ALL strings!
[11:31:11] 📍 MAIN HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:11] 📁 Logging to: /sdcard/Android/data/com.tencent.ig/files/all_strings_dump.txt
[11:31:11] ========================================

[11:31:11] 🚀 STARTING STRING DUMPER...
[11:31:11] 🔍 Looking for libanogs.so...
[11:31:12] 🚀 STARTING STRING DUMPER...
[11:31:12] 🔍 Looking for libanogs.so...
[11:31:22] 🚀 STARTING STRING DUMPER...
[11:31:22] 🔍 Looking for libanogs.so...
[11:31:25] 🚀 STARTING STRING DUMPER...
[11:31:25] 🔍 Looking for libanogs.so...
[11:31:30] 🎯 STRING COMPARISON #1
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFD8785C00 ->> 150.109.0.38 vs 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFD8785C00 ->> 150.109.0.38
[11:31:30]     Length: 12
[11:31:30]     String: "150.109.0.38"
[11:31:30]     Hex: 
[11:31:30]       31 35 30 2E 31 30 39 2E 30 2E 33 38 
[11:31:30]       1 5 0 . 1 0 9 . 0 . 3 8 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30]     Length: 7
[11:31:30]     String: "0.0.0.0"
[11:31:30]     Hex: 
[11:31:30]       30 2E 30 2E 30 2E 30 
[11:31:30]       0 . 0 . 0 . 0 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: '1'(0x31) vs '0'(0x30) = diff 1
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning 1)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #2
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFD8785C00 ->> 150.109.29.150 vs 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFD8785C00 ->> 150.109.29.150
[11:31:30]     Length: 14
[11:31:30]     String: "150.109.29.150"
[11:31:30]     Hex: 
[11:31:30]       31 35 30 2E 31 30 39 2E 32 39 2E 31 35 30 
[11:31:30]       1 5 0 . 1 0 9 . 2 9 . 1 5 0 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30]     Length: 7
[11:31:30]     String: "0.0.0.0"
[11:31:30]     Hex: 
[11:31:30]       30 2E 30 2E 30 2E 30 
[11:31:30]       0 . 0 . 0 . 0 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: '1'(0x31) vs '0'(0x30) = diff 1
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning 1)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #3
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFD8785C00 ->> 150.109.0.45 vs 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFD8785C00 ->> 150.109.0.45
[11:31:30]     Length: 12
[11:31:30]     String: "150.109.0.45"
[11:31:30]     Hex: 
[11:31:30]       31 35 30 2E 31 30 39 2E 30 2E 34 35 
[11:31:30]       1 5 0 . 1 0 9 . 0 . 4 5 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30]     Length: 7
[11:31:30]     String: "0.0.0.0"
[11:31:30]     Hex: 
[11:31:30]       30 2E 30 2E 30 2E 30 
[11:31:30]       0 . 0 . 0 . 0 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: '1'(0x31) vs '0'(0x30) = diff 1
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning 1)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #4
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFD8785C00 ->> 119.28.121.174 vs 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFD8785C00 ->> 119.28.121.174
[11:31:30]     Length: 14
[11:31:30]     String: "119.28.121.174"
[11:31:30]     Hex: 
[11:31:30]       31 31 39 2E 32 38 2E 31 32 31 2E 31 37 34 
[11:31:30]       1 1 9 . 2 8 . 1 2 1 . 1 7 4 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0xFFFFFFFFD87859E8 ->> 0.0.0.0
[11:31:30]     Length: 7
[11:31:30]     String: "0.0.0.0"
[11:31:30]     Hex: 
[11:31:30]       30 2E 30 2E 30 2E 30 
[11:31:30]       0 . 0 . 0 . 0 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: '1'(0x31) vs '0'(0x30) = diff 1
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning 1)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #5
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFDC548DD7 ->>  vs 0x5B63E9 ->> :NeedPop
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFDC548DD7 ->> [LONG_STRING]
[11:31:30]     Length: 0
[11:31:30]     String: [TOO LONG OR EMPTY]
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0x5B63E9 ->> :NeedPop
[11:31:30]     Length: 8
[11:31:30]     String: ":NeedPop"
[11:31:30]     Hex: 
[11:31:30]       3A 4E 65 65 64 50 6F 70 
[11:31:30]       : N e e d P o p 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: '.'(0x00) vs ':'(0x3A) = diff -58
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning -58)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #7
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFD8785C00 ->> SetLocaleId:998 vs 0xA21C5 ->> ilc_open_pipe
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFD8785C00 ->> SetLocaleId:998
[11:31:30]     Length: 15
[11:31:30]     String: "SetLocaleId:998"
[11:31:30]     Hex: 
[11:31:30]       53 65 74 4C 6F 63 61 6C 65 49 64 3A 39 39 38 
[11:31:30]       S e t L o c a l e I d : 9 9 8 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0xA21C5 ->> ilc_open_pipe
[11:31:30]     Length: 13
[11:31:30]     String: "ilc_open_pipe"
[11:31:30]     Hex: 
[11:31:30]       69 6C 63 5F 6F 70 65 6E 5F 70 69 70 65 
[11:31:30]       i l c _ o p e n _ p i p e 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: 'S'(0x53) vs 'i'(0x69) = diff -22
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning -22)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #8
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFD424AD721 ->> .apk vs 0x5AAA50 ->> .apk
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFD424AD721 ->> .apk
[11:31:30]     Length: 4
[11:31:30]     String: ".apk"
[11:31:30]     Hex: 
[11:31:30]       2E 61 70 6B 
[11:31:30]       . a p k 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0x5AAA50 ->> .apk
[11:31:30]     Length: 4
[11:31:30]     String: ".apk"
[11:31:30]     Hex: 
[11:31:30]       2E 61 70 6B 
[11:31:30]       . a p k 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: '.'(0x2E) vs '.'(0x2E) = diff 0
[11:31:30]   Iter 1: 'a'(0x61) vs 'a'(0x61) = diff 0
[11:31:30]   Iter 2: 'p'(0x70) vs 'p'(0x70) = diff 0
[11:31:30]   Iter 3: 'k'(0x6B) vs 'k'(0x6B) = diff 0
[11:31:30] 🎯 STRING COMPARISON #8
[11:31:30]   Iter 4: '.'(0x00) vs '.'(0x00) = diff 0
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] ✅ RESULT: Strings are EQUAL (returning 0)
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFD8785C00 ->> SetLocaleId:998 vs 0xA0DFF ->> ilc_close_pipe
[11:31:30] ════════════════════════════════════════

[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFD8785C00 ->> SetLocaleId:998
[11:31:30]     Length: 15
[11:31:30]     String: "SetLocaleId:998"
[11:31:30]     Hex: 
[11:31:30]       53 65 74 4C 6F 63 61 6C 65 49 64 3A 39 39 38 
[11:31:30]       S e t L o c a l e I d : 9 9 8 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0xA0DFF ->> ilc_close_pipe
[11:31:30]     Length: 14
[11:31:30]     String: "ilc_close_pipe"
[11:31:30]     Hex: 
[11:31:30]       69 6C 63 5F 63 6C 6F 73 65 5F 70 69 70 65 
[11:31:30]       i l c _ c l o s e _ p i p e 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: 'S'(0x53) vs 'i'(0x69) = diff -22
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning -22)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #10
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xAA8C52C0 ->> /data/app/~~nbQQUyVAdTGERv0YteqO2g==/com.tencent.ig-qPhv6P9gHgX-Gzz-ilXY_A==/base.apk vs 0xFFFFFFFD424AD6D0 ->> /data/app/~~nbQQUyVAdTGERv0YteqO2g==/com.tencent.ig-qPhv6P9gHgX-Gzz-ilXY_A==/base.apk
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xAA8C52C0 ->> /data/app/~~nbQQUyVAdTGERv0YteqO2g==/com.tencent.ig-qPhv6P9gHgX-Gzz-ilXY_A==/base.apk
[11:31:30]     Length: 85
[11:31:30]     String: "/data/app/~~nbQQUyVAdTGERv0YteqO2g==/com.tencent.ig-qPhv6P9gHgX-Gzz-ilXY_A==/base.apk"
[11:31:30]     Hex: 
[11:31:30]       2F 64 61 74 61 2F 61 70 70 2F 7E 7E 6E 62 51 51 
[11:31:30]       / d a t a / a p p / ~ ~ n b Q Q 
[11:31:30]       55 79 56 41 64 54 47 45 52 76 30 59 74 65 71 4F 
[11:31:30]       U y V A d T G E R v 0 Y t e q O 
[11:31:30]       32 67 3D 3D 2F 63 6F 6D 2E 74 65 6E 63 65 6E 74 
[11:31:30]       2 g = = / c o m . t e n c e n t 
[11:31:30]       2E 69 67 2D 71 50 68 76 36 50 39 67 48 67 58 2D 
[11:31:30]       . i g - q P h v 6 P 9 g H g X - 
[11:31:30]       47 7A 7A 2D 69 6C 58 59 5F 41 3D 3D 2F 62 61 73 
[11:31:30]       G z z - i l X Y _ A = = / b a s 
[11:31:30] 🎯 STRING COMPARISON #10
[11:31:30]       65 2E 61 70 6B 
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30]       e . a p k 
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFD8785C00 ->> SetLocaleId:998 vs 0xA38A2 ->> ilc_recv_pipe
[11:31:30] 
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 2:
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFD424AD6D0 ->> /data/app/~~nbQQUyVAdTGERv0YteqO2g==/com.tencent.ig-qPhv6P9gHgX-Gzz-ilXY_A==/base.apk
[11:31:30]     Function: 0xFFFFFFFFD8785C00 ->> SetLocaleId:998
[11:31:30]     Length: 85
[11:31:30]     Length: 15
[11:31:30]     String: "/data/app/~~nbQQUyVAdTGERv0YteqO2g==/com.tencent.ig-qPhv6P9gHgX-Gzz-ilXY_A==/base.apk"
[11:31:30]     String: "SetLocaleId:998"
[11:31:30]     Hex: 
[11:31:30]     Hex: 
[11:31:30]       2F 64 61 74 61 2F 61 70 70 2F 7E 7E 6E 62 51 51 
[11:31:30]       53 65 74 4C 6F 63 61 6C 65 49 64 3A 39 39 38 
[11:31:30]       / d a t a / a p p / ~ ~ n b Q Q 
[11:31:30]       S e t L o c a l e I d : 9 9 8 
[11:31:30]       55 79 56 41 64 54 47 45 52 76 30 59 74 65 71 4F 
[11:31:30] 
[11:31:30]       U y V A d T G E R v 0 Y t e q O 
[11:31:30]   📝 STRING 2:
[11:31:30]       32 67 3D 3D 2F 63 6F 6D 2E 74 65 6E 63 65 6E 74 
[11:31:30]     Function: 0xA38A2 ->> ilc_recv_pipe
[11:31:30]     Length: 13
[11:31:30]       2 g = = / c o m . t e n c e n t 
[11:31:30]     String: "ilc_recv_pipe"
[11:31:30]     Hex: 
[11:31:30]       2E 69 67 2D 71 50 68 76 36 50 39 67 48 67 58 2D 
[11:31:30]       69 6C 63 5F 72 65 63 76 5F 70 69 70 65 
[11:31:30]       . i g - q P h v 6 P 9 g H g X - 
[11:31:30]       i l c _ r e c v _ p i p e 
[11:31:30]       47 7A 7A 2D 69 6C 58 59 5F 41 3D 3D 2F 62 61 73 
[11:31:30] 
[11:31:30]       G z z - i l X Y _ A = = / b a s 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]       65 2E 61 70 6B 
[11:31:30]   Iter 0: 'S'(0x53) vs 'i'(0x69) = diff -22
[11:31:30]       e . a p k 
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning -22)
[11:31:30] 
[11:31:30] ════════════════════════════════════════

[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: '/'(0x2F) vs '/'(0x2F) = diff 0
[11:31:30]   Iter 1: 'd'(0x64) vs 'd'(0x64) = diff 0
[11:31:30]   Iter 2: 'a'(0x61) vs 'a'(0x61) = diff 0
[11:31:30]   Iter 3: 't'(0x74) vs 't'(0x74) = diff 0
[11:31:30]   Iter 4: 'a'(0x61) vs 'a'(0x61) = diff 0
[11:31:30]   Iter 5: '/'(0x2F) vs '/'(0x2F) = diff 0
[11:31:30]   Iter 6: 'a'(0x61) vs 'a'(0x61) = diff 0
[11:31:30]   Iter 7: 'p'(0x70) vs 'p'(0x70) = diff 0
[11:31:30]   Iter 8: 'p'(0x70) vs 'p'(0x70) = diff 0
[11:31:30]   Iter 9: '/'(0x2F) vs '/'(0x2F) = diff 0
[11:31:30]   Iter 10: '~'(0x7E) vs '~'(0x7E) = diff 0
[11:31:30]   Iter 11: '~'(0x7E) vs '~'(0x7E) = diff 0
[11:31:30]   Iter 12: 'n'(0x6E) vs 'n'(0x6E) = diff 0
[11:31:30]   Iter 13: 'b'(0x62) vs 'b'(0x62) = diff 0
[11:31:30]   Iter 14: 'Q'(0x51) vs 'Q'(0x51) = diff 0
[11:31:30]   Iter 15: 'Q'(0x51) vs 'Q'(0x51) = diff 0
[11:31:30]   Iter 16: 'U'(0x55) vs 'U'(0x55) = diff 0
[11:31:30]   Iter 17: 'y'(0x79) vs 'y'(0x79) = diff 0
[11:31:30]   Iter 18: 'V'(0x56) vs 'V'(0x56) = diff 0
[11:31:30]   Iter 19: 'A'(0x41) vs 'A'(0x41) = diff 0
[11:31:30]   Iter 20: 'd'(0x64) vs 'd'(0x64) = diff 0
[11:31:30]   Iter 21: 'T'(0x54) vs 'T'(0x54) = diff 0
[11:31:30]   Iter 22: 'G'(0x47) vs 'G'(0x47) = diff 0
[11:31:30]   Iter 23: 'E'(0x45) vs 'E'(0x45) = diff 0
[11:31:30]   Iter 24: 'R'(0x52) vs 'R'(0x52) = diff 0
[11:31:30]   Iter 25: 'v'(0x76) vs 'v'(0x76) = diff 0
[11:31:30]   Iter 26: '0'(0x30) vs '0'(0x30) = diff 0
[11:31:30]   Iter 27: 'Y'(0x59) vs 'Y'(0x59) = diff 0
[11:31:30]   Iter 28: 't'(0x74) vs 't'(0x74) = diff 0
[11:31:30]   Iter 29: 'e'(0x65) vs 'e'(0x65) = diff 0
[11:31:30]   Iter 30: 'q'(0x71) vs 'q'(0x71) = diff 0
[11:31:30]   Iter 31: 'O'(0x4F) vs 'O'(0x4F) = diff 0
[11:31:30]   Iter 32: '2'(0x32) vs '2'(0x32) = diff 0
[11:31:30]   Iter 33: 'g'(0x67) vs 'g'(0x67) = diff 0
[11:31:30]   Iter 34: '='(0x3D) vs '='(0x3D) = diff 0
[11:31:30]   Iter 35: '='(0x3D) vs '='(0x3D) = diff 0
[11:31:30]   Iter 36: '/'(0x2F) vs '/'(0x2F) = diff 0
[11:31:30]   Iter 37: 'c'(0x63) vs 'c'(0x63) = diff 0
[11:31:30]   Iter 38: 'o'(0x6F) vs 'o'(0x6F) = diff 0
[11:31:30]   Iter 39: 'm'(0x6D) vs 'm'(0x6D) = diff 0
[11:31:30]   Iter 40: '.'(0x2E) vs '.'(0x2E) = diff 0
[11:31:30]   Iter 41: 't'(0x74) vs 't'(0x74) = diff 0
[11:31:30]   Iter 42: 'e'(0x65) vs 'e'(0x65) = diff 0
[11:31:30]   Iter 43: 'n'(0x6E) vs 'n'(0x6E) = diff 0
[11:31:30]   Iter 44: 'c'(0x63) vs 'c'(0x63) = diff 0
[11:31:30]   Iter 45: 'e'(0x65) vs 'e'(0x65) = diff 0
[11:31:30]   Iter 46: 'n'(0x6E) vs 'n'(0x6E) = diff 0
[11:31:30]   Iter 47: 't'(0x74) vs 't'(0x74) = diff 0
[11:31:30]   Iter 48: '.'(0x2E) vs '.'(0x2E) = diff 0
[11:31:30]   Iter 49: 'i'(0x69) vs 'i'(0x69) = diff 0
[11:31:30]   Iter 50: 'g'(0x67) vs 'g'(0x67) = diff 0
[11:31:30]   Iter 51: '-'(0x2D) vs '-'(0x2D) = diff 0
[11:31:30]   Iter 52: 'q'(0x71) vs 'q'(0x71) = diff 0
[11:31:30]   Iter 53: 'P'(0x50) vs 'P'(0x50) = diff 0
[11:31:30]   Iter 54: 'h'(0x68) vs 'h'(0x68) = diff 0
[11:31:30]   Iter 55: 'v'(0x76) vs 'v'(0x76) = diff 0
[11:31:30]   Iter 56: '6'(0x36) vs '6'(0x36) = diff 0
[11:31:30]   Iter 57: 'P'(0x50) vs 'P'(0x50) = diff 0
[11:31:30]   Iter 58: '9'(0x39) vs '9'(0x39) = diff 0
[11:31:30]   Iter 59: 'g'(0x67) vs 'g'(0x67) = diff 0
[11:31:30]   Iter 60: 'H'(0x48) vs 'H'(0x48) = diff 0
[11:31:30]   Iter 61: 'g'(0x67) vs 'g'(0x67) = diff 0
[11:31:30]   Iter 62: 'X'(0x58) vs 'X'(0x58) = diff 0
[11:31:30]   Iter 63: '-'(0x2D) vs '-'(0x2D) = diff 0
[11:31:30]   Iter 64: 'G'(0x47) vs 'G'(0x47) = diff 0
[11:31:30]   Iter 65: 'z'(0x7A) vs 'z'(0x7A) = diff 0
[11:31:30]   Iter 66: 'z'(0x7A) vs 'z'(0x7A) = diff 0
[11:31:30] 🎯 STRING COMPARISON #11
[11:31:30]   Iter 67: '-'(0x2D) vs '-'(0x2D) = diff 0
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30]   Iter 68: 'i'(0x69) vs 'i'(0x69) = diff 0
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFDC40D844 ->> CloseDevInfoCollectEx:2 vs 0x5B091B ->> StartGP6
[11:31:30]   Iter 69: 'l'(0x6C) vs 'l'(0x6C) = diff 0
[11:31:30] ════════════════════════════════════════
[11:31:30]   Iter 70: 'X'(0x58) vs 'X'(0x58) = diff 0
[11:31:30]   📝 STRING 1:
[11:31:30]   Iter 71: 'Y'(0x59) vs 'Y'(0x59) = diff 0
[11:31:30]     Function: 0xFFFFFFFFDC40D844 ->> CloseDevInfoCollectEx:2
[11:31:30]   Iter 72: '_'(0x5F) vs '_'(0x5F) = diff 0
[11:31:30]     Length: 23
[11:31:30]   Iter 73: 'A'(0x41) vs 'A'(0x41) = diff 0
[11:31:30]     String: "CloseDevInfoCollectEx:2"
[11:31:30]   Iter 74: '='(0x3D) vs '='(0x3D) = diff 0
[11:31:30]     Hex: 
[11:31:30]   Iter 75: '='(0x3D) vs '='(0x3D) = diff 0
[11:31:30]       43 6C 6F 73 65 44 65 76 49 6E 66 6F 43 6F 6C 6C 
[11:31:30]   Iter 76: '/'(0x2F) vs '/'(0x2F) = diff 0
[11:31:30]       C l o s e D e v I n f o C o l l 
[11:31:30]   Iter 77: 'b'(0x62) vs 'b'(0x62) = diff 0
[11:31:30]   Iter 78: 'a'(0x61) vs 'a'(0x61) = diff 0
[11:31:30]       65 63 74 45 78 3A 32 
[11:31:30]   Iter 79: 's'(0x73) vs 's'(0x73) = diff 0
[11:31:30]       e c t E x : 2 
[11:31:30]   Iter 80: 'e'(0x65) vs 'e'(0x65) = diff 0
[11:31:30] 
[11:31:30]   Iter 81: '.'(0x2E) vs '.'(0x2E) = diff 0
[11:31:30]   📝 STRING 2:
[11:31:30]   Iter 82: 'a'(0x61) vs 'a'(0x61) = diff 0
[11:31:30]     Function: 0x5B091B ->> StartGP6
[11:31:30]   Iter 83: 'p'(0x70) vs 'p'(0x70) = diff 0
[11:31:30]     Length: 8
[11:31:30]   Iter 84: 'k'(0x6B) vs 'k'(0x6B) = diff 0
[11:31:30]     String: "StartGP6"
[11:31:30]   Iter 85: '.'(0x00) vs '.'(0x00) = diff 0
[11:31:30]     Hex: 
[11:31:30] ✅ RESULT: Strings are EQUAL (returning 0)
[11:31:30]       53 74 61 72 74 47 50 36 
[11:31:30] ════════════════════════════════════════

[11:31:30]       S t a r t G P 6 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: 'C'(0x43) vs 'S'(0x53) = diff -16
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning -16)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #12
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0xFFFFFFFFDC40D844 ->> CloseDevInfoCollectEx:2 vs 0x5B056B ->> StartGP3
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0xFFFFFFFFDC40D844 ->> CloseDevInfoCollectEx:2
[11:31:30]     Length: 23
[11:31:30]     String: "CloseDevInfoCollectEx:2"
[11:31:30]     Hex: 
[11:31:30]       43 6C 6F 73 65 44 65 76 49 6E 66 6F 43 6F 6C 6C 
[11:31:30]       C l o s e D e v I n f o C o l l 
[11:31:30]       65 63 74 45 78 3A 32 
[11:31:30]       e c t E x : 2 
[11:31:30] 
[11:31:30]   📝 STRING 2:
[11:31:30]     Function: 0x5B056B ->> StartGP3
[11:31:30]     Length: 8
[11:31:30]     String: "StartGP3"
[11:31:30]     Hex: 
[11:31:30]       53 74 61 72 74 47 50 33 
[11:31:30]       S t a r t G P 3 
[11:31:30] 
[11:31:30] 🔍 COMPARISON PROCESS:
[11:31:30]   Iter 0: 'C'(0x43) vs 'S'(0x53) = diff -16
[11:31:30] ❌ RESULT: Strings are DIFFERENT (returning -16)
[11:31:30] ════════════════════════════════════════

[11:31:30] 🎯 STRING COMPARISON #14
[11:31:30] 🎯 STRING COMPARISON #14
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 📍 HOOKED FUNCTION: 0x4DF6CC ->> String Comparison Hook
[11:31:30] 🔍 COMPARING: 0x131BE9A0 ->> true vs 0xA38C9 ->> true
[11:31:30] 🔍 COMPARING: 0xFFFFFFFD424AD640 ->> regist_mtp_info_receiver:0x78808dec60 vs 0xA21C5 ->> ilc_open_pipe
[11:31:30] ════════════════════════════════════════
[11:31:30] ════════════════════════════════════════
[11:31:30]   📝 STRING 1:
[11:31:30]   📝 STRING 1:
[11:31:30]     Function: 0x131BE9A0 ->> true
[11:31:30]     Function: 0xFFFFFFFD424AD640 ->> regist_mtp_info_receiver:0x78808dec60
[11:31:30]     Length: 4
[11:31:30]     Length: 37
[11:31:30]     String: "true"
[11:31:30]     String: "regist_mtp_info_receiver:0x78808dec60"
[11:31:30]     Hex: 
[11:31:30]     Hex: 
[11:31:30]       74 72 75 65 
[11:31:30]       72 65 67 69 73 74 5F 6D 74 70 5F 69 6E 66 6F 5F 
[11:31:30]       t r u e 
[11:31:30]       r e g i s t _ m t p _ i n f o
