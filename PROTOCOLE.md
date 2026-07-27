# Protocole de capture

Page compagnon de `index.html`. Elle **pilote un protocole et capture** — elle ne rend
aucun verdict, ne calcule aucune statistique, n'affiche aucun diagnostic. Toute
l'interprétation se fait en aval, sur le JSON qu'elle produit.

**En ligne** : https://nmulongo-sys.github.io/analyse-attaque/protocole.html
**Statut** : v1 (2026-07-27) • fichier HTML unique, aucune dépendance externe, hors ligne.

## Déroulement

13 prises enchaînées. Pour chacune, la page affiche la consigne, règle elle-même tempo,
subdivision et clic, joue deux mesures de décompte ignorées, enregistre la durée utile,
**s'arrête seule**, puis attend un appui pour la suivante.

Durée utile : `max(30 s, 48 attaques attendues)`. Total ≈ **8 min 40 s de jeu utile**,
soit environ 15 minutes avec décomptes et pauses.

Le bouton *Refaire cette prise* demande un motif ; la prise rejetée est **conservée** dans
le JSON avec `rejetee: true` et son motif. Rien n'est jamais supprimé.

La session est sauvegardée dans `localStorage` après chaque prise : un rechargement
accidentel ne perd pas les quinze minutes déjà jouées.

## Contraintes appliquées

- **Casque obligatoire** : le bouton de démarrage reste verrouillé tant que la
  confirmation n'est pas cochée. L'horodatage de la confirmation figure dans l'export.
- **Chaîne de détection figée** sur tout le protocole, aucun réglage exposé :
  fenêtre 1024, saut 256, sensibilité 2,50, porte −58 dBFS, échauffement 0,45 s,
  blanchiment adaptatif (constante 0,6 s), décalage de latence 0.
- **`ecart_min_ms = 30`** sur toutes les prises — volontairement bas. Le regroupement en
  gestes se fera en post-traitement, à plusieurs valeurs, sur le même flux : c'est le seul
  moyen de savoir si les doublons observés à 59–75 ms sont réels ou fabriqués par le
  paramètre.
- **Aucune fusion destructive** : le JSON contient le flux brut intégral. Le champ `chef`
  est un marquage informatif calculé à `chef_fenetre_ms` (120 ms), rien n'est retiré.
- `echoCancellation`, `noiseSuppression`, `autoGainControl` demandés à `false`, et
  **l'état effectif** renvoyé par `track.getSettings()` est journalisé — pas la valeur
  demandée. Si le système les réimpose, la page l'affiche et l'export en garde la trace.

## Le protocole

### Série A — validation du détecteur

Une attaque par pulsation, 60 bpm sur les temps (IOI 1000 ms), 48 attaques par prise.

| Prise | Geste | Ce que ça tranche |
|---|---|---|
| A1 | Acier, médiator, mi grave à vide | vérité terrain |
| A2 | Acier, médiator, mi aigu à vide | vérité terrain, registre haut |
| A3 | Nylon, index, corde à vide | vérité terrain, attaque douce |
| A4 | Acier, médiator, accord plaqué 6 cordes | étalement du balayage |
| A5 | Acier, médiator, palm mute faible | porte de niveau |
| A6 | Acier, hammer-on / pull-off alternés | attaques non pincées — réponse binaire |

### Série B — dispersion en fonction de l'intervalle

Geste identique partout : acier, médiator, mi grave à vide, **une attaque par pulsation de
grille**. Seul l'intervalle change. À jouer d'affilée, même position, même médiator.

| Prise | Réglage | IOI | Attaques |
|---|---|---|---|
| B1 | 60 bpm, noires | 1000 ms | 48 |
| B2 | 90 bpm, noires | 667 ms | 48 |
| B3 | 120 bpm, noires | 500 ms | 60 |
| B4 | 90 bpm, croches | 333 ms | 90 |
| B5 | 120 bpm, croches | 250 ms | 120 |

