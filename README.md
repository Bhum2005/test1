#include <WiFi.h>
#include <MQTT.h>
#include <Stepper.h>

const char ssid[] = "@JumboPlusIoT";
const char pass[] = "group014";

const char mqtt_broker[] = "test.mosquitto.org";
const char mqtt_client_id[] = "esp32AI14";
int MQTT_PORT = 1883;
unsigned long lastMsgTime = 0; 
const long interval = 1000;    
const char mqtt_status[]     = "group14/test1/status";
const char mqtt_mode[]       = "group14/test1/mode";
const char mqtt_modestatus[]       = "group14/test1/modestatus";
const char mqtt_motor[]      = "group14/test1/motor";
const char mqtt_motorstatus[]      = "group14/test1/motorstatus";
const char mqtt_ldr[]        = "group14/test1/ldr";
const char mqtt_angle[]      = "group14/test1/angle";
const char mqtt_reset[]      = "group14/test1/reset";
const char mqtt_lock[]       = "group14/test1/lock";
const char mqtt_lockstatus[] = "group14/test1/lockstatus";

#define LDR     32
#define LED     17
#define BTN     34

const int stepsPerRevolution = 2048; 
Stepper myStepper(stepsPerRevolution, 23, 22, 21, 19);

WiFiClient net;
MQTTClient client;

bool autoMode = true;      
int motorState = 0; 
bool autoStarted = true; 

bool btnLast = LOW;
bool lock = false;
// --- ตัวแปรเก็บค่า ---
long totalSteps = 0;      // เก็บจำนวน Step ทั้งหมดที่หมุนไป (เป็นค่าติดลบได้)
float currentAngle = 0.0; /
     
int ldr_value = 0;

void messageReceived(String &topic, String &payload) { 
  

  if (topic == mqtt_mode) { 
    Serial.println("Incoming: " + topic + " - " + payload);
    autoMode = !autoMode;
  } 
  

  if (topic == mqtt_motor && !autoMode) {  
    Serial.println("Incoming: " + topic + " - " + payload);
    motorState = payload.toInt(); 
  }
  
  if (topic == mqtt_reset) { 
    Serial.println("Incoming: " + topic + " - " + payload);
    if(payload == "reset"){ reset(); }
  }  
  
  if (topic == mqtt_lock) { 
    Serial.println("Incoming: " + topic + " - " + payload);
    lock = !lock; 
  }  
}

void reset() {
  if (currentAngle != 0) {
      long angleDiff = 0 - currentAngle;
      Serial.print("Returning: ");
      Serial.println(angleDiff);

      // คำนวณ Step
      int stepsToMove = (int)((float)angleDiff / 360.0 * stepsPerRevolution);
      myStepper.step(stepsToMove);
      
      currentAngle = 0;
      totalstep=0;
   }
   
   if(autoMode){
    autoMode = false;
    autoStarted = false;
   }
   motorState = 0;
}

void connect() {
  Serial.print("Connecting to WiFi...");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");

  Serial.print("Connecting to MQTT...");
  while (!client.connect(mqtt_client_id)) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nMQTT Connected.");

  client.subscribe("group14/test1/#");
}

void setup() {
  Serial.begin(9600);
  WiFi.begin(ssid, pass);
  
  myStepper.setSpeed(10); 
  pinMode(LED, OUTPUT);
  pinMode(BTN, INPUT);

 
  client.begin(mqtt_broker, MQTT_PORT, net);
  client.onMessage(messageReceived);
  delay(1000);
  connect();
}

