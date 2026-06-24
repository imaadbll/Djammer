# Système d'Alarme Anti-Intrusion — UCA21 + LoRaWAN + TagoIO

Un système d'alarme IoT pour détecter l'ouverture d'une porte, construit sur la carte **Fabien Ferrero UCA21 (2021, RFThings)**.

Lorsque la porte est ouverte, l'accéléromètre intégré détecte le mouvement, déclenche un buzzer local, et envoie une alerte via LoRaWAN vers The Things Network (TTN), qui transmet ensuite une notification push sur le téléphone via TagoIO.

---

## 🔧 Fonctionnement

1. Au démarrage, le système **s'arme automatiquement** et calibre l'accéléromètre (position actuelle = porte fermée).
2. Si la porte est ouverte et qu'un grand mouvement est détecté (`delta > 2.0g`) :
   - Le **buzzer sonne en continu** — fonctionne même sans réseau.
   - Si le board a rejoint TTN, une **trame LoRaWAN** (`0x01`) est envoyée.
3. TTN transmet la trame à **TagoIO** via un webhook.
4. TagoIO décode le payload et envoie une **notification push** sur le téléphone.
5. Appuyer sur **BT1 (D3)** pour désarmer et recalibrer.

---

## 🛒 Matériel

| Composant | Détails |
|-----------|---------|
| Carte | UCA21 (Fabien Ferrero / RFThings, 2021) |
| MCU | ATMega328PB @ 3.3V, 8MHz |
| Radio LoRa | RFM95W (SX1276), intégré |
| Accéléromètre | KXTJ3-1057, intégré (I2C, adresse 0x0E) |
| Buzzer | Buzzer actif, externe |
| Bouton désarmement | BT1 (D3), intégré |

---

## 🔌 Câblage

| Connexion | Pin |
|-----------|-----|
| Buzzer (+) | D5 |
| Buzzer (–) | GND |

Tout le reste (accéléromètre, radio LoRa, boutons) est déjà intégré à la carte.

---

## 📌 Mapping des pins (UCA21)

```
ATMega328PB        RFM95W
D8          <----> RST
D12 (MISO)  <----> MISO
D11 (MOSI)  <----> MOSI
D13 (SCK)   <----> CLK
D10 (SS)    <----> SEL
D6          <----> DIO0 / DIO1 / DIO2
D2          <----> BT0
D3          <----> BT1
```

---

## 📚 Librairies

| Librairie | Utilité |
|-----------|---------|
| MCCI LoRaWAN LMIC Library | Stack LoRaWAN (OTAA, uplink) |
| KXTJ3-1057 | Driver accéléromètre |

Installer via le gestionnaire de librairies Arduino IDE.

Créer également un fichier `lmic_project_config.h` dans le dossier du sketch :

```cpp
#define CFG_eu868 1
#define CFG_sx1276_radio 1
#define LMIC_ENABLE_arbitrary_clock_error 1
```

---

## 💻 Code Arduino