Série décisive : elle dit si la dispersion de production est **constante en millisecondes**
ou **proportionnelle à l'intervalle**. De cette réponse dépend le fait que la fenêtre de
tolérance de l'outil principal soit absolue ou relative.

### Série C — bénéfice de subdivision

Même instrument, même geste, **croches à 80 bpm dans les deux cas**. Seul le clic change.

| Prise | Clic | Clics émis |
|---|---|---|
| C1 | sur les temps seulement | 1 tous les 750 ms |
| C2 | sur les croches | 1 tous les 375 ms |

## Schéma du JSON

Un fichier unique pour tout le protocole.

```json
{
  "protocole": "analyse-attaque/capture",
  "version": 1,
  "date": "…ISO…",
  "chaine_figee": { "N": 1024, "HOP": 256, "sensibilite": 2.5, "ecart_min_ms": 30,
                    "porte_dbfs": -58, "echauffement_s": 0.45, "decalage_ms": 0,
                    "chef_fenetre_ms": 120 },
  "appareil": {
    "userAgent": "…", "sampleRate": 48000,
    "baseLatency_s": 0.0107, "outputLatency_s": 0.0213,
    "micro_getSettings": { "echoCancellation": false, "noiseSuppression": false,
                           "autoGainControl": false, "…": "état EFFECTIF" },
    "micro_label": "…", "casque_confirme": true, "casque_confirme_le": "…ISO…"
  },
  "prises": [ {
    "id": "B4", "serie": "B", "consigne": "…texte affiché, verbatim…",
    "rejetee": false, "motif_rejet": null,
    "instrument": { "cordes": "acier", "attaque": "mediator",
                    "corde": "mi grave (6)", "geste": "…" },
    "reglages": { "tempo": 90, "subdivision": 2, "temps_par_mesure": 4,
                  "clic": "toutes subdivisions", "decalage_ms": 0,
                  "sensibilite": 2.5, "ecart_min_ms": 30, "porte_dbfs": -58,
                  "echauffement_s": 0.45, "fenetre_fft": 1024, "saut_fft": 256,
                  "echantillonnage_hz": 48000, "decompte_mesures": 2 },
    "ancre_s": 12.34567,
    "pas_s": 0.333333,
    "debut_utile_s": 12.34567,
    "fin_utile_s": 42.34567,
    "debut_capture_s": 7.01234,
    "attaques_attendues": 90,
    "chef_fenetre_ms": 120,
    "detections": [ { "temps_s": 12.3512, "intensite_db": -8.71, "chef": true } ],
    "clics":      [ { "temps_s": 12.34567, "sur_temps": true } ]
  } ]
}
```

### Notes pour la session d'analyse

- Tous les instants sont dans la **même base de temps** (`AudioContext.currentTime` d'un
  contexte unique jamais fermé) : ils sont comparables d'une prise à l'autre.
- `detections[]` court de `debut_capture_s` à `fin_utile_s`, décompte compris. Trancher sur
  `debut_utile_s` pour ne garder que la fenêtre utile — la page ne le fait pas.
- L'écart à la grille se recalcule depuis `ancre_s` et `pas_s` seuls. Aucun écart n'est
  pré-calculé dans le fichier.
- `clics[]` permet de recontrôler l'ancre indépendamment.
- `baseLatency_s` et `outputLatency_s` donnent une première borne matérielle sans boucle
  acoustique. Elles ne remplacent pas une calibration : le biais absolu reste inexploitable,
  seule la dispersion l'est.
- `attaques_attendues` est le nombre de pulsations de grille dans la fenêtre utile. L'écart
  entre ce nombre et le nombre de gestes reconstitués mesure les manques et les doublons.

## Journal

### 2026-07-27 — v1
- Première version, conforme au cahier des charges reçu.
- Tests : `vproto` (table du protocole, contraintes dures, absence de toute fonction
  d'analyse dans le source, forme du JSON, conservation des prises rejetées) et `vmetro`
  (simulation du métronome horloge pilotée : ancre à deux mesures exactement, écart entre
  clics conforme au réglage, dérive à la grille inférieure à 0,01 ms sur les cinq
  configurations testées).
