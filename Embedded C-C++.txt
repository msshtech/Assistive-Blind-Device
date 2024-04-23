#include <NewPing.h>
#include <millisDelay.h>
#include <Arduino.h>
#include <WiFi.h>
#include <ESP_Mail_Client.h>
#include <TinyGPS++.h>
#include <SoftwareSerial.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <Wire.h>


#define WIFI_SSID "VisionOfDSNT"
#define WIFI_PASSWORD "@d12345ue"

#define SMTP_HOST "smtp.gmail.com"
#define SMTP_PORT 465

/* The sign in credentials */
#define AUTHOR_EMAIL "visiomwand01@gmail.com"
#define AUTHOR_PASSWORD "pcltiuwjrtuyhgdw"

/* Recipient's email*/
// #define RECIPIENT_EMAIL "allahditta@ue.edu.pk"
#define RECIPIENT_EMAIL "visiomwand01@gmail.com"


/* The SMTP Session object used for Email sending */
SMTPSession smtp;

/* Callback function to get the Email sending status */
void smtpCallback(SMTP_Status status);

/* Declare the message class */
SMTP_Message message;

/* Declare the session config data */
ESP_Mail_Session session;

static const int RXPin = 16, TXPin = 17;
static const uint32_t GPSBaud = 9600;

Adafruit_MPU6050 mpu;


#define triggerPin 18
#define echoPin 19
#define MAX_DISTANCE 100
#define volume 35
#define buzzer 27
#define rainSensor 34
#define emergencyButton 12
#define safeBtn 14

int distance = 0;
int slider = 0;
int rangeSet = 0;
int rainRead = 0;
float xAxis = 0;
float yAxis = 0;
float temperature = 0;


double latitude = 31.4960158;
double longitude = 74.2531461;

bool shbeepCheck = true;
bool lnbeepCheck = true;
bool emBtn = false;
bool sfBtn = false;
bool fallenCheck = false;
bool fallenDetect = false;


NewPing sonar(triggerPin, echoPin , MAX_DISTANCE);

// The TinyGPS++ object
TinyGPSPlus gps;

// The serial connection to the GPS device
SoftwareSerial ss(RXPin, TXPin);

millisDelay shortBeep;
millisDelay longBeep;
millisDelay fallenTimer;

void setup() {
  
  Serial.begin(115200);
  ss.begin(GPSBaud);
  
  Serial.println("Blind Stick");

  pinMode(volume, INPUT);
  pinMode(rainSensor, INPUT);
  
  pinMode(emergencyButton, INPUT_PULLUP);
  pinMode(safeBtn, INPUT_PULLUP);
  
  pinMode(buzzer, OUTPUT);

  digitalWrite(buzzer, LOW);

  Serial.print("Connecting to AP");
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  while (WiFi.status() != WL_CONNECTED){
    Serial.print(".");
    delay(200);
  }
  Serial.println("");
  Serial.println("WiFi connected.");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
  Serial.println();

  smtp.debug(1);

  /* Set the callback function to get the sending results */
  smtp.callback(smtpCallback);

  /* Set the session config */
  session.server.host_name = SMTP_HOST;
  session.server.port = SMTP_PORT;
  session.login.email = AUTHOR_EMAIL;
  session.login.password = AUTHOR_PASSWORD;
  session.login.user_domain = "";

  /* Set the message headers */
  message.sender.name = "Blind Person";
  message.sender.email = AUTHOR_EMAIL;
  message.subject = "Alert From Blind Person";
  message.addRecipient("Alert!", RECIPIENT_EMAIL);

  /*
  //Send raw text message
  String textMsg = "Hello World! - Sent from ESP board";
  message.text.content = textMsg.c_str();
  message.text.charSet = "us-ascii";
  message.text.transfer_encoding = Content_Transfer_Encoding::enc_7bit;
  
  message.priority = esp_mail_smtp_priority::esp_mail_smtp_priority_low;
  message.response.notify = esp_mail_smtp_notify_success | esp_mail_smtp_notify_failure | esp_mail_smtp_notify_delay;*/

  /* Set the custom message header */
  //message.addHeader("Message-ID: <abcde.fghij@gmail.com>");

  if (!mpu.begin()){
  }

  mpu.setAccelerometerRange(MPU6050_RANGE_8_G);
  Serial.print("Accelerometer range set to: ");
  switch (mpu.getAccelerometerRange()) {
  case MPU6050_RANGE_2_G:
    Serial.println("+-2G");
    break;
  case MPU6050_RANGE_4_G:
    Serial.println("+-4G");
    break;
  case MPU6050_RANGE_8_G:
    Serial.println("+-8G");
    break;
  case MPU6050_RANGE_16_G:
    Serial.println("+-16G");
    break;
  }

  mpu.setFilterBandwidth(MPU6050_BAND_21_HZ);
  Serial.print("Filter bandwidth set to: ");
  switch (mpu.getFilterBandwidth()) {
  case MPU6050_BAND_260_HZ:
    Serial.println("260 Hz");
    break;
  case MPU6050_BAND_184_HZ:
    Serial.println("184 Hz");
    break;
  case MPU6050_BAND_94_HZ:
    Serial.println("94 Hz");
    break;
  case MPU6050_BAND_44_HZ:
    Serial.println("44 Hz");
    break;
  case MPU6050_BAND_21_HZ:
    Serial.println("21 Hz");
    break;
  case MPU6050_BAND_10_HZ:
    Serial.println("10 Hz");
    break;
  case MPU6050_BAND_5_HZ:
    Serial.println("5 Hz");
    break;
  }

  Serial.println("");
  delay(100);


  shortBeep.start(400);
  longBeep.start(800);
  fallenTimer.start(20000);

  fallenCheck = true;
  
}

