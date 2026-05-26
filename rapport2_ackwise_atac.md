---

# Rapport détaillé — ATAC & ACKwise
**Source :** *ATAC: A Manycore Processor with On-Chip Optical Network*, MIT CSAIL Technical Report MIT-CSAIL-TR-2009-018, mai 2009.
**Auteurs :** Jason Miller, James Psota, George Kurian, Nathan Beckmann, Jonathan Eastep, Jifeng Liu, Mark Beals, Jurgen Michel, Lionel Kimerling, Anant Agarwal (MIT).

---

## 1. Contexte et motivation

### 1.1 Le problème de passage à l'échelle des manycores

L'article part du constat que l'industrie microprocesseur a abandonné la montée en fréquence au profit du parallélisme massif : des puces à 1 000 cœurs ou plus sont anticipées pour le milieu des années 2010. Cependant, la **scalabilité** de ces architectures est bloquée par deux goulots d'étranglement fondamentaux :

1. **Le coût de communication on-chip.** Dans un réseau maillé électrique (*mesh*), la latence entre deux cœurs croît avec leur distance physique (un cycle par saut). Pour 1 000 cœurs, la distance moyenne entre deux cœurs est de l'ordre de $\sqrt{N}$ sauts, soit environ 31 sauts. Cette hétérogénéité de latence rend la programmation difficile et génère de la contention lorsque des opérations globales (broadcasts pour la cohérence de cache, synchronisation, distribution d'instructions) se produisent.

2. **Le coût des broadcasts.** Les protocoles de cohérence de caches nécessitent des opérations de diffusion (invalidations globales). Dans un réseau maillé électrique, un broadcast est émulé par de multiples messages point-à-point, ce qui consomme une quantité de ressources proportionnelle à $N$, et génère de la congestion exponentielle au passage à l'échelle.

### 1.2 Limites des approches existantes

- **Bus partagé :** simple, latence uniforme, mais ne passe pas à l'échelle au-delà de quelques dizaines de cœurs (la longueur du bus augmente la résistance capacitive et ralentit l'horloge ; la contention croît avec le nombre de cœurs).
- **Réseau maillé point-à-point (ex. Raw [17])** : évite les longs fils globaux, mais la latence est hétérogène et les patterns de communication irréguliers ou broadcasts provoquent une congestion inacceptable à grande échelle.
- **Protocols de snooping :** les auteurs estiment que les architectures de snooping (ex. Kirman et al., 2006) ne peuvent pas passer à l'échelle à des centaines ou milliers de cœurs.
- **Directories classiques à presence bits (*full-map*)** : nécessitent $O(N)$ bits par entrée de directory pour tracker les $N$ cœurs potentiellement sharers. Pour 1 000 cœurs, chaque entrée de directory ferait 1 000 bits, ce qui est prohibitif en surface et en énergie. L'article référence les *limited directory schemes* [4] comme base de travail.

---

## 2. Architecture ATAC

### 2.1 Vue d'ensemble

ATAC (*All-To-All Computing*) est une architecture manycore à tuiles (*tiled*) ciblant un processus 11 nm en 2019, avec **1 024 cœurs**. La philosophie centrale est d'augmenter un réseau maillé électrique classique par un réseau optique global exploitant la technologie nanophotonique.

Chaque cœur contient :
- Un pipeline RISC simple (un ou deux issues, in-order)
- Un cache L1 données et instructions
- Une **portion du directory de cohérence de cache** (le directory est distribué uniformément sur tous les cœurs ; chaque cœur est le "home" d'un ensemble d'adresses déterminé statiquement)

### 2.2 Les deux réseaux : EMesh et ANet

Les 1 024 cœurs sont reliés par **deux réseaux complémentaires** :

| Réseau | Type | Usage |
|--------|------|--------|
| **EMesh** | Maillé 2D électrique point-à-point | Communications locales, prévisibles, courte distance |
| **ANet** | Optique + électrique | Communications longue distance, broadcasts, latence faible et uniforme |

### 2.3 L'ANet en détail

L'ANet est lui-même composé de trois sous-réseaux :

