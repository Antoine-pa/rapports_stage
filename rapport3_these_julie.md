---

# RAPPORT DÉTAILLÉ — Thèse de Julie Dumas
## « Représentation dynamique de la liste des copies pour le passage à l'échelle des protocoles de cohérence de cache »

**Auteure :** Julie Dumas  
**Direction :** Frédéric Pétrot (TIMA), co-encadrant Éric Guthmuller (CEA-Leti)  
**Soutenue le :** 13 décembre 2017 — Université de Grenoble  
**Jury :** F. de Dinechin (président), D. Etiemble (rapporteur), G. Sassatelli (rapporteur), Q. Meunier (examinateur)

---

## 1. Résumé général et contributions

### 1.1 Sujet exact

Le sujet de la thèse est la **représentation compacte et dynamique de la liste des copies** (*sharing set* ou *sharers*) dans les protocoles de cohérence de caches à répertoire, dans le contexte des architectures **manycores** (plusieurs dizaines à milliers de cœurs sur puce) à mémoire partagée cohérente. 

Le problème central est le suivant : dans un protocole à répertoire, chaque entrée du répertoire doit stocker l'ensemble des caches L1 qui possèdent une copie d'une ligne de cache donnée. La représentation naïve utilise un champ de bits complet de taille O(N) où N est le nombre de cœurs. Pour 1024 cœurs, ce champ de bits est deux fois plus grand que la ligne de cache elle-même (128 octets vs 64 octets), ce qui est prohibitif.

### 1.2 Contributions principales

La thèse propose **deux contributions majeures** :

1. **PyCCExplorer** — Un simulateur haut niveau pour la cohérence de caches, basé sur l'injection de traces. Il permet d'évaluer et comparer rapidement (380× plus vite que gem5 avec Ruby) différentes représentations de la liste des copies, en modélisant uniquement les états stables du protocole MOESI et le cache L2.

2. **DCC (Dynamic Coherent Cluster)** — Une nouvelle représentation dynamique de la liste des copies combinant :
   - Un champ de bits pour un **rectangle cohérent** (de forme et position variables dans le réseau 2D en grille),
   - Une **liste chaînée** limitée pour les copies hors du rectangle,
   - Un mode **diffusion** dégradé déclenché quand la liste chaînée est saturée.

### 1.3 Contexte et motivation

Les architectures manycores sont composées de dizaines à centaines de cœurs sur une même puce. Le modèle mémoire dominant est la mémoire partagée (SMP). Maintenir la cohérence des caches dans ce contexte nécessite un protocole efficace. Les protocoles basés sur l'**espionnage** (broadcast) sont inutilisables à grande échelle car ils saturent le réseau. Les protocoles à **répertoire** ciblés envoient des messages uniquement aux caches concernés, mais leur taille mémoire explose avec N.

Deux observations empiriques motivent DCC :
- **93,4 % des lignes de cache** d'une architecture 64 cœurs sont partagées par au plus 8 cœurs (mesure sur 13 applications PARSEC + Splash2).
- Les systèmes d'exploitation placent les tâches communicantes à **proximité topologique** (phénomène de localité spatiale des partages).

---

## 2. État de l'art présenté (Chapitre 3)

### 2.1 Protocoles basés sur l'espionnage

#### 2.1.1 Flexible Snooping [Stuecheli, Steffan, Torellas, 2006]
Dans un réseau en anneau, trois modes de communication sont définis : *lazy* (requête transmise de voisin en voisin, réponse au premier hit), *eager* (transmission immédiate + interrogation en parallèle de tous), et *oracle* (envoi direct au cache possédant la donnée). Un **prédicteur** choisit le mode optimal à chaque nœud. Limitation : non extensible à une topologie en grille 2D.

#### 2.1.2 In-Network Coherence Filtering (INCF) [Agarwal, Peh, Jha, 2009]
Des **filtres** au niveau des routeurs maintiennent une table indiquant dans quelles directions la donnée est partagée. Limitation : la taille des tables de filtres et le nombre de liens actifs augmentent avec le nombre de cœurs → mauvaise scalabilité.

**Conclusion générale :** les protocoles basés sur l'espionnage, même avec filtres, ne passent pas à l'échelle des manycores.

### 2.2 Protocoles avec répertoire limité

#### 2.2.1 Ackwise [Kurian, Miller, Psota et al., 2010]

Format du répertoire :
```
State | G (1 bit) | Keeper | Sharer_1 | ... | Sharer_{k-1}
```
- **G = 0** : mode précis, les k copies sont mémorisées par leur CID.
- **G = 1** : mode diffusion, seul un **compteur** du nombre de copies est mémorisé dans le champ `Sharer_k`. Les messages d'invalidation sont envoyés en broadcast mais le répertoire attend exactement ce nombre d'acquittements.
- Le **keeper** est le cache responsable de la cohérence et qui répond aux lectures.
- Taille de la représentation : ⌈log₂(N)⌉ × k bits. Pour N=64, k=5 → 31 bits.
- Exploré pour k entre 2 et 64 ; k=5 est un bon compromis (seuil optimal trouvé).

#### 2.2.2 Coarse Bit-Vector Directory [Gupta, Weber, Mowry, 1990]
Première représentation dynamique : mémorise quelques CIDs, et quand la limite est atteinte, bascule sur un champ de bits à grain grossier (chaque bit = un groupe de cœurs). Limitation : groupes définis à la conception, pas de flexibilité, problème de faux partages de groupes.

#### 2.2.3 Virtualisation des CIDs / Coherence Domain Restriction [Fu, Nguyen, Wentzlaff, 2015]
Utilise un champ de bits virtuel de taille k avec une **table de correspondance** par page entre bits virtuels et cœurs physiques. Mode dégradé en broadcast quand la table est pleine. Limitation : taille des tables de correspondance croît avec N.

### 2.3 Protocoles de cohérence à répertoire dynamique

#### 2.3.1 Liste chaînée [Thapar, Delagi, Flynn, 1993]

Format :
```
Entrée répertoire : State | Tag | Sharer_1 | Next_ptr
Entrée tas : CID | Next_ptr
```
La liste chaînée est mémorisée dans un **tas partagé** entre toutes les lignes de cache. Quand le tas est plein ou la liste atteint le seuil T, passage en diffusion. Variante **RWT** [Liu et al., 2015] avec write-through pour les lignes en mode cohérent.

Paramètres à ajuster lors de l'exploration architecturale : taille du seuil T et nombre d'entrées dans le tas H. Problème : le parcours de la liste ajoute de la latence.

#### 2.3.2 Segment Directory [Choi, Park, 1999]
Le répertoire contient un segment pointer (identifiant d'origine d'un segment) et un segment vector (champ de bits limité pour ce segment). Les segments sont alignés sur des puissances de 2. Extension : **Pool Directory** [Shukla, Chandrasekaran, 2015] : deux structures, l'une avec CIDs (comme Ackwise) et l'autre avec segments, stockées dans un tas partagé. Limitation : alignement forcé sur puissances de 2.