```cpp
#include <arduino_lmic.h>
#include <hal/hal.h>
#include <SPI.h>
#include <Wire.h>
#include <kxtj3-1057.h>

static const u1_t PROGMEM APPEUI[8]  = { 0x00,0x00,0x00,0x00,0x00,0x00,0x00,0x00 };
void os_getArtEui (u1_t* buf) { memcpy_P(buf, APPEUI, 8); }
static const u1_t PROGMEM DEVEUI[8]  = { 0xDA,0x80,0x07,0xD0,0x7E,0xD5,0xB3,0x70 };
void os_getDevEui (u1_t* buf) { memcpy_P(buf, DEVEUI, 8); }
static const u1_t PROGMEM APPKEY[16] = { 0x6B,0x91,0x17,0xB1,0x69,0x07,0x65,0xBA,0xE5,0xB1,0xA2,0x67,0x63,0xB1,0x36,0x97 };
void os_getDevKey (u1_t* buf) { memcpy_P(buf, APPKEY, 16); }

const lmic_pinmap lmic_pins = {
  .nss = 10,
  .rxtx = LMIC_UNUSED_PIN,
  .rst = 8,
  .dio = {6, 6, 6},
};

#define BUZZER_PIN 5
#define BT_DISARM  3

KXTJ3 accel(0x0E);
const float THRESHOLD = 2.0;
float baseX, baseY, baseZ;
bool alarmActive = false;
bool joined = false;

void calibrate() {
  delay(500);
  baseX = accel.axisAccel(X);
  baseY = accel.axisAccel(Y);
  baseZ = accel.axisAccel(Z);
  Serial.println("Armé & calibré");
}

void sendAlert() {
  if (!joined || (LMIC.opmode & OP_TXRXPEND)) return;
  uint8_t payload[1] = { 0x01 };
  LMIC_setTxData2(1, payload, sizeof(payload), 0);
  Serial.println("Alerte envoyée !");
}

void onEvent(ev_t ev) {
  if (ev == EV_JOINED) {
    joined = true;
    Serial.println("Connecté à TTN !");
    LMIC_setLinkCheckMode(0);
  }
  if (ev == EV_TXCOMPLETE) Serial.println("TX terminé");
}

void setup() {
  Serial.begin(9600);
  Wire.begin();
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BT_DISARM, INPUT_PULLUP);
  digitalWrite(BUZZER_PIN, LOW);
  accel.begin(12.5, 2, true, true);
  calibrate();
  os_init();
  LMIC_reset();
  LMIC_startJoining();
  Serial.println("Connexion à TTN...");
}

void loop() {
  os_runloop_once();

  if (digitalRead(BT_DISARM) == LOW) {
    alarmActive = false;
    digitalWrite(BUZZER_PIN, LOW);
    calibrate();
    delay(500);
  }

  if (!alarmActive) {
    float x = accel.axisAccel(X);
    float y = accel.axisAccel(Y);
    float z = accel.axisAccel(Z);
    float delta = abs(x-baseX) + abs(y-baseY) + abs(z-baseZ);
    if (delta > THRESHOLD) {
      alarmActive = true;
      Serial.println("PORTE OUVERTE !");
      sendAlert();
    }
  }

  if (alarmActive) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(300);
    digitalWrite(BUZZER_PIN, LOW);
    delay(300);
  }
}
```

---

## 📡 Configuration TTN

1. Créer un compte sur [thethingsnetwork.org](https://thethingsnetwork.org).
2. Créer une **Application**.
3. Enregistrer un **End Device** manuellement :
   - Plan de fréquence : EU868
   - Version LoRaWAN : 1.0.3
   - Activation : OTAA
4. Copier **DevEUI**, **AppEUI**, **AppKey** dans le code.

---

## 📱 Configuration TagoIO

1. Créer un compte sur [tago.io](https://tago.io).
2. Créer un device avec le connecteur **TTN**.
3. Ajouter ce **Payload Parser** :

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

4. Créer une **Action** : déclencher quand `intruder_alert = true` → notification push → `🚨 Porte ouverte - intrusion détectée !`
5. Installer l'**app TagoIO** sur le téléphone et autoriser les notifications.

---

## 🔗 Webhook TTN → TagoIO

Dans TTN Console → Application → Integrations → Webhooks → Custom webhook :
- Base URL : `https://api.tago.io`
- Header : `Device-Token: <votre token TagoIO>`
- Uplink message path : `/data`

---

## ⚙️ Réglage de la sensibilité

```cpp
const float THRESHOLD = 2.0;
```

Augmenter pour réduire les fausses alertes. Diminuer pour augmenter la sensibilité.

---

## 🏗️ Architecture

```
Carte UCA21
(accéléromètre détecte un mouvement)
        |
        +--> Buzzer sonne immédiatement (sans réseau)
        |
        +--> Trame LoRaWAN (si couverture TTN disponible)
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

## ⚠️ Notes

- Le **buzzer fonctionne immédiatement** au démarrage, sans connexion réseau.
- Les notifications téléphone nécessitent une couverture TTN (fonctionne sur le campus).
- Les trois pins DIO du RFM95W sont connectés au pin D6 sur cette carte.

---

## 👤 Auteur

Imad — Projet IoT Universitaire, 2025  
Carte : UCA21 de Fabien Ferrero / RFThings Vietnam
