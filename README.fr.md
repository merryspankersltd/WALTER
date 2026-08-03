# WALTER — Routeur solaire pour ESPHome

**[🇫🇷 Français](README.fr.md) · [🇬🇧 English](README.md)**

**(WALTER Always Loves Tiny Export Rates)**

> **WALTER** est un fork de
> [hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
> ajoutant le support natif du compteur triphasé **Shelly EM3 Pro / Pro 3EM**,
> destiné à rediriger le surplus photovoltaïque vers un **chauffe-eau
> électrique**.

Walter fait exactement ce qu'il dit : il redirige le surplus photovoltaïque
vers un chauffe-eau et maintient l'injection réseau aussi faible que possible.

## Pourquoi la somme triphasée ?

Sur un abonnement triphasé, le compteur additionne les trois phases et ne
facture que la valeur **nette**, donc `+500 +800 −1200 = +100` est une
importation de 100 W — et le bon moment pour chauffer l'eau, c'est quand cette
somme devient négative, c'est-à-dire qu'il existe un vrai surplus après
compensation inter-phases :

```
S_grid = P_A + P_B + P_C     (positif = import, négatif = surplus)
```

Piloter cette somme (plutôt qu'une seule phase) protège aussi les limites par
phase : le routeur ne s'allume que lorsque *toute l'installation* est en
surplus, il ne peut donc jamais pousser une phase au-delà de sa limite. Voir
`docs/fr/strategy.md` pour le raisonnement complet.

## Matériel

| Pièce | Modèle |
|-------|--------|
| MCU | ESP32 (esp32dev, framework esp-idf) |
| Régulateur | Gradateur triac RobotDyn RBD-340 40 A « version ventilée » |
| Compteur | Shelly EM3 Pro / Pro 3EM (HTTP RPC, interrogé localement — pas besoin de HA) |
| Charge | Résistance 2400 W (chauffe-eau) |
| Sécurité | limiteur de température (DS18B20 ou température HA) |

Câblage : `docs/fr/wiring_rbd40.md` · Procédure de sécurité :
`docs/fr/bench_test.md`

## Logiciel

- **Firmware** : ESPHome. `esp32-walter-solar-heater.yaml` + le nouveau
  package `solar_router/power_meter_shelly_em3.yaml` (interrogation du
  compteur triphasé).
- **Home Assistant** : templates et automatisations d'assistance dans
  `homeassistant/` (sécurité température de cuve, mode heures pleines/creuses,
  tableau de bord).

### Configuration ESPHome (aperçu)

```yaml
packages:
  solar_router:
    url: https://github.com/hacf-fr/Solar-Router-for-ESPHome/
    ref: main
    refresh: 1d
    files:
      - path: solar_router/common.yaml
      - path: solar_router/power_meter_shelly_em3.yaml
        vars:
          power_meter_ip_address: "192.168.1.42"
      - path: solar_router/regulator_triac.yaml
        vars:
          regulator_gate_pin: GPIO17
          regulator_zero_crossing_pin: GPIO23
      - path: solar_router/engine_1dimmer.yaml
        vars:
          green_led_pin: GPIO18
          yellow_led_pin: GPIO19
      - path: solar_router/temperature_limiter_home_assistant.yaml
        vars:
          temperature_sensor: "sensor.water_tank_temperature"
          red_led_pin: GPIO21
```

## Limites connues (v1)

- Le comptage d'énergie (`total_energy_diverted`) exige que le limiteur de
  température de la cuve soit actif, sinon le compteur surestime tant que le
  thermostat du ballon est ouvert.
- Le moteur est le régulateur progressif amont ; un branchement PID /
  prévisions est en attente dans `docs/fr/strategy.md`.

## Crédits et licence

Construit à partir de
[hacf-fr/Solar-Router-for-ESPHome](https://github.com/hacf-fr/Solar-Router-for-ESPHome)
(GPL-3.0). Voir `LICENSE`. Lisez
[l'avertissement](https://hacf-fr.github.io/Solar-Router-for-ESPHome/disclamer/)
avant de toucher du 230 V.
