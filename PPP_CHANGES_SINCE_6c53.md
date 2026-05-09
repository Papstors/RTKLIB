# PPP — Bugs corrigés et améliorations depuis le commit `6c53aa2`

> **Référence** : `6c53aa2` — *"Clean up b33→b34 merge for obs files with multiple codes for same freq."* (rtklibexplorer, 21 sept. 2021)
> **Tête analysée** : `28ad77c` (rtklibexplorer/RTKLIB, 1ᵉʳ mai 2026)
> **Périmètre** : tout ce qui touche PPP (statique, kinematic, fixed, AR), ses entrées (SP3, CLK, ANTEX, .BIA/.BSX, DCB, IONEX, SBAS-tropo, SSR), et ses interfaces (RTKPOST, RTKNAVI, RTKNAVI-Qt, RNX2RTKP, RTKRCV).

---

## 1. Bugs corrigés (impact algorithmique direct)

### 1.1 Filtre Kalman / Estimation
| Commit | Date | Description |
|---|---|---|
| `1ff4727` | 2024-12-15 | **CRITIQUE** — Conversion mètres ↔ secondes manquante lors de la mise à jour des biais d'horloge depuis les états Kalman. Si la position initiale ne converge pas, des valeurs corrompues sont utilisées. |
| `b99d11b` | 2023-09-02 | Bug dans `varerr()` pour les solutions PPP qui n'utilisent pas la combinaison iono-free. Utilise désormais E1/E2 ou E1/E5 dual-freq iono-free pour PPP. |
| `caefe57` | 2024-11-20 | `ppp_res` : le tableau `var[]` doit avoir de la place pour **toutes** les fréquences (overflow). |
| `1a66490` | 2022-07-20 | Bug d'accès hors-tableau (out-of-bounds) dans `ppp.c`. |
| `e150df2` | 2024-01-26 | PPP : évite d'appliquer un facteur slant ionosphérique deux fois. |
| `efe4e67` | 2022-07-22 | Bug de correction iono GLONASS dual-fréquence. |
| `8615182` | 2024-04-15 | `rtkpos udstate` : calcule la baseline pour la mise à jour des paramètres troposphériques. |
| `bf1a788` | 2024-04-15 | PPP `detslp_ll` : revert d'une mauvaise indexation dans `ssat[].slip[]`. |
| `eb9ee2a` | 2024-03-17 | `ppp_res` accepte les pointeurs `v`, `H`, `R` comme NULL. |

