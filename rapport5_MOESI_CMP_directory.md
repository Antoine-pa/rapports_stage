# Rapport 5 — Protocole MOESI_CMP_directory dans gem5
## Analyse détaillée et pistes pour un directory à taille limitée

---

## 1. Vue d'ensemble architecturale

### 1.1 Position dans gem5

Le protocole `MOESI_CMP_directory` est un protocole de cohérence de cache **à trois niveaux de contrôleurs** dans gem5/Ruby :

```
┌─────────────────────────────────────────────────────────────────┐
│                     CPU (Sequencer)                               │
│                          │                                        │
│                          ▼                                        │
│              ┌──────────────────────┐                            │
│              │   L1Cache Controller  │  (par cœur, I/D)          │
│              │   États: I,S,O,M,MM   │                            │
│              └──────────┬───────────┘                            │
│                         │  vnet 0 (L1↔L2 requests)               │
│                         │  vnet 2 (L1↔L2 responses)              │
│                         ▼                                        │
│              ┌──────────────────────┐                            │
│              │   L2Cache Controller  │  (partagé, bancs)         │
│              │   États: I,ILS,ILX,   │                            │
│              │   ILO,ILOX,ILOS,S,O,  │                            │
│              │   OLS,OLSX,SLS,M      │                            │
│              │   + ~50 transitoires  │                            │
│              └──────────┬───────────┘                            │
│                         │  vnet 1 (L2↔Dir requests/forwards)     │
│                         │  vnet 2 (L2↔Dir responses)             │
│                         ▼                                        │
│              ┌──────────────────────┐                            │
│              │  Directory Controller │  (global)                 │
│              │  États: I,S,O,M       │                            │
│              │  + DirectoryMemory    │                            │
│              └──────────┬───────────┘                            │
│                         │  requestToMemory / responseFromMemory   │
│                         ▼                                        │
│              ┌──────────────────────┐                            │
│              │   Mémoire (DRAM)      │                            │
│              └──────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Fichiers source du protocole

| Fichier | Rôle |
|---------|------|
| `MOESI_CMP_directory.slicc` | Déclaration du protocole, inclusion des `.sm` |
| `MOESI_CMP_directory-msg.sm` | Types de messages (Request, Response, Trigger) |
| `MOESI_CMP_directory-L1cache.sm` | Contrôleur L1 (interface CPU) |
| `MOESI_CMP_directory-L2cache.sm` | Contrôleur L2 (intermédiaire, directory local) |
| `MOESI_CMP_directory-dir.sm` | Contrôleur Directory (global, interface mémoire) |
| `MOESI_CMP_directory-dma.sm` | Contrôleur DMA |

### 1.3 Réseaux virtuels (virtual networks)

Le protocole utilise **3 réseaux virtuels** pour éviter les deadlocks :

| vnet | Direction | Type | Contenu |
|------|-----------|------|---------|
| 0 | L1 ↔ L2 | request | GETS, GETX, PUTS, PUTX, PUTO, INV, writebacks |
| 1 | L2 ↔ Dir | request/forward | GETS, GETX, PUTX, PUTO, INV, forwards |
| 2 | L1 ↔ L2 ↔ Dir | response | DATA, ACK, UNBLOCK, WB_ACK/NACK |

---

## 2. Types de messages

### 2.1 Requêtes de cohérence (`CoherenceRequestType`)

| Message | Description | Usage |
|---------|-------------|-------|
| `GETX` | Get eXclusive | Demande de copie exclusive (écriture) |
| `GETS` | Get Shared | Demande de copie partagée (lecture) |
| `PUTX` | Put eXclusive | Writeback d'une ligne en M |
| `PUTO` | Put Owned | Writeback d'une ligne en O |
| `PUTO_SHARERS` | Put Owned + sharers | Writeback O mais des sharers existent localement |
| `PUTS` | Put Shared | Writeback d'une ligne en S |
| `INV` | Invalidation | Invalider les copies des sharers |
| `WRITEBACK_CLEAN_DATA` | Clean WB (données) | Writeback propre avec données |
| `WRITEBACK_CLEAN_ACK` | Clean WB (sans données) | Writeback propre sans données |
| `WRITEBACK_DIRTY_DATA` | Dirty WB (données) | Writeback sale avec données |
| `DMA_READ` | Lecture DMA | Requête DMA de lecture |
| `DMA_WRITE` | Écriture DMA | Requête DMA d'écriture |

### 2.2 Réponses de cohérence (`CoherenceResponseType`)

| Message | Description |
|---------|-------------|
| `ACK` | Acquittement (pas de copie chez le répondeur) |
| `DATA` | Données (copie partagée) |
| `DATA_EXCLUSIVE` | Données (copie exclusive, personne d'autre ne l'a) |
| `UNBLOCK` | Déblocage du directory (requête satisfaite, partagée) |
| `UNBLOCK_EXCLUSIVE` | Déblocage exclusif (requête satisfaite, E/M) |
| `WB_ACK` | Acquittement de writeback |
| `WB_ACK_DATA` | Acquittement de writeback (demande données) |
| `WB_NACK` | Rejet de writeback (race condition) |
| `DMA_ACK` | Acquittement DMA |

### 2.3 Structure des messages

**RequestMsg :**
```
{ Addr addr, int Len, CoherenceRequestType Type, MachineID Requestor,
  MachineType RequestorMachine, NetDest Destination, DataBlock DataBlk,
  int Acks, MessageSizeType MessageSize, RubyAccessMode AccessMode,
  PrefetchBit Prefetch }
