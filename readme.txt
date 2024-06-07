ANDRÉ DE SOUSA NEVES – RM 553515
THAÍS GONÇALVES LEONCIO – RM 553892

========================================================

O projeto consiste num sistema de sensores para monitorar e coletar dados dos oceanos para futuros usos e pesquisas,a aplicação necessita das bibliotecas Adafruit SSD1306, DHT sensor library, RTClib ,PubSubClient e ArduinoJson sendo necessários os componetes: DHT22, ESP32, Potenciometro,Sensor de distância ultrassônico e uma Tela OLED monocromática.

Para seu uso é necessário apenas montar e colocar no lugar que ira monitorar que ele mandará informaçoes que captar a cada 4 segundos.

=========================================================
Código do Projeto

#include <DHT.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_GFX.h>
#include <Wire.h>
#include <PubSubClient.h>
#include <WiFi.h>
#include <ArduinoJson.h>
#define pino 36

int vso = 0 ;


const char *SSID = "Wokwi-GUEST";
const char *PASSWORD = ""; 


const char *BROKER_MQTT = "mqtt.tago.io";
const int BROKER_PORT = 1883;
const char *ID_MQTT = "esp32_mqtt";
const char *TOPIC_SUBSCRIBE_LED = "fiap/iot/led";
const char *TOPIC_PUBLISH_TEMP_HUMI = "tago/data/post";
const char *USERNAME = ""; 
const char *PASSWORD_MQTT = "45782f3f-438c-4903-ac2c-8fa5c4916061";  
const char *TOPIC_SEND_VARIABLE = "tago/data/post"; 

#define SCREEN_WIDTH 128 
#define SCREEN_HEIGHT 64 
#define SCREEN_ADDRESS 0x3C 
#define OLED_RESET -1 
#define PUBLISH_DELAY 2000
#define LEN(arr) ((int)(sizeof(arr) / sizeof(arr)[0]))
#define pinTrig 15 
#define pinEcho 2 

#define pinDHT 13 
#define DHTType 22 

WiFiClient espClient;
PubSubClient MQTT(espClient);
unsigned long publishUpdate = 0;
const int TAMANHO = 200;


Adafruit_SSD1306 oled(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET); 
DHT dht(pinDHT, DHTType); 


float tempoEcho = 0; 
float umidade = 0;
float temperatura = 0; 
String mare = "";
const float velocidadeSom = 340; 

void initWiFi();
void initMQTT();
void callbackMQTT(char *topic, byte *payload, unsigned int length);
void reconnectMQTT();
void reconnectWiFi();
void checkWiFIAndMQTT();

float medeDistancia(); 

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
  
  StaticJsonDocument<TAMANHO> json;
  DeserializationError error = deserializeJson(json, msg);
  
  if (error) {
    Serial.println("Falha na deserialização do JSON.");
    return;
  }

}

