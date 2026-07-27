# Analyse d'attaque — guitare

Outil de mesure du placement rythmique et de la dynamique du jeu de guitare, à partir du
micro du navigateur. Destiné à l'usage pédagogique : montrer à l'élève où tombent
réellement ses attaques par rapport à la grille, et comment il répartit ses accents.

**En ligne** : https://nmulongo-sys.github.io/analyse-attaque/
**Statut** : v1.4 (2026-07-27) • fichier HTML unique, aucune dépendance externe, hors ligne, mobile-first.

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

**Mesuré** : l'instant de chaque geste et son intensité crête, l'accroche du jeu à la
grille, la subdivision réellement jouée, et l'évolution de tout cela au fil de la séance.

**Non mesuré** : la hauteur des notes, l'accord joué, la propreté des cordes.

**Deux couches distinctes** : les *détections* (ce que le worklet trouve) et les *gestes*
(ce que le musicien a fait). Un accord plaqué produit plusieurs détections — le balayage
des cordes, et surtout le redéclenchement sur la résonance du corps dès la fin du temps
réfractaire. Sur une session réelle de 132 s, 293 détections correspondaient à 163 gestes.
Toutes les statistiques portent sur les gestes ; les détections restent visibles sur le
bandeau de flux, les fusionnées en trait bas pâle.

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

### Dimensionnement des canevas — piège à connaître

`prepare(cv)` sépare deux tailles qu'il ne faut jamais confondre :