```

Le champ `Acks` dans les requêtes forwardées transporte le **nombre de sharers** que le demandeur devra invalider. C'est le mécanisme central d'acquittement.

**ResponseMsg :**
```
{ Addr addr, CoherenceResponseType Type, MachineID Sender,
  MachineType SenderMachine, NetDest Destination, DataBlock DataBlk,
  bool Dirty, int Acks, MessageSizeType MessageSize }
```

---

## 3. Le contrôleur Directory (`-dir.sm`) — Analyse complète

### 3.1 Paramètres du contrôleur

```
machine(MachineType:Directory, "Directory protocol")
: DirectoryMemory * directory;
  Cycles directory_latency := 6;
  Cycles to_memory_controller_latency := 1;
```

- **`DirectoryMemory * directory`** : structure de données qui maintient l'état de cohérence de chaque bloc mémoire. Taille **illimitée** (une entrée par ligne de cache de toute la mémoire physique). **C'est ici que se situe le problème de scalabilité.**
- **`directory_latency`** : 6 cycles de latence pour les réponses du directory.
- **`to_memory_controller_latency`** : 1 cycle pour les requêtes vers la mémoire.

### 3.2 Structure d'une entrée du directory

```slicc
structure(Entry, desc="...", interface='AbstractCacheEntry', main="false") {
    State DirectoryState;       // État MOESI du bloc
    NetDest Sharers;            // Bitvector des sharers (TAILLE O(N) !)
    NetDest Owner;              // Bitvector du propriétaire (1 seul bit à 1)
    int WaitingUnblocks;        // Nombre d'unblocks attendus
}
```

**Points critiques pour le stage :**
- `NetDest Sharers` est un **bitvector pleine taille** sur tous les nœuds L2Cache du système. Pour N nœuds, c'est un vecteur de N bits. C'est **exactement le problème O(N)** que le stage doit résoudre.
- `NetDest Owner` est aussi un bitvector mais avec un seul bit à 1 (identifie le propriétaire en état M ou O).
- `main="false"` indique que la replacement policy ignore cette entrée (pas d'éviction possible dans le directory actuel).

### 3.3 États stables du Directory

| État | Permission | Description |
|------|-----------|-------------|
| **I** | Read_Write | Aucun cache ne possède la ligne. Mémoire à jour. |
| **S** | Read_Write | Plusieurs caches possèdent une copie partagée. Mémoire à jour. |
| **O** | Maybe_Stale | Un cache est propriétaire (dirty), potentiellement avec sharers. Mémoire peut-être périmée. |
| **M** | Maybe_Stale | Un seul cache possède une copie exclusive dirty. Mémoire périmée. |

### 3.4 États transitoires du Directory

| État | Description | Origine → Destination |
|------|-------------|----------------------|
| `IS_M` | Attente données mémoire (requête de lecture) | I → S (via mémoire) |
| `IS` | Données envoyées, attente unblock | I → S |
| `SS` | En S, attente unblocks multiples (GETS concurrent) | S → S |
| `OO` | En O, attente unblocks (GETS concurrent) | O → O |
| `MO` | En M, GETS reçu, forward envoyé | M → O |
| `MM_M` | Attente données mémoire pour GETX | I/S → M (via mémoire) |
| `MM` | Données envoyées pour GETX, attente unblock exclusif | → M |
| `MI` | Writeback en cours (PUTX reçu) | M → I |
| `MIS` | Writeback PUTO_SHARERS, garder dans sharers | M → S |
| `OS` | Writeback PUTO en cours | O → S |
| `OSS` | Writeback PUTO_SHARERS en cours | O → S |
| `WBI` | Writeback envoyé à mémoire, destination I | → I |
| `WBS` | Writeback envoyé à mémoire, destination S | → S |
| `XI_M` | DMA read, attente mémoire | I → I |
| `XI_M_U` | DMA write partiel, attente mémoire | I → I |
| `XI_U` | DMA, attente unblock | → I |
| `OI_D` | En O/M, DMA write, attente données du cache | O/M → I |
| `OD` | En O, DMA read en cours | O → O |
| `MD` | En M, DMA read en cours | M → M |

### 3.5 Événements du Directory

| Événement | Déclencheur |
|-----------|-------------|
| `GETX` | Requête de lecture exclusive reçue d'un L2 |
| `GETS` | Requête de lecture partagée reçue d'un L2 |
| `PUTX` | Writeback d'une ligne exclusive |
| `PUTO` | Writeback d'une ligne owned |
| `PUTO_SHARERS` | Writeback owned, mais des sharers locaux existent |
| `Unblock` | Unblock partagé (requête satisfaite) |
| `Last_Unblock` | Dernier unblock attendu |
| `Exclusive_Unblock` | Unblock exclusif (passage en E/M) |
| `Clean_Writeback` | Writeback propre (pas de données) |
| `Dirty_Writeback` | Writeback sale (avec données) |
| `Memory_Data_Cache` | Données retournées par la mémoire (pour un cache) |
| `Memory_Data_DMA` | Données retournées par la mémoire (pour DMA) |
| `Memory_Ack` | Acquittement d'écriture mémoire |
| `DMA_READ` | Requête DMA de lecture |
| `DMA_WRITE_LINE` | Requête DMA d'écriture (ligne complète) |
| `DMA_WRITE_PARTIAL` | Requête DMA d'écriture (partielle) |
| `Data` | Données reçues d'un cache (pour DMA) |
| `All_Acks` | Tous les acquittements reçus (trigger interne) |

### 3.6 Transitions principales du Directory

#### Lecture depuis l'état Invalid (I → IS_M → IS → S)

```
I + GETS → IS_M :
  1. allocDirEntry          // Allouer l'entrée du directory
  2. v_allocateTBE          // Allouer un TBE (transaction buffer)
  3. qf_queueMemoryFetchRequest  // Envoyer requête à la mémoire
  4. i_popIncomingRequestQueue   // Consommer le message

