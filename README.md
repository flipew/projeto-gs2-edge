Filipe Prado Menezes - RM98765
Pedro Henrique Salvitti - RM88166


Link do Projeto: https://wokwi.com/projects/380963897669274625


#Código do Projeto#

#include <WiFi.h>
#include <ArduinoJson.h>
#include <DHTesp.h>
#include <PubSubClient.h>
#include <ESP32Servo.h>

// Configurações de WiFi
const char *SSID = "Wokwi-GUEST";
const char *PASSWORD = "";  

const char *BROKER_MQTT = "broker.hivemq.com";
const int BROKER_PORT = 1883;
const char *ID_MQTT = "RAFA_mqtt";
const char *TOPIC_SUBSCRIBE_LED1 = "fiap/iot/led1";
const char *TOPIC_SUBSCRIBE_LED2 = "fiap/iot/led1";
const char *TOPIC_PUBLISH_TEMP_HUMI = "fiap/iot/temphumi";
const char *TOPIC_SUBSCRIBE_SERVO1 = "fiap/iot/servo1";
const char *TOPIC_SUBSCRIBE_SERVO2 = "fiap/iot/servo2";

// Configurações de Hardware
#define PIN_SERVO1 5 
#define PIN_SERVO2 4
#define PIN_DHT 12
#define PIN_LED1 26
#define PIN_LED2 27
#define PUBLISH_DELAY 2000

// Variáveis globais
WiFiClient espClient;
PubSubClient MQTT(espClient);
DHTesp dht;
unsigned long publishUpdate = 0;
TempAndHumidity sensorValues;
const int TAMANHO = 200;
Servo servo1;
Servo servo2;

// Protótipos de funções
void updateSensorValues();
void initWiFi();
void initMQTT();
void callbackMQTT(char *topic, byte *payload, unsigned int length);
void reconnectMQTT();
void reconnectWiFi();
void checkWiFIAndMQTT();

void updateSensorValues() {
  sensorValues = dht.getTempAndHumidity();
}

void initWiFi() {
  Serial.print("Conectando com a rede: ");
  Serial.println(SSID);
  WiFi.begin(SSID, PASSWORD);

  while (WiFi.status() != WL_CONNECTED) {
    delay(100);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Conectado com sucesso: ");
  Serial.println(SSID);
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());
}

void initMQTT() {
  MQTT.setServer(BROKER_MQTT, BROKER_PORT);
  MQTT.setCallback(callbackMQTT);
}
 
 void callbackMQTT(char *topic, byte *payload, unsigned int length) {
  String msg = String((char*)payload).substring(0, length);
  
  Serial.printf("Mensagem recebida via MQTT: %s do tópico: %s\n", msg.c_str(), topic);

  if (strcmp(topic, TOPIC_SUBSCRIBE_SERVO1) == 0) {
    int angle = atoi(msg.c_str());
    if (angle >= 0 && angle <= 180) {
      servo1.write(angle);
      Serial.printf("Servo 1 movido para o ângulo %d\n", angle);
    } else {
      Serial.println("Ângulo fora do intervalo válido (0-180) para o Servo 1");
    }
  } else if (strcmp(topic, TOPIC_SUBSCRIBE_SERVO2) == 0) {
    int angle = atoi(msg.c_str());
    if (angle >= 0 && angle <= 180) {
      servo2.write(angle);
      Serial.printf("Servo 2 movido para o ângulo %d\n", angle);
    } else {
      Serial.println("Ângulo fora do intervalo válido (0-180) para o Servo 2");
    }
  }
}

void reconnectMQTT() {
  while (!MQTT.connected()) {
    Serial.print("Tentando conectar com o Broker MQTT: ");
    Serial.println(BROKER_MQTT);

    if (MQTT.connect(ID_MQTT)) {
      Serial.println("Conectado ao broker MQTT!");
      MQTT.subscribe(TOPIC_SUBSCRIBE_SERVO1);
      MQTT.subscribe(TOPIC_SUBSCRIBE_SERVO2);
    } else {
      Serial.println("Falha na conexão com MQTT. Tentando novamente em 2 segundos.");
      delay(2000);
    }
  }
}

void checkWiFIAndMQTT() {
  if (WiFi.status() != WL_CONNECTED) reconnectWiFi();
  if (!MQTT.connected()) reconnectMQTT();
}

void reconnectWiFi(void) {
  if (WiFi.status() == WL_CONNECTED)
    return;

  WiFi.begin(SSID, PASSWORD); // Conecta na rede WI-FI

  while (WiFi.status() != WL_CONNECTED) {
    delay(100);
    Serial.print(".");
  }

  Serial.println();
  Serial.print("Wifi conectado com sucesso");
  Serial.print(SSID);
  Serial.println("IP: ");
  Serial.println(WiFi.localIP());
}

void setup() {
  Serial.begin(115200);

  initWiFi();
  initMQTT();
  servo1.attach(PIN_SERVO1);
  servo2.attach(PIN_SERVO2);
  pinMode(PIN_LED1, OUTPUT);
  pinMode(PIN_LED2, OUTPUT);
  digitalWrite(PIN_LED1, LOW);
  digitalWrite(PIN_LED2, LOW);
  dht.setup(PIN_DHT, DHTesp::DHT22);
}

void loop() {
  checkWiFIAndMQTT();
  MQTT.loop();

  if ((millis() - publishUpdate) >= PUBLISH_DELAY) {
    publishUpdate = millis();
    updateSensorValues();

    if (!isnan(sensorValues.temperature) && !isnan(sensorValues.humidity)) {
      StaticJsonDocument<TAMANHO> doc;
      doc["temperatura"] = sensorValues.temperature;
      doc["umidade"] = sensorValues.humidity;

      char buffer[TAMANHO];
      serializeJson(doc, buffer);
      MQTT.publish(TOPIC_PUBLISH_TEMP_HUMI, buffer);
      Serial.println(buffer);
    }
  }
}
/*
  TempAndHumidity data = dht.getTempAndHumidity(); 
  if (sensorValues.temperature >= 35 && sensorValues.temperature < 40) { 
    digitalWrite(PIN_LED1, HIGH);
    digitalWrite(PIN_LED2, LOW);
    servo1.write(90); 
    servo2.write(0); 
  }

  else if (sensorValues.temperature >= 40) { 
    digitalWrite(PIN_LED1, LOW);
    digitalWrite(PIN_LED2, HIGH);
    servo1.write(90); 
    servo2.write(90); 
  }

  else { 
    digitalWrite(PIN_LED1, LOW);
    digitalWrite(PIN_LED2, HIGH);
    servo1.write(0);
    servo2.write(0); 
  }
}
*/
