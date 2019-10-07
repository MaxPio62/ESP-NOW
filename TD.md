# ESP-NOW
Ce TD permet de faire communiquer 2 ESP entre eux (1 master et 1 slave). Le master va repérer le slave et lui transmettre des données. En allumant la LED de l'ESP master, celle du slave va également s'allumer.
# Matériel requis
2 ESP

2 LED

1 Breadboard
# Premier exercice
L'objectif de cet exercice est de faire clignoter la LED de l'ESP une fois par seconde.


```
int LED_BUILTIN = 22;

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
}

void loop() {
  digitalWrite(LED_BUILTIN, HIGH);   
  delay(1000);                       
  digitalWrite(LED_BUILTIN, LOW);    
  delay(1000);                       
}
```
# Deuxième exercice
Ici, on va utiliser deux programmes Arduino (1 slave et 1 master) pour jumeler les deux ESP et leur permettre de communiquer.
Concernant le programme du master :
Tout d'abord, inclure les bibliothèques :
```
#include <esp_now.h>
#include <WiFi.h> 
```
Ensuite, on initialise nos variables :
```
int LED_BUILTIN = 22;
unsigned long time1, time2;
bool stateLED;
```
Copie globale de l'esclave : 
```
esp_now_peer_info_t slave;
#define CHANNEL 3
#define PRINTSCANRESULTS 0
#define DELETEBEFOREPAIR 0
```
Initialiser l'ESP-NOW :
```
void InitESPNow() {
  WiFi.disconnect();
  if (esp_now_init() == ESP_OK) {
    Serial.println("ESPNow Init Success");
  }
  else {
    Serial.println("ESPNow Init Failed");
    ESP.restart();
  }
}
```
On va inclure la fonction pour scanner le slave en AP mode :
```
void ScanForSlave() {
  int8_t scanResults = WiFi.scanNetworks();
  // Réinitialisation pour chaque scan
  bool slaveFound = 0;
  memset(&slave, 0, sizeof(slave));

  Serial.println("");
  if (scanResults == 0) {
    Serial.println("No WiFi devices in AP Mode found");
  } else {
    Serial.print("Found "); Serial.print(scanResults); Serial.println(" devices ");
    for (int i = 0; i < scanResults; ++i) {
      // Affiche le SSID et le RSSI pour chaque appareil trouvé
      String SSID = WiFi.SSID(i);
      int32_t RSSI = WiFi.RSSI(i);
      String BSSIDstr = WiFi.BSSIDstr(i);

      if (PRINTSCANRESULTS) {
        Serial.print(i + 1);
        Serial.print(": ");
        Serial.print(SSID);
        Serial.print(" (");
        Serial.print(RSSI);
        Serial.print(")");
        Serial.println("");
      }
      delay(10);
      // Check si l'appareil actuel démarre en tant que "Slave"
      if (SSID.indexOf("Slave") == 0) {
        Serial.println("Found a Slave.");
        Serial.print(i + 1); Serial.print(": "); Serial.print(SSID); Serial.print(" ["); Serial.print(BSSIDstr); Serial.print("]"); Serial.print(" ("); Serial.print(RSSI); Serial.print(")"); Serial.println("");
        // Donne la MAC adresse du slave
        int mac[6];
        if ( 6 == sscanf(BSSIDstr.c_str(), "%x:%x:%x:%x:%x:%x%c",  &mac[0], &mac[1], &mac[2], &mac[3], &mac[4], &mac[5] ) ) {
          for (int ii = 0; ii < 6; ++ii ) {
            slave.peer_addr[ii] = (uint8_t) mac[ii];
          }
        }

        slave.channel = CHANNEL; 
        slave.encrypt = 0; 

        slaveFound = 1; // Dans notre exemple nous avaons un seul slave
        break;
      }
    }
  }

  if (slaveFound) {
    Serial.println("Slave Found, processing..");
  } else {
    Serial.println("Slave Not Found, trying again.");
  }

  WiFi.scanDelete();
}
```
Puis, on va regarder si le slave est jumelé avec un master :
```
bool manageSlave() {
  if (slave.channel == CHANNEL) {
    if (DELETEBEFOREPAIR) {
      deletePeer();
    }

    Serial.print("Slave Status: ");
    const esp_now_peer_info_t *peer = &slave;
    const uint8_t *peer_addr = slave.peer_addr;
    // check si la paire existe
    bool exists = esp_now_is_peer_exist(peer_addr);
    if ( exists) {
      Serial.println("Already Paired");
      return true;
    } else {
      esp_err_t addStatus = esp_now_add_peer(peer);
      if (addStatus == ESP_OK) {
        Serial.println("Pair success");
        return true;
      } else if (addStatus == ESP_ERR_ESPNOW_NOT_INIT) {        
        Serial.println("ESPNOW Not Init");
        return false;
      } else if (addStatus == ESP_ERR_ESPNOW_ARG) {
        Serial.println("Invalid Argument");
        return false;
      } else if (addStatus == ESP_ERR_ESPNOW_FULL) {
        Serial.println("Peer list full");
        return false;
      } else if (addStatus == ESP_ERR_ESPNOW_NO_MEM) {
        Serial.println("Out of memory");
        return false;
      } else if (addStatus == ESP_ERR_ESPNOW_EXIST) {
        Serial.println("Peer Exists");
        return true;
      } else {
        Serial.println("Not sure what happened");
        return false;
      }
    }
  } else {
    // Aucun slave trouvé dans le processus de scan
    Serial.println("No Slave found to process");
    return false;
  }
}

void deletePeer() {
  const esp_now_peer_info_t *peer = &slave;
  const uint8_t *peer_addr = slave.peer_addr;
  esp_err_t delStatus = esp_now_del_peer(peer_addr);
  Serial.print("Slave Delete Status: ");
  if (delStatus == ESP_OK) {
    Serial.println("Success");
  } else if (delStatus == ESP_ERR_ESPNOW_NOT_INIT) {
    Serial.println("ESPNOW Not Init");
  } else if (delStatus == ESP_ERR_ESPNOW_ARG) {
    Serial.println("Invalid Argument");
  } else if (delStatus == ESP_ERR_ESPNOW_NOT_FOUND) {
    Serial.println("Peer not found.");
  } else {
    Serial.println("Not sure what happened");
  }
}
```
On va demander au master d'envoyer une donnée au slave :
```
void sendData(uint8_t data) {
  const uint8_t *peer_addr = slave.peer_addr;
  Serial.print("Sending: "); Serial.println(data);
  esp_err_t result = esp_now_send(peer_addr, &data, sizeof(data));
  Serial.print("Send Status: ");
  if (result == ESP_OK) {
    Serial.println("Success");
  } else if (result == ESP_ERR_ESPNOW_NOT_INIT) {
    // How did we get so far!!
    Serial.println("ESPNOW not Init.");
  } else if (result == ESP_ERR_ESPNOW_ARG) {
    Serial.println("Invalid Argument");
  } else if (result == ESP_ERR_ESPNOW_INTERNAL) {
    Serial.println("Internal Error");
  } else if (result == ESP_ERR_ESPNOW_NO_MEM) {
    Serial.println("ESP_ERR_ESPNOW_NO_MEM");
  } else if (result == ESP_ERR_ESPNOW_NOT_FOUND) {
    Serial.println("Peer not found.");
  } else {
    Serial.println("Not sure what happened");
  }
} 
```
On va configurer le programme pour voir la réponse du slave vers le master :
```
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
  char macStr[18];
  snprintf(macStr, sizeof(macStr), "%02x:%02x:%02x:%02x:%02x:%02x",
           mac_addr[0], mac_addr[1], mac_addr[2], mac_addr[3], mac_addr[4], mac_addr[5]);
  Serial.print("Last Packet Sent to: "); Serial.println(macStr);
  Serial.print("Last Packet Send Status: "); Serial.println(status == ESP_NOW_SEND_SUCCESS ? "Delivery Success" : "Delivery Fail"); 
  Serial.println("");
}

void setup() {
{
  Serial.begin(115200);
  // Met l'appareil en mode STA
  WiFi.mode(WIFI_STA);
  Serial.println("ESPNow Example");
  Serial.print("STA MAC: "); Serial.println(WiFi.macAddress());
  InitESPNow();
  esp_now_register_send_cb(OnDataSent);
}
{
  ScanForSlave();
  pinMode(LED_BUILTIN, OUTPUT);
  digitalWrite(LED_BUILTIN, LOW);
  stateLED = false;

  time2 = millis();
}
}


void loop() {
  time1=millis();// prend le temps pour commencer le processus
  if((time1 - time2)>1000)  { //Wait for 1
    
    time2 = millis();// prend le temps pour commencer le processus
    if(stateLED){
      digitalWrite(LED_BUILTIN, LOW);    // met la LED en mode OFF (LOW est le niveau de tension)
      stateLED = false;
    }else {
      digitalWrite(LED_BUILTIN, HIGH);   // met la LED en mode ON (HIGH est le niveau de tension)      
      stateLED=true;
    }
    
    if (slave.channel == CHANNEL) { 
     
  
      bool isPaired = manageSlave();
      if (isPaired) {
        // si le slave est jumelé avec un master, le master enverra donc la donnée
        if(stateLED)
        {
          sendData(1);
          Serial.println("LED ON"); // envoie la donnée "1" lorsque la LED est en mode ON
        } else {
          sendData(0);
          Serial.println("LED OFF"); // envoie la donnée "0" lorsque la LED est en mode OF
        }
      
      } else {
      // si pas de slave jumelé avec un master
        ScanForSlave();
        Serial.println("Slave NOT paired");
      }
    }
  }  
}
```
Après avoir fait le programme Master, nous allons faire celui du slave :
```
#include <esp_now.h>
#include <WiFi.h>

#define CHANNEL 1
// Initialisation des variables
int LED_BUILTIN = 22;
bool stateLED;

// Initialisation de l'ESP
void InitESPNow() {
  WiFi.disconnect();
  if (esp_now_init() == ESP_OK) {
    Serial.println("ESPNow Init Success");
  }
  else {
    Serial.println("ESPNow Init Failed");
    ESP.restart();
  }
}

// Configuration de l'AP SSID
void configDeviceAP() {
  const char *SSID = "Slave_1";
  bool result = WiFi.softAP(SSID, "Slave_1_Password", CHANNEL, 0);
  if (!result) {
    Serial.println("AP Config failed.");
  } else {
    Serial.println("AP Config Success. Broadcasting with AP: " + String(SSID));
  }
}

void setup() {
  {
  Serial.begin(115200);
  Serial.println("ESPNow/Basic/Slave Example");
  WiFi.mode(WIFI_AP);
  // Configuration de l'appareil en AP mode
  configDeviceAP();
  // La MAC adresse du slave en AP mode
  Serial.print("AP MAC: "); Serial.println(WiFi.softAPmacAddress());
  InitESPNow();
  esp_now_register_recv_cb(OnDataRecv);
  }
  {
  pinMode(LED_BUILTIN, OUTPUT);
  stateLED =false;
  }
}
// Rappel quand la donnée est bien reçue par le slave provenant du master
void OnDataRecv(const uint8_t *mac_addr, const uint8_t *data, int data_len) {
  char macStr[18];
  snprintf(macStr, sizeof(macStr), "%02x:%02x:%02x:%02x:%02x:%02x",
           mac_addr[0], mac_addr[1], mac_addr[2], mac_addr[3], mac_addr[4], mac_addr[5]);
  Serial.print("Last Packet Recv from: "); Serial.println(macStr);
  Serial.print("Last Packet Recv Data: "); Serial.println(*data);
  if(*data == 0){
    digitalWrite(LED_BUILTIN, LOW);    // met la LED en mode OFF (LOW est le niveau de tension)
    stateLED = false;
  }else if(*data == 1){
    digitalWrite(LED_BUILTIN, HIGH);   // met la LED en mode ON (HIGH est le niveau de tension)      
     stateLED=true;
  }
  Serial.println("");
}

void loop() {
}
```
 


# Troisième exercice 
L'objectif de cet exercice est de faire en sorte que les ESP fonctionnement de la manière suivante:

j'allume 1 ESP, il fait clignoter sa LED une fois par seconde

j'allume un second ESP, il se connecte au réseau MESH et de façon synchro, les 2 ESP font clignoter leur LED deux fois par seconde
