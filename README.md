# Système d'Alarme Anti-Intrusion IoT

Projet IoT réalisé dans le cadre de mes études à l'Université Côte d'Azur.  
Un système d'alarme connecté qui détecte l'ouverture d'une porte via un accéléromètre, déclenche un buzzer local, et envoie une notification push sur téléphone via LoRaWAN.

---

## 🎯 Objectif

Concevoir un système d'alarme embarqué capable de :
- Détecter un mouvement brusque (ouverture de porte) grâce à un accéléromètre
- Déclencher une alarme sonore locale (buzzer)
- Envoyer une alerte à distance via LoRaWAN → TTN → TagoIO → téléphone

---

## 🛠️ Matériel utilisé

| Composant | Détails |
|-----------|---------|
| Carte de développement | UCA21 (ATMega328PB @ 3.3V, 8MHz) |
| Radio LoRa | RFM95W (SX1276), intégrée |
| Accéléromètre | KXTJ3-1057, intégré (I2C, adresse 0x0E) |
| Buzzer | Buzzer actif, externe |

---

## 🔌 Câblage

| Connexion | Pin |
|-----------|-----|
| Buzzer (+) | D5 |
| Buzzer (–) | GND |

Tout le reste est intégré à la carte.

---

## ⚙️ Fonctionnement

1. Au démarrage, le système calibre automatiquement l'accéléromètre (position de référence = porte fermée).
2. Si un mouvement brusque est détecté (`delta > 2.0g`) :
   - Le **buzzer sonne** immédiatement (sans réseau).
   - Une **trame LoRaWAN** est envoyée à TTN si la couverture est disponible.
3. TTN transmet la trame à **TagoIO** via webhook.
4. TagoIO envoie une **notification push** sur le téléphone.

---

## 💻 Code Arduino

```cpp
#include <Wire.h>
#include <kxtj3-1057.h>
#include <SPI.h>
#include <arduino_lmic.h>
#include <hal/hal.h>

static const u1_t PROGMEM APPEUI[8]  = { 0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00 };
void os_getArtEui (u1_t* buf) { memcpy_P(buf, APPEUI, 8); }
static const u1_t PROGMEM DEVEUI[8]  = { 0xDA,0x80,0x07,0xD0,0x7E,0xD5,0xB3,0x70 };
void os_getDevEui (u1_t* buf) { memcpy_P(buf, DEVEUI, 8); }
static const u1_t PROGMEM APPKEY[16] = { 0x6B,0x91,0x17,0xB1,0x69,0x07,0x65,0xBA,0xE5,0xB1,0xA2,0x67,0x63,0xB1,0x36,0x97 };
void os_getDevKey (u1_t* buf) { memcpy_P(buf, APPKEY, 16); }

const lmic_pinmap lmic_pins = {
  .nss = 10, .rxtx = LMIC_UNUSED_PIN, .rst = 8, .dio = {6,6,6},
};

#define BUZZER_PIN 5
KXTJ3 accel(0x0E);
const float THRESHOLD = 2.0;
float baseX, baseY, baseZ;
bool alarmActive = false;
bool joined = false;
bool needToSend = false;

void onEvent(ev_t ev) {
  if (ev == EV_JOINED) { joined = true; Serial.println("Joined TTN!"); LMIC_setLinkCheckMode(0); }
  if (ev == EV_TXCOMPLETE) Serial.println("TX complete");
}

bool readAccel(float &x, float &y, float &z) {
  Wire.beginTransmission(0x0E);
  Wire.write(0x06);
  if (Wire.endTransmission(false) != 0) return false;
  Wire.requestFrom(0x0E, 6);
  if (Wire.available() < 6) return false;
  int16_t rx = Wire.read() | (Wire.read() << 8);
  int16_t ry = Wire.read() | (Wire.read() << 8);
  int16_t rz = Wire.read() | (Wire.read() << 8);
  x = rx / 16384.0;
  y = ry / 16384.0;
  z = rz / 16384.0;
  return true;
}

void setup() {
  Serial.begin(9600);
  Wire.begin();
  delay(1000);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  accel.begin(12.5, 2, true, true);
  delay(500);
  readAccel(baseX, baseY, baseZ);
  Serial.println("Armé");
  os_init();
  LMIC_reset();
  LMIC_setClockError(MAX_CLOCK_ERROR * 2 / 100);
  LMIC_startJoining();
}

void loop() {
  float x, y, z;
  if (readAccel(x, y, z)) {
    float delta = abs(x-baseX) + abs(y-baseY) + abs(z-baseZ);
    if (!alarmActive && delta > THRESHOLD) {
      alarmActive = true;
      needToSend = true;
      Serial.println("PORTE OUVERTE !");
    }
  }

  if (needToSend && joined && !(LMIC.opmode & OP_TXRXPEND)) {
    uint8_t payload[1] = { 0x01 };
    LMIC_setTxData2(1, payload, sizeof(payload), 0);
    needToSend = false;
    Serial.println("Alerte envoyée !");
  }

  os_runloop_once();

  if (alarmActive) {
    for (int i = 0; i < 3; i++) {
      digitalWrite(BUZZER_PIN, HIGH); delay(100);
      digitalWrite(BUZZER_PIN, LOW);  delay(100);
    }
    delay(200);
  } else {
    delay(200);
  }
}
```