#### 2.3.3 Sponge Directory [Zhang et al., 2014]
Répertoire organisé comme un cache associatif par ensemble, avec des voies divisées en niveaux à type dynamique. Types possibles :
- **Invalid** : entrée inutilisée
- **Pointer Head** : tête d'un objet CID
- **Pointer Body** : corps d'un objet CID avec pointeur vers la tête
- **Vector Head** : tête d'un champ de bits complet
- **Vector Body** : corps d'un champ de bits complet

Un objet CID peut occuper moins de niveaux qu'un objet Vector, permettant un partage flexible. Originalité : deux représentations coexistent avec allocation dynamique.

#### 2.3.4 SPACE [Zhao, Shen, Du, 2010]
Analyse des **schémas de partage** récurrents. Pour 16 cœurs, 200 schémas représentent 80% des accès. Un tas partagé contient les schémas (champ de bits complet) et chaque entrée du répertoire pointe vers un schéma. Quand le tas est plein, fusion des schémas les plus proches (distance de Hamming). Limitation : taille du tas croît quadratiquement avec N.

### 2.4 Autres approches notables

#### 2.4.1 Scalable Interface Coherence [James, Laundrie, Gjessing, Sohi, 1990]
Chaque entrée du répertoire contient un pointeur vers le prochain cache L1 (liste chaînée distribuée sur les caches). Coût : O(log N) bits. Problème : la liste n'est pas ordonnée → latence de parcours élevée, risque de deadlocks dans le réseau.

#### 2.4.2 In-Network Cache Coherence (INCC) [Eisley, Peh, Shang, 2006]
Arbres virtuels de cohérence distribués dans les routeurs. Chaque entrée du répertoire local d'un routeur contient : directions (N, S, E, W), Root, Busy, Req, Valid. Avantage : taille d'entrée indépendante de N. Problème : profondeur des répertoires croît avec N, risques de deadlocks. Implémentation matérielle dans [Bernard, Nguyen, Guthmuller, Durand, 2013].

#### 2.4.3 Classifications des données [Cuesta, Ros, Gómez et al., 2011]
Désactivation de la cohérence pour les lignes privées (75% des lignes). Support OS nécessaire pour détecter les lignes privées au niveau pages mémoire. Résultats limités à 8 tuiles × 2 cœurs.

**Spatiotemporal Coherence Tracking** [Alisafaee, 2012] : utilise des entrées de répertoire pour des régions temporairement privées (plusieurs lignes consécutives sous un seul CID). Expérimentations limitées à 16 cœurs.

#### 2.4.4 Fenêtres temporelles — Tardis [Yu, Ding, 2015]
Chaque cache reçoit un droit de lecture/écriture pendant une fenêtre de temps définie par un compteur. Avantage : taille de l'entrée indépendante de N. Difficulté : dimensionnement des fenêtres temporelles.

### 2.5 Tableau récapitulatif de l'état de l'art

| Protocole | Type | Taille entrée | Passage à l'échelle |
|-----------|------|---------------|---------------------|
| Snooping complet | Espionnage | O(1) | Mauvais (trafic) |
| Flexible Snooping | Espionnage+filtre | O(N) par nœud | Anneau uniquement |
| INCF | Espionnage+filtre | O(N) par routeur | Mauvais |
| Full bit-vector | Répertoire précis | O(N) | Mauvais (taille) |
| Ackwise | Répertoire limité | O(k·log N) | Moyen |
| Coarse bit-vector | Répertoire limité+groupes | O(k + S) | Inflexible |
| CDR | Répertoire limité+virtuel | O(k + table) | Moyen |
| Liste chaînée | Dynamique | O(log N + log H) | Bon (avec tas) |
| Segment directory | Dynamique | O(seg+ptr) | Moyen (alignement) |
| Sponge directory | Dynamique | Voies partagées | Bon (complexe) |
| SPACE | Dynamique+motifs | O(log H) | Mauvais (grand N) |
| SIC | Liste distribuée | O(log N) | Deadlocks |
| INCC | Arbre distribué | O(4+2) bits | Saturation réseau |
| Tardis | Fenêtres temporelles | O(compteurs) | Incertain |
| **DCC** | **Dynamique+rectangle** | **O(log N + C + log H)** | **Bon** |

---

## 3. Protocole DCC — Description complète (Chapitre 4)

### 3.1 Concept fondamental

DCC repose sur l'idée que les caches partageant une ligne de cache ont tendance à être **géographiquement proches** dans le réseau 2D en grille, car l'OS optimise le placement. L'architecture cible est une **grille 2D de taille √N × √N** (ex : 8×8 pour 64 cœurs).

La liste des copies est partitionnée en deux sous-ensembles :
1. Les copies situées **dans un rectangle cohérent** (de forme et position variables), encodées comme un champ de bits de taille C.
2. Les copies **hors du rectangle**, mémorisées dans une liste chaînée de taille ≤ T dans un tas partagé.

Si la liste chaînée atteint T éléments ou si le tas est plein → passage en **mode diffusion** (imprécis).

### 3.2 Structure de l'entrée du répertoire

```
field : | broadcast | origin  | shape | bit vector | pointer |
size  : | 1 bit     | log₂(N) | I bits| C bits     | log₂(H) |
```

Détail de chaque champ :

| Champ | Taille | Rôle en mode précis | Rôle en mode diffusion |
|-------|--------|---------------------|------------------------|
| **broadcast** | 1 bit | 0 (mode précis) | 1 (mode diffusion) |
| **origin** | log₂(N) bits | CID du coin bas-gauche du rectangle (origine en coordonnées absolues dans la grille) | CID du keeper (cache responsable de la cohérence) |
| **shape** | I bits | Encode la forme du rectangle (ex : 1×16, 2×8, 3×5, 4×4, 5×3, 8×2, 16×1 pour C=16) | Inutilisé (réutilisé) |
| **bit vector** | C bits | Champ de bits encodant les copies présentes dans le rectangle (bit i = 1 si le cœur à la position relative i dans le rectangle a une copie) | Partie haute du compteur de copies |
| **pointer** | log₂(H) bits | Pointeur vers la première entrée du tas (liste chaînée) ; NULL si liste vide | Partie basse du compteur de copies |

**Exemple numérique pour N=64, C=16, H=128 :**
- broadcast : 1 bit
- origin : log₂(64) = 6 bits
- shape : pour C=16 et une grille 8×8, 5 formes valides → 3 bits (⌈log₂(5)⌉)
- bit vector : 16 bits
- pointer : log₂(128) = 7 bits
- **Total : 33 bits par entrée de répertoire**

### 3.3 Structure du tas partagé

Le tas partagé contient H entrées (H est un paramètre), chacune composée de :
```
Entrée de tas : | CID (log₂(N) bits) | next_ptr (log₂(H) bits) |
```

Pour N=64, H=128 : chaque entrée = 6 + 7 = 13 bits.  
Coût total du tas : 128 × 13 = 1664 bits.  
Rapporté à un cache L2 de 4096 lignes : 1664/4096 ≈ **0.41 bit par ligne**, négligeable.

**Pointeur nul/sentinel :** une entrée de tas où le pointeur pointe sur lui-même indique la fin de la liste chaînée (sentinel).

