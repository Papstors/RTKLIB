# Convergence PPP float du PE : analyse du code et plan d'amélioration

Branche : `claude/rtklib-ppp-convergence-re50p6` — base : `rtklibexplorer/RTKLIB` `main` du 31/08/2026 (commit `06e86442`).

Objectif visé par le PE (moteur de positionnement bâti sur `rtkpos()`/`pppos()`) : à chaque lancement, la solution PPP **float** converge vers la vraie position à 5 cm près (3D), sans biais résiduel de type 10–20 cm, sans PPP-RTK. La résolution d'ambiguïtés (IAR) reste un dernier recours, traité en section 8.

## 0. Résumé

1. **Mise à jour faite** : le dépôt a été avancé (fast-forward, aucune divergence locale) sur `upstream/main` : 173 commits, 107 fichiers. Côté PPP, upstream a ajouté depuis notre base : détection de sauts de cycle multi-fréquence en PPP, modèle VTEC (SSR 1264) pour initialiser les états iono, biais de code appliqués en absolu (OSB), correction `satposs()` pour les horloges SP3, mise à jour des IODE SSR. Compilation vérifiée (`rnx2rtkp`, CMake).
2. **Trois bugs corrigés** dans cette branche (section 4) : indexation fausse dans `udiono_ppp()` (deux occurrences) et variance d'horloge broadcast écrite sur le mauvais satellite dans `satposs()`.
3. **Constat principal** : le PPP de RTKLIB-EX est un PPP float iono-free "classique" (Takasu 2.4.3) : pas de PPP-AR (`ppp_ar.c` est un stub), pas de contrainte ionosphérique externe, pas de modélisation des ISB (biais inter-systèmes réinitialisés à chaque époque), pas de reprise d'état entre deux lancements, PCO satellites figés sur L1/L2, table de biais de code incomplète (pas de GPS L5, BDS-3, QZSS). Chacun de ces points coûte soit du temps de convergence, soit un biais résiduel float.
4. **Ce qui permet d'atteindre "≤ 5 cm à chaque lancement" en float** :
   - en **post-traitement** : le combiné forward/backward (`pos1-soltype=combined`) existe déjà et supprime la période de convergence ; il faut surtout des modèles/produits corrects (configs `data/config/ppp_static_igs.conf` et `ppp_kine_igs.conf` livrées) ;
   - en **forward / temps réel**, il faut ajouter au code : (P0-1) reprise d'état persistante (warm start), (P0-2) PPP non-combiné contraint par un modèle iono externe, (P0-3) biais OSB complets et cohérents, (P0-4) états ISB à marche aléatoire, (P0-5) PCO satellites cohérents avec la combinaison utilisée. Détail et pointeurs code en section 5.
5. Aucun jeu de données avec produits précis (SP3/CLK) n'est accessible depuis cet environnement (réseau limité à GitHub) : les configs ont été validées pour le chargement et l'exécution (mode broadcast), pas sur une convergence réelle. Le protocole de validation à dérouler est en section 7.

## 1. Mise à jour du dépôt

| | Avant | Après |
|---|---|---|
| HEAD | `28ad77c` (01/05/2026) | `06e86442` (31/08/2026) |
| Branche upstream suivie | `demo5` (figée depuis 07/2025) | `main` (tronc actif de rtklibexplorer) |
| Divergence locale | 0 commit | 0 commit (fast-forward) |

Commits upstream touchant directement le PPP depuis notre ancienne base : `192ea5b8` (sauts de cycle PPP multi-fréquences), `7d136456` (VTEC / SSR 1264, init iono), `9ada3b7` (biais de code absolus), `5c9cd9de` (`satposs` avec horloges SP3), `cc603448` (IODE SSR), `af7be189` (bug pointeur `udiono_ppp`), `0e9a16a0` (sortie `$SAT` PPP). Attention : upstream a aussi changé l'unité de `pos2-dopthres` (m/s) et la sémantique de `minfixsats/minholdsats/mindropsats` (nombre de satellites) : les anciens fichiers `.conf` RTK doivent être adaptés.