- la **hauteur CSS voulue**, figée une fois pour toutes dans `cv.dataset.h` (dupliquée dans
  l'attribut `data-h` du balisage) ;
- la **taille du tampon de rendu**, `cv.width` / `cv.height`, égale à la taille CSS
  multipliée par le rapport de pixels, plafonné à 2.

Écrire `cv.height` **se reflète dans l'attribut `height`**. Relire la hauteur voulue dans
cet attribut à l'image suivante la remultiplierait donc par le rapport de pixels, à chaque
image, indéfiniment. C'est le défaut corrigé en v1.1 ; le test `vregress` couvre ce cas.
`cv.style.height` est fixé explicitement, sans quoi l'élément prend pour hauteur de mise
en page la taille de son tampon.

### Ordonnancement du rendu

Seul le bandeau de flux est réellement animé, plafonné à ~30 images/s. Les trois autres
graphiques et le bilan ne sont redessinés que lorsque `E.maj` est vrai — positionné à
l'arrivée d'une attaque, après `recalculer()`, à l'effacement des mesures et au
redimensionnement. Redessiner l'ensemble à 60 Hz ne servait qu'à chauffer le téléphone.

### Prise de son

`getUserMedia` est appelé avec `echoCancellation`, `noiseSuppression` et
`autoGainControl` à `false`. Ces filtres écrasent les nuances de dynamique : sans cette
désactivation, le graphique d'intensité ne mesure plus rien. Certains Android les
réimposent malgré tout — les temps restent justes, les vélocités sont compressées.

### Référentiel temporel — le point le plus délicat

Un écart à la grille n'a de sens que rapporté à la référence contre laquelle l'attaque a
été jouée. Chaque attaque mémorise donc **sa propre ancre et son propre pas** :

```js
o = { t, ancre, pas, db, force }      // référence figée à l'instant de la détection
placer(o)                             // (re)calcule k, dev, pos à partir de CETTE référence
```

Seul le décalage de latence peut être rejoué après coup : c'est une translation pure, elle
préserve la structure. Tout le reste — nouveau contexte audio, nouvelle ancre de
métronome, changement de tempo ou de subdivision — définit un **nouveau référentiel**, et
les mesures antérieures deviennent incomparables. Dans ce cas `nouvelleSerie(motif)` les
efface en l'annonçant, plutôt que de les mélanger silencieusement.

C'est le défaut corrigé en v1.2 : les anciennes attaques étaient recalculées contre une
ancre qui n'existait pas au moment où elles avaient été jouées. Le test `vserie` couvre
les quatre points d'entrée.

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

### Fenêtre glissante — pourquoi il n'y a pas de chiffre global

Une séance d'entraînement n'est pas homogène. Sur la session de référence, l'accroche
passe de 34 % à 68 %, retombe à 18 %, puis remonte à 84 % — une moyenne sur les deux
minutes ne décrit aucun de ces moments. Les chiffres du bilan portent donc sur les
`PARAM.fenetre` derniers gestes (24 par défaut), et la courbe de progression montre le
reste. La valeur de session ne sert qu'à situer la fenêtre : « tu progresses, 25 points
au-dessus de ta moyenne ».

### Discriminateur de subdivision

R mesuré à la période `pas/m` est exactement la **m-ième harmonique** de la distribution
de phase. Une exécution en croches sur une grille de noires donne R₁ ≈ 0 et R₂ élevé ;
des triolets font ressortir R₃. La subdivision réellement jouée est donc l'argmax de
R(pas/m) pour m ∈ {1,2,3,4} — un jeu propre sur la grille réglée donne m = 1, puisque
des noires exactes tombent aussi sur toutes les subdivisions.

C'est ce qui distingue **« joue autre chose que ce qui est réglé »** de **« joue mal »**.
Sur la session de référence, le passage 29–82 s donnait R₁ = 0,27 et R₂ = 0,63 : le
placement était bon, la grille était fausse. Sans ce test, l'outil concluait à une chute
de niveau là où il y avait un simple changement de valeur rythmique.

### Statistique — pourquoi elle est circulaire

Les écarts à la grille sont des **phases**, pas des longueurs : ils sont bornés à ±pas/2
et se referment sur eux-mêmes. La moyenne et l'écart-type linéaires y sont trompeurs. Un
jeu sans aucun rapport avec la grille produit mécaniquement un écart-type de **pas/√12**
— à 80 bpm en croches, 108 ms — qu'on peut prendre pour une mesure de régularité alors que
c'est exactement le contraire : la signature du bruit uniforme.

`statistiques()` calcule donc, sur les phases `2π·dev/pas` :

- **R**, longueur du vecteur résultant — l'« accroche à la grille », de 0 à 1 ;
- le **test de Rayleigh** (`z = nR²`, p-valeur associée) ;
- le **biais** comme angle moyen, et la **régularité** comme écart-type circulaire
  `pas/2π · √(−2 ln R)`.

Tant que `p ≥ 0,001` ou `R ≤ 0,25`, l'app **n'affiche ni biais ni régularité** et explique
pourquoi. Le profil d'accentuation est neutralisé de la même façon : sans accroche, chaque
attaque est rangée dans une case arbitraire de la mesure.

`tempoJoue()` estime en parallèle le pas réellement joué, par recherche du maximum
d'accroche autour de l'intervalle médian entre attaques. Indépendant de la grille, il
répond à « à quel tempo joue-t-il », pas à « respecte-t-il le mien » — et permet de
diagnostiquer un simple écart de tempo plutôt que de conclure à un jeu irrégulier.

### Export JSON (version 3)

```json
{
  "outil": "analyse-attaque", "version": 3, "date": "…ISO…",
  "reglages": { "tempo": 60, "subdivision": 1, "temps_par_mesure": 4,
                "decalage_ms": 0, "sensibilite": 2.5, "ecart_min_ms": 55,
                "fusion_ms": 120, "fenetre_gestes": 24, "echantillonnage_hz": 48000 },
  "bilan": {
    "gestes": 163, "detections": 293,
    "fenetre": { "accroche": 0.73, "p": 1.2e-8, "biais_ms": -4.2, "sigma_ms": 127.2,
                 "subdivision_ajustee": 1, "harmoniques": [0.73, 0.35, 0.18, 0.07] },
    "session": { "accroche": 0.45, "sigma_ms": 200.1, "subdivision_ajustee": 1 },
    "tempo_joue_bpm": 60.1, "ecart_dynamique_db": 47.0
  },
  "gestes":     [ { "temps_s": 0.0, "detections": 2, "etalement_ms": 64.0,
                    "case_grille": 0, "position_mesure": 0,
                    "ecart_ms": 0.0, "intensite_db": -8.7 } ],
  "detections": [ { "temps_s": 0.0, "chef": true, "intensite_db": -8.7 } ]
}
```

`temps_s` est l'instant brut, hors décalage de latence ; `ecart_ms` l'intègre.
`chef` distingue le geste de ses redéclenchements absorbés.

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

### 2026-07-27 — v1.4, estimation du tempo par étages
Deuxième export réel (grille à 40 bpm, 42 gestes pour 51 détections, 44 s). La
synchronisation est bonne — la période optimale vaut 1506 ms, soit 39,8 bpm — mais l'outil
affichait « tempo joué : 58,7 bpm ». Trois défauts corrigés, tous trouvés par les bancs :

- **Estimation du tempo ancrée sur l'intervalle médian.** Avec un jeu mêlant plusieurs
  valeurs rythmiques, la médiane (1067 ms) tombait loin de la pulsation (1500 ms), et la
  plage explorée (±22 %) excluait la bonne réponse. Remplacée par une estimation par
  étages : passe grossière sur portée courte, puis affinages locaux sur portée croissante.
  Un simple balayage large ne suffit pas — la résolution requise est `p²/(4·durée)`.
- **Discriminateur de subdivision trop bavard.** Il retenait la moins mauvaise harmonique
  même quand toutes étaient basses, et diagnostiquait donc une subdivision sur du bruit.
  Il exige désormais une accroche alternative supérieure à 0,45 et à R₁ + 0,15.
- **Clé de cache du tempo indexée sur le seul effectif** : elle ne s'invalidait pas quand
  la série changeait à nombre de gestes constant. Elle inclut maintenant l'instant du
  premier geste.
- Chaînage des gestes borné : une suite de détections rapprochées ne peut plus s'agréger
  au-delà de 2,5 fois la fenêtre de fusion.
- Avertissement de tempo supprimé quand le discriminateur de subdivision a déjà parlé —
  les deux messages disaient la même chose de deux façons contradictoires.
- Test `vtempo` ajouté : rejeu des deux sessions réelles, plus un contrôle avec grille
  volontairement fausse.

*Note de méthode.* Le premier jeu d'essais de synthèse affirmait que l'estimateur devait
répondre « 40 bpm » sur un signal dont les gestes tombaient en réalité sur une grille de
doubles : la bonne réponse était 160 bpm pour une subdivision déclarée en noires.
L'attendu du test était faux, pas le code. Vérifier l'attendu avant de corriger.

### 2026-07-27 — v1.3, gestes, fenêtre glissante et subdivision réellement jouée
Analyse d'un export réel (293 détections, 132 s, métronome à 60 bpm, subdivision réglée
sur « noires »). La période qui maximise l'accroche est 999,07 ms, soit 60,06 bpm : la
synchronisation au métronome est certaine (p ≈ 6·10⁻¹⁶). Trois enseignements, trois
changements :