IS_M + Memory_Data_Cache → IS :
  1. d_sendDataMsg          // Envoyer données au demandeur
  2. q_popMemQueue          // Consommer réponse mémoire

IS + Unblock → S :
  1. w_deallocateTBE        // Libérer le TBE
  2. m_addUnlockerToSharers // Ajouter le demandeur aux Sharers
  3. j_popIncomingUnblockQueue
```

#### Écriture depuis l'état Invalid (I → MM_M → MM → M)

```
I + GETX → MM_M :
  1. allocDirEntry
  2. v_allocateTBE
  3. qf_queueMemoryFetchRequest
  4. i_popIncomingRequestQueue

MM_M + Memory_Data_Cache → MM :
  1. d_sendDataMsg          // Envoyer données + AckCount au demandeur
  2. q_popMemQueue

MM + Exclusive_Unblock → M :
  1. w_deallocateTBE
  2. cc_clearSharers        // Effacer tous les sharers
  3. e_ownerIsUnblocker     // Enregistrer le nouveau propriétaire
  4. j_popIncomingUnblockQueue
```

#### Écriture depuis l'état Shared (S + GETX) — avec invalidations

```
S + GETX → MM_M :
  1. v_allocateTBE
  2. qf_queueMemoryFetchRequest
  3. g_sendInvalidations    // *** ENVOYER INV À TOUS LES SHARERS ***
  4. i_popIncomingRequestQueue
```

**C'est ici que `NetDest Sharers` est utilisé pour l'invalidation multicast.** L'action `g_sendInvalidations` envoie un message INV avec `Destination = Sharers - Requestor`.

#### Forward depuis l'état Modified (M + GETS)

```
M + GETS → MO :
  1. v_allocateTBE
  2. f_forwardRequest       // Forward le GETS au propriétaire (Owner)
  3. i_popIncomingRequestQueue

MO + Unblock → O :
  1. w_deallocateTBE
  2. m_addUnlockerToSharers // Ajouter le demandeur aux Sharers
  3. j_popIncomingUnblockQueue
```

#### Forward exclusif depuis Modified (M + GETX)

```
M + GETX → MM :
  1. f_forwardRequest       // Forward le GETX au propriétaire (Owner)
  2. i_popIncomingRequestQueue

MM + Exclusive_Unblock → M :
  1. w_deallocateTBE
  2. cc_clearSharers
  3. e_ownerIsUnblocker     // Nouveau propriétaire
  4. j_popIncomingUnblockQueue
```

#### Écriture depuis Owned (O + GETX) — avec forward + invalidations

```
O + GETX → MM :
  1. f_forwardRequest       // Forward au propriétaire (Owner)
  2. g_sendInvalidations    // INV à tous les Sharers
  3. i_popIncomingRequestQueue
```

#### Writeback (M + PUTX)

```
M + PUTX → MI :
  1. v_allocateTBE
  2. a_sendWriteBackAck     // Envoyer WB_ACK au demandeur
  3. i_popIncomingRequestQueue

MI + Dirty_Writeback → WBI :
  1. c_clearOwner
  2. cc_clearSharers
  3. qw_queueMemoryWBFromCacheRequest  // Écrire données en mémoire
  4. i_popIncomingRequestQueue

WBI + Memory_Ack → I :
  1. clearWBAck
  2. w_deallocateTBE
  3. deallocDirEntry        // *** DÉSALLOUER L'ENTRÉE ***
  4. q_popMemQueue