## 2. Anatomie du PPP actuel

| Étape | Fonction | Fichier | Remarque |
|---|---|---|---|
| Point d'entrée | `rtkpos()` → `pntpos()` puis `pppos()` | `src/rtkpos.c:2431` | La SPP tourne à chaque époque (`STD_PREC_VAR_THRESH=0`) et **conditionne** les satellites PPP via `ssat.vs` |
| Prédiction | `udstate_ppp()` : position, horloges, tropo, iono, DCB L5, biais de phase | `src/ppp.c:578-780` | Horloges = bruit blanc réinitialisé chaque époque, **y compris les ISB** |
| Orbites/horloges | `satposs()` → `peph2pos()` (Neville ordre 10, horloge linéaire) | `src/ephemeris.c`, `src/preceph.c:567` | Sans fichier CLK, repli sur horloges SP3 (5–15 min) puis broadcast |
| Corrections | PCO/PCV sat et récepteur, wind-up, marées (IERS `dehanttideinel` + `hardisp` + pôle), Sagnac, relativité | `ppp_res()`, `corr_meas()`, `tides.c` | Marées et wind-up de bonne qualité ; yaw = nominal seulement |
| Tropo | Saastamoinen (atm. standard) + NMF, ZWD + gradients estimés | `trop_model_prec()`, `tropmapf()` | GMF seulement si `-DIERS_MODEL` (OFF par défaut dans CMake) |
| Iono | IFLC (défaut), ou états STEC par satellite (`est-stec`), ou correction IONEX/VTEC | `model_iono()`, `udiono_ppp()` | IONEX/VTEC jamais utilisés comme **contrainte** des états |
| Mesure / filtre | `ppp_res()` + `filter()` (EKF, forme `(I-KH)P`) | `src/ppp.c:967`, `src/rtkcmn.c` | Rejet pré-fit absolu (`maxinno`), post-fit 4σ un satellite à la fois, 8 itérations max |
| Sauts de cycle | LLI, GF (seuil fixe), MW (seuil 10 m) | `detslp_*()` | MW en mètres : 10 m ≈ 11 cycles WL, quasi inopérant |
| AR | `ppp_ar()` | `src/ppp_ar.c` | **Stub** : retourne 0 |
| Post-traitement | forward/backward + `smoother()` | `src/postpos.c:553` | `combined` et `combined-nophasereset` disponibles |

## 3. D'où vient un biais float de 10–20 cm, et où cela se joue dans le code

En PPP float iono-free, la position n'est bien déterminée que lorsque les ambiguïtés float (une par satellite, `IB(s,0)`) sont séparées de la position/horloge/ZTD par le changement de géométrie. Tout ce qui ralentit cette séparation ou injecte une erreur non modélisée corrélée avec la géométrie se traduit par un biais qui peut persister des dizaines de minutes :

