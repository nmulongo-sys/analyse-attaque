# Analyse d'attaque — guitare

Outil de mesure du placement rythmique et de la dynamique du jeu de guitare, à partir du
micro du navigateur. Destiné à l'usage pédagogique : montrer à l'élève où tombent
réellement ses attaques par rapport à la grille, et comment il répartit ses accents.

**En ligne** : https://nmulongo-sys.github.io/analyse-attaque/
**Statut** : `index.html` en v1.5 · dépôt en v1.6 (2026-07-27) • fichier HTML unique, aucune dépendance externe, hors ligne, mobile-first.

> La v1.6 n'a livré que `protocole.html`, `PROTOCOLE.md` et de la documentation : elle
> n'a pas touché `index.html`, resté au contenu de la v1.5. Numéroter le dépôt et l'app
> ensemble a fait chercher un `index.html` v1.6 qui n'existe pas — les deux sont
> désormais distingués ici.

## Page compagnon : protocole de capture

`protocole.html` ([en ligne](https://nmulongo-sys.github.io/analyse-attaque/protocole.html))
enchaîne 13 prises à chaîne de détection figée et produit **un seul JSON brut**, sans
aucune analyse : c'est l'instrument de mesure, quand `index.html` est l'outil
d'entraînement. Voir `PROTOCOLE.md` pour le déroulement et le schéma du fichier.

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
(ce que le musicien a fait). Un accord plaqué produit plusieurs détections — non pas à
cause du balayage des cordes, comme supposé jusqu'à la série A du protocole, mais du seul
redéclenchement sur la résonance du corps dès la fin du temps réfractaire. La prise A4
(accord plaqué six cordes) ne donne que 1,1 détection par geste, contre ×2,0 à ×3,6 sur
corde à vide : le balayage n'y est pour rien. Sur une session réelle de 132 s, 293 détections correspondaient à 163 gestes.
Toutes les statistiques portent sur les gestes ; les détections restent visibles sur le
bandeau de flux, les fusionnées en trait bas pâle.

## Architecture & conventions

Cette section vise la reprise sans réexplication : elle décrit ce que le code fait, les
décisions non évidentes, et les pièges déjà payés.

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

### Prise de son


`getUserMedia` est appelé avec `echoCancellation`, `noiseSuppression` et
`autoGainControl` à `false`. Ces filtres écrasent les nuances de dynamique : sans cette
désactivation, le graphique d'intensité ne mesure plus rien. Certains Android les
réimposent malgré tout — les temps restent justes, les vélocités sont compressées.

### Chaîne de détection (worklet `detecteur-attaque`)


Analyse par trames : `N = 1024` échantillons, saut `HOP = 256`, soit une trame toutes les
**5,33 ms** à 48 kHz. Tampon circulaire alimenté par quanta de 128 échantillons.

1. **Fenêtrage** de Hann, puis FFT radix-2 itérative avec tables de rotation
   précalculées (implémentation locale, pas de bibliothèque).
2. **Blanchiment adaptatif** — chaque bande de fréquence est divisée par sa propre crête
   glissante : `pic[k] = max(mg, pic[k]·r, 2e-4)` avec `r = exp(-hop/0,6 s)`.
   Sans cette étape, les partiels graves qui battent entre eux pendant la résonance
   produisent des montées d'énergie indiscernables d'une vraie attaque.
   Cette normalisation par crête glissante par bande, sans entraînement et en temps réel,
   est la méthode publiée par Stowell & Plumbley (2007) ; elle a été retrouvée ici
   empiriquement. Voir *Références*.
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

### Passages — la fenêtre n'enjambe jamais un silence


Un silence supérieur à `max(4 × pas, 3 s)` ouvre un nouveau **passage**. Une fenêtre
d'analyse à cheval sur une pause compare des gestes séparés de plusieurs secondes et fait
chuter l'accroche sans que le jeu ait changé : sur une prise réelle, 93 % avant la pause,
72 % affichés après. Les fenêtres à cheval sont exclues de la courbe, et les chiffres
portent sur le passage en cours.

Si le passage courant compte moins de huit gestes — on vient de reprendre après une
pause — l'outil remonte au dernier passage exploitable et le signale, plutôt que de
n'afficher plus rien : l'utilisateur vient de jouer, il attend un retour sur ce qu'il a
joué.

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

### Estimation du pas réellement joué


`tempoJoue()` cherche le pas de grille le plus fin qui explique le jeu, indépendamment de
la grille réglée, et le convertit en bpm via la subdivision déclarée. Il répond à « à quel
tempo joue-t-il », pas à « respecte-t-il le mien ». Deux pièges, tous deux rencontrés en
conditions réelles :

1. **Ancrer la recherche sur l'intervalle médian** (v1.3) : dès que le jeu mêle des
   valeurs rythmiques, la médiane glisse vers les valeurs courtes et la vraie pulsation
   sort de la plage explorée. Session à 40 bpm, médiane 1067 ms, plage [832, 1301] ms :
   la période réelle de 1500 ms était inatteignable, l'outil répondait 58,7 bpm.
2. **Balayer large à résolution fixe** : la résolution nécessaire vaut environ
   `p²/(4·durée)`. À 1 s de période sur 117 s de jeu, il faut mieux que 2 ms ; un balayage
   logarithmique en 300 points donne 9 ms au voisinage de 1 s, accumule 420 ms de dérive
   et manque complètement le maximum.

D'où une **estimation par étages** : passe grossière sur une portée courte (20 s, où
l'exigence de résolution est faible), puis deux affinages locaux à ±6 % en élargissant la
portée à 60 s puis à tout l'historique. Validée sur deux sessions réelles (39,8 bpm pour
une grille à 40, 60,1 bpm pour une grille à 60) et sur synthèses de 40 à 132 bpm.

Résultat mis en cache, recalculé tous les huit gestes — le coût est d'environ 200 000
itérations trigonométriques. **La clé de cache inclut l'instant du premier geste**, sans
quoi elle ne s'invalide pas lorsque la série change à effectif constant.

### Constantes et seuils de décision


| Constante | Valeur | Rôle |
|---|---|---|
| `PARAM.sens` | 2,50 | multiplicateur du seuil médian, réglable |
| `PARAM.ioi` | 55 ms | temps réfractaire du détecteur — **paramètre de comptage, pas de précision** (voir journal du 27/07) |
| `PARAM.fusion` | 120 ms | fenêtre de regroupement en gestes |
| `PARAM.fenetre` | 24 gestes | fenêtre glissante d'analyse |
| `FEN` | ±25 ms | fenêtre « dans la cible » — **valeur posée arbitrairement, à remplacer par une fenêtre relative** |
| `ODF_MAX` | 760 trames | ~4 s d'historique du bandeau de flux |
| porte de niveau | −58 dBFS | ne coupe rien, même en palm mute faible (série A) |
| échauffement | 0,45 s | convergence des crêtes par bande |
| bande utile | 1 → 6 kHz | bins retenus pour le flux |
| décroissance du blanchiment | 0,6 s | constante de la crête glissante par bande |
| historique du seuil | 45 trames | ≈ 240 ms de médiane glissante |

Seuils de décision, tous dans `bilan()` :

| Test | Condition | Conséquence |
|---|---|---|
| accroche significative | `p < 0,001` **et** `R > 0,25` | sans quoi biais et régularité ne sont pas affichés |
| subdivision autre | `R(pas/m) > 0,45` **et** `> R₁ + 0,15` | sans quoi le discriminateur se tait |
| rupture de passage | silence `> max(4 × pas, 3 s)` | ouvre un nouveau passage |
| qualificatif de placement | σ < 15 / 30 / 50 ms | très stable / stable / irrégulier / très irrégulier |
| alerte dynamique | étendue > 30 dB | signale de probables fausses détections |
| avertissement de tempo | écart > 6 % **et** subdivision ajustée = 1 | sinon le message de subdivision suffit |

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

### 2026-07-27 — consolidation de la documentation
Fusion de deux branches de documentation écrites en parallèle par deux sessions, et
correction de ce que les mesures du protocole ont périmé. Aucun changement de code.

- **Correctif : le README décrivait un algorithme supprimé.** La section statistique
  documentait encore l'estimation du tempo ancrée sur l'intervalle médian — celle retirée
  en v1.4 parce qu'elle répondait 58,7 bpm sur une grille à 40. Le remplacement de v1.4
  avait échoué silencieusement : chaîne cible inexacte, aucune assertion pour le signaler.
  L'estimation par étages est maintenant documentée dans sa propre section.
- **Correctif : l'explication des doublons par le balayage des cordes était fausse.** La
  prise A4 la contredit (1,1 détection par geste sur un accord plaqué). Section d'état
  corrigée ; les entrées de journal antérieures sont conservées telles quelles, elles
  décrivent l'état des connaissances à leur date.
- **Versionnement dissocié** entre le dépôt et `index.html`, qui n'a pas bougé depuis la
  v1.5. Numéroter les deux ensemble a fait chercher un fichier inexistant.
- Sous-sections d'architecture réordonnées selon le flux du traitement — prise de son →
  détection → référentiel → grille → analyse → rendu → export — au lieu de l'ordre
  accidentel produit par six révisions par insertion.
- Ajout d'une section « Constantes et seuils de décision » rassemblant les valeurs
  jusque-là dispersées en prose, et d'une section « Contributions ».

*Leçons consignées.* Toute réécriture programmatique vérifie que sa chaîne cible existe
avant de remplacer : une substitution sans effet ne laisse aucune trace et la
documentation dérive du code en silence. Et avant d'écrire dans un dépôt partagé par
plusieurs sessions, relire l'état distant : cette fusion a failli écraser l'analyse du
protocole, poussée pendant que je travaillais sur une copie antérieure.

### 2026-07-27 — analyse du premier protocole complet (13 prises)
Dépouillement de l'export `protocole-2026-07-27-03-17-23.json` : Android Chrome, casque
confirmé, chaîne figée, `ecart_min_ms = 30`, 14 prises dont un rejet motivé. Aucune
modification de code — cette entrée consigne des résultats de mesure.

**Série A — le détecteur ne manque rien à 1000 ms.** Vérité terrain de 48 attaques et
48 cases de grille par prise.

| Prise | Détections brutes | Cases occupées | Lecture |
|---|---|---|---|
| A1 acier / médiator / mi grave | 96 | 48/48 | complet, ×2,0 redéclenchements |
| A2 acier / médiator / mi aigu | 107 | 48/48 | complet, ×2,2 |
| A3 nylon / doigt | 173 | 48/48 | complet, ×3,6 |
| A4 accord plaqué 6 cordes | 52 | 29/48 | ×1,1 |
| A5 palm mute faible | 281 | 48/48 | accroche ≤ 0,31 — inexploitable |
| A6 hammer-on / pull-off | 151 | 46/48 | accroche 0,05–0,09 — inexploitable |

Aucune attaque manquée, ni sur acier ni sur nylon, ni au médiator ni au doigt. La porte à
−58 dBFS ne coupe rien, même en palm mute délibérément faible. Le problème n'est jamais la
détection : c'est le redéclenchement, ×2 à ×3,6 selon le timbre, le nylon au doigt étant
le pire cas.

- **A6 tranché : non.** Les hammer-on et pull-off produisent 151 détections sans aucune
  structure temporelle exploitable. À écarter de tout parcours de mesure.
- **A5 tranché : non.** Le palm mute faible sature le détecteur sans jamais s'accrocher.
  Un travail sur les étouffées ne sera pas mesurable par cette chaîne.
- **A4 contredit l'hypothèse du balayage.** L'accord plaqué ne donne que 1,1 détection par
  geste. Les doublons à 59–75 ms relevés en v1.3 et v1.5 ne venaient donc pas des cordes
  balayées ; l'attribution au seul redéclenchement sur résonance est confirmée.

**`ecart_min_ms` ne pilote pas la qualité de la mesure.** Balayage d'un réfractaire greedy
de 30 à 600 ms sur les treize prises : l'accroche ne bouge pas (A1 reste à 0,72–0,79, B3 à
0,79–0,82, C2 à 0,71–0,82) alors que le nombre de gestes retenus change du simple au
double. C'est un paramètre d'affichage et de comptage, pas de précision : à figer une fois
(120 ms) et à ne plus exposer.