---

## 📁 Fichier de configuration LMIC

Créer `lmic_project_config.h` dans le dossier du sketch :

```cpp
#define CFG_eu868 1
#define CFG_sx1276_radio 1
#define LMIC_ENABLE_arbitrary_clock_error 1
```

---

## 📚 Librairies

| Librairie | Utilité |
|-----------|---------|
| MCCI LoRaWAN LMIC Library | Stack LoRaWAN |
| KXTJ3-1057 | Accéléromètre |

---

## 📡 Configuration TTN

1. Créer un compte sur [thethingsnetwork.org](https://thethingsnetwork.org)
2. Créer une Application
3. Enregistrer un End Device (OTAA, EU868, LoRaWAN 1.0.3)
4. Copier DevEUI, AppEUI, AppKey dans le code

---

## 📱 Configuration TagoIO

1. Créer un compte sur [tago.io](https://tago.io)
2. Créer un device avec le connecteur TTN
3. Ajouter ce Payload Parser :

```javascript
let alert = false;
try {
  const raw = payload[0]?.payload?.uplink_message?.frm_payload;
  if (raw) {
    const decoded = Buffer.from(raw, 'base64');
    if (decoded[0] === 1) alert = true;
  }
} catch (e) {}

payload = [{
  variable: "intruder_alert",
  value: alert,
  unit: ""
}];
```

4. Créer une Action : `intruder_alert = true` → notification push → `🚨 Intrusion détectée !`
5. Installer l'app TagoIO sur le téléphone

---

## 🔗 Webhook TTN → TagoIO

Dans TTN Console → Integrations → Webhooks → Custom webhook :
- Base URL : `https://api.tago.io`
- Header : `Device-Token: <token TagoIO>`
- Uplink message path : `/data`

---

## 🏗️ Architecture

```
Carte embarquée
(accéléromètre détecte mouvement)
        |
        +--> Buzzer sonne (sans réseau)
        |
        +--> Trame LoRaWAN
                  |
            Passerelle TTN
                  |
          The Things Network
                  |
              Webhook
                  |
              TagoIO
                  |
        Notification Push → Téléphone
```

---

## ⚠️ Notes techniques

- Le buzzer fonctionne **sans connexion réseau** — l'alarme locale est indépendante.
- Les notifications téléphone nécessitent une couverture LoRaWAN (passerelle TTN à portée).
- Problème connu : LMIC interfère avec I2C sur cette carte. Résolu en utilisant des lectures I2C directes (registres bruts) au lieu de la librairie KXTJ3 dans la boucle principale.
- `LMIC_setClockError(MAX_CLOCK_ERROR * 2 / 100)` est nécessaire pour compenser l'imprécision de l'horloge à 8MHz.

---

## 👤 Auteur

Imad — Projet IoT, Université Côte d'Azur, 2025
