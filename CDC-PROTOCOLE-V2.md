# `protocole.html` v2 — cahier des charges

> 2026-07-27. Document d'exécution, destiné à une session de codage distincte.
> Dépôt : `nmulongo-sys/analyse-attaque`, branche `main`, `protocole.html` à la racine.
> La v1 existe et fonctionne : **ceci est une révision, pas une réécriture.**

## 1. Pourquoi une v2

Le premier protocole complet (`protocole-2026-07-27-03-17-23.json`, 14 prises) a livré ce
qu'on lui demandait, et a révélé deux manques qui coûtent des séances entières :

1. **Aucune calibration de latence n'existe, sur aucun appareil.** `baseLatency` et
   `outputLatency` sont inutilisables — l'appareil de référence déclare 4 ms de latence de
   sortie *au casque*. Sans calibration, le biais mesuré va de +130 à +440 ms selon les
   prises et n'est interprétable dans aucune.
2. **Deux prises sur treize ont été perdues pour décrochage de tempo.** B4 (+5,6 %) et B5
   (+6,7 %) : cinq à sept tours de phase complets sur 30 s, l'accroche s'effondre
   mécaniquement. Elles ont coûté une analyse pour rien.

La v2 ajoute donc une calibration en tête et un garde-fou de tempo en fin de prise. Rien
d'autre ne change dans la chaîne de détection.

## 2. Ce qui ne bouge pas — à ne pas « améliorer »

| Élément | Valeur | Motif |
|---|---|---|
| `ecart_min_ms` | **30**, sur toutes les prises | Le regroupement se fait en post-traitement, à plusieurs valeurs sur le même flux. C'est le seul moyen de savoir si un paquet de doublons est réel ou fabriqué par le paramètre. **Ne pas passer à 120** : cette valeur-là est celle de `comping`, pas celle de l'instrument. |
| Chaîne de détection | figée, identique à la v1 | Une chaîne qui change entre deux séries rend les séries incomparables. |
| Fusion des détections | aucune avant export | `chef` reste un marquage informatif. |
| État du micro | `track.getSettings()` **effectif** | Pas la valeur demandée. La v1 a bien fait, garder. |
| Casque | verrouillant | Sans confirmation, pas de démarrage. Seule exception : l'écran de calibration du §3. |
| Sauvegarde | `localStorage` après chaque prise | Quinze minutes de jeu ne se reperdent pas sur un rechargement. |
| Aucune analyse à l'écran | — | `protocole.html` est l'instrument de mesure. `index.html` est l'outil d'entraînement. Le seul chiffre nouveau affiché est celui du garde-fou (§4). |

## 3. Calibration par boucle acoustique — nouvel écran, en tête

Bloquant : tant qu'elle n'est pas faite ou explicitement passée, aucune prise ne démarre.

### 3.1 Déroulement

1. Écran d'instruction : **retirer le casque**, poser le téléphone micro vers le haut,
   haut-parleur dégagé, pièce calme. C'est la seule opération de tout le produit où le
   haut-parleur est autorisé, et elle l'exige : la boucle doit passer par l'air.
2. Émettre **24 clics à 1000 ms d'intervalle**, avec la synthèse de clic exacte des prises.
3. Le worklet les détecte comme des attaques. Pour chaque clic *k* :
   `δ_k = t_détecté − t_programmé`.
4. Écarter les deux premiers clics (montée de la chaîne audio), puis :
   `latence_ms` = **médiane** des δ restants ; `dispersion_ms` = **écart interquartile**.
