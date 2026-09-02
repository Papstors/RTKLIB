# Le chemin vers un PPP float fiable et une estimation d'erreur cohérente

Branche : `claude/rtklib-ppp-convergence-re50p6` — base : `rtklibexplorer/RTKLIB` `main` du 31/08/2026 (`06e86442`).

Cadre : le PE tourne en temps réel avec `rtkrcv`, en PPP non-combiné (`est-stec`), sur le flux CNES SSRA00CNE0 (orbites, horloges, biais de code et de phase par signal, VTEC). Chaque lancement est un démarrage à froid, ailleurs, dans d'autres conditions. On veut du float, l'IAR reste un dernier recours. Le symptôme à faire disparaître : une solution qui annonce 3 cm alors qu'elle est à 20 cm, sur E, N ou U selon la session.

## Le principe qui organise tout

Un filtre de Kalman fait deux choses : il estime des états, et il estime sa propre incertitude `P`. Les deux ne valent que ce que vaut le modèle. L'erreur réelle d'une position PPP se décompose ainsi :

```
erreur réelle = bruit (dans P)  +  erreurs non modélisées (invisibles pour P)
```

Le bruit est ce que vous avez déclaré dans `R` (mesures) et `Q` (processus). Les erreurs non modélisées sont tout le reste : une erreur d'horloge SSR qui persiste dix minutes, une ambiguïté initialisée sur du code multitrajet, un biais de code manquant sur un signal, une charge océanique absente. `P` ne les voit pas, et pire, comme le filtre suppose chaque époque indépendante, `P` décroît en 1/√N pendant que ces erreurs restent entières. C'est exactement « 3 cm annoncé, 20 cm réel ».

Il y a donc deux promesses distinctes à tenir, et elles n'ont pas la même difficulté :

1. **Être honnête** : que `P` reflète l'erreur réelle. C'est le plus facile, et c'est le préalable, parce que sans cela on ne peut ni mesurer ses progrès ni protéger le client.
2. **Être juste** : que l'erreur réelle soit petite, vite, à tous les coups. C'est le plus long, et chaque progrès se mesure avec l'instrument construit au point 1.

Le chemin ci-dessous suit cet ordre. Chaque étape soit rend une erreur visible dans `P`, soit l'enlève. Une étape n'est acquise que lorsque son critère de sortie est mesuré sur le banc de rejeu.

## Étape 0 — Construire l'instrument de mesure

**Pourquoi.** Aujourd'hui vous constatez les 20 cm après coup. Il faut pouvoir rejouer n'importe quelle session à l'identique, avec une référence, et sortir deux chiffres par session : l'erreur réelle par composante, et le ratio erreur/σ annoncé.

**Ce que le code permet déjà.** `rtkrcv` peut enregistrer le brut du rover (`logstr1`) et le flux SSR (`logstr3`). `rnx2rtkp` accepte un fichier `.rtcm3` de corrections SSR (`readpreceph()`, `src/postpos.c`) : RINEX du rover + log SSR + `pos1-sateph=brdc+ssrapc` rejouent exactement la session temps réel, avec `out-outstat=residual` pour les résidus et états par satellite.

**À faire.**
- Activer les deux logs en production (déjà dans `app/consapp/rtkrcv/conf/ppp_ssr_rt.conf`).
- Une antenne de coordonnées connues pour une campagne de 24 h, découpée en fenêtres de 1 h à froid, plus vos sessions réelles quand une référence existe.
- Un script de scoring (sur le modèle de `util/ppc-dataset/score_ppc_sol.py`) qui produit par fenêtre : temps pour passer et rester sous 5 cm 3D, erreur finale E/N/U, et la série `erreur/σ` par composante.

**Critère de sortie.** Vous savez dire, pour n'importe quelle session, combien vous étiez loin et de combien le PE se trompait sur lui-même. La première campagne montrera très probablement un ratio erreur/σ de 3 à 10 en fin de convergence et des plateaux : c'est le point de départ, pas un échec.

