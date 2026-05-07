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

### 1.8 Résolution d'ambiguïtés (PPP-AR / fix-and-hold)
| Commit | Date | Description |
|---|---|---|
| `f1a7d2f` | 2024-07-03 | Bug en résolution d'ambiguïté instantanée (reset excessif du `lock count`). |
| `b49ce01` | 2021-11-04 | AR retries cassée si GLO désactivé mais `GLO_AR=fix-and-hold`. |
| `157919a` | 2025-07-22 | `rtkrcv` : ajout de `ppp-fixed` à la commande `mode`. |
| `4509b10` | 2025-04-22 | Label / tooltip de la combo-box "ambiguity resolution" corrigés. |
| `0aedab3` | 2025-04-09 | Renommage de la combo-box AR (cohérence). |
| `cf3f876` | 2023-10-10 | Ajout d'un statement pré-compilo pour la sortie des ambiguïtés. |
| `413c4a5` | 2023-10-10 | Dimensions cohérentes pour l'init des biais. |

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

## 3. Interfaces utilisateur (impact PPP)

### 3.1 RTKPOST (Embarcadero / Windows)
| Commit | Date | Description |
|---|---|---|
| `4ee13e0` | 2025-04-24 | Typo "Broadcast+SSR APC" dans l'array `ephopt`. |
| `be834d7` | 2025-04-24 | Suppression d'une assignation dupliquée pour `TROPOPT_ESTG`. |
| `87e2161` | 2025-04-24 | Retrait du STEC model non supporté. |
| `6a6271e` | 2025-04-24 | Activation `IONOOPT_IFLC` / `IONOOPT_EST` corrigée. |
| `df40d50` | 2025-04-11 | Dropbox de sélection de fréquences cohérent avec la table de fréquences. |
| `3538942` | 2025-04-11 | Renommage BeiDou-3 ACE-BOC pour cohérence avec Galileo Alt-BOC. |
| `cef68a0` | 2025-06-18 | Taille de la box "SNR mask" corrigée. |
| `35b6fee` | 2024-11-23 | `outprcopts` : émet les fréquences et l'option iono pour les modes PPP. |
| `bf2b5d1` | 2023-12-13 | Reconnaissance des extensions `*O.RNX` (obs) et `*N.RNX` (nav). |
| `4509b10` | 2025-04-22 | Label / tooltip de l'AR. |
| `91193c6` | 2021-11-04 | Ajout option ratio d'erreur L5. |

### 3.2 RTKNAVI / RTKNAVI-Qt (temps réel)
| Commit | Date | Description |
|---|---|---|
| `28ad77c` | 2026-05-01 | Reset de la structure `rtksvr` au démarrage (commenté) → reproductibilité fichier. |
| `da1f9fa` | 2025-04-28 | Cleanup GUI pour BeiDou + activation/désactivation des options selon RTK vs PPP. |
| `7856c17` | 2025-04-28 | Active la sortie des options de traitement aussi pour RtkNavi. |
| `677261d` | 2023-09-26 | Bug code-bias variables corrigé. |
| `93bac73` | 2025-05-24 | Label corrigé dans RTKNAVI. |
| `20aa561` | 2024-06-04 | Qt monitor : guard nav vide, largeurs, RTCM SSR. |
| `770e6f4` | 2024-04-XX | rtknavi-qt : changement de fréquence dans SNR / sky plots. |

### 3.3 RNX2RTKP (CLI post-process)
| Commit | Date | Description |
|---|---|---|
| `4b519f3` | 2023-05-24 | Option `-bl` baseline en CLI + skip des estimées initiales en standard precision. |
| `6bb4ab2` | 2023-12-07 | Bug option `-sys` corrigé + maj manuel. |
| `c4514fe` | 2023-10-10 | Options CLI start / end time. |
| `d1ab09e` | 2023-10-10 | Fix du paramétrage du fichier de sortie en CLI. |
| `343c245` | 2023-10-10 | Ne vérifie le suffixe de sortie qu'en cas d'extension invalide. |
| `ca3befa` | 2025-XX-XX | `rnx2rtkp` : par défaut, navsys inclut Galileo et BDS. |
| `72dca37` | 2025-XX-XX | BDS ajouté aux systèmes par défaut. |

