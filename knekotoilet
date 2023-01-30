/*
  smart cat toilet for M5Stick C Plus
  item
  M5Atom Lite

  WEIGHT UNIT
  Load Cell 32kg https://ja.aliexpress.com/item/32860005302.html
  or  
  M5 SCALES KIT https://shop.m5stack.com/products/scale-kit-with-weight-unit

  library
  HX711 library https://github.com/RobTillaart/HX711
  ArduinoJson
  Ambient
*/

#include "HX711.h"
#include <M5Atom.h>

#include <Ambient.h>

#include <WiFi.h>
#include <stdlib.h>
#include "time.h"
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <WiFiClientSecure.h>

const char* server = "maker.ifttt.com";  // IFTTT Server URL

//=====ここから個別に設定が必要な項目=====

const char* ssid = "BCW710J-68BAA-G"; // Wi-FiのSSID
const char* password = "854778a8a7a58"; // Wi-Fiのパスワード

String makerEvent = "Smart_Cat_Toilet"; // Maker Webhooks
String makerKey = "bNVZKxt7T4HQG10ZY7Yc4t";  // Maker Webhooks

Ambient ambient;

unsigned int channelID = 57758;
const char* writeKey = "2c694e036102a28e";

const int place = 1;  //トイレのユニークナンバー

//ロードセルの係数設定 SCALES KITは200kg
// 200kg 30.000 | 32kg 100.8 | 20kg 127.15 | 5kg 420.0983
#define cal 30.000

//=====ここまで個別に設定が必要な項目=====

WiFiClient client;

const char* ntpServer = "ntp.jst.mfeed.ad.jp";
const long gmtOffset_sec = 9 * 3600;
const int daylightOffset_sec = 0;

#define FRONT 1

#define X_LOCAL 40
#define Y_LOCAL 40
#define X_F 30
#define Y_F 30

const int capacity = JSON_OBJECT_SIZE(2);
StaticJsonDocument<capacity> json_request;
char buffer[255];

HX711 scale;

uint8_t dataPin = 32;
uint8_t clockPin = 26;

uint32_t start, stop;
volatile float f;

int notZero = 0;
float oldWeight = 0.00;
boolean flg = false;

void wifiConnect() {
  WiFi.begin(ssid, password);              //  Wi-Fi APに接続
  while (WiFi.status() != WL_CONNECTED) {  //  Wi-Fi AP接続待ち
    M5.dis.drawpix(0, 0x000055);
    delay(500);
    M5.dis.drawpix(0, 0x000000);
    Serial.print(".");
  }
  Serial.print("WiFi connected\r\nIP address: ");
  Serial.println(WiFi.localIP());
  M5.dis.drawpix(0, 0x000000);  
}

void setup() {
  M5.begin(true,false,true);
  delay(50);
  M5.dis.drawpix(0, 0x000000);
  Serial.println("black");
  scale.begin(dataPin, clockPin);
  wifiConnect();
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  struct tm timeinfo;

  WiFi.disconnect(true);

  // TODO find a nice solution for this calibration..
  // load cell factor 32 KG
  scale.set_scale(cal);

  Serial.begin(115200);
  delay(1000);
  // reset the scale to zero = 0
  scale.tare();

  oldWeight = 1;
  M5.dis.drawpix(0, 0x000000);

}

//IFTTTにパラメータを送信
void sendToIFTTT(float value1) {

  int value2 = place;

  String url = "/trigger/" + makerEvent + "/with/key/" + makerKey;
  url += "?value1=";
  url += value1;
  url += "&value2=";
  url += value2;

  Serial.println("\nStarting connection to server...");
  if (!client.connect(server, 80)) {
    Serial.println("Connection failed!");
  } else {
    Serial.println("Connected to server!");
    // Make a HTTP request:
    client.println("GET " + url + " HTTP/1.1");
    client.print("Host: ");
    client.println(server);
    client.println("Connection: close");
    client.println();
    Serial.print("Waiting for response ");  //WiFiClientSecure uses a non blocking implementation

    while (!client.available()) {
      delay(50);  //
      Serial.print(".");
    }
    // if there are incoming bytes available
    // from the server, read them and print them:
    while (client.available()) {
      char c = client.read();
      Serial.write(c);
    }

    // if the server's disconnected, stop the client:
    if (!client.connected()) {
      Serial.println();
      Serial.println("disconnecting from server.");
      client.stop();
    }
  }
}

void loop() {
  M5.update();
  if (M5.Btn.wasReleased()) {
    scale.tare();
  }
  M5.dis.drawpix(0, 0x333300);
  // continuous scale 4x per second
  f = scale.get_units(5);
  float weight = f / 1000;
  Serial.print(f);
  Serial.print(" ");
  Serial.print(weight);
  Serial.print("-");
  Serial.print(oldWeight);
  Serial.print("=");
  Serial.println(weight - oldWeight);

  // 10g以上の変動が21回連続したら送信判定してゼロ点を変更する
  if (weight >= oldWeight + 0.01 || weight <= oldWeight - 0.01) {
    notZero++;
    Serial.println(notZero);
    if (notZero > 20) {
      float wDiff;
      wDiff = weight - oldWeight;
      Serial.print("Send Value = ");
      Serial.println(wDiff);

      //200g以上プラス変異があったらクラウド側に送信する
      if (wDiff >= 0.2) {
        M5.dis.drawpix(0, 0x005500);
        wifiConnect();
        M5.dis.drawpix(0, 0x005555);
        sendToIFTTT(wDiff);
        ambient.begin(channelID,writeKey, &client);
        ambient.set(1, wDiff);
        ambient.send();
        WiFi.disconnect(true);
        Serial.println("Wi-Fi disconnected.");
      }

      oldWeight = weight;  //ゼロ点をズラす
      notZero = 0;
    }
  } else notZero = 0;  
}
