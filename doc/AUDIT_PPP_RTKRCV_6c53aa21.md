# Audit PPP temps réel (rtkrcv) — RTKLIB demo5 b34, commit `6c53aa21` (2021-09-21)

**Périmètre** : cœur PPP et ses interfaces, chaîne temps réel rtkrcv/rtksvr, décodage RTCM3
(observations MSM + corrections SSR, compatibilité PPP-Wizard/CNES), décodeur SBF Septentrio
et remplacement de l'entrée RTCM3 par du SBF, optimisation CPU/latence, amélioration du PPP
float sans IAR (convergence, L1/L5, est-stec), PE « seamless » (single→PPP), architecture et
maintenabilité.

**Méthode** : audit statique du code au commit `6c53aa21`, chaque constat vérifié dans les
sources (référencées `fichier:ligne`). L'historique amont (`6c53aa21..origin/main`) a été
utilisé pour confirmer objectivement les bugs (75 correctifs postérieurs rien que sur
`rtcm3.c`/`ephemeris.c`/`septentrio.c`). Aucun test runtime : les points nécessitant une
validation sur données réelles sont signalés.

Convention de sévérité : **BLOQUANT** (fonctionnalité cassée pour le cas d'usage visé),
**CRITIQUE** (corruption/gel possible en production), **MAJEUR**, **MOYEN**, **MINEUR**.

---

## 1. Synthèse exécutive

Le constat central : **à ce commit, la chaîne « PPP temps réel avec corrections PPP-Wizard »
est structurellement incomplète**, indépendamment des bugs ponctuels :

1. **Les biais de phase RTCM3 1265-1270 (flux CNES CLK93/SSRA00CNE0) ne sont pas décodés du
   tout** — ils tombent dans le `default` du switch. `ssr->pbias` reste vide.
2. **Le VTEC 1264 n'est ni décodé ni exploitable** : aucun champ dans `nav_t`, aucun usage
   dans le moteur. Pas de contrainte ionosphérique SSR pour accélérer la convergence est-stec
   (toujours absent même dans demo5 récent).
3. **`ppp_ar()` est un stub qui retourne 0** (`ppp_ar.c:17-21`) : le PPP est float-only,
   tout le chemin FIX de `pppos` est du code mort.
4. **Le mode est-stec est cassé en L1/L5 pur** : l'état iono n'est initialisé que depuis les
   slots 0/1 ; un satellite sans L2/E5b est intégralement éliminé du PPP (phase et code).
5. **La chaîne temps réel a des défauts de robustesse sérieux** : I/O console telnet sous le
   verrou serveur global (gel du positionnement possible), débordement heap potentiel du
   buffer d'observations en multi-GNSS, époques jetées si le calcul dépasse le cycle de
   10 ms, SSR périmées appliquées sans contrôle de fraîcheur.
6. **Le décodeur SBF est utilisable mais bugué** : injection d'époques d'observation
   dupliquées dans le filtre toutes les ~12,5 min (retour `1` au lieu de `9` sur ion/UTC),
   signaux d'index ≥32 perdus (B2b, QZSS L1C/L1S). Trois patches d'une ligne corrigent
   l'essentiel ; le couple « stream1=SBF obs/eph + stream3=RTCM3 SSR » est déjà fonctionnel
   dans `rtksvr`.
7. **Coût CPU dominé par des choix évitables** : EKF avec inversion complète m×m et ~10
   mallocs par appel, états dimensionnés sur MAXSAT (nx≈630 → matrice P de 3,2 Mo copiée à
   chaque itération), `satposs` précis exécuté deux fois par époque, sélection d'éphémérides
   en scan linéaire.
8. **Pas de filet de sécurité** : la suite `test/utest` ne compile plus (référence
   `qzslex.o` supprimé), aucune CI, pas de golden files. C'est le préalable à tout le reste.

Le plan (§10) est organisé en 4 phases : filet de tests → correctifs bloquants/backports →
PPP-Wizard complet + SBF + convergence float → refactor PE seamless et performance.

---

## 2. Anomalies majeures — cœur PPP (`ppp.c`, `ephemeris.c`, interfaces)

### Bugs avérés

| Sév. | Localisation | Constat |
|---|---|---|
| MAJEUR | `ppp.c:1165-1168` | Boucle `for (i=0;i<MAXOBS;i++)` sans borne `i<n` : lecture hors de l'époque courante (post-traitement : époques suivantes ; temps réel : mémoire indéfinie) et écriture de `snr_rover` sur des satellites arbitraires → poids SNR corrompus dans `varerr()` (`ppp.c:1021-1023`) quand `weightmode=SNR`. Fix : `i<n&&i<MAXOBS` (comme `ppp.c:1118`). |
| MAJEUR | `ephemeris.c:798-801` | Repli horloge broadcast dans `satposs()` : `*var=SQR(STD_BRDCCLK)` écrase `var[0]` au lieu de `var[i]`. Le premier satellite de l'époque est dépondéré à tort, le satellite fautif garde une variance optimiste. Fix : `var[i]=...`. |
| MAJEUR | `ppp.c:411-421` | Alignement des biais de code SSR au datum d'horloge incohérent : `ix` n'est défini que pour GPS (L1W/L2W), GLO (L1P/L2P), GAL (**L1X/L7X** — faux pour CNES qui référence E1C/E5aQ) ; BDS/QZS/SBS gardent `ix=0` (slot C1C, sans rapport) ; pour i=2 (L5) c'est le biais **L2W** qui est soustrait à une mesure **L5**. Fix : table de codes de référence par (système, slot). |
| MAJEUR | `ppp.c:745-747` | En IFLC, `corr_meas` bascule sur le slot 2 si `L[1]==0` (`ppp.c:435-441`) mais `udbias_ppp` ne teste que `slip[0]||slip[1]` : un slip LLI sur L5 ne réinitialise pas le biais IF L1/L5. En prime la LC peut changer de définition par époque (L2 qui clignote) pour une ambiguïté unique → sauts non détectés. |
| MAJEUR | `ppp.c:459-506` | Détection de sauts de cycle GF/MW câblée sur les slots 0/1 uniquement : un satellite L1+L5 n'a aucune détection GF/MW (seul le LLI reste). Fix : décliner sur les paires (0,1) et (0,2). |
| MOYEN | `rtkcmn.c:2393-2399` | `readantex` : fréquences récepteur limitées à GPS ; satellites BDS : B1I rangé slot 1, B2I ne matche rien et écrase le slot précédent → PCO satellites BDS incohérents. |
| MINEUR | `ppp.c:1003,1006-1008` | `ppp_res` mélange `rtk->x` (gate iono, DCB L5) et l'état itéré `x` pour le calcul du résidu → valeurs périmées pendant les itérations. |
| MINEUR | `ppp.c:1054` | `rejc[maxfrq%2]++` : indexé par type (phase/code) au lieu de la fréquence (`maxfrq/2`). Compteurs de diagnostic faux. |
| MINEUR | `postpos.c:417` vs `rtksvr.c:636` | Logique d'option des biais de phase inversée entre post-traitement (`-ENA_FCB` = désactive !) et temps réel (`-DIS_FCB`). |
| MINEUR | `ppp.c:159-161` | `pppoutstat` imprime deux fois STD(i+2) pour les horloges ($CLK erroné). |

### Vérifié conforme (à ne pas « corriger »)

`satpos_ssr` (`ephemeris.c:604-699`) : signe orbite `rs -= R·deph`, `dts += dclk/c`,
cohérence IOD orbite/horloge (l.625), fenêtre `MAXAGESSR=90 s`, recentrage mi-intervalle
`udi`, matching IODE strict via `seleph(...,iode)`, gardes MAXECORSSR/MAXCCORSSR, CoM→APC
via `satantoff` seulement pour `EPHOPT_SSRCOM`. Attention réglage : **flux RTCM CNES =
APC ⇒ `EPHOPT_SSRAPC` ; IGS 4076 = CoM ⇒ `EPHOPT_SSRCOM`** — une erreur = biais radial
1-2 m, et aucune sélection automatique n'existe.

Limitation associée : `satantoff` (`preceph.c:595-614`) calcule le PCO en IF L1/L2 (GPS) ou
E1/E5b (GAL) — incohérence ~cm avec des produits alignés E1/E5a ou un traitement L1/L5.

---

## 3. Chaîne temps réel rtkrcv/rtksvr

### Bugs avérés

| Sév. | Localisation | Constat |
|---|---|---|
| CRITIQUE | `rtkrcv.c:1035-1041`, `rtkrcv.c:917-924` | `vt_printf` (write socket telnet bloquant, octet par octet, `vt.c:337-341`) appelé **sous `rtksvrlock`**. Un client telnet qui ne lit plus (fenêtre TCP nulle) gèle le thread serveur (bloqué dans `decoderaw`/`rtkpos`/`saveoutbuf`) → positionnement PPP arrêté. DoS trivial. Fix : copier sous verrou, imprimer verrou relâché (comme `prstatus`), socket en écriture non bloquante. |
| MAJEUR | `rtksvr.c:149` | `update_obs` écrit `svr->obs[index][iobs].data[n++]` **sans borner n à MAXOBS** (=96) alors que la source peut en contenir MAXOBS*2 : débordement heap silencieux en multi-GNSS tri-fréquence dense. Fix : `n<MAXOBS`. |
| MAJEUR | `rtksvr.c:653-656` | Si `rtkpos` dépasse `svr->cycle` (défaut 10 ms — courant en PPP multi-GNSS), les époques rover restantes du batch sont jetées (`prcout`) et les buffers remis à zéro : sous-échantillonnage silencieux des solutions. Fix : découpler la cadence de calcul du cycle de polling. |
| MAJEUR | `rtksvr.c:272-304` + `rtksvr.c:457-470` | SSR appliquées sans aucun contrôle de fraîcheur au moment de l'usage (seule la cohérence `iod[0]==iod[1]` est testée à la réception) : en cas de coupure du flux de corrections, des SSR périmées continuent de corriger silencieusement. Fix : garde `timediff(obs.time, ssr.t0)` avant application (`MAXAGESSR` existe côté `ephemeris.c` mais pas pour les biais). |
| MAJEUR | `rtksvr.c:627-634` | Synchronisation base : seule `svr->obs[1][0]` est consommée ; une rafale d'époques base dans un cycle (reconnexion NTRIP) désynchronise l'âge du différentiel. |
| MOYEN | `rtksvr.c:468` | `pbias[code-1]` sans garde `code>0` → lecture hors borne si code non renseigné. |
| MOYEN | `rtksvr.c:646-648` + `rtkcmn.c:1627-1630` | `timeset()` appelé à chaque solution accumule `timeoffset_` (statique globale non verrouillée, aussi écrite par `readfile`) → dérive de `timeget()`, horodatage NMEA base faux. |
| MOYEN | `rtksvr.c:399-431` | `decodefile` : fuite possible des tableaux de la `nav` locale hors peph/pclk. |
| FAIBLE | `stream.c:387-388` | Validation bitrate série : boucle `i<13` mais test `i>=14` → bitrate invalide accepté, lecture hors tableau sur macOS. Fix : `i>=13`. |
| FAIBLE | `rtksvr.c:988` | `rtksvrstop` : `pthread_join` sans timeout sous POSIX → un `restart` sur serveur gelé bloque la console. |

### Latence (architecture par polling)

- Boucle `rtksvrthread` (`rtksvr.c:585-676`) : `strread` non bloquant sur 3 streams +
  `sleepms(cycle-cputime)`, cycle défaut **10 ms** (`rtkrcv.c:103`). Une trame arrivée juste
  après le poll attend jusqu'à ~10 ms ; le calcul n'est déclenché qu'au cycle suivant la
  complétion d'époque. **Latence architecture : ~5-10 ms en moyenne, 20-25 ms pire cas**
  (hors temps de calcul), plus le drop d'époques ci-dessus.
- `send_nmea` forcé à ≥1 s (`rtksvr.c:851`) : amorçage VRS/NRTK lent.
- À vide : 3 `select()` + 6 lock/unlock + parsing `periodic_cmd` à 100 Hz → 100 wakeups/s
  (pénalisant en embarqué basse conso).

### Mémoire (embarqué)

`sizeof(rtksvr_t)` ≈ **5,3 Mo** mesuré avec les defines rtkrcv : 3×`rtcm_t` (845 Ko chacun,
contenant chacun un `nav_t` complet de 542 Ko avec `sbsion`/`sbssat`/`pcvs` inutilisés),
3×`raw_t` (659 Ko), `obs[3][MAXOBSBUF=128]` = 4,4 Mo alloués alors que base/corr n'utilisent
que l'indice [x][0]. La console copie 3×845 Ko à chaque affichage `ssr` (`rtkrcv.c:668`).

---

## 4. RTCM3 en entrée

### 4.1 Observations MSM

| Sév. | Localisation | Constat |
|---|---|---|
| MAJEUR | `rtcm3.c:113-118` | `msm_sig_cmp` sans B2a (5D/5P/5X), B1C (1D/1P/1X), B2b (7D) : observations BDS-3 modernes jetées (« unknown signal id »). Corrigé en demo5 2023 (`8e04ef65`). Idem GLONASS L3 CDMA/G1a/G2a (`rtcm3.c:89-94`, ajouté 2024) et NavIC. GPS/GAL/QZS/SBAS complets (L5/E5a/E5b/E6 présents). |
| MAJEUR | `rtcm3.c:1948-1985` + `rtklib.h:130,135` | `sigindex()` : avec NFREQ=3/NEXOBS=0 par défaut, tout code d'index ≥ NFREQ (E6, E5ab, B3) et tout 2e code d'une même bande sont jetés. Recompiler avec NEXOBS>0 pour ne rien perdre ; priorités forçables par options `-GLss/-ELss`. |
| MAJEUR | `rtcm3.c:2055-2083` | GLONASS MSM4/6 : sans FCN connue (avant réception d'un 1020), `fcn=-8` → phase et Doppler mis à 0, seul le code survit. |
| MOYEN | `rtcm3.c:2199-2203` | Message MSM tronqué : `return -1` sans propager le bit sync → époque multi-messages potentiellement flushée à tort ; pas de resynchronisation buffer après message invalide (corrigés plus tard : `8e04ef65`, `269c2d5e`, `7125b14d`). |
| MINEUR | `rtcm3.c:208-214,2090-2091` | `lossoflock()` compare les indicateurs de lock bruts sans normalisation 4 bits (MSM4/5) vs 10 bits (MSM6/7) : faux slips si le flux alterne les types MSM. |
| MINEUR | `rtcm.c:92` | `msmtype[7]` : seuls 6 éléments initialisés (NavIC non initialisé). |

CRC-24Q, bornes d'en-têtes SSR, `nsat*nsig≤64` : vérifiés corrects.

### 4.2 Corrections SSR — compatibilité PPP-Wizard/CNES

| Sév. | Localisation | Constat |
|---|---|---|
| CRITIQUE | `rtcm3.c:2677-2680` | **Messages phase bias 1265-1270 non décodés** : `decode_ssr7` existe mais n'est branché que sur des types fictifs 11-14 et les subtypes IGS 4076. Le flux CNES (1265 GPS, 1266 GLO, 1267 GAL, 1268 QZS, 1270 BDS) tombe dans le `default` → `pbias` jamais rempli, `corr_phase_bias` no-op, PPP-AR impossible. Fix = backport demo5 `ffc6aba1` (avril 2025) : mapping direct vers `decode_ssr7`, format compatible. |
| CRITIQUE | (absent) | **VTEC 1264 non décodé et sans support moteur** : pas de `case 1264`, pas de champ dans `nav_t`/`ssr_t` ; `IONOOPT_STEC` (`rtklib.h:397`) n'est branché nulle part ; `model_iono` (`ppp.c:888-913`) ignore le SSR iono. Toujours absent même dans demo5 récent → développement à faire (décodeur harmoniques sphériques + stockage + pseudo-mesure sur les états iono). |
| MAJEUR | `rtcm3.c:1935-1946` + `rtksvr.c:331-333` | `decode_ssr7` ne pose pas `ssr->update=1` et retourne 20 au lieu de `sync?0:10` → même décodés (via 4076), les pbias n'atteignent `svr->nav.ssr` qu'en piggy-back du prochain message orbite/horloge. |
| MAJEUR | `rtcm3.c:1552,1712,1373` | **SSR BDS inutilisable** : longueurs de champs 1258-1263 fausses (`ni=10,nj=24,offp=1` au lieu de `nj=8,offp=0`) et IODE incompatible (`decode_type1042` stocke l'AODE 5 bits alors que l'IOD SSR BDS = mod(toc/720, 240)) → matching `seleph` jamais satisfait. Fix = backport `f542de90`. |
| MAJEUR | `rtklib.h:789-806` + `rtcm3.c:1916-1918` | Discontinuity counter des phase biases (`sdc`) lu puis **jeté**, indicateurs integer/WL non stockés : impossible de réinitialiser une ambiguïté quand le CNES change de jeu de biais → sauts de N cycles non détectés. Yaw SSR stocké mais jamais utilisé (windup = yaw nominal). |
| MAJEUR | `rtksvr.c:283-299` | Une correction SSR reçue avant l'éphéméride au bon IODE est perdue (update remis à 0) → trous au changement d'IODE. Atténué par une entrée SBF (éphémérides complètes immédiates, cf. §5). |
| MOYEN | — | Patterns « invalid value » RTCM des champs SSR non testés au décodage (seulement bornés à l'usage). `nav.ssr[].t0` non réinitialisés au restart (fix ultérieur `28ad77c0`). |

**Signe des pbias** (`L -= pbias·freq/c`, convention Takasu) vs convention CNES : à valider
expérimentalement sur CLK93 avant tout usage AR — non tranchable par audit statique.

### 4.3 Ce qui manque pour un support PPP-Wizard complet (est-stec), priorisé

1. Décodage 1265-1270 (backport `ffc6aba1`) + fix `update=1`/`return sync?0:10` dans
   `decode_ssr7` + traitement `ret` dans rtksvr.
2. Décodage **et exploitation** du VTEC 1264 : contrainte/init des états iono en est-stec
   (n'existe nulle part, y compris demo5 actuel — développement original).
3. Un vrai moteur PPP-AR (`ppp_ar()` vide) exploitant pbias + indicateurs integer/WL +
   discontinuity counter (à ajouter à `ssr_t`).
4. SSR BDS : backport `f542de90`.
5. Application saine des biais : gardes d'âge/IOD sur pbias et cbias, référence horloge
   Galileo E1C/E5aQ dans `corr_meas`, gestion correcte du slot L5.
6. Réglage/automatisation APC (RTCM CNES) vs CoM (IGS 4076) + ANTEX chargé.
7. Signaux MSM manquants (backports `8e04ef65`/`e3df2214`) + build NEXOBS>0.
8. Robustesse parsing (sync bit MSM tronqué, resync buffer, init t0 au restart).

---

## 5. SBF Septentrio — remplacer l'entrée RTCM3

### 5.1 État du décodeur (`src/rcv/septentrio.c`, import b34 inchangé)

Blocs gérés : MeasEpoch 4027 (rev 1), MeasExtra (stub), GPSRawCA, GLORawCA, GALRawFNAV/INAV,
GEORawL1, BDSRaw, QZSRawL1CA, NavICRaw. **Absents** : Meas3 (4109-4111, format compact par
défaut), MeasEpochExtra (σP/σL), CNAV L2C/L5, B1C/B2a/B2b raw, blocs nav décodés, ReceiverSetup.

| Sév. | Localisation | Constat |
|---|---|---|
| HAUTE | `septentrio.c:356,394-397` | **Ion/UTC GPS/QZS retourne 1 (« observation ») au lieu de 9** : `rtksvr` déclenche alors `update_obs` avec le buffer d'obs périmé → **époques dupliquées injectées dans le filtre PPP toutes les ~12,5 min** (rafale page 18 sous-trames 4/5 sur tous les sats). Fix : `return 9`. |
| HAUTE | `septentrio.c:209,258` | Extension du numéro de signal fausse : `sig+=(info>>3)*32` au lieu de `sig=(info>>3)+32` → signaux d'index ≥32 jamais décodés : **BDS B2b, QZSS L1C/L1S, NavIC S**. |
| MOYENNE | `septentrio.c:396` | `memset(subfrm+id*30)` : off-by-one (efface la sous-trame suivante au lieu de celle consommée, `(id-1)*30`) → ion/UTC redécodé, amplifie le bug précédent. |
| MOYENNE | `septentrio.c:695-707` | BDS D2 page 102 : condition impossible (`id==1&&pgn==102` avec pgn sur 4 bits) — UTC BDS GEO jamais décodé. Attention en corrigeant : la branche écrit `subfrm+10*38` = octets 380-417 d'une ligne de **380 octets** (`rtklib.h:1194`) → débordement latent (toujours présent dans main), agrandir `subfrm` à ≥418. |
| BASSE | `septentrio.c:439-443` | Cast `gtime_t*` non aligné (UB sur ARM strict ; corrigé plus tard, `431ec5a8`). |
| BASSE | `septentrio.c:240-289` | Lock time type-1 (U2) et type-2 (U1) comparés dans la même échelle → faux LLI possibles au basculement maître/esclave. |
| BASSE | — | `nav.glo_fcn` jamais renseigné depuis l'ObsInfo → GLONASS muet tant que l'éphéméride GLO (~30 s) n'est pas décodée. |

Vérifié correct : endianness, CRC16, reconstruction phase 0,001 cycle / PR 1 mm / Doppler
0,0001 Hz, SNR 0,25 dB, half-cycle → LLI bit 1, valeurs do-not-use, mapping svid, deux jeux
d'éphémérides Galileo INAV/FNAV compatibles `seleph`.

### 5.2 SBF vs RTCM3 en entrée : bilan

**Gains** : éphémérides brutes complètes et immédiates (critique pour le matching IODE de
`update_ssr` au démarrage — atténue le constat §4.2 « correction perdue avant éphéméride ») ;
ion/UTC systèmes ; indépendance vis-à-vis de la config RTCM du récepteur (pas de MSM4 sans
Doppler/lock 4 bits/CN0 1 dB) ; résolutions natives et FCN GLONASS embarquée. Face à du MSM7
bien configuré le gain métrologique est marginal ; face à MSM4/5 il est net.

**Intégration rtkrcv** : `inpstr1-format=sbf` fonctionne (`rtkrcv.c:183,200-202`), routage
`input_raw` OK, éphémérides SBF → `svr->nav` OK, et **le mélange stream1=SBF + stream3=RTCM3
SSR est déjà fonctionnel** (`update_ssr` matche les IODE contre les éphémérides du stream 1,
les deux sets Galileo testés). Deux accrocs : les options réceptrices (`-GALINAV`, `-AUX1`,
`-EPHALL`…) sont **inaccessibles** (`ropts[]` codé en dur vide, `rtkrcv.c:400`) ; les conf
d'exemple documentent « 13:sbf » alors que l'enum réel est 12.

### 5.3 Plan de passage à l'entrée SBF

1. **P0 (config, zéro code)** : récepteur en `MeasEpoch` (pas Meas3 !) + MeasExtra +
   GPSRawCA/GLORawCA/GALRawINAV/BDSRaw/QZSRawL1CA/GEORawL1 à 1 Hz ; rtkrcv
   `inpstr1-format=sbf`, `inpstr3-format=rtcm3`, `misc-navmsgsel=0`.
2. **P0 (patch 3 lignes)** : `return 9` (:356), `(id-1)*30` (:396), `sig=(info>>3)+32`
   (:209/:258). Sans le premier, le filtre reçoit des époques dupliquées.
3. **P1** : exposer `inpstrX-rcvopt` dans rtkrcv → `ropts[]` ; options PPP
   `frequencies=l1+l2+l5` (E5a est en slot 2) ; cohérence codes ↔ biais SSR (C1C/C2W/C5Q GPS,
   C1C/C7Q/C5Q GAL).
4. **P2** : backport du décodeur demo5 main (Meas3, MeasExtra/σ récepteur → `stats-rcvstds`,
   blocs nav décodés, ReceiverSetup) — attention au débordement D2 (subfrm 380→418).
5. **P2 (validation)** : log simultané SBF + MSM7 du même récepteur → convbin → diff RINEX ;
   rejeu fichier SBF dans rtkrcv (vérif absence d'époques dupliquées) ; PPP statique 24 h
   SSR temps réel vs entrée RTCM3 ; monitor `ssr` pour le matching IODE.

---

## 6. PPP float sans IAR : convergence, premières époques, L1/L5

### 6.1 Verrous de convergence identifiés

| Sév. | Localisation | Constat |
|---|---|---|
| MAJEUR | `ppp.c:535-540` | Kinematic sans `dynamics` : position réinitialisée white-noise (VAR_POS=60²) à `rtk->sol.rr` (= sortie SPP) **à chaque époque**. Aucune mémoire de position : la convergence repose entièrement sur biais/iono/tropo. |
| MAJEUR | `rtkpos.c:2233-2239` + `ppp.c:945` | Le SPP est un point de défaillance unique : si `pntpos` échoue, l'époque PPP est perdue même filtre convergé ; un satellite rejeté du SPP (`ssat->vs`) est interdit de phase dans l'EKF. |
| MAJEUR | `ppp.c:86,666-688,903-906` | État iono libre : init P0−P1 brut (bruit code ×1,55) avec VAR_IONO=60², aucune contrainte externe (VTEC SSR/IONEX/Klobuchar) ; le DCB P1-P2 satellite n'est jamais appliqué dans `corr_meas` (`ppp.c:423-430` : seulement P1-C1/P2-C2) → absorbé par l'état iono. |
| MAJEUR | défaut `eratio=300` (`rtkcmn.c:213`) | σ_code ≈ 0,9 m (+ ×3² en IFLC, `ppp.c:351`) : quasi aucune information code au démarrage. ~100 est plus adapté au PPP. |
| MINEUR | `rtklib.h:1007` | `codesmooth` défini mais référencé nulle part dans le traitement : pas de lissage code. |
| MINEUR | `rtkcmn.c:1351` | `smoother()` ne combine que les solutions, pas les états : le F+B n'accélère pas la convergence du float. |
| MINEUR | `ppp.c:1037-1056` | Rejet post-fit 4σ par **satellite entier** (une mauvaise pseudodistance élimine aussi la phase du même satellite), une exclusion par itération (jusqu'à 8 refiltrages complets). |

Horloges/ISB réinitialisées white-noise chaque époque (`ppp.c:600-619`) : les ISB
GLO/GAL/BDS, pourtant stables, ne sont jamais filtrés dans le temps.

### 6.2 L1/L5 et est-stec

Mapping vérifié : `code2idx` (`rtkcmn.c:710-724`) place GPS L1/L2/L5 → slots 0/1/2, GAL
E1/E5b/E5a → 0/1/2. `opt->nf` est un **préfixe de slots** : L5 impose nf=3.

| Sév. | Localisation | Constat |
|---|---|---|
| BLOQUANT | `ppp.c:670-676` + `ppp.c:1002-1004` | **IONOOPT_EST cassé en L1+L5 pur** : `udiono_ppp` n'initialise l'état iono que depuis `P[0]`/`P[1]` (slots 0/1 en dur). Récepteur L1/L5 (GPS) ou E1/E5a (GAL sans E5b) : P[1]==0 → état iono = 0 → `ppp_res` fait `continue` pour **toutes** les observations du satellite. Le mode est-stec L1/L5 élimine donc les satellites concernés. Fix : init sur la première paire (0,f) disponible, f∈{1,2} (la formule est déjà générique en fréquence). |
| BLOQUANT | (conception) | Pas de configuration « L1+L5 » : aucun mécanisme pour sélectionner les slots {0,2} avec nf=2 ; il faut nf=3 avec un slot 1 potentiellement vide (et l'état ND activé, `ppp.c:110`). |
| MAJEUR | `ppp.c:341-342` | `EFACT_GPS_L5` ×10 appliqué **avant** la distinction phase/code : la phase L5 (l'observable la plus propre) est dépondérée ×10. Rédhibitoire en PPP L1/L5. Fix : facteur sur le code seulement. |
| MAJEUR | (récap) | Suppositions L1/L2 en dur : GF/MW slots 0/1 (`ppp.c:373-389,459-506`), DCB fichier limité à 3 entrées P1-P2/P1-C1/P2-C2 (`rtklib.h:837`, rien pour L5), référence SSR L2W pour i≥1 (`ppp.c:414`), `satantoff` IF L1-L2/E1-E5b (`preceph.c:595-614`), IFLC SPP slots 0/1 (`pntpos.c:123-151`). `seliflc()` (choix L1/L2 vs L1/L5, commit `640929b`) n'existe pas encore à ce commit. |

**Prérequis PPP est-stec L1/L5** : (1) fix init iono multi-paires + suppression du gate
`ppp.c:1003` ; (2) GF/MW (0,2) ; (3) EFACT_GPS_L5 sur code seul ; (4) biais SSR par slot
correct (§2) ; (5) PCO satellite cohérent avec le datum des produits ; (6) contrainte iono
externe (VTEC 1264, §4) pour les premières époques.

### 6.3 Qualité numérique du filtre

- `filter_` (`rtkcmn.c:1290-1308`) : update `P=(I−KH')P` **sans forme de Joseph ni
  resymétrisation** — l'asymétrie s'accumule sur des centaines d'états. Fix peu coûteux :
  `P=(P+Pᵀ)/2` après update, ou Joseph.
- Convention « x==0 ⇒ état inactif » (`rtkcmn.c:1317`) : piège systémique (un état
  légitimement nul est gelé ; `udiono` peut produire ion=0 si P0==P1).
- `ppp.c:991` (`SYS_IRN: k=4`) et `update_stat` supposent NSYS≥5 sans garde : une build à
  systèmes réduits ferait déborder IC() sur l'état tropo (non déclenché avec les makefiles
  standards, tout étant activé).

---

## 7. Optimisations CPU / latence du moteur

Dimensionnement mesuré (defines rtkrcv : tous systèmes, NFREQ=3) : MAXSAT=204, **nx=630**
états PPP (835 en tri-fréquence + ND), k≈110 états actifs, m≈120 mesures pour 30 satellites.

| Sév. | Localisation | Constat |
|---|---|---|
| MAJEUR | `rtkcmn.c:1290-1334` | `filter_` : 10 mallocs + 5 `matmul` + `matinv(m×m)` par appel ≈ 9 MFLOP, appelé 1-8×/époque (une réjection d'outlier = un refiltrage complet, `ppp.c:1044-1056`). R est **strictement diagonale** en PPP/SPP → une mise à jour séquentielle scalaire (m updates rang 1) supprime l'inversion et les mallocs : gain 3-10×. Garder la forme matricielle pour le RTK (R DD corrélée). |
| MAJEUR | `ppp.c:105-118,1185-1192` | États indexés par PRN sur MAXSAT : `Pp=zeros(630,630)` = calloc 3,2 Mo + `matcpy` 3,2 Mo à **chaque itération**, R rempli en O(nv²) pour une diagonale, ~30 malloc/free par époque. Indexation dense par satellite suivi (nx≈150) → P ÷18, + buffers persistants dans `rtk_t`. |
| MAJEUR | `pntpos.c:665` + `ppp.c:1174` | **`satposs` précis exécuté 2× par époque** (pntpos ne force que iono/tropo, pas sateph) : 2 interpolations Neville ordre 10 + `sunmoonpos` recalculé par satellite (`preceph.c:582` ; seul `eci2ecef` est mémoïsé). Cache (rs,dts) par époque + mémoïsation rsun : ~2× sur le poste éphémérides. |
| MOYEN | `ephemeris.c:422-459` | `seleph` : scan linéaire de `nav->n` (2 appels/sat/époque) — jusqu'à ~10⁶ comparaisons/époque sur gros fichiers nav. Index par satellite trié par toe + mémo du dernier hit. |
| MOYEN | `rtkcmn.c:1150-1166` | `matmul` naïf cache-hostile ; l'option `-DLAPACK` existe déjà (makefiles) : OpenBLAS = 2-5× sur l'algèbre restante. |
| MINEUR | `rtkcmn.c:3087-3099` | `trace` : arguments évalués même trace fermée (`time_str()`, `sqrt` par observation) + `fflush` par ligne. Macro paresseuse + build production sans `-DTRACE`. |
| MINEUR | makefiles | `-ansi` (pas de `restrict`), pas de `-march`/LTO ; NFREQ/systèmes surdimensionnés par défaut (voir indexation dense). |

**Top 5 gain/effort** : (1) update EKF séquentiel scalaire ; (2) indexation dense des états +
buffers persistants ; (3) suppression du double `satposs` + mémoïsation soleil/lune ;
(4) index d'éphémérides ; (5) build LAPACK/flags/trace paresseuse. Côté I/O : lecture
événementielle (`poll` bloquant sur les fd) + calcul déclenché à la complétion d'époque
(§3) supprime 5-10 ms de latence moyenne et les 100 wakeups/s.

---

## 8. PE seamless : single-point → PPP naturellement

### Obstacles dans le code actuel

1. `rtkpos.c:2233-2240` : le SPP (WLS `pntpos`) est un préalable dur — s'il échoue et
   `!dynamics`, retour 0 sans mise à jour du filtre.
2. `ppp.c:945` : la phase n'entre dans l'EKF que pour les satellites validés par le WLS code.
3. `ppp.c:536-539,607-618` : position et horloges du PPP re-priorisées chaque époque depuis
   la solution SPP — l'EKF n'est pas autonome.
4. `rtkpos.c:2260-2262` : si `pppos` ne produit rien, la solution single déjà calculée est
   jetée (sauf `outsingle`) — pas de dégradation gracieuse.
5. `pntpos.c:419,580-615` : le single ne passe jamais par `filter()` ; le Doppler n'entre
   jamais dans l'EKF (WLS vitesse séparé).
6. `ppp.c:888-913` : `model_iono` est un **switch exclusif** (modèle OU état estimé, jamais
   un état *contraint par* le modèle) — c'est le verrou conceptuel : le passage
   brdc→SSR change la config, pas les données. Idem `satpos_ssr` qui échoue sans SSR
   (`ephemeris.c:604`) au lieu de dégrader avec variance gonflée.
7. Réinitialisations dures (`ppp.c:660-665,731-735,546-552`) : toute transition de
   disponibilité se paie en reconvergence complète.

### Architecture cible

Un EKF unique, autonome, états pos/vel/clk+ISB/ZTD(+grad)/iono par satellite suivi/amb par
satellite×fréquence, alimenté par un vecteur de mesures à géométrie variable : codes
(toujours), Doppler (si dispo), phases (si dispo), **pseudo-mesures de contrainte**
(iono = modèle Klobuchar/IONEX/VTEC-SSR ± σ_modèle ; ZTD = Saastamoinen ± σ). Les
corrections SSR n'affectent que `satposs`, les biais et les σ : sans SSR le même filtre
produit un SPP filtré ; avec SSR il converge en PPP ; `pntpos` ne sert plus qu'à l'init/reset
et à l'intégrité. `sol.stat` déduit des mesures effectivement utilisées.

### Plan de refactor incrémental (risque croissant, chaque étape livrable)

1. **Découpler la survie du filtre du SPP** : continuer vers `pppos` si le filtre est
   initialisé même quand `pntpos` échoue ; remplacer le gate `ssat->vs` par des critères
   locaux (svh/elmin/satexclude déjà présents) ; conserver la solution single en repli.
   (rtkpos.c, ppp.c — risque faible, golden files inchangés sur les cas nominaux.)
2. **Iono/tropo contraints par pseudo-mesures** dans `ppp_res` (réutiliser `ionocorr`),
   l'init `udiono` devient un simple prior. Supprime la bascule de modèle. (+ c'est le
   véhicule naturel du VTEC 1264 du §4.)
3. **Doppler dans l'EKF** (porter `resdop` en lignes de mesure sur vel + dérive d'horloge ;
   `dynamics` par défaut en PPP).
4. **Autonomiser les priors** : pos/clk propagés depuis l'état filtré au lieu de
   `sol.rr`/`sol.dtr` ; contrôle de cohérence SPP↔filtre conservé comme garde-fou.
5. **Mode unique** : PMODE_SINGLE = profil du même pipeline (phases ignorées, iono
   contrainte serrée) ; fallback par satellite brdc↔SSR dans `satpos` avec variance gonflée.
   Derrière une option, activée par défaut après validation terrain.

---

## 9. Architecture & maintenabilité

| Sév. | Constat |
|---|---|
| MAJEUR | **Tests morts, pas de CI** : `test/utest/makefile` référence `qzslex.o` (source supprimée) → la suite ne compile pas ; aucun workflow CI ; les « tests » rnx2rtkp produisent des .pos sans comparaison de référence. |
| MAJEUR | Monolithes et API à nu : `rtkcmn.c` 4041 lignes (temps+matrices+trace+coords+iono/tropo+CRC), `rtklib.h` 1831 lignes / **323 fonctions EXPORT**, structures non opaques (`rtk_t` expose x/P bruts). |
| MAJEUR | Globals mutables : `postpos.c:58-84` (mono-instance), `fp_stat` (`rtkpos.c:99-101`), `eph_sel` (`ephemeris.c:109`), `timeoffset_`, cache statique `eci2ecef` (`rtkcmn.c:2216-2228`, « not thread-safe », dans le chemin de positionnement) → deux moteurs dans un process = data race. |
| MAJEUR | Dimensionnement compile-time (NFREQ/NEXOBS/ENAGLO…/MAXOBS) : tailles de structs et indices d'états figés à la compilation, builds ABI-incompatibles, activer un système coûte RAM/CPU partout. |
| MOYEN | Duplications : tables de signaux et structure des messages en double rtcm3/rtcm3e ; mapping signaux→obsd_t refait dans chacun des 12 drivers `src/rcv/`. |
| MOYEN | `mat()`/`imat()` → `fatalerr` → `exit()` sur OOM (`rtkcmn.c:970-987`) : un malloc raté tue le serveur entier. |

**Cible pragmatique (sans réécriture)** : (1) réparer utest + golden files rnx2rtkp
(single/DGPS/RTK/PPP, tolérances) + CI GitHub Actions — préalable à tout ; (2) scission de
`rtkcmn.c` en unités avec header interne, API publique réduite (~40 fonctions) documentée ;
(3) couche « obs provider » (RTCM3/SBF/UBX/RINEX derrière une interface epoch-read commune,
tables code↔signal unifiées) ; (4) extraction de l'EKF en module à contexte explicite (porte
le refactor §8 et supprime les statiques du chemin de calcul) ; (5) une seule config binaire,
dimensions effectives runtime via l'indexation dense ; (6) erreurs propagées au lieu
d'`exit()`.

---

## 10. Plan d'amélioration consolidé

### Phase 0 — Filet de sécurité (préalable, ~quelques jours)
- Réparer `test/utest`, créer des golden files rnx2rtkp (données de `test/data`), CI Linux gcc.
- Sans ce filet, aucune des phases suivantes n'est validable sereinement.

### Phase 1 — Correctifs à fort impact, faible risque (bugs avérés + backports ciblés)
*Fiabilise l'existant sans changer l'architecture.*
- Cœur PPP : `ppp.c:1165` (hors-bornes SNR), `ephemeris.c:801` (`var[i]`), gardes
  code>0/fraîcheur sur pbias/cbias, `-ENA_FCB`, symétrisation de P.
- Temps réel : I/O console hors verrou (CRITIQUE), borne `n<MAXOBS` dans `update_obs`,
  suppression du drop d'époques lié au cycle, garde d'âge SSR, `stream.c:388`.
- SBF : patch 3 lignes (return 9, memset, extension de signal) — débloque l'entrée SBF.
- RTCM3 : backports demo5 `ffc6aba1` (phase bias 1265-1270 + fix decode_ssr7),
  `f542de90` (SSR BDS), `8e04ef65` (MSM BDS-3 + sync bit), `28ad77c0` (t0 au restart),
  `269c2d5e`/`7125b14d` (robustesse parsing).

### Phase 2 — PPP-Wizard est-stec + entrée SBF + convergence float
- Est-stec multi-fréquences : init iono première paire disponible + suppression du gate
  `ppp.c:1003` ; GF/MW (0,2) ; EFACT_GPS_L5 sur code seul ; références de biais par
  (système, slot) ; option APC/CoM par source.
- VTEC 1264 : décodeur + stockage `nav_t` + pseudo-mesure sur les états iono (développement
  original, levier n°1 sur le temps de convergence).
- DCB P1-P2 appliqués dans `corr_meas` ; eratio PPP ~100 ; discontinuity counter stocké dans
  `ssr_t` et déclencheur de reset d'ambiguïté.
- SBF : `inpstrX-rcvopt` dans rtkrcv, puis backport du décodeur complet (Meas3, MeasExtra→σ) ;
  campagne de validation SBF vs MSM7 et PPP 24 h.

### Phase 3 — Performance CPU/latence
- Update EKF séquentiel scalaire (R diagonale) ; indexation dense des états (nx 630→~150) +
  buffers persistants ; suppression du double `satposs` + mémoïsation soleil/lune ; index
  d'éphémérides ; lecture streams événementielle + calcul déclenché par fin d'époque ;
  build LAPACK/flags ; allègement mémoire rtksvr (MAXOBSBUF, nav_t embarqués) pour l'embarqué.

### Phase 4 — PE seamless + maintenabilité de fond
- Refactor incrémental §8 (5 étapes) : découplage SPP/filtre → pseudo-mesures iono/tropo →
  Doppler dans l'EKF → priors autonomes → mode unique.
- Architecture §9 : extraction du module EKF, couche obs-provider (qui absorbe proprement
  SBF/RTCM3/UBX), scission rtkcmn, suppression des statiques du chemin de calcul.
- Optionnel selon besoin : moteur PPP-AR exploitant les pbias CNES (WL/NL ou ambiguïtés non
  différenciées) — dépend de la phase 2 (pbias + discontinuity counter + signe validé).

### Points à valider expérimentalement (non tranchables par audit statique)
- Signe des phase biases CLK93 vs convention `L -= pbias·freq/c`.
- Réglage APC/CoM effectif du flux utilisé.
- Gains réels de convergence de la contrainte VTEC (tuning des σ).