## Étape 1 — Rendre l'erreur annoncée honnête

**Pourquoi d'abord.** Cette étape ne rapproche pas la position de la vérité. Elle fait que le PE cesse d'annoncer 3 cm quand il est à 20. Elle est rapide, elle protège immédiatement l'usage, et elle devient l'instrument de toutes les étapes suivantes : chaque amélioration réelle doit faire baisser σ **et** garder le ratio erreur/σ proche de 1.

**Le mécanisme à corriger.** Trois lignes de `src/ppp.c` résument le problème :
- `var[nv]=varerr(...)` : bruit blanc de 3 mm en phase, 30 cm en code, à chaque époque, alors que le multitrajet est corrélé sur des minutes.
- `var[nv]+=vart+SQR(C)*vari+var_rs[i]` : l'erreur orbite/horloge SSR (URA) est ajoutée comme bruit blanc, alors qu'elle persiste 5 à 30 minutes. À 1 Hz, dix minutes divisent sa trace dans `P` par environ 24.
- `rtk->P[j+j*nx]+=SQR(prn[0])*tt` : l'ambiguïté a une marche aléatoire de 1e-4 m/√s, ce qui interdit à `P` de garder l'incertitude liée à la dérive des corrections.

**Ce qu'on fait, dans l'ordre du rendement.**
1. **Temps de corrélation dans `R`** (`varerr()`). Multiplier la variance de chaque observation par `max(1, τ/Δt)` avec τ ≈ 30–60 s pour la phase et 60–120 s pour le code. Une observation à 1 Hz n'apporte pas une information neuve chaque seconde ; ce facteur le dit au filtre. La position ne change quasiment pas, `P` cesse de s'effondrer.
2. **Facteur de variance a posteriori.** Après chaque `filter()`, calculer la statistique d'innovation normalisée `vᵀ(HPHᵀ+R)⁻¹v/nv` (la matrice est déjà formée dans `filter_()`, `src/rtkcmn.c`, il suffit de l'exposer) et son moyennage exponentiel sur 5 minutes. Si elle reste au-dessus de 1, `R` et `Q` sous-estiment la réalité : publier `sol.qr × s²`, et écrire `s²` dans le `.stat`.
3. **Dérive des corrections dans `Q`.** Monter `prnbias` (1e-4 → ≈ 1e-3 m/√s) pour que l'ambiguïté puisse suivre la dérive d'horloge SSR et que `P` la garde ; ou, plus propre, un état de Gauss-Markov par satellite (τ ≈ 15 min, σ² = URA). Le réglage se fait sur le banc : on cherche un ratio erreur/σ ≈ 1, pas un σ minimal.
4. **Garde-fou empirique.** À partir de la campagne, ajuster une courbe `σ_emp(t, composante)` de l'erreur réelle en fonction du temps depuis le démarrage à froid, et publier `max(σ_formel × s, σ_emp(t))`. C'est la garantie de cohérence tant que le reste n'est pas déployé, et un plancher ensuite.
5. **Ne pas dire « convergé » sur la seule diagonale de `P`.** Conditions cumulatives : σ formel × s sous seuil, statistique d'innovation ≈ 1 sur 5 minutes, au moins six satellites dont l'ambiguïté a plus de N minutes, position stable à 3 cm sur 60 s, aucune réinitialisation récente.

**Critère de sortie.** Sur la campagne, 95 % des erreurs sont sous 2σ annoncé, par composante, y compris pendant la convergence. Le σ sera plus grand qu'avant : c'est normal, il est vrai.

## Étape 2 — Supprimer ce qui biaise un satellite

**Pourquoi.** Le fait que la composante fautive change d'une session à l'autre désigne une erreur **par satellite**, projetée sur la géométrie du moment. Une erreur systématique de modèle frapperait toujours la hauteur. En PPP float, un seul satellite biaisé suffit : son ambiguïté absorbe le biais, la position se déplace dans sa direction, et tous les autres résidus restent normaux. Le filtre ne peut pas le voir, il faut l'empêcher en amont et le contrôler en aval.