**Série B — compromise par un décrochage de tempo, pas par l'outil.**

| Prise | Pas | Tempo joué | Écart | Dérive sur la prise | R |
|---|---|---|---|---|---|
| B1 | 1000 ms | 60,2 | −0,3 % | +129 ms | 0,70 |
| B2 | 667 ms | 91,5 | −1,6 % | +520 ms | 0,83 |
| B3 | 500 ms | 119,7 | +0,3 % | −80 ms | 0,82 |
| B4 | 333 ms | 85,2 | +5,6 % | −1591 ms | 0,40 |
| B5 | 250 ms | 112,5 | +6,7 % | −1875 ms | 0,18 |

En B4 et B5 le jeu n'est pas synchronisé au clic : cinq à sept tours de phase complets sur
30 s, l'accroche s'effondre mécaniquement. B5 ajoute une perte de détection réelle
(74 détections brutes pour ≈ 112 attaques effectivement jouées). Ces deux prises sont à
refaire, et `protocole.html` doit refuser une prise dont le tempo joué s'écarte de plus de
3 % du tempo réglé — sinon la page produit des JSON qui coûtent une analyse pour rien.

**Coefficient de la fenêtre de tolérance.** Dispersion mesurée par les intervalles entre
attaques successives, immune à la dérive :