```
ANet = ONet (optique) + ENet (électrique, envoi vers hub) + BNet (électrique, broadcast depuis hub)
```

**Découpage en clusters :**
Les 1 024 cœurs sont regroupés en **64 clusters de 16 cœurs** chacun. Chaque cluster partage un **Hub optique** (ONet Hub). Cette limite de 64 Hubs est imposée par les contraintes physiques des composants optiques (WDM, perte de propagation, puissance optique).

**ONet (Optical Network) :**
- Interconnecte les 64 Hubs via des guides d'onde (*waveguides*) en Si qui serpentent sur la puce et forment des **boucles continues**.
- Chaque Hub peut **émettre** un signal sur son guide d'onde via un modulateur optique (il dispose d'une longueur d'onde unique grâce au WDM), et **recevoir** les signaux de tous les autres Hubs via des filtres et photodetecteurs.
- Puisque les guides d'onde forment une boucle, un signal émis par n'importe quel Hub atteint **tous les autres Hubs en une seule opération** → chaque transmission sur l'ONet est potentiellement un **broadcast global efficace**.
- Le **WDM (Wavelength Division Multiplexing)** permet à un même guide d'onde de transporter simultanément $N$ canaux indépendants sur $N$ longueurs d'onde différentes, **sans contention et sans arbitration**.
- L'ONet se compose de : **64 guides d'onde de données** (un par cluster/Hub), **1 guide de contrôle de flux retour**, et **plusieurs guides de métadonnées** (type de message, tag pour désambiguïser plusieurs messages du même émetteur).
- **Latence fixe et uniforme** : l'ONet propagation time est de 2,5 ns (valeur de référence dans l'évaluation), indépendamment de la distance entre cœurs.

**ENet :**
- Réseau maillé électrique **intra-cluster** uniquement.
- Utilisé pour envoyer des données des cœurs du cluster vers leur Hub pour transmission sur l'ONet.