### 3.4 Paramètres du protocole

| Paramètre | Notation | Valeur choisie (64 cœurs) | Description |
|-----------|----------|--------------------------|-------------|
| Taille du rectangle | C | **16** | Aire maximale (puissance de 2) du rectangle de cohérence |
| Seuil liste chaînée | T | **4** | Nombre maximum d'éléments dans la liste chaînée |
| Taille du tas | H | **128** | Nombre d'entrées dans le tas partagé |
| Taille de la grille | √N × √N | 8×8 | Architecture cible |

### 3.5 Encodage du rectangle

Le rectangle est défini par :
- Son **origine** (x₀, y₀) : coordonnées du coin bas-gauche dans la grille globale (encodé sur log₂(N) bits = CID du cœur à cet emplacement).
- Sa **forme** (width × height) : encodée par un identifiant de forme sur I bits.

Pour C=16, les formes valides dans une grille 8×8 sont :
- 2×8, 8×2 (14 origines possibles)
- 4×4 (25 origines)
- 1×16, 16×1 (impossibles dans 8×8, sortent de la grille)
- 3×5 = 15 (non puissance de 2, possible mais non retenu dans la version simplifiée)
- 5×3 = 15 (idem)

Le nombre de formes à encoder est donc réduit, nécessitant 3 bits (5 formes valides maximum).

Le champ de bits encode les copies de manière relative à l'origine : le bit `i` correspond au cœur dont les coordonnées dans le repère local du rectangle sont `(i mod width, i / width)`. Le bit de poids fort est le premier.

**Exemple (Figure 4.1 et 4.3) :**
- Grille 8×8, rectangle 4×4 avec origine (1,1) (CID=9)
- Copies dans le rectangle : (1,1)=[bit 0], (4,1)=[bit 3], (2,2)=[bit 5], (4,3)=[bit 11], (3,4)=[bit 14]
- Bit vector : `0b1001010000010010` (bits 0,3,5,11,14 à 1)
- Copies hors rectangle : CID=55 (coordonnées (7,6)) et CID=58 (coordonnées (2,7)) → dans la liste chaînée
- Entrée de tas à l'adresse 0b10100 : {CID=55, ptr=0b10110}
- Entrée de tas à l'adresse 0b10110 : {CID=58, ptr=0b10110} (sentinel : pointe sur lui-même)

### 3.6 Mode diffusion (broadcast)

Déclenché quand :
- La liste chaînée tente d'atteindre T+1 éléments (ajout impossible car liste pleine), **ou**
- Le tas partagé n'a plus d'entrées libres.

À la transition vers le mode diffusion :
1. Le champ `broadcast` passe à 1.
2. Toutes les entrées de la liste chaînée sont libérées (retournées au tas).
3. Le champ `origin` devient le CID du **keeper** (responsable de la cohérence). Dans l'implémentation de la thèse, le keeper est la **première copie dans le champ de bits** au moment de la transition.
4. Les champs `shape`, `bit vector`, et `pointer` sont réutilisés pour mémoriser le **compteur de copies**.

En mode diffusion, lors d'une requête de lecture exclusive (invalidation) :
- Un message broadcast est envoyé à tous les caches L1.
- Le répertoire attend un nombre d'acquittements égal au compteur de copies.

### 3.7 Algorithmes de placement du rectangle

#### 3.7.1 Algorithme idéal

