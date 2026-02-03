# test1
#include <WiFi.h>
#include <MQTT.h>
#include <Stepper.h>

const char ssid[] = "@JumboPlusIoT";
const char pass[] = "group014";
// 
const char mqtt_broker[] = "test.mosquitto.org";
const char mqtt_client_id[] = "esp32AI14";
int MQTT_PORT = 1883;
unsigned long lastMsgTime = 0; // เก็บเวลาล่าสุดที่ส่งข้อมูล
const long interval = 1000;    // ระยะเวลาหน่วง (1000ms = 1 วินาที)
// --- Topics ---
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

// --- Global Variables ---
bool autoMode = true;      
int motorState = 0; 
bool autoStarted = true; 

bool btnLast = LOW;
bool lock = false; // เริ่มต้นควร Lock หรือ Unlock? (ตั้งเป็น false ตามโค้ดเดิม)

long currentAngle = 0;      
int ldr_value = 0;

// --- Function Declarations ---
void reset(); // ประกาศหัวฟังก์ชันไว้ก่อนเรียกใช้

void messageReceived(String &topic, String &payload) { 
  

  if (topic == mqtt_mode) { 
    Serial.println("Incoming: " + topic + " - " + payload);
    autoMode = !autoMode;
  } 
  
  // แปลง payload เป็น int เพื่อกำหนด motorState
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
    lock = !lock; // แก้ไข: ใช้ = เพื่อกำหนดค่าสลับ (Toggle)
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
   }
   
   // แก้ไขชื่อตัวแปรให้ตรง (autoMode, autoStarted)
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

  // *** สำคัญ: ต้อง Subscribe หัวข้อที่ต้องการฟัง ***
  client.subscribe("group14/test1/#"); // Subscribe ทุกหัวข้อภายใต้ path นี้
  // หรือจะเลือก Subscribe ทีละอัน:
  // client.subscribe(mqtt_mode);
  // client.subscribe(mqtt_motor);
  // client.subscribe(mqtt_reset);
  // client.subscribe(mqtt_lock);
}

void setup() {
  Serial.begin(9600);
  WiFi.begin(ssid, pass);
  
  myStepper.setSpeed(10); 
  pinMode(LED, OUTPUT);
  pinMode(BTN, INPUT);

  // *** สำคัญ: ผูกฟังก์ชันรับข้อความ ***
  client.begin(mqtt_broker, MQTT_PORT, net);
  client.onMessage(messageReceived);
  
  connect();
}

void loop() {
  client.loop();
  if (!client.connected()) connect();

  /* ===== ปุ่มกด ===== */
  bool btnNow = digitalRead(BTN);
  if (btnNow == HIGH && btnLast == LOW) { // ตรวจจับขอบขาขึ้น (Rising Edge)
    autoMode = !autoMode; 
    // Serial.println(autoMode ? "Mode: Auto" : "Mode: Manual");
    delay(50); // Debounce เล็กน้อย
  }
  btnLast = btnNow; // *** สำคัญ: อัปเดตสถานะปุ่มเดิม ***

  /* ===== อ่าน LDR ===== */
  int raw_value = analogRead(LDR);
  ldr_value = map(raw_value, 0, 4095, 0, 1023);
  /* ===== Auto Mode Logic ===== */
  if (autoMode) {
    if (!autoStarted) {
      autoStarted = true;
    }
    
    // Logic: มืดมาก -> 1, สว่าง -> 2 (แก้ไขช่วงเงื่อนไข)
    if (ldr_value > 3500 && ldr_value<4500) { // มืด
      motorState = 1;    
    }
    else if(ldr_value < 3500&& ldr_value > 2900) { // สว่าง (แก้จาก >2900 && <2500 ที่เป็นไปไม่ได้)
      motorState = 2;  
    }
    else {
      motorState = 0; // ช่วงกลางๆ ให้หยุด
    }
  }
  else {
    if(autoStarted){ // แก้คำผิด autostarted -> autoStarted
      motorState = 0;
      autoStarted = false;   
    }
  }

  /* ===== ควบคุมมอเตอร์จริง ===== */
  // หมายเหตุ: myStepper.step() เป็น blocking function (โปรแกรมจะรอจนหมุนเสร็จ)
  // ถ้า motorState ค้างอยู่ที่ 1 มันจะหมุนเพิ่มทีละ 90 องศา ไปเรื่อยๆ ทุกรอบ Loop
  
  if (motorState == 1 && lock == false) { 
    client.publish(mqtt_status, "Working");
    digitalWrite(LED, HIGH);
    
    myStepper.step(512); // หมุน 90 องศา
    currentAngle += 90; 
    
  } else if (motorState == 2 && lock == false) {
    client.publish(mqtt_status, "Reverse"); // ใน code เดิมเขียน Stop แต่ทำงานเหมือน Reverse?
    digitalWrite(LED, LOW);
    
    myStepper.step(-512); // หมุนกลับ 90 องศา
    currentAngle -= 90; 
    // Serial.println("-90");
    // Serial.println("angle : "+currentAngle);
  } else if (motorState == 0 && lock == false) {
    // ตัดไฟมอเตอร์เพื่อประหยัดพลังงานและกันร้อน
    digitalWrite(23, LOW);
    digitalWrite(21, LOW);
    digitalWrite(22, LOW);
    digitalWrite(19, LOW);
    // Serial.println("+0");
    // Serial.println("angle : "+currentAngle);
  }

  // Serial.println("Publishing MQTT data...");
    client.publish(mqtt_angle, String(currentAngle)); 
    client.publish(mqtt_lockstatus, lock ? "lock" : "unclock"); 
    client.publish(mqtt_ldr, String(ldr_value)); 
    client.publish(mqtt_modestatus, autoMode ? "Auto" : "Manual"); 

}