void loop() {
  
  distanceMeasure();
  rainsensorRead();
  buzzerAlert();
  gpsLocation();
  emergencybtnCheck();
  accelerometer();
  stickFallen();

  delay(500);
 
}


void distanceMeasure(){
  
  distance = sonar.ping_cm();
  slider = analogRead(volume);

  rangeSet = map(slider, 0, 4095, 0, 100);
  
  Serial.print("Ping: ");
  Serial.print(distance);
  Serial.println("cm");
  
  Serial.print("Volume: ");
  Serial.println(slider);
  
  Serial.print("Range Set: ");
  Serial.println(rangeSet);
  
}


void rainsensorRead(){

  rainRead = digitalRead(rainSensor);
  Serial.print("Rain: ");
  Serial.println(rainRead);
  
}


void buzzerAlert(){

  if( (distance > rangeSet) && (rainRead == 1) ){
    if(shortBeep.remaining() == 0){
      if(shbeepCheck == true){
        digitalWrite(buzzer, HIGH);
        shortBeep.restart();
        shbeepCheck = false;
      }else{
        digitalWrite(buzzer, LOW);
        shortBeep.restart();
        shbeepCheck = true;
      }
    }
  }

  else if( (distance < rangeSet) && (rainRead == 0) ){
    if(longBeep.remaining() == 0){
      if(lnbeepCheck == true){
        digitalWrite(buzzer, HIGH);
        longBeep.restart();
        lnbeepCheck = false;
      }else{
        digitalWrite(buzzer, LOW);
        longBeep.restart();
        lnbeepCheck = true;
      }
    }
  }

  else if( (distance > rangeSet) && (rainRead == 0) ){
    digitalWrite(buzzer, HIGH);
  }
  
  else{
    digitalWrite(buzzer, LOW);
  }
  
}


void gpsLocation(){
  // This sketch displays information every time a new sentence is correctly encoded.
  while (ss.available() > 0){
    gps.encode(ss.read());
    if (gps.location.isUpdated()){
      latitude = gps.location.lat();
      longitude = gps.location.lng();
      
      Serial.print("Latitude= ");
      Serial.print(latitude, 7);
      Serial.print(" Longitude= "); 
      Serial.println(longitude, 7);
    }
  }
}


void accelerometer(){

  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  xAxis = a.acceleration.x;
  yAxis = a.acceleration.y;
  temperature = temp.temperature;

  Serial.print("xAxis: ");
  Serial.print(xAxis);
  Serial.print(" yAxis: ");
  Serial.print(yAxis);
  Serial.print(" Temperature: ");
  Serial.println(temperature);

}