void reconnectMQTT() {
  while (!MQTT.connected()) {
    Serial.print("Tentando conectar com o Broker MQTT: ");
    Serial.println(BROKER_MQTT);

    if (MQTT.connect(ID_MQTT, USERNAME, PASSWORD_MQTT)) {
      Serial.println("Conectado ao broker MQTT!");
      MQTT.subscribe(TOPIC_SUBSCRIBE_LED);
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

void reconnectWiFi(void)
{
  if (WiFi.status() == WL_CONNECTED)
    return;

  WiFi.begin(SSID, PASSWORD); 

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
  oled.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS); 

  pinMode(pinTrig, OUTPUT); 
  pinMode(pinEcho, INPUT); 
  initWiFi();
  initMQTT();
}

void loop() {

  vso = analogRead(pino);
  int mapeado_valor_ph = map(vso, 0, 4095, 0.0, 14.0);
  checkWiFIAndMQTT();
  MQTT.loop();

  umidade = dht.readHumidity(); 
  temperatura = dht.readTemperature();


 
  oled.clearDisplay();

 
  oled.setTextSize(1);
  oled.setTextColor(WHITE);
  
  // Quadrado 1: Estado da mare
  oled.fillRect(0, 0, SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2, BLACK);
  oled.setCursor(5, 5);
  oled.setTextColor(WHITE);
  oled.print("PH: ");
  oled.print(mapeado_valor_ph);
  oled.setTextColor(WHITE);
  oled.setCursor(5, 20);
  oled.print(mare);

  // Quadrado 2: Distância
  oled.fillRect(SCREEN_WIDTH / 2, 0, SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2, BLACK);
  oled.setCursor(SCREEN_WIDTH / 2 + 5, 5);
  oled.setTextColor(WHITE);
  oled.print("Distancia");
  oled.setTextColor(WHITE);
  oled.setCursor(SCREEN_WIDTH / 2 + 5, 20);
  oled.print(medeDistancia()/10);
  oled.println(" cm");

  // Quadrado 3: Temperatura
  oled.fillRect(0, SCREEN_HEIGHT / 2, SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2, BLACK);
  oled.setCursor(5, SCREEN_HEIGHT / 2 + 5);
  oled.setTextColor(WHITE);
  oled.print("Temp");
  oled.setTextColor(WHITE);
  oled.setCursor(5, SCREEN_HEIGHT / 2 + 20);
  oled.print(temperatura);
  oled.write(248);
  oled.println(" C");

  // Quadrado 4: Umidade
  oled.fillRect(SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2, SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2, BLACK);
  oled.setCursor(SCREEN_WIDTH / 2 + 5, SCREEN_HEIGHT / 2 + 5);
  oled.setTextColor(WHITE);
  oled.print("Umidade");
  oled.setTextColor(WHITE);
  oled.setCursor(SCREEN_WIDTH / 2 + 5, SCREEN_HEIGHT / 2 + 20);
  oled.print(umidade);
  oled.println(" %");

  oled.display();
  delay(4000); 

  if (medeDistancia()/10 <= 100){
    mare = "mare morta";
  }
  else if (medeDistancia()/10 <= 200){
    mare = "mare baixa";
  }
  else if (medeDistancia()/10 <= 300){
    mare = "mare alta";
  }
  else {
    mare = "mareGrande";
  }

    if ((millis() - publishUpdate) >= PUBLISH_DELAY) {
    publishUpdate = millis();
      
      StaticJsonDocument<TAMANHO> doc_umidade;
      doc_umidade["variable"] = "umidade";
      doc_umidade["unit"] = "percent";
      doc_umidade["value"] = umidade;

      char buffer_umidade[TAMANHO];
      serializeJson(doc_umidade, buffer_umidade);
      MQTT.publish(TOPIC_SEND_VARIABLE, buffer_umidade);
      Serial.println(buffer_umidade);

      StaticJsonDocument<TAMANHO> doc_temperatura;
      doc_temperatura["variable"] = "temperatura";
      doc_temperatura["unit"] = "graus Celsius";
      doc_temperatura["value"] = temperatura;

      char buffer_temperatura[TAMANHO];
      serializeJson(doc_temperatura, buffer_temperatura);
      MQTT.publish(TOPIC_SEND_VARIABLE, buffer_temperatura);
      Serial.println(buffer_temperatura);

      StaticJsonDocument<TAMANHO> doc_distancia;
      doc_distancia["variable"] = "Distancia";
      doc_distancia["unit"] = "cm";
      doc_distancia["value"] = medeDistancia()/10;

      char buffer_distancia[TAMANHO];
      serializeJson(doc_distancia, buffer_distancia);
      MQTT.publish(TOPIC_SEND_VARIABLE, buffer_distancia);
      Serial.println(buffer_distancia);

      StaticJsonDocument<TAMANHO> doc_ph;
      doc_ph["variable"] = "ph";
      doc_ph["value"] = mapeado_valor_ph;

      char buffer_ph[TAMANHO];
      serializeJson(doc_ph, buffer_ph);
      MQTT.publish(TOPIC_SEND_VARIABLE, buffer_ph);
      Serial.println(buffer_ph);
    }



  }

float medeDistancia() 

{
  digitalWrite(pinTrig, HIGH);
  delayMicroseconds(10);
  digitalWrite(pinTrig, LOW);

  tempoEcho = pulseIn(pinEcho, HIGH); 

  
  float distancia_mm = tempoEcho * velocidadeSom / 1000 / 2;
  return distancia_mm;
}
