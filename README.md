# Analyse d'attaque — guitare

Outil de mesure du placement rythmique et de la dynamique du jeu de guitare, à partir du
micro du navigateur. Destiné à l'usage pédagogique : montrer à l'élève où tombent
réellement ses attaques par rapport à la grille, et comment il répartit ses accents.

**En ligne** : https://nmulongo-sys.github.io/analyse-attaque/
**Statut** : v1 (2026-07-27) • fichier HTML unique, aucune dépendance externe, hors ligne, mobile-first.

## Utilisation

Ouvrir `index.html` dans un navigateur, ou passer par GitHub Pages sur mobile.
Le micro exige un contexte sécurisé (HTTPS ou `localhost`).

1. **Activer le micro** — le navigateur demande l'autorisation.
2. Régler tempo, subdivision et mesure, puis lancer le **métronome**.
3. Jouer une dizaine de temps bien en place, puis **Caler sur le jeu** : le décalage de
   latence est ajusté pour ramener l'écart médian à zéro.
4. Lire les quatre graphiques, et exporter les mesures en JSON si besoin.

**Jouer au casque** quand le métronome est actif : le clic capté par le micro est détecté
comme une attaque.

Aucune donnée n'est envoyée ni stockée. L'app n'utilise pas `localStorage` : tout est en
mémoire et disparaît au rechargement. L'export JSON est le seul moyen de conserver une
session.

## Ce que l'outil mesure — et ce qu'il ne mesure pas

**Mesuré** : l'instant d'attaque de chaque événement sonore et son intensité crête.

**Non mesuré** : la hauteur des notes, l'accord joué, la propreté des cordes. Un accord
plaqué compte pour une attaque, pas six. Deux cordes égrenées à moins de l'écart minimum
fusionnent en une seule attaque.

## Architecture & conventions

### Structure du fichier

| Bloc | Rôle |
|---|---|
| `<style>` | Palette maison, mise en page mobile-first |
| Corps HTML | Six sections : prise de son, grille, placement, attaques, dynamique, profil, bilan |
| `<script id="src-worklet" type="text/plain">` | Code source de l'`AudioWorkletProcessor`, non exécuté par la page |
| `<script>` | Page : audio, métronome, grille, rendus canvas, export |

Le worklet est chargé en construisant un `Blob` à partir du `textContent` du
`<script type="text/plain">`, puis `audioWorklet.addModule(URL.createObjectURL(blob))`.
C'est ce qui permet de garder **un seul fichier** tout en utilisant un worklet, qui exige
normalement un module séparé. L'URL objet est révoquée juste après le chargement.

### Chaîne de détection (worklet `detecteur-attaque`)

Analyse par trames : `N = 1024` échantillons, saut `HOP = 256`, soit une trame toutes les
**5,33 ms** à 48 kHz. Tampon circulaire alimenté par quanta de 128 échantillons.

1. **Fenêtrage** de Hann, puis FFT radix-2 itérative avec tables de rotation
   précalculées (implémentation locale, pas de bibliothèque).
2. **Blanchiment adaptatif** — chaque bande de fréquence est divisée par sa propre crête
   glissante : `pic[k] = max(mg, pic[k]·r, 2e-4)` avec `r = exp(-hop/0,6 s)`.
   Sans cette étape, les partiels graves qui battent entre eux pendant la résonance
   produisent des montées d'énergie indiscernables d'une vraie attaque.