**En amont : garantir les entrées.**
- *Biais de code par signal.* En mode SSR, `corr_meas()` applique `nav->ssr[].cbias[code]` par signal : c'est correct si le CNES envoie le biais de chaque signal que vous utilisez. Vérifier sur le flux la liste des signaux couverts et faire refuser par le code tout signal sans biais (aujourd'hui le biais manquant vaut silencieusement 0).
- *Biais de phase SSR.* Les laisser désactivés en float (`misc-pppopt=-DIS_FCB`) : `decode_ssr7()` ignore le compteur de discontinuité, une discontinuité passerait pour une ambiguïté valide.
- *PCO satellite par fréquence.* `satantoff()` applique le PCO de la combinaison iono-free L1/L2 à la position du satellite quelle que soit la fréquence. En non-combiné, chaque fréquence doit recevoir `PCO_f − PCO_IF` (`corr_meas()`, `satantpcv()`). Quelques mm à cm sur Galileo et BeiDou.
- *Transitions d'IODE.* `update_ssr()` (`src/rtksvr.c:314`) ne garde que deux éphémérides par satellite ; Galileo change d'IODnav toutes les 10 minutes. Garder au moins quatre IODE et conserver la dernière correction valide pendant l'attente, sinon le satellite disparaît et revient avec une ambiguïté neuve initialisée sur du code.
- *Éclipses.* `yaw_angle()` est nominal ; exclure les satellites en éclipse en attendant les modèles.
- *GLONASS.* Son biais code récepteur par canal contamine le STEC de chaque satellite en non-combiné : rester sans GLONASS jusqu'à ce qu'il soit modélisé.

**En aval : contrôler les ambiguïtés.**
- *Pondération pendant la convergence.* Les premières minutes sont pilotées par le code ; un satellite bas avec 1–2 m de multitrajet fixe une ambiguïté fausse. Renforcer le poids d'élévation sur le code (ou masque de 15° sur le code seul) tant que la variance position dépasse ≈ 0.5 m².
- *Test d'innovation normalisé.* Remplacer les seuils absolus `maxinno` (5 m phase, 30 m code) par un test sur `HPHᵀ+R` : c'est la seule manière de rejeter 1–2 m de multitrajet quand le filtre est encore à 3 m d'incertitude.
- *Cohérence continue de chaque ambiguïté.* Comparer l'ambiguïté float à `L − P` lissé, corrigé du STEC ; un écart persistant au-delà de 3σ réinitialise l'ambiguïté avec sa variance initiale. Aujourd'hui rien ne remet en cause une ambiguïté hors saut de cycle. C'est la pièce centrale de l'étape : elle transforme un biais silencieux en une remontée honnête de `P`.
- *Sauts de cycle.* Le seuil MW de 10 m (≈ 11 cycles WL) est inopérant ; moyenne et variance glissantes (Blewitt) avec seuil 3σ, et seuil GF fonction de l'intervalle. Un saut manqué à froid est un biais définitif.
- *Séparation de solutions.* Sous-solutions en parallèle (GPS seul, Galileo seul, « tous sauf un » pour les satellites les plus pondérés) comparées à la solution complète ; un écart supérieur à ce que les covariances prédisent retire le satellite et gonfle σ. C'est le seul contrôle capable de voir un biais cohérent avec les résidus du filtre.

**Critère de sortie.** Plus de plateau : sur la campagne, l'erreur décroît de façon monotone, et les rares sessions encore fausses sont annoncées fausses (ratio erreur/σ ≈ 1 conservé).

## Étape 3 — Supprimer les biais systématiques en hauteur

**Pourquoi.** Une fois les biais satellite traités, ce qui reste en U est du modèle : troposphère, charge océanique, antenne. Ce sont des centimètres constants, invisibles pour `P`.

