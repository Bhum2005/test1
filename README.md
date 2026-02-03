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
