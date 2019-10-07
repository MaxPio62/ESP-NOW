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
Tout d'abord, initialiser les variables :
```
#include <esp_now.h>
#include <WiFi.h> 
```

 


# Troisième exercice 
L'objectif de cet exercice est de modifier le code de l'exercice précédent pour que les ESP fonctionnement de la manière suivante:

j'allume 1 ESP, il fait clignoter sa LED une fois par seconde

j'allume un second ESP, il se connecte au réseau MESH et de façon synchro, les 2 ESP font clignoter leur LED deux fois par seconde