void loop() {
  client.loop();
  if (!client.connected()) connect();

  bool btnNow = digitalRead(BTN);
  if (btnNow == HIGH && btnLast == LOW) {
    autoMode = !autoMode; 

    delay(50);
  }
  btnLast = btnNow; 
  int stepSize = 32;
  
  int raw_value = analogRead(LDR);
  ldr_value = map(raw_value, 0, 4095, 0, 1023);

  if (autoMode) {
    if (!autoStarted) {
      autoStarted = true;
    }
    
  
    if (ldr_value > 3500 && ldr_value<4500) { 
      motorState = 1;    
    }
    else if(ldr_value < 3500&& ldr_value > 2900) { 
      motorState = 2;  
    }
    else {
      motorState = 0; 
    }
  }
  else {
    if(autoStarted){ 
      motorState = 0;
      autoStarted = false;   
    }
  }

  
  if (motorState == 1 && lock == false) { 
    client.publish(mqtt_status, "Working");
    digitalWrite(LED, HIGH);
    
    myStepper.step(stepSize);
    totalSteps += stepSize;
    currentAngle = ((float)totalSteps / stepsPerRevolution) * 360.0;
    
  } else if (motorState == 2 && lock == false) {
    client.publish(mqtt_status, "Reverse"); 
    digitalWrite(LED, LOW);
    
    myStepper.step(-stepSize);
    totalSteps += stepSize;
    currentAngle = ((float)totalSteps / stepsPerRevolution) * 360.0;

  } else if (motorState == 0 && lock == false) {

    digitalWrite(23, LOW);
    digitalWrite(21, LOW);
    digitalWrite(22, LOW);
    digitalWrite(19, LOW);

  }

    client.publish(mqtt_angle, String(currentAngle)); 
    client.publish(mqtt_lockstatus, lock ? "lock" : "unclock"); 
    client.publish(mqtt_ldr, String(ldr_value)); 
    client.publish(mqtt_modestatus, autoMode ? "Auto" : "Manual"); 

}

#include <WiFi.h>
#include <PubSubClient.h>
#include <Wire.h>
#include <LCD_I2C.h>

// --- 1. ตั้งค่า WiFi และ MQTT ---
const char ssid[] = "@JumboPlusIoT";
const char pass[] = "group014";
const char mqtt_broker[] = "test.mosquitto.org";
const char mqtt_client_id[] = "esp32_group14_auto";// เปลี่ยนให้ไม่ซ้ำกับคนอื่น

WiFiClient espClient;
PubSubClient client(espClient);

// --- 2. ตั้งค่าอุปกรณ์ Hardware ---
LCD_I2C lcd(0x27, 16, 2); 

// const int pinEN1 = 12; 
const int pinIN1 = 25; 
const int pinIN2 = 23; 
const int ledStatus = 18; 
const int buttonPin = 19; 

// --- 3. ตัวแปรสถานะและ Debounce ---
int mode = 0;           // 0:Stop, 1:Reverse, 2:Forward
int lastMoveMode = 1;   // เก็บสถานะการหมุนล่าสุด (เริ่มต้นให้เป็น 1)
bool lastButtonState = HIGH;
unsigned long lastDebounceTime = 0;
unsigned long debounceDelay = 50;

// ฟังก์ชันสั่งการมอเตอร์และอัปเดตสถานะ (LCD + MQTT + LED)
void updateAction(int newMode) {
  mode = newMode;
  lcd.clear();
  lcd.setCursor(0, 0);

  if (mode == 1) { // ทวนเข็ม
   digitalWrite(pinIN1, HIGH); digitalWrite(pinIN2, LOW);
    digitalWrite(ledStatus, HIGH);
    lcd.print("Motor: REVERSE");
    lcd.setCursor(0, 1); lcd.print("LED: ON");
    lastMoveMode = 1;
    client.publish("esp32/motor/status", "REVERSE");
    client.publish("esp32/led/status", "ON");
  } 
  else if (mode == 2) { // ตามเข็ม
     digitalWrite(pinIN1, LOW); digitalWrite(pinIN2, HIGH);
    digitalWrite(ledStatus, HIGH);
    lcd.print("Motor: FORWARD");
    lcd.setCursor(0, 1); lcd.print("LED: ON");
    lastMoveMode = 2;
    client.publish("esp32/motor/status", "FORWARD");
    client.publish("esp32/led/status", "ON");
  } 
  else { // หยุด (mode 3 หรือค่าอื่นๆ)
    digitalWrite(pinIN1, LOW); digitalWrite(pinIN2, LOW);
    digitalWrite(ledStatus, LOW);
    lcd.print("Motor: STOPPED");
    lcd.setCursor(0, 1); lcd.print("LED: OFF");
    client.publish("esp32/motor/status", "STOPPED");
    client.publish("esp32/led/status", "OFF");
    mode = 3; 
  }
}