**BNet (Broadcast Network) :**
- Arbre de broadcast électrique **intra-cluster**.
- Utilisé pour rediffuser aux cœurs du cluster les données reçues par leur Hub depuis l'ONet.
- Plusieurs BNets peuvent coexister dans un cluster (dans l'évaluation : 2 BNets, notés BNet0 et BNet1).

**Technologie optique utilisée :**
- Source lumineuse : lasers **hors puce** couplés dans un guide d'onde on-chip (1,5 W laser, 0,2 W de puissance optique dans le guide).
- Guides d'onde en **Si** (fabriqués en processus CMOS standard), pertes < 0,3 dB/cm.
- **Modulateurs** (switches optiques) : insertion loss 1 dB, surface < 50 µm², débit > 20 Gbps, énergie de commutation < 25 fJ (caractéristiques prévues pour 2012).
- **Filtres** (ring resonators) : sélectionnent une longueur d'onde spécifique.
- **Photodetecteurs** : responsivité > 1 A/W, bande passante > 20 GHz, surface < 20 µm², capacitance < 1 fF.

---

## 3. Protocole de cohérence ACKwise

### 3.1 Principe général

ACKwise est un protocole de cohérence de cache à base de **directory**, dérivé du protocole **MOESI** [15]. Le directory est distribué sur tous les cœurs (chaque cœur héberge le directory d'un sous-ensemble d'adresses). ACKwise est nommé ainsi car il maintient un **compte du nombre de sharers** après dépassement de la capacité de la liste des copies, et n'attend des ACKs que de ce nombre précis de sharers en réponse à une invalidation broadcast.

**Contrainte absolue :** les **évictions silencieuses sont interdites** (*silent evictions must be avoided*). Un cœur qui expulse une ligne de son cache doit obligatoirement notifier le directory, sinon le compte de sharers serait incorrect.

### 3.2 Structure d'une entrée de directory

Chaque entrée du directory contient **4 champs** (voir Figure 6 du papier) :

```
[ State | KeeperID | Sharer5 | Sharer4 | Sharer3 | Sharer2 | Sharer1 | G ]
```

| Champ | Description |
|-------|-------------|
| **State** | État MOESI du bloc de données : I (Invalid), E (Exclusive), S (Shared), M (Modified), O (Owned) |
| **G (Global bit)** | Mis à 1 si le nombre de sharers dépasse la capacité de la liste (> 6 sharers au total, keeper inclus) |
| **KeeperID** | ID du cœur qui possède une copie à jour du bloc (propre en états E et S, sale en états M et O) |
| **Sharers1–5** | **Liste de 5 IDs de sharers** quand G=0 ; **compte total des sharers** quand G=1 |

**Comportement de la liste :**
- **G=0, liste non pleine :** les IDs des sharers sont stockés explicitement dans Sharers1–5. Au total, le directory peut donc tracker **6 sharers** (le Keeper + 5 dans la liste).
- **G=0, liste pleine, nouveau sharer arrive :** G est mis à 1 et le compte total (7 = 6+1) est stocké dans le champ Sharers1–5.
- **G=1 :** le directory ne connaît plus les identités précises des sharers. Il sait seulement : (a) l'ID du Keeper, et (b) le **nombre total de sharers**. Ce champ est incrémenté à chaque nouveau sharer.

**Pourquoi cette structure est compacte :** au lieu d'un vecteur de présence de $N$ bits (O(N)), ACKwise n'utilise que **5 IDs de cœurs + 1 bit + 1 ID Keeper**. Pour 1 024 cœurs, un ID nécessite 10 bits, donc la liste des sharers occupe : 1 (G) + 10 (KeeperID) + 5×10 (Sharers) = **61 bits** seulement, indépendamment de N.

### 3.3 États MOESI et rôles

ACKwise utilise les 5 états MOESI :

| État | Signification | Copies possibles |
|------|---------------|-----------------|
| **I** (Invalid) | Bloc non présent dans aucun cache | Aucune |
| **E** (Exclusive) | Une seule copie propre, le Keeper peut la modifier sans notifier | 1 copie (le Keeper) |
| **S** (Shared) | Plusieurs copies propres en lecture seule | ≥ 1 copie |
| **M** (Modified) | Une seule copie sale (Keeper = propriétaire unique) | 1 copie (le Keeper) |
| **O** (Owned) | Une copie sale (le Keeper = owner) + des copies propres | ≥ 1 (Keeper dirty + sharers clean) |

### 3.4 Opération pour une requête de lecture (ShReq — requête de copie partagée)

**Cas (a) — État Invalid :**
1. Le directory reçoit un ShReq du Requester R.
2. Il transmet un **MemReq** au contrôleur mémoire M.
3. M récupère le bloc en DRAM et envoie **directement** les données au Requester (ShRep).
4. M envoie un ACK (MemRep) au directory.
5. Le directory passe l'état à **E** et set KeeperID = ID de R.

**Cas (b) — État valide (E, S, M ou O) :**
1. Le directory reçoit un ShReq de R.
2. Il transmet un **ForReq** (request forwarded) au Keeper K.
3. K envoie les données **directement** au Requester R (ShRep / msg 5).
4. K envoie un ACK (ForRep) au directory.
5. Les états appropriés changent dans le cache de K et dans le directory (règles MOESI).
6. Le directory essaie d'ajouter l'ID de R à la liste des sharers :
   - Si G=0 et la liste a de la place → ajouter R.
   - Si G=0 mais la liste est pleine → mettre G=1, stocker le count = 7 dans la liste.
   - Si G=1 → incrémenter le count de 1.

### 3.5 Opération pour une requête d'écriture (ExReq — requête de copie exclusive)

**Cas (a) — État Invalid :**
- Identique au cas de lecture, sauf que l'état final dans le directory est **M** (Modified) au lieu de E.

**Cas (b) — État valide (E, S, M ou O) :**
1. Le directory reçoit un ExReq de R.
2. Il effectue simultanément **deux actions** :
   - Transmet un **ForReq** au Keeper K.
   - **Si G=0** : envoie un **InvReq (multicast)** aux cœurs listés dans Sharers1–5 (en préfixant la liste des sharers à l'adresse d'invalidation, transmis sur ENet puis ONet).
   - **Si G=1** : envoie un **InvReq (broadcast)** à **tous les cœurs** de la puce.
3. Le Keeper K, en recevant le ForReq :
   - Envoie les données **directement** au Requester R (ExRep / msg 5).
   - Envoie un ForRep (ACK) au directory.
   - **Invalide son propre cache** pour ce bloc.
4. Chaque Sharer S, en recevant un InvReq :
   - Invalide son cache pour ce bloc.
   - Envoie un **InvRep (ACK)** au directory.
5. Le directory **attend tous les ACKs** :
   - Si G=0 : attend autant d'ACKs qu'il y a d'entrées dans Sharers1–5 (+ le ForRep du Keeper).
   - Si G=1 : attend exactement le nombre de sharers encodé dans le champ Sharers1–5.
6. Après réception de tous les ACKs : état → **M**, G → 0, KeeperID → ID de R.

### 3.6 Fonctionnement du multicast et broadcast

**Multicast (G=0) :** le message d'invalidation contient la liste des IDs de sharers en préfixe. Il est envoyé sur l'ENet intra-cluster vers le Hub, puis sur l'ONet. Chaque Hub récepteur filtre les messages destinés aux cœurs de son cluster et les redistribue via le BNet.

**Broadcast (G=1) :** le message est reçu par tous les Hubs et diffusé à tous les cœurs via le BNet de chaque cluster. Le count stocké dans le directory permet de n'attendre que le bon nombre d'ACKs, même si des cœurs n'avaient pas de copie (ils ignorent simplement l'invalidation).