```

### 3.7 Mécanisme d'invalidation — Détail de `g_sendInvalidations`

```slicc
action(g_sendInvalidations, "g", desc="Send invalidations to sharers, not including the requester") {
    peek(requestQueue_in, RequestMsg) {
      if ((getDirectoryEntry(in_msg.addr).Sharers.count() > 1) ||
          ((getDirectoryEntry(in_msg.addr).Sharers.count() > 0) &&
           (getDirectoryEntry(in_msg.addr).Sharers.isElement(in_msg.Requestor) == false))) {
        enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
          out_msg.addr := address;
          out_msg.Type := CoherenceRequestType:INV;
          out_msg.Requestor := in_msg.Requestor;
          out_msg.Destination.addNetDest(getDirectoryEntry(in_msg.addr).Sharers);
          out_msg.Destination.remove(in_msg.Requestor);
          out_msg.MessageSize := MessageSizeType:Invalidate_Control;
        }
      }
    }
}
```

**Observations clés :**
1. Le message INV est envoyé avec `Destination = Sharers - Requestor` (multicast ciblé).
2. Le message est un **seul enqueue** avec une destination multicast : Garnet se charge de répliquer le message vers chaque nœud de la `NetDest`.
3. Le champ `Acks` dans le message `d_sendDataMsg` contient le nombre d'ACKs que le demandeur devra collecter (calculé depuis `Sharers.count()`).

### 3.8 Mécanisme d'AckCount

L'action `d_sendDataMsg` calcule le nombre d'ACKs et l'inclut dans la réponse DATA :

```slicc
if (getDirectoryEntry(in_msg.addr).Sharers.isElement(in_msg.OriginalRequestorMachId)) {
    out_msg.Acks := (getDirectoryEntry(in_msg.addr).Sharers.count()) - 1;
} else {
    out_msg.Acks := getDirectoryEntry(in_msg.addr).Sharers.count();
}
```

Le demandeur (dans le contrôleur L2Cache) maintient un compteur `NumExtPendingAcks` dans son TBE et décrémente à chaque ACK reçu. Quand il atteint 0, un `TriggerType:ALL_ACKS` est injecté dans la `triggerQueue`.

### 3.9 Gestion des conflits (stall/recycle)

Les requêtes arrivant pendant un état transitoire sont **recyclées** :

```slicc
transition({MM_M, MM, MO, MI, MIS, OS, OSS, WBI, WBS, XI_M, XI_M_U, XI_U, OI_D, OD, MD},
           {GETS, GETX, PUTO, PUTO_SHARERS, PUTX, DMA_READ, DMA_WRITE_LINE, DMA_WRITE_PARTIAL}) {
    zz_recycleRequest;
}
```

Le recyclage remet le message en queue après un délai (`recycle_latency`).

---

## 4. Le contrôleur L2Cache — Rôle d'intermédiaire

### 4.1 Double rôle du L2

Le L2 joue un rôle crucial dans ce protocole : il est à la fois **cache** et **directory local** pour les L1 de son cluster.

**Quand la donnée est en L2 (états S, O, M, OLS, OLSX, SLS) :**
- Le L2 répond directement aux requêtes L1 sans contacter le directory global.
- Il maintient un `localDirectory` (structure `PerfectCacheMemory`) qui traque quels L1 locaux ont une copie.

**Quand la donnée n'est PAS en L2 mais des L1 locaux l'ont (états ILS, ILX, ILO, ILOX, ILOS, ILOSX) :**
- Le L2 forward les requêtes vers le L1 local propriétaire/sharer.
- Il contacte le directory global seulement si nécessaire (GETX qui nécessite des invalidations globales).

### 4.2 Structure du directory local (L2)

```slicc
structure(DirEntry, desc="...", interface="AbstractCacheEntry", main="false") {
    NetDest Sharers;       // L1 locaux qui partagent
    MachineID Owner;       // L1 local propriétaire
    bool OwnerValid;       // Owner est valide ?
    State DirState;        // État du directory local
}
```

Ce `localDirectory` est de type `PerfectCacheMemory` — taille illimitée (parfait pour la modélisation mais irréaliste en hardware).

### 4.3 Interactions L2 ↔ Directory Global

Le L2 communique avec le directory global sur vnet 1 (requests) et vnet 2 (responses). Les messages globaux incluent :
- `GETS`, `GETX` : quand le L2 ne peut pas satisfaire localement
- `PUTX`, `PUTO`, `PUTO_SHARERS` : writebacks vers le directory
- `WRITEBACK_DIRTY_DATA`, `WRITEBACK_CLEAN_ACK` : données de writeback
- `UNBLOCK`, `UNBLOCK_EXCLUSIVE` : signaler que la requête est complète

---

## 5. Le contrôleur L1Cache

### 5.1 États stables du L1

| État | Description |
|------|-------------|
| `I` | Invalid |
| `S` | Shared (lecture seule) |
| `O` | Owned (dirty, lecture seule, peut répondre aux forwards) |
| `M` | Modified (dirty, exclusif, read/write) |
| `M_W` | Modified, en lockout (timeout pour éviter livelock) |
| `MM` | Modified et localement modifié |
| `MM_W` | Modified localement modifié, en lockout |

### 5.2 États transitoires du L1

| État | Description |
|------|-------------|
| `IM` | Issued GETX, attend données |
| `IS` | Issued GETS, attend données |
| `SM` | Issued GETX depuis S, attend données (copie ancienne disponible) |
| `OM` | Issued GETX depuis O, données reçues, attend ACKs |
| `SI` | Issued PUTS, attend WB_ACK |
| `OI` | Issued PUTO, attend WB_ACK |
| `MI` | Issued PUTX, attend WB_ACK |
| `II` | Issued PUT, a vu Fwd, attend WB_ACK |

### 5.3 Évictions du L1

Le L1 **ne fait PAS d'évictions silencieuses**. Chaque éviction envoie un message au L2 :
- `PUTS` pour une ligne en S
- `PUTO` pour une ligne en O
- `PUTX` pour une ligne en M/MM

C'est un point crucial : **l'interdiction des évictions silencieuses est déjà en place** dans ce protocole, ce qui est requis pour Ackwise.

---

## 6. Flux de messages complets — Diagrammes de séquence

### 6.1 Read miss complet (I → S, données en mémoire)

```
L1 ──[GETS]──→ L2 ──[GETS]──→ Directory
                              Directory: état I → IS_M
                              Directory ──[MemRead]──→ DRAM
                              DRAM ──[MemData]──→ Directory
                              Directory: état IS_M → IS
                              Directory ──[DATA]──→ L2
