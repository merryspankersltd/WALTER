# Câblage — ESP32 + RBD-340 (gradateur 40 A « version ventilée »)

> ⚠️ Il s'agit d'une installation **230 V**. Coupez le circuit, respectez les
> règles locales et faites vérifier le raccordement final par un électricien
> qualifié. Le côté secteur du gradateur et du ballon est sous tension — ne
> jamais y toucher sous tension.

## Module utilisé

Gradateur triac style RobotDyn, 1 canal, **40 A**, avec dissipateur +
ventilateur de refroidissement (référencé « RBD-340 / gradateur 40 A version
ventilée » ou équivalent). Il comprend :

* **Côté secteur isolé** : `AC-IN` / `AC-OUT` (monté en série avec la charge)
* **Détecteur de passage à zéro** : sortie `Z-C` (une impulsion à chaque
  passage à zéro du secteur)
* **Entrée de gâchette triac** : `DIM` / `PWM` — le signal de commande de phase
* **Côté commande** : `VCC` (logique 3,3 V — les vrais RobotDyn acceptent le
  3,3 V), `GND`
* **Ventilateur** : alimenté par un **rail 5 VDC séparé** sur la version 40 A ;
  sur certaines révisions, le ventilateur est auto-géré thermiquement

| Broche du gradateur | Broche ESP32 (exemple) | Rôle ESPHome |
|---|---|---|
| `Z-C` | `GPIO23` | `regulator_zero_crossing_pin` (entrée interruption) |
| `DIM` | `GPIO17` | `regulator_gate_pin` (gâchette triac) |
| `VCC` | `3V3` | alimentation logique du module |
| `GND` | `GND` | masse commune, côté isolé |
| `5V` (ventilateur) | `5V` | alimentation auxiliaire du ventilateur uniquement |
| `FAN`* | `GPIO25` (optionnel) | commande ventilateur sur les révisions qui l'exposent |

\* le ventilateur est généralement auto-géré par le contrôleur thermique
embarqué ; si la carte expose une entrée PWM `FAN` à la place, connectez-la à
un GPIO PWM et activez-la dès que la charge commute.

### Côté puissance (230 V)

| AC-IN | alimentation fixe (phase, en amont du ballon) |
|---|---|
| AC-OUT | vers la résistance du ballon (série) |
| N | neutre direct vers le ballon (ne jamais gradateur le neutre) |

Le thermostat du ballon reste dans le circuit et protège même à 100 %
d'ouverture.

## Circuit recommandé

- Disjoncteur 16 A et circuit dédié (le ballon est une alimentation permanente
  de pièce humide selon la NF C 15-100 ; un DDR 30 mA comme exigé)
- Section des fils ≥ 2,5 mm² (10,4 A nominal à 2400 W, très largement
  dimensionnée)
- Alimentation 5 V du ventilateur : petite alimentation régulée 5 V, séparée
  de l'alimentation ESP et du secteur

## Broches ESP32 utilisées dans la configuration d'exemple

| GPIO | Fonction |
|------|----------|
| `GPIO17` | gâchette triac (`regulator_gate_pin`) |
| `GPIO23` | détecteur de passage à zéro (`regulator_zero_crossing_pin`) |
| `GPIO18` | LED verte (régulation active / déviation) |
| `GPIO19` | LED jaune (réseau / état du compteur) |
| `GPIO21` | LED rouge (limite de température atteinte) |

Remarques :

- Tous les GPIO ESP32 peuvent détecter le passage à zéro, mais la gâchette doit
  être sur une broche capable de sortie (utiliser les GPIO classiques 0-33, pas
  les entrées seules 34-39).
- Gardez le fil Z-C court et éloigné du côté AC pour éviter de fausses
  impulsions dues aux commutations du triac ; le composant `ac_dimmer`
  d'ESPHome les filtre.