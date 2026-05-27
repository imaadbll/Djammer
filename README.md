# Djammer 🎸📡

**Djammer** est un instrument de musique électronique et sans fil innovant. En combinant la détection de mouvement par accéléromètre et la technologie de communication radio longue portée (LoRa), le Djammer permet de contrôler des fréquences sonores et de jouer de la musique en temps réel d'un simple geste du poignet.

Ce projet a été développé dans le cadre du module **Connexion sans fil** de notre cursus en Électronique.

---

## 📝 Présentation du Projet

Le système est basé sur une architecture Émetteur/Récepteur composée de deux cartes microcontrôleurs distinctes :

1. **Le Bracelet Émetteur (Wrist Board) :** Porté au poignet, il utilise un accéléromètre pour mesurer l'inclinaison de l'axe Y en temps réel. Ces données de mouvement sont instantanément encapsulées et envoyées par radio.
2. **La Station Réceptrice (Buzzer Board) :** Placée à distance, elle intercepte les paquets radio LoRa, analyse la valeur d'inclinaison reçue et la convertit instantanément en une fréquence musicale envoyée à un buzzer piézoélectrique.

Le résultat est un thérémine moderne et portable : inclinez votre main vers le haut ou vers le bas pour faire varier la hauteur des notes de manière fluide et interactive !

---

## 🛠️ Matériel Requis (Hardware)

Le projet s'appuie sur la plateforme matérielle universitaire **UCA21**, configurée avec les composants suivants :

* **Microcontrôleurs :** 2× Cartes de développement basées sur l'architecture ATmega328P.
* **Capteur de mouvement :** 1× Accéléromètre analogique (intégré sur la carte émettrice pour mesurer le pitch/l'inclinaison).
* **Module Radio :** 2× Transceivers LoRa (Semtech SX127x ou équivalent) opérant sur les bandes de fréquences autorisées pour la communication à basse puissance.
* **Actionneur Sonore :** 1× Buzzer piézoélectrique (intégré sur la carte réceptrice pour la génération des tonalités).
* **Alimentation :** Câbles Mini-USB pour la programmation et l'alimentation autonome via une batterie externe (Power Bank) ou un ordinateur.

---

## 📡 Focus Module : Connexion sans fil

L'originalité du **Djammer** réside dans l'exploitation de la technologie **LoRa (Long Range)**, étudiée au cours de ce module. 

Bien que LoRa soit traditionnellement utilisé pour l'Internet des Objets (IoT) sur de très longues distances avec un taux de rafraîchissement faible, nous l'avons ici optimisé pour une communication locale à **haute performance**. 

### Spécifications de la liaison sans fil :
* **Latence ultra-faible :** Un paquet de données d'inclinaison est envoyé toutes les **30 millisecondes**, assurant une réponse sonore instantanée sans décalage perceptible pour le musicien.
* **Protocole Point-à-Point :** Communication directe entre l'émetteur et le récepteur sans passer par une passerelle ou un réseau tiers (mode Peer-to-Peer).
* **Robustesse :** Haute immunité aux interférences grâce à la modulation par étalement de spectre (Chirp Spread Spectrum).

---

## 💻 Structure du Code Source

Pour contourner les limitations de mise en cache de l'IDE Arduino et garantir une compilation portable, le code intègre la bibliothèque matérielle localement. Chaque croquis est structuré de la manière suivante :

* `sketch_may27a.ino` : Le programme principal (Gestion des boucles de lecture/transmission pour l'émetteur ou de réception/génération pour le récepteur).
* `UCA_Board.h` : Le fichier d'en-tête qui déclare la classe de contrôle de la carte et les prototypes des fonctions matérielles (`initLoRa`, `getAccelerationY`, etc.).
* `UCA_Board.cpp` : L'implémentation bas niveau des registres du microcontrôleur, les configurations des broches SPI pour le module LoRa, et les calculs de conversion de l'accéléromètre.

---

## 🚀 Installation et Utilisation

### 1. Configuration de l'IDE Arduino
* Sélectionnez le type de carte : **UCA21 (Advanced options)**.
* Configurez le processeur / chip sur : **ATmega328P** (pour éviter les erreurs de signature d'EEPROM).

### 2. Déploiement
1. Flashez la carte Réceptrice avec le code du synthétiseur/buzzer.
2. Flashez la carte Émettrice avec le code du capteur de poignet.
3. Ouvrez le **Moniteur Série** (réglé sur **115200 baud**) pour observer la transmission des valeurs de tilt en direct :
   ```text
   Sent tilt: 0.15
   Sent tilt: -1.42
   Sent tilt: -3.73