L2 ──[DATA]──→ L1
L1: état I → IS → S
L1 ──[UNBLOCK]──→ L2 ──[UNBLOCK]──→ Directory
                              Directory: addSharers(L2), état IS → S
```

### 6.2 Write miss avec invalidation (S → M, sharers existent)

```
L1 ──[GETX]──→ L2 ──[GETX]──→ Directory
                              Directory: état S
                              Directory ──[MemRead]──→ DRAM (pour les données)
                              Directory ──[INV]──→ L2_sharer1, L2_sharer2, ...
                              (chaque sharer invalide localement, L2 invalide ses L1)
                              L2_sharer ──[ACK]──→ L2_demandeur (via le Requestor dans INV)
                              DRAM ──[MemData]──→ Directory
                              Directory ──[DATA + AckCount]──→ L2_demandeur
L2_demandeur: attend tous les ACKs
              quand tous reçus: ──[DATA_EXCLUSIVE]──→ L1
L1: état IM → M
L1 ──[EXCLUSIVE_UNBLOCK]──→ L2 ──[EXCLUSIVE_UNBLOCK]──→ Directory
                              Directory: clearSharers, setOwner(L2), état → M
```

### 6.3 Forward (M + GETS → O avec forwarding)

```
L1_new ──[GETS]──→ L2_new ──[GETS]──→ Directory
                              Directory: état M, Owner = L2_owner
                              Directory ──[GETS forward]──→ L2_owner
L2_owner ──[GETS forward]──→ L1_owner
L1_owner: état M → O, envoie données
L1_owner ──[DATA]──→ L2_owner ──[DATA]──→ L2_new ──[DATA]──→ L1_new
L1_new ──[UNBLOCK]──→ ... → Directory
                              Directory: addSharers(L2_new), état M → O (ou MO → O)
```

---

## 7. Problème de scalabilité et objectif du stage

### 7.1 Le problème : `NetDest` en O(N)

La structure `NetDest Sharers` dans l'entrée du directory est un **bitvector de N bits** où N est le nombre total de nœuds L2Cache. Pour un système à 64 cœurs, c'est 64 bits. Pour 1024 cœurs, c'est 1024 bits = 128 octets **par entrée de directory**, soit le double d'une ligne de cache de 64 octets.

De plus, le directory actuel est de **taille illimitée** (`DirectoryMemory` alloue paresseusement une entrée pour chaque ligne de cache de la mémoire physique). En hardware réel, le directory est un cache (taille limitée) et la sharelist doit être compacte.

### 7.2 Objectif en deux étapes

1. **Étape 1 :** Rendre le directory de **taille limitée** (nombre d'entrées borné, nécessité d'une politique de remplacement et de gestion des évictions/recalls).
2. **Étape 2 :** Remplacer `NetDest Sharers` par une représentation compacte (Ackwise, DCC, ou autre).

---

## 8. Comment rendre le directory limité

### 8.1 Approche recommandée : Utiliser `CacheMemory` comme directory

Au lieu de `DirectoryMemory` (taille illimitée), utiliser `CacheMemory` qui possède déjà :
- Associativité configurable (N-way set-associative)
- Politique de remplacement (LRU, pseudo-random, etc.)
- Gestion des tags et sets

**Modification dans le `.sm` :**

```slicc
machine(MachineType:Directory, "Limited Directory protocol")
: CacheMemory * directoryCache;    // Remplace DirectoryMemory
  Cycles directory_latency := 6;
  Cycles to_memory_controller_latency := 1;
  // ... message buffers inchangés ...
```

### 8.2 Gestion de l'éviction d'une entrée directory (Recall)

Quand le directory cache est plein et qu'une nouvelle entrée doit être créée, il faut **évicter** une entrée existante. Cela nécessite un mécanisme de **recall** :

1. Le directory sélectionne une victime (LRU par exemple).
2. Si la victime est en état **I** : simple désallocation, pas de message.
3. Si la victime est en état **S** : envoyer des invalidations (`INV`) à tous les sharers, attendre les ACKs, puis libérer l'entrée.
4. Si la victime est en état **O** ou **M** : forwarder un message au propriétaire pour qu'il fasse un writeback, invalider les sharers, attendre tous les ACKs et le writeback, écrire en mémoire, puis libérer l'entrée.

**Nouveaux états transitoires nécessaires :**

```slicc
// Recall states
S_Recall, AccessPermission:Busy, desc="Recall en cours, était en S, attend INV_ACKs";
O_Recall, AccessPermission:Busy, desc="Recall en cours, était en O, attend WB + INV_ACKs";
M_Recall, AccessPermission:Busy, desc="Recall en cours, était en M, attend WB";
```

**Nouvel événement :**

```slicc
Replacement, desc="Éviction d'une entrée du directory cache (recall)";
```

**Nouvelles actions :**

```slicc
action(recall_sendInvalidations, "rsi", desc="Send recall invalidations to all sharers") {
    // Envoyer INV à tous les sharers de la victime
    enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
        out_msg.Type := CoherenceRequestType:INV;
        out_msg.Destination.addNetDest(getDirectoryEntry(address).Sharers);
        // ... 
    }
}