| Mécanisme | Ordre de grandeur | Où dans le code |
|---|---|---|
| Ambiguïtés float pilotées par le code (bruit + multitrajet + biais de code non corrigés) | 10–30 cm pendant 15–40 min | `udbias_ppp()` init `Lc-Pc` ; `varerr()` `eratio` ; `init_bias_ix()` (codes non couverts) |
| ISB réinitialisés chaque époque : chaque système ajoute une inconnue par époque, la géométrie multi-GNSS ne "paie" pas | convergence ×1.5–2 plus lente en multi-GNSS | `udclk_ppp()` (`src/ppp.c:620`) |
| Datum d'horloge produit ≠ combinaison utilisée (ex. `l1+l2+l5` → GPS L1/L5 sans OSB C5, IFCB L5 variable) | biais code de plusieurs m absorbé lentement, phase biaisée de qq cm | `seliflc()`, `corr_meas()`, `code2bias()` |
| PCO satellite calculé sur L1/L2 (ou E1/E5b, B1I/B2I) quelle que soit la combinaison | jusqu'à qq cm radial (GAL E5a vs E5b, GPS L5) | `satantoff()` (`src/preceph.c:738`) |
| ZHD a priori par atmosphère standard + NMF | 5–15 mm en hauteur, plus si pression réelle éloignée | `trop_model_prec()`, `tropmapf()` |
| Horloges satellites interpolées (SP3 5/15 min si pas de CLK 30 s) | 2–10 cm de bruit corrélé | `pephpos()` (repli), `pephclk1()` |
| Yaw nominal en éclipse (wind-up faux) | qq cm sur les satellites concernés | `yaw_angle()` |
| Sauts de cycle non détectés (MW seuil 10 m) → ambiguïté figée sur une valeur fausse | biais franc, parfois > 20 cm | `detslp_mw()` `THRES_MW_JUMP` |
| Satellites bons exclus car rejetés par la SPP mono-fréquence (`ionoopt=brdc` forcé) | géométrie affaiblie | `pntpos()`, `ppp_res()` test `ssat.vs` |
| Antenne : ANTEX pas dans le repère des produits (igs14 vs igs20), ARP absent | 1–3 cm, surtout en hauteur | `setpcv()`, `ant1-anttype=*` |

## 4. Bugs corrigés dans cette branche

1. `src/ppp.c` `udiono_ppp()` : l'initialisation de l'état iono testait `rtk->ssat[i].outc[0]` avec `i` = index d'observation au lieu de `sat-1` (introduit par le commit VTEC `7d136456`). Avec le bon index, la condition aurait bloqué définitivement tout satellite réapparaissant après une coupure > `GAP_RESION` (l'état n'est jamais initialisé, donc jamais validé, donc `outc` n'est jamais remis à zéro). La condition a été retirée : l'état est initialisé dès qu'il est nul, comme dans le code d'origine (la remise à zéro sur coupure longue est déjà faite juste au-dessus).
2. `src/ppp.c` `udiono_ppp()` (mode SSR) : biais de code lu dans `nav->ssr[obs->sat-1]` (premier satellite de l'époque) au lieu de `nav->ssr[sat-1]`.
3. `src/ephemeris.c` `satposs()` : en repli horloge broadcast, `*var=SQR(STD_BRDCCLK)` écrasait la variance du **premier** satellite au lieu de `var[i]` ; le satellite concerné gardait une variance nulle et n'était donc pas dépondéré.

Vérification : compilation OK, exécution `rnx2rtkp` en `ppp-static` et `ppp-kine` (`est-stec`) sur `test/data/rinex/07590920.05o` (broadcast) : 120/120 époques Q=6.

## 5. Améliorations de code à faire, par priorité

### P0 — nécessaires pour "toujours ≤ 5 cm" en forward

**P0-1. Reprise d'état persistante (warm start).** Aujourd'hui `rtkinit()` remet tout à zéro à chaque lancement ; rien n'est sauvegardé en fin de run (`rtkfree()`). Ajouter `pppsavestate()`/`pppload­state()` (fichier binaire ou JSON) contenant : temps, `x`/`P` complets, `ssat[].{lock,outc,slip,gf,mw,phw,pt,ph}`, `sol`. Au démarrage, si l'écart de temps est inférieur à un seuil (option `misc-pppopt=-STATEFILE=... -STATEMAXAGE=...`) : position (statique), ZTD/gradients et ISB restaurés avec leur covariance gonflée d'une marche aléatoire fonction de l'écart ; ambiguïtés restaurées uniquement si continuité de phase (pas de LLI, GF/MW cohérents), sinon réinitialisées. Effet : en statique, convergence immédiate ; en cinématique, ZTD + ISB déjà connus divisent le temps de convergence. Fichiers : `src/rtkpos.c` (`rtkinit`/`rtkfree`), `src/ppp.c` (`pppos`), `src/rtksvr.c` (temps réel), `src/postpos.c`.

**P0-2. PPP non-combiné avec contrainte ionosphérique externe.** En mode `est-stec`, ajouter dans `ppp_res()` une pseudo-observation par satellite sur l'état `II(sat)` : valeur `iontec()` (IONEX) ou `ionvtec()` (SSR 1264), variance = variance du produit (IONEX final ≈ 2–4 TECU soit 0.3–0.6 m sur L1, mappée) et, en option, pondérée par l'élévation. Initialiser les états iono depuis IONEX quand il est chargé (`udiono_ppp()` ne le fait que pour VTEC SSR). C'est le levier le plus documenté pour ramener la convergence float de 20–30 min à 5–10 min sans AR. Ajouter aussi un modèle de dérive (`prniono` dépendant de l'élévation est déjà là).

