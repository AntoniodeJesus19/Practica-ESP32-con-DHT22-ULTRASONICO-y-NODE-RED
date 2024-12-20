# Practica-ESP32-con-DHT22-ULTRASONICO-y-NODE-RED

Utilizaremos la ESP32 en un entorno de adquisición de datos, le conectaremos un sensor DHT22 y un Ultrasónico; todo esto lo mandaremos a NODE-RED.

## Intrucciones

1- Abrimos Node-red en el CMD como administrador.

2- Entramos a la direccion [localhost:1880](https:localhost:1880).

3- Colocamos un bloque **mqqtt in**.

![]()

4- Configurar el bloque con el puerto mqtt con el ip 35.172.255.228 el cual se obtuvo del [broker](https://www.emqx.com/en/mqtt/public-mqtt5-broker).publico con el comando nslookup broker.emqx.in en el CMD.

![]()

5- Colocar el bloque json y configurarlo asi.

![]()

6- Colocamos tres bloques function y lo configuramos con el siguente codigo.

```
msg.payload = msg.payload.TEMPERATURA;
msg.topic = "TEMPERATURA";
return msg;
```

```
msg.payload = msg.payload.HUMEDAD;
msg.topic = "HUMEDAD";
return msg;
```

```
msg.payload = msg.payload.DISTANCIA;
msg.topic = "DISTANCIA";
return msg;
```

7- Colocamos los bloques de chart y gauge, los configuramos en el dashboard haciendo grupos, luego haciendo click en los chart y los gauge los colocamos en el grupo de nuestra elección.
![]()
![]()
![]()

8- Programación en ESP32
```

#include <ArduinoJson.h>
#include <WiFi.h>
#include <PubSubClient.h>
#define BUILTIN_LED 2
#include "DHTesp.h"
const int DHT_PIN = 15;
DHTesp dhtSensor;
// Update these with values suitable for your network.
const int Trigger = 4;   //Pin digital 2 para el Trigger del sensor
const int Echo = 16;   //Pin digital 3 para el Echo del sensor

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "35.172.255.228";
String username_mqtt="AntonioM";
String password_mqtt="12345678";

WiFiClient espClient;
PubSubClient client(espClient);
unsigned long lastMsg = 0;
#define MSG_BUFFER_SIZE  (50)
char msg[MSG_BUFFER_SIZE];
int value = 0;

void setup_wifi() {

  delay(10);
  // We start by connecting to a WiFi network
  Serial.println();
  Serial.print("Connecting to ");
  Serial.println(ssid);

  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  randomSeed(micros());

  Serial.println("");
  Serial.println("WiFi connected");
  Serial.println("IP address: ");
  Serial.println(WiFi.localIP());
}

void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Message arrived [");
  Serial.print(topic);
  Serial.print("] ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Switch on the LED if an 1 was received as first character
  if ((char)payload[0] == '1') {
    digitalWrite(BUILTIN_LED, LOW);   
    // Turn the LED on (Note that LOW is the voltage level
    // but actually the LED is on; this is because
    // it is active low on the ESP-01)
  } else {
    digitalWrite(BUILTIN_LED, HIGH);  
    // Turn the LED off by making the voltage HIGH
  }

}

void reconnect() {
  // Loop until we're reconnected
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    // Create a random client ID
    String clientId = "ESP8266Client-";
    clientId += String(random(0xffff), HEX);
    // Attempt to connect
    if (client.connect(clientId.c_str(), username_mqtt.c_str() , password_mqtt.c_str())) {
      Serial.println("connected");
      // Once connected, publish an announcement...
      client.publish("outTopic", "hello world");
      // ... and resubscribe
      client.subscribe("inTopic");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" try again in 5 seconds");
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  pinMode(BUILTIN_LED, OUTPUT);     // Initialize the BUILTIN_LED pin as an output
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dhtSensor.setup(DHT_PIN, DHTesp::DHT22);
  
  pinMode(Trigger, OUTPUT); //pin como salida
  pinMode(Echo, INPUT);  //pin como entrada
  digitalWrite(Trigger, LOW);//Inicializamos el pin con 0

}

void loop() {


  long t; //timepo que demora en llegar el eco
  long d; //distancia en centimetros

  digitalWrite(Trigger, HIGH);
  delayMicroseconds(10);          //Enviamos un pulso de 10us
  digitalWrite(Trigger, LOW);
  
  t = pulseIn(Echo, HIGH); //obtenemos el ancho del pulso
  d = t/59;             //escalamos el tiempo a una distancia en cm
  
  Serial.print("Distancia: ");
  Serial.print(d);      //Enviamos serialmente el valor de la distancia
  Serial.print("cm");
  Serial.println();
  delay(1000);          //Hacemos una pausa de 100ms


delay(1000);
TempAndHumidity  data = dhtSensor.getTempAndHumidity();
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  unsigned long now = millis();
  if (now - lastMsg > 2000) {
    lastMsg = now;
    //++value;
    //snprintf (msg, MSG_BUFFER_SIZE, "hello world #%ld", value);

    StaticJsonDocument<128> doc;

    doc["DEVICE"] = "ESP32";
    //doc["Anho"] = 2022;
    //doc["Empresa"] = "Educatronicos";
    doc["TEMPERATURA"] = String(data.temperature, 1);
    doc["HUMEDAD"] = String(data.humidity, 1);
   doc["NOMBRE"]="ANTONIO DE JESUS MH";
   doc["DISTANCIA"]= String(d) + "cm";

    String output;
    
    serializeJson(doc, output);

    Serial.print("Publish message: ");
    Serial.println(output);
    Serial.println(output.c_str());
    client.publish("PRACTICA", output.c_str());
  }
}

```
-En estas lineas se modifica el broker que tengamos al momento, el username y la contraseña.

![]()

-En estas lineas podemos modificar la información que mandamos al node-red, IMPORTANTE en client.publish ponemos el nombre del TOPICO que nos piden en los demas bloques chart y gauge.

![]()

9- Conexión de WOKWI

![](https://github.com/AntoniodeJesus19/Practica-ESP32-con-DHT22-ULTRASONICO-y-NODE-RED/blob/main/Captura%20de%20pantalla%202024-12-14%20122830.png?raw=true)

## Resultados

![]()

![]()

![]()

![]()

# Créditos

Desarrollado por Antonio de Jesús Mentado Huerta

- [GitHub](https://github.com/AntoniodeJesus19)