action(recall_forwardWriteback, "rfw", desc="Ask owner to writeback for recall") {
    // Demander au propriétaire de renvoyer ses données
    enqueue(forwardNetwork_out, RequestMsg, directory_latency) {
        out_msg.Type := CoherenceRequestType:GETX;
        out_msg.Requestor := machineID;  // Le directory est le demandeur
        out_msg.Destination.addNetDest(getDirectoryEntry(address).Owner);
        // ...
    }
}
```

### 8.3 Transitions pour le recall

```slicc
// Déclenchement du recall (quand le cache est plein et qu'on a besoin d'allouer)
transition(S, Replacement, S_Recall) {
    recall_sendInvalidations;
}

transition(S_Recall, All_Acks, I) {
    deallocDirEntry;
    popTriggerQueue;
}

transition(M, Replacement, M_Recall) {
    recall_forwardWriteback;
}

transition(M_Recall, Dirty_Writeback, WBI) {
    c_clearOwner;
    qw_queueMemoryWBFromCacheRequest;
    i_popIncomingRequestQueue;
}

transition(O, Replacement, O_Recall) {
    recall_forwardWriteback;
    recall_sendInvalidations;
}
```

### 8.4 Alternative : Modifier `DirectoryMemory` en C++

On peut aussi modifier directement `src/mem/ruby/structures/DirectoryMemory.{hh,cc}` pour limiter le nombre d'entrées :

```cpp
// DirectoryMemory.hh
class DirectoryMemory {
    // Ajouter :
    uint64_t m_max_entries;     // Limite d'entrées
    uint64_t m_current_entries; // Compteur courant
    // Politique de remplacement (LRU, etc.)
    
    // Modifier allocate() pour gérer le cas "plein"
    AbstractCacheEntry* allocate(Addr addr, AbstractCacheEntry* entry);
    // Retourner nullptr si plein → le contrôleur SLICC gère le recall
};
```

### 8.5 Comparaison des approches

| Approche | Complexité dev | Réalisme HW | Flexibilité |
|----------|---------------|-------------|-------------|
| `CacheMemory` pour le directory | Moyenne | Élevé | Très bonne (politiques de remplacement existantes) |
| Modifier `DirectoryMemory` | Élevée | Élevé | Bonne |
| Logique de limitation dans SLICC pur | Faible | Moyen | Limitée |

**Recommandation : utiliser `CacheMemory`** car gem5 fournit déjà toute l'infrastructure (remplacement, associativité, tags). Il suffit de remplacer la déclaration de paramètre et d'adapter les fonctions `getDirectoryEntry`/`allocateDirectoryEntry`.

---

## 9. Passer à une sharelist compacte — Ackwise vs alternatives

### 9.1 Option 1 : Ackwise (simple mais performances moyennes)

**Principe :**
- Remplacer `NetDest Sharers` par une liste de k pointeurs + 1 bit global (G).
- G=0 : les k pointeurs stockent les IDs des sharers.
- G=1 : un compteur remplace la liste (broadcast + comptage ACKs).

**Modification de l'entrée directory :**

```slicc
structure(Entry, desc="Entrée directory Ackwise",
          interface="AbstractCacheEntry", main="false") {
    State DirectoryState;
    // Remplace NetDest Sharers :
    bool GlobalBit;            // G bit : overflow
    MachineID Owner;           // Propriétaire (Keeper)
    MachineID Sharer0;         // Pointeur sharer 1
    MachineID Sharer1;         // Pointeur sharer 2
    MachineID Sharer2;         // Pointeur sharer 3
    MachineID Sharer3;         // Pointeur sharer 4
    int NumSharers;            // Nombre de sharers remplis (0..4) ou compteur si G=1
    int WaitingUnblocks;
}
```

**Impact sur les transitions :**

1. **`g_sendInvalidations` (G=0) :** Au lieu d'un seul message multicast avec `NetDest`, envoyer un message point-à-point par sharer dans la liste. Itérer sur Sharer0..Sharer3 si non null.

2. **`g_sendInvalidations` (G=1) :** Envoyer un broadcast (destination = tous les nœuds). Le champ `Acks` dans DATA contient `NumSharers` (le compteur).

3. **Ajout d'un sharer :**
   - Si G=0 et liste non pleine : ajouter le CID dans le prochain slot.
   - Si G=0 et liste pleine : G ← 1, NumSharers ← k+1 (les k existants + le nouveau).
   - Si G=1 : NumSharers += 1.

4. **Suppression d'un sharer (PutS / éviction explicite) :**
   - Si G=0 : retirer le CID de la liste, tasser.
   - Si G=1 : NumSharers -= 1. Si NumSharers ≤ k : impossible de revenir en G=0 car on a perdu les identités.

**Avantages :**
- Simple à implémenter dans SLICC.
- Structure de données fixe et petite (~61 bits pour k=5, N=1024).
- Pas besoin de tas partagé ni de logique combinatoire complexe.

**Inconvénients :**
- Taux de broadcasts élevé dès que G=1 (×12.67 par rapport à DCC dans les benchmarks de la thèse).
- Irréversibilité : une fois G=1, on ne peut jamais revenir en mode précis sans invalider tout le monde.
- Trafic réseau supérieur (+16.8% de trafic par rapport au champ de bits complet).

### 9.2 Option 2 : Dir_i_B (Limited Pointer + Broadcast)

Identique à Ackwise dans le principe mais sans le concept de "keeper" :
- i pointeurs (log₂N bits chacun).
- Quand plein → broadcast sur le prochain GETX.
- Plus simple qu'Ackwise car pas de distinction keeper/sharer.

C'est essentiellement ce que fait Ackwise sans la notion d'Owner séparé. Dans le protocole MOESI_CMP_directory, on a déjà un `Owner` séparé, donc la distinction existe.

### 9.3 Option 3 : DCC (plus performant mais plus complexe)

Rectangle cohérent + liste chaînée + tas partagé. Voir rapport 3 pour les détails. Plus performant (×12 moins de broadcasts qu'Ackwise) mais nécessite :
- Un bloc tiling combinatoire (complexité 2^n).
- Un tas partagé entre toutes les lignes du cache L2.
- Une logique de conversion CID ↔ coordonnées (2D mesh).

### 9.4 Option 4 : Approche hybride simplifiée (recommandée pour commencer)

**Idée : Dir_k_B sans Ackwise complet — "Limited Pointer with Broadcast Fallback"**

```slicc
structure(Entry, desc="Entrée directory limitée") {
    State DirectoryState;
    NetDest Owner;              // 1 bit owner (inchangé)
    bool OverflowBit;           // Mode broadcast actif
    int SharerCount;            // Compteur total de sharers
    MachineID SharerList[K];    // K pointeurs (K=4 ou 5)
    int SharerListSize;         // Nombre d'entrées utilisées (0..K)
    int WaitingUnblocks;
}
```

**Logique simplifiée :**

```
Ajout sharer (X) :
  if OverflowBit == 0:
    if SharerListSize < K:
      SharerList[SharerListSize] = X
      SharerListSize++
      SharerCount++
    else:
      OverflowBit = 1
      SharerCount++
  else:
    SharerCount++

