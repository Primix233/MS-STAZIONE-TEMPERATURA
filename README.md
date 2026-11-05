# MS-STAZIONE-TEMPERATURA
Progetto TPSIT per una stazione di rivelazione della temperatura utilizzando l IESP32 come microcontrollore e un DS18B20 come sensore di temperatura
-----------------------------------------------------------------------------------------------------------------------------------------------------------------

//funziona

#include <WiFi.h>

//collegamento ESP32 come access point

// Configura i dettagli della rete generata dall'ESP32
const char* ssidAP     = "ESP32_Temp_Hotspot";
const char* passwordAP = "123456789"; // Minimo 8 caratteri (o NULL per rete aperta)

void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println();
  Serial.println("Avvio della rete Access Point...");

  // Imposta l'ESP32 in modalità Access Point
  WiFi.mode(WIFI_AP);
  
  // Crea la rete Wi-Fi
  bool result = WiFi.softAP(ssidAP, passwordAP);

  if (result) {
    Serial.println("Hotspot creato con successo!");
    Serial.print("Nome rete (SSID): ");
    Serial.println(ssidAP);
    Serial.print("Indirizzo IP dell'ESP32: ");
    Serial.println(WiFi.softAPIP()); // Di default è solitamente 192.168.4.1
  } else {
    Serial.println("Errore durante la creazione dell'Hotspot.");
  }
}

void loop() {
  // Stampa periodicamente il numero di dispositivi connessi all'ESP32
  int connectedDevices = WiFi.softAPgetStationNum();
  Serial.print("Dispositivi connessi all'hotspot: ");
  Serial.println(connectedDevices);

  delay(5000);
}

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

//collegamento ESP32 al wifi
//funziona

#include <WiFi.h>

// Sostituisci con le credenziali della tua rete Wi-Fi
const char* ssid     = "NOME_DELLA_TUA_RETE";
const char* password = "PASSWORD_DELLA_TUA_RETE";

void setup() {
  Serial.begin(115200);
  delay(1000);

  Serial.println();
  Serial.print("Connessione a: ");
  Serial.println(ssid);

  // Imposta l'ESP32 in modalità Wi-Fi Station
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);

  // Attende la connessione
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("");
  Serial.println("Wi-Fi connesso!");
  Serial.print("Indirizzo IP assegnato: ");
  Serial.println(WiFi.localIP());
}

void loop() {
  // Il loop rimane vuoto se la connessione è gestita in background.
  // Puoi verificare se la connessione rimane attiva:
  if (WiFi.status() != WL_CONNECTED) {
    Serial.println("Connessione Wi-Fi persa!");
  }
  delay(5000);
}

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

// collegamento ESP32 con DS18B20
//non funzia

#include <OneWire.h>
#include <DallasTemperature.h>

constexpr uint8_t SENSOR_PIN = 4;

OneWire oneWire(SENSOR_PIN);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(115200);
  sensors.begin();

  Serial.print("Sensori DS18B20 trovati: ");
  Serial.println(sensors.getDeviceCount());
}

void loop() {
  sensors.requestTemperatures();

  float tempC = sensors.getTempCByIndex(0);

  if (tempC == DEVICE_DISCONNECTED_C) {
    Serial.println("Errore: sensore non rilevato. Controlla collegamenti e pin.");
  } else {
    Serial.print("Temperatura: ");
    Serial.print(tempC, 2);
    Serial.println(" °C");
  }

  delay(2000);
}

-----------------------------------------------------------------------------------------------



// Lettura DS18B20 su ESP32 senza librerie OneWire/DallasTemperature.
// Collega DATA del sensore al GPIO 4.

constexpr uint8_t SENSOR_PIN = 4;

// Porta il pin a LOW oppure lo rilascia.
// Il pin viene rilasciato mettendolo in INPUT: serve una resistenza
// esterna da 4,7 kΩ tra DATA e 3,3 V.
void setBusLow() {
  pinMode(SENSOR_PIN, OUTPUT);
  digitalWrite(SENSOR_PIN, LOW);
}

void releaseBus() {
  pinMode(SENSOR_PIN, INPUT);
}

bool oneWireReset() {
  releaseBus();
  delayMicroseconds(5);

  noInterrupts();
  setBusLow();
  delayMicroseconds(480);
  releaseBus();
  delayMicroseconds(70);
  bool sensorPresent = (digitalRead(SENSOR_PIN) == LOW);
  interrupts();

  delayMicroseconds(410);
  return sensorPresent;
}

void oneWireWriteBit(bool bitValue) {
  noInterrupts();
  setBusLow();

  if (bitValue) {
    delayMicroseconds(6);
    releaseBus();
    delayMicroseconds(64);
  } else {
    delayMicroseconds(60);
    releaseBus();
    delayMicroseconds(10);
  }

  interrupts();
}

bool oneWireReadBit() {
  noInterrupts();
  setBusLow();
  delayMicroseconds(3);
  releaseBus();
  delayMicroseconds(10);
  bool bitValue = (digitalRead(SENSOR_PIN) == HIGH);
  delayMicroseconds(53);
  interrupts();

  return bitValue;
}

void oneWireWriteByte(uint8_t value) {
  for (uint8_t i = 0; i < 8; i++) {
    oneWireWriteBit(value & 0x01);
    value >>= 1;
  }
}

uint8_t oneWireReadByte() {
  uint8_t value = 0;

  for (uint8_t i = 0; i < 8; i++) {
    if (oneWireReadBit()) {
      value |= (1 << i);
    }
  }

  return value;
}

uint8_t calculateCRC8(const uint8_t* data, uint8_t length) {
  uint8_t crc = 0;

  for (uint8_t i = 0; i < length; i++) {
    uint8_t currentByte = data[i];

    for (uint8_t bit = 0; bit < 8; bit++) {
      uint8_t mix = (crc ^ currentByte) & 0x01;
      crc >>= 1;

      if (mix) {
        crc ^= 0x8C;
      }

      currentByte >>= 1;
    }
  }

  return crc;
}

bool readTemperature(float& temperatureC) {
  // Avvia la conversione della temperatura.
  if (!oneWireReset()) {
    return false;
  }

  oneWireWriteByte(0xCC); // Skip ROM: usa il sensore presente sul bus
  oneWireWriteByte(0x44); // Convert T
  delay(750);             // Attende la conversione (fino a 12 bit)

  // Legge i 9 byte della memoria del sensore.
  if (!oneWireReset()) {
    return false;
  }

  oneWireWriteByte(0xCC); // Skip ROM
  oneWireWriteByte(0xBE); // Read Scratchpad

  uint8_t scratchpad[9];

  for (uint8_t i = 0; i < 9; i++) {
    scratchpad[i] = oneWireReadByte();
  }

  // Verifica che i dati ricevuti siano validi.
  if (calculateCRC8(scratchpad, 8) != scratchpad[8]) {
    return false;
  }

  int16_t rawTemperature =
      (static_cast<int16_t>(scratchpad[1]) << 8) | scratchpad[0];

  temperatureC = rawTemperature / 16.0;
  return true;
}

void setup() {
  Serial.begin(115200);
  releaseBus();
  Serial.println("Lettura DS18B20 avviata");
}

void loop() {
  float temperatureC;

  if (readTemperature(temperatureC)) {
    Serial.print("Temperatura: ");
    Serial.print(temperatureC, 2);
    Serial.println(" °C");
  } else {
    Serial.println("Sensore non rilevato o lettura non valida.");
  }

  delay(2000);
}
-------------------------------------------------------------------------------------------------------------------------------------------------------------------