| Intervalle | σ locale | σ / intervalle |
|---|---|---|
| 333 ms (B4) | 20,0 ms | 6,0 % |
| 375 ms (C1) | 20,5 ms | 5,5 % |
| 375 ms (C2) | 24,8 ms | 6,6 % |
| 667 ms (B2) | 47,2 ms | 7,1 % |
| 1000 ms (B1) | 57,6 ms | 5,8 % |
| 1000 ms (A1) | 70,3 ms | 7,0 % |

σ ≈ **6 % de l'intervalle**, stable de 333 à 1000 ms sur sept mesures indépendantes. B3
(11,1 %) et B5 (12,2 %) s'en écartent mais ne reposent que sur 17 et 18 intervalles, et ne
sont pas comptés. Conséquence pour `FEN`, aujourd'hui constante à ±25 ms : la valeur est
juste par accident autour de 375–400 ms, absurdement sévère à 1000 ms et laxiste à 250 ms.
Une fenêtre relative à 6 % de l'intervalle est proposée. **Non implémentée** : la loi
demande confirmation sur une seconde série. Rapportée au seuil perceptif de Friberg &
Sundberg (2,5 %), la production se situe à 2,4× la finesse de la perception — sens attendu.

**Série C — bénéfice de subdivision : lecture partagée.** Même geste, croches à 80 bpm,
seul le clic change. C1 (clic sur les temps) : R = 0,66, σ locale 20,5 ms. C2 (clic sur les
croches) : R = 0,77, σ locale 24,8 ms. Le clic subdivisé améliore nettement l'ancrage de
phase et n'améliore pas la régularité locale ; la chute de dispersion attendue d'après
Repp (2003) n'apparaît pas. Une prise par condition, ≈ 54 gestes : indicatif, non
concluant. À rejouer en trois répétitions alternées avant d'en tirer une règle.