**P0-3. Biais de code OSB complets et cohérents.** `init_bias_ix()` (`src/preceph.c:69`) ne connaît que GPS L1/L2 (pas C5*), GLONASS C1/C2, Galileo E1/E5a/E5b (pas E6), BDS B1I/B3 seulement (pas B1C/B2a/B2b), aucun code QZSS. `readbiaf()` ignore les biais de phase. `pntpos()` applique les biais en mode différentiel alors que `corr_meas()` les applique en absolu. À faire : table générique indexée par (système, code) sans limite `MAX_CODE_BIASES=4`, lecture des OSB de phase (nécessaire pour l'AR, section 8), même mode dans SPP et PPP, avertissement si un code utilisé n'a pas de biais. Tant que ce n'est pas fait : rester en `pos1-frequency=l1+l2` (cf. configs livrées) et ne pas activer L5 GPS en IFLC.

**P0-4. ISB en marche aléatoire.** Dans `udclk_ppp()`, ne réinitialiser en bruit blanc que l'horloge GPS `IC(0)` ; conserver `IC(1..4)` comme ISB (état = différence par rapport à GPS, ou horloge par système à marche aléatoire lente, `prn` ≈ 1e-4 m/√s), initialisés depuis `sol.dtr[i]` la première fois. Adapter `ppp_res()` (H = 1 sur `IC(0)` et sur l'ISB du système) et `update_stat()` (calcul de `dtr[]`). Gain : la géométrie GAL/BDS contribue pleinement à la position.

**P0-5. PCO satellite cohérent avec la combinaison utilisée.** `satantoff()` reçoit `opt->nf`/`seliflc()` et calcule la combinaison iono-free des PCO des deux fréquences réellement utilisées (ou renvoie le PCO par fréquence en mode non-combiné). Corriger le mapping ANTEX BDS dans `readantex()` (`src/rtkcmn.c:2538`) : aujourd'hui C01→slot 0, C02→slot 1, C06/C07 ignorés, alors que les slots RTKLIB BDS sont B1I(2)→0, B2/B2b(7)→1, B2a(5)→2, B3(6)→3, B1C(1)→4. Les valeurs sont identiques dans `igs14.atx` (pas d'effet aujourd'hui) mais divergent dans `igs20.atx` pour BDS-3.

**P0-6. Produits et chaînes d'entrée (sans code, mais bloquant).** Horloges 30 s obligatoires (avertir au niveau `trace(2)` si `nav->nc==0` en mode `precise`), SP3 des jours J-1/J/J+1 pour l'interpolation aux bornes, ERP, ANTEX du même repère que les produits (`igs20.atx`), BLQ station, `pos1-posopt6=on` si les produits traversent minuit.

### P1 — accélération et robustesse de la convergence

**P1-1. Troposphère.** Activer GMF par défaut (option CMake `IERS_MODEL`, ou mieux rendre GMF indépendant du reste de `IERS_MODEL`), a priori ZHD par GPT2w/GPT3 (pression/température réalistes) au lieu de l'atmosphère standard, VMF1/VMF3 sur grille en option. Baisser `prntrop` par défaut (1e-4 → 5e-5 m/√s) ; `trop_model_prec()` `src/ppp.c:865`.