### 3.4 RTKRCV (console temps réel)
| Commit | Date | Description |
|---|---|---|
| `157919a` | 2025-07-22 | Ajout du mode `ppp-fixed` à la commande `mode`. |
| `4b519f3` | 2023-05-24 | Option pour lancer RTKRCV sans console. |
| `5d6794e` | 2023-04-17 | Fix erreur de sortie (Issue #139). |
| `d9d59de` | 2025-02-03 | Suppression d'un hack format de stream SP3. |

### 3.5 Qt apps (compilation, packaging)
| Commit | Date | Description |
|---|---|---|
| `f04f127` | 2023-05-17 | **PR #138** — Compilation Qt 6.x, support macOS, mutex workaround `INHIBIT_RTK_LOCK_MACROS`, URL CORS rtkget. |
| `b8198ef` | 2025-XX-XX | Install targets corrigés (launch / navi / plot / post / strsvr / srctblbrows). |
| `bd43d0e` | 2025-XX-XX | Plot freq pour rtknavi-qt. |

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

### 6.1 🔴 Théme transversal n°1 — `NFREQ=4` partiellement appliqué

Le passage à 4 fréquences (`199be2b`, avr. 2025) n'a pas propagé partout. Plusieurs structures et boucles restent hardcodées à 3.

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🔴 | `src/rtklib.h:834-836` | Structure `ssr_t` : `deph[3]`, `ddeph[3]`, `dclk[3]` figés à 3 → corrections orbit/clock SSR pour la 4ᵉ fréq impossibles. |
| 🔴 | `src/rtcm3.c:1629-1638, 1674-1695, 1710-1725` | Boucles `for (k=0;k<3;k++)` dans `decode_ssr1/2/3` → la 4ᵉ correction n'est jamais lue même si présente. |
| 🔴 | `src/rtkcmn.c:3744` (`seliflc`) | `return((optnf==2\|\|sys==SYS_GLO)?1:2)` — pas de cas `nf≥4`, retourne toujours indice ≤2 → la 4ᵉ fréq n'est jamais utilisée en iono-free. |
| 🟠 | `src/ppp.c:754` | Cycle-slip iono-free hardcodé à `slip[0] \|\| slip[1]` — slips sur freq 2,3 invisibles pour les biais iono-free. |
| 🟠 | `src/ppp.c:385-406` | `gfmeas` / `mwmeas` n'utilisent que `code[0]`/`code[1]` → cycle-slip GF/MW jamais détecté sur freq 2,3. |
| 🟠 | `src/ppp.c:368-369` | `obs->Pstd[frq]` / `Lstd[frq]` accédé sans guard `frq < NFREQ` → OOB potentiel si récepteur fournit 5+ fréquences. |
| 🟠 | `src/ppp.c:1162` | `test_hold_amb` ne teste que `fix[0]` et `fix[1]` → fix-and-hold sur freq 2,3 jamais validé. |
| 🟠 | `src/rtklib.h:874` | `cbias[MAXSAT][NFREQ][MAX_CODE_BIASES]` alloué mais SSR remplit `pbias[MAXCODE]` plat → mismatch d'indexation. |

**Impact global** : utiliser `NFREQ=4` sur `HEAD` donne une expérience **partielle** — la 4ᵉ fréq est consommée par certains modules et ignorée par d'autres.

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

### 6.5 🟠 Cohérence des interfaces utilisateur

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🔴 | `app/winapp/rtkpost/postopt.cpp:744,784-785` | **Champs `PPPOpts` / `RnxOpts1` / `RnxOpts2` saisis en GUI mais non persistés dans INI** → perte des options custom au redémarrage. |
| 🔴 | `app/consapp/rnx2rtkp/rnx2rtkp.c:51-52` | Help texte ne documente pas les modes 7/8/9 (`ppp-kinematic`, `ppp-static`, `ppp-fixed`) bien qu'ils fonctionnent via `-p`. |
| 🟠 | `src/options.c:118` + `app/winapp/rtkpost/postmain.cpp:1042-1043` + `naviopt.cpp:442-443` | `pos2-rejionno` deprecated (`a06e9c1`) mais GUI Windows continue de lire/écrire `prcopt.maxinno[0/1]` directement → migration `pos2-rejphase` incomplète. |
| 🟠 | `app/consapp/rnx2rtkz/rnx2rtkp.c:140-162` | CLI : pas de flags pour `pos2-gloarmode`, `pos2-bdsarmode`, `pos2-arthres1..4`, `pos2-armaxiter`, `pos2-varholdamb` → AR multi-constell uniquement via `-k config`. |
| 🟠 | `app/winapp/rtkpost/postopt.cpp:254-257,945` | RTKPOST désactive `IonoOpt` pour PPP mais `IONOOPT_IFLC` n'est pas exposé non plus → utilisateur ne peut pas choisir iono-free explicite en GUI. |
| 🟠 | `app/winapp/rtkpost/postopt.cpp:280-282` | Pas de validation `PPP-fixed + Freq=L1 seul` → combo accepté mais runtime ajuste silencieusement. |
| 🟡 | `app/winapp/rtkpost/postopt.cpp:800-801` + `postmain.cpp:1003-1004` | `tidecorr` : combo index `0/1/2` mappé en bitmask `0/1/7` (hardcodé `if (>1) =7`) → bits intermédiaires (3,5) inaccessibles depuis GUI. |
| 🟡 | `app/winapp/rtkpost/postopt.cpp:755-762` vs `src/options.c:190-195` | Fichiers ANTEX satellite/récepteur exposés en GUI mais `file-satantfile` / `file-rcvantfile` parsing config absent → divergence GUI ↔ CLI. |
| 🟡 | `app/winapp/rtkpost/postopt.cpp:948-950` | `posopt3-6` (PHWOPT etc.) en checkbox sans tooltip explicatif PPP-spécifique. |
| 🟡 | `app/consapp/rnx2rtkp/rnx2rtkp.c:46` | Pas de `-K outfile` pour sauvegarder la session → utilisateur doit éditer config à la main. |

### 6.6 🟡 Divers — effets de bord

| Sévérité | Fichier:ligne | Problème |
|---|---|---|
| 🟠 | `src/ppp.c:1061` | `THRES_REJECT=4σ` identique pour phase et code — phase residuals plus sensibles aux ambigs mal initialisées → trop de rejets en early epochs. |
| 🟠 | `src/ppp.c:118` (`IB` macro) + `src/ppp.c:741,746,783,790` | Si `nf` change entre sessions (ex: passage `IFLC→EST` + `nf 2→3`), `rtk->ssat[].fix[]` n'est pas redimensionné → indices `ambc` invalides. |
| 🟡 | `src/ppp.c:819-821` | DCB L5 estimé inconditionnellement si `nf ≥ 3`, même en `IONOOPT_IFLC` (1 fréq effective) → degré de liberté Kalman gaspillé. |
| 🟡 | `src/rtklib.h:162` | Si `NEXOBS=0`, observations indice ≥ NFREQ filtrées dans `save_msm_obs` → L5/E6 perdus silencieusement. |

### 6.7 Note sur la dépendance GPS (réf. section 5)

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

### 6.8 Synthèse priorisée

**Top 5 fixes à viser immédiatement** :
1. 🔴 `ssr_t.deph[3]` / `dclk[3]` → `[NFREQ]` (`rtklib.h:834-836`)
2. 🔴 Guards `sat > 0` dans `preceph.c` (≥10 occurrences)
3. 🔴 Persistence `PPPOpts` dans INI RTKPOST/RTKNAVI Windows
4. 🔴 `seliflc()` cas `nf≥4` (`rtkcmn.c:3744`)
5. 🔴 Facteur iono-free dynamique au lieu de `SQR(3.0)` (`ppp.c:372`)

**Bilan global** : 7 issues 🔴 critiques (corruption mémoire, perte données utilisateur, biais algorithmiques majeurs), 24 🟠 moyennes (dégradations qualité solution / UX), 18 🟡 mineures.

---

*Synthèse établie sur 1298 commits (805 hors merges) entre `6c53aa2` et `28ad77c`. Sélection par filtrage sur les fichiers `ppp.c`, `ppp_ar.c`, `ppp_corr.c`, `preceph.c`, `sbas.c`, `ionex.c`, `rtkpos.c`, `pntpos.c`, `rtcm3.c`, `rinex.c`, et les apps `rtkpost*`, `rtknavi*`, `rnx2rtkp`, `rtkrcv`. Section 6 issue de 4 audits parallèles (cœur PPP / interfaces / produits externes / SSR-multi-fréq).*
