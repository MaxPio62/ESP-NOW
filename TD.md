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
Dans cet exercice, on va coupler les 2 ESP entre eux en configurant l'un comme master et l'autre comme slave.

Pour commencer, inclure les bibliothèques : 
```
#include <esp_now.h>
#include <WiFi.h>
```
Ensuite initialiser les variables :
```int LED_BUILTIN = 22;
unsigned long time1, time2, time3;
bool stateLED;


esp_now_peer_info_t slave;
#define CHANNEL 3
#define PRINTSCANRESULTS 0
#define DELETEBEFOREPAIR 0
```
Initialiser la fonction ESPNow :
```void InitESPNow() {
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
La fonction ScanForSlave en Access Point : 
```
void ScanForSlave() {
  int8_t scanResults = WiFi.scanNetworks();
  
  bool slaveFound = 0;
  memset(&slave, 0, sizeof(slave));

  Serial.println("");
  if (scanResults == 0) {
    Serial.println("No WiFi devices in AP Mode found");
  } else {
    Serial.print("Found "); Serial.print(scanResults); Serial.println(" devices ");
    for (int i = 0; i < scanResults; ++i) {
      
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
      // Check if the current device starts with `Slave`
      if (SSID.indexOf("Slave") == 0) {
        
        Serial.println("Found a Slave.");
        Serial.print(i + 1); Serial.print(": "); Serial.print(SSID); Serial.print(" ["); Serial.print(BSSIDstr); Serial.print("]"); Serial.print(" ("); Serial.print(RSSI); Serial.print(")"); Serial.println("");
        
        int mac[6];
        if ( 6 == sscanf(BSSIDstr.c_str(), "%x:%x:%x:%x:%x:%x%c",  &mac[0], &mac[1], &mac[2], &mac[3], &mac[4], &mac[5] ) ) {
          for (int ii = 0; ii < 6; ++ii ) {
            slave.peer_addr[ii] = (uint8_t) mac[ii];
          }
        }

        slave.channel = CHANNEL; 
        slave.encrypt = 0; 

        slaveFound = 1;
        
        
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
Ici, nous allons voir si le slave est repéré par le master (parité entre le master et le slave) :
```
bool manageSlave() {
  if (slave.channel == CHANNEL) {
    if (DELETEBEFOREPAIR) {
      deletePeer();
    }

    Serial.print("Slave Status: ");
    const esp_now_peer_info_t *peer = &slave;
    const uint8_t *peer_addr = slave.peer_addr;
    
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



void sendData(uint8_t data) {
  data++;
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
Ici, on va afficher si la LED s'allume (LED ON) ou si elle est éteinte (LED OFF) :
```
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
  char macStr[18];
  snprintf(macStr, sizeof(macStr), "%02x:%02x:%02x:%02x:%02x:%02x",
           mac_addr[0], mac_addr[1], mac_addr[2], mac_addr[3], mac_addr[4], mac_addr[5]);
  Serial.print("Last Packet Sent to: "); Serial.println(macStr);
  Serial.print("Last Packet Send Status: "); Serial.println(status == ESP_NOW_SEND_SUCCESS ? "Delivery Success" : "Delivery Fail");
}

void setup() {
{
  Serial.begin(115200);
  
  WiFi.mode(WIFI_STA);
  Serial.println("Exemple");
  
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
  time3 = millis();
}
}


void loop() {
  time1 = millis();
  
  if ((time1 - time3) > 3000){
  
  
  if (slave.channel == CHANNEL) { 
    
    bool isPaired = manageSlave();
    if (isPaired) {
      
      if(stateLED)
      {
        sendData(1);
      Serial.println("LED ON");
      } else {
      
      Serial.println("LED OFF");
      }
    
    } else {
    
      ScanForSlave();
      Serial.println("Slave not paired");
  }
  }time3=millis();
  }
  time1=millis();
  if((time1 - time2)>1000)  { //Wait for 1
    if(stateLED){
  digitalWrite(LED_BUILTIN, LOW);    
  stateLED = false;
      }else {
  digitalWrite(LED_BUILTIN, HIGH);         
  stateLED=true;
      }
  time2 = millis();
}

}
```


# Troisième exercice 
L'objectif de cet exercice est de modifier le code de l'exercice précédent pour que les ESP fonctionnement de la manière suivante:

j'allume 1 ESP, il fait clignoter sa LED une fois par seconde

j'allume un second ESP, il se connecte au réseau MESH et de façon synchro, les 2 ESP font clignoter leur LED deux fois par seconde