**Le biais reste inexploitable.** De +130 à +440 ms selon les prises. `outputLatency`
déclaré à 4 ms par Chrome Android, invraisemblable au casque. Aucune calibration par boucle
acoustique n'est prévue dans le protocole : à ajouter en tête de séance.

*Note de méthode.* La couverture de grille brute sous-estime le jeu réel dès que la latence
non calibrée dépasse `pas/2` — en B4, 215 ms contre 166 ms : les attaques basculent dans la
case suivante et sont comptées comme des trous. Aux pas courts, tout comptage de couverture
doit être recentré sur le biais circulaire avant d'être lu. L'accroche R, elle, est
invariante par translation de phase et n'est pas concernée.

### 2026-07-27 — v1.6, page de protocole de capture
Ajout de `protocole.html` et `PROTOCOLE.md`. Séparation nette des deux usages : `index.html`
est l'outil d'entraînement, qui interprète en temps réel ; `protocole.html` est
l'instrument de mesure, qui ne fait que piloter et capturer.

Motif : les trois premières séances ne comparaient pas la même tâche — valeurs mêlées
contre grille grossière d'un côté, un geste par clic de l'autre. Aucune conclusion sur la
dispersion n'est possible tant que le geste, l'instrument, la corde et le rapport
geste/clic ne sont pas tenus fixes. La page les impose.