### 1.2 Biais de code (.BIA / .BSX / DCB / OSB)
| Commit | Date | Description |
|---|---|---|
| `d574080` | 2026-03-30 | **Refonte DCB/OSB pour PPP** — Corrige un bug de chargement OSB qui corrompt les données nav lorsqu'aucune éphéméride broadcast n'est fournie. Stocke directement les OSB dans la table (au lieu de calculer les DCB d'abord). Étendu à toutes les fréquences. Suppression des tables de biais récepteur jamais utilisées. |
| `9ada3b7` | 2026-05-01 | **Applique les biais de code en absolu** (et non en différentiel) pour SSR et solutions post-process. |
| `5b59da3` | 2026-05-01 | Certains fichiers `.BIA` laissent le champ SVN vide — parsing modifié pour gérer ce cas. |
| `590403b` | 2026-04-20 | `preceph` : corrige l'indexation système de `code_bias_ix`. |
| `a67d8a8` | 2026-04-27 | `code2bias` : protège contre `code == 0` et contre `code_ix` non trouvé. |
| `c0138bf` | 2023-08-16 | **Ajout du support des biais satellite dans les fichiers `.BIA` / `.BSX`**. |
| `bb6f5c5` | 2023-08-23 | Nettoyage du commit précédent pour le support des nouveaux fichiers de biais. |
| `620fa28` | 2023-09-08 | Overflow de tableau dans la table de biais de code. |
| `a4b7514` | 2023-09-26 | Problème dans la structure de biais de code. |
| `e4e0ff5` | 2024-01-19 | `readbiaf` : corrige position et largeur des champs numériques. |
| `a581be8` | 2024-01-27 | Initialise la table d'index de biais aussi à la lecture des fichiers DCB. |
| `677261d` | 2023-09-26 | RTKNAVI / RTKNAVI-Qt : corrige le bug introduit par les changements récents sur les biais de code. Remplace les valeurs hard-codées par des `#define`. |
| `ba4a1d8` | 2023-09-26 | Déclaration manquante de `k` dans `pppoutstat()`. |

### 1.3 SSR (corrections en temps réel)
| Commit | Date | Description |
|---|---|---|
| `28ad77c` | 2026-05-01 | **Initialise les timestamps des corrections SSR dans `rtksvrinit`**. Permet à RTKNAVI d'être reproductible en post-traitement de fichiers (avec reset de `rtksvr` au démarrage). |
| `f542de9` | 2026-05-01 | **Décodage SSR clock/orbit BeiDou opérationnel** — corrige les longueurs de champs erronées, dérive les IODE conformément à l'ICD BeiDou. |
| `d09ea8f` | 2025-04-12 | `rtcm3 decode_ssr3` : corrige une perte de précision (warning compilateur). |
| `401e177` | 2024-04-09 | `rtcm3e encode_ssr3 / encode_ssr7` : suppression d'une assignation inutile à `nsat`. |
| `4ee13e0` | 2025-04-24 | Correction de typo dans l'entrée `ephopt` "Broadcast+SSR APC". |

### 1.4 Éphémérides précises (SP3) et horloges (CLK / ANTEX)
| Commit | Date | Description |
|---|---|---|
| `2e1ddfa` | 2026-02-20 | **Supprime la dépendance à BRDC quand des produits précis sont utilisés.** |
| `397d998` | 2026-04-28 | `readsp3` : gère les lignes de commentaires arbitraires. |
| `9f08ea4` | 2026-02-24 | Simplifie le check de l'horloge broadcast dans `ephemeris.c`. |
| `bb6f5c5` | 2023-08-23 | (idem ci-dessus, biais) |
| `ebf532a` | 2024-11-22 | RINEX clk 3.04 : corrige l'offset vers le système satellite dans le header. |
| `d3dc227` | 2026-04-27 | RINEX header : corrige le parsing du système pour les fichiers d'horloges. |
| `bb7f7ac` | 2024-12-06 | `seph2clk` : corrige l'expansion récursive. |
| `52374d0` | 2023-09-27 | Corrige la sélection des éphémérides broadcast GLONASS dans la sortie FCN. |
| `63153ee` | 2023-09-28 | Corrige les filtres de fichiers pour éphémérides/horloges précises. |
| `423ecd2` | 2025-05-21 | Options de format d'entrée corrigées + ajout de `CLK`. |
| `d9d59de` | 2025-02-03 | `rtkrcv` : suppression d'un hack apparent du format de stream SP3. |
| `241eaf8` | 2023-09-26 | Ajout des terminaisons de fichiers pour les noms longs IGS. |

### 1.5 Combinaison iono-free
| Commit | Date | Description |
|---|---|---|
| `640929b` | 2026-04-28 | **Iono-free unifiée** — utilise les slots fréq 1+2 pour toutes les constellations si `freqs="L1+L2"`, sinon L1/L2 pour GLO et freq 1+3 pour les autres. |
| `0bba414` | 2026-04-27 | Corrige une mauvaise constante introduite dans le commit ci-dessus : `SYS_BDS` → `SYS_CMP`. |
| `ce99498` | 2022-03-16 | **Étend l'option iono-free à L1/L5** + maj du biais de phase initial via `code2freq`. |
| `6a6271e` | 2025-04-24 | Corrige l'activation de `IONOOPT_IFLC` / `IONOOPT_EST`. |
| `87e2161` | 2025-04-24 | Retire l'option non supportée "STEC model" du tableau `ionoopt`. |

### 1.6 Antennes / PCO / PCV
| Commit | Date | Description |
|---|---|---|
| `c7ffd86` | 2023-10-13 | Ajout de commentaires sur la gestion PCO/PCV. |
| `c7d0035` | 2024-03-04 | `pcvss`, `pcvsr` : commentaires de code corrigés pour ces variables. |
| `4706813` | 2024-04-23 | `pcv` type et code : parsing avec `setstr`. |
| `79a6eee` | 2024-06-03 | Qt GUI option saveClose : remise à zéro du `pcv0`. |
| `15445ce` | 2025-05-17 | `rtksvr` : sélection automatique d'antenne, position en mode statique. |
| `640ec88` | 2023-10-19 | Sauvegarde / restauration des antennes définies préalablement. |
| `ec8508c` | 2023-10-04 | Ajout d'info sur la position satellite et la gestion APC (Antenna Phase Center). |

### 1.7 Ionosphère / Troposphère / SBAS-tropo
| Commit | Date | Description |
|---|---|---|
| `558048a` | 2024-11-13 | `sbstropcorr` : cache thread-safe (corrige les races en multi-thread). |
| `ee6bf10` | 2024-02-06 | `ionppp` : prise en compte de la hauteur du récepteur. |
| `516cc98` / `47c8279` | 2024-07-01 | `ionppp` : la position du récepteur est en **mètres** (correction d'unité). |
| `6ecb173` | 2024-04-11 | `ionex readtec` : ré-init des valeurs du header pour chaque fichier. |
| `b21f49c` | 2024-04-09 | SBAS : évite des opérations float simple précision avec destinations double. |
| `a321f35` | 2024-08-08 | SBAS subtype 7 : protège le nombre de satellites. |
| `621cf6b` | 2024-04-13 | SBAS `decode_sbstype6` : évite une lecture hors-bornes depuis `iodf[]`. |
| `5675351` | 2025-02-28 | Option `tidecorr` : passe en bitmask (granularité fine sur les marées). |

### 1.8 Résolution d'ambiguïtés (RTK uniquement — PPP-AR ⚠️ non implémenté)

> ⚠️ **`src/ppp_ar.c` est un stub vide** depuis 2016 (commit Takasu *"1.1 delete codes"*). La fonction `ppp_ar()` retourne 0 inconditionnellement. Le mode `PMODE_PPP_FIXED` existe dans le parser et la GUI mais ne fait **rien de plus que `PPP_KINEMA`** côté résolution d'ambiguïté. Aucun commit depuis 6c53 ne l'a réimplémenté. Tous les commits ci-dessous concernent l'AR **RTK** (`rtkpos.c`/`lambda.c`), pas PPP.

| Commit | Date | Description (AR **RTK** — pas PPP) |
|---|---|---|
| `f1a7d2f` | 2024-07-03 | Bug en résolution d'ambiguïté instantanée (reset excessif du `lock count`). |
| `b49ce01` | 2021-11-04 | AR retries cassée si GLO désactivé mais `GLO_AR=fix-and-hold`. |
| `157919a` | 2025-07-22 | `rtkrcv` : ajout de `ppp-fixed` à la commande `mode` (mode reconnu, mais sans effet réel sur l'AR). |
| `4509b10` | 2025-04-22 | Label / tooltip de la combo-box "ambiguity resolution" (RTK). |
| `0aedab3` | 2025-04-09 | Renommage de la combo-box AR. |
| `cf3f876` | 2023-10-10 | Statement pré-compilo pour la sortie des ambiguïtés. |
| `413c4a5` | 2023-10-10 | Dimensions cohérentes pour l'init des biais. |

**Conséquence pratique** : ne pas attendre de fix d'ambiguïté en PPP avec ce fork. Pour PPP-AR fonctionnel, il faut un autre fork (PPP-Wizard, Net_Diff, ou rtklib-py qui a une implémentation partielle).

### 1.9 Cycle-slip / qualité observations
| Commit | Date | Description |
|---|---|---|
| `c535238` | 2021-12-15 | **Détection de cycle-slip par test Doppler** + revamp de la pondération obs (toute combinaison elev × SNR × stdev récepteur). |
| `1d1b772` | 2022-01-18 | Cycle-slip Doppler : retire les erreurs d'horloge, gère les taux différents base/rover. |
| `fa8e46c` | 2022-04-04 | Cycle-slip Doppler en filtre **backwards**. |
| `8037d29` | 2023-07-29 | Augmente la variance d'observation si le flag half-cycle est armé. |

### 1.10 Pré-fit residuals & rejet de phase
| Commit | Date | Description |
|---|---|---|
| `62af402` | 2026-02-20 | **PR #805** — Limite différenciée des pré-fit residuals à la première itération PPP (convergence). |
| `6a3d711` | 2026-02-24 | Révision de l'ajustement du seuil. |
| `a06e9c1` | 2026-02-20 | **PR #803** — Option `pos2-rejionno` *deprecated* → nouvelle option **`pos2-rejphase`**. |
| `e3db89f` | 2023-10-19 | Augmente le nombre de décimales pour les seuils d'innovation. |
| `434e50b` | 2023-10-19 | Commentaire indique l'ordre des seuils dans le tableau. |

### 1.11 Hors-bornes / robustesse parsing
| Commit | Date | Description |
|---|---|---|
| `bf31bd6` | 2024-04-14 | `scanf` : largeurs maximales de chaînes (anti-overflow). |
| `269c2d5` | 2025-04-11 | RTCM3 invalide : reprend après le préambule valide suivant (pas de perte de messages SSR). |

---

## 2. Améliorations fonctionnelles

### 2.1 Algorithmes PPP
- **Pré-fit residuals adaptatifs** (`62af402`, fév. 2026) — meilleure convergence en début de session.
- **PPP sans broadcast** (`2e1ddfa`, fév. 2026) — utilisable avec uniquement SP3+CLK.
- **Refonte OSB** (`d574080`, mar. 2026) — stockage direct, étendu à toutes les fréquences.
- **Iono-free configurable** (`640929b`, avr. 2026) — comportement uniforme inter-constellations.
- **Iono-free L1/L5** (`ce99498`, mar. 2022).
- **Vérification précis vs broadcast** simplifiée (`9f08ea4`, fév. 2026).
- **Corrections de marées en bitmask** (`5675351`, fév. 2025) — sélection fine.
- **Fonctions soleil/lune haute précision désactivées par défaut** (`62a2abf`, avr. 2025) — gain de vitesse important en RTKPOST, <1 mm de différence en PPP.

### 2.2 SSR / temps réel
- **Décodage SSR BeiDou** (`f542de9`).
- **Init timestamps SSR** (`28ad77c`) — RTKNAVI reproductible en post-traitement.
- **Reprise après corruption RTCM3** (`269c2d5`).

### 2.3 Performance
- **`matmul` cache-aware** (`fbb9ea2`, mar. 2024) + `dot2`/`dot3` spécialisés (`d96a5c7`).
- **`initx` reloopé et inliné** (`f91603c`, mar. 2024).
- **Parallel make** (`4b06263`, `c9a127f`, oct. 2023).
- **Optimisation RTKPOST** (`66a532e`, août 2024) — **>2× plus rapide** par flags compilo cohérents.

---

## 3. Interfaces CLI (rnx2rtkp / rtkrcv)

> Sections RTKPOST / RTKNAVI / RTKNAVI-Qt / Qt apps **retirées** : périmètre projet = ligne de commande uniquement.

### 3.1 RNX2RTKP (CLI post-process)
| Commit | Date | Description |
|---|---|---|
| `4b519f3` | 2023-05-24 | Option `-bl` baseline en CLI + skip des estimées initiales en standard precision (gain perf rnx2rtkp). |
| `6bb4ab2` | 2023-12-07 | **Bug option `-sys`** corrigé + maj manuel. |
| `c4514fe` | 2023-10-10 | Options CLI start / end time. |
| `d1ab09e` | 2023-10-10 | Fix du paramétrage du fichier de sortie en CLI. |
| `343c245` | 2023-10-10 | Ne vérifie le suffixe de sortie qu'en cas d'extension invalide. |
| `ca3befa` | 2026-04-25 | `rnx2rtkp` : par défaut, navsys inclut Galileo et BDS. |
| `72dca37` | 2026-04-25 | BDS ajouté aux systèmes par défaut. |
| `93fdbcd` | 2023-09-21 | gfortran lib dans makefile rnx2rtkp pour gcc (gain perf). |
| `423ecd2` | 2025-05-21 | Options de format d'entrée corrigées + ajout `CLK`. |

### 3.2 RTKRCV (console temps réel — y compris rejeu `.tag`)
| Commit | Date | Description |
|---|---|---|
| `15445ce` | 2025-05-17 | **Sélection auto antenne + position en mode statique.** |
| `569197d` | 2024-11-11 | `prstatus` rempli pour 5 à 7 fréquences — **nécessaire avec NFREQ=4** sinon `.stat` tronqué. |
| `9c9ec53` | 2024-07-31 | Rework des position options — vérifier compat avec config rcv existante. |
| `9f1f9ac` | 2024-08-06 | Support des receiver options dans rtkrcv. |
| `92046a8` | 2024-05-17 | Stats out streams respectent désormais `solopt.sstat` level. |
| `e854997` | 2025-02-03 | Commandes `mark` et `mode` (logs marqueurs). |
| `c2ace2f` | 2025-01-08 | Format unicore en option. |
| `493d80e` | 2024-06-16 | `--detach` from console. |
| `02ec964` | 2024-06-05 | `--version` flag. |
| `7c926a0` / `36e8924` | 2024-05 | Shell command execution → option de compilation (par défaut désactivée). |
| `4b519f3` | 2023-05-24 | Option pour lancer RTKRCV sans console. |
| `5d6794e` | 2023-04-17 | Fix erreur de sortie (Issue #139). |
| `d9d59de` | 2025-02-03 | Suppression d'un hack format de stream SP3. |
| `157919a` | 2025-07-22 | Mode `ppp-fixed` ajouté à `mode` (cosmétique — `ppp_ar.c` reste vide). |
| `59dd319` | 2025-02-03 | `prstatus` affiche le thread rtk server en hex. |

---

## 4. Recommandations pour migration depuis 6c53

| Priorité | Raison | Commit cible minimum |
|---|---|---|
| 🔴 Haute | Corruption silencieuse Kalman clock bias | `1ff4727` (déc. 2024) |
| 🔴 Haute | Bugs OSB/DCB (corruption nav) | `d574080` (mar. 2026) |
| 🟠 Moyenne | iono-free PPP cassé hors L1/L2 | `b99d11b` (sept. 2023) |
| 🟠 Moyenne | Cycle-slip Doppler (qualité solution PPP-K) | `c535238` (déc. 2021) |
| 🟠 Moyenne | Support .BIA / .BSX (produits IGS modernes) | `c0138bf` (août 2023) |
| 🟡 Confort | PPP sans broadcast + pre-fit adaptatif + `pos2-rejphase` | `2e1ddfa` … `a06e9c1` (fév. 2026) |
| 🟡 Confort | Iono-free unifiée multi-constell | `640929b` + `0bba414` (avr. 2026) |

**Cibles de version recommandées** :
- **b34l** (mai 2025) — dernier point stable b34, reçoit la plupart des corrections critiques sans la bascule DCB→OSB.
- **EX 2.5.0** (juil. 2025) — tag de transition demo5 → EX, base solide.
- **HEAD `28ad77c`** (mai 2026) — bénéfices complets PPP modernes (OSB absolus, BeiDou SSR, iono-free unifiée).

---

## 5. ⚠️ Dépendance résiduelle à GPS (non corrigée)

> **TL;DR** : Oui, la dépendance structurelle à GPS héritée de l'origine de RTKLIB **est toujours présente** sur `HEAD` (`28ad77c`). Elle a été partiellement atténuée mais pas levée. Si GPS manque (urban canyon, scénarios multi-constell sans GPS), le PPP peut être instable.

### 5.1 Architecture concernée (inchangée depuis 6c53)

L'horloge récepteur PPP reste référencée à GPS dans `src/ppp.c:1138-1141` :

```c
rtk->sol.dtr[0]=rtk->x[IC(0,opt)]/CLIGHT;                       /* GPS */
rtk->sol.dtr[1]=(rtk->x[IC(1,opt)]-rtk->x[IC(0,opt)])/CLIGHT;   /* GLO-GPS */
rtk->sol.dtr[2]=(rtk->x[IC(2,opt)]-rtk->x[IC(0,opt)])/CLIGHT;   /* GAL-GPS */
rtk->sol.dtr[3]=(rtk->x[IC(3,opt)]-rtk->x[IC(0,opt)])/CLIGHT;   /* BDS-GPS */
```

À chaque époque, `udclk_ppp` (`src/ppp.c:606-624`) ré-initialise les états horloge en **bruit blanc** depuis le SPP de `pntpos` :

```c
for (i=0;i<NSYS;i++) {
    if (rtk->opt.sateph==EPHOPT_PREC) {
        /* time of prec ephemeris is based gpst */
        /* neglect receiver inter-system bias  */
        dtr=rtk->sol.dtr[0];   // <-- tous les systèmes prennent dtr[0] (GPS)
    } else {
        dtr=i==0?rtk->sol.dtr[0]:rtk->sol.dtr[0]+rtk->sol.dtr[i];
    }
    initx(rtk,CLIGHT*dtr,VAR_CLK,IC(i,&rtk->opt));
}
```

### 5.2 Mécanisme de défaillance sans GPS

Dans `pntpos` (`src/pntpos.c:344-351`), si aucun satellite GPS/QZS n'est observé, `mask[0]=0` et la contrainte de rang force `x[3]=0`. L'horloge récepteur réelle est alors **absorbée dans les biais inter-systèmes** `x[4]/x[5]/x[6]/x[7]`.

Conséquence en cascade :
1. `sol.dtr[0] ≈ 0` (fictif).
2. `udclk_ppp` ré-initialise les états horloge à 0 (mode `EPHOPT_PREC`) ou à des valeurs incohérentes (mode broadcast).
3. **Innovations énormes** au premier filtrage → convergence dégradée, voire divergence en kinematic.

### 5.3 Ce qui a été partiellement atténué depuis 6c53

| Commit | Date | Effet |
|---|---|---|
| `351bf2d` | 2024-02 | `pntpos` : QZS a maintenant son propre offset via le define `QZSDT`. Plus poolé avec GPS. **Sans GPS mais avec QZS observé**, `mask[0]` reste à 1 → `dtr[0]` cohérent. |
| `78a5892` | 2024 | Allocations corrigées quand `QZSDT` est activé. |
| `fc5075e` | 2024 | `pntpos rescode()` : correction du calcul de variance par fréquence. |
| `1ff4727` | 2024-12 | Unités m↔s corrigées sur le clock bias Kalman — n'élimine pas la dépendance, mais évite la corruption silencieuse des états horloge si la position initiale ne converge pas. |
| `b4b8d83` | 2025-05 | `rtkpos` : test de position initiale assoupli (norme < ½ rayon Terre suffit comme "init"). |
| `2e1ddfa` | 2026-02 | PPP **sans broadcast** quand des produits précis sont fournis — réduit certaines dépendances aux nav GPS, mais pas la référence horloge. |

### 5.4 Ce qui reste à corriger

1. **Mode `EPHOPT_PREC`** : le commentaire `/* neglect receiver inter-system bias */` est **explicite** — les ISB du récepteur sont volontairement ignorés, tout est calé sur `dtr[0]`. Sans GPS, instable.
2. **Pas de sélection dynamique** d'un système de référence alternatif si GPS manque.
3. **`FREQL1`** reste la fréquence de référence pour les facteurs de scaling iono dans `pntpos.c:323`, `rtkpos.c:1297-1298`, `ppp.c:684`/`995`. Mais `FREQL1` (1.57542 GHz) est partagée avec Galileo E1 et BeiDou B1C → **ce n'est pas une dépendance à recevoir GPS**, juste une constante de ratio. ✅ Cette partie est OK.

### 5.5 Recommandations pratiques

- **PPP sans GPS garanti** : préférer `EPHOPT_BRDC` ou `EPHOPT_BRDC+SSR` à `EPHOPT_PREC`. La branche `else` de `udclk_ppp` est plus robuste car elle utilise `dtr[0]+dtr[i]` qui se compense partiellement même si `dtr[0]=0`.
- **PPP urban canyon** : prévoir une logique applicative qui rejette les époques sans GPS pendant la convergence initiale (~10-30 premières époques), ou ré-initialise le filtre.
- **Activer `QZSDT`** (déjà actif sur `HEAD`) si tu opères en zone Asie-Pacifique : QZSS peut alors servir de référence quand GPS dégrade.
- **Surveiller** la trace `udclk_ppp` (level 3) et les innovations initiales : un saut > 1 ms entre époques avec et sans GPS révèle le problème.

### 5.6 Patch potentiel (à proposer en PR upstream)

Une vraie correction nécessiterait de réécrire `udclk_ppp` pour **élire dynamiquement** un système de référence (celui avec le plus d'observations sur l'époque), et de propager cohérement les états horloge entre époques au lieu de les ré-initialiser en bruit blanc depuis SPP. Cela impliquerait aussi :
- modifier `pntpos` pour que `dtr[0]` reflète l'horloge absolue du récepteur, peu importe le système de référence interne ;
- ou, plus simple, inverser le rôle quand GPS est absent : utiliser le premier système observé comme ancrage et stocker les `dtr[i]` comme offsets relatifs à lui.

Aucune PR ouverte sur `rtklibexplorer/RTKLIB` n'aborde ce sujet à date d'analyse (mai 2026).

---

## 6. Pistes de correction supplémentaires (audit complémentaire)

> Issues non encore traitées en upstream, identifiées par audit ciblé du code à `HEAD` (`28ad77c`). 57 problèmes recensés, regroupés par thème. Méthodologie : 4 agents d'analyse en parallèle sur (a) cœur PPP, (b) interfaces, (c) parsers produits externes, (d) SSR/temps réel/multi-fréq.

### 6.1 🟠 Théme transversal n°1 — `NFREQ>3` partiellement appliqué

Le passage à 4+ fréquences (`199be2b`, avr. 2025) n'a pas propagé partout. Plusieurs structures et boucles restent hardcodées à 3.

> **CORRECTION post-relecture** : `ssr_t.deph[3]` / `ddeph[3]` / `dclk[3]` représentent les composantes radial/along/cross de l'orbite et le polynôme c0/c1/c2 de l'horloge — **PAS des indices de fréquence**. Les corrections SSR1 (orbite) et SSR2 (horloge) sont **scalaires par satellite** et s'appliquent à toutes les fréquences. Les biais SSR3 (code) et SSR7 (phase) sont par signal-code via `cbias[MAXCODE]` / `pbias[MAXCODE]`, couvrant tous les signaux Mosaic X5. **Donc NFREQ>3 ne dégrade PAS la couverture SSR**, contrairement à ce qui était indiqué dans une version antérieure de cette section.

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🟠 | `src/rtkcmn.c:3744` (`seliflc`) | Retourne au max 2 — la 4ᵉ-6ᵉ fréq n'est jamais sélectionnée pour iono-free → uniquement freq 0+f2 où f2 ≤ 2 |
| 🟠 | `src/ppp.c:754` | Cycle-slip iono-free hardcodé à `slip[0] \|\| slip[1]` — slips sur freq 2-5 invisibles pour les biais iono-free |
| 🟠 | `src/ppp.c:385-406` | `gfmeas` / `mwmeas` n'utilisent que `code[0]`/`code[1]` → cycle-slip GF/MW jamais détecté sur freq 2-5 (LLI seul reste) |
| 🟠 | `src/ppp.c:368-369` | `obs->Pstd[frq]` / `Lstd[frq]` accédé sans guard `frq < NFREQ` → OOB potentiel si récepteur fournit > NFREQ stdevs |
| 🟠 | `src/ppp.c:1162` | `test_hold_amb` ne teste que `fix[0]` et `fix[1]` → fix-and-hold sur freq 2-5 jamais validé (pertinent quand PPP-AR sera en place) |
| 🟡 | `src/ppp.c:372` | Facteur iono-free `SQR(3.0)` valide pour GPS L1/L2 uniquement |

### 6.2 🔴 Théme transversal n°2 — Variances et facteurs hardcodés GPS-only

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🔴 | `src/ppp.c:372` | `var *= (ionoopt==IONOOPT_IFLC)?SQR(3.0):1.0` — facteur 3.0 valide uniquement pour GPS L1/L2. Pour Galileo E1/E5, BeiDou B1C/B2a, GLO L1/L4, le facteur correct dépend du ratio de fréq réel. Commentaire FIXME présent dans le code. |
| 🔴 | `src/ppp.c:355-357` | Seul `EFACT_GPS_L5` existe. Galileo E5a, BeiDou B2a, IRNSS L5 n'ont aucun facteur d'erreur fréq → variance L5 sous-estimée pour non-GPS. |
| 🟠 | `src/ppp.c:695` | Process noise iono `prn[1]` uniforme tous systèmes — Galileo/BeiDou ont des dispersions iono différentes. |
| 🟡 | `src/ppp.c:85` | `VAR_BIAS=SQR(60.0)` identique toutes fréqs — trop pessimiste pour L5/E5a (λ≈24cm). |

### 6.3 🟠 Robustesse parsing produits externes

**Pattern récurrent** : `satid2no(prn)` retourne 0 pour PRN invalide, mais le code accède à `nav->X[sat-1]` sans guard.

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🔴 | `src/preceph.c:486,494,502,504` | `readbiaf` : pas de check `sat > 0` → accès `cbias[-1]`. |
| 🔴 | `src/preceph.c:453-458` | `code2bias` : check `sat ≤ MAXSAT` mais pas `sat > 0`. |
| 🔴 | `src/preceph.c:454` | `code_bias_ix[sys_ix][code]` indexé sans check `code < MAXCODE` → OOB si nouveau code RTCM v3.3+. |
| 🟠 | `src/preceph.c:221,240,244,249,252` | `readsp3b` : `peph.pos[sat-1]` sans guard. |
| 🟠 | `src/rinex.c:1584,1608-1609` | `readrnxclk` : `nav->pclk[].clk[sat-1]` sans guard. |
| 🟠 | `src/preceph.c:597,603,615,626,627,676,677` | `pephclk` / `pephpos` accèdent `[sat-1]` sans guard. |
| 🟠 | `src/preceph.c:134-139` | SP3-d > 85 sats : `sats[]` peut déborder `MAXSAT`. |
| 🟠 | `src/preceph.c:479-485` | `readbiaf` : check `strlen < 91` mais accès `buff[70..90]` → cas limite à 90 caractères. |
| 🟡 | `src/ionex.c:126-171` | `readionexh` ne réinitialise que `ver` entre fichiers — `lats/lons/hgts/nexp/rb` persistent → interpolation TEC sur mauvaise grille en multi-fichiers. |
| 🟡 | `src/preceph.c:99-100` | `code_bias_ix` BeiDou : seulement `CODE_L2I`, `CODE_L6I` — manque L8T, L6D, L5D modernes. |
| 🟡 | `src/preceph.c:719` | `satantoff` → `searchpcv` sans guard `sat > 0`. |

### 6.4 🟠 SSR / temps réel — gaps de couverture

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🔴 | `src/rtcm3.c:2604-2618` | `decode_type4076` (IGS-SSR) : subtypes IRNSS **141-147 absents** du switch — messages IRNSS-SSR silencieusement ignorés. |
| 🔴 | `src/rtcm3.c` | **SSR-IM201** (standard IGS officiel à venir) non supporté — incompatibilité future avec NTRIP migrant. |
| 🔴 | `src/rtksvr.c:318-324` | Aucun test `ssr_time ≤ current_time` — corrections SSR avec timestamp futur appliquées telles quelles → divergence. |
| 🟠 | `src/rtksvr.c:330-340` | IODE matching SSR↔broadcast : seulement GPS/GAL/QZS/GLO. BeiDou/SBAS/IRNSS non couverts → corrections appliquées avec IODE désynchronisé. |
| 🟠 | `src/rtcm3.c:2015-2019` | Pas de flag `pbias_valid` dans `ssr_t` après décodage SSR7 → biais nuls appliqués comme si intentionnels. |
| 🟠 | RTCM3 SSR header | Pas de détection rollover/reboot encodeur SSR (saut IOD) → fausse détection nouvelle éphéméride. |
| 🟡 | `src/rtksvr.c:836-837` | `t0[]` initialisé à `time0` au lieu de `{0}` (déjà partiellement résolu par `28ad77c` mais incomplet). |
| 🟡 | `src/stream.c:111,1667,1724,1962` | NTRIP : pas de watchdog par message → SSR fragmenté/bloqué donne corrections stales >10s. |
| 🟡 | `src/rtkcmn.c:722-735` | `code2idx` peut retourner -1 pour codes BeiDou B1C/B2a/B3I, Galileo E6, IRNSS récents si tables `code2freq_*` incomplètes → biais ignorés silencieusement. |

### 6.5 🟠 Cohérence interfaces CLI

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🔴 | `app/consapp/rnx2rtkp/rnx2rtkp.c:51-52` | Help text ne documente pas les modes 7/8/9 (`ppp-kinematic`, `ppp-static`, `ppp-fixed`) bien qu'ils fonctionnent via `-p`. |
| 🟠 | `app/consapp/rnx2rtkp/rnx2rtkp.c:140-162` | CLI : pas de flags pour `pos2-gloarmode`, `pos2-bdsarmode`, `pos2-arthres1..4`, `pos2-armaxiter`, `pos2-varholdamb` → AR multi-constell uniquement via `-k config`. *(Note : AR PPP non opérationnelle de toute façon — cf. §1.8)* |
| 🟠 | Structure `solopt_t` + `outprcopts` | Patches `35b6fee` et `569197d` modifient le format `.stat` (ajout fréquences/iono pour PPP, prstatus 5-7 freqs). Vos parsers `.stat` doivent être validés. |
| 🟡 | `app/consapp/rnx2rtkp/rnx2rtkp.c:46` | Pas de `-K outfile` pour sauvegarder la session → édition config manuelle obligatoire. |

> Sections "Cohérence GUI" (PPPOpts non persistés, `pos2-rejionno` legacy, `tidecorr` bitmask GUI…) **retirées** : hors périmètre projet.

### 6.6 ⚠️ Reproductibilité du rejeu `.tag` — bug `cmd_restart` rtkrcv

**Symptôme** : rejouer 2× le même fichier `.tag` via la commande `restart` de `rtkrcv` produit des solutions différentes. Cause : `cmd_restart` (`app/consapp/rtkrcv/rtkrcv.c:1078-1085`) appelle `stopsvr` puis `startsvr` **sans** `rtksvrinit` entre les deux → la structure `svr` (`nav`, `ssr`, `rtk`, état Kalman, historique observations) persiste entre les runs.

**Statut upstream** : aucun fix dans rtkrcv. Le commit `28ad77c` (mai 2026) ne corrige que RTKNAVI (Embarcadero), pas rtkrcv. Le commit `68a355b` (Jens Reimann, mai 2024) corrige une **inversion `getFilePath()`/`setFilePath()` dans rtknavi-qt** (probablement le souvenir du bug d'inversion) — mais c'est aussi côté UI Qt, pas rtkrcv.

**Workaround actuel** : quitter et relancer le binaire `rtkrcv` à chaque rejeu (vs utiliser `restart` interactif).

**Fix recommandé** : patch maison de 3 lignes dans `cmd_restart`, détaillé en **§7.7**. Inclus dans le sprint comme patch #10.

**Validation** : `md5sum run1.pos == md5sum run2.pos` après deux exécutions consécutives sur le même `.tag`.

### 6.7 Autres effets de bord 🟡

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🟠 | `src/ppp.c:1061` | `THRES_REJECT=4σ` identique pour phase et code — phase residuals plus sensibles aux ambigs mal initialisées → trop de rejets en early epochs. |
| 🟠 | `src/ppp.c:118` (`IB` macro) + `src/ppp.c:741,746,783,790` | Si `nf` change entre sessions (ex: passage `IFLC→EST` + `nf 2→3`), `rtk->ssat[].fix[]` n'est pas redimensionné → indices `ambc` invalides. |
| 🟡 | `src/ppp.c:819-821` | DCB L5 estimé inconditionnellement si `nf ≥ 3`, même en `IONOOPT_IFLC` (1 fréq effective) → degré de liberté Kalman gaspillé. |
| 🟡 | `src/rtklib.h:162` | Si `NEXOBS=0`, observations indice ≥ NFREQ filtrées dans `save_msm_obs` → L5/E6 perdus silencieusement. |

### 6.8 Note sur la dépendance GPS (réf. section 5)

L'audit confirme la dépendance documentée en section 5 (`src/ppp.c:618-622`) et propose un **pattern de fix** validé par revue indépendante :

```c
int ref_sys = -1;
for (int i=0; i<NSYS; i++) {
    if (rtk->sol.dtr[i] != 0.0) { ref_sys = i; break; }
}
if (ref_sys < 0) ref_sys = 0;  /* fallback GPS */
dtr = (i == ref_sys)
      ? rtk->sol.dtr[i]
      : (rtk->sol.dtr[i] - rtk->sol.dtr[ref_sys]);
```

À ouvrir en PR upstream après validation jeux de tests.

### 6.9 Synthèse priorisée

**Top 5 fixes à viser immédiatement** :
1. 🔴 `ssr_t.deph[3]` / `dclk[3]` → `[NFREQ]` (`rtklib.h:834-836`)
2. 🔴 Guards `sat > 0` dans `preceph.c` (≥10 occurrences)
3. 🔴 Persistence `PPPOpts` dans INI RTKPOST/RTKNAVI Windows
4. 🔴 `seliflc()` cas `nf≥4` (`rtkcmn.c:3744`)
5. 🔴 Facteur iono-free dynamique au lieu de `SQR(3.0)` (`ppp.c:372`)

**Bilan global** : 7 issues 🔴 critiques (corruption mémoire, perte données utilisateur, biais algorithmiques majeurs), 24 🟠 moyennes (dégradations qualité solution / UX), 18 🟡 mineures.

---

## 7. Sprint 2 semaines — cherry-pick sur 6c53

> Stratégie minimaliste : on reste sur 6c53 (testé en prod), on n'apporte que des patches isolés, individuellement défendables, à risque de régression bas. Les fixes plus larges (`.BIA`, OSB, `NFREQ=4`, cycle-slip Doppler, refactor DCB/OSB) sont écartés et reportés sur un sprint 2 ultérieur.

### 7.1 Liste des 9 patches retenus

| # | Prio | Commit / Source | Fichier(s) | LOC | Ce qu'on corrige | Justification équipe |
|---|---|---|---|---|---|---|
| 1 | 🔴 | `1ff4727` | `rtkpos.c` | ~3 | Conversion m↔s manquante sur Kalman clock bias states | *"On corrige une unité oubliée — corruption silencieuse si l'init position ne converge pas."* |
| 2 | 🔴 | `b99d11b` | `ppp.c` | ~10 | `varerr()` PPP iono-free pour systèmes non L1/L2 | *"Sans ça, variance fausse pour Galileo E1/E5 → divergence en multi-constell."* |
| 3 | 🔴 | `1a66490` | `ppp.c` | ~5 | Out-of-bounds dans `ppp.c` (PR M. Valgur, mergée upstream) | *"Bug mémoire détecté par contrib externe, fix accepté en upstream."* |
| 4 | 🟠 | `efe4e67` | `rtkpos.c` | ~5 | Correction iono GLONASS dual-fréq | *"Erreur d'unité dans le calcul iono GLONASS."* |
| 5 | 🟠 | `b49ce01` | `rtkpos.c` | ~10 | AR cassée si GLO=off + `GLO_AR=fix-and-hold` (RTK) | *"Configuration GLO_AR ignorée silencieusement."* |
| 6 | 🟠 | `5cfa31b` | `rinex.c` | ~3 | Roundoff timestamps RINEX (entrées 60s parasites avec `-TADJ`) | *"Évite des duplications d'époques en sortie RINEX."* |
| 7 | 🟠 | `f1a7d2f` | `rtkpos.c` | ~5 | AR instantanée — reset excessif lock count (RTK) | *"Optimisation conservative de la logique de reset."* |
| 8 | 🟠 | `269c2d5` | `rtcm3.c` | ~15 | Resync RTCM3 après message invalide | *"Évite la perte de messages SSR/RTCM en temps réel."* |
| **9** | 🟠 | `6bb4ab2` | `rnx2rtkp.c` + `manual.docx` | ~10 | **Bug option `-sys` CLI rnx2rtkp** | *"Combinaison de constellations CLI mal interprétée — bug visible en post-process."* |
| **10** | 🟠 | **patch maison** (cf. §7.7) | `app/consapp/rtkrcv/rtkrcv.c` | ~3 | **Reproductibilité `restart` rtkrcv** : ajouter `rtksvrinit(&svr)` dans `cmd_restart` entre `stopsvr` et `startsvr` | *"Garantit qu'un rejeu via `restart` produit le même résultat — critique pour notre validation."* |
| **11** | 🟠 | `569197d` | `rtkrcv.c` | ~10 | `prstatus` rempli pour 5 à 7 fréquences | *"Nécessaire avec `NFREQ=4` : sans ça notre `.stat` parser voit des champs vides ou tronqués."* |
| **12** | 🟡 | `9c9ec53` | `options.c` + apps | ~50 | Rework des position options | *"Cohérence config rcv. Vérifier que nos `.conf` chargent toujours sans warning."* |
| **13** | 🟢 | **flags compil CM5** | Makefile | 0 | `-DNFREQ=5 -mcpu=cortex-a76 -O3 -flto -ffast-math -DSUNPOS_ORIG -DMOONPOS_ORIG -DTRACE=2` (cf. §10.6) | *"NFREQ=5 capture B1C BDS-3 + E6 Galileo — étape vers NFREQ=6 en sprint 2. Flags Cortex-A76 ciblent le CM5 production."* |
| **14** | 🔴 | **cluster Septentrio** : `670690a` + `784056a` + `253bfb3` + `26ed828` + `79991f2` | `src/rcv/septentrio.c` | ~250 | Refonte parser SBF (buffer par époque + commit fin d'époque) + flush début lecture + multi-stream thread-safe + fix rejeu | *"Mosaic X5 = notre récepteur. Sans ce cluster on perd des observations en streaming et on a des datasets incohérents au rejeu `.tag`. Cohérent à prendre en bloc."* |
| **15** | 🟠 | Septentrio annexes : `d2c94b3` + `431ec5a` + `682d64b` + `8080af0` + `62d2b69` | `src/rcv/septentrio.c` | ~80 | RCVSTDS option + GLO unaligned access + SBAS sbslongcorrh + UTC BDS CNAV + GPS raw cnav | *"Compléments du cluster #14 — qualité/pondération obs et fixes de décodage spécifiques signaux Mosaic."* |

**Total** : ~455 lignes de code touchées + 1 flag de compilation, sur 7-8 fichiers source distincts.

**EXCLUS du sprint** (à argumenter en revue) :
- ❌ `c0138bf` support `.BIA` — gros feature, ~200 lignes + suite de fixes (sprint 2)
- ❌ `d574080` refonte DCB/OSB — refactor tardif, propres bugs (sprint 2)
- ❌ `199be2b` `NFREQ=3→4` complet — change defaults UI/freq slots ; on prend juste le flag compil et `569197d` qui suffisent côté CLI
- ❌ `c535238` cycle-slip Doppler — gros feature, change comportement (sprint 2)
- ❌ Tous patches GUI (RTKPOST/RTKNAVI/Qt) — hors périmètre projet
- ❌ `2e1ddfa` PPP sans broadcast — feature récente (fév. 2026), risque immature
- ❌ `15445ce` auto-antenna selection — feature, à évaluer en sprint 2

### 7.2 Calendrier 2 semaines

```
S1.J1   Geler 6c53 prod • capturer .pos + .stat + .trace référence
        sur 5-10 scénarios prod diversifiés (rnx2rtkp post-process
        ET rtkrcv replay .tag) — toutes constellations / modes PPP
        Métriques baseline : RMS H/V, %fix, TTFF, %obs rejet,
                              checksums .stat et .trace
S1.J2   Vérifier `git cherry-pick` propre des patches 1-12 sur 6c53
        Rédiger patch #10 (rtkrcv cmd_restart cf §7.7)
S1.J3   Appliquer patches 1→12 dans l'ordre, build propre à chaque
        étape avec `-DNFREQ=4`. Un commit isolé par patch.
S1.J4   Smoke par patch : 1 scénario rapide à chaque étape
        (rnx2rtkp + rtkrcv replay), inspection .pos/.stat/.trace.
S1.J5   Run complet A/B (6c53 vs 6c53+patches+NFREQ=4) sur les
        5-10 scénarios — sortie .pos + .stat + .trace pour chacun.

S2.J6   Triage Δscore (§7.3) — investiguer toute dégradation > seuil
S2.J7   Bisect inter-patch en cas de régression isolée
S2.J8   Validation ciblée :
        - patch #1 : forcer init position non-convergente
        - patch #2 : dataset multi-constell sans GPS L1/L2
        - patch #10 : rtkrcv `restart` 2× même .tag → diff binaire = 0
        - patch #11 : vérifier .stat colonnes 5-7 freqs renseignées
        - flag NFREQ=4 : observer freq3 dans .stat
S2.J9   Doc : 1 page par patch (hash upstream, diff, scénario témoin,
        métrique avant/après, gotchas .stat/.trace)
S2.J10  Review équipe • décision go/no-go par patch
S2.J11  Canary 1 instance rtkrcv 24-48h en temps réel
S2.J12  Monitoring renforcé canary (RMS, fix-rate, fraîcheur SSR)
S2.J13  Décision deploy progressif
S2.J14  Buffer • write-up • rétrospective
```

### 7.3 Critère de non-régression — `Δscore`

Pour chaque scénario on calcule un score composite :

```
Δscore = w1 · (RMS_H_after − RMS_H_before) / RMS_H_before
       + w2 · (%fix_before − %fix_after) / 100
       + w3 · (TTFF_after − TTFF_before) / TTFF_before
```
avec `w1=0.5`, `w2=0.3`, `w3=0.2`.

**Critère pass** :
- moyenne `Δscore < 0.02` (2% d'amélioration globale) **ET**
- aucun scénario individuel ne dégrade de plus de 5% **ET**
- **patch #10 validé** : `rtkrcv` lancé deux fois sur le même `.tag` via `restart` produit `.pos` binairement identiques (`md5sum` égaux) **ET**
- **format `.stat` cohérent** : nos parsers existants ne lèvent pas d'erreur sur les nouvelles sorties (impacts patches `35b6fee` / `569197d`).

L'objectif n'est **pas la perfection** : *"on prouve que la moyenne s'améliore et qu'aucun scénario ne casse."*

### 7.4 Risques résiduels à signaler en kick-off

1. **Dépendance GPS structurelle non corrigée** (cf. section 5) — sprint dédié, refactor `udclk_ppp` + `pntpos`.
2. **PPP-AR reste un no-op** (cf. §1.8 — `ppp_ar.c` stub vide depuis 2016). Le mode `pos1-posmode=9` (`ppp-fixed`) tourne en réalité comme `ppp-kinematic`. **Si vos configs prod l'utilisent, sachez-le.** Sprint 2 = importer une implémentation tierce (rtklib-py / PPP-Wizard) + support `.BIA`.
3. **`NFREQ=4` partiellement appliqué** (cf. section 8) — vous gagnez les observations 4ᵉ freq mais sans iono-free, sans SSR sur cette freq, sans cycle-slip GF/MW. À fiabiliser en sprint 2.
4. **Pas de support produits IGS modernes** (`.BIA`, OSB absolus) — sprint 2.
5. **Patch #10 est maison** — pas accepté upstream, à maintenir nous-mêmes (mais c'est 3 lignes triviales).
6. **Format `.stat` change** avec patches `569197d` (et `35b6fee` indirect) — vérifier nos parsers prod **avant** déploiement.

### 7.5 Brief équipe — talking points

> **Contexte** : on est sur 6c53 (sept. 2021), validé en prod. 4½ ans de fixes upstream à intégrer mais on ne veut pas tout réavaler en bloc. Périmètre projet : CLI uniquement (`rnx2rtkp` post-process + `rtkrcv` temps réel).
>
> **Approche** : sprint chirurgical de 12 patches isolés + 1 flag de compilation (~125 LOC totales), ciblant les bugs critiques **sans** changement de comportement majeur. Aucun feature, aucun refactor.
>
> **Garantie** : chaque patch commit-isolé → rollback granulaire. Validation A/B sur scénarios prod existants. Critère : moyenne ≥ baseline et aucun scénario ne casse, **plus** un test reproductibilité strict (`md5sum` identique sur rejeu rtkrcv).
>
> **Plus-value spécifique Mosaic X5** : cluster Septentrio (#14 + #15) qui fiabilise le parser SBF + RCVSTDS + signaux modernes ; compilation `-DNFREQ=4` + patch `569197d` → on exploite 4 fréquences en observations, lecture `.stat` cohérente. NFREQ=6 (toutes fréquences Mosaic) reporté au sprint 2 après patches §8.4.
>
> **Hors-périmètre explicite** : support `.BIA`, OSB, PPP-AR, refonte DCB/OSB, GUIs → sprint 2.
>
> **Effort estimé** : ~10 j/h dev (1 dev plein), 4-5 j/h validation, ~2 j/h doc.

### 7.6 Outillage à préparer (J0)

- Script harness `compare_pos.py` : input `pos_baseline/*.pos` + `pos_candidate/*.pos` → table Δscore par scénario + verdict pass/fail
- Script `replay_rtkrcv.sh` : lance `rtkrcv` deux fois consécutives sur le même `.tag`, vérifie `md5sum` identiques (test du patch #10)
- Script `parse_stat.py` : valide que les `.stat` candidat parsent toujours avec nos parsers existants (test du patch #11 et `35b6fee`)
- Snapshot 5-10 scénarios + leurs `.pos`/`.stat`/`.trace` baseline figés (avant tout patch)
- Branche `sprint-1-cherrypicks` à partir du tag de prod actuel
- Procédure rollback documentée (revert d'un commit = revert d'un patch)

### 7.7 Patch #10 — code prêt à committer

**Fichier** : `app/consapp/rtkrcv/rtkrcv.c`

**Diff** (à appliquer après cherry-pick des patches 1-9, 11-12) :

```diff
--- a/app/consapp/rtkrcv/rtkrcv.c
+++ b/app/consapp/rtkrcv/rtkrcv.c
@@ -1078,6 +1078,9 @@ static void cmd_restart(char **args, int narg, vt_t *vt)
     trace(3,"cmd_restart:\n");

     stopsvr(vt);
+    /* re-init server state for replay reproducibility */
+    /* Without this, .nav, .ssr, .rtk etc. persist between runs */
+    rtksvrinit(&svr);
     if (!startsvr(vt)) return;
     vt_printf(vt,"rtk server restart\n");
 }
```

**Justification équipe** : sans `rtksvrinit` entre `stopsvr` et `startsvr`, la structure `svr` conserve les éphémérides, corrections SSR, état Kalman et historique observations du run précédent. Conséquence : rejouer 2× le même fichier `.tag` via la commande `restart` donne des solutions différentes — critique pour notre validation et nos tests de non-régression. Le patch est strictement additif (3 lignes), sans effet sur le démarrage initial qui appelle déjà `rtksvrinit` ligne 1831.

**Test associé** :
```bash
# Avant patch #10 : md5sum différents attendus
rtkrcv -m start_replay.cmd > run1.pos && md5sum run1.pos
rtkrcv -m start_replay.cmd > run2.pos && md5sum run2.pos
# Avec patch #10 : md5sum identiques requis (critère pass)
```

---

## 8. NFREQ multi-fréquence pour Septentrio Mosaic X5 — état d'avancement et reste à faire

> Le récepteur cible projet est le **Septentrio Mosaic X5** (format SBF, parser `src/rcv/septentrio.c`). Capacités : GPS L1/L2/L5, GLO L1/L2/L3, Galileo E1/E5a/E5b/E5-AltBOC/E6, BeiDou B1I/B1C/B2a/B2b/B3I, QZSS L1/L2/L5/L6, NavIC L5, SBAS L1/L5 — soit **jusqu'à 5-6 fréquences par constellation**. RTKLIB définit `MAXFREQ=6` (`rtklib.h:99`) et `NFREQ=3` par défaut (`rtklib.h:157`). L'audit (§6.1) a montré que `NFREQ>3` est partiellement appliqué dans les algos. Cette section discute le compromis et le reste à faire.

### 8.1 ✅ Ce qui marche avec `-DNFREQ=4`

- Stockage des observations 4ᵉ fréq (`obs->L[3]`, `obs->P[3]`, `obs->code[3]`, `obs->SNR[3]`, `obs->LLI[3]`) — structure `obsd_t` (`rtklib.h:586-591`) est dimensionnée `[NFREQ+NEXOBS]`.
- Décodage des 4 fréquences depuis u-blox UBX (`rcv/ublox.c`).
- Décodage RTCM3 MSM (4 fréquences observation).
- Sortie RINEX 4 fréquences (`convbin`).
- SPP (single point positioning) utilise les 4 freqs si disponibles.
- Sortie `.pos` standard (positions identiques avec/sans `NFREQ=4` si on n'utilise pas la 4ᵉ freq dans le filtre).
- Sortie `.stat` colonnes 5-7 freqs renseignées **après cherry-pick `569197d`** (patch #11).

### 8.2 Cartographie freq slot par constellation (Mosaic X5)

Vue d'ensemble lue dans `src/rtkcmn.c:611-706` (`code2freq_*`). **Le mapping détermine quelles fréquences sont accessibles à NFREQ donné** :

#### Galileo (5 signaux Mosaic X5)
| Slot | Signal | Acquis si NFREQ ≥ |
|---|---|---|
| 0 | E1 | 1 |
| 1 | **E5b** ⚠️ (E5b avant E5a) | 2 |
| 2 | E5a | 3 |
| 3 | E6 | 4 |
| 4 | E5-AltBOC (codes L8I/L8Q/L8X) | 5 |

#### BeiDou (6 signaux Mosaic X5)
| Slot | Signal | Acquis si NFREQ ≥ |
|---|---|---|
| 0 | B1I (legacy) | 1 |
| 1 | B2I / B2b | 2 |
| 2 | B2a | 3 |
| 3 | B3I | 4 |
| 4 | **B1C** ⚠️ (commit `35dfb17` a déplacé B1C de slot 1 à slot 4 pour éviter conflit avec B2I) | 5 |
| 5 | B2ab | 6 |

→ **`NFREQ=4` rate B1C** — code dominant en BDS-3. Pour PPP BeiDou-3 propre, **`NFREQ≥5`**. Alternative : changer `pos1-codepri-...` pour forcer B1C en slot 0.

#### GLONASS (3 freqs legacy + G1a/G2a CDMA gen3)
| Slot | Signal |
|---|---|
| 0 | G1 (incl. G1a CDMA via codes L4*) |
| 1 | G2 (incl. G2a CDMA) |
| 2 | G3 |

#### GPS (3 freqs)
- L1 / L2 / L5 → NFREQ=3 suffit

#### QZSS (4 freqs)
- L1 / L2 / L5 / L6 → NFREQ=4 nécessaire pour L6

### 8.3 Ce qui ne marche pas / partiellement avec NFREQ>3

| Sévérité | Problème | Fichier:ligne | Effet PPP Mosaic X5 |
|---|---|---|---|
| 🟠 | `seliflc()` retourne au max 2 | `rtkcmn.c:3744` | iono-free PPP n'utilise que freq 0+f2 (f2 ≤ 2) → freqs ≥3 ignorées pour combinaison principale |
| 🟠 | `gfmeas()` / `mwmeas()` n'utilisent que `code[0]` et `code[1]` | `ppp.c:385-406` | Cycle-slip GF / Melbourne-Wubbena uniquement entre freq 0,1 → cycle-slip freq 2-5 via LLI seul |
| 🟠 | `EFACT_GPS_L5` seul existe (pas `EFACT_GAL_L5` etc.) | `ppp.c:355-357` | Variance L5/E5a/B2a sous-estimée pour non-GPS → poids trop fort en filtre |
| 🟠 | `obs->Pstd[frq]` / `Lstd[frq]` accédés sans guard | `ppp.c:368-369` | OOB potentiel si récepteur fournit > NFREQ stdevs |
| 🟠 | `test_hold_amb()` ne teste que `fix[0]` et `fix[1]` | `ppp.c:1162` | Fix-and-hold sur freq 2-5 jamais validé (impact PPP-AR futur) |
| 🟠 | Slip iono-free hardcodé `slip[0] \|\| slip[1]` | `ppp.c:754` | Slip freq 2-5 ne réinit pas biais iono-free |
| 🟡 | Facteur iono-free `SQR(3.0)` valide pour GPS L1/L2 uniquement | `ppp.c:372` | Variance iono-free légèrement biaisée pour combinaisons E1/E5x ou B1/B2x |
| 🟡 | DCB L5 (`uddcb_ppp`) seul biais récepteur estimé | `ppp.c:819-821` | NFREQ≥4 nécessite des biais récepteur additionnels (DCB E5b, E6, B1C, B3) — actuellement absorbés dans les ambiguïtés |

> ⚠️ **Note SSR clarifiée** : `ssr_t.deph[3]/ddeph[3]/dclk[3]` sont des composantes RAC + polynôme C0/C1/C2 — **pas des indices de fréquence**. Les corrections SSR orbit/clock sont scalaires par satellite et s'appliquent à toutes les fréquences. Les biais SSR3 (code) et SSR7 (phase) sont indexés par signal-code (`MAXCODE=70`), couvrant tous les signaux Mosaic X5.

### 8.4 Choix NFREQ pour le projet — analyse coût/bénéfice

| NFREQ | Mosaic X5 manqué | Verdict |
|---|---|---|
| 3 (défaut RTKLIB) | E5b/E5a/E6 GAL, B2a/B3/B1C BDS, L6 QZS | Vieille config, sous-exploite massivement |
| 4 | **B1C BDS-3**, E5-AltBOC GAL, L6 QZS, G1a/G2a GLO | Trade-off : pas de B1C → sous-exploite BDS-3 modernes |
| **5** ✅ | B2ab BDS, G2a GLO | **Recommandé** — capture B1C + B2a + B3 + B1I pour BDS, E1+E5b+E5a+E6 pour Galileo |
| 6 (= MAXFREQ) | — | Maximum, exploite AltBOC GAL et B2ab BDS |

**Recommandation révisée** :

- **Sprint 1 : `NFREQ=5`** (et non 4 comme précédemment) — capture B1C BeiDou-3 et E6 Galileo, frontière où les bugs §8.3 commencent à mordre mais sans aller à l'extrême
- **Sprint 2 : `NFREQ=6`** uniquement après patches §8.5 (extension `seliflc`, `gfmeas`/`mwmeas` paramétrés, `EFACT_*_L5` par constellation)

**Pourquoi pas NFREQ=4** :
- B1C est le code BDS-3 dominant — toute station IGS récente émet B1C, tous les produits SSR le couvrent
- L'écart entre NFREQ=4 et NFREQ=5 est faible en RAM (+~3 KB) et CPU (+~2 % filtre Kalman)
- Patch #11 (`569197d`) ouvre déjà `prstatus` à 5-7 freqs

**Conséquence pratique par scénario** :
| Scénario | NFREQ=3 | NFREQ=4 | **NFREQ=5** | NFREQ=6 |
|---|---|---|---|---|
| SPP / standalone | ✅ 3 freqs | ✅ 4 | ✅ 5 | ✅ 6 |
| PPP iono-free L1/L2 GPS classique | ✅ | ✅ | ✅ | ✅ |
| PPP iono-free GPS+GAL+BDS-3 multi-constell | 🟡 | 🟡 (sans B1C) | ✅ | ✅ |
| PPP exploitant E1/E5a + E1/E5b simultané | ❌ `seliflc` | ❌ | ❌ | ❌ |
| PPP temps réel + SSR | ✅ | ✅ | ✅ | ✅ (clarifié — voir §8.3) |
| Cycle-slip robuste freqs ≥2 | LLI seul | idem | idem | idem |
| PPP-fixed (AR) | ❌ | ❌ | ❌ | ❌ (`ppp_ar.c` vide) |

⚠️ **Alternative à explorer** : modifier `code2freq_BDS` pour mettre B1C en slot 0 ou 2 (pré-empte B1I/B2a). Évite la dépendance à NFREQ=5+. ~5 lignes mais modifie l'ordre des colonnes RINEX → impact parsers `.stat` aval. Décision projet.

### 8.5 Sprint 2 — patches à écrire pour rendre NFREQ≥5 propre

Estimation à partir de l'audit corrigé. Aucun de ces patches n'existe upstream — c'est du dev maison. **Le patch n°4 (ssr_t) est retiré** : analyse approfondie a montré que `ssr_t.deph[3]` n'est pas lié aux fréquences (composantes RAC/clock).

| Tâche | Effort estimé | Fichiers | Description |
|---|---|---|---|
| 1. Étendre `seliflc()` pour `nf≥4` | ½ jour | `rtkcmn.c:3744` | Choix dynamique freq2 selon système et NFREQ |
| 2. Paramétrer `gfmeas`/`mwmeas` | 1 jour | `ppp.c:385-406` | Boucle sur paires (0, f2) avec f2 ∈ {1, 2, …, nf-1} |
| 3. `EFACT_*_L5` par constellation | ½ jour | `ppp.c:355-357` + `rtklib.h` | Ajouter `EFACT_GAL_L5`, `EFACT_CMP_L5`, `EFACT_IRN_L5` |
| 4. Facteur iono-free dynamique | ½ jour | `ppp.c:372` | Calculer `SQR(f1/(f1-f2))` selon paire réelle |
| 5. Slip iono-free utilisant `f2` dynamique | ½ jour | `ppp.c:754` | Référencer `slip[seliflc(nf,sys)]` |
| 6. `test_hold_amb` boucle sur toutes freqs | ¼ jour | `ppp.c:1162` | `for (f=0; f<opt->nf; f++)` |
| 7. Guards `obs->Pstd[frq]` / `Lstd[frq]` | ¼ jour | `ppp.c:368-369` | `if (frq < NFREQ)` avant accès |
| 8. **DCB récepteur étendu** (E5b, E6, B1C, B3) | 1-2 jours | `ppp.c:819-821`, `rtklib.h` | Estimer 1 DCB par freq>2 (ou par paire avec freq 0). Sans cela les biais récepteur sont absorbés par les ambiguïtés et la convergence est lente |

**Total sprint 2 spécifique NFREQ≥5** : ~4-5 jours. Combiner avec support `.BIA` + PPP-AR pour un sprint 2 complet ~3-4 semaines.

### 8.6 Validation Mosaic X5 spécifique

Tests à ajouter au harness :
- Compter le nombre d'observations `freq[3]`, `freq[4]` (et `freq[5]` si NFREQ=6) dans `.stat` (proxy : SNR colonnes 5+)
- Vérifier que **B1C est bien observé** (NFREQ ≥ 5) : `grep "C2I.*C1P\|C1P" .stat` ou inspection des codes RINEX `*.obs` (signal `C1P` = B1C BDS)
- Vérifier que les positions PPP en mode iono-free L1/L2 sont **identiques** entre `NFREQ=3`, `4`, `5` (les fréqs ≥3 ne doivent pas dégrader la solution sur les modes existants)
- Tracer (`.trace` level 3+) la sélection `seliflc` pour confirmer le comportement attendu
- Mesurer le gain SPP avec/sans fréqs supplémentaires sur scénarios à faible visibilité
- **Spécifique Mosaic X5** : vérifier que les SBF MeasEpoch + EndOfMeas reçus sur 5-6 fréqs sont bien décodés sur `670690a` + `784056a` cluster (sinon perte d'observations sur signaux modernes BDS-3 / Galileo)
- **Test convergence BDS-3 only** : forcer config `pos1-navsys=2` (BDS seul) avec NFREQ=4 vs 5 → différence visible si B1C absent

---

## 9. Documentation — décision doc-as-code

> Décision projet : **doc-as-code** sur ce dépôt git, rendu via **Astro** (statique, accessible aux non-tech). Confluence éventuel uniquement en miroir lecture-seule plus tard si besoin externe.

### 9.1 Pourquoi doc-as-code

- **Version doc liée au commit code** : un tag git = un état complet (spec + design + test + code). Confluence ne sait pas le faire.
- **Revue PR = revue doc** : sign-off auditable via CODEOWNERS + required reviewers.
- **Diff/blame natifs** : qui a changé quoi, quand, pourquoi.
- **Traçabilité automatisable** : matrice spec ↔ test ↔ code générée par CI.

### 9.2 Arborescence cible

```
docs/
├── specs/          # exigences fonctionnelles (REQ-*)
├── design/         # doc design / architecture (DES-*)
├── tests/          # protocoles tests (TST-*)
├── adr/            # Architecture Decision Records (ADR-NNNN)
└── runbooks/       # procédures ops (rollback, canary, replay…)
```

### 9.3 Traçabilité

- **IDs stables** : `REQ-PPP-001`, `DES-PPP-001`, `TST-PPP-001`, `ADR-0001`
- Front-matter YAML par doc avec champs `id`, `status`, `linked-to`, `verified-by`
- CI génère une **matrice de traçabilité** (REQ → DES → TST → commit)

### 9.4 ADRs

Une décision = un ADR daté et numéroté. Exemples typiques pour ce projet :
- ADR-0001 : Choix de rester sur 6c53 + cherry-pick (vs migration b34l)
- ADR-0002 : Compilation `-DNFREQ=4` pour u-blox X5
- ADR-0003 : Patch maison rtkrcv `cmd_restart` (cf. §7.7)
- ADR-0004 : Périmètre projet = CLI uniquement (UI exclue)
- ADR-0005 : Sprint 2 = `.BIA` + PPP-AR
- ADR-0006 : Doc-as-code + Astro

### 9.5 Validation CI

- Liens cassés (lychee ou markdown-link-check)
- IDs orphelins / dupliqués (script maison sur front-matter)
- Schéma front-matter valide
- Tests intégrés (TST-* exécutables liés à un script de test)
- Build Astro doit passer

### 9.6 Rendu archivé par release

À chaque tag git :
- Astro build → static site déployé (Pages ou serveur interne)
- Snapshot HTML/PDF archivé pour audit (avec hash commit)
- Matrice de traçabilité figée dans la release

### 9.7 Hors-périmètre immédiat

- Miroir Confluence : reporté, à n'envisager que si parties prenantes externes le demandent explicitement
- Workflow signature électronique : à voir selon contraintes qualité/réglementaires

---

## 10. Optimisation PE temps réel — latence / CPU / RAM

> Synthèse de 3 audits parallèles ciblés sur le périmètre **rtkrcv en temps réel avec rejeu `.tag`**. Le but : mesurer le delta 6c53 → HEAD, identifier les goulots actuels et lister les actions.

### 10.1 Vue d'ensemble — bilan 6c53 → HEAD

| Aspect | 6c53 (réf.) | HEAD nu | HEAD optimisé* | Verdict |
|---|---|---|---|---|
| **Latence end-to-end** | ~25 ms / époque | 15-18 ms | 10-12 ms (event-driven) | ✅ ~30 % gain natif, ~50 % avec patches maison |
| **CPU** | 100 % baseline | 95-110 % | 85-90 % | ⚠️ Net presque neutre, gains annulés par SOFA + heap migration ; +10-15 % avec compile flags |
| **RAM runtime initial** | ~5.7 MB | ~5.7 MB | ~3-4 MB | ✅ Quasi identique en standard, réductible avec compile-time tuning |
| **RAM pic Kalman** | ~1.2 MB | ~1.3 MB (NFREQ=4) | ~0.6 MB (NFREQ=3) | ⚠️ NFREQ=4 ajoute ~100 KB |
| **Binaire statique** | ~950 KB | ~1.5 MB | ~1.2 MB (gc-sections) | ❌ +550 KB (libopenblas + SOFA + features) |

*HEAD optimisé = HEAD + compile flags reco + runtime tuning + patches maison §10.5

### 10.2 Latence

#### Gains acquis depuis 6c53 (~6-10 ms / époque)
| Commit | Gain estimé | Mécanisme |
|---|---|---|
| `fbb9ea2` matmul cache-aware | 3-8 ms | Hoisting tests β=0, spécialisation NN/NT/TN/TT, accès séquentiel cache |
| `d96a5c7` `dot2`/`dot3` | 0.5-1 ms | Inline pour n=2,3 (appelé ~50×/époque) |
| `f91603c` `initx` refactor | 0.3-0.7 ms | Boucles deux-passes au lieu d'une ternaire, symétrie `P[i,j]` |
| `28ad77c` init SSR `t0[]` | < 1 ms variance | Élimine jitter timetag initial sur replay `.tag` |
| `269c2d5` resync RTCM3 | variable | Évite scan 10-100 bytes après msg invalide |
| `f542de9` BeiDou SSR opérationnel | latence app. corrections BDS | |

#### Goulots actuels (HEAD) — top 5
1. **`svr->cycle = 100 ms` par défaut** dans rtksvr → `sleepms(cycle - cputime)` polling passif → 60+ ms d'inactivité par cycle. **Action runtime immédiate : passer à 20-50 ms.**
2. **Décodage RTCM3 MSM** : 2-4 ms, pas de parallélisation inter-messages.
3. **Filtre PPP Kalman** : 1-3 ms, Cholesky O(n³), pas de SIMD.
4. **NTRIP reconnect** intermittent : 100-5000 ms (aucun watchdog par défaut). **Action : timeout 30s → 5-10s.**
5. **Copies inter-buffers** rtksvr (`sbuf` → `pbuf` → décodage) : 0.1-0.5 ms × N obs.

#### Patches maison à écrire (après sprint 1)
| Patch | Scope | Gain | Risque |
|---|---|---|---|
| Polling → event-driven (`sleepms` → `select()` 50 ms) | ~100 LOC | -2-5 ms | Moyen |
| Buffer SSR par timestamp + apply-on-rtkpos | ~200 LOC | -0.5 ms variance | Faible |
| `matmul` parallèle (OpenMP/SIMD/AVX) | ~50 LOC | -3-5 ms sur grand `nx` | Moyen |
| Élim. copie obs brutes (ring buffer direct) | ~300 LOC | -0.2 ms | Élevé |

#### Méthode de mesure latence
Instrumentation 4-points dans rtksvr + script post-mortem :
```c
#ifdef TRACE_LATENCY
    tick_raw_in     = tickget();  /* arrivée raw msg */
    tick_decoded    = tickget();  /* post-RTCM3 decode */
    tick_rtkpos_in  = tickget();  /* pré-rtkpos */
    tick_rtkpos_out = tickget();  /* post-solution */
#endif
```
```bash
grep "latency:" rtkrcv.log | awk '{sum+=$NF; n++} END {print "Mean:", sum/n " µs"}'
```

### 10.3 CPU

#### Gains depuis 6c53 (~15-20 % théoriques)
- `fbb9ea2` matmul cache-aware : +5-10 %
- `d96a5c7` `dot2`/`dot3` : +2-3 %
- `f06df21` cache sun/moon (THREADLOCAL + timediff < 1µs) : +40-60 % sur ces fonctions, ~90 % cache hit en temps réel
- `62a2abf` SOFA OFF par défaut (`-DSUNPOS_ORIG` `-DMOONPOS_ORIG`) : +30-50 % sur sun/moon (< 1 mm de différence sur la solution)
- `558048a` `sbstropcorr` cache thread-safe : +5-10 %

#### Régressions (~5-12 % cumulés)
- `8158789` import SOFA (epv00.c 152 KB, moon98.c 25 KB) : si **non** désactivé via define
- `5a3566d` heap migration : malloc/free par époque (-2-5 %) sur cible embarquée
- NFREQ=4 (état Kalman étendu) : +3-5 %
- Code biases boucles ajoutées (preceph.c) : +1-2 %

→ **Bilan net = -5 % à +5 %** selon config. Avec compile flags optimaux **+10-15 %**.

#### Hot paths actuels (HEAD)
| Rang | Fonction | % CPU |
|---|---|---|
| 1 | Filtre Kalman (`filter`, `kalman update`) | 25-30 % |
| 2 | `ppp_res` (résidus + géométrie sat) | 20-25 % |
| 3 | `geodist` / `satazel` / `antmodel` | 12-18 % |
| 4 | Sun/moon + ECI↔ECEF | 8-15 % SOFA ON / 2-3 % SOFA OFF |
| 5 | Interpolation SP3 (`pephpos` Neville) | 5-10 % |
| 6 | Stream / RTCM / RINEX I/O | 5-8 % |

#### Patches maison CPU (après sprint 1)
| Patch | Gain |
|---|---|
| Factoriser `satazel` (appelé 2× par obs : setup + résidus) | -10-12 % sur `ppp_res`, ~-0.5 ms / époque (30 sat) |
| Cache `searchpcv` par antenna type + time | -2-5 % sur `peph2pos` |
| Compile `-DTRACE_DISABLE` en prod | -5-10 % runtime |
| Unroll boucles 3D `geodist`/`satazel` | -3-5 % sur géométrie |

#### Méthode profilage
```bash
# perf (production-like, faible overhead)
perf record -g -F 100 -e cycles:u --output=perf.data ./rtkrcv -s cfg
perf report --stdio | head -50
# Flamegraph
perf script | stackcollapse-perf.pl | flamegraph.pl > perf.svg

# callgrind (cache miss + branche)
valgrind --tool=callgrind --cache-sim=yes ./rtkrcv -s cfg
kcachegrind callgrind.out.*

# RDTSC instrumentation (ARM : cntvct_el0)
uint64_t t0 = __rdtsc();
/* ... code ... */
trace(0, "ppp_res: %llu cycles\n", __rdtsc() - t0);
```

### 10.4 RAM

#### Gains depuis 6c53
- `d574080` (mar 2026) : suppression `nav->rbias[MAXRCV][NFREQ][MAX_CODE_BIASES]` jamais utilisée → **-9 KB** runtime (×3 nav)
- `5a3566d` (mai 2025) : heap migration → évite stack overflow (>8 MB Linux)
- `bb73478` : `rtksvr_t` sur heap dans RTKNAVI-Qt
- `8a6a188`, `8774401`, `b88dfee` : free explicite sur paths d'erreur → -9 KB par redémarrage en erreur
- `f10fa74` : pas d'alloc inutile `nav->seph` en post-process → -3 KB

#### Régressions
- `27290f4` réintroduction libopenblas.a (Embarcadero) : **+429 KB** binaire statique (négligeable pour rtkrcv)
- `f35cf00` MAXPRNCMP 46→50 : +3.6 KB net (4 sats × structures cumulées)
- SSR BeiDou + iono-free unifiée + cache tropo SBAS : ~+6 KB code
- Import SOFA `8158789` : **+450 KB** binaire si compilé sans `-DSUNPOS_ORIG -DMOONPOS_ORIG`

#### Top consommateurs runtime (HEAD)
1. **Matrice P Kalman pic** : ~1.3 MB (nx² × 8 bytes, NFREQ=4 multi-constell). Inévitable structurellement.
2. **Buffers stream** `buff[3] + pbuf[3] + sbuf[2]` × 32 KB : **224 KB**. Configurable via `rtksvrstart(buffsize)`.
3. **`raw[3]`** subframe buffers : ~340 KB.
4. **`nav` allocations** (eph, geph, cbias, pcvs) : ~250 KB.
5. **`obs[3][MAXOBSBUF]`** : ~100-150 KB selon `MAXOBSBUF` et `MAXOBS`.
6. **`rtcm[3]`** : ~50 KB.
7. **`ssat[MAXSAT]`** : ~71 KB (322 B × 218 sats).

#### Réductions actionnables
| Action | Gain RAM |
|---|---|
| `-Os -fdata-sections -ffunction-sections -Wl,--gc-sections` | -30 KB binaire |
| `-DMAXOBS=32` (si scénario rover seul, < 30 obs/époque) | -3 MB pic théorique (ou plus selon usage `obs[3][MAXOBSBUF]`) |
| `-DNFREQ=3` (si non requis par X5 — voir §8) | -15 % RAM Kalman, -10 % buffers obs |
| `buffsize 32 KB → 8 KB` (NTRIP basse latence) | -144 KB |
| Compilation conditionnelle MAXSAT=100 (GPS/GLO/GAL/QZS uniquement) | -40 % RAM initiale (matériel < 256 MB) |
| `peph[]` borné à `24h / interval` (~2880 entries) | Évite croissance unbounded en post-process long |
| `navsel=1` (rover seul) au lieu de 0 (all systems) | -20-30 % allocs `nav.eph` / `nav.peph` |

#### Méthode de mesure RAM
```bash
# Statique (binaire)
size $(which rtkrcv)

# Runtime (live)
watch -n 1 'grep -E "VmRSS|VmHWM|VmPeak" /proc/$(pgrep rtkrcv)/status'

# Heap profiling
valgrind --tool=massif ./rtkrcv -s cfg
ms_print massif.out.<PID>

# Heaptrack (timeline interactif)
heaptrack ./rtkrcv -s cfg
heaptrack_gui heaptrack.rtkrcv.<PID>.zst
```

### 10.5 Synthèse — actions consolidées sur les 3 axes

#### Compile flags recommandés (impact mixte latency / CPU / RAM)
```makefile
# Production rtkrcv
CFLAGS = -std=c99 -O3 -march=native -mtune=native \
         -ffast-math -fno-strict-aliasing -flto \
         -fomit-frame-pointer \
         -fdata-sections -ffunction-sections \
         -DNDEBUG \
         -DSUNPOS_ORIG -DMOONPOS_ORIG \
         -DTRACE=2 \
         -DNFREQ=4 -DNEXOBS=3
LDFLAGS = -Wl,--gc-sections,--strip-all -flto

# Cibles ARM (Cortex-A9 / ARMv7 + NEON)
CFLAGS += -mfpu=neon -mfloat-abi=hard
# Cibles x86_64 + AVX2
CFLAGS += -mavx2 -mfma
```
**Gains attendus** : -5-15 % CPU, -2-8 ms latence, -30 KB binaire.

#### Configuration runtime rtkrcv recommandée
| Paramètre | Défaut | Recommandé | Effet |
|---|---|---|---|
| `svr->cycle` | 100 ms | **20-50 ms** | -10-50 ms latence par cycle, +20-40 % CPU |
| `buffsize` | 32768 | 8192 (NTRIP basse BP) | -144 KB RAM |
| `trace level` | 2 | **1** | -5-10 % CPU |
| NTRIP timeout | 30 s | 5-10 s | -25 s downtime SSR pic |
| `MAXOBSBUF` | 128 | 64 si rover seul | -50 % RAM `obs[]` |
| `pos2-niter` | 5 | 3 si convergent rapide | -CPU ppp_res |
| `pos2-elmin` | 10° | 15° (selon scénario) | Moins de sat marginaux |

#### Patches maison à planifier (sprint 2 ou 3)
| Patch | Axe | Scope | Gain |
|---|---|---|---|
| Event-driven server loop (`select()` au lieu de `sleepms`) | latence | ~100 LOC | -2-5 ms |
| Buffer SSR par timestamp | latence | ~200 LOC | -0.5 ms variance |
| Factoriser `satazel` (1× au lieu de 2×) | CPU | ~30 LOC | -10-12 % `ppp_res` |
| Cache `searchpcv` antenna+time | CPU | ~50 LOC | -2-5 % `peph2pos` |
| `matmul` SIMD/OpenMP | CPU + latence | ~50 LOC | -3-5 ms |
| Borner `peph[]` (anti-fuite long-run) | RAM | ~20 LOC | Croissance plafonnée |

### 10.6 Cible build : Raspberry Pi Compute Module 5 (CM5)

> Cible projet : **CM5** (Broadcom BCM2712, 4× ARM Cortex-A76 ARMv8.2-A @ 2.4 GHz, NEON, 4-16 GB LPDDR4X).

#### Profil ARMv8.2-A vs ARMv7 (ce que ça change)

| Aspect | ARMv7 (Pi 3 / older) | ARMv8.2-A AArch64 (CM5) |
|---|---|---|
| Strict alignment | Strict sur load/store génériques | Permissif sauf LDP/STP/atomics/NEON spécifiques |
| `431ec5a` GLO subframe unaligned | **Bloquant** (segfault) | Non bloquant en pratique mais à prendre quand même (instructions vectorisées peuvent fauter) |
| NEON | Optionnel | Obligatoire (intégré ISA) |
| FP16 | Optionnel | Présent sur Cortex-A76 |
| Pointeurs | 32-bit | 64-bit (impact taille structures) |
| Atomic | LDREX/STREX | LDADD/CAS plus efficaces |
| LSE atomics | Non | Oui (Cortex-A76 supporte) |

#### Recette compile CM5 (production rtkrcv)

```makefile
# CM5 — Cortex-A76 production
CC = gcc
CFLAGS = -std=c99 -O3 \
         -mcpu=cortex-a76 -mtune=cortex-a76 \
         -march=armv8.2-a+crc+crypto+fp16 \
         -ffast-math -fno-strict-aliasing -flto \
         -fomit-frame-pointer \
         -fdata-sections -ffunction-sections \
         -DNDEBUG \
         -DSUNPOS_ORIG -DMOONPOS_ORIG \
         -DTRACE=2 \
         -DNFREQ=5 -DNEXOBS=3 \
         -DENAGLO -DENAQZS -DENACMP -DENAGAL -DENAIRN \
         -DSVR_REUSEADDR
LDFLAGS = -Wl,--gc-sections,--strip-all -flto
LIBS = -lm -lpthread

# Alternative `-mcpu=native` sur la machine de build CM5
# Mais préférable d'expliciter pour reproductibilité CI
```

**Notes** :
- `-mcpu=cortex-a76` active automatiquement les bonnes options NEON / FP16 / LSE atomics
- `-flto` (Link-Time Optimization) gain ~3-5 % sur AArch64 GCC ≥ 11
- `-march=armv8.2-a+crypto+fp16` : `crypto` peu utile à RTKLIB mais cohérent ; `fp16` peut aider sur certaines opérations matricielles si compilateur l'utilise
- **Ne pas activer** `-mcpu=cortex-a72` (CM4/Pi 4) — vous perdez ~5 % perf sur CM5
- Cross-compile possible depuis x86_64 avec `aarch64-linux-gnu-gcc`

#### Profilage natif sur CM5

```bash
# perf disponible sur Raspberry Pi OS (apt install linux-perf)
sudo perf record -F 200 -g -e cycles:u ./rtkrcv -s cfg
sudo perf report --stdio | head -50

# Sysfs CPU info
cat /proc/cpuinfo  # vérifier "cpu implementer" + "cpu part" = 0xd0b (Cortex-A76)

# Surveillance température (CM5 throttle à 80°C)
vcgencmd measure_temp
vcgencmd get_throttled  # 0x0 = pas de throttle
```

⚠️ **Throttling thermique CM5** : sans dissipateur actif, le CM5 throttle aggressivement sous charge soutenue. Pour un `rtkrcv` 24/7, **prévoir un dissipateur + ventilo**, sinon les gains de §10.5 sont annulés par la baisse de fréquence dynamique.

#### Threading rtkrcv sur 4 cœurs CM5

Le CM5 a 4 cœurs A76. `rtkrcv` est mono-thread principal + threads stream. Pour exploiter :
- Pin du thread principal sur cœur dédié (`taskset -c 2 ./rtkrcv ...`)
- Cœur 0 réservé à l'OS / IRQ
- Cœurs 1-3 pour rtkrcv + workers
- Avec patch `matmul` SIMD/OpenMP (sprint 3) : `OMP_NUM_THREADS=3` exploite réellement les 3 cœurs disponibles

#### RAM CM5 : moins critique que sur cible embarquée

Variantes CM5 à 4 / 8 / 16 GB → la RAM n'est plus un facteur limitant. Les sections §10.4 sur RAM perdent en priorité **sauf** si :
- Vous avez d'autres process critiques (NTRIP cast, MQTT publisher, watchdog, logger…) qui se partagent la RAM
- Vous lancez plusieurs `rtkrcv` en parallèle sur le même CM5 (multi-récepteur)

#### Spécifique kernel / Pi OS

- **Raspberry Pi OS Bookworm 64-bit** ou Ubuntu 24.04 ARM64 — cohérent avec ARMv8.2-A
- `_POSIX_C_SOURCE=200112L` (commit `a2cc5a9`) compatible — pas de souci POSIX
- Lib `librt` désormais conditionnée Linux uniquement (`eafc728`) — OK
- I/O : `mmap` plus rapide que `read` pour les gros `.tag` — non utilisé par RTKLIB par défaut, opportunité d'optim

### 10.7 Roadmap optimisation

**Sprint 1 (en cours)** — voir §7 :
- ✅ Cherry-picks fixes critiques (déjà incluent `fbb9ea2`, `d96a5c7`, `f91603c` côté perf)
- ✅ Patch maison rtkrcv `cmd_restart` (réplicabilité `.tag` — patch #10)
- ✅ Compile flag `-DNFREQ=4` (X5)

**Sprint 1.5 — quick wins** (1-2 jours, à intégrer si temps reste sur sprint 1) :
- Compile flags optimaux (§10.5) → bench A/B simple sur dataset témoin
- Configuration runtime tunée (`svr->cycle` 20-50 ms, trace level 1, NTRIP timeout 10 s)
- `-DSUNPOS_ORIG -DMOONPOS_ORIG -DTRACE=2`

**Sprint 2 — optimisations algos + features manquantes** (3-4 semaines) :
- Support `.BIA` / OSB
- Patches NFREQ=4 propres (§8.4)
- Factoriser `satazel`, cache `searchpcv`
- PPP-AR (rtklib-py partiel ou PPP-Wizard)

**Sprint 3 — refactor temps réel + concurrence** (2-3 semaines) :
- Event-driven server loop
- SSR timestamp buffer
- `matmul` SIMD si métriques le justifient
- Dépendance GPS (§5) — refactor `udclk_ppp` + `pntpos`

---

## 11. Autres améliorations notables non couvertes plus haut

> Survey ciblé pour ne rien laisser de côté. Périmètre toujours CLI (rnx2rtkp + rtkrcv).

### 11.1 Récepteurs binaires (parsers `src/rcv/*.c`)

#### Septentrio Mosaic X5 (`src/rcv/septentrio.c`) — récepteur projet 🎯

**Cluster critique 2024-2025** — refonte du parser SBF + cascade de fixes. À prendre en bloc cohérent (sinon perte d'observations ou incohérences inter-époques) :

| Prio | Commit | Date | Effet |
|---|---|---|---|
| 🔴 | `670690a` | 2024-06-28 | **Refonte parser** : observations bufferisées en mémoire puis commit en fin d'époque (vs commit immédiat). 197 lignes touchées. |
| 🔴 | `784056a` | 2024-07-11 | Complément à `670690a` : observations passées à RTKLIB **même si l'event-end-block manque**. Critique en streaming temps réel si Mosaic X5 omet des EOEB. |
| 🔴 | `253bfb3` | 2024-08-07 | Flush au début des routines de lecture (déplace le check) → évite datasets incohérents inter-époques (bug Jens Reimann). |
| 🔴 | `26ed828` | 2024-08-25 | Static data déplacé dans `struct raw` → permet **multi-stream SBF simultané, thread-safe**. Important si vous décodez plusieurs streams (rover + base). |
| 🔴 | `79991f2` | 2025-05-25 | **Bug rejeu / 2ᵉ passe** : sur conversion ou rejeu, `convrnx` rejetait observations non plus récentes que le buffer → données manquantes. **Pertinent pour vous : `.tag` replay et conversion SBF→RINEX.** |
| 🟠 | `d2c94b3` | 2024-08-10 | SNR, doppler, GLO FCN, **option `RCVSTDS`** (stddev observations dans RINEX → meilleure pondération filtre). |
| 🟠 | `431ec5a` | 2024-08-08 | GLO raw cnav : évite accès subframe **non aligné** → critique si build ARM strict alignment. |
| 🟠 | `682d64b` | 2024-08-29 | `sbslongcorrh` : décodage temps corrigé (SBAS sur Mosaic). |
| 🟠 | `8080af0` | 2025-07-10 | Décodage UTC BDS CNAV corrigé. |
| 🟠 | `62d2b69` | 2024-06-27 | `decode_gpsrawcnav` `decode_frame` fix. |
| 🟡 | `3a319cd` | 2024-08-25 | Implémente le rx setup block. |
| 🟡 | `0724c41` | 2024-08-29 | `ID_GEOALM` (case ajouté, pas implémenté). |
| 🟡 | `5126471` | 2024-08-29 | `flushobuf` : passe la return value. |
| 🟡 | `85d3098`, `2f23eb3`, `12b26b7`, `f781271`, `f3fae72`, `d79439f` | 2024 | Cascade de fixes mineurs (guards, redondance, updates incrementales). |

→ **Recommandation forte** : intégrer ce cluster Septentrio comme un **groupe de patches consécutifs** dans le sprint, pas en cherry-pick isolé. Les commits sont cohérents : appliquer `670690a` sans `784056a` ou `253bfb3` peut introduire des régressions transitoires.

#### Autres récepteurs (pour info, non utilisés actuellement)

##### u-blox (`src/rcv/ublox.c`)
| Commit | Description |
|---|---|
| `c671b39` | Table signaux X20 |
| `9859df3` | Galileo E5a F/NAV support |
| `b3bd537` | `rxmrawx` : détection bit half-cycle subtract |
| `8a10dd5` | Defaults `MAX_STD_CP` / `STD_SLIP` |
| `4df03b9` | SFRBX firmware F9P (Galileo nav length changée) |

##### Unicore (`src/rcv/unicore.c`) — parser nouveau
| Commit | Description |
|---|---|
| `572268e` | Nouveau parser binaire Unicore |
| `9673752` | QZSS L1CB (L1E) et L1S (L1Z) |
| `d771a40` | Support option `RCVSTDS` |

##### Novatel / Bynav / Tersus
| Commit | Description |
|---|---|
| `f2269ca` | Bynav M2 series dans novatel.c |
| `d9bd56d` | Bynav Galileo code E1B → E1BC |
| `f018851` | Tersus `bd2ephemb` |

#### Unicore (`src/rcv/unicore.c`) — parser nouveau ou refondu
| Commit | Description |
|---|---|
| `572268e` | Nouveau parser binaire Unicore + makefile updates |
| `9673752` | QZSS L1CB (L1E) et L1S (L1Z) |
| `d771a40` | Support option `RCVSTDS` |
| `edfc770` | Fix recording stddev observations |
| `1c9152a` | Fix message length |
| `11ae800` | `decode_obsvmb` retourne 0 si pas d'observations |

#### Novatel / Bynav / Tersus
| Commit | Description |
|---|---|
| `f2269ca` | **Support Bynav M2 series** dans novatel.c |
| `d9bd56d` | Bynav Galileo code E1B → E1BC |
| `f018851` | Support Tersus `bd2ephemb` |
| `8d658e5` | Comparaisons time tolérance vs zéro |

### 11.2 RTCM3 — nouveaux messages et fixes

| Commit | Description |
|---|---|
| `511410b` | **Support type 1013 (system parameters)** |
| `e3df221` | MSM : signal types **R3, R4, R6, L9** (BeiDou modernes) |
| `45079b0` | Messages 1001-1004, 1009-1012 : limite max satellites corrigée |
| `8e04ef6` | Nouveaux codes BeiDou en RTCM3→RINEX, freq 3 BDS B2A→B3, MSM sync bit fix sur erreur de length |
| `2e1540b` | `encode_type1012` : assignment fcn inutile retiré |
| `30482bf` | GLONASS désactivé dans build : divers fixes |
| `784e62d` | `rtcm2` : décodage observations complet |

### 11.3 RINEX — versions 3.04 / 3.05 / 4

| Commit | Date | Description |
|---|---|---|
| `0654233` | 2025-04-25 | **RINEX 3.05 navigation GLONASS et RTCM3** |
| `84224bb` | 2024-12-11 | `convrnx` : RINEX 3.05 et **codes RINEX 4** |
| `3097353` | 2025-05-29 | rtkconv : fix versions 3.05+ |
| `687a894` | 2025-06-04 | RINEX : lecture nav GLONASS corrigée |
| `ebf532a` | 2024-11-22 | RINEX clk 3.04 : offset système header corrigé |
| `d3dc227` | 2026-04-27 | RINEX header : parsing système fichier clk |

### 11.4 Tides solides — refonte significative

Avant 6c53 : tides en partie en Fortran ou external. Depuis :
| Commit | Description |
|---|---|
| `e00b170` | **`hardisp` traduit en C et inliné** |
| `8c9d5dc` | **`dehanttideinel` traduit en C et inliné** |
| `e7093aa` | Code de référence IERS `hardisp` importé |
| `817ec8a` | Library IERS mise à jour |
| `a425d7e` | **IERS secular mean pole mis à jour** (vrai impact PPP statique long-run) |
| `92f8a6d` | `tidedisp` utilise `ecef2pos` pour latitude plus précise |
| `5675351` | `tidecorr` en bitmask |
| `9a2f2ec`, `be9a673` | VMF1_HT subroutine fix (×2) |
| `8158789` | Import partiel SOFA C library pour sun/moon |

### 11.5 ERP / EOP / référentiels

| Commit | Description |
|---|---|
| `954b0cf` | `readerp` : support **IGS UT1-TAI offsets** |
| `c570df8` | qtapps : reconnaissance extensions `.EOF` `.ERP` |

### 11.6 Sortie solution (NMEA / KML / GPX / stat)

| Commit | Description |
|---|---|
| `7d7a3d7` | **NMEA GST sentence** (output) |
| `4994033` | `outnmea_gga` : ajout `refstationid` |
| `f361dac` | `outnmea_gsa` : respect du compteur de satellites |
| `180aadd` | `pos2kml` : option position moyenne unique + sortie CSV |
| `de71695` | `convgpx`, `convkml` : libère le buffer solution |
| `10fc00e` | rtkplot : reconnaît extension `.nma` comme NMEA |
| `24f0dc3` | `MAXSOLMSG` augmenté à 32768 (overflow) |
| `3524b5d` | `outsolstat` : OOB fix |
| `e6a3f09` | `outsolstat` : buffer agrandi |

### 11.7 Outils CLI annexes

#### convbin
| Commit | Description |
|---|---|
| `5bea1ca` | Support unicore dans usage |
| `89cd7e4` | Defaults : toutes fréquences |
| `8491151` | Time : utilise start/end si fournis |
| `7f67c28` | Init GLONASS FCN |
| `788504f` | Default time tolerance 0.0→0.005 (sync RTKCONV, évite duplications) |

#### str2str
| Commit | Description |
|---|---|
| `4d1057d` | **Support fichiers de log** |
| `1c178e7` | Messages par output stream |
| `cf4ab29` | `readcmd` : évite tailles constantes |
| `505a034` | Usage : exemple `-msg` avant `-out` |
| `70f31aa` | Fix null pointer deref si pas de log files |

#### Tous consapp
| Commit | Description |
|---|---|
| `02ec964` | Flag **`--version`** (utile pour vos releases) |
| `493d80e` | `--detach from console` (str2str + rtkrcv) |
| `b4202eb` | Codes de sortie normalisés |

### 11.8 Postpos / multi-file (utile rnx2rtkp)

| Commit | Description |
|---|---|
| `5964d8a` | Étend la fenêtre temps obs base pour interpolation |
| `1a47317` | Aligne header ref position |
| `b88dfee` | `procpos` : catch malloc failure (anti-crash) |
| `a1c9242` | Allocation pour tous les infiles en unit periods |
| `e995e50` | Évite over-allocation dans `ifile[]` |
| `87d9060` | `rtkinit` : évite duplication sans `rtkfree` |

### 11.9 Modes positionnement (rnx2rtkp + rtkrcv)

| Commit | Description |
|---|---|
| `6577aa2`/`1d1b7e9` | Mode **`Static-Start`** ajouté (manquait) |
| `4b519f3` | **Moving-base RTK** amélioré + baseline constraint indep + skip estimées initiales SPP |
| `1854c43`, `5cbcc16`, `ba05d40` | Bugs solutions backwards-only / fwd+bwd |
| `1574edf` | DGPS mode : fixes |
| `fa8e46c` | Cycle slip Doppler en backwards filter |

### 11.10 Build system — passage CMake (significatif pour vous)

Si vous voulez moderniser votre build :
| Commit | Description |
|---|---|
| `e82cd7d` | Support CMake initial |
| `8e62852` | Revert (instable) |
| `bb563a0` | **Re-add CMake support** |
| `fbf5688` | CMake : support modèle IERS |
| `ffe4d49` | Tests unitaires existants ajoutés au CMake |
| `14dd080` | CMake : tests inclus pour console apps |
| `3273ed9` | Build : passage **ANSI C → C99** |
| `a2cc5a9` | `_POSIX_C_SOURCE` 199506 → 200112L (POSIX 2001) |
| `7320527` | `#if 0` → `#ifdef RTK_DISABLED` (toggleable) |
| `6c88804` | Refactor `.gitignore` par sous-dossier |

→ **Recommandation** : si vous restez sur les Makefiles historiques, OK. Si votre équipe veut moderniser, CMake est mature **et inclut maintenant les tests unitaires** — gros levier pour votre stratégie doc-as-code + non-régression.

### 11.11 Tests unitaires (`test/utest/`)

| Commit | Description |
|---|---|
| `65cd81b` | **Tests unitaires activés** |
| `5b7516d` | Fix printout tests |
| `ffe4d49` | Tests existants intégrés CMake |
| `14dd080` | Tests pour console apps (CMake) |

→ Très utile pour votre `Δscore` de validation : vous pouvez ajouter des tests sur les patches du sprint 1, et l'infra CMake de tests est déjà disponible.

### 11.12 Robustesse / threading (au-delà des OOB déjà listés en §6.3)

| Commit | Description |
|---|---|
| `81c7df0` | `strtok` → **`strtok_r`** (thread-safe) |
| `1aa29c9` | `gmtime_r` (thread-safe) |
| `558048a` | `sbstropcorr` cache thread-safe |
| `810fa5a` | `time_str` non-réentrant supprimé → `time2str` |
| `5b023e9` | `time2str` : limite buffer 40 caractères |
| `fec7f29` | `satno2id` : taille buffer cohérente |
| `bf31bd6` | `scanf` : largeurs max strings |
| `9d63cbb` | Misc UDP stream fixes |
| `e194fd4` | NTRIP `rsp_ntripc` : check longueur basic auth |
| `d23b208` | `openserial` macOS : OOB high baud rates |
| `9e2dddf` | `openserial` : OOB potentiel `bs[]` |
| `04525c3` | `writeserial` : patch non-WIN32 |
| `94b9f68` | `decodeftppath` : null ptr deref |
| `7098ab7` | `readmembuf` : wrap avant compare write ptr |

### 11.13 Métadonnées et observations

| Commit | Description |
|---|---|
| `907f29d` | **Indices fréquence BDS : B1C avant B2ab** (ordre des colonnes RINEX, attention parser) |
| `ec7cf85` | Station info : ajout marker type, observer, agency (utile métadonnées) |
| `e854997` | rtkrcv : commandes `mark` et `mode` (logs marqueurs) |

### 11.14 Documentation — sample configs

| Commit | Description |
|---|---|
| `d359e1b` | Update sample config files |
| `702a6d0` | Exemple config u-blox F9P PPK |
| `ec9a81b` | Updates documentation in/out code, suppression params obsolètes |

### 11.15 Synthèse — angles à intégrer selon votre stack

| Si vous… | Regardez en priorité |
|---|---|
| Utilisez **Septentrio Mosaic X5** (récepteur projet) | **§11.1 cluster Septentrio (refonte parser + cascade fixes) — patches #14/#15 du sprint** |
| Évoluez vers d'autres récepteurs (u-blox, Unicore…) | §11.1 sections "autres récepteurs" |
| Convertissez du RTCM3 / RINEX 3.05+ | §11.2 + §11.3 (codes BeiDou modernes, GLO nav 3.05) |
| Faites du PPP statique long-run | §11.4 tides IERS mean pole + VMF1 fixes |
| Sortez du NMEA GST ou GGA avec refstationid | §11.6 |
| Voulez moderniser le build (CMake + tests) | §11.10 + §11.11 |
| Faites du multi-file post-process | §11.8 |
| Tournez en multi-thread (rtkrcv) | §11.12 (strtok_r, gmtime_r, sbstropcorr cache) |

---

*Synthèse établie sur 1298 commits (805 hors merges) entre `6c53aa2` et `28ad77c`. Périmètre projet = CLI uniquement. Sections 1-6 = analyse et audits. Section 7 = sprint 1. Section 8 = NFREQ=4 plan. Section 9 = doc-as-code (Astro). Section 10 = optim latence/CPU/RAM. Section 11 = autres améliorations (parsers récepteurs, RTCM3 / RINEX 3.05, tides/IERS, NMEA, CMake, tests unitaires, robustesse threading).*