**P1-2. Pondération.** Remplacer le facteur fixe `SQR(3.0)` de `varerr()` (`FIXME` dans le code) par le vrai facteur d'amplification de la combinaison (2.98 L1/L2, 2.59 L1/L5 et E1/E5a, 2.81 E1/E5b), ajouter une pondération robuste type IGG3 sur résidus normalisés, et un test pré-fit normalisé par la covariance d'innovation `H P Hᵀ + R` au lieu des seuils absolus `maxinno` (5 m / 30 m) qui laissent passer 1–3 m de multitrajet code pendant la convergence.

**P1-3. Découpler le PPP de la SPP.** `ppp_res()` exige `ssat.vs` issu de `pntpos()`, qui tourne en mono-fréquence broadcast (`ionoopt=brdc`, `tropopt=saas` forcés) et peut rejeter de bons satellites (RAIM, résidu). Utiliser l'iono-free dans la SPP en mode PPP, ou ne conditionner que sur la disponibilité éphéméride/santé.

**P1-4. Sauts de cycle.** MW : remplacer le seuil fixe `THRES_MW_JUMP=10` m par moyenne/variance glissantes (Blewitt) avec seuil ≈ 3σ ou 1 cycle WL ; GF : seuil dépendant de l'intervalle (`thresslip × f(dt)`) sinon 0.05 m est trop serré à 30 s en iono active et trop lâche à 1 Hz ; après saut, réinitialiser avec `Lc-Pc` mais variance issue de `varerr()` plutôt que `VAR_BIAS=60²`.

**P1-5. Filtre.** `filter_()` utilise `(I-KH)P` ; passer en forme de Joseph (ou symétriser `P` après update) pour les longues sessions avec `MAXSAT` états iono ; vérifier la positivité des diagonales avant `matinv`.

**P1-6. GLONASS.** Biais code inter-fréquence récepteur non modélisé (seule `VAR_GLO_IFB` gonfle la variance) : estimer un IFB code par satellite (état à faible bruit) ou par canal ; sans cela GLONASS n'apporte que sa phase.

**P1-7. Attitude en éclipse.** `yaw_angle()` = yaw nominal uniquement ; implémenter les modèles d'éclipse (GPS IIR/IIF, GAL, BDS) ou exclure les satellites en éclipse (`|β|` < 4° et ombre) au lieu du seul Block IIA (`posopt4`).

**P1-8. Iono de second ordre** en mode iono-free (quelques mm à 2 cm en forte activité) à partir de l'IONEX et d'un champ dipolaire.

### P2 — hygiène

- `EFACT_GPS_L5=10` dépondère L5 d'un facteur 10 en non-combiné sans justification documentée ; à revoir avec l'IFCB.
- Journaliser dans le fichier `.stat` les $ISB, la variance position et le nombre d'ambiguïtés "matures" (âge > N époques) pour tracer objectivement la convergence.
- Tests : `test/utest/t_ppp.c` ne couvre que marées/Soleil-Lune. Utiliser `util/simobs` (simulateur d'observations à partir d'un SP3) pour un test de non-régression en boucle fermée au mm.

## 6. Configuration recommandée

Deux fichiers ajoutés : `data/config/ppp_static_igs.conf` et `data/config/ppp_kine_igs.conf`. Points clés et justification :

