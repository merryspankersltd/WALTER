# Câblage — ESP32 + RobotDyn gradateur AC 40 A « avec capteur de courant »

> ⚠️ Il s'agit d'une installation **230 V**. Coupez le circuit, respectez les
> règles locales et faites vérifier le raccordement final par un électricien
> qualifié. Le côté secteur du gradateur et du ballon est sous tension — ne
> jamais y toucher sous tension.

## Module utilisé

Gradateur triac style RobotDyn, 1 canal, **40 A**, variante premium « avec
capteur de courant » (NTC sur dissipateur + ventilateur 5 V + capteur de
courant CT intégré). Il comprend :

* **Côté secteur isolé** : `AC-IN` / `AC-OUT` (monté en série avec la charge)
* **Détecteur de passage à zéro** : sortie `Z-C` (une impulsion à chaque
  passage à zéro du secteur)
* **Entrée de gâchette triac** : `DIM` — le signal de commande de phase
* **Côté commande** : `VCC` (logique 3,3 V — la carte supporte 3,3/5 V), `GND`
* **NTC de dissipateur** : sortie analogique `TEMP` (température du
  dissipateur)
* **Ventilateur de refroidissement** : `5V` + `FAN` (commande de vitesse) +
  connecteur 2 broches `FAN CON` (fils du ventilateur)
* **Capteur de courant** : sortie analogique `CUR` (courant de charge, CT) +
  un petit potentiomètre « current precision » pour régler son gain

### Brochage côté basse tension (marquage de la carte)

Groupe de 8 broches :

| `TEMP` | `FAN` | `CUR` | `5V` |
|---|---|---|---|
| `VCC` | `GND` | `Z-C` | `DIM` |

plus :

* un potentiomètre tournevis marqué **« current precision »** — calibrage du
  gain du CT (à régler sur banc, voir `bench_test.md`)
* un connecteur 2 broches marqué **`FAN CON`** — le ventilateur lui-même
  (alimenté par le rail 5 V de la carte)

### Raccordements (ESP32)

| Broche gradateur | Broche ESP32 (exemple) | Rôle ESPHome |
|---|---|---|
| `Z-C` | `GPIO23` | `regulator_zero_crossing_pin` (entrée interruption) |
| `DIM` | `GPIO17` | `regulator_gate_pin` (gâchette triac) |
| `VCC` | `3V3` | alimentation logique + capteurs — **utiliser 3,3 V** pour que `TEMP`/`CUR` restent dans la plage ADC |
| `GND` | `GND` | masse commune, côté isolé |
| `TEMP` | `GPIO34` | `dimmer_temp_pin` (entrée ADC, NTC dissipateur) |
| `CUR` | `GPIO35` | `dimmer_current_pin` (entrée ADC, CT) |
| `5V` | `5V` | rail 5 V de la section ventilateur |
| `FAN` | relié à `5V` | **refroidissement câblé en permanence à 100 %** (voir ci-dessous) |
| `FAN CON` | — | connecteur ventilateur (déjà câblé sur la carte) |

> **Ventilateur à 100 % en permanence — par conception.** Reliez la broche
> `FAN` à la broche `5V` pour que le ventilateur tourne à pleine vitesse même
> avant le démarrage du firmware (et même si l'ESP ou le Wi-Fi meurt). Un
> ventilateur piloté par le firmware ajoute un mode de défaillance (contrôleur
> bloqué = surchauffe silencieuse) ; le ventilateur permanent le supprime.
> Bruit et durée de vie sont sans importance dans un local technique.

> **`VCC` en 3,3 V.** Les sorties `TEMP` et `CUR` sont des diviseurs/sorties CT
> référencées à `VCC` ; alimenter la carte en 5 V peut les faire dépasser 3,3 V
> et endommager les broches ADC de l'ESP32. La carte supporte officiellement la
> logique 3,3 V.

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
| `GPIO34` | NTC dissipateur (`dimmer_temp_pin`, ADC1 entrée seule) |
| `GPIO35` | CT courant de charge (`dimmer_current_pin`, ADC1 entrée seule) |
| `GPIO18` | LED verte (régulation active / déviation) |
| `GPIO19` | LED jaune (réseau / état du compteur) |
| `GPIO21` | LED rouge (sécurité gradateur active : surchauffe / surintensité) |

Remarques :

- Tous les GPIO ESP32 peuvent détecter le passage à zéro, mais la gâchette doit
  être sur une broche capable de sortie (utiliser les GPIO classiques 0-33, pas
  les entrées seules 34-39).
- `TEMP` et `CUR` sont analogiques — utiliser des broches ADC (ADC1 de l'ESP32
  : 32-39 ; l'exemple utilise les entrées seules 34/35).
- Gardez le fil Z-C court et éloigné du côté AC pour éviter de fausses
  impulsions dues aux commutations du triac ; le composant `ac_dimmer`
  d'ESPHome les filtre.