- **Le chiffre global ne décrivait aucun instant de jeu.** L'accroche variait de 18 % à
  84 % au cours de la séance. Les statistiques portent désormais sur une fenêtre glissante
  de 24 gestes, et une courbe de progression a été ajoutée. La valeur de session ne sert
  plus qu'à situer la fenêtre.
- **La subdivision réglée n'était pas celle jouée.** Entre 29 et 82 s, R₁ = 0,27 mais
  R₂ = 0,63 : passage joué en croches sur une grille de noires. L'outil concluait à une
  chute de niveau là où il y avait un changement de valeur rythmique. Ajout du
  discriminateur harmonique (argmax de R à pas/m, m ∈ 1..4) : quand une autre subdivision
  explique mieux le jeu, l'outil le dit et propose le réglage, au lieu de blâmer le
  musicien.
- **Le détecteur se redéclenchait sur la résonance.** 44 % des intervalles entre
  détections tombaient sous 120 ms, groupés entre 61 et 75 ms — collés au plancher
  réfractaire de 55 ms, donc artefact et non balayage de médiator. Introduction d'une
  couche « gestes » : les détections séparées de moins de `PARAM.fusion` (120 ms par
  défaut) sont fusionnées, instant du premier déclenchement, intensité du plus fort.
  293 détections → 163 gestes. Le bandeau de flux montre les fusionnées en trait bas pâle.
- Verdict reformulé pour ne plus se contredire : l'accroche dit s'il existe une relation
  avec la grille, la dispersion en millisecondes dit si elle est musicalement bonne.
