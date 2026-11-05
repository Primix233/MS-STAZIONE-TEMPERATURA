# MS-STAZIONE-TEMPERATURA
Progetto TPSIT per una stazione di rivelazione della temperatura utilizzando l IESP32 come microcontrollore e un DS18B20 come sensore di temperatura
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

#include <OneWire.h>
#include <DallasTemperature.h>

// Pin GPIO dell'ESP32 collegato al pin dati (DQ) del DS18B20
const int SENSOR_PIN = 4;

// Configurazione bus OneWire e riferimento a DallasTemperature
OneWire oneWire(SENSOR_PIN);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(115200);
  sensors.begin();
  Serial.println("Inizializzazione sensore DS18B20...");
}

void loop() {
  // Invia il comando per misurare la temperatura
  sensors.requestTemperatures(); 

  // Legge la temperatura in gradi Celsius
  float tempC = sensors.getTempCByIndex(0);

  if (tempC != DEVICE_DISCONNECTED_C) {
    Serial.print("Temperatura: ");
    Serial.print(tempC);
    Serial.println(" °C");
  } else {
    Serial.println("Errore: Sensore non rilevato!");
  }

  delay(2000); // Aggiorna ogni 2 secondi
}


-------------------------------------------------------------------------------------------------------------------------------------------------------------------

//prendere il tempo NTP

#include <WiFi.h>
#include <time.h>

// Credenziali Wi-Fi
const char* ssid     = "NOME_DELLA_TUA_RETE";
const char* password = "PASSWORD_DELLA_TUA_RETE";

// Configurazione NTP
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 3600;       // Fuso orario Italia: UTC+1 = 3600 secondi
const int   daylightOffset_sec = 3600;  // Ora legale (estate): +1 ora = 3600 secondi (imposta a 0 in inverno)

// Funzione per stampare l'ora corrente
void printLocalTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    Serial.println("Errore: impossibile ottenere l'ora da NTP!");
    return;
  }
  
  // Stampa l'ora formattata (HH:MM:SS) e la data (DD/MM/YYYY)
  Serial.println(&timeinfo, "%d/%m/%Y %H:%M:%S");
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  // Connessione al Wi-Fi
  Serial.print("Connessione a ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi Connesso!");

  // Configurazione e sincronizzazione orario NTP
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  Serial.println("Sincronizzazione orario con il server NTP...");
}

void loop() {
  printLocalTime();
  delay(2000); // Stampa l'ora ogni 2 secondi
}

-------------------------------------------------------------------------------------------------------------------------------------------------------------------