void stickFallen(){

  sfBtn = digitalRead(safeBtn);

  if( ( (xAxis < -8) || (xAxis > 8) || (yAxis > -2) ) && (fallenCheck == true) ){
    Serial.println("Person Fallen Timer Start");
    fallenCheck = false;
    fallenDetect = true;
    digitalWrite(buzzer, HIGH);
    delay(100);
    digitalWrite(buzzer, LOW);
    fallenTimer.restart();
  }

  if( (sfBtn == 0) && (fallenDetect == true) ){
    Serial.println("No Issue");
    digitalWrite(buzzer, HIGH);
    delay(1000);
    digitalWrite(buzzer, LOW);
    fallenCheck = true;
    fallenDetect = false;
  }

  else if( (fallenDetect == true) && (fallenTimer.remaining() == 0) ){

    Serial.println("Person Fallen Alert!");
    digitalWrite(buzzer, HIGH);
    delay(100);
    digitalWrite(buzzer, LOW);
    delay(100);
    digitalWrite(buzzer, HIGH);
    delay(100);
    digitalWrite(buzzer, LOW);
    delay(100);
    digitalWrite(buzzer, HIGH);
    delay(100);
    digitalWrite(buzzer, LOW);
    /*Send HTML message*/
    /*Send HTML message*/
    String htmlMsg = "https://www.google.com/maps?z=15&q=" + String(latitude,7) + "," + String(longitude,7) + " "
    + "| " + "Environment Temperature: " + String(temperature) + " C | Person Fallen";
    message.html.content = htmlMsg.c_str();
    message.html.content = htmlMsg.c_str();
    message.text.charSet = "us-ascii";
    message.html.transfer_encoding = Content_Transfer_Encoding::enc_7bit;
    
    /* Connect to server with the session config */
    if (!smtp.connect(&session))
      return;
      
    if (!MailClient.sendMail(&smtp, &message))
      Serial.println("Error sending Email, " + smtp.errorReason());

      fallenCheck = true;
      fallenDetect = false;
      
  }
  
}


void emergencybtnCheck(){
  
  emBtn = digitalRead(emergencyButton);
  if(emBtn == 0){
    Serial.println("Emergency Button Pressed");

    digitalWrite(buzzer, HIGH);
    delay(1000);
    digitalWrite(buzzer, LOW);
    
    /*Send HTML message*/
    String htmlMsg = "https://www.google.com/maps?z=15&q=" + String(latitude,7) + "," + String(longitude,7) + " "
    + "| " + "Environment Temperature: " + String(temperature) + " C | Emergency";
    message.html.content = htmlMsg.c_str();
    message.html.content = htmlMsg.c_str();
    message.text.charSet = "us-ascii";
    message.html.transfer_encoding = Content_Transfer_Encoding::enc_7bit;
  
    /* Connect to server with the session config */
    if (!smtp.connect(&session))
      return;
      
    if (!MailClient.sendMail(&smtp, &message))
      Serial.println("Error sending Email, " + smtp.errorReason());
      
  }
  
}


void smtpCallback(SMTP_Status status){
  /* Print the current status */
  Serial.println(status.info());

  /* Print the sending result */
  if (status.success()){
    Serial.println("----------------");
    ESP_MAIL_PRINTF("Message sent success: %d\n", status.completedCount());
    ESP_MAIL_PRINTF("Message sent failled: %d\n", status.failedCount());
    Serial.println("----------------\n");
    struct tm dt;

    for (size_t i = 0; i < smtp.sendingResult.size(); i++){
      /* Get the result item */
      SMTP_Result result = smtp.sendingResult.getItem(i);
      time_t ts = (time_t)result.timestamp;
      localtime_r(&ts, &dt);

      ESP_MAIL_PRINTF("Message No: %d\n", i + 1);
      ESP_MAIL_PRINTF("Status: %s\n", result.completed ? "success" : "failed");
      ESP_MAIL_PRINTF("Date/Time: %d/%d/%d %d:%d:%d\n", dt.tm_year + 1900, dt.tm_mon + 1, dt.tm_mday, dt.tm_hour, dt.tm_min, dt.tm_sec);
      ESP_MAIL_PRINTF("Recipient: %s\n", result.recipients.c_str());
      ESP_MAIL_PRINTF("Subject: %s\n", result.subject.c_str());
    }
    Serial.println("----------------\n");
  }
}
