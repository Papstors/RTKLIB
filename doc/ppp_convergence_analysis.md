# Convergence PPP float du PE : analyse du code et plan d'amélioration

Branche : `claude/rtklib-ppp-convergence-re50p6` — base : `rtklibexplorer/RTKLIB` `main` du 31/08/2026 (commit `06e86442`).

Objectif visé par le PE (moteur de positionnement bâti sur `rtkpos()`/`pppos()`) : à chaque lancement, la solution PPP **float** converge vers la vraie position à 5 cm près (3D), sans biais résiduel de type 10–20 cm, sans PPP-RTK. La résolution d'ambiguïtés (IAR) reste un dernier recours, traité en section 8.

## 0. Résumé

1. **Mise à jour faite** : le dépôt a été avancé (fast-forward, aucune divergence locale) sur `upstream/main` : 173 commits, 107 fichiers. Côté PPP, upstream a ajouté depuis notre base : détection de sauts de cycle multi-fréquence en PPP, modèle VTEC (SSR 1264) pour initialiser les états iono, biais de code appliqués en absolu (OSB), correction `satposs()` pour les horloges SP3, mise à jour des IODE SSR. Compilation vérifiée (`rnx2rtkp`, CMake).
2. **Trois bugs corrigés** dans cette branche (section 4) : indexation fausse dans `udiono_ppp()` (deux occurrences) et variance d'horloge broadcast écrite sur le mauvais satellite dans `satposs()`.
3. **Constat principal** : le PPP de RTKLIB-EX est un PPP float iono-free "classique" (Takasu 2.4.3) : pas de PPP-AR (`ppp_ar.c` est un stub), pas de contrainte ionosphérique externe, pas de modélisation des ISB (biais inter-systèmes réinitialisés à chaque époque), pas de reprise d'état entre deux lancements, PCO satellites figés sur L1/L2, table de biais de code incomplète (pas de GPS L5, BDS-3, QZSS). Chacun de ces points coûte soit du temps de convergence, soit un biais résiduel float.
4. **Cible : temps réel avec `rtkrcv`, PPP non-combiné (`est-stec`) et le flux CNES SSRA00CNE0, en démarrage à froid à chaque lancement, lieu et conditions différents** (sections 10 à 12). Le warm start (RT-1) ne sert donc qu'aux biais matériels du récepteur (DCB, ISB), pas à la position ni aux ambiguïtés. Le combiné forward/backward ne s'applique donc qu'au rejeu de validation. Pour atteindre "≤ 5 cm à chaque lancement" en float temps réel, il faut ajouter au code : (RT-1/P0-1) reprise d'état persistante (warm start) branchée dans `rtksvrstart/stop`, (RT-2) gestion des transitions d'IODE SSR, (RT-4) survie aux coupures de flux sans perdre les ambiguïtés, (P0-2/RT-5) PPP non-combiné contraint par le VTEC SSR 1264 déjà décodé, (P0-3) biais OSB complets et cohérents, (P0-4) états ISB à marche aléatoire, (P0-5) PCO satellites cohérents avec la combinaison utilisée. Hypothèse `est-stec` : le VTEC 1264 est aujourd'hui utilisé avec une variance fixe de 100 m² (aucun poids) et son indicateur de qualité est ignoré ; il manque un état DCB récepteur par système, indispensable dès que le STEC est contraint. Configs livrées : `app/consapp/rtkrcv/conf/ppp_ssr_rt.conf` (temps réel, `est-stec`) et `data/config/ppp_static_igs.conf` / `ppp_kine_igs.conf` (rejeu, post-traitement).
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