Points de conception qui comptent :
- `ecart_min_ms = 30`, volontairement bas, sur toutes les prises : le regroupement se fera
  en post-traitement à plusieurs valeurs sur le même flux, seul moyen de savoir si les
  doublons à 59–75 ms sont réels ou fabriqués par le paramètre.
- Aucune fusion destructive avant export ; `chef` n'est qu'un marquage informatif.
- État **effectif** du micro journalisé via `track.getSettings()`, pas la valeur demandée.
- Casque verrouillant : sans confirmation, pas de démarrage.
- Sauvegarde `localStorage` après chaque prise — quinze minutes de jeu ne se perdent pas
  sur un rechargement.

### 2026-07-27 — v1.5, découpage en passages
Troisième export réel (90 bpm, subdivision croches, 52 gestes pour 82 détections). Prise
de loin la meilleure : période produite 334 ms — le pas réglé au millième —, accroche
85 %, dispersion 30 ms, biais −4 ms, 80 % des gestes dans ±35 ms.

- **Défaut trouvé dans les données** : la prise contenait un silence de 17 s, et la
  fenêtre glissante l'enjambait. L'outil affichait 78 % là où le jeu valait 85 %, et 93 %
  sur son meilleur passage. Un silence supérieur à `max(4 × pas, 3 s)` ouvre désormais un
  nouveau passage ; aucune fenêtre n'est mesurée à cheval sur une pause, et les fenêtres
  concernées sont exclues de la courbe de progression.
- Repli sur le dernier passage exploitable quand le passage courant compte moins de huit
  gestes, avec mention explicite dans le diagnostic.
- Le réglage du clic (muet / sur les temps / toutes subdivisions) et le nombre de passages
  figurent maintenant dans l'export : sans eux, impossible de savoir *a posteriori* si une
  prise relevait ou non de la condition « bénéfice de subdivision ».
- Chaque geste exporté porte son numéro de passage.
- Test `vsilence` ajouté : rejeu de la prise réelle, vérification que l'accroche remonte
  au-dessus de 80 % et que 27 fenêtres à cheval sont bien écartées.

*Note d'analyse.* Les deux premières séances suggéraient une dispersion constante en
proportion de l'intervalle (14,7 % puis 12,4 %) — hypothèse de type loi de Weber. La
troisième donne 9,0 % et la contredit. Elle diffère cependant sur deux axes à la fois :
intervalle plus court **et** flux continu d'un geste par pas de grille, ce dernier
constituant en soi la condition de « bénéfice de subdivision » décrite par Repp (2003).
Les données disponibles ne permettent pas de séparer les deux causes.

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