3. **Flux spectral** — somme des différences positives du spectre blanchi entre deux
   trames, moyennée sur les bandes utiles (jusqu'à 6 kHz).
4. **Seuil adaptatif** — médiane du flux sur les 45 dernières trames (≈ 240 ms), multipliée
   par la sensibilité, plus un plancher de `1e-4`.
5. **Sélection de pic** — maximum local sur trois trames, au-dessus du seuil, avec un
   écart minimum imposé depuis l'attaque précédente. Le test porte sur la trame `f1`,
   une trame en arrière, d'où un retard structurel d'une trame dans le traitement mais
   pas dans l'horodatage.
6. **Horodatage** — `currentFrame` du worklet, jamais `requestAnimationFrame`. L'instant
   retenu est `t1 - hop` : le pic de flux tombe une trame après la première montée
   d'énergie. Biais résiduel mesuré < 3 ms.
7. **Intensité** — après détection, la crête absolue est suivie pendant 6 trames (≈ 32 ms)
   avant émission, pour capter le sommet de l'attaque et non son amorce.

Garde-fous : porte de niveau à **−58 dBFS** (rien n'est détecté sous ce seuil) et
**échauffement de 0,45 s** au démarrage, le temps que les crêtes par bande convergent.

Sensibilité et écart minimum sont modifiables à chaud via `port.postMessage`.

### Prise de son

`getUserMedia` est appelé avec `echoCancellation`, `noiseSuppression` et
`autoGainControl` à `false`. Ces filtres écrasent les nuances de dynamique : sans cette
désactivation, le graphique d'intensité ne mesure plus rien. Certains Android les
réimposent malgré tout — les temps restent justes, les vélocités sont compressées.

### Grille et écarts

La grille est ancrée sur `E.ancre` (instant `AudioContext.currentTime` du temps 1), avec
un pas de `60 / tempo / subdivision`. Pour chaque attaque :

```
k   = round((t + décalage - ancre) / pas)     case de grille la plus proche
dev = t + décalage - (ancre + k·pas)          écart signé, négatif = en avance
pos = k mod (subdivision × temps par mesure)  position dans la mesure
```

Changer le tempo, la subdivision, la mesure ou le décalage déclenche `recalculer()`, qui
réévalue `k`, `dev` et `pos` de toutes les attaques mémorisées — les instants bruts `t`
ne sont jamais altérés.

Fenêtre « dans la cible » : **±25 ms** (constante `FEN`).

Le métronome utilise un ordonnanceur à anticipation (`setInterval` de 25 ms, horizon
120 ms) qui programme des oscillateurs carrés : 1600 Hz sur le premier temps, 1100 Hz sur
les autres temps, 820 Hz sur les subdivisions.

### Lecture des quatre graphiques

- **Placement** — l'élément central. Axe horizontal = écart à la grille en ms, axe
  vertical = ordre chronologique (la plus récente en bas). Diamètre du point = intensité,
  opacité = fraîcheur, cercle = premier temps de la mesure. Trait or pointillé = médiane
  des 16 dernières attaques. Bande verte = fenêtre ±25 ms.
- **Attaques détectées** — flux spectral (courbe pleine), seuil adaptatif (pointillé or),
  attaques retenues (traits verticaux colorés par écart). Défilement sur ~4 s
  (`ODF_MAX = 760` trames), la trame la plus récente au bord droit.
- **Dynamique** — barre par attaque, hauteur = intensité crête en dB relatifs sur 36 dB,
  couleur selon la position dans la mesure.
- **Profil d'accentuation** — moyenne d'intensité par position de la mesure, en dB
  relatifs au maximum. C'est le graphique qui révèle l'accentuation involontaire.

### Bilan

- **Écart moyen** : biais systématique, se corrige au décalage de latence.
- **Régularité (σ)** : écart-type des écarts — la vraie mesure de stabilité, elle ne se
  calibre pas.
- **Dans ±25 ms**, **écart dynamique** (amplitude max − min en dB).

### Export JSON

```json
{
  "outil": "analyse-attaque", "version": 1, "date": "…ISO…",
  "reglages": { "tempo": 80, "subdivision": 2, "temps_par_mesure": 4,
                "decalage_ms": 0, "sensibilite": 2.5, "ecart_min_ms": 55,
                "echantillonnage_hz": 48000 },
  "attaques": [ { "temps_s": 0.0, "case_grille": 0, "position_mesure": 0,
                  "ecart_ms": 0.0, "intensite_db": 0.0 } ]
}
```

`temps_s` est l'instant brut, hors décalage de latence ; `ecart_ms` l'intègre.

### Palette

Crème `#f4ece0`, papier `#fbf6ee`, encre `#382d24`, trait `#d9c9b1`, sépia `#8b6f4e`,
or `#b8860b`. Codes d'état : avance `#4a6b8a`, cible `#6b8f5a`, retard `#a8543a`.
Titres en Cormorant Garamond, texte en Work Sans, avec repli système — aucune police
n'est chargée depuis le réseau.

### Performances mesurées au banc

Signal synthétique de corde pincée (inharmonicité, amortissement dépendant du rang des
partiels, transitoire de plectre), 5 tirages × 16 attaques par condition, worklet réel
instrumenté sous Node :

| Bruit de fond | Détectées | Fausses | Biais | σ | Écart max |
|---|---|---|---|---|---|
| −60 dB | 80/80 | 6 | 0,5 ms | 6,9 ms | 30 ms |
| −52 dB | 80/80 | 0 | 2,2 ms | 2,4 ms | 6,0 ms |
| −44 dB | 79/80 | 0 | 2,5 ms | 2,3 ms | 6,7 ms |
| −38 dB | 79/80 | 0 | 3,0 ms | 1,9 ms | 6,7 ms |

À confirmer sur guitare et micro réels : le banc est synthétique.

## Journal de développement

### 2026-07-27 — révision initiale (v1)
- Première version : détection d'attaques et mesure de placement/dynamique, en fichier
  unique sans dépendance.
- Chaîne de traitement en `AudioWorklet` (trame de 5,33 ms) plutôt qu'en `AnalyserNode` :
  ce dernier applique un lissage temporel qui écrase précisément les transitoires à
  détecter, et un sondage en `requestAnimationFrame` plafonnerait la résolution à ~17 ms.
- **Décision principale — blanchiment adaptatif.** La première implémentation utilisait
  un flux spectral sur log-magnitude. Banc d'essai : 16 attaques sur 16 détectées, mais
  27 fausses, dues au battement entre partiels graves pendant la résonance. Le passage à
  la normalisation par bande par crête glissante ramène les fausses détections à 0 dans
  les conditions réalistes.
- **Écarté — regroupement en bandes logarithmiques** (16, 24, 32, 48 bandes). Meilleur sur
  un signal de test à sinusoïdes pures, nettement moins bon dès que le signal comporte
  inharmonicité, amortissement par rang et transitoire de plectre. Conservé ici en note
  pour éviter de refaire l'essai.
- Correction du biais d'horodatage : l'instant retenu passe de `t1 - hop/2` à `t1 - hop`,
  biais résiduel ramené de ~5 ms à < 3 ms.
- Correction de l'alignement de la courbe de flux : elle est ancrée au bord droit, afin
  de partager l'axe temporel des marqueurs d'attaque tant que le tampon n'est pas plein.
- `echoCancellation`, `noiseSuppression` et `autoGainControl` désactivés à l'ouverture du
  micro.
- Validation : contrôle de syntaxe, exécution headless du worklet sous Node sur signaux
  synthétiques, et parcours jsdom de la page (rendus, bilan, réglages, calage, export,
  état vide).

## Suite envisagée

**Étape 2 — vérification d'accord.** Chromagramme sur fenêtre longue (8192 points,
≈ 2,7 Hz/bin, nécessaire pour séparer un demi-ton dans le grave), déclenché par l'attaque
détectée à l'étape 1. Dans un contexte pédagogique l'accord attendu est connu : le
problème n'est pas la *détection* en classe ouverte mais la *vérification* contre un
gabarit, plus fiable, et capable d'indiquer quelle corde manque. Verdict ternaire :
conforme / confusion identifiée / indéterminé — jamais de verdict autoritaire faux.

## Licence

Aucun fichier `LICENSE` dans le dépôt : tous droits réservés par défaut.