- Export en version 3 : bilan structuré, gestes et détections séparés.
- Test `vreel` ajouté : rejeu des 293 détections réelles à travers le code embarqué, avec
  vérification que le passage en croches est identifié et que la progression finale est
  reconnue.

### 2026-07-27 — v1.2, référentiel temporel et honnêteté statistique
Session de test réelle : 122 attaques, métronome actif, accords plaqués réguliers.
Résultat affiché : σ = 115,4 ms, 9 % dans ±25 ms — soit très exactement ce que produirait
un tirage uniforme (pas/√12 = 108 ms, 50/375 = 13 %). Autrement dit l'app mesurait du
bruit et le présentait comme une performance. Diagnostic et correctifs :

- **Cause première — recalcul contre une référence postérieure.** `recalculer()`
  réévaluait *toutes* les attaques mémorisées contre l'ancre courante. Redémarrer le
  métronome posait une nouvelle ancre, donc randomisait la phase de tout l'historique.
  Chaque attaque emporte désormais sa propre ancre et son propre pas.
- **Second défaut — changement de subdivision sans réancrage.** Le pas de calcul changeait
  mais `E.prochain` / `E.k` continuaient sur l'ancienne cadence : la grille calculée et les
  clics entendus divergeaient définitivement.
- **Troisième défaut — base de temps repartant de zéro.** Arrêter le micro ferme
  l'`AudioContext` ; le rouvrir en crée un autre dont `currentTime` repart près de 0,
  tandis que les attaques mémorisées gardaient les horodatages de l'ancien.
- Introduction de `nouvelleSerie(motif)` : toute redéfinition du référentiel efface les
  mesures en l'annonçant, au lieu de mélanger deux repères.
- **Statistique circulaire et test de Rayleigh.** L'app refuse désormais d'afficher biais
  et régularité tant que l'accroche à la grille n'est pas significative, et explique ce
  qu'elle observe. Ajout de l'accroche (R) et du tempo réellement joué, qui distingue un
  jeu irrégulier d'un simple écart de tempo.
- Neutralisation du profil d'accentuation en l'absence d'accroche.
- **Export et effacement pilotés par la présence de mesures**, plus par l'état du micro :
  ils étaient grisés après l'arrêt du micro alors que l'interface annonçait que les
  mesures restaient disponibles.
- Avertissement au-delà de 30 dB d'écart dynamique, indice de fausses détections faibles.
- Tests ajoutés : `vstats` (cinq scénarios de jeu à propriétés connues — serré, biaisé,
  lâche, tempo décalé, uniforme) et `vserie` (les quatre points d'entrée du référentiel,
  plus l'état des boutons).

### 2026-07-27 — v1.1, correctif d'affichage
- **Défaut bloquant corrigé : emballement du dimensionnement des canevas.** `prepare()`
  lisait la hauteur voulue dans l'attribut `height` du canevas, puis y écrivait la taille
  du tampon de rendu (`cv.height = h × devicePixelRatio`), qui se reflète dans ce même
  attribut. À l'image suivante la valeur écrite était relue et remultipliée : sur un écran
  à 2,75×, 260 → 715 → 1966 → 5406 px, à 60 images par seconde. La page enflait sans
  limite et l'affichage tremblait en continu. La hauteur voulue est désormais figée au
  premier appel dans `cv.dataset.h`, hors de portée de l'écriture du tampon.
- `cv.style.height` fixé explicitement : sans lui, l'élément prenait pour hauteur de mise
  en page la taille de son tampon, et le tracé n'occupait que le haut du cadre.
- Rapport de pixels plafonné à 2 — au-delà, le coût de tracé sur mobile n'achète plus rien
  de visible.
- Rendu conditionnel : le bandeau de flux est plafonné à ~30 images/s, les autres
  graphiques ne sont redessinés que sur changement effectif (`E.maj`).
- Redessin forcé sur `resize` et `orientationchange`, le canevas étant effacé par sa
  réallocation.
- Test de non-régression ajouté : 300 passes de rendu à DPR 2,75, vérification que le
  tampon reste stable, puis rotation à 780 px / DPR 3. Les validations précédentes
  n'appelaient les fonctions de dessin qu'une ou deux fois — soit exactement le nombre
  d'appels qui ne révèle pas ce défaut.

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