**À faire.**
- *Troposphère.* `trop_model_prec()` prend un ZHD par atmosphère standard et NMF (`IERS_MODEL` OFF par défaut). Passer à un a priori GPT3 (pression, température réelles au lieu de l'atmosphère standard) et GMF ou VMF ; réduire la variance initiale du ZTD (`VAR_ZTD = 0.6²`) quand l'a priori est bon ; `prntrop` ≈ 5e-5 m/√s.
- *Charge océanique.* `rtkrcv` ne lit jamais de BLQ : le bit OTL de `tidecorr` est sans effet en temps réel, et un BLQ par site est impossible quand le lieu change. Embarquer un modèle global sur grille (FES2014b ou équivalent, onze ondes) dans `tides.c`. Près des côtes, c'est 3 à 5 cm en hauteur, l'objectif entier.
- *Antenne.* ANTEX du même repère que les corrections (`igs20.atx`), type d'antenne exact dans `ant1-anttype` (pas d'en-tête RINEX en temps réel), ARP renseigné.

**Critère de sortie.** Le biais moyen en U sur la campagne est nul aux 2 cm près, quelle que soit la station.

## Étape 4 — Accélérer la convergence à froid

**Pourquoi maintenant.** Tant que le float n'est pas honnête et sans biais, accélérer ne fait que converger plus vite vers une position fausse. Une fois les étapes 1 à 3 acquises, le temps de convergence devient le sujet, et le non-combiné avec le VTEC CNES est l'outil.

**Le verrou à connaître.** En `est-stec`, le code n'a pas d'état DCB récepteur (sauf L5). Le biais P1−P2 du récepteur, environ 1 m, est absorbé par les STEC et l'horloge tant que les STEC sont libres. Dès qu'on contraint les STEC au VTEC, ce biais ressort en résidu de code et se transfère dans la position. **Le DCB récepteur par système est donc le préalable à toute contrainte iono.**

**À faire, dans l'ordre.**
1. *DCB récepteur par système* : un état constant par système, `H = 1` sur le code de la seconde fréquence, init 0 avec variance (1 m)². Généraliser `uddcb_ppp()` / `ID()`.
2. *ISB en marche aléatoire* : `udclk_ppp()` réinitialise chaque époque toutes les horloges, ISB compris. Ne réinitialiser que l'horloge GPS et garder les ISB comme états lents. Sans cela, chaque système supplémentaire coûte une inconnue par époque et n'accélère rien.
3. *Calibration matérielle persistée* : DCB, ISB, IFB sont propres au récepteur, stables sur des jours, indépendants du lieu. Les sauvegarder et les recharger avec une variance modérée (≈ 0.3 m) : c'est la seule chose qu'un démarrage à froid a le droit d'hériter.
4. *Contrainte VTEC continue.* Aujourd'hui le VTEC 1264 n'initialise les STEC qu'avec une variance fixe de 100 m² (`VAR_SSR_VTEC`, `src/ionex.c:24`), donc sans aucun poids, et son indicateur de qualité `qi` (décodé en TECU, `src/rtcm3.c:2120`) n'est pas utilisé. Ajouter dans `ppp_res()` une pseudo-observation par satellite sur l'état STEC, de variance `(max(qi, 1 TECU) × 0.1624 m/TECU × mapping)²` bornée entre ≈ 0.2 m et ≈ 1 m, désactivée si le message a plus de deux fois son intervalle de mise à jour. C'est le levier de vitesse.
5. *Bruit STEC adaptatif* : `prniono` plus grand les premières minutes et fonction de l'activité mesurée.

**Critère de sortie.** Sur la campagne : 5 à 10 minutes pour passer sous 10 cm horizontal, 10 à 20 minutes pour 5 cm 3D, à toutes les fenêtres, ratio erreur/σ toujours ≈ 1.

## Étape 5 — Ne pas reperdre la convergence