**Fenêtre de tolérance relative.** `FEN` vaut ±25 ms, valeur posée arbitrairement : juste
par accident autour de 375–400 ms, absurdement sévère à 1000 ms, laxiste à 250 ms. Le
dépouillement du premier protocole donne σ ≈ 6 % de l'intervalle, stable de 333 à 1000 ms
sur sept mesures indépendantes. Une fenêtre relative à 6 % est proposée — **non
implémentée**, la loi demande confirmation sur une seconde série.

**Retirer `ecart_min_ms` de l'interface.** Le balayage de 30 à 600 ms montre que l'accroche
n'en dépend pas alors que le nombre de gestes double : paramètre de comptage, pas de
précision. À figer à 120 ms et à ne plus exposer.

**Calibration par boucle acoustique en tête de séance.** Le biais reste inexploitable, de
+130 à +440 ms selon les prises, et l'`outputLatency` déclaré par Chrome Android (4 ms au
casque) est invraisemblable. Sans calibration, seule la dispersion est lisible.

**Vérification d'accord.** Chromagramme sur fenêtre longue (8192 points, ≈ 2,7 Hz/bin,
nécessaire pour séparer un demi-ton dans le grave), déclenché par l'attaque détectée. Dans
un contexte pédagogique l'accord attendu est connu : le problème n'est pas la *détection*
en classe ouverte mais la *vérification* contre un gabarit, plus fiable, et capable
d'indiquer quelle corde manque. Verdict ternaire : conforme / confusion identifiée /
indéterminé — jamais de verdict autoritaire faux.

## Contributions

Signalements et suggestions par *issue*, propositions de code par *pull request*, relues au
cas par cas. Trois conventions valent pour toute proposition touchant l'analyse : aucune
statistique n'est affichée tant que sa significativité n'est pas établie ; tout changement
de comportement du détecteur s'accompagne d'un banc de test reproductible ; et l'état
distant du dépôt se relit avant écriture, plusieurs sessions pouvant y travailler.

## Références

- Stowell, D., & Plumbley, M. D. (2007). *Adaptive whitening for improved real-time audio
  onset detection.* Proc. International Computer Music Conference (ICMC).
  http://epubs.surrey.ac.uk/811731/ — normalisation de chaque bande par un maximum récent,
  sans entraînement, en temps réel : la méthode employée à l'étape 2 de la chaîne de
  détection.
- Dixon, S. (2006). *Onset detection revisited.* Proc. 9th International Conference on
  Digital Audio Effects (DAFx-06), Montréal, 133–137.
  https://www.dafx.de/paper-archive/2006/papers/p_133.pdf — comparaison des fonctions de
  détection d'attaque et du seuillage adaptatif par médiane glissante.
- Friberg, A., & Sundberg, J. (1995). *Time discrimination in a monotonic, isochronous
  sequence.* Journal of the Acoustical Society of America, 98(5), 2524–2531.
  https://doi.org/10.1121/1.413218 — seuil différentiel d'environ 6 ms pour des intervalles
  de 100 à 240 ms, et 2,5 % au-delà. Référence perceptive à laquelle est rapportée la
  dispersion mesurée.
- Repp, B. H. (2003). *Rate limits in sensorimotor synchronization with auditory sequences.*
  https://pubmed.ncbi.nlm.nih.gov/14607773/ — bénéfice de subdivision, condition testée par
  la série C du protocole.
- Repp, B. H. (2005). *Sensorimotor synchronization: a review of the tapping literature.*
  https://link.springer.com/article/10.3758/BF03206433 — asynchronie moyenne et SDasy, les
  deux grandeurs que l'outil nomme biais et régularité.
- Repp, B. H., & Su, Y.-H. (2013). *Sensorimotor synchronization: a review of recent
  research (2006–2012).* https://pubmed.ncbi.nlm.nih.gov/23397235/

## Licence

Aucun fichier `LICENSE` dans le dépôt : tous droits réservés par défaut.