5. **Rejet** si moins de 20 δ retenus, ou si l'écart interquartile dépasse **15 ms**.
   Message explicite (« la pièce est trop bruyante » / « le haut-parleur n'est pas capté »),
   proposition de refaire, aucune valeur enregistrée.
6. Succès : afficher la valeur, demander de **remettre le casque**, et verrouiller à nouveau
   la confirmation casque avant la première prise.

### 3.2 Stockage

`localStorage`, indexé par appareil, **plus recopie intégrale dans le JSON exporté** :

```js
calibration: {
  empreinte:      "<hachage court>",
  latence_ms:     187.4,
  dispersion_ms:  4.1,
  n:              22,
  deltas_ms:      [ ... ],        // bruts, tous conservés
  date:           "2026-07-27T21:03:11Z",
  sr:             48000,
  passee:         false            // true si l'opérateur a explicitement sauté l'étape
}
```

Empreinte : hachage court de `navigator.userAgent` + `sampleRate` +
`AudioContext.baseLatency` arrondi. Grossière à dessein : elle sert à ne pas réutiliser la
latence d'un autre téléphone, pas à identifier une machine.

**Les δ bruts sont exportés.** La médiane peut être recalculée autrement plus tard ; une
médiane seule ne se réouvre pas.

### 3.3 Bouton « passer »

Autorisé, mais il écrit `passee: true` et affiche sur chaque prise suivante un bandeau
« prise non calibrée ». Une série non calibrée reste exploitable pour σ, ρ et R — pas pour
le biais.

## 4. Garde-fou de tempo — fin de prise

### 4.1 Critère

À la fin de chaque prise, calculer le **tempo réellement joué** et comparer au tempo réglé.
Rejet au-delà de **3 %** d'écart.

Estimation retenue : **intervalle médian entre gestes successifs**, converti en bpm via la
subdivision déclarée.

> **Pourquoi la médiane est légitime ici et ne l'est pas dans `index.html`.** La v1.3
> d'`analyse-attaque` a dû abandonner l'estimation ancrée sur l'intervalle médian, parce
> qu'en jeu libre les valeurs rythmiques se mêlent, la médiane glisse vers les valeurs
> courtes et la vraie pulsation sort de la plage explorée (session à 40 bpm, réponse
> 58,7 bpm). Le protocole, lui, **impose un geste par clic** : la distribution des
> intervalles est unimodale par construction. Le piège ne s'applique pas. Ne pas importer
> ici l'estimation par étages, inutilement coûteuse.

### 4.2 Comportement

- Écart ≤ 3 % → prise acceptée, enregistrée.
- Écart > 3 % → **prise refusée**, motif affiché avec le chiffre (« 85,2 bpm joués pour 80
  réglés, +6,5 % »), proposition immédiate de refaire. La prise refusée est **conservée dans
  le JSON** avec `retenue: false` et son motif : une prise ratée documentée vaut mieux
  qu'un trou.
- **Avertissement à mi-parcours** : à 15 s, si l'écart dépasse déjà 3 %, afficher un signal
  discret permettant d'interrompre. Économise 15 s à chaque décrochage, et évite de finir
  une prise qu'on sait perdue.

### 4.3 Ce qui n'est pas un critère de rejet

La **dérive** cumulée sur la prise est affichée et exportée, mais **ne bloque rien**. B2
présentait +520 ms de dérive pour une accroche R = 0,83 : c'est une bonne prise. Aucun seuil
de dérive n'est défendable en l'état — ne pas en inventer un.

## 5. Liste des prises — seconde série

Onze prises, ~25 min avec les réglages. Ordre imposé, alternance comprise.

| # | Code | Pas | Tempo | Subdivision | Consigne |
|---|---|---|---|---|---|
| 0 | CAL | — | — | — | Calibration §3, sans casque |
| 1 | B1 | 1000 ms | 60 | noires | un geste par clic |
| 2 | B2 | 667 ms | 90 | noires | idem |
| 3 | B3 | 500 ms | 120 | noires | idem |
| 4 | B4 | 333 ms | 90 | croches | idem — **refaite** |
| 5 | B5 | 250 ms | 120 | croches | idem — **refaite** |
| 6 | C1 | 375 ms | 80 | croches | clic **sur les temps** |
| 7 | C2 | 375 ms | 80 | croches | clic **sur les croches** |
| 8 | C1 | 375 ms | 80 | croches | répétition 2 |
| 9 | C2 | 375 ms | 80 | croches | répétition 2 |
| 10 | C1 | 375 ms | 80 | croches | répétition 3 |
| 11 | C2 | 375 ms | 80 | croches | répétition 3 |

Conditions tenues fixes sur toute la série, sans exception : **même guitare, même corde,
même médiator, même position, même main droite.** Les séries A (timbres et gestes) ne sont
pas refaites : leurs conclusions sont acquises.

Durée de prise : **30 s**, inchangée.

L'alternance stricte C1/C2/C1/C2/C1/C2 est le point du protocole à ne pas assouplir : elle
répartit la fatigue et l'échauffement également entre les deux conditions. Six prises
groupées par condition ne diraient rien.

## 6. Ce que la série doit trancher

À rappeler dans l'écran d'accueil, pour que l'opérateur sache ce qu'il produit :

1. **La loi des 6 %** — σ_locale / intervalle, mesurée sur cinq pas de 250 à 1000 ms. La
   première série donne 5,5 à 7,1 %, sur sept mesures, un appareil, une séance. Cette
   seconde série la confirme ou la casse. C'est elle qui décide de `COEF_FEN` dans `comping`.
2. **Le bénéfice de subdivision** — C1 contre C2, trois répétitions. La première série donne
   +11 points d'accroche pour le clic subdivisé, et **aucun gain** de régularité locale. Une
   prise par condition ne suffit pas à conclure.
3. **La latence réelle de l'appareil**, pour la première fois.

## 7. Schéma d'export — v2

Reprendre le schéma de `PROTOCOLE.md` v1, avec trois ajouts et rien de retiré :

```js
{
  version: 2,
  calibration: { ... },            // §3.2, y compris quand passee: true
  appareil:    { ... },            // inchangé, getSettings() effectif
  prises: [{
    code:        "B4",
    retenue:     true,             // NOUVEAU — false si refusée §4.2
    motif_rejet: null,             // NOUVEAU — "tempo +6,5 %" | "interruption extérieure"
    tempo_regle: 90,
    tempo_joue:  95.8,             // NOUVEAU — médiane des intervalles
    ecart_pct:   6.5,              // NOUVEAU
    derive_ms:   -1591,            // affichée, non bloquante
    detections:  [ ... ],          // brut intégral, inchangé
    ...
  }]
}
```

Le champ `version: 2` est obligatoire : le dépouillement doit pouvoir refuser un fichier v1
présenté comme une seconde série.

## 8. Tests avant push

1. `node --check` sur le script extrait.
2. Intégrité des références DOM (tout `getElementById` a sa cible).
3. Calibration en headless : injecter 24 δ synthétiques de médiane connue + deux valeurs
   aberrantes → vérifier que la médiane tient et que l'écart interquartile est correct.
4. Calibration avec 10 δ seulement → rejet, aucune valeur écrite.
5. Garde-fou : série synthétique à +2,8 % → acceptée ; à +3,2 % → refusée, prise conservée
   avec son motif.
6. Export v2 relu : `version`, `calibration`, `retenue`, `motif_rejet`, `tempo_joue`,
   `ecart_pct` tous présents sur toutes les prises, y compris refusées.
7. Rechargement en cours de série → les prises déjà faites sont retrouvées.

## 9. Documentation

`PROTOCOLE.md` mis à jour : déroulement de la calibration, critère de rejet, schéma v2.
`README.md` d'`analyse-attaque` : **une seule entrée de journal datée ajoutée en tête**, les
sept entrées antérieures intactes. Journal append-only, sans exception.

## 10. Hors périmètre — ne pas traiter dans cette v2

- La vérification d'accord par chromagramme (point ouvert 6).
- Le ratio de swing.
- Les timbres idiomatiques.
- Toute analyse affichée à l'écran : `protocole.html` capture, il n'interprète pas.