**Pourquoi.** En temps réel, une convergence acquise se perd sur un trou de flux, une transition d'IODE ou une coupure récepteur, et on repart de zéro sans le savoir.

**À faire.**
- `maxout` est compté en époques (`++outc` dans `udbias_ppp()`) : à 1 Hz, 20 vaut 20 s. Le rendre temporel, et au retour d'une coupure conserver l'ambiguïté avec une variance gonflée de `prnbias²·Δt` tant qu'aucun saut de cycle n'est détecté.
- Remplacer le rejet binaire des corrections trop vieilles (`MAXAGESSR = 90 s`) par une variance croissante avec l'âge, comme `EXTERR_CLK` en mode `precise`.
- Historique d'IODE (étape 2) et journal des trous par satellite dans le `.stat`.

**Critère de sortie.** Une coupure simulée de 10 s, 2 min, 10 min pendant une session convergée ne ramène pas l'erreur au-dessus de 5 cm, ou l'annonce.

## Étape 6 — L'IAR, en dernier recours et comme confirmation

Avec les étapes 1 à 5, le float converge de façon monotone, honnête, en 10 à 20 minutes pour 5 cm 3D. C'est la limite physique du float avec un VTEC global à quelques TECU : la hauteur et la séparation ambiguïté/ZTD ne se font que par le mouvement de la géométrie. Obtenir 5 cm 3D en moins de 5 minutes à froid, à tous les coups, n'est pas accessible en float. C'est précisément ce que les biais de phase du flux CNES permettent : WL fixée en 1 à 2 minutes, NL en 3 à 8 minutes.

La stratégie cohérente avec « IAR en dernier recours » : le float reste la solution de référence, et une AR de **confirmation** ne remplace le float que si le fix lui est cohérent (écart fix/float sous 3σ, ratio validé), et retombe sur le float sinon. Techniquement : `ppp_ar.c` est un stub, `readbiaf()` et `decode_ssr7()` doivent lever les drapeaux et gérer les discontinuités de biais de phase, puis WL par MW lissé et NL par LAMBDA sur les ambiguïtés du non-combiné. À n'ouvrir qu'une fois l'étape 4 mesurée.

## Le chemin en une page

| Étape | Ce qu'elle apporte | Où dans le code | Critère de sortie mesuré |
|---|---|---|---|
| 0 Instrument | Rejeu à l'identique, erreur et ratio erreur/σ par session | logs `rtkrcv`, `rnx2rtkp` + `.rtcm3`, script de scoring | Chaque session est mesurable |
| 1 Honnêteté | `P` reflète l'erreur réelle | `varerr()`, `filter_()` (NIS), `prnbias`, sortie σ, détecteur de convergence | 95 % des erreurs < 2σ |
| 2 Biais satellite | Plus de plateau à 20 cm dans une direction aléatoire | biais de code par signal, PCO/fréquence, IODE, `-DIS_FCB`, test d'innovation normalisé, cohérence des ambiguïtés, MW/GF, séparation de solutions | Convergence monotone, erreurs résiduelles annoncées |
| 3 Biais hauteur | U sans biais | GPT3 + GMF/VMF, OTL global dans `tides.c`, ANTEX/ARP | Biais U < 2 cm |
| 4 Vitesse | 5 cm 3D en 10–20 min à froid | DCB récepteur, ISB, calibration persistée, contrainte VTEC avec `qi`, STEC adaptatif | Temps de convergence sur toutes les fenêtres |
| 5 Tenue | Une convergence acquise ne se perd pas | `maxout` temporel, âge SSR en variance, IODE | Coupures simulées absorbées ou annoncées |
| 6 IAR | Raccourcir la fin, sans remplacer le float | biais de phase CNES, `ppp_ar.c` | Fix seulement s'il confirme le float |

Les étapes 1 et 2 se mènent en parallèle avec l'étape 0 ; l'étape 4 n'a de sens qu'après 1 à 3 ; l'étape 6 n'a de sens qu'après 4.