### 3.7 Types de messages — Table complète

| Message | Numéro | Direction | Description |
|---------|--------|-----------|-------------|
| **ExReq** | 1 | R → H | Requête de copie exclusive |
| **ShReq** | 1 | R → H | Requête de copie partagée |
| **ExRep** | 2 ou 5 | M → R (2) ou K → R (5) | Copie exclusive retournée au requester |
| **ShRep** | 2 ou 5 | M → R (2) ou K → R (5) | Copie partagée retournée au requester |
| **InvReq** | 3 | H → Sharers | Invalidation envoyée aux sharers |
| **InvRep** | 4 | Sharer → H | ACK d'invalidation |
| **ForReq** | 3 | H → K | Requête (ShReq/ExReq) forwardée au Keeper |
| **ForRep** | 4 | K → H | ACK pour ForReq |
| **MemReq** | 2 | H → M | Requête forwardée au contrôleur mémoire |
| **MemRep** | 2 | M → H | ACK du contrôleur mémoire au directory |

*(H = Home/directory, R = Requester, K = Keeper, M = Memory controller, S = Sharer)*

### 3.8 Flux de messages pour les cas typiques

**Cas (a) — Read/Write miss, état Invalid (données en mémoire) :**
```
R --[1: ExReq/ShReq]--> H
H --[2: MemReq]-------> M
M --[2: ExRep/ShRep]--> R   (données directement)
M --[2: MemRep]-------> H   (ACK)
```
Latence : 3 hops (R→H, H→M, M→R), avec aller-retour off-chip.

**Cas (b) — Read miss, données présentes chez le Keeper :**
```
R --[1: ShReq]----> H
H --[3: ForReq]--> K
K --[5: ShRep]---> R   (données directement)
K --[4: ForRep]--> H   (ACK)
```
Latence : 3 hops (R→H, H→K, K→R), entièrement on-chip.

**Cas (c) — Write miss (ExReq), données partagées :**
```
R --[1: ExReq]-----------> H
H --[3: ForReq]----------> K
H --[3: InvReq]----------> S1, S2, ... (multicast ou broadcast)
K --[5: ExRep]-----------> R   (données)
K --[4: ForRep]----------> H   (ACK)
S1,S2,...--[4: InvRep]--> H   (ACKs d'invalidation)
(après tous les ACKs) : H met à jour l'état → M, G→0, KeeperID→R
```
Latence : dominée par la collecte de tous les ACKs d'invalidation.