- Pas de run PPP avec produits précis ni flux SSR possible ici (pas d'accès aux serveurs IGS depuis le bac à sable) : les ordres de grandeur de la section 3 viennent de la littérature et de l'expérience PPP, pas d'une mesure sur ce code.
- Les valeurs de bruit de processus proposées (section 6) sont des points de départ à régler sur les données du PE.
- Les corrections de la section 4 ont été validées par compilation et exécution en mode broadcast, sans jeu de référence.

## 10. Temps réel avec rtkrcv (cible principale)

Le PE tourne en temps réel : pas de combiné forward/backward, et les produits sont un flux SSR RTCM3 (ou des SP3/CLK ultra-rapides chargés par ftp/http en mode `precise`, dont la moitié prédite des horloges est trop bruitée pour l'objectif). Ce que le code fait aujourd'hui, puis ce qui manque.

### 10.1 Chaîne temps réel dans le code

- Flux : `inpstr1` rover, `inpstr3` corrections. `decoderaw()` → `update_ssr()` (`src/rtksvr.c:314`) ne copie une correction dans `nav.ssr` que si son drapeau `update` est levé **et** si son IODE correspond à l'éphéméride courante ou précédente reçue du rover ; sinon elle est ignorée.
- `satpos_ssr()` (`src/ephemeris.c`) : orbite broadcast + ΔR/A/C, horloge broadcast + polynôme SSR, âge maximal 90 s (`MAXAGESSR`), 10 s pour l'horloge haute cadence, variance issue de l'URA SSR (`DEFURASSR` 0.15 m si absent). `brdc+ssrapc` = corrections référencées au centre de phase (IGS, CNES), `brdc+ssrcom` ajoute `satantoff()`.
- Biais de code SSR (1059/1242/1260) : appliqués dans `corr_meas()` pour le PPP, pas dans la SPP (`prange()`).
- Biais de phase SSR (1265–1270) : appliqués aux observations par `corr_phase_bias()` sauf `misc-pppopt=-DIS_FCB` ; `decode_ssr7()` ne lève pas `update` et ignore le compteur de discontinuité `sdc`.
- VTEC (1264) : décodé et copié dans `nav.vtec`, utilisé uniquement pour initialiser les états iono.
- Persistance : à l'arrêt `rtkrcv` sauvegarde les éphémérides (`savenav()`, `rtkrcv.nav`) et les relit au démarrage (`readnav()`), rien pour l'état du filtre ; `restart` → `rtkinit()`.
- Galileo : `setseleph(SYS_GAL,1)` force l'I/NAV en mode SSR ; le récepteur doit donc sortir l'I/NAV et le fournisseur SSR doit s'y référer (cas IGS et CNES).
- Rejeu : `rnx2rtkp` accepte un fichier `.rtcm3` de corrections SSR (`readpreceph()`), donc un log rover + un log SSR (`logstr1`, `logstr3`) rejouent exactement la session temps réel.

### 10.2 Points spécifiques temps réel, à ajouter aux P0/P1

**RT-1 (P0). Warm start.** C'est le chantier n° 1 en temps réel. Brancher `pppsavestate()/pppload­state()` dans `rtksvrstart()`/`rtksvrstop()` à côté de `readnav()/savenav()` (`app/consapp/rtkrcv/rtkrcv.c:1933,1988`), plus un point de reprise périodique (toutes les 30 s) pour survivre à un crash ou un `restart`. Restauration conditionnelle : âge de l'état, continuité de phase par satellite (LLI, GF, MW), sinon seules position/ZTD/ISB sont reprises.

**RT-2 (P0). Transitions d'IODE.** `update_ssr()` n'accepte que deux jeux d'éphémérides (courant, précédent). Galileo change d'IODnav toutes les 10 min, GPS toutes les 2 h : à chaque transition la correction est perdue jusqu'à ce que rover et flux soient alignés, ce qui crée des trous de plusieurs dizaines de secondes par satellite ; `outc` monte et l'ambiguïté est réinitialisée au-delà de `maxout`. Garder un historique d'au moins quatre IODE par satellite, conserver la dernière correction valide pendant l'attente (l'âge la borne déjà) et journaliser les trous par satellite.

**RT-3 (P0). Biais de phase SSR en float.** Ne pas les appliquer (`-DIS_FCB`) : toute discontinuité passe inaperçue et casse l'ambiguïté. Pour l'AR future : lever `update` dans `decode_ssr7()`, exploiter `sdc` pour réinitialiser l'ambiguïté concernée.

**RT-4 (P0). Coupures de flux sans perte des ambiguïtés.** `maxout` est compté en époques (`++outc` à chaque époque dans `udbias_ppp()`) : à 1 Hz, `pos2-aroutcnt=20` vaut 20 s. Une coupure NTRIP ou récepteur de deux minutes efface toutes les ambiguïtés et relance la convergence. Rendre le seuil temporel (secondes), et au retour conserver l'ambiguïté avec variance gonflée de `prnbias²·Δt` tant qu'aucun saut de cycle n'est détecté ; avec RT-1 la coupure est absorbée.

**RT-5 (P0). Contrainte iono par le VTEC 1264** : c'est P0-2 avec une source déjà disponible dans `nav.vtec` ; pseudo-observations dans `ppp_res()`, variance dérivée de la qualité du message et de l'élévation.

**RT-6 (P1). Latence.** Horloges SSR mises à jour toutes les 5 s (IGS03, CNES), latence NTRIP de 2 à 5 s : `MAXAGESSR=90` s convient, mais remplacer le seuil binaire par une variance croissante avec l'âge (comme `EXTERR_CLK` en mode `precise`).

**RT-7 (P1). Choix du flux.** IGS03 : GPS+GLONASS, ≈ 3 cm orbite / 0.15 ns horloge. SSRA00CNE0 (CNES) : GPS/GAL/GLO/BDS, biais code et phase, VTEC 1264. À froid, compter 15–30 min de convergence float ; avec RT-1 et RT-5, quelques minutes.

**RT-8 (P1). Indicateur de convergence en sortie.** `out-maxsolstd` permet de ne publier que sous un écart-type filtre (optimiste) ; ajouter dans `sol_t` un critère fondé sur l'âge médian des ambiguïtés et le nombre de satellites matures.

**RT-9 (P2).** `prange()` n'applique pas les biais SSR dans la SPP (impact limité au filtrage `vs`) ; mesurer le temps par époque sur la cible embarquée en `est-stec` (≈ 300 états).

### 10.3 Config rtkrcv livrée

`app/consapp/rtkrcv/conf/ppp_ssr_rt.conf` : rover sur `inpstr1`, flux CNES SSRA00CNE0 sur `inpstr3`, `pos1-sateph=brdc+ssrapc`, `pos1-ionoopt=est-stec`, `l1+l2`, GPS+GAL+BDS (pas de GLONASS, voir 11.1), ZTD + gradients, marées solides + OTL, `stats-prniono=0.002`, `misc-pppopt=-DIS_FCB -GAP_RESION=120`, `pos2-aroutcnt=120` (2 min à 1 Hz), `misc-navmsgsel=rover` (éphémérides du seul rover pour l'appariement d'IODE), `ant1-anttype` à renseigner explicitement (pas d'en-tête RINEX en temps réel), logs rover et SSR activés pour le rejeu. Chargement vérifié avec `rtkrcv -o`.

### 10.4 Validation temps réel

1. Enregistrer rover + SSR (`logstr1`, `logstr3`) sur une antenne de coordonnées connues, 24 h.
2. Rejouer avec `rnx2rtkp` (RINEX du rover + `.rtcm3` SSR, `sateph=brdc+ssrapc`) en fenêtres de 1 h à froid : mêmes métriques qu'en section 7.
3. Tester `stop`/`start` et coupures de flux simulées (10 s, 2 min, 10 min) : la reprise doit rester sous 5 cm après RT-1 et RT-4.

## 11. Hypothèse retenue : PPP non-combiné (`est-stec`) avec le flux CNES

Le flux SSRA00CNE0 (CNES PPP-WIZARD) apporte orbites et horloges GPS/GAL/GLO/BDS, biais de code et de phase par signal, et le VTEC en harmoniques sphériques (1264). C'est la configuration qui permet un PPP non-combiné contraint, donc la convergence float la plus rapide sans AR. Ce que le code fait en `est-stec`, et ce qui doit changer.

### 11.1 Le mode `est-stec` dans le code

| Point | État actuel | Conséquence |
|---|---|---|
| États | position, horloge par système, ZTD + gradients, un STEC par satellite (`II`), DCB récepteur L5 seulement (`ID`, si `nf≥3`), biais de phase par fréquence et satellite (`IB(s,f)`) | ≈ 300 états avec `MAXSAT` ; `filter()` compresse les états nuls |
| Observations | code et phase bruts par fréquence, coefficient iono `C=±(f1/f)²`, pas d'amplification ×3 du bruit | modèle plus riche, mais le rang dépend des biais récepteur (ci-dessous) |
| Init STEC | `udiono_ppp()` : VTEC 1264 via `ionvtec()` si présent, sinon `(P1−P2)` corrigé des biais ; variance fixe `VAR_SSR_VTEC = SQR(10.0)` (`src/ionex.c:24`) ; l'indicateur de qualité `qi` (décodé en TECU, `src/rtcm3.c:2120`) n'est pas utilisé | le VTEC n'a **aucun poids** : il ne fait qu'initialiser, la convergence reste celle d'un PPP non contraint |
| Marche aléatoire STEC | `prn[1]/sin(el)` (`stats-prniono`) | à régler : 0.001 iono calme, 0.003–0.005 iono active ou mobile rapide |
| Utilisation d'un satellite | seulement si son état STEC est initialisé (`ppp_res()` : `x[II]==0` → ignoré) | sans VTEC, il faut les deux codes |
| Biais de code SSR | appliqués par signal dans `corr_meas()` avec un index direct par code : toutes les fréquences envoyées par le CNES sont couvertes (contrairement à la table fichier de `init_bias_ix()`) | OK en temps réel ; la SPP (`prange()`) ne les applique pas |
| Biais de phase SSR | `-DIS_FCB` recommandé en float (RT-3) | sans effet sur le float non-combiné |
| DCB récepteur | absent (sauf L5). Le biais P1−P2 du récepteur (plusieurs ns, soit ≈ 1 m) est absorbé par les STEC et l'horloge | invisible tant que le STEC est libre ; **dès que le STEC est contraint au VTEC, il ressort en résidu de code et biaise la solution** |
| GLONASS | IFB code récepteur par canal non modélisé | en non-combiné il contamine le STEC de chaque satellite GLONASS ; exclure GLONASS (`navsys=41`) tant que ce n'est pas modélisé |
| PCO satellite | `satantoff()` applique le PCO de la combinaison IF L1/L2 à la position du satellite, quelle que soit la fréquence | en non-combiné, chaque fréquence devrait recevoir `ΔPCO_f = PCO_f − PCO_IF` (mm à cm sur GAL/BDS) |
| Init biais de phase L1 | `udbias_ppp()` : pour `f=0`, `ion=0`, donc `bias = L1 − P1` biaisé de `2·I1` (mètres) avec `VAR_BIAS = 60²` | correct grâce à la variance initiale, mais l'état STEC déjà initialisé permettrait une init cohérente |
| Pondération L5 | `EFACT_GPS_L5 = 10` sur GPS/QZS L5 en non-combiné | à revoir si L5 est ajouté (IFCB) |

### 11.2 Chantiers spécifiques `est-stec` + CNES

**SC-1 (P0). Contrainte VTEC continue.** À chaque époque, dans `ppp_res()`, une pseudo-observation par satellite sur `II(sat)` : `v = ion_vtec − x[II]`, `H = 1`, variance `σ² = (max(qi, 1 TECU) × 0.1624 m/TECU × mapping)²` bornée (plancher ≈ 0.2 m, plafond ≈ 1 m) et inflatée à basse élévation. Utiliser `qi` et `udint` (ne pas contraindre si l'âge du 1264 dépasse 2 × `udint`). Remplacer `VAR_SSR_VTEC` par cette variance pour l'initialisation. C'est P0-2 / RT-5 concrétisé avec la source CNES.

**SC-2 (P0). DCB récepteur par système, préalable à SC-1.** Un état constant (marche aléatoire très faible) par système, `H = 1` sur le code de la seconde fréquence (convention : biais nul sur le code de première fréquence, absorbé par l'horloge), init 0 avec variance (1 m)². Sans lui, la contrainte VTEC transfère le DCB récepteur dans la position. Généraliser `uddcb_ppp()`/`ID()` (aujourd'hui limité au L5).

**SC-3 (P0). ISB en marche aléatoire** (P0-4), indispensable en multi-GNSS non-combiné.

**SC-4 (P1). PCO satellite par fréquence** en mode non-combiné (`corr_meas()`, `satantpcv()`), cohérent avec le datum APC du CNES.

**SC-5 (P1). Init du biais de phase cohérente avec le STEC** : `bias_f = L_f − P_f + 2·C_f·x[II]` au lieu de `ion = 0` pour `f = 0`.

**SC-6 (P1). Bruit STEC adaptatif** : `prniono` fonction de l'élévation (déjà) et de l'activité mesurée (variance des innovations iono), plus grand en début de convergence.

**SC-7 (P1). GLONASS** : IFB code par satellite en état, sinon rester sans GLONASS.

**SC-8 (P2).** Journaliser `qi`, l'âge du 1264 et le nombre de STEC contraints dans le `.stat` ; vérifier avec le CNES le datum Galileo (I/NAV) et la liste des signaux portés par les biais.

### 11.3 Attente réaliste

Non-combiné sans contrainte (état actuel) : même convergence que l'iono-free, 15–30 min à froid. Avec SC-1 + SC-2 : la littérature et les résultats PPP-WIZARD donnent typiquement 5–10 min pour passer sous 10 cm horizontal et 10–20 min pour 5 cm 3D en float, la hauteur restant la composante lente. Avec RT-1 (warm start), la reprise est immédiate en statique et de quelques minutes en cinématique. Le 5 cm 3D « à chaque lancement » sans AR passe donc par la combinaison SC-1, SC-2 et RT-1, puis RT-2/RT-4 pour ne pas reperdre la convergence sur les trous de flux.

## 12. Démarrage à froid à chaque lancement : ce qui compte vraiment

Chaque lancement se fait ailleurs, dans d'autres conditions, sans état antérieur. La question n'est donc plus « comment reprendre une convergence » mais « comment rendre la convergence à froid rapide, monotone et sans plateau biaisé, à tous les coups ». Cela reclasse les chantiers.

### 12.1 Ce qui change dans les priorités

| Chantier | Avant | Cold start à chaque fois |
|---|---|---|
| RT-1 warm start | n° 1 | Réduit à une **calibration matérielle** : DCB récepteur par système (SC-2), ISB (SC-3), IFB GLONASS (SC-7) sont propres au récepteur, stables sur des jours, indépendants du lieu. Les sauvegarder et les recharger avec une variance modérée (≈ 0.3 m) enlève trois inconnues lentes de chaque convergence. Position, ZTD, STEC, ambiguïtés : jamais repris. |
| SC-1 contrainte VTEC + SC-2 DCB récepteur | P0 | **Le levier n° 1.** Sans contrainte iono, le float à froid met 15–30 min ; avec le VTEC CNES pondéré par sa qualité, 5–10 min sous 10 cm horizontal. |
| P0-4 / SC-3 ISB | P0 | Renforcé : à froid, chaque système supplémentaire ne « paie » que si son ISB n'est pas réinitialisé à chaque époque. |
| RT-2 IODE, RT-4 coupures | P0 | Inchangés : un trou pendant la convergence la remet à zéro. |
| Tropo (P1-1) | P1 | Monte en P0 : à froid, la hauteur est la composante lente, et l'a priori ZHD (atmosphère standard, `tropmodel()` à humidité nulle dans `trop_model_prec()`) plus NMF pèsent directement sur le temps de convergence en hauteur. GPT3 + GMF/VMF, `VAR_ZTD = 0.6²` réduit à ≈ 0.1² quand l'a priori est bon. |
| Charge océanique | config | `rtkrcv` ne lit **jamais** de BLQ (aucun `readblq()` dans `rtkrcv.c`/`rtksvr.c`) : `tidecorr` bit 2 est sans effet en temps réel, et un BLQ par site est impossible quand le lieu change. Ajouter un modèle global de charge océanique sur grille (FES2014b ou équivalent, 11 ondes, interpolation) embarqué dans `tides.c`. Erreur évitée : jusqu'à 3–5 cm en hauteur près des côtes, soit l'objectif entier. |
| Pondération / rejet (P1-2) | P1 | Monte en P0 : à froid, les premières minutes sont pilotées par le code ; un multitrajet code sur un satellite bas fixe une ambiguïté float fausse, et c'est le plateau à 20 cm. |

### 12.2 Anti-plateau : rendre la convergence monotone

1. **Élévation et SNR pendant la convergence** : pondération en `1/sin²(el)` renforcée (ou masque de 15° sur le code seul) tant que la variance position dépasse ≈ 0.5 m², puis retour à 10°. Les satellites bas apportent de la géométrie à la phase, pas au code.
2. **Test d'innovation normalisé** (`H P Hᵀ + R`) au lieu des seuils absolus `maxinno` : c'est la seule manière de rejeter 1–2 m de multitrajet code quand le filtre est encore à 3 m d'incertitude.
3. **Contrôle de cohérence des ambiguïtés** : pour chaque satellite, comparer en continu l'ambiguïté float à la moyenne glissante de `L − P` corrigée du STEC ; au-delà de 3σ sur 60 s, réinitialiser l'ambiguïté (variance `VAR_BIAS`) plutôt que de la laisser tirer la position. Aujourd'hui rien ne remet en cause une ambiguïté hors saut de cycle.
4. **STEC à froid** : init depuis le VTEC avec la variance de `qi`, contrainte continue (SC-1), bruit `prniono` plus grand les 5 premières minutes.
5. **Détecteur de convergence** (RT-8) : publier « converged » seulement si la variance filtre **et** un critère indépendant (stabilité de la position sur 60 s < 3 cm, âge médian des ambiguïtés > N min) sont satisfaits ; sinon le PE annonce 20 cm alors qu'il est à 20 cm.
6. **Sauts de cycle** (P1-4) : un saut non détecté à froid est un biais définitif ; MW glissant et GF dépendant de l'intervalle sont indispensables à 1 Hz.

### 12.3 Attente réaliste à froid, et place de l'IAR

Avec SC-1, SC-2, SC-3, la tropo et l'anti-plateau : convergence float monotone, 5–10 min sous 10 cm horizontal, 10–20 min pour 5 cm 3D, sans plateau biaisé. C'est la limite physique du float avec un VTEC global à quelques TECU : la composante hauteur et la séparation ambiguïté/ZTD ne se font que par le mouvement de la géométrie. Obtenir 5 cm 3D en moins de 5 min à froid, à tous les coups, n'est pas accessible en float. C'est précisément ce que les biais de phase du flux CNES permettent : WL fixée en 1–2 min, NL en 3–8 min, puis 2–3 cm. La stratégie cohérente avec « IAR en dernier recours » est donc : un float rendu fiable (sections 5, 10, 11, 12), et une AR **de confirmation** qui ne remplace le float que si le fix est cohérent avec lui (écart fix/float sous 3σ, ratio validé), et retombe sur le float sinon. Le float reste la solution de référence, l'AR ne fait qu'en raccourcir la fin.

## 13. σ annoncé 3 cm, erreur réelle 20 cm : rendre l'estimation d'erreur cohérente

Symptôme : le filtre annonce 3 cm (`sdn/sde/sdu` dans le `.pos`, issus de `sol.qr` = diagonale de `P`, `src/ppp.c:1166`), la réalité est à 10–20 cm sur E, N ou U selon la session. Ce n'est pas un défaut de réglage isolé : la covariance d'un EKF ne contient que ce qui a été déclaré dans `R` (bruit de mesure) et `Q` (bruit de processus), et elle suppose chaque époque indépendante. Toute erreur **constante ou lentement variable** qui n'est pas un état du filtre est invisible pour `P`, et la moyenne sur N époques la fait « disparaître » de `P` en 1/√N alors qu'elle reste entière dans la position.

### 13.1 Le mécanisme dans le code

| Source d'erreur corrélée dans le temps | Comment le code la traite | Effet sur σ |
|---|---|---|
| Orbite/horloge SSR : erreur de 3–5 cm par satellite, corrélée sur 5–30 min | `var_rs[i]` (URA SSR) ajouté à `var[nv]` **à chaque époque** comme bruit blanc (`src/ppp.c:1090`) | à 1 Hz, 600 époques en 10 min : la contribution à `P` est divisée par ≈ 24, l'erreur réelle ne bouge pas |
| Multitrajet phase (corrélé 1–10 min) et code (corrélé plusieurs minutes) | `varerr()` : `a + b/sin(el)` blanc, 3 mm phase, 0.3 m code | idem : `P` converge en quelques minutes vers des mm formels |
| Ambiguïté float biaisée par le code des premières minutes | état constant + `prnbias = 1e-4 m/√s` (`src/ppp.c:828`) ; rien ne la remet en cause hors saut de cycle | la position se cale sur l'ambiguïté fausse, `P` la croit bonne |
| Biais de code satellite manquant ou faux (signal non couvert), PCO fréquence, IODE, éclipse | erreur constante sur **un** satellite, absorbée par son ambiguïté et sa position projetée | biais de position dans la direction du satellite, tous les autres résidus semblent bons |
| ZHD a priori, mapping, charge océanique absente (`rtkrcv` sans BLQ) | pas d'état, pas de variance | biais hauteur pur, σ_U formel inchangé |
| Bruit de processus trop faible (`prntrop`, `prnpos=0` en statique) | filtre rigide | le filtre ne peut plus corriger un biais entré tôt |

Conséquence : la valeur 3 cm est une **précision formelle**, jamais une exactitude. Elle sera toujours optimiste tant que (a) le modèle n'est pas complet et (b) la corrélation temporelle des erreurs n'est pas représentée. Les sections 5, 10, 11 et 12 traitent (a) ; cette section traite (b) et les contrôles indépendants.

### 13.2 Rendre `P` honnête

**CV-1 (P0). Bruit de mesure à durée de corrélation.** À 1 Hz, une observation n'apporte pas une information nouvelle indépendante à chaque seconde. Deux implémentations équivalentes : n'utiliser que les époques à 10–30 s pour la mise à jour de covariance, ou, plus simple et sans perdre la solution 1 Hz, multiplier `R` par `max(1, τ/Δt)` avec `τ` = temps de corrélation (phase 30–60 s, code 60–120 s). Dans `varerr()` (`src/ppp.c:~360`), un facteur `opt->err[...]` supplémentaire suffit. C'est le correctif le plus efficace sur la cohérence σ/erreur ; il ralentit la décroissance formelle de `P` sans changer la position.

**CV-2 (P0). Erreurs orbite/horloge SSR comme processus corrélé, pas comme bruit blanc.** Soit un état de Gauss-Markov d'ordre 1 par satellite (τ ≈ 15 min, σ² = `var_urassr`) ajouté au résidu de phase et de code (`+MAXSAT` états), soit, à moindre coût, transférer cette incertitude dans le bruit de processus de l'ambiguïté (`prnbias` de 1e-4 à ≈ 1e-3 m/√s) pour que l'ambiguïté puisse suivre la dérive d'horloge SSR et que `P` la garde. À évaluer sur le rejeu (section 10.4) : l'objectif est un ratio erreur/σ proche de 1, pas un σ minimal.

**CV-3 (P0). Facteur de variance a posteriori et test NIS.** Après chaque `filter()`, calculer la statistique d'innovation normalisée `vᵀ (H P Hᵀ + R)⁻¹ v / nv` (la matrice est déjà formée dans `filter_()`, `src/rtkcmn.c`, il suffit de l'exposer) et son moyennage exponentiel sur 5 min. Si elle dépasse durablement 1, c'est que `R` ou `Q` sous-estiment la réalité : gonfler `R` (adaptatif) ou, a minima, publier `sol.qr × s²`. Ajouter `s²` et le NIS dans le `.stat`.

**CV-4 (P0). Contrôle de cohérence des ambiguïtés** (section 12.2, point 3) : c'est la seule défense contre le biais d'un seul satellite. Chaque ambiguïté est comparée en continu à `L − P` lissé (corrigé du STEC) ; un écart persistant > 3σ réinitialise l'ambiguïté avec sa variance initiale, ce qui **augmente honnêtement `P`** au lieu de laisser le biais.

**CV-5 (P1). Séparation de solutions (intégrité).** Calculer en parallèle, à faible coût, des sous-solutions (GPS seul, Galileo seul, ou « tous sauf un satellite » pour les N satellites les plus pondérés) et comparer à la solution complète. Un écart supérieur à ce que les covariances prédisent signale un biais non modélisé ; on gonfle σ et on retire le satellite responsable. Sans ce type de contrôle, aucun filtre ne peut détecter un biais cohérent avec ses propres résidus.

**CV-6 (P1). σ empirique calibré sur le temps depuis le démarrage.** À partir de la campagne de validation (sections 7 et 10.4), ajuster une courbe `σ_emp(t, composante)` (par exemple `a + b·exp(−t/τ)`) sur l'erreur réelle observée, par composante E/N/U, et publier `max(σ_formel × s, σ_emp(t))`. C'est la garantie pragmatique de cohérence tant que CV-1 à CV-5 ne sont pas déployés, et un garde-fou ensuite.

**CV-7 (P1). Détecteur de convergence** (RT-8, 12.2 point 5) : ne jamais publier « convergé » sur la seule diagonale de `P`. Conditions cumulatives : σ formel × s² sous seuil, NIS ≈ 1 sur 5 min, ≥ 6 satellites avec ambiguïté âgée de plus de N minutes, stabilité de la position sur 60 s < 3 cm, aucune réinitialisation d'ambiguïté récente.

### 13.3 Diagnostiquer vos sessions à 20 cm

Le fait que la composante fautive change d'une session à l'autre (E, N ou U) désigne une erreur **par satellite** projetée sur la géométrie du moment, plutôt qu'une erreur systématique de modèle (qui frapperait toujours U). Sur les logs rover + SSR rejoués (section 10.4), avec `out-outstat=residual` :

1. Tracer par satellite `$SAT` (résidus phase, `lock`, `slipc`, `rejc`) et `$ION` ; un satellite dont le résidu de phase est petit mais dont l'ambiguïté a été initialisée pendant un pic de résidu de code est le suspect habituel.
2. Retirer le satellite suspect (`pos1-exclsats`) et rejouer : si l'erreur disparaît, c'est un biais satellite (biais de code du signal, PCO, IODE, éclipse) ou une ambiguïté initialement biaisée.
3. Si l'erreur est toujours en U : ZHD/mapping, charge océanique (absente en temps réel), antenne (PCO/ARP) ; comparer `$TROP` à un ZTD de référence.
4. Comparer la même session en `est-stec` et en `dual-freq` : un écart de plusieurs cm signe un problème de biais de code ou de STEC, pas de géométrie.
5. Calculer sur toutes les sessions le ratio erreur/σ par composante : c'est la métrique à suivre ; l'objectif est une distribution proche de χ² (95 % des erreurs sous 2σ), pas un σ petit.

