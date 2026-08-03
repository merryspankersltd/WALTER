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

## 5. Test du limiteur de température

- [ ] Alimentez une valeur de test (p. ex. un `input_number`) dans l'entité
      `temperature_sensor`
- [ ] Montez-la au-dessus de `Stop temperature` → `Safety limit reached` = ON,
      LED rouge, et le routeur doit tomber à 0 %
- [ ] Descendez-la sous `Restart temperature` → la régulation reprend

## 6. Avant le vrai ballon

- [ ] Sur le banc, confirmez la marge de puissance : la résistance du ballon
      fait au maximum 2400 W, et le routeur ne doit jamais commander plus. Les
      limites par phase sont respectées automatiquement car le routeur ne
      s'allume que lorsque la somme triphasée est négative
- [ ] Ne remplacez la charge de banc par la résistance du ballon qu'après un
      historique propre : plusieurs heures de comportement automatique correct
      en plein soleil / nuages
- [ ] Une fois installé sur le ballon, pointez le limiteur de température vers
      la température réelle de la cuve

Ne repassez pas en mode sans surveillance : les premiers jours, vérifiez le
tableau de bord HA / `energy_diverted` et la courbe de température de la cuve
avant de faire confiance au système.