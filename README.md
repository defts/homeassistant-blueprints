# Home Assistant Blueprints — Zendure SolarFlow

Blueprints Home Assistant pour optimiser l'autoconsommation solaire avec des systèmes Zendure SolarFlow et le tarif EDF Tempo.

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
https://github.com/defts/homeassistant-blueprints/blob/main/blueprints/zendure_mode_jour/zendure_mode_jour.yaml
https://github.com/defts/homeassistant-blueprints/blob/main/blueprints/zendure_soc_reserve_soir/zendure_soc_reserve_soir.yaml
https://github.com/defts/homeassistant-blueprints/blob/main/blueprints/zendure_charge_nocturne_tempo/zendure_charge_nocturne_tempo.yaml
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

## Prérequis

- [Intégration Zendure](https://github.com/Zendure/ha-zendure) configurée
- [Intégration RTE Tempo](https://github.com/hekmon/rtetempo) pour le capteur couleur
- Intégration `sun` (native HA)

## Licence

MIT
