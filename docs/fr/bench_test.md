# Test sur banc — à faire avant de laisser WALTER piloter le ballon

WALTER (et le moteur Solar Router for ESPHome sur lequel il repose) peut
commuter de la puissance réelle. Réalisez toute la procédure sur un banc
d'abord, avec une charge résistive factice (p. ex. un radiateur halogène ou
des lampes à incandescence, loin du ballon final), et ne passez au ballon
qu'une fois chaque étape validée.

> ⚠️ La logique de sécurité du limiteur de température peut contenir des bugs :
> validez soigneusement le comportement avant de laisser le système
> fonctionner sans surveillance (avertissement amont).

## 1. Vérifications statiques (ESP seul, sans charge AC)

Alimentez l'ESP en USB uniquement.

- [ ] L'ESP se connecte au Wi-Fi et l'API Home Assistant voit le nœud
- [ ] `http://<walter-ip>/` (web_server) répond
- [ ] Aucune LED rouge de sécurité allumée

## 2. Mise en service du compteur (sans routage)

Activez le compteur d'énergie (interrupteur « Power Meter » dans HA, ou
réglez `power_meter_activated_at_start: "1"`).

- [ ] Les journaux montrent `EM3 response status: 200`
- [ ] `sensor.real_power` suit l'installation — c'est la **somme arithmétique
      triphasée** ; comparez-la à la somme des puissances des trois canaux
      Shelly à un moment où les phases diffèrent
- [ ] Convention de signe correcte : positif = import, négatif = export. Si
      inversée, réglez `power_sign: "-1"` dans les vars du compteur
- [ ] Faites varier quelques centaines de watts dans la maison ; `real_power`
      réagit en une ou deux secondes

## 3. Test manuel du gradateur (charge de banc)

- [ ] Le ventilateur tourne déjà à 100 % au démarrage (FAN câblé sur 5 V — sans
      intervention du firmware)
- [ ] `Dimmer Heatsink Temperature` indique l'ambiance (voir calibrage)
- [ ] `Dimmer Load Current` ≈ 0 A régulateur fermé
- [ ] `Regulator Opening` à 0 % → lampe éteinte
- [ ] 30 % → lampe nettement atténuée, sans scintillement / ronronnement
- [ ] 100 % → lampe pleinement allumée
- [ ] Balayez 0-100 % ; sous ~5 % le triac peut s'éteindre (normal)
- [ ] Surveillez le dissipateur et le ventilateur : tièdes mais pas brûlants à
      100 % pendant 10 min

## 4. Mode automatique (charge de banc)

- [ ] Connectez la charge de banc à la place du ballon
- [ ] Activez `Activate Solar Routing`
- [ ] Réglez d'abord `Target grid exchange` à une petite valeur **positive**
      (p. ex. `+100`) pour observer la réaction
- [ ] Quand l'installation passe en surplus net (production PV supérieure aux
      consommations après compensation inter-phases), WALTER monte la charge
- [ ] Interrompez le surplus (p. ex. surchargez une phase pour rendre la somme
      positive) : WALTER doit réduire la charge en quelques secondes
- [ ] Coupez le compteur (débranchez / éteignez le Shelly) : `real_power`
      devient `NaN`, le routeur tombe à 0 % en ~1 s (sécurité)
- [ ] Rétablissez le compteur : la régulation reprend

## 5. Calibrage de la sécurité carte (NTC dissipateur + capteur de courant)

- [ ] **NTC dissipateur** : carte à l'ambiance, comparez `Dimmer Heatsink
      Temperature` à un thermomètre posé sur le dissipateur ; si écart,
      ajustez les vars `ntc_*` (résistance série / configuration du diviseur)
      dans `dimmer_safety.yaml`. Vérifiez sur un second point : réchauffez le
      dissipateur (sèche-cheveux) à ~50-60 °C et re-vérifiez
- [ ] **Capteur de courant** : à 100 % d'ouverture avec la charge de banc,
      comparez `Dimmer Load Current` à une pince ampèremétrique sur le fil
      AC-OUT ; ajustez le potentiomètre « current precision » (et/ou
      `current_calibration_factor`) jusqu'à concordance. Re-vérifiez à ~30 %
      (la coupure de phase n'est pas sinusoïdale — le RMS compte)
- [ ] `Current Sensor Failure` est OFF une fois le CT lu
- [ ] `Triac Stuck ON` est OFF régulateur fermé et charge connectée

## 6. Tests de sécurité

- [ ] **Surchauffe du dissipateur** : chauffez le dissipateur près du NTC
      au-dessus de `heatsink_stop_temperature` (défaut 80 °C) → `Dimmer Safety
      Active` = ON, LED rouge, le routeur tombe à 0 % ; une fois refroidi sous
      `heatsink_restart_temperature` (défaut 60 °C) → la régulation reprend
- [ ] **Surintensité** : réglez temporairement `overcurrent_current` sous le
      courant réel de la charge de banc (p. ex. `"3.0"`) → le routeur tombe à
      0 % en ~1 s, LED rouge allumée ; restaurez la valeur par défaut ensuite
- [ ] **Alarme sans puissance** : routeur forcé à 100 % (`Router Level`
      manuel) et charge débranchée → `Boiler Not Powered` = ON
- [ ] **Sécurité perte de capteur** : débranchez le fil `TEMP` → la sécurité
      s'engage (LED rouge, 0 %) ; rebranchez → reprise à la lecture suivante

## 7. Avant le vrai ballon

- [ ] Sur le banc, confirmez la marge de puissance : la résistance du ballon
      fait au maximum 2400 W, et le routeur ne doit jamais commander plus. Les
      limites par phase de l'abonnement sont respectées automatiquement car le
      routeur ne s'allume que lorsque la somme triphasée est négative
- [ ] Ne remplacez la charge de banc par la résistance du ballon qu'après un
      historique propre : plusieurs heures de comportement automatique correct
      en plein soleil / nuages
- [ ] Une fois installé sur le ballon, surveillez la température de cuve dans
      HA — le thermostat du ballon reste la protection finale

Ne repassez pas en mode sans surveillance : les premiers jours, vérifiez le
tableau de bord HA / `energy_diverted` et la courbe de température de la cuve
avant de faire confiance au système.
