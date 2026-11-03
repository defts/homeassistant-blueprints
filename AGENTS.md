# Blueprints Home Assistant — consignes pour les agents

## Projet

Blueprints Home Assistant publics :

- Zendure SolarFlow (SF800 Plus, SF2400 AC Pro) avec optimisation tarifaire Tempo EDF (l'essentiel de ce fichier) ;
- poussoir mural sur Sonoff ZBMINIR2 (Zigbee2MQTT) : appui simple / double / long.

Repo **public** : rien de propre à l'installation (noms de pièces, IP, domaines, chemins de la
config perso) dans les blueprints, la doc ou ce fichier.

## Contexte matériel

- 6 panneaux solaires Jolywood bifaciaux 500W (3 kWc crête)
- SolarFlow 800 Plus : 2 panneaux, batterie intégrée 1.92 kWh
- SolarFlow 2400 AC Pro : 4 panneaux, batterie intégrée 2.88 kWh
- Capacité totale : 4.8 kWh (extensible avec AB2000/AB3000L)
- Zendure Manager : pilote les 2 SolarFlow via `select.zendure_manager_operation`

## Contexte tarifaire

- Abonnement EDF Tempo : 3 couleurs de jours (Bleu, Blanc, Rouge)
- Heures creuses : 22h00 - 06h00 (moins cher, idéal pour charger)
- Heures pleines : 06h00 - 22h00 (plus cher, on décharge les batteries)
- Capteur Tempo : `sensor.rte_tempo_prochaine_couleur` (valeurs : Bleu, Blanc, Rouge)

## Stratégie d'optimisation

Le cycle quotidien est découpé en 4 phases :

1. **Nuit (22h→6h)** — Charge nocturne en heures creuses selon couleur Tempo
2. **Matin (6h→soleil)** — Décharge pour couvrir les heures pleines, SOC min à 10%
3. **Journée (soleil→sunset)** — Mode smart, stockage du surplus solaire
4. **Soir (sunset→22h)** — Décharge jusqu'au SOC réserve calculé pour le lendemain

Le "SOC réserve" est dynamique : il se base sur `sun.sun` (lever du soleil) et la capacité batterie réelle (`sensor.zendure_manager_total_kwh`) pour calculer combien de % garder pour couvrir le matin suivant.

## Structure du repo

```
blueprints/
  zendure_mode_jour/zendure_mode_jour.yaml                           # Mode smart + SOC min jour
  zendure_soc_reserve_soir/zendure_soc_reserve_soir.yaml             # Calcul SOC réserve au sunset
  zendure_charge_nocturne_tempo/zendure_charge_nocturne_tempo.yaml       # Charge nocturne Tempo
  poussoir_sonoff_zbminir2/poussoir_sonoff_zbminir2.yaml             # Poussoir ZBMINIR2 : simple/double/long
```

Déploiement : copier le YAML dans `/config/blueprints/automation/defts/`, puis recharger les
automatisations. La branche par défaut est
`master` (les URL d'import et `source_url` doivent pointer dessus).

Chaque blueprint est indépendant et réutilisable. Les entités SolarFlow sont sélectionnées via des target selectors (N SolarFlow supportés).

## Conventions

- **Langue** : Labels et descriptions en français (utilisateurs cibles francophones)
- **Format** : YAML standard Home Assistant Blueprint
- **Entités** : Toujours passées en `input`, jamais hardcodées
- **Valeurs par défaut** : Fournies pour tous les paramètres numériques
- **Sécurité batterie** : SOC plancher à 10% minimum (jamais vider complètement)
- **Fallbacks** : Les templates Jinja2 utilisent `| float(valeur_par_defaut)` pour éviter les erreurs si un sensor est indisponible
- **Mode** : `mode: single` sur les blueprints Zendure (pas d'exécution parallèle) ;
  `mode: queued` pour les boutons (aucun appui perdu)
- **Commits publics** : pas d'attribution IA (ni Co-Authored-By, ni « Generated with »)

## Entités Home Assistant utilisées

| Type | Exemples | Rôle |
|------|----------|------|
| `select.zendure_manager_operation` | Manager global | smart, off, manual... |
| `number.sfXXX_input_limit` | Par SolarFlow | Puissance de charge (W) |
| `number.sfXXX_soc_set` | Par SolarFlow | SOC minimum (%) |
| `select.sfXXX_ac_mode` | Par SolarFlow | Mode AC (input/output) |
| `sensor.zendure_manager_total_kwh` | Manager global | Capacité totale (kWh) |
| `sensor.rte_tempo_prochaine_couleur` | RTE Tempo | Couleur du lendemain |
| `sun.sun` | HA natif | Lever/coucher soleil |

## Tests

Pour valider le calcul du SOC réserve dans les Outils de développement → Modèle :

```jinja2
{% set sunrise = state_attr('sun.sun', 'next_rising') | as_datetime | as_local %}
{% set offset = 60 %}
{% set effective = sunrise + timedelta(minutes=offset) %}
{% set morning = (now() + timedelta(days=1)).replace(hour=6, minute=0, second=0, microsecond=0) %}
{% set hours = ((effective - morning).total_seconds() / 3600) %}
{% set hours = [hours, 0] | max %}
{% set capacity = states('sensor.zendure_manager_total_kwh') | float(4.8) * 1000 %}
{% set soc = (hours * 500 / capacity * 100) | round(0) %}
Sunrise demain: {{ sunrise.strftime('%H:%M') }}
Effective sunrise: {{ effective.strftime('%H:%M') }}
Heures à couvrir: {{ hours | round(1) }}h
Capacité: {{ capacity }}Wh
SOC réserve: {{ [[soc, 10] | max, 45] | min }}%
```
