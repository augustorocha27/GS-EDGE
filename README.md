# Controle de Ventilação Automatizada com Base em Temperatura

Este projeto utiliza o **ESP32** com um **sensor DHT22** para medir a temperatura e a umidade ambiente. O objetivo é automatizar o controle de ventilação de um ambiente ajustando a posição de uma **janela** ou **persiana** (simulada por um **servo motor**) com base na temperatura ambiente. O sistema também se comunica com o servidor MQTT para enviar dados e receber comandos de controle remoto.

## Componentes Utilizados

- **ESP32**: Microcontrolador com conectividade WiFi.
- **Sensor DHT22**: Sensor de temperatura e umidade.
- **Servo Motor**: Para controlar fisicamente a abertura e o fechamento de uma janela ou persiana.
- **Broker MQTT**: Para comunicação remota e controle do sistema via internet.

## Funcionalidades

- **Leitura de Temperatura e Umidade**: O sensor DHT22 coleta as medições de temperatura e umidade do ambiente.
- **Controle de Ventilação**: Com base na temperatura medida, o servo motor ajusta a posição de uma janela ou persiana. Quanto mais alta a temperatura, maior será a abertura.
- **MQTT**: Comunicação via protocolo MQTT para monitoramento remoto e controle manual do servo motor. 
- **Controle Automático e Manual**: O sistema ajusta automaticamente a posição do servo com base na temperatura, mas também permite controle manual via mensagens MQTT.

### Exemplo de Controle Automático:

- Se a temperatura for **menor que 25°C**, o servo fica fechado (janela fechada).
- Se a temperatura estiver **entre 25°C e 30°C**, o servo ficará em **90 graus** (janela meio aberta).
- Se a temperatura for **acima de 30°C**, o servo estará em **180 graus** (janela completamente aberta).

## Bibliotecas Utilizadas

- **WiFi.h**: Biblioteca para conectar o ESP32 à rede WiFi.
- **PubSubClient.h**: Biblioteca MQTT para comunicação com o broker MQTT.
- **DHTesp.h**: Biblioteca para ler dados do sensor DHT22.
- **ESP32Servo.h**: Biblioteca para controlar o servo motor no ESP32.

## Circuito

- **Sensor DHT22**: Conecte o pino **VCC** ao pino **3.3V**, o pino **GND** ao **GND** e o pino **DATA** ao **pino 25** do ESP32.
- **Servo Motor**: Conecte o fio **VCC** ao **5V**, o fio **GND** ao **GND** e o fio de controle ao **pino 18** do ESP32.

## Código

Aqui está o código completo para o projeto:

```cpp
#include <WiFi.h>
#include <PubSubClient.h>
#include <DHTesp.h>
#include <ESP32Servo.h>  // Usando a biblioteca compatível com o ESP32

const int DHT_PIN = 25; 
DHTesp dht;

const int SERVO_PIN = 18;  
Servo meuServo;          

const char* ssid = "Wokwi-GUEST";
const char* password = "";
const char* mqtt_server = "test.mosquitto.org";

WiFiClient espClient;
PubSubClient client(espClient);

// Função para conectar ao WiFi
void setup_wifi() {
  Serial.print("Conectando ao WiFi...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi conectado. IP: ");
  Serial.println(WiFi.localIP());
}

// Função de callback para mensagens MQTT
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Mensagem recebida em [");
  Serial.print(topic);
  Serial.print("]: ");
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();

  // Se a mensagem recebida for para controlar o servo
  if (String(topic) == "ThinkIOT/comandos") {
    int angulo = atoi((char*)payload);  
    if (angulo >= 0 && angulo <= 180) {
      meuServo.write(angulo);  
      Serial.print("Movendo o servo para: ");
      Serial.println(angulo);
    }
  }
}

// Função para reconectar ao broker MQTT
void reconnect() {
  while (!client.connected()) {
    String clientId = "ESP32Client-";
    clientId += String(random(0xffff), HEX);
    if (client.connect(clientId.c_str())) {
      Serial.println("Conectado!");
      client.subscribe("ThinkIOT/comandos");
    } else {
      Serial.print("Falha na conexão MQTT, rc=");
      Serial.println(client.state());
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  setup_wifi();
  client.setServer(mqtt_server, 1883);
  client.setCallback(callback);
  dht.setup(DHT_PIN, DHTesp::DHT22);
  meuServo.attach(SERVO_PIN);  
  meuServo.write(90);  // Servo no centro
}

void loop() {
  if (!client.connected()) reconnect();
  client.loop();

  TempAndHumidity data = dht.getTempAndHumidity();
  if (isnan(data.temperature) || isnan(data.humidity)) {
    Serial.println("Erro ao ler o sensor DHT!");
    return;
  }

  String temperatura = String(data.temperature, 2);
  String umidade = String(data.humidity, 1);

  client.publish("/ThinkIOT/temp", temperatura.c_str());
  client.publish("/ThinkIOT/hum", umidade.c_str());

  Serial.println("Dados enviados:");
  Serial.print("Temperatura: "); Serial.println(temperatura);
  Serial.print("Umidade: "); Serial.println(umidade);

  // Exemplo de controle automático do servo com base na temperatura
  int angulo = map(data.temperature, 0, 40, 0, 180);  
  meuServo.write(angulo);  

  Serial.print("Temperatura: "); Serial.print(data.temperature); Serial.print(" -> Movendo servo para: ");
  Serial.println(angulo);

  delay(5000);
}
