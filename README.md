# Home Assistant Blueprints

Blueprints Home Assistant :

- **Zendure SolarFlow** : optimiser l'autoconsommation solaire avec le tarif EDF Tempo (ci-dessous).
- **Sonoff ZBMINIR2** : trois gestes (simple, double, long) sur un poussoir mural, voir [plus bas](#sonoff-zbminir2).

# Zendure SolarFlow

## Principe

```
22h            6h           sunrise+1h        sunset          22h
  |── Charge ──|── Décharge ──|── Smart/Solar ──|── Décharge ──|
  |  nocturne  |   matinale   |  surplus stock  |   soirée     |
  |  (HC)      |   (HP)       |                 |  → réserve   |
```

- **Nuit** : charge en heures creuses si jour Blanc ou Rouge (Tempo)
- **Matin** : décharge pour couvrir les heures pleines jusqu'au soleil
- **Journée** : mode smart, le surplus solaire recharge les batteries
- **Soir** : décharge jusqu'au SOC de réserve calculé pour le lendemain matin

## Blueprints

### 🌅 Mode Jour (`zendure_mode_jour.yaml`)
Déclenché à 6h. Passe le manager en mode smart et abaisse le SOC minimum à 10% pour permettre la décharge.
À l'heure de fin (22h par défaut), passe le manager en arrêt — sauf si l'entité
optionnelle « Dérogation nuit » (un `input_boolean`, ex. mode vacances) est active :
le mode jour continue alors toute la nuit pour consommer le solaire stocké.

### 🌇 SOC Réserve Soir (`zendure_soc_reserve_soir.yaml`)
Déclenché au coucher du soleil. Calcule dynamiquement le SOC à conserver pour couvrir le lendemain matin, basé sur :
- L'heure du lever du soleil (`sun.sun`)
- La capacité totale des batteries (`sensor.zendure_manager_total_kwh`)
- La consommation moyenne estimée

### 🔋 Charge Nocturne Tempo (`zendure_charge_nocturne_tempo.yaml`)
Déclenché à 22h. Charge tous les SolarFlow à puissance max en heures creuses si le lendemain est Blanc ou Rouge.

## Installation

### Import depuis GitHub

Dans Home Assistant → Paramètres → Automatisations → Blueprints → Importer :

```
https://github.com/defts/homeassistant-blueprints/blob/master/blueprints/zendure_mode_jour/zendure_mode_jour.yaml
https://github.com/defts/homeassistant-blueprints/blob/master/blueprints/zendure_soc_reserve_soir/zendure_soc_reserve_soir.yaml
https://github.com/defts/homeassistant-blueprints/blob/master/blueprints/zendure_charge_nocturne_tempo/zendure_charge_nocturne_tempo.yaml
```

### Manuel

Copier les fichiers YAML dans `/config/blueprints/automation/defts/`.

## Configuration

Chaque blueprint est configurable via l'interface HA. Les principales valeurs :

| Paramètre | Défaut | Description |
|-----------|--------|-------------|
| Consommation matin | 500 W | Consommation moyenne 6h→soleil |
| Décalage solaire | 60 min | Délai après sunrise avant production utile |
| SOC plancher | 10 % | Protection batterie, ne jamais descendre en dessous |
| SOC réserve max | 45 % | Plafond de réserve même en hiver |
| Couleurs Tempo | Blanc, Rouge | Jours où la charge nocturne est autorisée |
| Dérogation nuit | (aucune) | `input_boolean` optionnel : actif → pas d'arrêt en fin de journée |

## Prérequis

- [Intégration Zendure](https://github.com/Zendure/ha-zendure) configurée
- [Intégration RTE Tempo](https://github.com/hekmon/rtetempo) pour le capteur couleur
- Intégration `sun` (native HA)

# Sonoff ZBMINIR2

## 🔘 Poussoir Sonoff ZBMINIR2 (`poussoir_sonoff_zbminir2.yaml`)

Associe une action à chacun des trois gestes d'un poussoir branché sur un
[Sonoff ZBMINIR2](https://www.zigbee2mqtt.io/devices/ZBMINIR2.html) en relais
détaché : **appui simple**, **double appui** et **appui long**. Idéal avec des
ampoules connectées : le relais reste fermé, le poussoir ne fait qu'envoyer des
gestes à Home Assistant.

Import :

```
https://github.com/defts/homeassistant-blueprints/blob/master/blueprints/poussoir_sonoff_zbminir2/poussoir_sonoff_zbminir2.yaml
```

| Entrée | Description |
|--------|-------------|
| Module ZBMINIR2 | Le module (seuls les ZBMINIR2 sont proposés) |
| Appui simple | Actions d'un appui court, en général basculer la lumière |
| Double appui | Actions d'un double appui rapide (facultatif) |
| Appui long | Actions d'un appui maintenu 2 à 3 s (facultatif) |

### Prérequis

- Zigbee2MQTT, module en firmware **1.1.0 ou plus** (c'est lui qui remonte le
  double appui et l'appui long).
- `detach_relay_mode` activé (relais détaché).
- `external_trigger_mode` sur **`pulse`** pour un poussoir. En `edge`
  (interrupteur classique), seul l'appui simple est remonté.
- Les déclencheurs « double » et « long » n'apparaissent dans Home Assistant
  qu'après le premier événement de chaque type : faire un double appui et un
  appui long une fois avant de configurer.

### Bon à savoir

- **L'appui long part pendant le maintien**, au bout de 2 à 3 s (seuil du
  firmware, non réglable). Relâché avant, l'appui compte comme un appui simple.
- Le firmware n'envoie **rien au relâchement** ni en continuant à maintenir :
  pas de variation continue « tant que j'appuie ». Demande en cours :
  [Koenkk/zigbee2mqtt#33593](https://github.com/Koenkk/zigbee2mqtt/issues/33593).
- **Va-et-vient** : passer les deux interrupteurs en poussoirs (un interrupteur
  restant ferait un appui long sans fin). Avec des mécanismes de va-et-vient
  transformés en poussoirs (ressort), les navettes existantes suffisent : au repos,
  S1–S2 doit être ouvert ; sinon, croiser les navettes d'un seul des deux mécanismes.
- Les poussoirs d'un même module envoient les mêmes événements : impossible de
  savoir lequel a été pressé.
- `mode: queued` : aucun appui n'est perdu, même si une action dure un peu.

## Licence

MIT