---

## 4. Comparaison avec les directories classiques

### 4.1 Avantages en surface / mémoire

| Approche | Taille d'une entrée de directory (N=1024 cœurs) |
|----------|------------------------------------------------|
| Full-map (presence bits) | 1024 bits par entrée |
| ACKwise | ~61 bits (1 bit G + 10 bits KeeperID + 5×10 bits Sharers) |
| **Gain ACKwise** | **~16× plus compact** |

De plus, contrairement à un directory full-map, la taille d'une entrée ACKwise est **indépendante de N** : elle ne croît pas avec le nombre de cœurs.

### 4.2 Impact sur les performances

Les auteurs comparent l'ANet (avec ACKwise) à un réseau maillé électrique pur (pEMesh) :

**Latence de base on-chip (sans contention) :**
- ANet : 2,71 cycles en moyenne (43,3% de la latence totale)
- pEMesh : 5,12 cycles (55,3%)
- **ANet est 47,1% plus rapide** en latence de base.

**Délai de queuing on-chip :**
- ANet : 0,78 cycles (12,5%)
- pEMesh : 1,37 cycles (14,8%)
- **ANet a 43,1% moins de queuing delay**.

**Gain IPC global :**
- Jusqu'à **39% d'IPC supérieur** pour ANet vs pEMesh dans les conditions nominales.
- Avec un taux de partage élevé (wb=3 broadcast networks) : **57,5% de gain moyen**.
- Sur une plage de miss rates (1% à 15%) : **+33,6% en moyenne**.

### 4.3 Cas dégénérés et limitations

**1. Limitation du G bit — broadcast forcé :**
Quand G=1, **tout ExReq déclenche un broadcast global** à tous les 1 024 cœurs, même si en réalité le nombre réel de sharers est faible. Le directory ne peut plus faire de multicast ciblé. C'est un cas dégénéré pour les données très partagées.

**2. Nombre de BNets insuffisant :**
Avec un seul BNet par cluster (wb=1), le pEMesh **surpasse** l'ANet à cause des délais de queuing excessifs à la réception des broadcasts. Il faut au moins 2 BNets pour que l'ANet soit avantageux.

**3. Interdiction des silent evictions :**
ACKwise exige que chaque éviction d'un cœur soit signalée au directory. Cela génère du trafic supplémentaire (non modélisé dans l'évaluation) et augmente la pression sur le réseau.

**4. Saturation off-chip memory :**
Quand la bande passante mémoire off-chip est insuffisante (< Bthres), la performance est entièrement dominée par la latence mémoire et le réseau on-chip devient le facteur secondaire. L'avantage de l'ANet sur pEMesh ne se manifeste pleinement qu'au-delà de Bthres.

**5. Limite du ONet à 64 Hubs :**
La physique des composants optiques (plage WDM, puissance, pertes) limite l'ONet à 64 Hubs. Pour des puces à plus de 1 024 cœurs (avec n < 16 cœurs/cluster), cette limite deviendrait un frein. Pour une puce de 400 mm², un ONet à 384 Hubs consommerait toute la surface.