## Annexe A — Ce qui a été fait sur la branche

- Dépôt avancé (fast-forward) sur `rtklibexplorer/main` du 31/08/2026 : 173 commits. Upstream a changé l'unité de `pos2-dopthres` (m/s) et la sémantique de `minfixsats/minholdsats/mindropsats` (nombre de satellites) : adapter les anciens `.conf` RTK.
- Trois bugs corrigés : `udiono_ppp()` indexait `ssat[]` par index d'observation (condition retirée, l'état STEC est initialisé dès qu'il est nul, comme dans le code d'origine) ; `udiono_ppp()` en mode SSR lisait le biais du premier satellite de l'époque ; `satposs()` écrivait la variance d'horloge broadcast sur le premier satellite (`*var` → `var[i]`).
- Configs : `app/consapp/rtkrcv/conf/ppp_ssr_rt.conf` (temps réel, `est-stec`, CNES, sans GLONASS, `-DIS_FCB -GAP_RESION=120`, `aroutcnt=120`, logs rover et SSR), `data/config/ppp_static_igs.conf` et `ppp_kine_igs.conf` (rejeu avec produits IGS). Chargement et exécution vérifiés ; aucune convergence réelle mesurée ici (pas d'accès aux produits ni au flux depuis l'environnement).

## Annexe B — Inventaire des points de code cités

| Sujet | Fonction, fichier | Constat |
|---|---|---|
| Entrée PPP | `rtkpos()` → `pntpos()` → `pppos()`, `src/rtkpos.c` | La SPP tourne à chaque époque et conditionne les satellites PPP via `ssat.vs` |
| Horloges, ISB | `udclk_ppp()`, `src/ppp.c` | Toutes les horloges réinitialisées en bruit blanc chaque époque |
| STEC | `udiono_ppp()`, `ionvtec()`, `src/ppp.c`, `src/ionex.c` | Init VTEC à variance fixe 100 m², `qi` ignoré, pas de contrainte continue |
| DCB récepteur | `uddcb_ppp()`, `ID()` | L5 seulement |
| Mesures, variances | `ppp_res()`, `varerr()`, `src/ppp.c` | Bruit blanc, URA SSR blanc, rejet pré-fit absolu, post-fit 4σ un satellite à la fois |
| Ambiguïtés | `udbias_ppp()` | Init `L − P`, `prnbias` 1e-4, aucune remise en cause hors saut |
| Sauts de cycle | `detslp_gf()`, `detslp_mw()` | GF seuil fixe, MW seuil 10 m |
| Filtre | `filter_()`, `src/rtkcmn.c` | Forme `(I−KH)P`, innovation non exposée |
| SSR | `update_ssr()`, `src/rtksvr.c` ; `satpos_ssr()`, `src/ephemeris.c` | Deux IODE par satellite, âge max 90 s binaire, variance URA (0.15 m par défaut) |
| Biais SSR | `corr_meas()`, `corr_phase_bias()`, `decode_ssr7()` | Code par signal OK, phase sans drapeau ni discontinuité |
| PCO satellite | `satantoff()`, `src/preceph.c` ; `readantex()`, `src/rtkcmn.c` | Toujours IF L1/L2 ; mapping ANTEX BDS décalé |
| Tropo | `trop_model_prec()`, `tropmapf()` | Atmosphère standard + NMF (GMF si `IERS_MODEL`) |
| Marées | `tidedisp()`, `src/tides.c` ; `rtkrcv.c` | Modèles IERS corrects ; aucun BLQ lu en temps réel |
| Attitude | `yaw_angle()` | Nominal seulement |
| AR | `ppp_ar()`, `src/ppp_ar.c` | Stub |
| Persistance | `readnav()/savenav()`, `app/consapp/rtkrcv/rtkrcv.c` | Éphémérides seulement |
| Rejeu | `readpreceph()`, `src/postpos.c` | Accepte un `.rtcm3` SSR |