À chaque accès (ajout ou suppression d'une copie), essaye **tous les rectangles possibles** de toutes les formes et origines, et sélectionne celui qui couvre le plus de copies. C'est la borne maximale de performance.

Pour N=64 (grille 8×8) et C=16 : 87 cas à essayer (7+7 pour 2×8 et 8×2, 24+24 pour 3×5 et 5×3, 25 pour 4×4). Le nombre de cas à tester croît rapidement avec N → trop coûteux en matériel pour N>64.

**Variante "idéal dégradée"** : algorithme idéal mais avec seuil T et tas fini. Utilisable en simulation mais reste trop coûteux pour une implémentation matérielle directe.

**Variante "idéal sans limitation"** : pas de seuil, tas infini. Donne la borne absolue supérieure des performances, sans aucun message de diffusion généré.

#### 3.7.2 Algorithme premier touché

Processus :
1. Premier accès : rectangle de taille **1×1** centré sur le premier cœur.
2. Accès suivants : tentative d'**agrandissement** du rectangle pour englober le nouveau cœur. Si l'aire résultante ≤ C, le rectangle est mis à jour.
3. Si impossible : ajout dans la liste chaînée. Si liste chaînée pleine → mode diffusion.

Propriété clé : **conservateur** — une copie admise dans le rectangle y reste définitivement (le rectangle ne peut que grandir).  
Problème : si le premier cœur (initialisation) est éloigné des cœurs de calcul, le rectangle sera mal placé et les cœurs de calcul se retrouveront dans la liste chaînée.

#### 3.7.3 Algorithme combinatoire (implémentable en matériel)

C'est l'algorithme retenu pour l'implémentation matérielle. Il est basé sur un **bloc combinatoire** nommé **bloc tiling** qui s'exécute en un seul cycle d'horloge.

**Entrées du bloc tiling :**
- n coordonnées de copies (sous forme de quadruplets (xmin, ymin, xmax, ymax))
- La première entrée peut représenter le rectangle courant (avec son "poids" = nombre de copies qu'il contient)
- Les autres entrées sont les éléments de la liste chaînée + le nouveau cœur demandeur

**Modes de fonctionnement :**

*Mode optimal* (s ≤ n - 2, où s = nombre total de copies) :
- Chaque entrée = coordonnées d'une seule copie (xmin=xmax, ymin=ymax).
- L'algorithme cherche le meilleur rectangle parmi toutes les combinaisons.
- Résultat identique à l'algorithme idéal.

*Mode sous-optimal* (s + 2 > n) :
- La première entrée = rectangle courant (xmin, ymin, xmax, ymax) avec poids w = nombre de copies dans le rectangle.
- Les n-1 autres entrées = copies de la liste chaînée + nouvelle copie.
- L'algorithme cherche le meilleur rectangle parmi les combinaisons.

La condition pour que le mode sous-optimal soit utile : **n ≥ T + 2** (1 pour le rectangle, T pour la liste chaînée, 1 pour la nouvelle copie). Avec T=4 → n ≥ 6.

### 3.8 Algorithme combinatoire — Description détaillée du bloc tiling

Le bloc tiling est composé de 4 sous-blocs en pipeline combinatoire :

#### 3.8.1 Bloc de calcul des rectangles

Pour chaque **combinaison** de k entrées parmi n (∀k ∈ [2, n]), calcule le rectangle englobant minimal :
```
xmin = min(e.xmin ∀e ∈ combi)
xmax = max(e.xmax ∀e ∈ combi)
ymin = min(e.ymin ∀e ∈ combi)
ymax = max(e.ymax ∀e ∈ combi)
```
Nombre total de blocs : ∑ₖ₌₂ⁿ C(n,k)  
Complexité en comparateurs pour ce bloc : 4×(k-1) par bloc de calcul.

#### 3.8.2 Bloc d'élagage (pruning)

Pour chaque rectangle englobant calculé, teste si :
```
(xmax - xmin) × (ymax - ymin) ≤ C
```
Deux implémentations possibles :
1. **Avec multiplication** : calcule l'aire puis compare à C.
2. **Avec LUT** : table précalculée de toutes les paires (Δx, Δy) valides (LUT symétrique, optimisée en triant Δx≤Δy avec comparateur+2 mux).

La version LUT donne de meilleures fréquences (1.33 GHz vs 1.21 GHz pour 6 entrées).

#### 3.8.3 Bloc de sélection

Sélectionne le rectangle qui englobe le plus de copies parmi ceux validés par l'élagage, en utilisant un **encodeur de priorité** de ∑ₖ₌₂ⁿ C(n,k) entrées.

En mode sous-optimal, un **deuxième encodeur de priorité** parallèle sélectionne le meilleur rectangle **qui contient également le rectangle courant** (première entrée). Le poids w est additionné pour connaître le nombre exact de copies. Le meilleur des deux résultats est retenu.

Si aucune combinaison de k≥2 entrées ne satisfait la contrainte d'aire :
- Mode optimal : choisit une seule copie comme rectangle 1×1.
- Mode sous-optimal : le rectangle courant est conservé.

#### 3.8.4 Bloc de génération de sortie

Transforme l'index de la combinaison gagnante en liste de copies (champ de bits dans le repère local du rectangle). Sortie : champ de bits de C bits + liste des copies non placées (→ liste chaînée).

### 3.9 Algorithme combinatoire complet (Algorithme 1 de la thèse)

```
Entrées: s (nb copies), S (ensemble coordonnées), c (coord. rectangle actuel), w (poids rectangle)
Résultat: ensemble des copies dans le rectangle

1. comb = toutes les sous-ensembles de S de taille k, pour k ∈ [2, n], n=|S|
2. Pour i = 1 à |comb| :
   // Calcul des rectangles
3.   xmin ← min(e.xmin ∀e ∈ combi)
4.   xmax ← max(e.xmax ∀e ∈ combi)
5.   ymin ← min(e.ymin ∀e ∈ combi)
6.   ymax ← max(e.ymax ∀e ∈ combi)
   // Élagage
7.   Δx ← xmax - xmin
8.   Δy ← ymax - ymin
9.   Si Δx × Δy ≤ C :
10.    comb_ok ← comb_ok ∪ {combi}
// Sélection (cas général)
11. Trouver comb_best tel que |comb_best| ≥ |combi| ∀combi ∈ comb_ok
// Cas sous-optimal
12. Si s + 2 > n :
13.   Trouver comb_bestc tel que |comb_bestc| ≥ |combi| ∀combi ∈ comb_ok avec c ∈ combi
14.   Si |comb_best| ≤ |comb_bestc| + w - 1 :
15.     comb_best ← comb_bestc
// Fallback si aucune combinaison valide
16. Si comb_best = ∅ :
17.   Si s + 2 > n (sous-optimal) :
18.     comb_best ← c
19.   Sinon :
20.     comb_best ← un élément quelconque de S
```

### 3.10 Coût matériel théorique du bloc tiling

En fonction du nombre d'entrées n :

| Ressource | Formule | Pour n=6 |
|-----------|---------|---------|
| Comparateurs | (2n-3)·2ⁿ - n + 3 | 1 467 |
| Soustractions | 2ⁿ⁺¹ - 2(n+1) | 114 |
| Multiplications ou LUTs | 2ⁿ - n - 1 | 57 |

La complexité est **exponentielle** en n → minimiser n est crucial. Avec T=4 et n=6, le coût reste raisonnable.

### 3.11 Synthèse matérielle du bloc tiling

**Technologie :** 22nm FDSOI 22FDX GlobalFoundries, coin lent (0.72V, 125°C, sans polarisation substrat).  
**Outil :** Synopsys Design Compiler.  
**Coordonnées d'entrée :** 5 bits (= 1024 cœurs).  
**Registres d'E/S** inclus dans les mesures (négligeables).

| Entrées n | Fréquence max (LUT) | Fréquence max (mult.) | Aire (LUT) | Aire (mult.) |
|-----------|--------------------|-----------------------|------------|--------------|
| 2 | ~2.0 GHz | ~1.9 GHz | ~2 000 μm² | ~2 500 μm² |
| 4 | ~1.7 GHz | ~1.5 GHz | ~4 500 μm² | ~7 000 μm² |
| **6** | **1.33 GHz** | **1.21 GHz** | **7 635 μm²** | **10 938 μm²** |
| 8 | ~1.1 GHz | ~0.95 GHz | ~40 000 μm² | > 40 000 μm² |

Comparaison : un cache L2 de 256 ko en même technologie a une fréquence de **1.25 GHz** et une surface de ~450 000 μm².

**Conclusion :** avec 6 entrées (LUT), le bloc tiling est compatible en fréquence avec un cache L2, et n'occupe que **7635/450000 ≈ 1.7%** de la surface du cache L2.

---

## 4. Modèle de protocole de cohérence — États et transitions

La thèse ne décrit pas une machine à états FSM complète avec états transitoires pour DCC (cela constituerait un travail futur, cf. section 7.3.2). Elle travaille à un niveau d'abstraction plus haut, modélisant **uniquement les états stables MOESI** :

| État | Description |
|------|-------------|
| **Modified (M)** | Ligne modifiée localement, mémoire principale non à jour, seule copie |
| **Owned (O)** | Ligne modifiée, plusieurs copies possibles, mémoire principale non à jour |
| **Exclusive (E)** | Seule copie valide, mémoire principale à jour |
| **Shared (S)** | Copie valide parmi d'autres, mémoire principale possiblement à jour |
| **Invalid (I)** | Ligne invalide |

### 4.1 Interface du répertoire (définie dans PyCCExplorer)

Les fonctions suivantes sont implémentées pour chaque représentation de la liste des copies :

| Fonction | Déclencheur | Action |
|----------|-------------|--------|
| **Read** | ReadReq (lecture L1) | Satisfait depuis une copie ou L2 ; met à jour la liste |
| **ReqEx** | WriteReq, UpgradeReq, SCUpgradeReq, ReadExReq | Invalide toutes les copies ; met à jour la liste |
| **Delete** | Éviction d'une ligne du cache L2 | Invalide toutes les copies L1 (cache inclusif) |
| **Writeback** | Writeback L1 | Met à jour L2 ; supprime la copie modifiée de la liste |
| **Clean** | Clean Eviction L1 (ajout à gem5) | Supprime une copie propre de la liste |

### 4.2 Flux de messages pour les opérations principales

#### 4.2.1 Lecture (Read Miss au L2)
1. L1_i envoie ReadReq au L2
2. L2 consulte le répertoire → trouve une ou plusieurs copies
3. En mode précis (G=0 pour Ackwise, broadcast=0 pour DCC) : L2 envoie ReadForward au keeper/sharer le plus proche
4. Le sharer envoie DataResp à L1_i (response directe, pas par le répertoire)
5. Le sharer envoie Ack au répertoire (pour mise à jour de la liste des copies)
6. Si aucune copie dans L1 : L2 répond directement depuis sa propre donnée
7. Si miss complet (L2 et L1 vides) : L2 envoie requête à la mémoire principale (latence 100 cycles dans le modèle)

#### 4.2.2 Écriture / Lecture exclusive (ReqEx)
1. L1_i envoie ReadExReq/UpgradeReq au L2
2. L2 consulte le répertoire → liste des copies
3. **Mode précis :** L2 envoie Invalidate à chaque copie individuellement (1 message par copie, 1 cycle par envoi)
4. Chaque L1_j envoie InvAck au L2 (répertoire attend tous les Acks)
5. **Mode diffusion :** L2 envoie 1 message broadcast, attend N_copies Acks
6. L2 envoie Data à L1_i avec l'exclusivité

#### 4.2.3 Eviction propre (Clean Eviction)
1. L1_i envoie CleanEviction au L2 (message ajouté à gem5)
2. L2 reçoit la notification → appelle la fonction Clean
3. Le répertoire supprime L1_i de la liste des copies
4. Si DCC : si L1_i était dans le rectangle, le bit correspondant est mis à 0 et le bloc tiling peut être re-exécuté ; si L1_i était dans la liste chaînée, l'entrée est libérée et retournée au tas

#### 4.2.4 Remplacement d'une ligne au L2 (Eviction L2)
1. L2 choisit une ligne à évincer (politique LRU/pseudo-aléatoire)
2. L2 appelle la fonction Delete pour cette ligne
3. Messages d'invalidation envoyés à chaque copie de la liste des copies
4. Quand tous les Acks reçus, la ligne L2 est libérée
5. En mode inclusif : toutes les copies L1 de cette ligne sont forcément invalidées

#### 4.2.5 Passage en mode diffusion (DCC spécifique)
1. Lors d'un ajout de copie : liste chaînée atteint T ou tas plein
2. Libération des entrées de tas → liste chaînée vide
3. `broadcast` mis à 1
4. `origin` ← CID du keeper (première copie dans le champ de bits)
5. `bit vector` + `pointer` ← compteur de copies (nombre total de copies à ce moment)
6. Reexecution du bloc tiling non nécessaire

### 4.3 Comportement détaillé du répertoire Ackwise (implémenté dans PyCCExplorer)

**Lecture (G=0) :**
- Vérification que le keeper est disponible
- ReadForward au keeper
- keeper → Data → L1_demandeur
- Ack au L2
- Mise à jour : si espace disponible, CID ajouté dans la liste ; sinon G=1

**Lecture exclusive (G=0) :**
- Invalidation individuelle de chaque copie (boucle sur la liste)
- Attente de k Acks
- L2 → Data → L1_demandeur

**Lecture (G=1) :**
- Le keeper répond (broadcast vers le keeper uniquement pour une lecture)
- Compteur incrémenté

**Lecture exclusive (G=1) :**
- Message d'invalidation en diffusion
- Attente d'exactement N_copies Acks (valeur du compteur)
- L2 → Data → L1_demandeur

### 4.4 Comportement détaillé du répertoire Liste chaînée (implémenté dans PyCCExplorer)

**Lecture (mode précis) :**
- Récupération du premier élément de la liste chaînée (CID de la première copie)
- Forward au cache correspondant
- Le cache copie → Data → L2 → L1_demandeur (passe par L2)
- Ajout de la nouvelle copie en fin de liste chaînée

**Lecture (mode diffusion) :**
- Envoi au keeper (mémorisé dans l'entrée de répertoire)
- Compteur incrémenté

**Lecture exclusive (mode précis) :**
- Parcours complet de la liste chaînée : Invalidate à chacun
- Attente de tous les Acks (1 par copie)

**Lecture exclusive (mode diffusion) :**
- Broadcast Invalidate
- Attente de N_copies Acks

---

## 5. Méthode d'évaluation (Chapitre 5)

### 5.1 Comparaison des méthodes de simulation

| Méthode | Précision | Temps simulation | Adapté exploration |
|---------|-----------|------------------|--------------------|
| gem5 classique | Instructions | 54h pour 13 applis | Non (lent) |
| gem5 + Ruby | Cycle accurate | Plusieurs jours/appli | Non (très lent) |
| SystemC CABA | Bit/cycle accurate | >3 jours pour 16 cœurs | Non |
| RTL + émulateur | RTL | Synthèse requise | Non (développement) |
| Graphite/Flexus | Instructions | Similaire gem5 | Non |
| **PyCCExplorer** | **États stables** | **6-12h total** | **Oui** |

### 5.2 Flot de simulation PyCCExplorer (3 étapes)

**Étape 1 — Extraction des traces avec gem5 :**

Paramètres gem5 :
| Paramètre | Valeur |
|-----------|--------|
| CPU | 64 cœurs Alpha 2 GHz |
| L1 instructions | 32 kB, 2-way |
| L1 données | 64 kB, 2-way |
| L2 partagé | 256 kB × 64 bancs, 8-way |
| Taille de ligne | 64 octets |
| Réseau | Full crossbar (topologie plate) |
| Mode | Full System, modèle mémoire classique (non Ruby) |

Modifications apportées à gem5 :
1. **Ajout de requêtes `CleanEviction`** : envoyées par L1 lors d'évictions propres (gem5 ne les génère pas car L2 n'est pas inclusif par défaut).
2. **Ajout de moniteurs** entre L1 et le réseau : un moniteur par cache L1 (128 au total = 64×2, un pour instructions + un pour données). Enregistrement de {tick, source_CID, adresse, type_requête}.

Temps total d'extraction : ~105 heures (2 passes par application). Base de données résultante : 58 Go.

13 applications benchmarked :
- **PARSEC** : blackscholes, bodytrack, canneal, dedup, ferret, fluidanimate, freqmine, streamcluster, vips, x264
- **Splash2** : fft, ocean_cp, ocean_ncp

Seule la **Region Of Interest (ROI)** est simulée (caches redémarrés à froid au début de la ROI).

**Étape 2 — Simulation avec PyCCExplorer :**

Architecture logicielle du simulateur Python :
- **Main** (238 loc) : lecture base de données, injection requêtes, configuration
- **Cache L2** (290 loc) : ensembles + voies, remplacement LRU/pseudo-aléatoire, inclusif (contrairement à gem5)
- **Répertoire + protocoles** (135 loc commun + ~150 loc par représentation) : interface Read/ReqEx/Delete/Writeback/Clean
- **Réseau** (161 loc) : grille 2D, routage X-first, 2 canaux (requêtes + réponses), support broadcast matériel

Représentations implémentées :
- Espionnage (97 loc)
- Champ de bits complet (120 loc)
- Ackwise (180 loc)
- Liste chaînée (~150 loc)
- DCC idéal, premier touché, combinatoire (~150 loc chacun)

Simplifications : états stables uniquement (5 états MOESI), pas de contention réseau modélisée, latence réseau = nombre de routeurs traversés (Manhattan), broadcast = support matériel (1 message envoyé, réseau le réplique).

**Étape 3 — Analyse des résultats :**
- Latence : tick_réponse - tick_requête, mesurée en cycles
- Trafic : compteurs par canal (requêtes et réponses) sur fenêtres de 10 000 cycles
- Messages de diffusion : compteur dédié

### 5.3 Placement optimisé des tâches (recuit simulé)

Car gem5 utilise un réseau plat (crossbar), l'OS ne place pas les tâches proche topologiquement. Pour corriger ce biais lors de l'évaluation de DCC (qui exploite la localité), un placement est calculé hors-ligne par **recuit simulé** minimisant la fonction d'énergie :

```
E = Σᵢ Σⱼ d(i,j) × l(i,j)
```

où d(i,j) = distance Manhattan entre les cœurs i et j sur la grille, et l(i,j) = nombre de lignes de cache partagées par les caches des cœurs i et j.

Paramètres du recuit : T₀=60000, décroissance Ti+1 = 0.9 × Ti, 100 paliers de température, 400 échanges de cœurs par palier. Placement calculé une fois, réutilisé pour toutes les simulations DCC.

---

## 6. Résultats et évaluation (Chapitre 6)

### 6.1 Exploration architecturale préliminaire

#### Ackwise — Seuil k

Simulations pour k ∈ {2, 3, 4, 5, 8, 64} :
- Latence normalisée à k=5 : k=2 est nettement plus mauvais ; k≥5 donne des améliorations marginales.
- **Valeur retenue : k=5** (31 bits par entrée de répertoire)
- Résultats cohérents avec les travaux d'Ackwise originaux [Kurian et al., 2010].

#### Liste chaînée — Seuil T et taille de tas H

Seuil T (avec tas infini) :
- T=2 : taux de broadcast élevé car beaucoup de lignes partagées par plus de 2 cœurs
- T=9 vs T=3 : différence notable entre T=2 et T=3 (passage de ~25% à ~15% de lignes en diffusion pour certaines applications)
- Distribution des lignes : majorité partagées par ≤8 cœurs (pic à 64 pour les données globales)

Taille du tas H :
- Avec seuil, usage moyen du tas < 100 entrées, maximum < 300
- Tas de 256 entrées satisfait la quasi-totalité des cas
- **Valeur retenue : H=256** (1/16ème du nombre de lignes de cache)

### 6.2 Classement des représentations de référence

#### Latence (Figure 6.6, Table 6.1)

| Protocole | Latence relative au champ de bits |
|-----------|----------------------------------|
| Champ de bits complet | Référence (0%) |
| Ackwise (k=5) | +1.9% en moyenne |
| Liste chaînée (T=5, H=256) | +6.4% en moyenne |
| Espionnage | +42.7% en moyenne |

Notes :
- Pour dedup et freqmine, Ackwise < champ de bits car ces applis ont beaucoup d'écritures (ReqEx) → le mode diffusion d'Ackwise invalide d'un seul broadcast vs invalidations individuelles.
- Espionnage : latence sous-estimée (pas de modélisation de la contention réseau, réelle latence encore plus élevée).

#### Trafic (Figure 6.10, Table 6.1)

| Protocole | Trafic requête (vs champ de bits) | Trafic réponse (vs champ de bits) |
|-----------|----------------------------------|----------------------------------|
| Champ de bits | Référence | Référence |
| Liste chaînée | +5% | +6.5% |
| Ackwise (k=5) | +16.8% | +16.2% |
| Espionnage | ×6.32 (+532%) | ×40.88 (+3888%) |

Note : espionnage génère un trafic de réponses bien supérieur (chaque cache L1 répond individuellement à chaque broadcast).

#### Messages de diffusion

| Protocole | Nombre de diffusions (vs DCC idéal) |
|-----------|-------------------------------------|
| Champ de bits | 0 (pas de diffusion) |
| DCC idéal | Référence |
| DCC combinatoire | +10.4% |
| DCC premier touché | +57.4% |
| Liste chaînée | +114% (×2.14 vs DCC idéal) |
| Ackwise | ×12.67 (un ordre de grandeur de plus) |
| Espionnage | ×789 (pratiquement tout est en diffusion) |

### 6.3 Paramétrage DCC

À partir de l'exploration architecturale :
- **Taille rectangle C = 16** : bon compromis entre couverture des copies et surcoût mémoire ; la moyenne d'occupation du rectangle pour des lignes partagées est < moitié de C, indiquant que C=16 est suffisant.
- **Seuil T = 4** : avec T=4, le bloc tiling nécessite n=6 entrées minimum ; passer à T=5 rendrait le bloc tiling trop coûteux (complexité exponentielle).
- **Taille tas H = 128** : usage moyen < 50 entrées, maximum < 150 pour la plupart des applications.

### 6.4 Résultats comparatifs complets de DCC (Table 6.1)

| Protocole | Latence | Trafic req | Trafic rép | Nb diffusions |
|-----------|---------|-----------|------------|---------------|
| Espionnage | +42.7% | ×6.32 | ×40.88 | ×789 |
| **Champ de bits** | **+0% (Réf)** | **+0% (Réf)** | **+0% (Réf)** | **0 (pas de diff.)** |
| Ackwise k=5 | +1.9% | +16.8% | +16.2% | ×12.67 |
| Liste chaînée | +6.4% | +5% | +6.5% | +114% |
| DCC idéal | +4.8% | +3.9% | +4% | +0% (Réf) |
| DCC premier touché | +5.2% | +3.9% | +5.3% | +57.4% |
| **DCC combinatoire** | **+3.4%** | **+3.4%** | **+4.7%** | **+10.4%** |

**Analyse comparative :**
- Pour la **latence** : DCC combinatoire se classe 2ème (meilleur que la liste chaînée, légèrement moins bon qu'Ackwise).
- Pour le **trafic** : DCC se classe 2ème derrière le champ de bits complet, nettement meilleur qu'Ackwise (+17% → +3%).
- Pour les **diffusions** : DCC est 1er parmi les protocoles qui génèrent des broadcasts. DCC combinatoire est à +10.4% de DCC idéal, contre ×12.67 pour Ackwise (1 ordre de grandeur d'écart) et ×789 pour l'espionnage.
- L'algorithme **combinatoire est très proche de l'idéal** (meilleur que premier touché sur tous les critères).

### 6.5 Validation de PyCCExplorer

- Divergence totale avec gem5 : <3% des requêtes (0.14% read-already-present, 2.08% eviction-unknown-block, 0.51% eviction-L1-unknown-block).
- Même classement obtenu qu'avec le simulateur Noxim [Xu et al., 2011] (cycle-accurate SystemC).
- Même tendances qu'avec les simulations Graphite d'Ackwise [Kurian et al., 2010] (même application Ocean, même architecture 64 cœurs).
- Vitesse : PyCCExplorer traite 110 000 transactions/seconde vs 290 avec gem5+Ruby → **380× plus rapide**.

---

## 7. Travaux futurs identifiés (Chapitre 7)

### 7.1 Améliorations de PyCCExplorer

1. **Plus de 64 cœurs** : gem5 est limité à 64 cœurs Alpha ; utiliser un autre simulateur ou modifier gem5 (complexe à cause des champs ISA et du recompile Linux).
2. **Autres topologies réseau** : actuellement seul le 2D mesh est implémenté (bus, hypercube, tore possible).
3. **Autres représentations** : ajouter coarse bit-vector directory, Tardis, etc.
4. **Méthodes décentralisées** : filtres de Bloom, méthodes sans répertoire central.

### 7.2 Limites et pistes pour DCC

1. **Simulation avec initialisation** : les traces sont extraites sans la phase d'initialisation (ROI uniquement). L'algorithme premier touché serait probablement plus défavorisé avec l'initialisation.
2. **Sans placement optimal** : les simulations DCC supposent un placement de tâches optimisé (recuit simulé). Simuler sans ce placement permettrait de mesurer l'influence réelle du placement.
3. **Autres architectures** : clusters, architectures >64 cœurs.

### 7.3 Pistes de recherche DCC

#### 7.3.1 Choix du keeper amélioré
Actuellement, le keeper = première copie dans le champ de bits lors du passage en mode diffusion. Un meilleur choix serait la copie la plus **centrale** dans la grille (minimise la latence de réponse). Également, gestion du cas où le keeper évince sa ligne.

#### 7.3.2 Intégration du bloc tiling dans une architecture complète
La logique de contrôle et la machine à états complète (avec états transitoires) doit être développée pour une implémentation réelle.

#### 7.3.3 Tas partagé pour les rectangles de cohérence

Deux solutions proposées pour réduire le coût de l'entrée du répertoire en exploitant le fait que >50% des lignes sont privées :

**Solution 1 — Un niveau d'indirection selon privé/partagé :**
```
Nouveau répertoire : | Private (1 bit) | DCC_ptr ou CID |
Tas DCC : Origin | Shape | Bit-vector | Pointer
```

**Solution 2 — Deux niveaux d'indirection selon mode :**
```
Nouveau répertoire : | Private (1 bit) | Broadcast (1 bit) | Ptr |
Tas DCC (lignes partagées précises) : Origin | Shape | Bit-vector | Pointer
Tas Broadcast (lignes en mode diffusion) : Keeper | Nb_copies
Tas liste chaînée : CID | Pointer
```
Dimensionnement complexe mais potentiellement très économique en mémoire.

---

## 8. Ce qu'il faudra implémenter dans gem5 (synthèse)

Pour implémenter DCC (et Ackwise) dans gem5/Ruby avec le réseau Garnet, voici la liste précise des éléments à développer ou modifier :

### 8.1 Structure de données du répertoire

Créer une nouvelle représentation de la liste des copies dans les fichiers SLICC du protocole directory. Les structures à coder sont :

```
// Entrée principale du répertoire DCC
struct DCCDirectoryEntry {
    bool         broadcast;    // 1 bit
    NodeID       origin;       // log2(N) bits = CID de l'origine ou du keeper
    uint8_t      shape;        // 3 bits (pour C=16, grille 8×8)
    Bit[C]       bit_vector;   // C=16 bits
    HeapIdx      heap_ptr;     // log2(H)=7 bits, pointeur vers tas
    int          copy_count;   // Compteur (en mode broadcast), réutilise bit_vector+heap_ptr
    MachineID    state;        // MOESI states
};

// Entrée du tas partagé
struct HeapEntry {
    NodeID       cid;          // log2(N)=6 bits
    HeapIdx      next;         // log2(H)=7 bits, NULL_PTR si fin de liste
};

// Tas partagé (global au cache L2, partagé entre toutes les lignes)
HeapEntry heap[H];             // H=128 entrées
bool      heap_free[H];        // Bitmap des entrées libres
```

### 8.2 Fonctions principales à implémenter

#### 8.2.1 Gestion du tas

```python
# Allouer une entrée du tas
function allocate_heap() -> HeapIdx:
    Chercher une entrée libre dans heap_free[]
    Si aucune : retourner HEAP_FULL (déclenchera mode diffusion)
    Marquer comme occupée et retourner l'index

# Libérer une entrée du tas
function free_heap(idx: HeapIdx):
    heap_free[idx] = true

# Libérer toute la liste chaînée d'une entrée
function free_linked_list(ptr: HeapIdx):
    while ptr != NULL_PTR:
        next = heap[ptr].next
        free_heap(ptr)
        ptr = next
```

#### 8.2.2 Ajout d'une copie

```
function add_copy(entry: DCCDirectoryEntry, new_cid: NodeID):
    Si entry.broadcast == 1 :
        entry.copy_count += 1
        return OK
    
    // Convertir new_cid en coordonnées (x, y)
    (nx, ny) = cid_to_coords(new_cid)
    
    // Vérifier si la copie est déjà dans le rectangle
    Si coords_in_rectangle(nx, ny, entry.origin, entry.shape) :
        // Mettre le bit correspondant à 1
        bit_idx = coords_to_local_idx(nx, ny, entry.origin, entry.shape)
        entry.bit_vector[bit_idx] = 1
        return OK
    
    // Tenter d'ajouter dans la liste chaînée
    Si linked_list_size(entry.heap_ptr) >= T :
        // Seuil atteint → mode diffusion
        go_broadcast(entry)
        return BROADCAST
    
    new_entry = allocate_heap()
    Si new_entry == HEAP_FULL :
        // Tas plein → mode diffusion
        go_broadcast(entry)
        return BROADCAST
    
    // Exécuter le bloc tiling pour recalculer le rectangle
    result = tiling_block(entry, new_cid)
    Mettre à jour entry.origin, entry.shape, entry.bit_vector, entry.heap_ptr
    return OK
```

#### 8.2.3 Bloc tiling (algorithme combinatoire)

```
function tiling_block(entry: DCCDirectoryEntry, new_cid: NodeID):
    // Construire la liste des n points d'entrée
    n = T + 2  // mode sous-optimal si nb_copies > n - 2
    
    // Mode optimal : déplier le bit_vector en coordonnées individuelles
    // Mode sous-optimal : entrée 1 = rectangle courant (xmin,ymin,xmax,ymax, weight)
    //                      entrées 2..n = copies liste chaînée + nouvelle copie
    
    points = build_input_points(entry, new_cid)
    
    best_rect = None
    best_count = 0
    
    for k in range(2, len(points)+1):
        for combo in combinations(points, k):
            (xmin, xmax, ymin, ymax) = bounding_box(combo)
            area = (xmax - xmin + 1) * (ymax - ymin + 1)
            if area <= C:
                count = sum(p.weight for p in combo)
                if count > best_count:
                    best_count = count
                    best_rect = (xmin, xmax, ymin, ymax, combo)
    
    // Cas sous-optimal : tenter de conserver le rectangle courant
    if sous_optimal:
        // ... (voir Algorithme 1 complet)
    
    return best_rect
```

#### 8.2.4 Passage en mode diffusion

```
function go_broadcast(entry: DCCDirectoryEntry):
    // Choisir le keeper (première copie dans le bit_vector)
    keeper_cid = first_set_bit_to_cid(entry.bit_vector, entry.origin, entry.shape)
    
    // Compter les copies totales
    n_copies = popcount(entry.bit_vector)
    n_copies += linked_list_size(entry.heap_ptr)
    
    // Libérer la liste chaînée
    free_linked_list(entry.heap_ptr)
    
    // Mettre à jour l'entrée
    entry.broadcast = 1
    entry.origin = keeper_cid
    entry.copy_count = n_copies
    entry.heap_ptr = NULL_PTR
    entry.bit_vector = 0
```

### 8.3 Machine à états SLICC à modifier/créer

Dans gem5/Ruby, les protocoles à répertoire utilisent le langage SLICC. Il faudra créer un protocole `DCC_Two_Level` (ou modifier `MESI_Two_Level` / `MOESI_Two_Level`) avec :

**États stables du répertoire :**
- `I` (Invalid) : ligne non présente dans aucun L1
- `S` (Shared) : une ou plusieurs copies dans des L1, en mode précis DCC
- `M` (Modified) : une seule copie exclusive dans un L1
- `S_Broadcast` : plusieurs copies, mode diffusion activé
- `M_Broadcast` : cas dégénéré (normalement impossible avec M)

**États transitoires (à définir) :**
- `I_W` : en attente de réponse de la mémoire principale (compulsory miss)
- `S_W` : en attente d'acquittements d'invalidation (ReqEx en mode précis)
- `S_B_W` : en attente d'acquittements d'invalidation (ReqEx en mode broadcast)
- `M_W` : en attente de writeback depuis un L1 Modified

**Messages à définir :**
- `ReadReq` : requête de lecture L1 → L2
- `ReadExReq` : requête de lecture exclusive L1 → L2
- `UpgradeReq` : requête de passage Shared→Exclusive L1 → L2
- `CleanEviction` : éviction propre L1 → L2 (**à ajouter/vérifier dans gem5**)
- `Writeback` : écriture différée L1 → L2
- `Forward` : L2 → L1 (pour satisfaire une lecture depuis une autre copie)
- `Invalidate` : L2 → L1 (invalidation lors d'une ReqEx)
- `InvAck` : acquittement d'invalidation L1 → L2
- `DataResp` : réponse avec donnée L1 → L1_demandeur ou L2 → L1_demandeur
- `Broadcast_Inv` : L2 → tous les L1 (mode diffusion, invalidation)
- `Broadcast_Ack` : L1 → L2 (acquittement broadcast)

### 8.4 Modifications à apporter à gem5 (identiques à celles de la thèse)

1. **Requêtes `CleanEviction`** : vérifier si gem5 (version actuelle) génère ces requêtes nativement ou s'il faut les ajouter comme dans la thèse.

2. **Inclusivité du cache L2** : le protocole Ruby nécessite un cache L2 inclusif (toutes les lignes en L1 sont en L2). gem5 Ruby MESI_Two_Level gère déjà l'inclusivité, mais il faut vérifier la compatibilité.

3. **Bloc tiling** : peut être implémenté en C++ dans le contrôleur du répertoire SLICC, appelé lors des transitions d'état.

4. **Tas partagé** : implémenter comme une structure partagée entre toutes les entrées de cache L2, avec allocation/libération dynamique.

### 8.5 Paramètres configurables

| Paramètre | Valeur par défaut | Description |
|-----------|-------------------|-------------|
| `dcc_rect_size` | 16 | Aire maximale du rectangle de cohérence (C) |
| `dcc_threshold` | 4 | Seuil de la liste chaînée (T) |
| `dcc_heap_size` | 128 | Nombre d'entrées du tas (H) |
| `dcc_tiling_inputs` | 6 | Nombre d'entrées du bloc tiling (n = T+2) |
| `dcc_tiling_lut` | true | Utiliser LUT (true) ou multiplications (false) |
| `dcc_algorithm` | "combinatoire" | Algorithme de placement (idéal/premier_touche/combinatoire) |

### 8.6 Topologie réseau Garnet

DCC est conçu pour un **réseau 2D maillé** (mesh). Il faudra configurer Garnet pour ce type de réseau et s'assurer que les CIDs des cœurs correspondent aux coordonnées (x, y) de la grille :

```
CID = y * sqrt(N) + x
x = CID mod sqrt(N)
y = CID / sqrt(N)
```

La fonction `cid_to_coords()` et `coords_to_cid()` sont essentielles pour le bloc tiling.

---

## 9. Références clés de la bibliographie

- **[KMP+10]** Kurian et al. — Ackwise (ATAC, 1000-core, Graphite simulator)
- **[TDF93]** Thapar, Delagi, Flynn — Liste chaînée originale
- **[GWM90]** Gupta, Weber, Mowry — Coarse bit-vector directory
- **[ZSS+14]** Zhang et al. — Sponge directory
- **[ZSD10]** Zhao, Shen, Du — SPACE
- **[CP99]** Choi, Park — Segment directory
- **[FNW15]** Fu, Nguyen, Wentzlaff — Coherence Domain Restriction
- **[APJ09a]** Agarwal, Peh, Jha — INCF
- **[EPS06]** Eisley, Peh, Shang — INCC
- **[JLGS90]** James et al. — Scalable Interface Coherence
- **[YD15]** Yu, Ding — Tardis
- **[BBB+11]** Binkert et al. — gem5
- **[AKPJ09]** Agarwal, Krishna, Peh, Jha — Garnet (NoC dans gem5)
- **[WOT+95]** Woo et al. — Splash2 benchmarks
- **[BKSL08]** Bienia et al. — PARSEC benchmarks
- **[BGOS12]** Butko et al. — Évaluation des modes de gem5

---

## 10. Résumé exécutif pour l'implémentation gem5

Le travail à implémenter dans gem5 consiste en :

1. **Un nouveau protocole SLICC** (`DCC_Two_Level` ou similaire) avec les états stables I/S/M/S_Broadcast et les états transitoires nécessaires.

2. **Deux nouvelles structures de données C++** dans le contrôleur du répertoire :
   - L'entrée de répertoire DCC (33 bits pour 64 cœurs, C=16, H=128)
   - Le tas partagé (128 × 13 bits = 1664 bits total pour un cache L2)

3. **Le bloc tiling en C++** : algorithme combinatoire avec n=6 entrées, version LUT ou multiplication, exécuté en O(2ⁿ) opérations combinatoires lors de chaque mise à jour de la liste des copies.

4. **Les fonctions de conversion CID ↔ coordonnées** pour le réseau Garnet 2D mesh.

5. **La logique de transition** entre mode précis et mode diffusion, avec gestion correcte du keeper et du compteur de copies.

6. **Vérification de l'ajout de `CleanEviction`** dans gem5 (présent dans certaines versions récentes sous le nom `WritebackClean` ou `CleanEvict`).

7. **Configuration Garnet** pour un réseau 2D mesh avec routage XY (déjà disponible dans gem5/Garnet).

Les performances attendues de DCC combinatoire vs. Ackwise (k=5) pour un budget mémoire similaire (~31-33 bits/entrée) : trafic ×0.83 moins élevé, diffusions ×12× moins nombreuses, latence +1.5% plus élevée.