| Option | Valeur | Pourquoi |
|---|---|---|
| `pos1-frequency` | `l1+l2` | Combinaisons L1/L2, E1/E5b, B1I/B2I : cohérentes avec `satantoff()` et avec la table de biais actuelle. Ne pas passer en `l1+l2+l5` avant P0-3/P0-5 |
| `pos1-soltype` | `combined` | Post-traitement : lissage forward/backward, plus de période de convergence. `forward` pour simuler le temps réel |
| `pos1-elmask` | 10° | Compromis géométrie/multitrajet PPP |
| `pos1-tidecorr` | 7 | Marées solides + charge océanique (BLQ) + pôle (ERP) |
| `pos1-ionoopt` / `tropopt` | `dual-freq` / `est-ztdgrad` | Iono-free + ZWD et gradients estimés |
| `pos1-posopt1..6` | on, on, precise, on, off, on | PCV sat, PCV récepteur, wind-up, éclipse IIA, pas de RAIM SPP, saut d'horloge minuit |
| `pos1-navsys` | 41 (GPS+GAL+BDS) | GLONASS à ajouter après P1-6 |
| `pos2-armode` | off | Pas d'AR PPP dans ce code |
| `stats-eratio*` | 100 | Récepteur géodésique : σ code 0.3 m / σ phase 3 mm |
| `stats-errphase` / `errphaseel` | 0.003 m | Modèle a + b/sin(el) |
| `stats-prntrop` | 5e-5 m/√s | ≈ 3 mm/√h de marche aléatoire ZWD |
| `stats-prnpos` | 0 (statique) | Position inconnue constante |
| `ant1-anttype` | `*` | Type d'antenne et ARP lus dans l'en-tête RINEX, doivent exister dans l'ANTEX |
| `file-satantfile` / `rcvantfile` | `igs20.atx` | Même repère que les produits IGS20/repro3 (le dépôt ne livre qu'`igs14.atx`) |

Produits à fournir : SP3 + CLK 30 s (finaux ou rapides IGS/CODE/GFZ/WHU), fichier `.BIA` OSB du même centre (CODE/CAS), `.erp`, ANTEX, BLQ station.

## 7. Protocole de validation à dérouler

1. Données : 3 à 5 stations IGS/EUREF de coordonnées ITRF connues (SINEX), 24 h à 30 s (et 1 Hz si disponible), produits finaux ; jour calme et jour d'activité iono.
2. Découper chaque journée en fenêtres de 1 h (démarrage à froid à chaque fenêtre) et lancer `rnx2rtkp` en `forward` avec `out-solstatic=all` et `out-outstat=residual`.
3. Métriques par fenêtre : temps pour atteindre et rester sous 5 cm 3D (et 10 cm), erreur finale 3D et par composante, taux de fenêtres ≤ 5 cm à 60 min (cible 100 %), biais moyen hauteur.
4. Rejouer avec `combined` (cible : 100 % des époques ≤ 5 cm) puis, après P0-1, en forward avec état restauré.
5. Diagnostic : `$TROP`, `$CLK`, `$SAT` du `.stat` ; tracer résidus phase/code par satellite pour identifier les biais de modèle (signature en élévation = tropo/PCV, signature par système = ISB/biais code).
6. Un script de scoring dédié (`util/ppp-convergence/score_ppp.py`, à écrire) sur le modèle de `util/ppc-dataset/score_ppc_sol.py`.

## 8. IAR en dernier recours : ce qu'il faudrait

`ppp_ar.c` est vide et l'option `modear=4` a disparu ; `ppp_res()` conserve cependant le mécanisme `xa/Pa`, `MAX_STD_FIX` et `test_hold_amb()`. Un PPP-AR sans PPP-RTK exige : (a) produits de biais de phase OSB (CODE, CNES, WHU) → étendre `readbiaf()` aux enregistrements de phase (`L1C`, `L2W`, …) et les appliquer dans `corr_meas()` ; (b) ambiguïtés WL par MW lissé puis NL par LAMBDA sur les ambiguïtés IF (réutiliser `lambda()` de `lambda.c`), avec AR partielle et validation ratio ; (c) hold sur les ambiguïtés fixées. Ce chantier n'a de sens qu'une fois P0-3 réalisé (biais de code et de phase du même produit), et le float doit déjà être sans biais, sinon l'AR fixe des entiers faux.

## 9. Limites de cette analyse

- Pas de run PPP avec produits précis possible ici (pas d'accès aux serveurs IGS depuis le bac à sable) : les ordres de grandeur de la section 3 viennent de la littérature et de l'expérience PPP, pas d'une mesure sur ce code.
- Les valeurs de bruit de processus proposées (section 6) sont des points de départ à régler sur les données du PE.
- Les corrections de la section 4 ont été validées par compilation et exécution en mode broadcast, sans jeu de référence.