**6. Limites du modèle analytique (sources d'erreur reconnues) :**
- Les instruction cache misses ne sont pas modélisés.
- Le queuing delay au directory lui-même n'est pas modélisé.
- Les évictions ne sont pas modélisées (elles augmenteraient l'utilisation réseau).
- Le contrôle de congestion réseau n'est pas modélisé (queues infinies supposées).

---

## 5. Résultats expérimentaux

### 5.1 Méthodologie

L'évaluation est basée sur un **modèle analytique** (pas de simulation cycle-à-cycle), car les simulateurs actuels (2009) ne peuvent pas simuler des systèmes à 1 024 cœurs. Le modèle est un modèle de processeur **in-order** focalisé sur la latence des requêtes mémoire, avec modélisation du queuing M/D/1 (queues infinies) dans les réseaux on-chip et off-chip. Le modèle utilise une recherche binaire pour résoudre la non-linéarité des équations de CPI (dépendance mutuelle entre CPI et queuing delays).

**Benchmark synthétique** dérivé de PARSEC :
- Fréquence de références données : 0,3 par cycle
- Fraction reads / writes : 2/3 / 1/3
- Miss rate : 4%
- Nombre moyen de sharers : 4
- Fraction mémoire off-chip : 70%
- Fraction writes causant des broadcasts d'invalidation : 10%

**Configuration système :**
- 1 024 cœurs, 64 clusters, 1 GHz, cache line 64 B
- Latence hop électrique : 1 ns ; propagation optique : 2,5 ns
- Off-chip bandwidth : 280 GB/s
- Largeur de lien ONet : 2 lanes × 32 bits = 64 bits par canal optique

### 5.2 Résultats quantitatifs

**Comparaison globale ANet vs pEMesh :**

| Métrique | ANet | pEMesh | Gain ANet |
|----------|------|--------|-----------|
| Latence mémoire moyenne | 6,26 cycles | 9,26 cycles | −32,4% |
| Latence base on-chip | 2,71 cycles (43,3%) | 5,12 cycles (55,3%) | −47,1% |
| Queuing delay on-chip | 0,78 cycles (12,5%) | 1,37 cycles (14,8%) | −43,1% |
| Latence off-chip | 2,77 cycles | 2,77 cycles | 0% |
| **IPC (wb=2, 4 sharers moy.)** | **≈ 39% supérieur** | — | +39% |

**Effet du nombre de BNets (wb) :**
- wb=1 : pEMesh > ANet (queuing excessif à la réception)
- wb=2 : ANet +39% sur pEMesh (choix retenu, bon compromis coût/perf.)
- wb=3 : ANet légèrement supérieur à wb=2 (+quelques % pour cas nominal)
- wb=4, wb=5 : gains marginaux supplémentaires

**Effet du miss rate (1% à 15%) :**
- ANet supérieur sur toute la plage.
- Gain moyen : **+33,6%**.

**Effet de la bande passante off-chip (40 à 400 GB/s) :**
- En dessous de Bthres : performance dominée par la mémoire, ANet ≈ pEMesh.
- Au-dessus de Bthres : **ANet +39%** sur pEMesh.
- Au-delà de Bsat : les deux saturent (off-chip BW n'est plus le goulot).

**Effet du nombre de sharers (1 à 64) :**
- pEMesh se dégrade fortement avec le nombre de sharers (broadcasts électriques coûteux).
- ANet (wb=2) se dégrade aussi, mais moins vite (queuing au Hub).
- ANet (wb=3) surpasse pEMesh de **57,5% en moyenne** sur toute la plage.

---

## 6. Ce qu'il faut retenir pour l'implémenter dans gem5

### 6.1 Structures de données à modéliser

**1. Entrée de directory :**
```cpp
struct ACKwiseDirectoryEntry {
    MOESIState state;        // I, E, S, M, O
    bool global_bit;         // G : sharers ont dépassé la capacité de la liste
    NodeID keeper_id;        // ID du cœur avec copie à jour
    union {
        NodeID sharer_ids[5]; // G=0 : liste des sharers (jusqu'à 5 IDs)
        uint32_t sharer_count; // G=1 : nombre total de sharers
    };
};
```

**2. Table de directory :**
- Distribuée : chaque cœur (nœud gem5) héberge une portion du directory.
- La politique d'allocation des adresses aux "home nodes" est **statique** (par exemple, adresse % N_cores).

**3. Compteur d'ACKs en attente :**
- Le directory doit maintenir, pour chaque ExReq en cours de traitement, un compteur des ACKs attendus (InvReps + ForRep du Keeper).
- Quand G=1, ce compteur est initialisé à `sharer_count`. Quand G=0, il est calculé directement depuis la liste.

### 6.2 Machines à états à implémenter

**Pour le directory (Home node) :**

| Transition | Événement | Action |
|-----------|-----------|--------|
| I → E | ShReq reçu | Envoyer MemReq à MC ; set KeeperID |
| I → M | ExReq reçu | Envoyer MemReq à MC ; set KeeperID |
| E/S/M/O → S | ShReq reçu | Forward ForReq au Keeper ; ajouter requester à la liste (ou incrémenter count) |
| E/S/M/O → M | ExReq reçu | Forward ForReq au Keeper + InvReq aux sharers ; attendre ACKs |
| en_attente_ACKs → M | Tous ACKs reçus | Set state=M, G=0, KeeperID=requester |

**Pour le cache L1 (Requester) :**

| Transition | Événement | Action |
|-----------|-----------|--------|
| I → S | ShRep reçu | Mettre à jour le cache, state=S |
| I → M | ExRep reçu | Mettre à jour le cache, state=M |
| M → I | InvReq reçu | Invalider, envoyer InvRep au directory |
| S → I | InvReq reçu | Invalider, envoyer InvRep au directory |
| E → M | Write hit | Transition silencieuse locale |

**Pour le Keeper :**

| Transition | Événement | Action |
|-----------|-----------|--------|
| M/O → I/S | ForReq (ExReq) reçu | Envoyer données au Requester (ExRep/msg 5) + ForRep au directory + invalider |
| E/S/O → S | ForReq (ShReq) reçu | Envoyer données au Requester (ShRep/msg 5) + ForRep au directory |

### 6.3 Comportements spécifiques à implémenter dans gem5/SLICC

1. **Tracking des sharers avec G bit :** La logique de transition G=0 → G=1 (quand la liste est pleine) et l'incrémentation du count quand G=1.

2. **Pas de silent evictions :** Dans le contrôleur de cache L1, toute expulsion d'une ligne en état non-Invalid doit envoyer un message de notification au directory (un type de message supplémentaire non détaillé dans le papier — à définir, par exemple un `WBReq` pour writeback ou `PutS`/`PutM` dans la terminologie gem5).

3. **Multicast ciblé (G=0) :** Quand G=0, les InvReqs sont envoyés **uniquement** aux IDs listés dans Sharers1–5. Dans gem5/Garnet, cela se traduit par autant de messages point-à-point que d'entrées dans la liste.

4. **Broadcast (G=1) :** Quand G=1, un seul message est émis avec destination = ALL. Dans gem5, le réseau Garnet ne supporte pas nativement le broadcast ; il faudra soit émuler cela par des messages point-à-point vers tous les nœuds, soit implémenter un mécanisme de multicast dans Garnet.

5. **Comptage des ACKs :** Le directory entre dans un état d'attente (`BUSY` / `WaitingACKs`) après avoir émis les InvReqs, et ne complète la transition vers l'état M qu'après avoir reçu **exactement** le nombre attendu d'ACKs (calculé depuis la liste ou le count).

6. **Rôle du Keeper :** Le Keeper est le seul cœur qui peut fournir directement les données (forwarding). Il doit être capable de répondre à un ForReq en envoyant simultanément les données au Requester et un ACK au directory.

7. **Distribution du directory :** Dans gem5, il faut configurer le système de telle façon que chaque `RubySystem` node corresponde à un cœur qui est à la fois processeur, cache L1, et directory partiel. La politique home = `address % N` est à implémenter dans le `AddressMapping`.

8. **Interface avec le réseau Garnet :** ACKwise utilise les deux réseaux (EMesh et ANet/ONet) différemment selon le type de message. Pour une implémentation dans gem5, on pourrait modéliser l'ONet comme un réseau à latence fixe (2,5 ns) et faible contention, et l'EMesh comme réseau maillé standard.

---

## Synthèse

ACKwise est un protocole MOESI à directory compact qui remplace le vecteur de présence O(N) par une liste de 5 IDs + 1 Keeper + 1 bit global. Il exploite les broadcasts optiques de l'architecture ATAC pour gérer les invalidations à grande échelle sans stocker l'identité de tous les sharers. La clé de son implémentation dans gem5 est : (1) la structure d'entrée de directory avec transition G=0/G=1, (2) l'interdiction des silent evictions, et (3) la logique de comptage des ACKs en attente pour les ExReqs.