Invalidation :
  if OverflowBit == 0:
    envoyer INV à chaque SharerList[i] (multicast ciblé)
    AckCount = SharerListSize (- 1 si demandeur est dans la liste)
  else:
    envoyer INV en broadcast (tous les nœuds)
    AckCount = SharerCount (- 1 si demandeur est un sharer)

Suppression sharer (PutS de X) :
  SharerCount--
  if OverflowBit == 0:
    retirer X de SharerList, tasser
    SharerListSize--
  else:
    if SharerCount <= K:
      // IMPOSSIBLE de revenir en mode précis (on ne sait plus qui sont les sharers)
      // Option : rester en broadcast
      // Option avancée (Ackwise-like) : demander à tous les caches de se réidentifier
```

---

## 10. Recommandation : par quoi commencer ?

### 10.1 Plan d'implémentation progressif

**Phase 1 — Directory à taille limitée (2-3 semaines) :**
1. Copier le protocole `MOESI_CMP_directory` en `MOESI_CMP_limited_dir`.
2. Remplacer `DirectoryMemory` par `CacheMemory` dans `-dir.sm`.
3. Implémenter le mécanisme de recall (invalidation + writeback sur éviction d'entrée).
4. Tester avec une taille de directory variable et mesurer l'impact des recalls.

**Phase 2 — Sharelist limitée type Ackwise simplifié (2-3 semaines) :**
1. Remplacer `NetDest Sharers` par une structure à K pointeurs + overflow bit.
2. Modifier `g_sendInvalidations` pour gérer les deux modes (précis/broadcast).
3. Modifier `d_sendDataMsg` pour calculer l'AckCount différemment.
4. Modifier `m_addUnlockerToSharers` pour la logique d'ajout.
5. Ajouter la logique de suppression dans les transitions PutS.

**Phase 3 (optionnelle) — DCC ou amélioration :**
- Si le temps le permet, implémenter DCC avec le bloc tiling.

### 10.2 Faut-il tester Ackwise d'abord ?

**Oui, Ackwise est le bon point de départ** pour plusieurs raisons :

1. **Simplicité :** La structure de données est fixe (pas de tas, pas de liste chaînée). L'implémentation SLICC reste proche du protocole original.

2. **Pas de dépendance topologique :** Ackwise ne nécessite pas de réseau 2D mesh ni de conversion CID↔coordonnées, contrairement à DCC.

3. **Protocole MOESI déjà en place :** Le protocole source est MOESI, Ackwise est conçu pour MOESI, la correspondance est directe.

4. **Comparaison avec DCC ensuite :** Une fois Ackwise fonctionnel, on aura un baseline pour comparer avec DCC.

5. **Pas besoin d'un mécanisme plus simple :** Ackwise EST la version la plus simple d'un directory limité avec gestion de l'overflow. Un Dir_i_B pur est essentiellement identique. Il n'y a pas vraiment "plus simple" qui soit intéressant scientifiquement.

### 10.3 Peut-on faire plus simple qu'Ackwise ?

**L'approche la plus simple possible est Dir_0_B** (pas de pointeurs du tout) :
- L'entrée ne stocke que l'état + le compteur de sharers.
- TOUT GETX déclenche un broadcast.
- L'AckCount est toujours le compteur.

C'est trivialement implémentable mais les performances sont catastrophiques (proche du snooping). Ce n'est intéressant que comme baseline expérimentale.

**La véritable question est : Ackwise complet (avec keeper) ou juste des pointeurs limités ?**

Pour le stage, je recommande :
1. Commencer par **Dir_K_B** (K pointeurs + broadcast fallback, sans keeper séparé). C'est le cas de base.
2. Puis éventuellement enrichir en Ackwise complet (ajout du keeper qui forward les données directement).

Le concept de "keeper" dans Ackwise est essentiellement ce que le protocole MOESI_CMP_directory appelle déjà `Owner` — le cache qui possède la copie dirty et peut forward. **Le protocole a déjà ce mécanisme**. Donc implémenter "Ackwise" revient surtout à remplacer `NetDest Sharers` par la liste de pointeurs + G bit + compteur.

---

## 11. Résumé des modifications concrètes à apporter

### 11.1 Fichier `-dir.sm` — Modifications minimales pour Phase 2

| Section | Modification |
|---------|-------------|
| Paramètre machine | `CacheMemory * directoryCache` au lieu de `DirectoryMemory * directory` |
| Structure Entry | Remplacer `NetDest Sharers` par K pointeurs + overflow + compteur |
| `getDirectoryEntry` | Adapter au lookup dans CacheMemory |
| `allocateDirectoryEntry` | Gérer le cas "cache plein" (recall ou reject) |
| `g_sendInvalidations` | Mode précis (itérer la liste) ou broadcast (tous les nœuds) |
| `d_sendDataMsg` | AckCount = SharerListSize ou SharerCount selon OverflowBit |
| `m_addUnlockerToSharers` | Logique d'ajout avec overflow |
| Nouvelles actions PutS | Logique de suppression d'un sharer |
| Recall (nouveau) | Transitions pour éviction d'entrée directory |

### 11.2 Fichiers inchangés

- `-L1cache.sm` : pas de modification nécessaire (les évictions explicites sont déjà en place).
- `-L2cache.sm` : pas de modification nécessaire (il forward déjà correctement vers le directory global).
- `-msg.sm` : éventuellement ajouter un `RECALL` type si on veut un message spécifique pour le recall, sinon réutiliser INV.

---

## 12. Métriques à collecter pour l'évaluation

| Métrique | Description | Pertinence |
|----------|-------------|-----------|
| Taux de broadcasts | % de GETX qui déclenchent un broadcast (G=1) | Efficacité de la sharelist |
| Taux de recalls | % d'accès qui déclenchent une éviction du directory | Dimensionnement du dir cache |
| Latence moyenne | Cycles entre requête et satisfaction | Performance globale |
| Trafic réseau | Nombre de messages/flits dans Garnet | Pression réseau |
| Taille du directory | Nombre d'entrées × bits par entrée | Coût matériel |
| Hit rate directory | % de requêtes trouvant leur entrée dans le dir cache | Efficacité du cache |

---

## Annexe A : Table complète des transitions du Directory

| État initial | Événement | État final | Actions |
|-------------|-----------|-----------|---------|
| I | GETS | IS_M | allocDirEntry, v_allocateTBE, qf_queueMemoryFetchRequest, pop |
| I | GETX | MM_M | allocDirEntry, v_allocateTBE, qf_queueMemoryFetchRequest, pop |
| S | GETS | SS | v_allocateTBE, qf_queueMemoryFetchRequest, n_incrementOutstanding, pop |
| S | GETX | MM_M | v_allocateTBE, qf_queueMemoryFetchRequest, g_sendInvalidations, pop |
| O | GETS | OO | f_forwardRequest, n_incrementOutstanding, pop |
| O | GETX | MM | f_forwardRequest, g_sendInvalidations, pop |
| M | GETS | MO | v_allocateTBE, f_forwardRequest, pop |
| M | GETX | MM | f_forwardRequest, pop |
| M | PUTX | MI | v_allocateTBE, a_sendWriteBackAck, pop |
| O | PUTO | OS | a_sendWriteBackAck, pop |
| IS_M | Memory_Data_Cache | IS | d_sendDataMsg, q_popMemQueue |
| IS | Unblock | S | w_deallocateTBE, m_addUnlockerToSharers, pop |
| IS | Exclusive_Unblock | M | w_deallocateTBE, cc_clearSharers, e_ownerIsUnblocker, pop |
| SS | Unblock | (reste SS) | m_addUnlockerToSharers, o_decrementOutstanding, pop |
| SS | Last_Unblock | S | w_deallocateTBE, m_addUnlockerToSharers, o_decrementOutstanding, pop |
| MM_M | Memory_Data_Cache | MM | d_sendDataMsg, q_popMemQueue |
| MM | Exclusive_Unblock | M | w_deallocateTBE, cc_clearSharers, e_ownerIsUnblocker, pop |
| MO | Exclusive_Unblock | M | w_deallocateTBE, cc_clearSharers, e_ownerIsUnblocker, pop |
| MO | Unblock | O | w_deallocateTBE, m_addUnlockerToSharers, pop |
| MI | Dirty_Writeback | WBI | c_clearOwner, cc_clearSharers, qw_queueMemoryWB, pop |
| MI | Clean_Writeback | I | c_clearOwner, cc_clearSharers, w_deallocateTBE, deallocDirEntry, pop |
| WBI | Memory_Ack | I | clearWBAck, w_deallocateTBE, deallocDirEntry, q_popMemQueue |
| WBS | Memory_Ack | S | clearWBAck, w_deallocateTBE, q_popMemQueue |
| (transitoires) | GETS/GETX/PUT... | (même) | zz_recycleRequest |
