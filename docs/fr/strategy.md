# WALTER — stratégie de régulation

WALTER est un **routeur solaire** pour un chauffe-eau électrique de 2400 W.
C'est un fork de
[hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
(ESPHome / Home Assistant). Il ajoute le support natif du compteur triphasé
**Shelly EM3 Pro / Pro 3EM**.

## Le problème (contrats triphasés)

Les installations résidentielles triphasées répartissent leurs charges sur les
trois phases (pièces de vie, cuisine, chauffage, froid, onduleurs PV...) qui ne
s'équilibrent presque jamais.

Sur un abonnement triphasé, le compteur traite **un seul registre arithmétique
bidirectionnel** : les puissances des trois phases sont additionnées et seule
la valeur **nette** est facturée. Une exportation de −1200 W sur une phase
compense entièrement une importation de +1300 W sur les deux autres.

Le seul signal de commande cohérent avec la facturation est donc :

```
S_grid = P_A + P_B + P_C      (W, '+' = import, '−' = surplus)
```

C'est exactement ce que le Shelly EM3 Pro fournit dans `total_act_power`, et
c'est ce que WALTER ramène à zéro :

- `S_grid > 0` → l'installation importe globalement → le chauffe-eau reste
  éteint, même si un « surplus » local existe sur une phase — les phases se
  compensent au compteur.
- `S_grid < 0` → surplus net après compensation inter-phases → WALTER grade
  le chauffe-eau pour absorber exactement ce surplus, ramenant `S_grid → 0`.

**La sécurité d'équilibre de phase vient gratuitement** : comme le routeur ne
s'allume que lorsque la somme est négative, il ne peut jamais pousser une phase
au-delà de sa limite — dans ce cas, la somme ne serait simplement pas négative.

## Logique de déviation

Le moteur est un **régulateur progressif proportionnel** (amont) :

```
delta = −(S_grid − Target) * reactivity / 1000      [%]
new_level = clamp(0..100, level + delta)
```

- Le compteur est interrogé une fois par seconde.
- `Target` (0 W par défaut) est l'échange réseau souhaité. Les valeurs
  positives maintiennent une petite importation (utile en heures creuses pour
  recharger la cuve rapidement) ; les valeurs négatives laissent fuir ce
  surplus (pertinent seulement avec un contrat de vente du surplus).
- **Réactivités montante/descendante** : gains asymétriques réglables —
  évitent l'oscillation quand le thermostat du ballon fait varier la somme.
- Sécurité : compteur injoignable / données périmées → `real_power = NaN` →
  0 %.

La relation entre l'ouverture du triac (%) et la puissance délivrée est
**non linéaire** (gradation par coupure de phase). Peu importe : la boucle se
referme sur le `S_grid` *mesuré*, pas sur une estimation en boucle ouverte,
donc elle converge vers la cible quelle que soit la charge.

## Stratégie tarifaire (heures pleines / heures creuses)

Sur les abonnements résidentiels classiques heures pleines/heures creuses, la
production PV coïncide avec les heures les plus chères — le moment idéal pour
l'autoconsommation.

- **Heures pleines (chères)** : WALTER est le moyen principal de chauffer
  l'eau, de préférence sur le surplus PV (cible 0 W → le ballon absorbe tout
  le surplus net, jusqu'à sa puissance nominale).
- **Heures creuses (bon marché)** : deux montages possibles

  1. **Contact nuit classique en amont du gradateur** (contact existant du
     ballon) : en heures creuses le contact alimente le ballon directement,
     en court-circuitant WALTER — pas de marche forcée nécessaire. Recommandé
     pour la résilience (le ballon chauffe même si WALTER/ESP est hors ligne).
  2. **Gradateur seul en série** : utiliser le planificateur de marche forcée
     de WALTER à 100 % entre Begin et End (heures creuses) pour remplacer
     l'ancien contact nuit. Inconvénient : pas de chauffage si WALTER est hors
     ligne.

- Avec un contrat **vente du surplus**, préférez `Target ≈ −100…−500 W`
  (laisser une petite fuite au réseau plutôt que de faire chauffer le ballon
  depuis le réseau). Sans contrat d'achat, gardez `Target = 0`
  (autoconsommation maximale).
- Le préchauffage par prévisions météo est une piste pour v2 (pré-chauffer en
  heures creuses avant les jours ensoleillés pour libérer le soleil du matin
  pour le reste de la maison).

## Niveaux de sécurité

1. **Limiteur de température** (température de la cuve → chute à 0 % à la
   `Stop temperature`, reprise sous la `Restart temperature`).
2. **Sécurité compteur** : pas de données / données périmées → ballon éteint.
3. **Le thermostat du ballon reste dans le circuit** (protège aussi à 100 %).
4. Test sur banc avant installation (voir `bench_test.md`).

## Pas en v1 (en attente)

- Charges lourdes supplémentaires (p. ex. un second chauffe-eau) — chacune
  avec son propre routeur ou une stratégie de partage du surplus.
- Préchauffage par prévisions météo et moteur type PID.
- Capteur de production PV (nécessite un canal EM libre ou l'API du
  micro-onduleur) pour calculer la consommation réelle avant le ballon et
  améliorer le comptage d'énergie.
- Supervision de la limite de puissance souscrite (Grid-limit).

## Pourquoi un fork plutôt qu'un projet existant

- **hacf-fr/Solar-Router-for-ESPHome** — le plus proche du besoin : ESPHome,
  packages séparés (compteur / moteur / régulateur / sécurité /
  planificateur) ; il ne manquait que le support EM3 triphasé.
- **frtz13/Zero-Surplus-Dimmer** — boucle simple pour ESP8266, sans couches de
  sécurité.
- **x-real-ip/zero-grid** — PID + DAC + régulateur de tension externe, autre
  famille matérielle ; son concept « P_grid → 0 » équivaut à la cible de
  WALTER.
- **robotdyn-dimmer/ACRouter** — ESP-IDF natif (pas ESPHome), concept de modes
  intéressant (OFF / AUTO / ECO / BOOST) ; gardé comme modèle mental seulement
  (voir les automatisations Home Assistant pour les modes équivalents
  OFF / AUTO / MARCHE FORCÉE).