// ฟังก์ชันรับข้อความจาก MQTT
void callback(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (int i = 0; i < length; i++) message += (char)payload[i];
  
  if (String(topic) == "esp32/motor/control") {
    if (message == "FORWARD") updateAction(2);
    else if (message == "REVERSE") updateAction(1);
    else if (message == "STOP") updateAction(3);
  }
  else if (String(topic) == "esp32/led/control") {
    if (message == "ON") {
      // เงื่อนไข: ให้มอเตอร์หมุนกลับด้านจากครั้งก่อนหน้า
      if (lastMoveMode == 1) updateAction(2);
      else updateAction(1);
    }
    else if (message == "OFF") {
      updateAction(3);
    }
  }
}

void setup() {
  Serial.begin(115200);
  Wire.begin(21, 22);
  
  // เริ่มต้น LCD
  lcd.begin(); 
  lcd.backlight();
  lcd.print("WiFi Connecting");

  // ขามอเตอร์และปุ่ม
  pinMode(pinIN1, OUTPUT); pinMode(pinIN2, OUTPUT);
  pinMode(ledStatus, OUTPUT); 
  pinMode(buttonPin, INPUT_PULLUP);

  // เชื่อมต่อ WiFi
  WiFi.begin(ssid, pass);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  lcd.clear();
  lcd.print("WiFi Connected");
  client.setServer(mqtt_broker, 1883);
  client.setCallback(callback);
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    if (client.connect(mqtt_client_id)) {
      Serial.println("connected");
      client.subscribe("esp32/motor/control");
      client.subscribe("esp32/led/control");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      delay(5000);
    }
  }
}

void loop() {
  if (!client.connected()) reconnect();
  client.loop();

  // ระบบปุ่มกด Physical Button (วนโหมด 1-2-3)
  int reading = digitalRead(buttonPin);
  if (reading != lastButtonState) {
    lastDebounceTime = millis();
  }

  if ((millis() - lastDebounceTime) > debounceDelay) {
    static bool buttonPressed = false;
    if (reading == LOW && !buttonPressed) {
      buttonPressed = true;
      mode++;
      if (mode > 3) mode = 1;
      updateAction(mode);
    } else if (reading == HIGH) {
      buttonPressed = false;
    }
  }
  lastButtonState = reading;
}

#include <ESP32Servo.h>

Servo myservo;
const int ldrPin = 34;
const int servoPin = 18;

void setup() {
  Serial.begin(115200);
  myservo.attach(servoPin);
}

void loop() {
  int ldrValue = analogRead(ldrPin); // อ่านค่า 0-4095

  // กรณี A: แสงมาก = หมุนเร็วทางขวา, แสงน้อย = หยุด
  // map ค่า 0-4095 ไปเป็น 90-180 (90 คือหยุด, 180 คือเร็วสุด)
  int speedVal = map(ldrValue, 0, 4095, 90, 180);

  myservo.write(speedVal);

  Serial.print("LDR: ");
  Serial.print(ldrValue);
  Serial.print(" | Servo Val: ");
  Serial.println(speedVal);
  
  delay(50);
}

const int ldrPin = 34;      // ขาที่ต่อ LDR (ต้องเป็นขาที่อ่าน Analog ได้)
const int motorPin = 18;    // ขาที่ต่อเข้า Driver Motor (ENA หรือ IN1)

void setup() {
  Serial.begin(115200);
  pinMode(motorPin, OUTPUT);
}

void loop() {
  // 1. อ่านค่าจาก LDR (ค่าที่ได้จะอยู่ระหว่าง 0 - 4095)
  int ldrValue = analogRead(ldrPin);

  // 2. แปลงค่า (Mapping)
  // map(ค่าดิบ, ค่าต่ำสุดเดิม, ค่าสูงสุดเดิม, ค่าต่ำสุดใหม่, ค่าสูงสุดใหม่);
  // แปลงจาก 0-4095 ให้เหลือ 0-255 สำหรับ PWM
  int motorSpeed = map(ldrValue, 0, 4095, 0, 255);

  // *ลูกเล่นเสริม: ถ้าอยากให้ ยิ่งมืด ยิ่งหมุนเร็ว ให้สลับเลขข้างหลัง
  // int motorSpeed = map(ldrValue, 0, 4095, 255, 0);

  // 3. สั่งงาน Motor
  analogWrite(motorPin, motorSpeed); // สำหรับ ESP32 เวอร์ชั่นใหม่ใช้คำสั่งนี้ได้เลย

  // แสดงผลดูค่า
  Serial.print("LDR: ");
  Serial.print(ldrValue);
  Serial.print(" | Speed: ");
  Serial.println(motorSpeed);

  delay(50);
}
